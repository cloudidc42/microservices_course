# Part 86: Message Brokers Comparison

## บทนำ

ในระบบ Microservices การสื่อสารแบบ Asynchronous ผ่าน Message Broker เป็นสิ่งสำคัญมาก บทนี้จะเปรียบเทียบ Message Broker ยอดนิยม 4 ตัว ได้แก่ Apache Kafka, RabbitMQ, NATS และ Apache Pulsar พร้อมตัวอย่างโค้ด TypeScript ที่ใช้งานได้จริง

---

## 1. การเปรียบเทียบ Message Brokers

### 1.1 ตารางเปรียบเทียบหลัก

| Feature | Kafka | RabbitMQ | NATS | Pulsar |
|---------|-------|----------|------|--------|
| Throughput | สูงมาก (มากกว่า 1M msg/s) | ปานกลาง (50K msg/s) | สูงมาก (10M msg/s) | สูงมาก (2M msg/s) |
| Latency | ต่ำ (1-5ms) | ต่ำมาก (<1ms) | ต่ำมาก (<1ms) | ต่ำ (1-5ms) |
| Message Ordering | Per-partition | Per-queue | ไม่รับประกัน | Per-partition |
| Durability | สูงมาก | สูง | ต้องการ JetStream | สูงมาก |
| Protocol | Custom (TCP) | AMQP 0-9-1 | NATS Protocol | Pulsar Protocol |
| Retention | Log-based (permanent) | Queue-based (ลบหลัง consume) | Memory/JetStream | Tiered Storage |
| Replay | รองรับ | ไม่รองรับ | JetStream เท่านั้น | รองรับ |
| Schema Registry | Confluent Schema Registry | ไม่มีในตัว | ไม่มีในตัว | มีในตัว |

### 1.2 Use Cases ที่เหมาะสม

**Kafka เหมาะกับ:**
- Event Sourcing และ CQRS
- Log aggregation
- Stream processing
- Data pipeline ขนาดใหญ่

**RabbitMQ เหมาะกับ:**
- Task queues
- Request/Reply patterns
- Complex routing (topic, fanout, direct)
- งานที่ต้องการ latency ต่ำมาก

**NATS เหมาะกับ:**
- Microservices messaging ที่เรียบง่าย
- IoT messaging
- Request/Reply แบบ synchronous
- Edge computing

**Pulsar เหมาะกับ:**
- Multi-tenant environments
- Geo-replication
- Tiered storage (hot/warm/cold)
- Unified messaging (Queue + Stream)

---

## 2. Apache Pulsar: TypeScript Producer/Consumer

### 2.1 ติดตั้ง Dependencies

```bash
npm install pulsar-client
npm install --save-dev @types/node
```

### 2.2 Pulsar Producer พร้อม Schema

```typescript
// pulsar/producer.ts
import Pulsar from 'pulsar-client';

// Define message schema
interface OrderEvent {
  orderId: string;
  customerId: string;
  items: Array<{
    productId: string;
    quantity: number;
    price: number;
  }>;
  totalAmount: number;
  status: 'CREATED' | 'CONFIRMED' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED';
  createdAt: string;
}

class PulsarOrderProducer {
  private client: Pulsar.Client;
  private producer: Pulsar.Producer | null = null;

  constructor(private readonly serviceUrl: string = 'pulsar://localhost:6650') {
    this.client = new Pulsar.Client({
      serviceUrl,
      operationTimeoutSeconds: 30,
      connectionTimeoutMs: 10000,
    });
  }

  async initialize(topicName: string): Promise<void> {
    this.producer = await this.client.createProducer({
      topic: topicName,
      // Partitioned topic configuration
      messageRoutingMode: 'RoundRobinDistribution',
      // Schema configuration
      schema: {
        type: Pulsar.SchemaType.Json,
        jsonSchema: JSON.stringify({
          type: 'record',
          name: 'OrderEvent',
          fields: [
            { name: 'orderId', type: 'string' },
            { name: 'customerId', type: 'string' },
            { name: 'totalAmount', type: 'double' },
            { name: 'status', type: 'string' },
            { name: 'createdAt', type: 'string' },
          ],
        }),
      },
      // Compression
      compressionType: 'ZSTD',
      // Batching
      batchingEnabled: true,
      batchingMaxPublishDelayMs: 10,
      batchingMaxMessages: 1000,
      // Send timeout
      sendTimeoutMs: 30000,
    });

    console.log(`Pulsar producer initialized for topic: ${topicName}`);
  }

  async sendOrder(order: OrderEvent, partitionKey?: string): Promise<string> {
    if (!this.producer) {
      throw new Error('Producer not initialized. Call initialize() first.');
    }

    const messageData = Buffer.from(JSON.stringify(order));

    const messageId = await this.producer.send({
      data: messageData,
      partitionKey: partitionKey || order.customerId,
      // Properties (headers)
      properties: {
        eventType: 'OrderEvent',
        version: '1.0',
        source: 'order-service',
        correlationId: order.orderId,
      },
      // Delivery delay (milliseconds)
      deliverAfter: 0,
      // Event time
      eventTimestamp: Date.now(),
    });

    console.log(`Order sent with messageId: ${messageId.toString()}`);
    return messageId.toString();
  }

  async sendBatch(orders: OrderEvent[]): Promise<string[]> {
    const messageIds: string[] = [];

    for (const order of orders) {
      const messageId = await this.sendOrder(order, order.customerId);
      messageIds.push(messageId);
    }

    return messageIds;
  }

  async flush(): Promise<void> {
    if (this.producer) {
      await this.producer.flush();
    }
  }

  async close(): Promise<void> {
    if (this.producer) {
      await this.producer.close();
    }
    await this.client.close();
    console.log('Pulsar producer closed');
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const producer = new PulsarOrderProducer('pulsar://localhost:6650');

  try {
    await producer.initialize('persistent://public/default/orders');

    const order: OrderEvent = {
      orderId: `order-${Date.now()}`,
      customerId: 'customer-001',
      items: [
        { productId: 'prod-001', quantity: 2, price: 299.99 },
        { productId: 'prod-002', quantity: 1, price: 149.99 },
      ],
      totalAmount: 749.97,
      status: 'CREATED',
      createdAt: new Date().toISOString(),
    };

    await producer.sendOrder(order);
    await producer.flush();
  } finally {
    await producer.close();
  }
}

main().catch(console.error);
```

### 2.3 Pulsar Consumer พร้อม Partitioned Topics

```typescript
// pulsar/consumer.ts
import Pulsar from 'pulsar-client';

interface OrderEvent {
  orderId: string;
  customerId: string;
  totalAmount: number;
  status: string;
  createdAt: string;
}

type MessageHandler = (order: OrderEvent, messageId: string) => Promise<void>;

class PulsarOrderConsumer {
  private client: Pulsar.Client;
  private consumer: Pulsar.Consumer | null = null;
  private isRunning = false;
  private processingCount = 0;

  constructor(
    private readonly serviceUrl: string = 'pulsar://localhost:6650',
    private readonly consumerConfig: {
      subscriptionName: string;
      subscriptionType: Pulsar.SubscriptionType;
      maxRedeliverCount?: number;
      deadLetterTopic?: string;
    }
  ) {
    this.client = new Pulsar.Client({
      serviceUrl,
      operationTimeoutSeconds: 30,
    });
  }

  async initialize(topicPattern: string | string[]): Promise<void> {
    const topics = Array.isArray(topicPattern)
      ? topicPattern
      : [topicPattern];

    this.consumer = await this.client.subscribe({
      // รองรับทั้ง single topic และ multiple topics
      topic: typeof topicPattern === 'string' ? topicPattern : undefined,
      topics: Array.isArray(topicPattern) ? topicPattern : undefined,
      // Subscription configuration
      subscription: this.consumerConfig.subscriptionName,
      subscriptionType: this.consumerConfig.subscriptionType,
      // Acknowledge timeout
      ackTimeoutMs: 10000,
      // Negative acknowledge redelivery delay
      nAckRedeliverTimeoutMs: 60000,
      // Dead Letter Queue
      deadLetterPolicy: this.consumerConfig.maxRedeliverCount
        ? {
            maxRedeliverCount: this.consumerConfig.maxRedeliverCount,
            deadLetterTopic: this.consumerConfig.deadLetterTopic || `${topics[0]}-DLQ`,
          }
        : undefined,
      // Pull mode configuration
      receiverQueueSize: 1000,
      // Priority level for Shared subscription
      priorityLevel: 0,
    });

    console.log(`Pulsar consumer initialized, subscription: ${this.consumerConfig.subscriptionName}`);
  }

  async startConsuming(handler: MessageHandler): Promise<void> {
    if (!this.consumer) {
      throw new Error('Consumer not initialized.');
    }

    this.isRunning = true;
    console.log('Starting to consume messages...');

    while (this.isRunning) {
      try {
        // รับ message ด้วย timeout
        const message = await this.consumer.receive(5000);

        if (message) {
          this.processingCount++;

          try {
            const data = JSON.parse(message.getData().toString()) as OrderEvent;
            const messageId = message.getMessageId().toString();

            // บันทึก metadata
            const properties = message.getProperties();
            console.log(`Received message: ${messageId}`, {
              eventType: properties['eventType'],
              topic: message.getTopicName(),
              partition: message.getMessageId().getPartition?.() ?? 'N/A',
            });

            // ประมวลผล message
            await handler(data, messageId);

            // Acknowledge สำเร็จ
            await this.consumer.acknowledge(message);
            this.processingCount--;
          } catch (processingError) {
            console.error('Error processing message:', processingError);
            // Negative acknowledge เพื่อให้ retry
            await this.consumer.negativeAcknowledge(message);
            this.processingCount--;
          }
        }
      } catch (receiveError: any) {
        if (receiveError.message?.includes('timeout')) {
          // Timeout ปกติ ไม่ใช่ error
          continue;
        }

        if (this.isRunning) {
          console.error('Error receiving message:', receiveError);
          await new Promise((resolve) => setTimeout(resolve, 1000));
        }
      }
    }
  }

  async stopConsuming(): Promise<void> {
    this.isRunning = false;

    // รอให้ประมวลผล messages ที่ค้างอยู่เสร็จ
    while (this.processingCount > 0) {
      await new Promise((resolve) => setTimeout(resolve, 100));
    }

    if (this.consumer) {
      await this.consumer.close();
    }
    await this.client.close();
    console.log('Pulsar consumer stopped');
  }
}

// Order processor handler
async function processOrder(order: OrderEvent, messageId: string): Promise<void> {
  console.log(`Processing order: ${order.orderId}`, {
    customerId: order.customerId,
    amount: order.totalAmount,
    status: order.status,
    messageId,
  });

  // จำลองการประมวลผล
  await new Promise((resolve) => setTimeout(resolve, 100));
  console.log(`Order ${order.orderId} processed successfully`);
}

// ตัวอย่างการใช้งาน
async function main() {
  const consumer = new PulsarOrderConsumer(
    'pulsar://localhost:6650',
    {
      subscriptionName: 'order-processor-subscription',
      subscriptionType: Pulsar.SubscriptionType.KeyShared,
      maxRedeliverCount: 3,
      deadLetterTopic: 'persistent://public/default/orders-DLQ',
    }
  );

  await consumer.initialize('persistent://public/default/orders');

  // Handle graceful shutdown
  process.on('SIGINT', async () => {
    console.log('Shutting down...');
    await consumer.stopConsuming();
    process.exit(0);
  });

  await consumer.startConsuming(processOrder);
}

main().catch(console.error);
```

---

## 3. NATS JetStream: TypeScript Publisher/Consumer

### 3.1 ติดตั้ง Dependencies

```bash
npm install nats
npm install --save-dev @types/node
```

### 3.2 NATS JetStream Publisher พร้อม Persistence

```typescript
// nats/jetstream-publisher.ts
import {
  connect,
  NatsConnection,
  JetStreamClient,
  JetStreamManager,
  StreamConfig,
  RetentionPolicy,
  StorageType,
  DiscardPolicy,
  StringCodec,
  JSONCodec,
} from 'nats';

interface EventMessage<T = unknown> {
  id: string;
  type: string;
  source: string;
  timestamp: string;
  data: T;
  metadata?: Record<string, string>;
}

interface UserCreatedEvent {
  userId: string;
  email: string;
  name: string;
  role: string;
}

class NatsJetStreamPublisher {
  private connection: NatsConnection | null = null;
  private js: JetStreamClient | null = null;
  private jsm: JetStreamManager | null = null;
  private readonly jc = JSONCodec();

  async connect(servers: string | string[] = 'nats://localhost:4222'): Promise<void> {
    this.connection = await connect({
      servers: Array.isArray(servers) ? servers : [servers],
      // Reconnect configuration
      maxReconnectAttempts: -1, // ไม่จำกัด
      reconnectTimeWait: 2000,
      // Authentication (optional)
      // token: process.env.NATS_TOKEN,
      // user: process.env.NATS_USER,
      // pass: process.env.NATS_PASS,
    });

    this.js = this.connection.jetstream();
    this.jsm = await this.connection.jetstreamManager();

    console.log('Connected to NATS JetStream');
  }

  async createStream(config: Partial<StreamConfig> & { name: string }): Promise<void> {
    if (!this.jsm) throw new Error('Not connected');

    const streamConfig: StreamConfig = {
      name: config.name,
      subjects: config.subjects || [`${config.name}.>`],
      // Retention policy
      retention: config.retention || RetentionPolicy.Limits,
      // Storage type
      storage: config.storage || StorageType.File,
      // Replication
      num_replicas: config.num_replicas || 1,
      // Limits
      max_msgs: config.max_msgs || 1_000_000,
      max_bytes: config.max_bytes || 1024 * 1024 * 1024, // 1GB
      max_age: config.max_age || 7 * 24 * 60 * 60 * 1e9, // 7 days in nanoseconds
      max_msg_size: config.max_msg_size || 1024 * 1024, // 1MB
      // Discard policy
      discard: config.discard || DiscardPolicy.Old,
      // Deduplication window
      duplicate_window: config.duplicate_window || 2 * 60 * 1e9, // 2 minutes
    };

    try {
      // ตรวจสอบว่า stream มีอยู่แล้วหรือไม่
      await this.jsm.streams.info(config.name);
      console.log(`Stream ${config.name} already exists, updating...`);
      await this.jsm.streams.update(config.name, streamConfig);
    } catch {
      // Stream ไม่มีอยู่ สร้างใหม่
      await this.jsm.streams.add(streamConfig);
      console.log(`Stream ${config.name} created`);
    }
  }

  async publish<T>(
    subject: string,
    data: T,
    options?: {
      messageId?: string;
      headers?: Record<string, string>;
      timeout?: number;
    }
  ): Promise<{ seq: number; duplicate: boolean }> {
    if (!this.js) throw new Error('Not connected');

    const event: EventMessage<T> = {
      id: options?.messageId || `msg-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      type: subject.split('.').pop() || 'unknown',
      source: process.env.SERVICE_NAME || 'unknown-service',
      timestamp: new Date().toISOString(),
      data,
      metadata: options?.headers,
    };

    // สร้าง headers
    const headers: Record<string, string> = {
      'Nats-Msg-Id': event.id, // สำหรับ deduplication
      'Content-Type': 'application/json',
      ...options?.headers,
    };

    // Publish พร้อม acknowledgment
    const ack = await this.js.publish(
      subject,
      this.jc.encode(event),
      {
        msgID: event.id, // Deduplication ID
        timeout: options?.timeout || 5000,
        headers: Object.entries(headers).reduce((acc, [k, v]) => {
          acc.set(k, v);
          return acc;
        }, new (require('nats').headers)()),
      }
    );

    console.log(`Published to ${subject}: seq=${ack.seq}, duplicate=${ack.duplicate}`);
    return { seq: ack.seq, duplicate: ack.duplicate };
  }

  async publishBatch<T>(
    subject: string,
    messages: T[],
    batchSize = 100
  ): Promise<Array<{ seq: number; duplicate: boolean }>> {
    const results: Array<{ seq: number; duplicate: boolean }> = [];

    for (let i = 0; i < messages.length; i += batchSize) {
      const batch = messages.slice(i, i + batchSize);
      const batchPromises = batch.map((msg) => this.publish(subject, msg));
      const batchResults = await Promise.all(batchPromises);
      results.push(...batchResults);
    }

    return results;
  }

  async close(): Promise<void> {
    if (this.connection) {
      await this.connection.drain();
      await this.connection.close();
      console.log('NATS connection closed');
    }
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const publisher = new NatsJetStreamPublisher();

  try {
    await publisher.connect(['nats://localhost:4222']);

    // สร้าง stream
    await publisher.createStream({
      name: 'USERS',
      subjects: ['users.>'],
      num_replicas: 1,
    });

    // Publish event
    const userEvent: UserCreatedEvent = {
      userId: `user-${Date.now()}`,
      email: 'user@example.com',
      name: 'John Doe',
      role: 'customer',
    };

    const result = await publisher.publish('users.created', userEvent, {
      headers: { 'X-Service': 'user-service' },
    });

    console.log('Published:', result);
  } finally {
    await publisher.close();
  }
}

main().catch(console.error);
```

### 3.3 NATS JetStream Consumer พร้อม Durable Subscription

```typescript
// nats/jetstream-consumer.ts
import {
  connect,
  NatsConnection,
  JetStreamClient,
  JetStreamManager,
  ConsumerConfig,
  AckPolicy,
  DeliverPolicy,
  ReplayPolicy,
  JSONCodec,
  NatsError,
} from 'nats';

interface EventMessage<T = unknown> {
  id: string;
  type: string;
  source: string;
  timestamp: string;
  data: T;
}

type EventHandler<T> = (event: EventMessage<T>, ack: () => Promise<void>, nak: (delay?: number) => Promise<void>) => Promise<void>;

class NatsJetStreamConsumer {
  private connection: NatsConnection | null = null;
  private js: JetStreamClient | null = null;
  private jsm: JetStreamManager | null = null;
  private readonly jc = JSONCodec();
  private isRunning = false;

  async connect(servers: string | string[] = 'nats://localhost:4222'): Promise<void> {
    this.connection = await connect({
      servers: Array.isArray(servers) ? servers : [servers],
      maxReconnectAttempts: -1,
      reconnectTimeWait: 2000,
    });

    this.js = this.connection.jetstream();
    this.jsm = await this.connection.jetstreamManager();
    console.log('Connected to NATS JetStream');
  }

  async createConsumer(
    streamName: string,
    consumerConfig: Partial<ConsumerConfig> & { durable_name: string }
  ): Promise<void> {
    if (!this.jsm) throw new Error('Not connected');

    const config: ConsumerConfig = {
      durable_name: consumerConfig.durable_name,
      // Deliver policy - ส่งข้อความจากที่ไหน
      deliver_policy: consumerConfig.deliver_policy || DeliverPolicy.New,
      // Ack policy
      ack_policy: consumerConfig.ack_policy || AckPolicy.Explicit,
      // Ack wait time (nanoseconds)
      ack_wait: consumerConfig.ack_wait || 30 * 1e9, // 30 seconds
      // Max delivery attempts
      max_deliver: consumerConfig.max_deliver || 3,
      // Filter subject
      filter_subject: consumerConfig.filter_subject,
      // Replay policy
      replay_policy: consumerConfig.replay_policy || ReplayPolicy.Instant,
      // Max ack pending
      max_ack_pending: consumerConfig.max_ack_pending || 1000,
      // Heartbeat
      idle_heartbeat: consumerConfig.idle_heartbeat,
    };

    try {
      await this.jsm.consumers.info(streamName, consumerConfig.durable_name);
      console.log(`Consumer ${consumerConfig.durable_name} already exists`);
    } catch {
      await this.jsm.consumers.add(streamName, config);
      console.log(`Consumer ${consumerConfig.durable_name} created`);
    }
  }

  async subscribe<T>(
    streamName: string,
    durableName: string,
    handler: EventHandler<T>,
    options?: {
      filterSubject?: string;
      maxMessages?: number;
      concurrency?: number;
    }
  ): Promise<void> {
    if (!this.js) throw new Error('Not connected');

    this.isRunning = true;

    // Subscribe ด้วย durable consumer
    const subscription = await this.js.subscribe(
      options?.filterSubject || `${streamName.toLowerCase()}.>`,
      {
        config: {
          durable_name: durableName,
          ack_policy: AckPolicy.Explicit,
          deliver_policy: DeliverPolicy.New,
          max_deliver: 3,
        },
      }
    );

    console.log(`Subscribed with durable: ${durableName}`);

    // ประมวลผล messages
    const semaphore = options?.concurrency || 10;
    let activeCount = 0;

    for await (const message of subscription) {
      if (!this.isRunning) break;

      // Concurrency control
      while (activeCount >= semaphore) {
        await new Promise((resolve) => setTimeout(resolve, 10));
      }

      activeCount++;

      // ประมวลผลแบบ async
      (async () => {
        try {
          const event = this.jc.decode(message.data) as EventMessage<T>;

          const ack = async () => {
            message.ack();
          };

          const nak = async (delay?: number) => {
            if (delay) {
              message.nak(delay);
            } else {
              message.nak();
            }
          };

          await handler(event, ack, nak);
        } catch (error) {
          console.error('Error processing message:', error);
          message.nak(5000); // Retry after 5 seconds
        } finally {
          activeCount--;
        }
      })();

      if (options?.maxMessages && subscription.getProcessed() >= options.maxMessages) {
        break;
      }
    }
  }

  async stop(): Promise<void> {
    this.isRunning = false;
    if (this.connection) {
      await this.connection.drain();
      await this.connection.close();
    }
    console.log('NATS consumer stopped');
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const consumer = new NatsJetStreamConsumer();

  try {
    await consumer.connect('nats://localhost:4222');

    await consumer.createConsumer('USERS', {
      durable_name: 'user-processor',
      filter_subject: 'users.>',
      max_deliver: 3,
    });

    process.on('SIGINT', async () => {
      await consumer.stop();
      process.exit(0);
    });

    await consumer.subscribe<{ userId: string; email: string }>(
      'USERS',
      'user-processor',
      async (event, ack, nak) => {
        console.log('Processing event:', event.type, event.data);
        try {
          // ประมวลผล
          await ack();
        } catch (error) {
          await nak(5000);
        }
      },
      { filterSubject: 'users.>', concurrency: 5 }
    );
  } catch (error) {
    console.error('Fatal error:', error);
    await consumer.stop();
  }
}

main().catch(console.error);
```

---

## 4. Message Ordering Strategies

### 4.1 Kafka Partition-based Ordering

```typescript
// kafka/ordering-producer.ts
import { Kafka, Producer, Message, CompressionTypes } from 'kafkajs';

interface OrderedMessage {
  key: string;   // Partition key สำหรับ ordering
  value: unknown;
  headers?: Record<string, string>;
}

class KafkaOrderedProducer {
  private kafka: Kafka;
  private producer: Producer;

  constructor(brokers: string[] = ['localhost:9092']) {
    this.kafka = new Kafka({
      clientId: 'ordered-producer',
      brokers,
      retry: {
        initialRetryTime: 100,
        retries: 8,
      },
    });

    this.producer = this.kafka.producer({
      // Idempotent producer สำหรับ exactly-once
      idempotent: true,
      // Max in-flight requests เมื่อใช้ idempotent
      maxInFlightRequests: 1,
      // Acks ต้องเป็น -1 (all) สำหรับ idempotent
      allowAutoTopicCreation: false,
    });
  }

  async connect(): Promise<void> {
    await this.producer.connect();
    console.log('Kafka producer connected');
  }

  // ส่ง messages พร้อม ordering guarantee ตาม key
  async sendOrdered(topic: string, messages: OrderedMessage[]): Promise<void> {
    const kafkaMessages: Message[] = messages.map((msg) => ({
      key: msg.key,
      value: JSON.stringify(msg.value),
      headers: msg.headers
        ? Object.entries(msg.headers).reduce(
            (acc, [k, v]) => ({ ...acc, [k]: Buffer.from(v) }),
            {}
          )
        : undefined,
    }));

    await this.producer.send({
      topic,
      messages: kafkaMessages,
      compression: CompressionTypes.GZIP,
    });

    console.log(`Sent ${messages.length} messages to ${topic}`);
  }

  // กลยุทธ์การเลือก partition key
  getPartitionKey(entityId: string, strategy: 'entity' | 'hash' | 'round-robin' = 'entity'): string {
    switch (strategy) {
      case 'entity':
        // ใช้ entity ID โดยตรง - messages ของ entity เดียวกันไปที่ partition เดียวกัน
        return entityId;

      case 'hash':
        // Hash ของ entity ID - กระจาย load ให้สม่ำเสมอ
        const hash = Array.from(entityId).reduce((acc, char) => {
          return ((acc << 5) - acc) + char.charCodeAt(0);
        }, 0);
        return Math.abs(hash).toString();

      case 'round-robin':
        // ไม่มี key - Kafka จะ round-robin โดยอัตโนมัติ
        return '';

      default:
        return entityId;
    }
  }

  async disconnect(): Promise<void> {
    await this.producer.disconnect();
    console.log('Kafka producer disconnected');
  }
}

// ตัวอย่างการใช้ ordering สำหรับ order events
async function demonstrateOrdering() {
  const producer = new KafkaOrderedProducer(['localhost:9092']);
  await producer.connect();

  try {
    // Events ของ order เดียวกัน - ต้องมี ordering guarantee
    const orderId = 'order-12345';
    const orderEvents: OrderedMessage[] = [
      {
        key: producer.getPartitionKey(orderId),
        value: { orderId, event: 'CREATED', timestamp: Date.now() },
      },
      {
        key: producer.getPartitionKey(orderId),
        value: { orderId, event: 'PAYMENT_INITIATED', timestamp: Date.now() + 100 },
      },
      {
        key: producer.getPartitionKey(orderId),
        value: { orderId, event: 'PAYMENT_CONFIRMED', timestamp: Date.now() + 200 },
      },
      {
        key: producer.getPartitionKey(orderId),
        value: { orderId, event: 'SHIPPED', timestamp: Date.now() + 300 },
      },
    ];

    await producer.sendOrdered('order-events', orderEvents);
    console.log(`Sent ${orderEvents.length} ordered events for order ${orderId}`);
  } finally {
    await producer.disconnect();
  }
}

demonstrateOrdering().catch(console.error);
```

---

## 5. At-least-once vs Exactly-once

### 5.1 Kafka Idempotent Producer (At-least-once ที่ปลอดภัยกว่า)

```typescript
// kafka/idempotent-producer.ts
import { Kafka, Producer, Transaction } from 'kafkajs';

class KafkaIdempotentProducer {
  private kafka: Kafka;
  private producer: Producer;

  constructor(brokers: string[] = ['localhost:9092']) {
    this.kafka = new Kafka({
      clientId: 'idempotent-producer',
      brokers,
    });

    // Idempotent producer - ป้องกัน duplicate เมื่อ retry
    this.producer = this.kafka.producer({
      idempotent: true,            // เปิด idempotent mode
      maxInFlightRequests: 5,      // จำกัด in-flight requests
      transactionalId: undefined,  // ไม่ใช้ transactions ในโหมดนี้
    });
  }

  async connect(): Promise<void> {
    await this.producer.connect();
  }

  // At-least-once ที่ปลอดภัย - ไม่มี duplicate จาก producer side
  async sendIdempotent(topic: string, key: string, value: unknown): Promise<void> {
    // แม้จะ retry ข้อความจะไม่ duplicate ใน Kafka
    await this.producer.send({
      topic,
      messages: [
        {
          key,
          value: JSON.stringify(value),
        },
      ],
      acks: -1,  // รอ ack จากทุก replica
    });
  }

  async disconnect(): Promise<void> {
    await this.producer.disconnect();
  }
}
```

### 5.2 Kafka Transactions (Exactly-once)

```typescript
// kafka/transactional-producer.ts
import { Kafka, Producer, Transaction, EachMessagePayload } from 'kafkajs';

class KafkaTransactionalProducer {
  private kafka: Kafka;
  private producer: Producer;

  constructor(
    brokers: string[] = ['localhost:9092'],
    private readonly transactionalId: string = 'my-transactional-producer'
  ) {
    this.kafka = new Kafka({
      clientId: 'transactional-producer',
      brokers,
    });

    // Transactional producer สำหรับ exactly-once
    this.producer = this.kafka.producer({
      idempotent: true,              // ต้องเปิด
      maxInFlightRequests: 1,        // ต้องเป็น 1 สำหรับ transactions
      transactionalId,               // Unique ID สำหรับ producer นี้
    });
  }

  async connect(): Promise<void> {
    await this.producer.connect();
    console.log(`Transactional producer connected: ${this.transactionalId}`);
  }

  // Exactly-once: อ่านจาก topic A แล้วเขียนไป topic B ใน transaction เดียว
  async processExactlyOnce(
    sourceMessages: EachMessagePayload[],
    processedTopic: string,
    transform: (message: unknown) => unknown
  ): Promise<void> {
    const transaction = await this.producer.transaction();

    try {
      // 1. ประมวลผล messages
      const processedMessages = sourceMessages.map((msg) => {
        const value = msg.message.value
          ? JSON.parse(msg.message.value.toString())
          : null;
        return {
          key: msg.message.key?.toString(),
          value: JSON.stringify(transform(value)),
          headers: { 'X-Source-Offset': msg.message.offset },
        };
      });

      // 2. เขียน messages ที่ประมวลผลแล้ว
      await transaction.send({
        topic: processedTopic,
        messages: processedMessages,
      });

      // 3. Commit offsets ใน transaction เดียวกัน (สำคัญมาก!)
      await transaction.sendOffsets({
        consumerGroupId: 'my-consumer-group',
        topics: sourceMessages.reduce<Record<string, Record<number, string>>>((acc, msg) => {
          if (!acc[msg.topic]) acc[msg.topic] = {};
          // +1 เพื่อ commit offset ถัดไป
          acc[msg.topic][msg.partition] = (parseInt(msg.message.offset) + 1).toString();
          return acc;
        }, {}),
      });

      // 4. Commit transaction
      await transaction.commit();
      console.log('Transaction committed successfully');
    } catch (error) {
      // Rollback transaction เมื่อเกิด error
      await transaction.abort();
      console.error('Transaction aborted:', error);
      throw error;
    }
  }

  async disconnect(): Promise<void> {
    await this.producer.disconnect();
  }
}

// ตัวอย่างการใช้งาน
async function demonstrateExactlyOnce() {
  const producer = new KafkaTransactionalProducer(
    ['localhost:9092'],
    'order-processor-txn-1'
  );

  await producer.connect();

  try {
    // จำลอง source messages
    const sourceMessages: EachMessagePayload[] = [
      {
        topic: 'raw-orders',
        partition: 0,
        message: {
          key: Buffer.from('order-001'),
          value: Buffer.from(JSON.stringify({ orderId: 'order-001', amount: 100 })),
          offset: '10',
          timestamp: Date.now().toString(),
          attributes: 0,
          headers: {},
          size: 0,
        },
        heartbeat: async () => {},
        pause: () => () => {},
      },
    ];

    // ประมวลผลแบบ exactly-once
    await producer.processExactlyOnce(
      sourceMessages,
      'processed-orders',
      (raw: any) => ({
        ...raw,
        processedAt: new Date().toISOString(),
        status: 'PROCESSED',
      })
    );
  } finally {
    await producer.disconnect();
  }
}

demonstrateExactlyOnce().catch(console.error);
```

---

## 6. Message Replay

### 6.1 Kafka Consumer จาก Specific Offset

```typescript
// kafka/replay-consumer.ts
import { Kafka, Consumer, EachMessagePayload, SeekEntry } from 'kafkajs';

interface ReplayOptions {
  topic: string;
  partition: number;
  fromOffset: string | 'earliest' | 'latest';
  toOffset?: string;
  fromTimestamp?: number; // Unix timestamp ใน milliseconds
}

class KafkaReplayConsumer {
  private kafka: Kafka;
  private consumer: Consumer;

  constructor(
    brokers: string[] = ['localhost:9092'],
    private readonly groupId: string = 'replay-consumer'
  ) {
    this.kafka = new Kafka({
      clientId: 'replay-consumer',
      brokers,
    });

    this.consumer = this.kafka.consumer({ groupId });
  }

  async connect(): Promise<void> {
    await this.consumer.connect();
    console.log('Kafka replay consumer connected');
  }

  // Replay จาก specific offset
  async replayFromOffset(
    options: ReplayOptions,
    handler: (payload: EachMessagePayload) => Promise<void>
  ): Promise<void> {
    await this.consumer.subscribe({
      topic: options.topic,
      fromBeginning: options.fromOffset === 'earliest',
    });

    // Seek ไปที่ offset ที่ต้องการ
    this.consumer.on('consumer.group_join', async () => {
      if (options.fromOffset !== 'earliest' && options.fromOffset !== 'latest') {
        await this.consumer.seek({
          topic: options.topic,
          partition: options.partition,
          offset: options.fromOffset,
        });
        console.log(`Seeking to offset ${options.fromOffset} on partition ${options.partition}`);
      }
    });

    await this.consumer.run({
      eachMessage: async (payload) => {
        // หยุดเมื่อถึง toOffset
        if (
          options.toOffset &&
          parseInt(payload.message.offset) > parseInt(options.toOffset)
        ) {
          await this.consumer.stop();
          return;
        }

        await handler(payload);
      },
    });
  }

  // Replay จาก timestamp
  async replayFromTimestamp(
    topic: string,
    partitions: number[],
    fromTimestamp: number,
    handler: (payload: EachMessagePayload) => Promise<void>
  ): Promise<void> {
    const admin = this.kafka.admin();
    await admin.connect();

    try {
      // หา offsets ที่ตรงกับ timestamp
      const offsetsByTime = await admin.fetchTopicOffsetsByTime(topic, fromTimestamp);

      await this.consumer.subscribe({ topic, fromBeginning: false });

      // Seek ไปที่ offsets ที่คำนวณได้
      this.consumer.on('consumer.group_join', async () => {
        for (const offsetInfo of offsetsByTime) {
          if (partitions.includes(offsetInfo.partition)) {
            await this.consumer.seek({
              topic,
              partition: offsetInfo.partition,
              offset: offsetInfo.offset,
            });
            console.log(`Replay partition ${offsetInfo.partition} from offset ${offsetInfo.offset}`);
          }
        }
      });

      await this.consumer.run({
        eachMessage: handler,
      });
    } finally {
      await admin.disconnect();
    }
  }

  async stop(): Promise<void> {
    await this.consumer.stop();
    await this.consumer.disconnect();
    console.log('Replay consumer stopped');
  }
}

// ตัวอย่าง: Replay events จาก 1 ชั่วโมงที่แล้ว
async function replayLastHour() {
  const consumer = new KafkaReplayConsumer(['localhost:9092'], 'replay-group-001');
  await consumer.connect();

  const oneHourAgo = Date.now() - 60 * 60 * 1000;

  await consumer.replayFromTimestamp(
    'order-events',
    [0, 1, 2], // partitions
    oneHourAgo,
    async ({ message, partition, topic }) => {
      const event = JSON.parse(message.value?.toString() || '{}');
      console.log(`Replaying: partition=${partition}, offset=${message.offset}`, event);
    }
  );
}
```

### 6.2 NATS Replay Policy

```typescript
// nats/replay-consumer.ts
import {
  connect,
  JetStreamManager,
  DeliverPolicy,
  ReplayPolicy,
} from 'nats';

async function setupReplayConsumer() {
  const nc = await connect({ servers: 'nats://localhost:4222' });
  const jsm = await nc.jetstreamManager();
  const js = nc.jetstream();

  // Consumer สำหรับ replay ทั้งหมด
  await jsm.consumers.add('ORDERS', {
    durable_name: 'order-replay-all',
    deliver_policy: DeliverPolicy.All,  // เริ่มจากข้อความแรกสุด
    ack_policy: require('nats').AckPolicy.Explicit,
    replay_policy: ReplayPolicy.Instant, // ส่งทันที
  });

  // Consumer สำหรับ replay จาก timestamp
  const fromTime = new Date(Date.now() - 3600000); // 1 ชั่วโมงที่แล้ว
  await jsm.consumers.add('ORDERS', {
    durable_name: 'order-replay-from-time',
    deliver_policy: DeliverPolicy.StartTime,
    opt_start_time: fromTime.toISOString(),
    ack_policy: require('nats').AckPolicy.Explicit,
  });

  // Consumer สำหรับ replay จาก sequence number
  await jsm.consumers.add('ORDERS', {
    durable_name: 'order-replay-from-seq',
    deliver_policy: DeliverPolicy.StartSequence,
    opt_start_seq: 100, // เริ่มจาก sequence 100
    ack_policy: require('nats').AckPolicy.Explicit,
  });

  // Consumer แบบ Original Rate Replay (replay ด้วย rate เดิม)
  await jsm.consumers.add('ORDERS', {
    durable_name: 'order-replay-original-rate',
    deliver_policy: DeliverPolicy.All,
    replay_policy: ReplayPolicy.Original, // ส่งในอัตราเดิม
    ack_policy: require('nats').AckPolicy.Explicit,
  });

  await nc.drain();
  await nc.close();
}

setupReplayConsumer().catch(console.error);
```

---

## 7. Dead Letter Queue (DLQ) Patterns

### 7.1 Universal DLQ Handler

```typescript
// dlq/dlq-handler.ts
import { Kafka, Consumer, Producer, EachMessagePayload } from 'kafkajs';

interface DLQMessage {
  originalTopic: string;
  originalPartition: number;
  originalOffset: string;
  originalKey: string | null;
  originalValue: string | null;
  originalHeaders: Record<string, string>;
  errorMessage: string;
  errorStack?: string;
  failedAt: string;
  retryCount: number;
  maxRetries: number;
}

class KafkaDLQHandler {
  private kafka: Kafka;
  private producer: Producer;
  private consumer: Consumer;
  private dlqConsumer: Consumer;

  constructor(
    brokers: string[] = ['localhost:9092'],
    private readonly config: {
      sourceTopic: string;
      dlqTopic: string;
      retryTopic: string;
      consumerGroupId: string;
      maxRetries: number;
      retryDelayMs: number;
    }
  ) {
    this.kafka = new Kafka({ clientId: 'dlq-handler', brokers });
    this.producer = this.kafka.producer({ idempotent: true });
    this.consumer = this.kafka.consumer({ groupId: config.consumerGroupId });
    this.dlqConsumer = this.kafka.consumer({
      groupId: `${config.consumerGroupId}-dlq-processor`,
    });
  }

  async connect(): Promise<void> {
    await this.producer.connect();
    await this.consumer.connect();
    await this.dlqConsumer.connect();
  }

  // ส่ง message ไปยัง DLQ
  async sendToDLQ(
    originalPayload: EachMessagePayload,
    error: Error,
    retryCount: number
  ): Promise<void> {
    const dlqMessage: DLQMessage = {
      originalTopic: originalPayload.topic,
      originalPartition: originalPayload.partition,
      originalOffset: originalPayload.message.offset,
      originalKey: originalPayload.message.key?.toString() || null,
      originalValue: originalPayload.message.value?.toString() || null,
      originalHeaders: Object.entries(originalPayload.message.headers || {}).reduce(
        (acc, [k, v]) => ({
          ...acc,
          [k]: Buffer.isBuffer(v) ? v.toString() : String(v),
        }),
        {}
      ),
      errorMessage: error.message,
      errorStack: error.stack,
      failedAt: new Date().toISOString(),
      retryCount,
      maxRetries: this.config.maxRetries,
    };

    await this.producer.send({
      topic: this.config.dlqTopic,
      messages: [
        {
          key: originalPayload.message.key,
          value: JSON.stringify(dlqMessage),
          headers: {
            'X-DLQ-Original-Topic': Buffer.from(originalPayload.topic),
            'X-DLQ-Error': Buffer.from(error.message),
            'X-DLQ-Retry-Count': Buffer.from(retryCount.toString()),
          },
        },
      ],
    });

    console.error(
      `Message sent to DLQ: topic=${this.config.dlqTopic}, retries=${retryCount}`,
      { error: error.message }
    );
  }

  // ประมวลผล message พร้อม retry logic
  async processWithRetry(
    payload: EachMessagePayload,
    handler: (payload: EachMessagePayload) => Promise<void>
  ): Promise<void> {
    let retryCount = 0;

    // อ่าน retry count จาก headers
    const retryCountHeader = payload.message.headers?.['X-Retry-Count'];
    if (retryCountHeader) {
      retryCount = parseInt(
        Buffer.isBuffer(retryCountHeader) ? retryCountHeader.toString() : String(retryCountHeader)
      );
    }

    try {
      await handler(payload);
    } catch (error) {
      retryCount++;

      if (retryCount >= this.config.maxRetries) {
        // ส่งไป DLQ
        await this.sendToDLQ(payload, error as Error, retryCount);
      } else {
        // ส่งไป retry topic พร้อม delay
        await this.sendToRetry(payload, error as Error, retryCount);
      }
    }
  }

  // ส่งไป retry topic
  private async sendToRetry(
    payload: EachMessagePayload,
    error: Error,
    retryCount: number
  ): Promise<void> {
    const retryDelay = this.config.retryDelayMs * Math.pow(2, retryCount - 1); // Exponential backoff

    await this.producer.send({
      topic: this.config.retryTopic,
      messages: [
        {
          key: payload.message.key,
          value: payload.message.value,
          headers: {
            ...payload.message.headers,
            'X-Retry-Count': Buffer.from(retryCount.toString()),
            'X-Retry-After': Buffer.from((Date.now() + retryDelay).toString()),
            'X-Original-Topic': Buffer.from(payload.topic),
            'X-Last-Error': Buffer.from(error.message),
          },
        },
      ],
    });

    console.log(`Message sent to retry topic: retry=${retryCount}, delay=${retryDelay}ms`);
  }

  // DLQ consumer สำหรับ manual inspection และ reprocessing
  async startDLQProcessor(
    reprocessHandler?: (dlqMessage: DLQMessage) => Promise<boolean>
  ): Promise<void> {
    await this.dlqConsumer.subscribe({
      topic: this.config.dlqTopic,
      fromBeginning: false,
    });

    await this.dlqConsumer.run({
      eachMessage: async (payload) => {
        const dlqMessage = JSON.parse(
          payload.message.value?.toString() || '{}'
        ) as DLQMessage;

        console.log('DLQ message received:', {
          originalTopic: dlqMessage.originalTopic,
          error: dlqMessage.errorMessage,
          retryCount: dlqMessage.retryCount,
          failedAt: dlqMessage.failedAt,
        });

        // ถ้ามี reprocess handler ให้ลอง reprocess
        if (reprocessHandler) {
          const shouldReprocess = await reprocessHandler(dlqMessage);

          if (shouldReprocess && dlqMessage.originalValue) {
            // ส่ง message กลับไปที่ original topic
            await this.producer.send({
              topic: dlqMessage.originalTopic,
              messages: [
                {
                  key: dlqMessage.originalKey
                    ? Buffer.from(dlqMessage.originalKey)
                    : null,
                  value: Buffer.from(dlqMessage.originalValue),
                  headers: {
                    'X-Reprocessed-From-DLQ': Buffer.from('true'),
                    'X-Original-Failed-At': Buffer.from(dlqMessage.failedAt),
                  },
                },
              ],
            });
            console.log(`Message reprocessed from DLQ to ${dlqMessage.originalTopic}`);
          }
        }
      },
    });
  }

  async disconnect(): Promise<void> {
    await this.producer.disconnect();
    await this.consumer.disconnect();
    await this.dlqConsumer.disconnect();
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const dlqHandler = new KafkaDLQHandler(['localhost:9092'], {
    sourceTopic: 'orders',
    dlqTopic: 'orders-dlq',
    retryTopic: 'orders-retry',
    consumerGroupId: 'order-processor',
    maxRetries: 3,
    retryDelayMs: 1000,
  });

  await dlqHandler.connect();

  // เริ่ม DLQ processor
  await dlqHandler.startDLQProcessor(async (dlqMessage) => {
    // logic สำหรับตัดสินใจว่าจะ reprocess หรือไม่
    const isTransientError = dlqMessage.errorMessage.includes('timeout');
    return isTransientError;
  });
}

main().catch(console.error);
```

---

## 8. Broker Monitoring: Prometheus Metrics

### 8.1 Kafka Metrics Configuration

```yaml
# kubernetes/kafka-prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kafka-metrics-config
  namespace: kafka
data:
  metrics.yaml: |
    lowercaseOutputName: true
    lowercaseOutputLabelNames: true
    rules:
      # Broker metrics
      - pattern: kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec, topic=(.+)><>Count
        name: kafka_server_brokertopicmetrics_messagesinpersec_count
        labels:
          topic: "$1"

      - pattern: kafka.server<type=BrokerTopicMetrics, name=BytesInPerSec, topic=(.+)><>Count
        name: kafka_server_brokertopicmetrics_bytesinpersec_count
        labels:
          topic: "$1"

      - pattern: kafka.server<type=BrokerTopicMetrics, name=BytesOutPerSec, topic=(.+)><>Count
        name: kafka_server_brokertopicmetrics_bytesoutpersec_count
        labels:
          topic: "$1"

      # Consumer lag
      - pattern: kafka.server<type=FetcherLagMetrics, name=ConsumerLag, clientId=(.+), topic=(.+), partition=(.+)><>Value
        name: kafka_server_fetcherlagmetrics_consumerlag
        labels:
          client_id: "$1"
          topic: "$2"
          partition: "$3"

      # Under replicated partitions
      - pattern: kafka.server<type=ReplicaManager, name=UnderReplicatedPartitions><>Value
        name: kafka_server_replicamanager_underreplicatedpartitions

      # Active controller count
      - pattern: kafka.controller<type=KafkaController, name=ActiveControllerCount><>Value
        name: kafka_controller_kafkacontroller_activecontrollercount

      # Request handler idle ratio
      - pattern: kafka.server<type=KafkaRequestHandlerPool, name=RequestHandlerAvgIdlePercent><>MeanRate
        name: kafka_server_kafkarequesthandlerpool_requesthandleravgidlepercent_meanrate

---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: kafka-broker
  namespace: kafka
spec:
  selector:
    matchLabels:
      app: kafka
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### 8.2 Prometheus Alerts สำหรับ Kafka

```yaml
# monitoring/kafka-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kafka-alerts
  namespace: monitoring
spec:
  groups:
    - name: kafka.rules
      interval: 30s
      rules:
        # Consumer lag สูง
        - alert: KafkaConsumerLagHigh
          expr: |
            kafka_consumergroup_lag_sum > 10000
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Kafka consumer lag is high"
            description: "Consumer group {{ $labels.consumergroup }} has lag {{ $value }} on topic {{ $labels.topic }}"

        # Under-replicated partitions
        - alert: KafkaUnderReplicatedPartitions
          expr: |
            kafka_server_replicamanager_underreplicatedpartitions > 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Kafka has under-replicated partitions"
            description: "Broker {{ $labels.instance }} has {{ $value }} under-replicated partitions"

        # No active controller
        - alert: KafkaNoActiveController
          expr: |
            sum(kafka_controller_kafkacontroller_activecontrollercount) == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Kafka has no active controller"

        # Offline partitions
        - alert: KafkaOfflinePartitions
          expr: |
            kafka_controller_kafkacontroller_offlinepartitionscount > 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Kafka has offline partitions"

        # Broker down
        - alert: KafkaBrokerDown
          expr: |
            up{job="kafka"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Kafka broker is down"
```

### 8.3 TypeScript Prometheus Client สำหรับ Custom Metrics

```typescript
// monitoring/broker-metrics.ts
import { Registry, Counter, Gauge, Histogram, collectDefaultMetrics } from 'prom-client';

class MessageBrokerMetrics {
  private readonly registry: Registry;

  // Counters
  readonly messagesProduced: Counter<string>;
  readonly messagesConsumed: Counter<string>;
  readonly messagesFailed: Counter<string>;
  readonly dlqMessages: Counter<string>;

  // Gauges
  readonly consumerLag: Gauge<string>;
  readonly activeConsumers: Gauge<string>;
  readonly connectionPoolSize: Gauge<string>;

  // Histograms
  readonly producerLatency: Histogram<string>;
  readonly consumerProcessingDuration: Histogram<string>;
  readonly messageSize: Histogram<string>;

  constructor(prefix = 'message_broker') {
    this.registry = new Registry();
    collectDefaultMetrics({ register: this.registry });

    this.messagesProduced = new Counter({
      name: `${prefix}_messages_produced_total`,
      help: 'Total number of messages produced',
      labelNames: ['topic', 'broker', 'status'],
      registers: [this.registry],
    });

    this.messagesConsumed = new Counter({
      name: `${prefix}_messages_consumed_total`,
      help: 'Total number of messages consumed',
      labelNames: ['topic', 'consumer_group', 'status'],
      registers: [this.registry],
    });

    this.messagesFailed = new Counter({
      name: `${prefix}_messages_failed_total`,
      help: 'Total number of failed message processings',
      labelNames: ['topic', 'consumer_group', 'error_type'],
      registers: [this.registry],
    });

    this.dlqMessages = new Counter({
      name: `${prefix}_dlq_messages_total`,
      help: 'Total number of messages sent to DLQ',
      labelNames: ['source_topic', 'dlq_topic', 'error_type'],
      registers: [this.registry],
    });

    this.consumerLag = new Gauge({
      name: `${prefix}_consumer_lag`,
      help: 'Current consumer lag per topic partition',
      labelNames: ['topic', 'partition', 'consumer_group'],
      registers: [this.registry],
    });

    this.activeConsumers = new Gauge({
      name: `${prefix}_active_consumers`,
      help: 'Number of active consumers',
      labelNames: ['consumer_group'],
      registers: [this.registry],
    });

    this.connectionPoolSize = new Gauge({
      name: `${prefix}_connection_pool_size`,
      help: 'Current connection pool size',
      labelNames: ['broker', 'type'],
      registers: [this.registry],
    });

    this.producerLatency = new Histogram({
      name: `${prefix}_producer_latency_seconds`,
      help: 'Producer message publish latency in seconds',
      labelNames: ['topic', 'broker'],
      buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
      registers: [this.registry],
    });

    this.consumerProcessingDuration = new Histogram({
      name: `${prefix}_consumer_processing_duration_seconds`,
      help: 'Consumer message processing duration in seconds',
      labelNames: ['topic', 'consumer_group'],
      buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 5, 10],
      registers: [this.registry],
    });

    this.messageSize = new Histogram({
      name: `${prefix}_message_size_bytes`,
      help: 'Message size in bytes',
      labelNames: ['topic'],
      buckets: [100, 1000, 10000, 100000, 1000000],
      registers: [this.registry],
    });
  }

  async getMetrics(): Promise<string> {
    return this.registry.metrics();
  }

  getContentType(): string {
    return this.registry.contentType;
  }
}

// Singleton instance
export const brokerMetrics = new MessageBrokerMetrics();

// Express endpoint สำหรับ /metrics
import express from 'express';

const app = express();

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', brokerMetrics.getContentType());
  res.send(await brokerMetrics.getMetrics());
});

app.listen(9090, () => {
  console.log('Metrics server listening on :9090');
});
```

---

## 9. Migration Guide: RabbitMQ → Kafka

### 9.1 TypeScript Migration Example

```typescript
// migration/rabbitmq-to-kafka-migration.ts
import amqplib, { Connection, Channel } from 'amqplib';
import { Kafka, Producer, Consumer } from 'kafkajs';

// =========================================================
// Step 1: สร้าง Abstraction Layer
// =========================================================

interface MessageBrokerAdapter {
  connect(): Promise<void>;
  publish(topic: string, message: unknown, key?: string): Promise<void>;
  subscribe(topic: string, handler: (message: unknown) => Promise<void>): Promise<void>;
  disconnect(): Promise<void>;
}

// =========================================================
// Step 2: RabbitMQ Adapter (existing)
// =========================================================

class RabbitMQAdapter implements MessageBrokerAdapter {
  private connection: Connection | null = null;
  private channel: Channel | null = null;

  constructor(private readonly url: string = 'amqp://localhost') {}

  async connect(): Promise<void> {
    this.connection = await amqplib.connect(this.url);
    this.channel = await this.connection.createChannel();
    console.log('Connected to RabbitMQ');
  }

  async publish(exchange: string, message: unknown, routingKey = ''): Promise<void> {
    if (!this.channel) throw new Error('Not connected');

    await this.channel.assertExchange(exchange, 'topic', { durable: true });
    this.channel.publish(
      exchange,
      routingKey,
      Buffer.from(JSON.stringify(message)),
      { persistent: true }
    );
  }

  async subscribe(queue: string, handler: (message: unknown) => Promise<void>): Promise<void> {
    if (!this.channel) throw new Error('Not connected');

    await this.channel.assertQueue(queue, { durable: true });
    this.channel.prefetch(1);

    await this.channel.consume(queue, async (msg) => {
      if (!msg) return;

      try {
        const content = JSON.parse(msg.content.toString());
        await handler(content);
        this.channel?.ack(msg);
      } catch (error) {
        this.channel?.nack(msg, false, false);
        console.error('Error processing message:', error);
      }
    });
  }

  async disconnect(): Promise<void> {
    await this.channel?.close();
    await this.connection?.close();
    console.log('Disconnected from RabbitMQ');
  }
}

// =========================================================
// Step 3: Kafka Adapter (target)
// =========================================================

class KafkaAdapter implements MessageBrokerAdapter {
  private kafka: Kafka;
  private producer: Producer | null = null;
  private consumers: Map<string, Consumer> = new Map();
  private consumerGroupCounter = 0;

  constructor(brokers: string[] = ['localhost:9092']) {
    this.kafka = new Kafka({
      clientId: 'migrated-service',
      brokers,
    });
  }

  async connect(): Promise<void> {
    this.producer = this.kafka.producer({ idempotent: true });
    await this.producer.connect();
    console.log('Connected to Kafka');
  }

  async publish(topic: string, message: unknown, key?: string): Promise<void> {
    if (!this.producer) throw new Error('Not connected');

    await this.producer.send({
      topic,
      messages: [
        {
          key: key ? Buffer.from(key) : undefined,
          value: Buffer.from(JSON.stringify(message)),
          headers: {
            'Content-Type': Buffer.from('application/json'),
            'Produced-At': Buffer.from(new Date().toISOString()),
          },
        },
      ],
    });
  }

  async subscribe(topic: string, handler: (message: unknown) => Promise<void>): Promise<void> {
    const groupId = `migrated-consumer-${this.consumerGroupCounter++}`;
    const consumer = this.kafka.consumer({ groupId });
    this.consumers.set(topic, consumer);

    await consumer.connect();
    await consumer.subscribe({ topic, fromBeginning: false });

    await consumer.run({
      eachMessage: async ({ message }) => {
        try {
          const content = JSON.parse(message.value?.toString() || '{}');
          await handler(content);
        } catch (error) {
          console.error('Error processing Kafka message:', error);
          throw error; // Kafka consumer จะ retry อัตโนมัติ
        }
      },
    });
  }

  async disconnect(): Promise<void> {
    await this.producer?.disconnect();
    for (const consumer of this.consumers.values()) {
      await consumer.disconnect();
    }
    console.log('Disconnected from Kafka');
  }
}

// =========================================================
// Step 4: Migration Strategy - Dual Write
// =========================================================

class DualWriteMigrationAdapter implements MessageBrokerAdapter {
  private rabbitMQ: RabbitMQAdapter;
  private kafka: KafkaAdapter;
  private migrationProgress: Map<string, number> = new Map(); // topic -> percentage (0-100)

  constructor() {
    this.rabbitMQ = new RabbitMQAdapter();
    this.kafka = new KafkaAdapter();
  }

  async connect(): Promise<void> {
    await Promise.all([
      this.rabbitMQ.connect(),
      this.kafka.connect(),
    ]);
  }

  setMigrationProgress(topic: string, percentage: number): void {
    this.migrationProgress.set(topic, Math.min(100, Math.max(0, percentage)));
    console.log(`Migration progress for ${topic}: ${percentage}%`);
  }

  async publish(topic: string, message: unknown, key?: string): Promise<void> {
    const progress = this.migrationProgress.get(topic) || 0;

    if (progress < 100) {
      // ยังส่งไป RabbitMQ
      await this.rabbitMQ.publish(topic, message, key);
    }

    if (progress > 0) {
      // เริ่มส่งไป Kafka
      await this.kafka.publish(topic, message, key);
    }

    console.log(`Dual write: ${topic} (${progress}% to Kafka)`);
  }

  async subscribe(topic: string, handler: (message: unknown) => Promise<void>): Promise<void> {
    const progress = this.migrationProgress.get(topic) || 0;

    if (progress < 100) {
      // ยัง consume จาก RabbitMQ
      await this.rabbitMQ.subscribe(topic, handler);
    }

    if (progress > 0) {
      // เริ่ม consume จาก Kafka
      await this.kafka.subscribe(topic, handler);
    }
  }

  async disconnect(): Promise<void> {
    await Promise.all([
      this.rabbitMQ.disconnect(),
      this.kafka.disconnect(),
    ]);
  }
}

// =========================================================
// Step 5: ใช้งาน Migration
// =========================================================

async function migrateOrderService() {
  const broker = new DualWriteMigrationAdapter();
  await broker.connect();

  // Phase 1: 0% - ยังใช้ RabbitMQ 100%
  broker.setMigrationProgress('orders', 0);

  // Phase 2: 50% - Dual write
  broker.setMigrationProgress('orders', 50);
  await broker.publish('orders', { orderId: 'test-001', amount: 100 });

  // Phase 3: 100% - ย้ายไป Kafka ทั้งหมด
  broker.setMigrationProgress('orders', 100);
  await broker.publish('orders', { orderId: 'test-002', amount: 200 });

  await broker.disconnect();
}

migrateOrderService().catch(console.error);
```

---

## 10. Multi-Broker Pattern: BrokerAbstraction Interface

```typescript
// broker/broker-abstraction.ts
// Interface หลักสำหรับ Message Broker

export interface BrokerMessage<T = unknown> {
  id: string;
  key?: string;
  data: T;
  headers?: Record<string, string>;
  timestamp: Date;
  topic: string;
}

export interface PublishOptions {
  key?: string;
  headers?: Record<string, string>;
  partitionKey?: string;
  delay?: number;
  messageId?: string;
}

export interface SubscribeOptions {
  groupId?: string;
  fromBeginning?: boolean;
  concurrency?: number;
  maxRetries?: number;
  dlqTopic?: string;
}

export interface BrokerAdapter {
  readonly name: string;
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  publish<T>(topic: string, message: T, options?: PublishOptions): Promise<string>;
  subscribe<T>(
    topic: string,
    handler: (message: BrokerMessage<T>) => Promise<void>,
    options?: SubscribeOptions
  ): Promise<void>;
  createTopic(name: string, config?: TopicConfig): Promise<void>;
  getMetrics(): Promise<BrokerMetrics>;
}

export interface TopicConfig {
  partitions?: number;
  replicationFactor?: number;
  retentionMs?: number;
  maxMessageSize?: number;
}

export interface BrokerMetrics {
  messagesProduced: number;
  messagesConsumed: number;
  messagesFailed: number;
  avgLatencyMs: number;
  pendingMessages: number;
}

// =========================================================
// Kafka Adapter Implementation
// =========================================================

import { Kafka, Producer, Consumer } from 'kafkajs';

class KafkaBrokerAdapter implements BrokerAdapter {
  readonly name = 'kafka';
  private kafka: Kafka;
  private producer: Producer | null = null;
  private consumers: Consumer[] = [];
  private metrics: BrokerMetrics = {
    messagesProduced: 0,
    messagesConsumed: 0,
    messagesFailed: 0,
    avgLatencyMs: 0,
    pendingMessages: 0,
  };

  constructor(private readonly brokers: string[]) {
    this.kafka = new Kafka({ clientId: 'broker-abstraction', brokers });
  }

  async connect(): Promise<void> {
    this.producer = this.kafka.producer({ idempotent: true });
    await this.producer.connect();
  }

  async disconnect(): Promise<void> {
    await this.producer?.disconnect();
    await Promise.all(this.consumers.map((c) => c.disconnect()));
  }

  async publish<T>(topic: string, message: T, options?: PublishOptions): Promise<string> {
    if (!this.producer) throw new Error('Not connected');

    const startTime = Date.now();
    const messageId = options?.messageId || `kafka-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

    await this.producer.send({
      topic,
      messages: [
        {
          key: options?.key ? Buffer.from(options.key) : undefined,
          value: Buffer.from(JSON.stringify(message)),
          headers: {
            'X-Message-Id': Buffer.from(messageId),
            ...Object.entries(options?.headers || {}).reduce(
              (acc, [k, v]) => ({ ...acc, [k]: Buffer.from(v) }),
              {}
            ),
          },
        },
      ],
    });

    this.metrics.messagesProduced++;
    this.metrics.avgLatencyMs =
      (this.metrics.avgLatencyMs + (Date.now() - startTime)) / 2;

    return messageId;
  }

  async subscribe<T>(
    topic: string,
    handler: (message: BrokerMessage<T>) => Promise<void>,
    options?: SubscribeOptions
  ): Promise<void> {
    const consumer = this.kafka.consumer({
      groupId: options?.groupId || 'default-group',
    });

    this.consumers.push(consumer);
    await consumer.connect();
    await consumer.subscribe({ topic, fromBeginning: options?.fromBeginning ?? false });

    await consumer.run({
      eachMessage: async ({ topic, message }) => {
        try {
          const brokerMessage: BrokerMessage<T> = {
            id: message.headers?.['X-Message-Id']?.toString() || message.offset,
            key: message.key?.toString(),
            data: JSON.parse(message.value?.toString() || '{}'),
            headers: Object.entries(message.headers || {}).reduce(
              (acc, [k, v]) => ({
                ...acc,
                [k]: Buffer.isBuffer(v) ? v.toString() : String(v),
              }),
              {}
            ),
            timestamp: new Date(parseInt(message.timestamp)),
            topic,
          };

          await handler(brokerMessage);
          this.metrics.messagesConsumed++;
        } catch (error) {
          this.metrics.messagesFailed++;
          throw error;
        }
      },
    });
  }

  async createTopic(name: string, config?: TopicConfig): Promise<void> {
    const admin = this.kafka.admin();
    await admin.connect();

    try {
      await admin.createTopics({
        topics: [
          {
            topic: name,
            numPartitions: config?.partitions || 3,
            replicationFactor: config?.replicationFactor || 1,
            configEntries: [
              {
                name: 'retention.ms',
                value: (config?.retentionMs || 7 * 24 * 60 * 60 * 1000).toString(),
              },
              {
                name: 'max.message.bytes',
                value: (config?.maxMessageSize || 1048576).toString(),
              },
            ],
          },
        ],
      });
    } finally {
      await admin.disconnect();
    }
  }

  async getMetrics(): Promise<BrokerMetrics> {
    return { ...this.metrics };
  }
}

// =========================================================
// BrokerAbstraction Manager
// =========================================================

class BrokerAbstractionManager {
  private adapters: Map<string, BrokerAdapter> = new Map();
  private defaultAdapter: string | null = null;

  registerAdapter(adapter: BrokerAdapter, isDefault = false): void {
    this.adapters.set(adapter.name, adapter);
    if (isDefault || !this.defaultAdapter) {
      this.defaultAdapter = adapter.name;
    }
    console.log(`Registered broker adapter: ${adapter.name}`);
  }

  async connectAll(): Promise<void> {
    await Promise.all(
      Array.from(this.adapters.values()).map((adapter) => adapter.connect())
    );
    console.log('All broker adapters connected');
  }

  async disconnectAll(): Promise<void> {
    await Promise.all(
      Array.from(this.adapters.values()).map((adapter) => adapter.disconnect())
    );
    console.log('All broker adapters disconnected');
  }

  getAdapter(name?: string): BrokerAdapter {
    const adapterName = name || this.defaultAdapter;
    if (!adapterName) throw new Error('No adapter registered');

    const adapter = this.adapters.get(adapterName);
    if (!adapter) throw new Error(`Adapter not found: ${adapterName}`);

    return adapter;
  }

  async publish<T>(
    topic: string,
    message: T,
    options?: PublishOptions & { broker?: string }
  ): Promise<string> {
    return this.getAdapter(options?.broker).publish(topic, message, options);
  }

  async subscribe<T>(
    topic: string,
    handler: (message: BrokerMessage<T>) => Promise<void>,
    options?: SubscribeOptions & { broker?: string }
  ): Promise<void> {
    return this.getAdapter(options?.broker).subscribe(topic, handler, options);
  }

  async getAllMetrics(): Promise<Record<string, BrokerMetrics>> {
    const metrics: Record<string, BrokerMetrics> = {};
    for (const [name, adapter] of this.adapters) {
      metrics[name] = await adapter.getMetrics();
    }
    return metrics;
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const manager = new BrokerAbstractionManager();

  // ลงทะเบียน adapters
  manager.registerAdapter(new KafkaBrokerAdapter(['localhost:9092']), true);

  await manager.connectAll();

  try {
    // Publish ไป default broker (Kafka)
    const messageId = await manager.publish('orders', {
      orderId: 'order-001',
      amount: 999.99,
    });
    console.log('Published with ID:', messageId);

    // Subscribe จาก default broker
    await manager.subscribe<{ orderId: string; amount: number }>(
      'orders',
      async (message) => {
        console.log('Received order:', message.data);
      },
      { groupId: 'order-processor' }
    );
  } finally {
    await manager.disconnectAll();
  }
}

main().catch(console.error);
```

---

## สรุป

| หัวข้อ | เนื้อหาสำคัญ |
|--------|-------------|
| การเปรียบเทียบ Brokers | Kafka เหมาะกับ throughput สูง, RabbitMQ เหมาะกับ latency ต่ำ, NATS เหมาะกับ simplicity, Pulsar เหมาะกับ multi-tenant |
| Pulsar Producer/Consumer | รองรับ partitioned topics, schema validation, และ batching |
| NATS JetStream | persistence ด้วย durable subscription, replay policy หลากหลาย |
| Message Ordering | ใช้ partition key เพื่อรับประกัน ordering ใน Kafka และ Pulsar |
| At-least-once vs Exactly-once | Idempotent producer ป้องกัน duplicate, Transactions รับประกัน exactly-once |
| Message Replay | Kafka ใช้ seek offset, NATS ใช้ DeliverPolicy |
| Dead Letter Queue | Pattern สำหรับ error handling พร้อม retry และ exponential backoff |
| Broker Monitoring | Prometheus metrics สำหรับ consumer lag, throughput, และ availability |
| Migration RabbitMQ → Kafka | Dual-write pattern สำหรับ zero-downtime migration |
| Multi-broker Abstraction | Interface pattern สำหรับ broker-agnostic code |

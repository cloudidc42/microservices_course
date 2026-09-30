# Part 13: Message Queues & Asynchronous Communication

## บทนำ

Message Queue คือ middleware ที่ช่วยให้ services สื่อสารกันแบบ **asynchronous** โดยไม่ต้องรอ response ทันที ทำให้ระบบ resilient, scalable, และ loosely coupled มากขึ้น

---

## 13.1 ทำไมต้องใช้ Message Queue?

### ปัญหาของ Synchronous Communication

```
Order Service
    ↓ POST /notify
Notification Service ← ถ้า service นี้ slow/down → Order Service ช้า/fail ด้วย!

หรือ

Order Service
    ↓ ส่ง email
    ↓ ส่ง SMS
    ↓ อัพเดท analytics
    ↓ อัพเดท loyalty points
    ↓ ส่ง webhook ให้ 3rd party
= Response time สูงมาก!
```

### Message Queue แก้ได้อย่างไร?

```
Order Service → publish "ORDER_CREATED" → Message Queue
                                              ├── Notification Service (subscribe)
                                              ├── Analytics Service (subscribe)
                                              ├── Loyalty Service (subscribe)
                                              └── Webhook Service (subscribe)

Order Service ไม่ต้องรอ services เหล่านี้ = Fast response!
และถ้า Notification Service down → message ยังอยู่ใน queue รอ
```

### Use Cases ที่เหมาะกับ Message Queue

| Use Case | ตัวอย่าง |
|----------|----------|
| **Fire and Forget** | ส่ง email, SMS, push notification |
| **Decoupling** | Order created → หลาย service ทำงาน |
| **Load Leveling** | ปรับ traffic spikes |
| **Batch Processing** | รวม events แล้วประมวลผลเป็น batch |
| **Scheduled Tasks** | Delayed jobs |
| **Event Sourcing** | บันทึก events ทุกการเปลี่ยนแปลง |

---

## 13.2 RabbitMQ

RabbitMQ ใช้ **AMQP protocol** มี concepts หลัก:
- **Producer** - ส่ง message
- **Exchange** - รับ message จาก Producer แล้ว route ไป Queues
- **Queue** - เก็บ messages
- **Consumer** - รับ messages จาก Queue

### Exchange Types

```
Direct Exchange:
Producer → Exchange → "routing_key=order.created" → Queue: order.created

Topic Exchange:
Producer → Exchange → "order.*" → Queue: all order events
                    → "*.created" → Queue: all created events

Fanout Exchange:
Producer → Exchange → Queue 1 (broadcast)
                    → Queue 2
                    → Queue 3

Headers Exchange:
Producer → Exchange → match headers → Queue
```

### RabbitMQ Setup

```javascript
// npm install amqplib

// src/messaging/rabbitmq.js
const amqp = require('amqplib');
const logger = require('../utils/logger');
const config = require('../config');

class RabbitMQ {
  constructor() {
    this.connection = null;
    this.channel = null;
    this.reconnectDelay = 5000;
    this.maxReconnectAttempts = 10;
    this.reconnectAttempts = 0;
  }

  async connect() {
    try {
      this.connection = await amqp.connect(config.rabbitmq.url);

      this.connection.on('error', (err) => {
        logger.error('RabbitMQ connection error', { error: err.message });
        this.reconnect();
      });

      this.connection.on('close', () => {
        logger.warn('RabbitMQ connection closed, reconnecting...');
        this.reconnect();
      });

      this.channel = await this.connection.createConfirmChannel();

      this.channel.on('error', (err) => {
        logger.error('RabbitMQ channel error', { error: err.message });
      });

      // Set prefetch (how many unacked messages per consumer)
      await this.channel.prefetch(10);

      logger.info('RabbitMQ connected');
      this.reconnectAttempts = 0;
    } catch (error) {
      logger.error('RabbitMQ connection failed', { error: error.message });
      this.reconnect();
    }
  }

  async reconnect() {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      logger.error('Max RabbitMQ reconnect attempts reached');
      process.exit(1);
    }

    this.reconnectAttempts++;
    const delay = this.reconnectDelay * this.reconnectAttempts;
    logger.info(`Reconnecting to RabbitMQ in ${delay}ms (attempt ${this.reconnectAttempts})`);

    await new Promise(resolve => setTimeout(resolve, delay));
    await this.connect();
  }

  // Setup Exchange and Queue
  async setupExchange(exchangeName, type = 'topic', options = {}) {
    await this.channel.assertExchange(exchangeName, type, {
      durable: true,
      ...options,
    });
  }

  async setupQueue(queueName, options = {}) {
    return this.channel.assertQueue(queueName, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': `${queueName}.dlx`, // Dead Letter Exchange
        'x-message-ttl': 24 * 60 * 60 * 1000, // 24 hours TTL
        ...options.arguments,
      },
      ...options,
    });
  }

  async bindQueue(queueName, exchangeName, routingKey) {
    await this.channel.bindQueue(queueName, exchangeName, routingKey);
  }

  // Publish message
  async publish(exchangeName, routingKey, message, options = {}) {
    const content = Buffer.from(JSON.stringify({
      ...message,
      _meta: {
        timestamp: new Date().toISOString(),
        messageId: options.messageId || `msg-${Date.now()}-${Math.random().toString(36).slice(2)}`,
      },
    }));

    return new Promise((resolve, reject) => {
      this.channel.publish(
        exchangeName,
        routingKey,
        content,
        {
          persistent: true,          // Survive broker restart
          contentType: 'application/json',
          contentEncoding: 'utf-8',
          timestamp: Math.floor(Date.now() / 1000),
          ...options,
        },
        (err) => {
          if (err) {
            logger.error('Failed to publish message', { exchangeName, routingKey, error: err.message });
            reject(err);
          } else {
            logger.debug('Message published', { exchangeName, routingKey });
            resolve();
          }
        }
      );
    });
  }

  // Consume messages
  async consume(queueName, handler, options = {}) {
    await this.channel.consume(
      queueName,
      async (msg) => {
        if (!msg) return;

        const content = JSON.parse(msg.content.toString());
        const retryCount = msg.properties.headers?.['x-retry-count'] || 0;

        try {
          logger.debug('Processing message', { queue: queueName, retryCount });
          await handler(content, msg);
          this.channel.ack(msg);
          logger.debug('Message acknowledged', { queue: queueName });
        } catch (error) {
          logger.error('Message processing failed', {
            queue: queueName,
            error: error.message,
            retryCount,
          });

          const maxRetries = options.maxRetries || 3;

          if (retryCount < maxRetries) {
            // Retry with delay
            const delay = Math.pow(2, retryCount) * 1000; // Exponential backoff
            setTimeout(() => {
              this.channel.nack(msg, false, false); // Move to DLQ
            }, delay);

            // Republish with retry count
            await this.publish(
              '',
              queueName,
              content,
              {
                headers: { 'x-retry-count': retryCount + 1 },
                expiration: delay.toString(),
              }
            );
          } else {
            // Max retries exceeded - dead letter
            logger.error('Message sent to DLQ', { queue: queueName, retryCount });
            this.channel.nack(msg, false, false);
          }
        }
      },
      { noAck: false, ...options }
    );

    logger.info(`Consumer started for queue: ${queueName}`);
  }

  async close() {
    await this.channel?.close();
    await this.connection?.close();
    logger.info('RabbitMQ connection closed');
  }

  isConnected() {
    return this.connection && !this.connection.connection.stream.destroyed;
  }
}

module.exports = new RabbitMQ();
```

### Event Bus (Abstraction Layer)

```javascript
// src/messaging/eventBus.js
const rabbitmq = require('./rabbitmq');
const logger = require('../utils/logger');

const EXCHANGE = 'microservices';

// Event types
const EVENTS = {
  // User events
  USER_REGISTERED: 'user.registered',
  USER_VERIFIED: 'user.verified',
  USER_UPDATED: 'user.updated',

  // Order events
  ORDER_CREATED: 'order.created',
  ORDER_CONFIRMED: 'order.confirmed',
  ORDER_CANCELLED: 'order.cancelled',
  ORDER_SHIPPED: 'order.shipped',
  ORDER_DELIVERED: 'order.delivered',

  // Payment events
  PAYMENT_INITIATED: 'payment.initiated',
  PAYMENT_COMPLETED: 'payment.completed',
  PAYMENT_FAILED: 'payment.failed',
  PAYMENT_REFUNDED: 'payment.refunded',

  // Product events
  PRODUCT_CREATED: 'product.created',
  PRODUCT_UPDATED: 'product.updated',
  PRODUCT_OUT_OF_STOCK: 'product.out_of_stock',
  PRODUCT_RESTOCKED: 'product.restocked',

  // Notification events
  NOTIFICATION_EMAIL: 'notification.email',
  NOTIFICATION_SMS: 'notification.sms',
  NOTIFICATION_PUSH: 'notification.push',
};

class EventBus {
  async initialize() {
    await rabbitmq.connect();
    await rabbitmq.setupExchange(EXCHANGE, 'topic');
    logger.info('EventBus initialized');
  }

  async publish(eventType, payload) {
    try {
      await rabbitmq.publish(EXCHANGE, eventType, {
        type: eventType,
        payload,
        publishedAt: new Date().toISOString(),
      });
      logger.info('Event published', { eventType });
    } catch (error) {
      logger.error('Failed to publish event', { eventType, error: error.message });
      throw error;
    }
  }

  async subscribe(serviceName, eventPattern, handler, options = {}) {
    const queueName = `${serviceName}.${eventPattern.replace(/\./g, '_').replace(/\*/g, 'wildcard')}`;

    await rabbitmq.setupQueue(queueName, options);
    await rabbitmq.bindQueue(queueName, EXCHANGE, eventPattern);
    await rabbitmq.consume(queueName, async (message) => {
      logger.info('Event received', { eventType: message.type, service: serviceName });
      await handler(message.payload, message);
    }, options);
  }
}

module.exports = { eventBus: new EventBus(), EVENTS };
```

---

## 13.3 Order Service - Publishing Events

```javascript
// order-service/src/services/orderService.js
const { eventBus, EVENTS } = require('../messaging/eventBus');
const orderRepository = require('../repositories/orderRepository');
const logger = require('../utils/logger');

class OrderService {
  async createOrder(orderData) {
    const order = await orderRepository.create(orderData);

    // Publish event (fire and forget)
    await eventBus.publish(EVENTS.ORDER_CREATED, {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      total: order.total,
      shippingAddress: order.shippingAddress,
      createdAt: order.createdAt,
    });

    return order;
  }

  async confirmOrder(orderId) {
    const order = await orderRepository.updateStatus(orderId, 'confirmed');

    await eventBus.publish(EVENTS.ORDER_CONFIRMED, {
      orderId: order.id,
      userId: order.userId,
      confirmedAt: new Date().toISOString(),
    });

    return order;
  }

  async cancelOrder(orderId, userId, reason) {
    const order = await orderRepository.findById(orderId);

    if (!order) throw new Error('Order not found');
    if (order.userId !== userId) throw new Error('Access denied');
    if (!['pending', 'confirmed'].includes(order.status)) {
      throw new Error(`Cannot cancel order with status: ${order.status}`);
    }

    const updated = await orderRepository.updateStatus(orderId, 'cancelled', { cancellationReason: reason });

    await eventBus.publish(EVENTS.ORDER_CANCELLED, {
      orderId: order.id,
      userId: order.userId,
      reason,
      cancelledAt: new Date().toISOString(),
    });

    return updated;
  }

  async shipOrder(orderId, trackingInfo) {
    const order = await orderRepository.updateStatus(orderId, 'shipped', { trackingInfo });

    await eventBus.publish(EVENTS.ORDER_SHIPPED, {
      orderId: order.id,
      userId: order.userId,
      trackingInfo,
      shippedAt: new Date().toISOString(),
    });

    return order;
  }
}

module.exports = new OrderService();
```

---

## 13.4 Notification Service - Consuming Events

```javascript
// notification-service/src/messaging/subscribers.js
const { eventBus, EVENTS } = require('../messaging/eventBus');
const emailService = require('../services/emailService');
const smsService = require('../services/smsService');
const pushService = require('../services/pushService');
const logger = require('../utils/logger');

async function setupSubscribers() {
  // Subscribe to order events
  await eventBus.subscribe(
    'notification-service',
    'order.*',  // All order events
    async (payload, message) => {
      const eventType = message.type;
      logger.info('Processing order event', { eventType, orderId: payload.orderId });

      switch (eventType) {
        case EVENTS.ORDER_CREATED:
          await handleOrderCreated(payload);
          break;
        case EVENTS.ORDER_CONFIRMED:
          await handleOrderConfirmed(payload);
          break;
        case EVENTS.ORDER_CANCELLED:
          await handleOrderCancelled(payload);
          break;
        case EVENTS.ORDER_SHIPPED:
          await handleOrderShipped(payload);
          break;
        case EVENTS.ORDER_DELIVERED:
          await handleOrderDelivered(payload);
          break;
      }
    },
    { maxRetries: 3 }
  );

  // Subscribe to user events
  await eventBus.subscribe(
    'notification-service',
    'user.*',
    async (payload, message) => {
      switch (message.type) {
        case EVENTS.USER_REGISTERED:
          await emailService.sendWelcomeEmail(payload.email, payload.firstName);
          break;
        case EVENTS.USER_VERIFIED:
          await emailService.sendVerifiedEmail(payload.email, payload.firstName);
          break;
      }
    }
  );

  // Subscribe to payment events
  await eventBus.subscribe(
    'notification-service',
    'payment.*',
    async (payload, message) => {
      switch (message.type) {
        case EVENTS.PAYMENT_COMPLETED:
          await handlePaymentCompleted(payload);
          break;
        case EVENTS.PAYMENT_FAILED:
          await handlePaymentFailed(payload);
          break;
        case EVENTS.PAYMENT_REFUNDED:
          await handlePaymentRefunded(payload);
          break;
      }
    }
  );

  logger.info('All notification subscribers set up');
}

async function handleOrderCreated(payload) {
  const { orderId, userId, total, items } = payload;

  // Get user details from User Service (or from event payload if included)
  const userEmail = payload.userEmail || await getUserEmail(userId);

  await Promise.allSettled([
    emailService.sendOrderConfirmation({
      to: userEmail,
      orderId,
      total,
      items,
    }),
    pushService.sendNotification(userId, {
      title: 'คำสั่งซื้อสำเร็จ',
      body: `คำสั่งซื้อ #${orderId} ได้รับการยืนยันแล้ว`,
      data: { orderId, type: 'order_created' },
    }),
  ]);
}

async function handleOrderShipped(payload) {
  const { orderId, userId, trackingInfo } = payload;
  const userEmail = await getUserEmail(userId);

  await Promise.allSettled([
    emailService.sendShippingNotification({
      to: userEmail,
      orderId,
      trackingNumber: trackingInfo.trackingNumber,
      carrier: trackingInfo.carrier,
      estimatedDelivery: trackingInfo.estimatedDelivery,
    }),
    smsService.send(payload.userPhone, `คำสั่งซื้อ #${orderId} กำลังจัดส่ง หมายเลขพัสดุ: ${trackingInfo.trackingNumber}`),
    pushService.sendNotification(userId, {
      title: 'พัสดุถูกจัดส่งแล้ว',
      body: `คำสั่งซื้อ #${orderId} กำลังส่งไปหาคุณ`,
    }),
  ]);
}

async function handlePaymentCompleted(payload) {
  const { orderId, userId, amount, paymentMethod } = payload;
  const userEmail = await getUserEmail(userId);

  await emailService.sendPaymentReceipt({
    to: userEmail,
    orderId,
    amount,
    paymentMethod,
    transactionId: payload.transactionId,
  });
}

module.exports = { setupSubscribers };
```

---

## 13.5 Inventory Service - Stock Management

```javascript
// inventory-service/src/messaging/subscribers.js
const { eventBus, EVENTS } = require('../messaging/eventBus');
const inventoryRepository = require('../repositories/inventoryRepository');
const logger = require('../utils/logger');

async function setupSubscribers() {
  // Decrease stock when order is confirmed
  await eventBus.subscribe(
    'inventory-service',
    EVENTS.ORDER_CONFIRMED,
    async (payload) => {
      logger.info('Decreasing stock for confirmed order', { orderId: payload.orderId });

      for (const item of payload.items) {
        try {
          await inventoryRepository.decreaseStock(item.productId, item.quantity);
          logger.info('Stock decreased', { productId: item.productId, quantity: item.quantity });
        } catch (error) {
          logger.error('Failed to decrease stock', {
            productId: item.productId,
            error: error.message,
          });
          // Publish compensation event
          await eventBus.publish('inventory.stock_decrease_failed', {
            orderId: payload.orderId,
            productId: item.productId,
            error: error.message,
          });
        }
      }
    }
  );

  // Restore stock when order is cancelled
  await eventBus.subscribe(
    'inventory-service',
    EVENTS.ORDER_CANCELLED,
    async (payload) => {
      logger.info('Restoring stock for cancelled order', { orderId: payload.orderId });

      if (payload.items) {
        for (const item of payload.items) {
          await inventoryRepository.increaseStock(item.productId, item.quantity);
        }
      }
    }
  );

  // Check low stock and publish notification
  await eventBus.subscribe(
    'inventory-service',
    EVENTS.ORDER_CONFIRMED,
    async (payload) => {
      for (const item of payload.items) {
        const stock = await inventoryRepository.getStock(item.productId);
        if (stock.quantity <= stock.lowStockThreshold) {
          await eventBus.publish(EVENTS.PRODUCT_OUT_OF_STOCK, {
            productId: item.productId,
            currentStock: stock.quantity,
            threshold: stock.lowStockThreshold,
          });
        }
      }
    }
  );
}

module.exports = { setupSubscribers };
```

---

## 13.6 Dead Letter Queue (DLQ) Pattern

```javascript
// src/messaging/deadLetterHandler.js
const amqp = require('amqplib');
const config = require('../config');
const logger = require('../utils/logger');

class DeadLetterHandler {
  async setupDLQ(originalQueueName) {
    const channel = await this.getChannel();

    const dlxName = `${originalQueueName}.dlx`;
    const dlqName = `${originalQueueName}.dlq`;

    // Setup Dead Letter Exchange
    await channel.assertExchange(dlxName, 'direct', { durable: true });

    // Setup Dead Letter Queue
    await channel.assertQueue(dlqName, {
      durable: true,
      arguments: {
        // Messages in DLQ expire after 7 days for review
        'x-message-ttl': 7 * 24 * 60 * 60 * 1000,
      },
    });

    await channel.bindQueue(dlqName, dlxName, originalQueueName);
    logger.info(`DLQ setup for ${originalQueueName}`);
  }

  // Process failed messages manually (for debugging/reprocessing)
  async processDLQ(dlqName, handler) {
    const channel = await this.getChannel();

    await channel.consume(dlqName, async (msg) => {
      if (!msg) return;

      const content = JSON.parse(msg.content.toString());
      const deathInfo = msg.properties.headers?.['x-death'];

      logger.info('Processing DLQ message', {
        queue: dlqName,
        deathCount: deathInfo?.[0]?.count,
        originalQueue: deathInfo?.[0]?.queue,
        reason: deathInfo?.[0]?.reason,
      });

      try {
        await handler(content, { deathInfo });
        channel.ack(msg);
      } catch (error) {
        logger.error('DLQ message processing failed', { error: error.message });
        channel.nack(msg, false, false); // Discard permanently
      }
    });
  }

  // Replay DLQ messages back to original queue
  async replayDLQ(dlqName, targetQueue, limit = 100) {
    const channel = await this.getChannel();
    let count = 0;

    while (count < limit) {
      const msg = await channel.get(dlqName, { noAck: false });
      if (!msg) break;

      try {
        // Requeue to original queue
        await channel.sendToQueue(
          targetQueue,
          msg.content,
          {
            persistent: true,
            headers: {
              ...msg.properties.headers,
              'x-replayed': true,
              'x-replayed-at': new Date().toISOString(),
            },
          }
        );
        channel.ack(msg);
        count++;
      } catch (error) {
        channel.nack(msg, false, true); // Requeue to DLQ
        break;
      }
    }

    logger.info(`Replayed ${count} messages from DLQ: ${dlqName}`);
    return count;
  }
}

module.exports = new DeadLetterHandler();
```

---

## 13.7 Priority Queue

```javascript
// src/messaging/priorityQueue.js

// Setup priority queue (RabbitMQ supports up to 255 priority levels)
async function setupPriorityQueue(channel, queueName, maxPriority = 10) {
  return channel.assertQueue(queueName, {
    durable: true,
    arguments: {
      'x-max-priority': maxPriority,
    },
  });
}

// Publish with priority
async function publishWithPriority(channel, queueName, message, priority = 0) {
  channel.sendToQueue(
    queueName,
    Buffer.from(JSON.stringify(message)),
    {
      persistent: true,
      priority, // 0 = lowest, maxPriority = highest
    }
  );
}

// Example usage
// Priority levels:
// 10: Critical (payment failure, security alerts)
// 7: High (order confirmation)
// 5: Normal (notifications)
// 2: Low (analytics, reports)

async function sendNotification(userId, notification, isUrgent = false) {
  await publishWithPriority(
    channel,
    'notifications',
    { userId, ...notification },
    isUrgent ? 10 : 5
  );
}
```

---

## 13.8 Request-Reply Pattern over Message Queue

```javascript
// src/messaging/rpcClient.js
const amqp = require('amqplib');
const { v4: uuidv4 } = require('uuid');

class RPCClient {
  constructor(channel) {
    this.channel = channel;
    this.correlationMap = new Map();
    this.replyQueue = null;
  }

  async initialize() {
    // Create exclusive reply queue
    const { queue } = await this.channel.assertQueue('', {
      exclusive: true,
      autoDelete: true,
    });
    this.replyQueue = queue;

    // Listen for replies
    await this.channel.consume(this.replyQueue, (msg) => {
      if (!msg) return;

      const correlationId = msg.properties.correlationId;
      const resolve = this.correlationMap.get(correlationId);

      if (resolve) {
        resolve(JSON.parse(msg.content.toString()));
        this.correlationMap.delete(correlationId);
        this.channel.ack(msg);
      }
    }, { noAck: false });
  }

  call(queueName, message, timeout = 5000) {
    return new Promise((resolve, reject) => {
      const correlationId = uuidv4();

      const timer = setTimeout(() => {
        this.correlationMap.delete(correlationId);
        reject(new Error(`RPC timeout for ${queueName}`));
      }, timeout);

      this.correlationMap.set(correlationId, (response) => {
        clearTimeout(timer);
        resolve(response);
      });

      this.channel.sendToQueue(
        queueName,
        Buffer.from(JSON.stringify(message)),
        {
          replyTo: this.replyQueue,
          correlationId,
          persistent: false, // RPC usually doesn't need persistence
        }
      );
    });
  }
}

// RPC Server
class RPCServer {
  constructor(channel) {
    this.channel = channel;
  }

  async register(queueName, handler) {
    await this.channel.assertQueue(queueName, { durable: false });
    await this.channel.prefetch(1);

    await this.channel.consume(queueName, async (msg) => {
      if (!msg) return;

      const request = JSON.parse(msg.content.toString());
      let response;

      try {
        response = { success: true, data: await handler(request) };
      } catch (error) {
        response = { success: false, error: error.message };
      }

      this.channel.sendToQueue(
        msg.properties.replyTo,
        Buffer.from(JSON.stringify(response)),
        { correlationId: msg.properties.correlationId }
      );

      this.channel.ack(msg);
    });
  }
}

module.exports = { RPCClient, RPCServer };
```

---

## 13.9 Kafka Introduction

Apache Kafka ต่างจาก RabbitMQ ในแง่:
- **Log-based** - messages ถูกเก็บถาวร (ไม่ถูกลบหลัง consume)
- **Consumer Groups** - scaling consumers แบบ partition-based
- **Replay** - สามารถ replay messages เก่าได้
- **High Throughput** - เหมาะกับ millions of events/second

```javascript
// npm install kafkajs

// src/messaging/kafka.js
const { Kafka, Partitioners } = require('kafkajs');
const config = require('../config');
const logger = require('../utils/logger');

const kafka = new Kafka({
  clientId: config.kafka.clientId,
  brokers: config.kafka.brokers,
  ssl: config.kafka.ssl,
  sasl: config.kafka.sasl,
  retry: {
    initialRetryTime: 100,
    retries: 8,
  },
});

// Producer
class KafkaProducer {
  constructor() {
    this.producer = kafka.producer({
      createPartitioner: Partitioners.LegacyPartitioner,
      allowAutoTopicCreation: true,
    });
  }

  async connect() {
    await this.producer.connect();
    logger.info('Kafka producer connected');
  }

  async publish(topic, messages) {
    if (!Array.isArray(messages)) {
      messages = [messages];
    }

    const kafkaMessages = messages.map(msg => ({
      key: msg.key || null,
      value: JSON.stringify(msg.value || msg),
      headers: {
        'content-type': 'application/json',
        timestamp: Date.now().toString(),
        ...msg.headers,
      },
    }));

    await this.producer.send({ topic, messages: kafkaMessages });
    logger.debug('Message published to Kafka', { topic, count: messages.length });
  }

  async publishBatch(topicMessages) {
    await this.producer.sendBatch({ topicMessages });
  }

  async disconnect() {
    await this.producer.disconnect();
  }
}

// Consumer
class KafkaConsumer {
  constructor(groupId) {
    this.consumer = kafka.consumer({ groupId });
  }

  async connect() {
    await this.consumer.connect();
    logger.info('Kafka consumer connected');
  }

  async subscribe(topics) {
    if (!Array.isArray(topics)) topics = [topics];

    for (const topic of topics) {
      await this.consumer.subscribe({
        topic,
        fromBeginning: false,
      });
    }
  }

  async run(handler) {
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const value = JSON.parse(message.value.toString());
        const key = message.key?.toString();

        logger.debug('Processing Kafka message', { topic, partition, key });

        try {
          await handler({ topic, partition, key, value, message });
        } catch (error) {
          logger.error('Kafka message processing failed', {
            topic, partition, key, error: error.message,
          });
          // In Kafka, we don't nack - we can implement retry topic or DLQ pattern
          await this.publishToDeadLetter(topic, { value, key, error: error.message });
        }
      },
    });
  }

  async publishToDeadLetter(originalTopic, data) {
    const producer = new KafkaProducer();
    await producer.connect();
    await producer.publish(`${originalTopic}.dlq`, data);
    await producer.disconnect();
  }

  async disconnect() {
    await this.consumer.disconnect();
  }
}

// Admin (for topic management)
class KafkaAdmin {
  constructor() {
    this.admin = kafka.admin();
  }

  async connect() {
    await this.admin.connect();
  }

  async createTopics(topics) {
    await this.admin.createTopics({
      topics: topics.map(t => ({
        topic: t.name,
        numPartitions: t.partitions || 3,
        replicationFactor: t.replication || 1,
        configEntries: [
          { name: 'retention.ms', value: String(t.retentionMs || 7 * 24 * 60 * 60 * 1000) },
          { name: 'cleanup.policy', value: t.cleanup || 'delete' },
        ],
      })),
    });
  }

  async disconnect() {
    await this.admin.disconnect();
  }
}

module.exports = { KafkaProducer, KafkaConsumer, KafkaAdmin };
```

### Kafka Topic Setup

```javascript
// scripts/setup-kafka-topics.js
const { KafkaAdmin } = require('../src/messaging/kafka');

async function setupTopics() {
  const admin = new KafkaAdmin();
  await admin.connect();

  await admin.createTopics([
    { name: 'user-events', partitions: 3, replication: 1, retentionMs: 7 * 24 * 60 * 60 * 1000 },
    { name: 'order-events', partitions: 6, replication: 1, retentionMs: 30 * 24 * 60 * 60 * 1000 },
    { name: 'payment-events', partitions: 3, replication: 1, retentionMs: 90 * 24 * 60 * 60 * 1000 },
    { name: 'product-events', partitions: 3, replication: 1, retentionMs: 7 * 24 * 60 * 60 * 1000 },
    { name: 'notification-events', partitions: 3, replication: 1, retentionMs: 1 * 24 * 60 * 60 * 1000 },
    // Dead letter queues
    { name: 'order-events.dlq', partitions: 1, replication: 1, retentionMs: 30 * 24 * 60 * 60 * 1000 },
  ]);

  console.log('Kafka topics created');
  await admin.disconnect();
}

setupTopics().catch(console.error);
```

---

## 13.10 Docker Compose สำหรับ RabbitMQ & Kafka

```yaml
# docker-compose.messaging.yml
version: '3.8'

services:
  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    container_name: rabbitmq
    hostname: rabbitmq
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: ${RABBITMQ_PASSWORD:-admin123}
      RABBITMQ_DEFAULT_VHOST: /
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
      - ./rabbitmq/definitions.json:/etc/rabbitmq/definitions.json:ro
      - ./rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf:ro
    networks:
      - messaging
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 10s
      timeout: 5s
      retries: 10

  # Kafka + Zookeeper
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log
    networks:
      - messaging

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
    volumes:
      - kafka-data:/var/lib/kafka/data
    networks:
      - messaging
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 10

  # Kafka UI (Optional - for development)
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    networks:
      - messaging

  # Redis for pub/sub (lightweight alternative)
  redis-pubsub:
    image: redis:7-alpine
    container_name: redis-pubsub
    command: redis-server --requirepass ${REDIS_PASSWORD:-redis123}
    ports:
      - "6380:6379"
    networks:
      - messaging
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD:-redis123}", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

networks:
  messaging:
    driver: bridge

volumes:
  rabbitmq-data:
  zookeeper-data:
  zookeeper-logs:
  kafka-data:
```

---

## 13.11 Message Idempotency

```javascript
// src/middleware/idempotent.js
// ป้องกัน message ถูก process มากกว่าครั้ง

const Redis = require('ioredis');
const config = require('../config');

const redis = new Redis(config.redis);

class IdempotencyHandler {
  constructor(ttlSeconds = 24 * 60 * 60) { // 24 hours
    this.ttl = ttlSeconds;
  }

  async isProcessed(messageId) {
    return redis.exists(`processed:${messageId}`);
  }

  async markProcessed(messageId) {
    await redis.setex(`processed:${messageId}`, this.ttl, '1');
  }

  // Wrapper for message handlers
  wrap(handler) {
    return async (message, rawMsg) => {
      const messageId = message._meta?.messageId || rawMsg?.properties?.messageId;

      if (!messageId) {
        // No message ID - can't ensure idempotency, process anyway
        return handler(message, rawMsg);
      }

      const alreadyProcessed = await this.isProcessed(messageId);
      if (alreadyProcessed) {
        logger.warn('Duplicate message detected, skipping', { messageId });
        return; // Skip
      }

      await handler(message, rawMsg);
      await this.markProcessed(messageId);
    };
  }
}

module.exports = new IdempotencyHandler();

// Usage:
// await eventBus.subscribe(
//   'service-name',
//   'order.*',
//   idempotencyHandler.wrap(async (payload) => {
//     // Process message
//   })
// );
```

---

## 13.12 Testing with Message Queues

```javascript
// tests/messaging.test.js
const { eventBus, EVENTS } = require('../src/messaging/eventBus');
const orderService = require('../src/services/orderService');

// Mock the event bus for unit tests
jest.mock('../src/messaging/eventBus');

describe('Order Service Events', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    eventBus.publish.mockResolvedValue();
  });

  describe('createOrder', () => {
    it('should publish ORDER_CREATED event after creating order', async () => {
      const orderData = {
        userId: 'user-1',
        items: [{ productId: 'prod-1', quantity: 2, price: 100 }],
        total: 200,
      };

      const order = await orderService.createOrder(orderData);

      expect(eventBus.publish).toHaveBeenCalledWith(
        EVENTS.ORDER_CREATED,
        expect.objectContaining({
          orderId: order.id,
          userId: 'user-1',
          total: 200,
        })
      );
    });
  });

  describe('cancelOrder', () => {
    it('should publish ORDER_CANCELLED event', async () => {
      const orderId = 'order-1';
      const userId = 'user-1';

      // Setup: create order first
      await setupOrder(orderId, userId, 'confirmed');

      await orderService.cancelOrder(orderId, userId, 'Changed mind');

      expect(eventBus.publish).toHaveBeenCalledWith(
        EVENTS.ORDER_CANCELLED,
        expect.objectContaining({
          orderId,
          userId,
          reason: 'Changed mind',
        })
      );
    });
  });
});

// Integration test with actual RabbitMQ (requires running RabbitMQ)
describe('Event Integration Tests', () => {
  if (process.env.SKIP_INTEGRATION) return;

  let receivedMessages = [];

  beforeAll(async () => {
    await eventBus.initialize();

    // Setup test subscriber
    await eventBus.subscribe('test-service', 'order.*', async (payload, message) => {
      receivedMessages.push({ type: message.type, payload });
    });
  });

  afterAll(async () => {
    // Cleanup
  });

  beforeEach(() => {
    receivedMessages = [];
  });

  it('should receive ORDER_CREATED event', async () => {
    await eventBus.publish(EVENTS.ORDER_CREATED, {
      orderId: 'test-order-1',
      userId: 'test-user-1',
    });

    // Wait for message
    await new Promise(resolve => setTimeout(resolve, 500));

    expect(receivedMessages).toHaveLength(1);
    expect(receivedMessages[0].type).toBe(EVENTS.ORDER_CREATED);
    expect(receivedMessages[0].payload.orderId).toBe('test-order-1');
  });
});
```

---

## สรุป Part 13

ในบทนี้เราได้เรียนรู้:
- **Message Queue ทำไม** - decoupling, reliability, scalability
- **RabbitMQ** - AMQP, Exchange types, Queues, Publisher/Consumer
- **Event Bus** - abstraction layer สำหรับ event-driven communication
- **Dead Letter Queue** - จัดการ failed messages
- **Priority Queue** - process urgent messages ก่อน
- **Request-Reply over Queue** - RPC pattern
- **Kafka** - log-based messaging, consumer groups, high throughput
- **Idempotency** - ป้องกัน duplicate processing
- **Testing** - mock vs integration tests

**Next:** Part 14 - Error Handling & Resilience Patterns

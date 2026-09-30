# Part 86: Message Broker Comparison

## บทนำ

การเลือก Message Broker ที่เหมาะสมส่งผลต่อทั้ง performance, reliability, และ operational complexity ของระบบ บทนี้เปรียบเทียบ RabbitMQ, Apache Kafka, AWS SQS/SNS, Google Pub/Sub, และ Azure Service Bus พร้อมแนวทางการเลือกที่เหมาะกับ use case

---

## Comparison Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                     MESSAGE BROKER COMPARISON MATRIX                         │
├────────────────┬────────────┬────────────┬────────────┬──────────┬───────────┤
│                │  RabbitMQ  │   Kafka    │  AWS SQS   │ GCP Pub/ │ Azure SB  │
│                │            │            │   /SNS     │   Sub    │           │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Throughput     │ 20K msg/s  │ 1M+ msg/s  │ 3K/10K     │ 10M/s   │ 1M msg/s  │
│                │            │            │ (SQS/SNS)  │          │           │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Message        │ No replay  │ Yes        │ No replay  │ Yes      │ No replay │
│ Replay         │            │ (retention)│            │          │           │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Delivery       │ At-least-  │ At-least-  │ At-least-  │ At-least-│ At-least- │
│ Guarantee      │ once       │ once*      │ once       │ once     │ once      │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Ordering       │ Per-queue  │ Per-       │ Per-       │ Per-key  │ Per-      │
│                │            │ partition  │ message    │          │ session   │
│                │            │            │ group      │          │           │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Routing        │ Complex    │ Topic-     │ Simple     │ Topic-   │ Topic-    │
│                │ (exchanges)│ based      │            │ based    │ based     │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Cost           │ Self-hosted│ Self-host/ │ Pay-per-   │ Pay-per- │ Pay-per-  │
│                │            │ Confluent  │ message    │ message  │ message   │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Ops Complexity │ Medium     │ High       │ Low        │ Low      │ Low       │
├────────────────┼────────────┼────────────┼────────────┼──────────┼───────────┤
│ Best for       │ Task       │ Event      │ Simple     │ GCP      │ Azure     │
│                │ queues,    │ streaming, │ queuing,   │ workloads│ workloads │
│                │ routing    │ log proc.  │ serverless │          │           │
└────────────────┴────────────┴────────────┴────────────┴──────────┴───────────┘
```

---

## RabbitMQ: เหมาะสำหรับ Task Queues

### เมื่อไหร่ควรใช้ RabbitMQ?
- Task queues ที่มี complex routing logic
- Microservices ที่ต้องการ flexible message routing
- Self-hosted infrastructure
- ไม่ต้องการ message replay

```typescript
// services/order/src/messaging/rabbitmq.module.ts

import { Module } from '@nestjs/common';
import { RabbitMQModule } from '@golevelup/nestjs-rabbitmq';
import { ConfigService } from '@nestjs/config';

@Module({
  imports: [
    RabbitMQModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        exchanges: [
          {
            name: 'order.events',
            type: 'topic',
            options: { durable: true },
          },
          {
            name: 'order.commands',
            type: 'direct',
            options: { durable: true },
          },
          // Dead Letter Exchange
          {
            name: 'order.dlx',
            type: 'topic',
            options: { durable: true },
          },
        ],
        uri: config.get<string>('RABBITMQ_URI'),
        connectionInitOptions: { wait: true },
        channels: {
          'channel-1': {
            prefetchCount: 10,
            default: true,
          },
        },
      }),
    }),
  ],
  exports: [RabbitMQModule],
})
export class MessagingModule {}
```

```typescript
// services/order/src/messaging/order-publisher.service.ts

import { Injectable } from '@nestjs/common';
import { AmqpConnection } from '@golevelup/nestjs-rabbitmq';

@Injectable()
export class OrderPublisher {
  constructor(private readonly amqp: AmqpConnection) {}

  async publishOrderCreated(order: Order): Promise<void> {
    await this.amqp.publish(
      'order.events',     // exchange
      'order.created',    // routing key
      {
        orderId: order.id,
        userId: order.userId,
        totalAmount: order.totalAmount,
        items: order.items,
        createdAt: order.createdAt,
      },
      {
        persistent: true,  // survive broker restart
        messageId: `order-created-${order.id}`,
        timestamp: Date.now(),
        headers: {
          'x-correlation-id': order.id,
          'x-source-service': 'order-service',
        },
      }
    );
  }
}
```

```typescript
// services/notification/src/listeners/order.listener.ts

import { RabbitSubscribe, Nack } from '@golevelup/nestjs-rabbitmq';
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class OrderEventListener {
  private readonly logger = new Logger(OrderEventListener.name);

  @RabbitSubscribe({
    exchange: 'order.events',
    routingKey: 'order.created',
    queue: 'notification-service.order-created',
    queueOptions: {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': 'order.dlx',
        'x-dead-letter-routing-key': 'notification.order.created.failed',
        'x-message-ttl': 86400000, // 24 hours
        'x-max-retries': 3,
      },
    },
  })
  async handleOrderCreated(
    payload: OrderCreatedEvent,
    amqpMsg: Message,
  ): Promise<void | Nack> {
    try {
      await this.notificationService.sendOrderConfirmation(payload);
      // Message จะถูก ack อัตโนมัติ
    } catch (error) {
      const retryCount = (amqpMsg.properties.headers?.['x-retry-count'] || 0) as number;
      
      if (retryCount < 3) {
        this.logger.warn(
          `Retrying order notification (attempt ${retryCount + 1}/3)`,
          { orderId: payload.orderId }
        );
        // Nack กับ requeue = true จะ retry
        return new Nack(true);
      }
      
      this.logger.error(
        'Max retries reached, sending to DLQ',
        { orderId: payload.orderId, error: error.message }
      );
      // Nack กับ requeue = false จะส่งไป DLQ
      return new Nack(false);
    }
  }
}
```

---

## Apache Kafka: เหมาะสำหรับ Event Streaming

### เมื่อไหร่ควรใช้ Kafka?
- Event streaming / event log
- High throughput (> 100K messages/second)
- ต้องการ message replay
- Event-sourcing pattern
- Analytics pipeline

```typescript
// services/order/src/messaging/kafka.module.ts

import { Module } from '@nestjs/common';
import { ClientsModule, Transport } from '@nestjs/microservices';
import { ConfigService } from '@nestjs/config';

@Module({
  imports: [
    ClientsModule.registerAsync([
      {
        name: 'KAFKA_CLIENT',
        inject: [ConfigService],
        useFactory: (config: ConfigService) => ({
          transport: Transport.KAFKA,
          options: {
            client: {
              clientId: 'order-service',
              brokers: config.get<string[]>('KAFKA_BROKERS', ['kafka:9092']),
              ssl: config.get<boolean>('KAFKA_SSL', false),
              sasl: config.get('KAFKA_SASL_ENABLED')
                ? {
                    mechanism: 'plain',
                    username: config.get('KAFKA_USERNAME'),
                    password: config.get('KAFKA_PASSWORD'),
                  }
                : undefined,
            },
            producer: {
              allowAutoTopicCreation: false,
              transactionalId: 'order-service-producer',
              idempotent: true,         // Exactly-once semantics
              maxInFlightRequests: 5,
            },
            consumer: {
              groupId: 'order-service-consumer-group',
              sessionTimeout: 30000,
              heartbeatInterval: 10000,
              maxBytesPerPartition: 1048576, // 1MB
            },
          },
        }),
      },
    ]),
  ],
  exports: [ClientsModule],
})
export class KafkaModule {}
```

```typescript
// services/order/src/messaging/kafka-producer.service.ts

import { Injectable, Inject, Logger, OnModuleInit } from '@nestjs/common';
import { ClientKafka, KafkaHeaders } from '@nestjs/microservices';
import { Message } from 'kafkajs';

@Injectable()
export class KafkaOrderProducer implements OnModuleInit {
  private readonly logger = new Logger(KafkaOrderProducer.name);

  constructor(
    @Inject('KAFKA_CLIENT')
    private readonly kafkaClient: ClientKafka,
  ) {}

  async onModuleInit(): Promise<void> {
    await this.kafkaClient.connect();
  }

  async publishOrderEvent(
    eventType: string,
    orderId: string,
    payload: unknown,
  ): Promise<void> {
    const message: Message = {
      key: orderId,           // orderId เป็น partition key - ordering per order
      value: JSON.stringify({
        eventType,
        timestamp: new Date().toISOString(),
        version: '1.0',
        data: payload,
      }),
      headers: {
        [KafkaHeaders.CORRELATION_ID]: orderId,
        'source-service': 'order-service',
        'event-type': eventType,
        'schema-version': '1.0',
      },
    };

    await this.kafkaClient.emit(
      `order.events.${eventType.toLowerCase()}`,
      message
    ).toPromise();

    this.logger.debug(`Published ${eventType} for order ${orderId}`);
  }
}
```

```typescript
// services/analytics/src/consumers/order-events.consumer.ts
// Kafka Consumer ใน NestJS

import { Controller } from '@nestjs/common';
import {
  MessagePattern,
  Payload,
  Ctx,
  KafkaContext,
} from '@nestjs/microservices';

@Controller()
export class OrderEventsConsumer {
  @MessagePattern('order.events.order.created')
  async handleOrderCreated(
    @Payload() message: KafkaMessage,
    @Ctx() context: KafkaContext,
  ): Promise<void> {
    const originalMessage = context.getMessage();
    const partition = context.getPartition();
    const { offset } = originalMessage;
    const topic = context.getTopic();
    
    try {
      const event = JSON.parse(originalMessage.value.toString());
      await this.analyticsService.processOrderCreated(event.data);
      
      // Manual commit (ถ้าตั้งค่า autoCommit: false)
      const heartbeat = context.getHeartbeat();
      await heartbeat();
      
    } catch (error) {
      this.logger.error(
        `Failed to process order created event`,
        {
          topic,
          partition,
          offset: offset.toString(),
          error: error.message,
        }
      );
      
      // ส่ง DLQ
      await this.dlqProducer.send({
        topic: 'order.events.dlq',
        messages: [originalMessage],
      });
    }
  }
}
```

### Kafka Topic Configuration

```yaml
# kafka/topics/order-events.yaml
# ใช้ kafka-topics.sh หรือ Strimzi Operator

# Topics configuration
topics:
  - name: order.events.order.created
    partitions: 12       # จำนวน partitions = parallelism
    replicationFactor: 3 # HA
    config:
      retention.ms: "604800000"        # 7 days
      retention.bytes: "10737418240"   # 10 GB per partition
      min.insync.replicas: "2"
      compression.type: "lz4"
      cleanup.policy: "delete"
      
  - name: order.events.dlq
    partitions: 3
    replicationFactor: 3
    config:
      retention.ms: "2592000000"  # 30 days for DLQ
      min.insync.replicas: "2"
```

---

## AWS SQS/SNS: เหมาะสำหรับ AWS Ecosystem

### เมื่อไหร่ควรใช้ SQS/SNS?
- AWS-native workloads
- Serverless architecture (Lambda triggers)
- Simple queuing โดยไม่ต้อง manage infrastructure
- Fan-out patterns (SNS → multiple SQS queues)

```typescript
// services/order/src/messaging/sqs.service.ts

import { Injectable, Logger } from '@nestjs/common';
import {
  SQSClient,
  SendMessageCommand,
  ReceiveMessageCommand,
  DeleteMessageCommand,
  Message,
} from '@aws-sdk/client-sqs';
import {
  SNSClient,
  PublishCommand,
} from '@aws-sdk/client-sns';

@Injectable()
export class SQSSNSService {
  private readonly logger = new Logger(SQSSNSService.name);
  private readonly sqsClient: SQSClient;
  private readonly snsClient: SNSClient;

  constructor(private readonly config: ConfigService) {
    this.sqsClient = new SQSClient({
      region: config.get('AWS_REGION'),
      credentials: {
        accessKeyId: config.get('AWS_ACCESS_KEY_ID'),
        secretAccessKey: config.get('AWS_SECRET_ACCESS_KEY'),
      },
    });

    this.snsClient = new SNSClient({
      region: config.get('AWS_REGION'),
    });
  }

  // ใช้ SNS เพื่อ fan-out ไปหลาย services
  async publishToSNS(
    topicArn: string,
    eventType: string,
    payload: unknown,
    attributes?: Record<string, string>,
  ): Promise<void> {
    const command = new PublishCommand({
      TopicArn: topicArn,
      Message: JSON.stringify(payload),
      Subject: eventType,
      MessageAttributes: {
        eventType: {
          DataType: 'String',
          StringValue: eventType,
        },
        ...Object.entries(attributes || {}).reduce((acc, [key, value]) => {
          acc[key] = { DataType: 'String', StringValue: value };
          return acc;
        }, {} as Record<string, any>),
      },
      // Message deduplication (สำหรับ FIFO topics)
      MessageDeduplicationId: `${eventType}-${Date.now()}`,
      MessageGroupId: (payload as any).orderId || 'default',
    });

    await this.snsClient.send(command);
  }

  // SQS consumer
  async consumeMessages(
    queueUrl: string,
    handler: (message: Message) => Promise<void>,
  ): Promise<void> {
    while (true) {
      try {
        const receiveCommand = new ReceiveMessageCommand({
          QueueUrl: queueUrl,
          MaxNumberOfMessages: 10,
          WaitTimeSeconds: 20,   // Long polling
          AttributeNames: ['All'],
          MessageAttributeNames: ['All'],
          VisibilityTimeout: 30, // 30 seconds to process
        });

        const response = await this.sqsClient.send(receiveCommand);

        if (!response.Messages || response.Messages.length === 0) {
          continue;
        }

        await Promise.all(
          response.Messages.map(async (message) => {
            try {
              await handler(message);
              
              // Delete after successful processing
              await this.sqsClient.send(
                new DeleteMessageCommand({
                  QueueUrl: queueUrl,
                  ReceiptHandle: message.ReceiptHandle!,
                })
              );
            } catch (error) {
              this.logger.error(
                'Failed to process SQS message',
                {
                  messageId: message.MessageId,
                  error: error.message,
                }
              );
              // ไม่ delete - message จะ reappear หลัง VisibilityTimeout
            }
          })
        );
      } catch (error) {
        this.logger.error('SQS consumer error', error);
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }
  }
}
```

```typescript
// Terraform configuration สำหรับ SQS/SNS
```

```hcl
# infrastructure/terraform/messaging/sqs-sns.tf

# SNS Topic สำหรับ Order Events
resource "aws_sns_topic" "order_events" {
  name                        = "order-events.fifo"
  fifo_topic                  = true
  content_based_deduplication = false
  
  tags = {
    Service     = "order-service"
    Environment = var.environment
  }
}

# SQS Queue สำหรับ Notification Service
resource "aws_sqs_queue" "notification_service_queue" {
  name                        = "notification-service-orders.fifo"
  fifo_queue                  = true
  content_based_deduplication = true
  
  # Visibility timeout ต้องมากกว่า Lambda timeout
  visibility_timeout_seconds  = 300   # 5 minutes
  message_retention_seconds   = 86400 # 24 hours
  
  # Dead Letter Queue
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.notification_dlq.arn
    maxReceiveCount     = 3
  })
}

# Dead Letter Queue
resource "aws_sqs_queue" "notification_dlq" {
  name                      = "notification-service-orders-dlq.fifo"
  fifo_queue                = true
  message_retention_seconds = 1209600 # 14 days
}

# Subscribe SQS to SNS
resource "aws_sns_topic_subscription" "notification_subscription" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.notification_service_queue.arn
  
  filter_policy = jsonencode({
    eventType = ["order.created", "order.cancelled"]
  })
}
```

---

## Google Cloud Pub/Sub

### เมื่อไหร่ควรใช้ Google Pub/Sub?
- GCP-native workloads
- Global message distribution
- Very high throughput (10M+ messages/second)
- Integration กับ BigQuery, Dataflow

```typescript
// services/order/src/messaging/pubsub.service.ts

import { Injectable, Logger, OnModuleInit } from '@nestjs/common';
import { PubSub, Message, Subscription } from '@google-cloud/pubsub';

@Injectable()
export class PubSubService implements OnModuleInit {
  private readonly logger = new Logger(PubSubService.name);
  private pubsub: PubSub;
  private subscriptions: Map<string, Subscription> = new Map();

  async onModuleInit(): Promise<void> {
    this.pubsub = new PubSub({
      projectId: process.env.GOOGLE_CLOUD_PROJECT,
    });
  }

  async publish(
    topicName: string,
    data: unknown,
    attributes?: Record<string, string>,
  ): Promise<string> {
    const topic = this.pubsub.topic(topicName, {
      batching: {
        maxMessages: 1000,
        maxMilliseconds: 100,
      },
      flowControl: {
        maxOutstandingMessages: 100,
      },
    });

    const messageId = await topic.publishMessage({
      json: data,
      attributes: {
        source: 'order-service',
        environment: process.env.NODE_ENV || 'production',
        ...attributes,
      },
    });

    return messageId;
  }

  async subscribe(
    subscriptionName: string,
    handler: (message: Message) => Promise<void>,
  ): Promise<void> {
    const subscription = this.pubsub.subscription(subscriptionName, {
      flowControl: {
        maxMessages: 100,
        allowExcessMessages: false,
      },
      ackDeadline: 60, // 60 seconds
    });

    this.subscriptions.set(subscriptionName, subscription);

    subscription.on('message', async (message: Message) => {
      try {
        await handler(message);
        message.ack();
      } catch (error) {
        this.logger.error('Failed to process Pub/Sub message', {
          messageId: message.id,
          error: error.message,
        });
        message.nack(); // จะ retry ตาม backoff policy
      }
    });

    subscription.on('error', (error) => {
      this.logger.error('Pub/Sub subscription error', error);
    });
  }
}
```

---

## Azure Service Bus

### เมื่อไหร่ควรใช้ Azure Service Bus?
- Azure-native workloads
- Enterprise messaging patterns
- Transaction support
- Message sessions สำหรับ ordered processing

```typescript
// services/order/src/messaging/azure-servicebus.service.ts

import { Injectable, Logger } from '@nestjs/common';
import {
  ServiceBusClient,
  ServiceBusSender,
  ServiceBusMessage,
  ProcessErrorArgs,
} from '@azure/service-bus';

@Injectable()
export class AzureServiceBusService {
  private readonly logger = new Logger(AzureServiceBusService.name);
  private client: ServiceBusClient;
  private senders: Map<string, ServiceBusSender> = new Map();

  constructor(private readonly config: ConfigService) {
    this.client = new ServiceBusClient(
      config.get('AZURE_SERVICE_BUS_CONNECTION_STRING')
    );
  }

  async publish(
    topicName: string,
    eventType: string,
    payload: unknown,
    sessionId?: string, // สำหรับ ordered processing
  ): Promise<void> {
    let sender = this.senders.get(topicName);
    if (!sender) {
      sender = this.client.createSender(topicName);
      this.senders.set(topicName, sender);
    }

    const message: ServiceBusMessage = {
      body: payload,
      contentType: 'application/json',
      subject: eventType,
      sessionId,  // Messages กับ same sessionId จะ process ตามลำดับ
      messageId: `${eventType}-${Date.now()}`,
      applicationProperties: {
        sourceService: 'order-service',
        eventType,
        schemaVersion: '1.0',
      },
    };

    await sender.sendMessages(message);
  }

  async subscribe(
    topicName: string,
    subscriptionName: string,
    handler: (message: any) => Promise<void>,
  ): Promise<void> {
    const receiver = this.client.createReceiver(
      topicName,
      subscriptionName,
      { receiveMode: 'peekLock' }
    );

    receiver.subscribe({
      processMessage: async (message) => {
        try {
          await handler(message.body);
          await receiver.completeMessage(message);
        } catch (error) {
          // Dead letter หลังจาก max delivery count
          await receiver.abandonMessage(message);
          this.logger.error('Message processing failed', {
            messageId: message.messageId,
            deliveryCount: message.deliveryCount,
            error: error.message,
          });
        }
      },
      processError: async (error: ProcessErrorArgs) => {
        this.logger.error('Service Bus error', error.error);
      },
    });
  }
}
```

---

## Performance Characteristics & Cost Comparison

```
Performance Benchmark (approximate):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Kafka (3 brokers, 12 partitions):
  Producer throughput: 1,000,000+ msgs/s
  Consumer throughput: 500,000+ msgs/s
  P99 latency:         < 10ms
  Message retention:   Configurable (days to forever)

RabbitMQ (clustered, 3 nodes):
  Producer throughput: 100,000 msgs/s
  Consumer throughput: 100,000 msgs/s
  P99 latency:         < 5ms
  Message retention:   Until consumed + TTL

AWS SQS Standard:
  Producer throughput: Unlimited (auto-scaling)
  Consumer throughput: 3,000 msgs/s per queue (standard)
  P99 latency:         < 100ms
  Message retention:   Up to 14 days

Google Pub/Sub:
  Producer throughput: 10,000,000 msgs/s
  Consumer throughput: 10,000,000 msgs/s
  P99 latency:         < 100ms
  Message retention:   Up to 7 days (with snapshot up to 31 days)
```

```
Cost Comparison (USD, approximate):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Scenario: 10M messages/day, 1KB each = 10GB/day

RabbitMQ (self-hosted):
  EC2 m5.xlarge x3:  ~$600/month
  EBS storage:        ~$100/month
  Total:             ~$700/month

Kafka (Confluent Cloud):
  Basic cluster:      ~$200/month
  Storage 10GB/day:   ~$300/month
  Total:             ~$500/month

AWS SQS Standard:
  $0.40 per 1M requests
  10M msgs/day x 30 = 300M msgs/month
  Total:             ~$120/month

Google Pub/Sub:
  $0.04 per GB
  10GB/day x 30 = 300GB/month
  Total:             ~$12/month ← ถูกมาก!

Azure Service Bus:
  Standard: $0.01 per 1M messages
  Premium: from $677/month (dedicated)
  Standard total:    ~$3/month
```

---

## Migration Strategies

### จาก RabbitMQ ไป Kafka

```typescript
// tools/migration/src/rabbitmq-to-kafka.ts
// Dual-write strategy สำหรับ zero-downtime migration

@Injectable()
export class DualWritePublisher {
  private readonly migrationPhase: 'rabbitmq-only' | 'dual-write' | 'kafka-only';

  constructor(
    private readonly rabbitPublisher: RabbitMQPublisher,
    private readonly kafkaPublisher: KafkaPublisher,
    private readonly configService: ConfigService,
  ) {
    this.migrationPhase = this.configService.get(
      'MIGRATION_PHASE',
      'rabbitmq-only'
    );
  }

  async publish(event: string, payload: unknown): Promise<void> {
    switch (this.migrationPhase) {
      case 'rabbitmq-only':
        await this.rabbitPublisher.publish(event, payload);
        break;
        
      case 'dual-write':
        // Write ไปทั้งสอง - consumers ใช้ Kafka แต่ยัง fallback ได้
        await Promise.all([
          this.rabbitPublisher.publish(event, payload),
          this.kafkaPublisher.publish(event, payload),
        ]);
        break;
        
      case 'kafka-only':
        await this.kafkaPublisher.publish(event, payload);
        break;
    }
  }
}
```

```
Migration Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase 1 (สัปดาห์ 1-2): Setup Kafka cluster
Phase 2 (สัปดาห์ 3-4): Dual-write (produce ไปทั้ง RabbitMQ + Kafka)
Phase 3 (สัปดาห์ 5-6): Migrate consumers ทีละ service
Phase 4 (สัปดาห์ 7-8): Switch to Kafka-only, monitor
Phase 5 (สัปดาห์ 9): Decommission RabbitMQ
```

---

## Decision Framework

```
เลือก Message Broker อย่างไร?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

คำถาม 1: ต้องการ message replay ไหม?
  ใช่ → Kafka หรือ GCP Pub/Sub
  ไม่ → ไปคำถาม 2

คำถาม 2: Cloud provider ที่ใช้คือ?
  AWS → SQS/SNS
  GCP → Pub/Sub
  Azure → Service Bus
  On-premise/Hybrid → ไปคำถาม 3

คำถาม 3: Throughput เท่าไหร่?
  > 100K msgs/s → Kafka
  < 100K msgs/s → RabbitMQ

คำถาม 4: ต้องการ complex routing ไหม?
  ใช่ (multiple exchange types) → RabbitMQ
  ไม่ → ขึ้นอยู่กับปัจจัยอื่น

คำถาม 5: มี team expertise ไหม?
  Kafka expertise ต่ำ → เริ่มด้วย RabbitMQ หรือ managed service
  High operational burden ไม่ต้องการ → Managed service
```

---

## สรุป

| Broker | Strengths | Weaknesses | Best For |
|--------|-----------|------------|---------|
| RabbitMQ | Flexible routing, low latency | No replay, lower throughput | Task queues, complex routing |
| Kafka | High throughput, replay, event log | High complexity, resource intensive | Event streaming, analytics |
| AWS SQS/SNS | Managed, scalable, cheap | Limited features | AWS ecosystem, serverless |
| GCP Pub/Sub | Very high throughput, managed | GCP lock-in | GCP ecosystem, big data |
| Azure Service Bus | Enterprise features, sessions | Azure lock-in, cost | Azure ecosystem, enterprise |

สำหรับ Thai tech companies ที่ใช้ cloud:
- ถ้า use AWS → SQS/SNS เป็น default, Kafka เมื่อต้องการ replay/streaming
- ถ้า use GCP → Pub/Sub เป็น default
- ถ้า self-hosted → RabbitMQ สำหรับ general purpose, Kafka สำหรับ event streaming
- ถ้า scale ใหญ่ (เช่น Shopee, Lazada Thailand) → Kafka เป็นหลัก

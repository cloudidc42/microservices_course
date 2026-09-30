# Part 39: Message Broker Patterns - Advanced Kafka

## บทนำ

Apache Kafka เป็น distributed event streaming platform ที่ขับเคลื่อน Microservices ระดับ World-Class ในบทนี้เราจะเรียนรู้ pattern ขั้นสูงสำหรับ Kafka รวมถึง Exactly-Once Semantics, Kafka Streams, Schema Evolution, และ Multi-Tenant Kafka

## 1. Kafka Architecture Deep Dive

```
┌─────────────────────────────────────────────────────┐
│                   Kafka Cluster                      │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │          │
│  │          │  │          │  │          │          │
│  │ Topic A  │  │ Topic A  │  │ Topic A  │          │
│  │ Part 0   │  │ Part 1   │  │ Part 2   │          │
│  │(Leader)  │  │(Leader)  │  │(Leader)  │          │
│  │ Part 1   │  │ Part 2   │  │ Part 0   │          │
│  │(Follower)│  │(Follower)│  │(Follower)│          │
│  └──────────┘  └──────────┘  └──────────┘          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 2. Producer Patterns

### 2.1 Exactly-Once Producer (Idempotent + Transactional)

```javascript
// kafka/exactlyOnceProducer.js
const { Kafka, CompressionTypes } = require('kafkajs');

class ExactlyOnceProducer {
  constructor(config) {
    this.kafka = new Kafka({
      clientId: config.clientId,
      brokers: config.brokers,
    });
    
    this.producer = this.kafka.producer({
      // Enable idempotent producer (exactly-once within single partition)
      idempotent: true,
      
      // Enable transactions for cross-partition exactly-once
      transactionalId: config.transactionalId,
      
      maxInFlightRequests: 5, // Must be <= 5 for idempotent
      
      // Performance tuning
      compression: CompressionTypes.SNAPPY,
      
      retry: {
        initialRetryTime: 100,
        maxRetryTime: 30000,
        retries: 8,
        multiplier: 2,
        factor: 0.2,
        restartOnFailure: async () => true,
      },
    });
  }
  
  async connect() {
    await this.producer.connect();
  }
  
  async sendWithTransaction(messages) {
    const transaction = await this.producer.transaction();
    
    try {
      for (const { topic, messages: msgs } of messages) {
        await transaction.send({
          topic,
          messages: msgs.map(msg => ({
            key: msg.key,
            value: typeof msg.value === 'object' 
              ? JSON.stringify(msg.value) 
              : msg.value,
            headers: {
              'message-id': msg.id || require('crypto').randomUUID(),
              'created-at': new Date().toISOString(),
              'source-service': process.env.SERVICE_NAME || 'unknown',
            },
          })),
        });
      }
      
      await transaction.commit();
      console.log('Transaction committed successfully');
      
    } catch (error) {
      await transaction.abort();
      console.error('Transaction aborted:', error);
      throw error;
    }
  }
  
  // Consume from one topic and produce to another in single transaction
  async consumeTransformProduce(consumer, inputTopic, outputTopic, transformFn) {
    await consumer.run({
      eachBatch: async ({ batch, resolveOffset, heartbeat, commitOffsetsIfNecessary }) => {
        for (const message of batch.messages) {
          const transaction = await this.producer.transaction();
          
          try {
            const inputValue = JSON.parse(message.value.toString());
            const outputValue = await transformFn(inputValue);
            
            // Send to output topic
            await transaction.send({
              topic: outputTopic,
              messages: [{
                key: message.key,
                value: JSON.stringify(outputValue),
              }],
            });
            
            // Commit offset as part of same transaction
            await transaction.sendOffsets({
              consumerGroupId: consumer.groupId,
              topics: [{
                topic: inputTopic,
                partitions: [{
                  partition: batch.partition,
                  offset: (BigInt(message.offset) + 1n).toString(),
                }],
              }],
            });
            
            await transaction.commit();
            resolveOffset(message.offset);
            await heartbeat();
            
          } catch (error) {
            await transaction.abort();
            throw error; // Will trigger retry
          }
        }
      },
    });
  }
  
  async disconnect() {
    await this.producer.disconnect();
  }
}

module.exports = ExactlyOnceProducer;
```

### 2.2 Partitioning Strategies

```javascript
// kafka/customPartitioner.js

// Custom partitioner for business logic routing
const customPartitioner = () => {
  return ({ topic, partitionMetadata, message }) => {
    const partitionCount = partitionMetadata.length;
    
    // Priority messages go to partition 0
    if (message.headers?.priority === 'high') {
      return 0;
    }
    
    // Route by region
    const region = message.headers?.region?.toString();
    if (region) {
      const regionMap = { 'ap': 0, 'us': 1, 'eu': 2 };
      if (regionMap[region] !== undefined) {
        return regionMap[region] % partitionCount;
      }
    }
    
    // Default: consistent hash by message key
    if (message.key) {
      const keyStr = message.key.toString();
      let hash = 0;
      for (let i = 0; i < keyStr.length; i++) {
        hash = ((hash << 5) - hash) + keyStr.charCodeAt(i);
        hash |= 0; // Convert to 32bit integer
      }
      return Math.abs(hash) % partitionCount;
    }
    
    // No key: round-robin
    return Math.floor(Math.random() * partitionCount);
  };
};

// Usage
const producer = kafka.producer({
  createPartitioner: customPartitioner,
});
```

## 3. Consumer Patterns

### 3.1 Consumer Group with Parallel Processing

```javascript
// kafka/parallelConsumer.js
const { Kafka } = require('kafkajs');
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
const os = require('os');

class ParallelKafkaConsumer {
  constructor(config) {
    this.kafka = new Kafka({
      clientId: config.clientId,
      brokers: config.brokers,
    });
    
    this.consumer = this.kafka.consumer({
      groupId: config.groupId,
      // Fetch larger batches for parallel processing
      maxBytesPerPartition: 1024 * 1024, // 1MB per partition
      sessionTimeout: 60000,
      heartbeatInterval: 3000,
    });
    
    this.workerCount = config.workerCount || os.cpus().length;
    this.processorFn = config.processorFn;
    this.topic = config.topic;
  }
  
  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({ topic: this.topic, fromBeginning: false });
    
    await this.consumer.run({
      partitionsConsumedConcurrently: 4, // Process 4 partitions in parallel
      eachBatch: async ({ batch, resolveOffset, heartbeat, isRunning, isStale }) => {
        console.log(`Processing batch: ${batch.messages.length} messages from partition ${batch.partition}`);
        
        // Split batch into chunks for parallel processing
        const chunkSize = Math.ceil(batch.messages.length / this.workerCount);
        const chunks = [];
        
        for (let i = 0; i < batch.messages.length; i += chunkSize) {
          chunks.push(batch.messages.slice(i, i + chunkSize));
        }
        
        // Process chunks in parallel
        await Promise.all(
          chunks.map(async (chunk) => {
            for (const message of chunk) {
              if (!isRunning() || isStale()) return;
              
              try {
                await this.processorFn(
                  JSON.parse(message.value.toString()),
                  {
                    topic: batch.topic,
                    partition: batch.partition,
                    offset: message.offset,
                    timestamp: message.timestamp,
                    headers: message.headers,
                  }
                );
                
                resolveOffset(message.offset);
              } catch (error) {
                console.error(`Failed to process message at offset ${message.offset}:`, error);
                // In real scenario: send to DLQ
                await this.sendToDeadLetterQueue(message, error);
                resolveOffset(message.offset); // Skip failed message
              }
              
              await heartbeat();
            }
          })
        );
      },
    });
  }
  
  async sendToDeadLetterQueue(message, error) {
    const dlqProducer = this.kafka.producer();
    await dlqProducer.connect();
    
    await dlqProducer.send({
      topic: `${this.topic}.dlq`,
      messages: [{
        key: message.key,
        value: message.value,
        headers: {
          ...message.headers,
          'dlq-reason': error.message,
          'dlq-timestamp': new Date().toISOString(),
          'original-offset': message.offset,
          'original-partition': String(message.partition),
        },
      }],
    });
    
    await dlqProducer.disconnect();
  }
  
  async stop() {
    await this.consumer.disconnect();
  }
}

module.exports = ParallelKafkaConsumer;
```

### 3.2 Consumer Lag Monitor

```javascript
// monitoring/consumerLagMonitor.js
const { Kafka } = require('kafkajs');
const prometheus = require('prom-client');

class ConsumerLagMonitor {
  constructor(config) {
    this.kafka = new Kafka({
      clientId: 'lag-monitor',
      brokers: config.brokers,
    });
    
    this.admin = this.kafka.admin();
    this.groups = config.groups;
    
    // Prometheus metrics
    this.lagGauge = new prometheus.Gauge({
      name: 'kafka_consumer_group_lag',
      help: 'Consumer group lag per partition',
      labelNames: ['group', 'topic', 'partition'],
    });
    
    this.offsetGauge = new prometheus.Gauge({
      name: 'kafka_consumer_group_offset',
      help: 'Consumer group current offset',
      labelNames: ['group', 'topic', 'partition'],
    });
    
    this.latestOffsetGauge = new prometheus.Gauge({
      name: 'kafka_topic_latest_offset',
      help: 'Latest offset in topic partition',
      labelNames: ['topic', 'partition'],
    });
  }
  
  async start() {
    await this.admin.connect();
    
    setInterval(async () => {
      try {
        await this.collectMetrics();
      } catch (error) {
        console.error('Error collecting Kafka metrics:', error);
      }
    }, 15000); // Every 15 seconds
  }
  
  async collectMetrics() {
    for (const groupId of this.groups) {
      try {
        const [offsets, description] = await Promise.all([
          this.admin.fetchOffsets({ groupId }),
          this.admin.describeGroups([groupId]),
        ]);
        
        if (!offsets || description.groups[0].state === 'Dead') continue;
        
        for (const topicOffsets of offsets) {
          // Get latest offsets for topic
          const latestOffsets = await this.admin.fetchTopicOffsets(topicOffsets.topic);
          const latestMap = new Map(
            latestOffsets.map(p => [p.partition, parseInt(p.offset)])
          );
          
          for (const partition of topicOffsets.partitions) {
            const currentOffset = parseInt(partition.offset);
            const latestOffset = latestMap.get(partition.partition) || 0;
            const lag = Math.max(0, latestOffset - currentOffset);
            
            this.lagGauge.set(
              { group: groupId, topic: topicOffsets.topic, partition: String(partition.partition) },
              lag
            );
            
            this.offsetGauge.set(
              { group: groupId, topic: topicOffsets.topic, partition: String(partition.partition) },
              currentOffset
            );
            
            this.latestOffsetGauge.set(
              { topic: topicOffsets.topic, partition: String(partition.partition) },
              latestOffset
            );
            
            // Alert on high lag
            if (lag > 10000) {
              console.warn(`HIGH LAG ALERT: group=${groupId} topic=${topicOffsets.topic} partition=${partition.partition} lag=${lag}`);
            }
          }
        }
      } catch (error) {
        console.error(`Error collecting metrics for group ${groupId}:`, error);
      }
    }
  }
  
  async stop() {
    await this.admin.disconnect();
  }
}

module.exports = ConsumerLagMonitor;
```

## 4. Schema Evolution with Avro + Schema Registry

### 4.1 Schema Registry Client

```javascript
// kafka/schemaRegistry.js
const { SchemaRegistry } = require('@kafkajs/confluent-schema-registry');
const { Kafka } = require('kafkajs');

class AvroKafkaProducer {
  constructor(config) {
    this.kafka = new Kafka({ clientId: config.clientId, brokers: config.brokers });
    this.registry = new SchemaRegistry({ host: config.schemaRegistryUrl });
    this.producer = this.kafka.producer();
    this.schemaCache = new Map();
  }
  
  async connect() {
    await this.producer.connect();
  }
  
  async getSchemaId(topic, schema) {
    const subject = `${topic}-value`;
    
    if (this.schemaCache.has(subject)) {
      return this.schemaCache.get(subject);
    }
    
    const { id } = await this.registry.register(
      { type: 'AVRO', schema: JSON.stringify(schema) },
      { subject }
    );
    
    this.schemaCache.set(subject, id);
    return id;
  }
  
  async send(topic, messages, schema) {
    const schemaId = await this.getSchemaId(topic, schema);
    
    const encodedMessages = await Promise.all(
      messages.map(async (msg) => ({
        key: msg.key ? Buffer.from(msg.key) : null,
        value: await this.registry.encode(schemaId, msg.value),
        headers: msg.headers,
      }))
    );
    
    await this.producer.send({ topic, messages: encodedMessages });
  }
  
  async disconnect() {
    await this.producer.disconnect();
  }
}

class AvroKafkaConsumer {
  constructor(config) {
    this.kafka = new Kafka({ clientId: config.clientId, brokers: config.brokers });
    this.registry = new SchemaRegistry({ host: config.schemaRegistryUrl });
    this.consumer = this.kafka.consumer({ groupId: config.groupId });
  }
  
  async start(topics, handler) {
    await this.consumer.connect();
    await this.consumer.subscribe({ topics, fromBeginning: false });
    
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        // Decode Avro message automatically
        const decodedValue = await this.registry.decode(message.value);
        
        await handler({
          topic,
          partition,
          offset: message.offset,
          key: message.key?.toString(),
          value: decodedValue,
          headers: message.headers,
        });
      },
    });
  }
  
  async stop() {
    await this.consumer.disconnect();
  }
}

// Schema definitions with version history
const OrderEventSchemaV1 = {
  type: 'record',
  name: 'OrderEvent',
  namespace: 'com.example.ecommerce',
  fields: [
    { name: 'orderId', type: 'string' },
    { name: 'customerId', type: 'string' },
    { name: 'status', type: { type: 'enum', name: 'OrderStatus', symbols: ['CREATED', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'] } },
    { name: 'totalAmount', type: 'double' },
    { name: 'currency', type: 'string' },
    { name: 'createdAt', type: 'long', logicalType: 'timestamp-millis' },
  ],
};

// V2: Added optional fields (backward compatible)
const OrderEventSchemaV2 = {
  ...OrderEventSchemaV1,
  fields: [
    ...OrderEventSchemaV1.fields,
    { name: 'region', type: ['null', 'string'], default: null },  // New optional field
    { name: 'metadata', type: ['null', { type: 'map', values: 'string' }], default: null },
  ],
};

module.exports = { AvroKafkaProducer, AvroKafkaConsumer, OrderEventSchemaV1, OrderEventSchemaV2 };
```

## 5. Kafka Streams (Stream Processing)

### 5.1 Real-time Order Analytics

```javascript
// streams/orderAnalytics.js
const { Kafka } = require('kafkajs');
const Redis = require('ioredis');

class OrderAnalyticsStream {
  constructor(config) {
    this.kafka = new Kafka({
      clientId: 'order-analytics',
      brokers: config.brokers,
    });
    
    this.consumer = this.kafka.consumer({ groupId: 'order-analytics-group' });
    this.producer = this.kafka.producer();
    this.redis = new Redis(config.redisUrl);
    
    // State store for windowed aggregations
    this.windowDuration = 5 * 60 * 1000; // 5 minute windows
  }
  
  async start() {
    await this.consumer.connect();
    await this.producer.connect();
    
    await this.consumer.subscribe({
      topics: ['orders.events'],
      fromBeginning: false,
    });
    
    await this.consumer.run({
      eachMessage: async ({ message }) => {
        const event = JSON.parse(message.value.toString());
        await this.processEvent(event, parseInt(message.timestamp));
      },
    });
  }
  
  async processEvent(event, eventTimestamp) {
    const windowKey = this.getWindowKey(eventTimestamp);
    
    switch (event.type) {
      case 'ORDER_CREATED':
        await this.updateWindowedCount(windowKey, 'orders_created', 1);
        break;
      
      case 'ORDER_COMPLETED':
        await this.updateWindowedSum(windowKey, 'revenue', event.payload.totalAmount);
        await this.updateWindowedCount(windowKey, 'orders_completed', 1);
        await this.updateCategoryRevenue(windowKey, event.payload);
        break;
      
      case 'ORDER_CANCELLED':
        await this.updateWindowedCount(windowKey, 'orders_cancelled', 1);
        break;
    }
    
    // Emit aggregated metrics downstream
    await this.emitWindowMetrics(windowKey, eventTimestamp);
  }
  
  getWindowKey(timestamp) {
    const windowStart = Math.floor(timestamp / this.windowDuration) * this.windowDuration;
    return `analytics:window:${windowStart}`;
  }
  
  async updateWindowedCount(windowKey, metric, increment) {
    const pipeline = this.redis.pipeline();
    pipeline.hincrby(windowKey, metric, increment);
    pipeline.pexpire(windowKey, this.windowDuration * 2); // Keep 2x window duration
    await pipeline.exec();
  }
  
  async updateWindowedSum(windowKey, metric, amount) {
    const pipeline = this.redis.pipeline();
    pipeline.hincrbyfloat(windowKey, metric, amount);
    pipeline.pexpire(windowKey, this.windowDuration * 2);
    await pipeline.exec();
  }
  
  async updateCategoryRevenue(windowKey, order) {
    if (!order.items) return;
    
    const pipeline = this.redis.pipeline();
    
    for (const item of order.items) {
      pipeline.hincrbyfloat(`${windowKey}:categories`, item.categoryId, item.subtotal);
    }
    
    pipeline.pexpire(`${windowKey}:categories`, this.windowDuration * 2);
    await pipeline.exec();
  }
  
  async emitWindowMetrics(windowKey, eventTimestamp) {
    const windowStart = Math.floor(eventTimestamp / this.windowDuration) * this.windowDuration;
    
    // Only emit every minute to avoid overloading downstream
    const lastEmit = await this.redis.get(`${windowKey}:last_emit`);
    const now = Date.now();
    
    if (lastEmit && now - parseInt(lastEmit) < 60000) return;
    
    const [metrics, categoryMetrics] = await Promise.all([
      this.redis.hgetall(windowKey),
      this.redis.hgetall(`${windowKey}:categories`),
    ]);
    
    if (!metrics) return;
    
    await this.producer.send({
      topic: 'analytics.window.metrics',
      messages: [{
        key: String(windowStart),
        value: JSON.stringify({
          windowStart: new Date(windowStart).toISOString(),
          windowEnd: new Date(windowStart + this.windowDuration).toISOString(),
          metrics: {
            ordersCreated: parseInt(metrics.orders_created || 0),
            ordersCompleted: parseInt(metrics.orders_completed || 0),
            ordersCancelled: parseInt(metrics.orders_cancelled || 0),
            revenue: parseFloat(metrics.revenue || 0),
            avgOrderValue: metrics.orders_completed > 0
              ? parseFloat(metrics.revenue || 0) / parseInt(metrics.orders_completed)
              : 0,
          },
          byCategory: categoryMetrics 
            ? Object.fromEntries(
                Object.entries(categoryMetrics).map(([k, v]) => [k, parseFloat(v)])
              )
            : {},
        }),
      }],
    });
    
    await this.redis.setex(`${windowKey}:last_emit`, 120, String(now));
  }
  
  async stop() {
    await this.consumer.disconnect();
    await this.producer.disconnect();
    await this.redis.quit();
  }
}

module.exports = OrderAnalyticsStream;
```

## 6. Multi-Tenant Kafka

```javascript
// kafka/multiTenantKafka.js
const { Kafka } = require('kafkajs');

class MultiTenantKafkaManager {
  constructor(config) {
    this.kafka = new Kafka({
      clientId: 'mt-manager',
      brokers: config.brokers,
    });
    
    this.admin = this.kafka.admin();
    this.tenantProducers = new Map();
    this.topicPrefix = config.topicPrefix || 'tenant';
  }
  
  async connect() {
    await this.admin.connect();
  }
  
  // Provision topics for a new tenant
  async provisionTenant(tenantId, config = {}) {
    const topics = [
      `${this.topicPrefix}.${tenantId}.orders.events`,
      `${this.topicPrefix}.${tenantId}.inventory.events`,
      `${this.topicPrefix}.${tenantId}.notifications`,
      `${this.topicPrefix}.${tenantId}.audit.log`,
    ];
    
    await this.admin.createTopics({
      topics: topics.map(topic => ({
        topic,
        numPartitions: config.partitions || 6,
        replicationFactor: config.replicationFactor || 3,
        configEntries: [
          { name: 'retention.ms', value: String(config.retentionDays * 86400000 || 7 * 86400000) },
          { name: 'cleanup.policy', value: 'delete' },
          { name: 'min.insync.replicas', value: '2' },
          // Quota configuration per tenant
          { name: 'quota.producer.byte-rate', value: String(config.producerBytesPerSec || 10485760) }, // 10MB/s
        ],
      })),
      waitForLeaders: true,
    });
    
    console.log(`Provisioned ${topics.length} topics for tenant ${tenantId}`);
    return topics;
  }
  
  // Get/create producer for a tenant
  async getTenantProducer(tenantId) {
    if (this.tenantProducers.has(tenantId)) {
      return this.tenantProducers.get(tenantId);
    }
    
    const producer = this.kafka.producer({
      idempotent: true,
      transactionalId: `tenant-${tenantId}-${Date.now()}`,
    });
    
    await producer.connect();
    this.tenantProducers.set(tenantId, producer);
    return producer;
  }
  
  // Send event for a specific tenant
  async sendEvent(tenantId, eventType, payload) {
    const producer = await this.getTenantProducer(tenantId);
    const topic = `${this.topicPrefix}.${tenantId}.${this.eventTypeToTopic(eventType)}`;
    
    await producer.send({
      topic,
      messages: [{
        key: payload.id || payload.orderId,
        value: JSON.stringify({
          tenantId,
          eventType,
          payload,
          timestamp: new Date().toISOString(),
          correlationId: payload.correlationId || require('crypto').randomUUID(),
        }),
        headers: {
          'tenant-id': tenantId,
          'event-type': eventType,
        },
      }],
    });
  }
  
  eventTypeToTopic(eventType) {
    if (eventType.startsWith('ORDER_')) return 'orders.events';
    if (eventType.startsWith('INVENTORY_')) return 'inventory.events';
    if (eventType.startsWith('NOTIFICATION_')) return 'notifications';
    return 'audit.log';
  }
  
  // Remove tenant and all their topics
  async deprovisionTenant(tenantId) {
    const { topics } = await this.admin.fetchTopicMetadata({
      topics: [],
    });
    
    const tenantTopics = topics
      .filter(t => t.name.startsWith(`${this.topicPrefix}.${tenantId}.`))
      .map(t => t.name);
    
    if (tenantTopics.length > 0) {
      await this.admin.deleteTopics({ topics: tenantTopics });
      console.log(`Deprovisioned ${tenantTopics.length} topics for tenant ${tenantId}`);
    }
    
    // Disconnect tenant producer
    const producer = this.tenantProducers.get(tenantId);
    if (producer) {
      await producer.disconnect();
      this.tenantProducers.delete(tenantId);
    }
  }
  
  async disconnect() {
    for (const [, producer] of this.tenantProducers) {
      await producer.disconnect();
    }
    await this.admin.disconnect();
  }
}

module.exports = MultiTenantKafkaManager;
```

## 7. Outbox Pattern (Guaranteed Message Delivery)

```javascript
// patterns/outbox.js
const { Pool } = require('pg');
const { Kafka } = require('kafkajs');

class OutboxPattern {
  constructor(config) {
    this.db = new Pool(config.database);
    this.kafka = new Kafka({
      clientId: 'outbox-publisher',
      brokers: config.kafka.brokers,
    });
    this.producer = this.kafka.producer({ idempotent: true });
    this.pollInterval = config.pollInterval || 1000;
  }
  
  // SQL: Create outbox table
  // CREATE TABLE outbox (
  //   id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  //   aggregate_type VARCHAR(255) NOT NULL,
  //   aggregate_id VARCHAR(255) NOT NULL,
  //   event_type VARCHAR(255) NOT NULL,
  //   payload JSONB NOT NULL,
  //   published_at TIMESTAMPTZ,
  //   created_at TIMESTAMPTZ DEFAULT NOW()
  // );
  // CREATE INDEX idx_outbox_unpublished ON outbox(created_at) WHERE published_at IS NULL;
  
  // Write to DB + outbox in same transaction
  async persistAndQueue(client, event) {
    await client.query(
      `INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload)
       VALUES ($1, $2, $3, $4)`,
      [event.aggregateType, event.aggregateId, event.type, JSON.stringify(event.payload)]
    );
  }
  
  // Background publisher: reads outbox and publishes to Kafka
  async startPublisher() {
    await this.producer.connect();
    
    console.log('Starting outbox publisher...');
    
    this.publisherInterval = setInterval(async () => {
      try {
        await this.publishPendingEvents();
      } catch (error) {
        console.error('Outbox publisher error:', error);
      }
    }, this.pollInterval);
  }
  
  async publishPendingEvents() {
    const client = await this.db.connect();
    
    try {
      await client.query('BEGIN');
      
      // Lock and fetch unpublished events (SKIP LOCKED = no blocking)
      const result = await client.query(`
        SELECT * FROM outbox
        WHERE published_at IS NULL
        ORDER BY created_at ASC
        LIMIT 100
        FOR UPDATE SKIP LOCKED
      `);
      
      if (result.rows.length === 0) {
        await client.query('ROLLBACK');
        return;
      }
      
      const messages = result.rows.map(event => ({
        topic: this.getTopicForEvent(event.aggregate_type, event.event_type),
        messages: [{
          key: event.aggregate_id,
          value: JSON.stringify({
            id: event.id,
            aggregateType: event.aggregate_type,
            aggregateId: event.aggregate_id,
            type: event.event_type,
            payload: event.payload,
            createdAt: event.created_at,
          }),
          headers: {
            'aggregate-type': event.aggregate_type,
            'event-type': event.event_type,
            'outbox-id': event.id,
          },
        }],
      }));
      
      // Send to Kafka (group by topic for efficiency)
      const topicMessages = this.groupByTopic(messages);
      await this.producer.sendBatch({ topicMessages });
      
      // Mark as published
      const ids = result.rows.map(r => r.id);
      await client.query(
        `UPDATE outbox SET published_at = NOW() WHERE id = ANY($1)`,
        [ids]
      );
      
      await client.query('COMMIT');
      
      console.log(`Published ${result.rows.length} events from outbox`);
      
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  getTopicForEvent(aggregateType, eventType) {
    const mapping = {
      'order': 'orders.events',
      'product': 'products.events',
      'inventory': 'inventory.events',
      'payment': 'payments.events',
    };
    return mapping[aggregateType.toLowerCase()] || `${aggregateType.toLowerCase()}.events`;
  }
  
  groupByTopic(messages) {
    const grouped = new Map();
    for (const { topic, messages: msgs } of messages) {
      if (!grouped.has(topic)) grouped.set(topic, []);
      grouped.get(topic).push(...msgs);
    }
    return Array.from(grouped.entries()).map(([topic, messages]) => ({ topic, messages }));
  }
  
  async stopPublisher() {
    if (this.publisherInterval) {
      clearInterval(this.publisherInterval);
    }
    await this.producer.disconnect();
  }
}

// Usage in service
async function createOrderWithOutbox(orderData) {
  const outbox = new OutboxPattern({ database: dbConfig, kafka: kafkaConfig });
  const client = await db.connect();
  
  try {
    await client.query('BEGIN');
    
    // Insert order
    const { rows: [order] } = await client.query(
      'INSERT INTO orders (customer_id, total_amount, status) VALUES ($1, $2, $3) RETURNING *',
      [orderData.customerId, orderData.totalAmount, 'CREATED']
    );
    
    // Queue event (same transaction = atomically)
    await outbox.persistAndQueue(client, {
      aggregateType: 'order',
      aggregateId: order.id,
      type: 'ORDER_CREATED',
      payload: { orderId: order.id, customerId: order.customer_id },
    });
    
    await client.query('COMMIT');
    return order;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

module.exports = { OutboxPattern, createOrderWithOutbox };
```

## 8. Kafka Administration & Operations

```javascript
// kafka/admin.js
const { Kafka } = require('kafkajs');

class KafkaAdmin {
  constructor(config) {
    this.kafka = new Kafka({
      clientId: 'kafka-admin',
      brokers: config.brokers,
    });
    this.admin = this.kafka.admin();
  }
  
  async connect() {
    await this.admin.connect();
  }
  
  async getClusterInfo() {
    const { brokers, controller, clusterId } = await this.admin.describeCluster();
    return { brokers, controllerId: controller, clusterId };
  }
  
  async getConsumerGroupLag(groupId) {
    const offsets = await this.admin.fetchOffsets({ groupId });
    const result = {};
    
    for (const topic of offsets) {
      result[topic.topic] = {};
      
      const topicOffsets = await this.admin.fetchTopicOffsets(topic.topic);
      const latestMap = new Map(topicOffsets.map(p => [p.partition, parseInt(p.offset)]));
      
      let totalLag = 0;
      for (const partition of topic.partitions) {
        const latest = latestMap.get(partition.partition) || 0;
        const current = parseInt(partition.offset);
        const lag = Math.max(0, latest - current);
        result[topic.topic][partition.partition] = lag;
        totalLag += lag;
      }
      
      result[topic.topic].total = totalLag;
    }
    
    return result;
  }
  
  async resetConsumerGroup(groupId, topic, toEarliest = false) {
    // First, ensure consumer group is not active
    const description = await this.admin.describeGroups([groupId]);
    if (description.groups[0].state !== 'Empty' && description.groups[0].state !== 'Dead') {
      throw new Error(`Consumer group ${groupId} is active (state: ${description.groups[0].state}). Stop consumers first.`);
    }
    
    const topicOffsets = await this.admin.fetchTopicOffsets(topic);
    
    const offsets = topicOffsets.map(partition => ({
      partition: partition.partition,
      offset: toEarliest ? '0' : partition.offset, // Reset to beginning or latest
    }));
    
    await this.admin.setOffsets({ groupId, topic, partitions: offsets });
    console.log(`Reset consumer group ${groupId} for topic ${topic} to ${toEarliest ? 'earliest' : 'latest'}`);
  }
  
  async getTopicStats(topic) {
    const [metadata, offsets] = await Promise.all([
      this.admin.fetchTopicMetadata({ topics: [topic] }),
      this.admin.fetchTopicOffsets(topic),
    ]);
    
    const topicMeta = metadata.topics[0];
    const totalMessages = offsets.reduce((sum, p) => sum + parseInt(p.offset), 0);
    
    return {
      topic,
      partitions: topicMeta.partitions.length,
      replicas: topicMeta.partitions[0]?.replicas?.length || 0,
      totalMessages,
      partitionDetails: offsets.map(p => ({
        partition: p.partition,
        latestOffset: parseInt(p.offset),
      })),
    };
  }
  
  async disconnect() {
    await this.admin.disconnect();
  }
}

module.exports = KafkaAdmin;
```

## 9. Kafka Docker Compose (Production-Ready)

```yaml
# docker-compose.kafka.yml
version: '3.8'
services:
  zookeeper-1:
    image: confluentinc/cp-zookeeper:7.5.0
    hostname: zookeeper-1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_SERVER_ID: 1
      ZOOKEEPER_SERVERS: zookeeper-1:2888:3888;zookeeper-2:2888:3888;zookeeper-3:2888:3888
    volumes:
      - zk1-data:/var/lib/zookeeper/data
      - zk1-logs:/var/lib/zookeeper/log
  
  kafka-1:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka-1
    depends_on: [zookeeper-1]
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_LOG_RETENTION_HOURS: 168  # 7 days
      KAFKA_LOG_SEGMENT_BYTES: 1073741824  # 1GB
      KAFKA_LOG_RETENTION_CHECK_INTERVAL_MS: 300000
      KAFKA_COMPRESSION_TYPE: snappy
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
      KAFKA_JMX_PORT: 9999
      KAFKA_JMX_HOSTNAME: kafka-1
    volumes:
      - kafka1-data:/var/lib/kafka/data
  
  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on: [kafka-1]
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-1:29092,kafka-2:29092,kafka-3:29092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
      SCHEMA_REGISTRY_AVRO_COMPATIBILITY_LEVEL: BACKWARD
  
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:29092,kafka-2:29092,kafka-3:29092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081

volumes:
  zk1-data:
  zk1-logs:
  kafka1-data:
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Exactly-Once Semantics** - Idempotent producer + Transactional API
2. **Custom Partitioner** - Route messages ตาม priority/region/key
3. **Parallel Consumer** - Process partitions แบบ concurrent ด้วย eachBatch
4. **Consumer Lag Monitor** - Prometheus metrics สำหรับ tracking consumer health
5. **Schema Evolution** - Avro + Schema Registry สำหรับ backward-compatible changes
6. **Kafka Streams** - Real-time windowed aggregations สำหรับ analytics
7. **Multi-Tenant Kafka** - Topic isolation ต่อ tenant ด้วย provisioning/deprovisioning
8. **Outbox Pattern** - Guarantee delivery โดยไม่ต้องใช้ 2PC transactions
9. **Kafka Administration** - Consumer group management, lag checking, offset reset

## แบบฝึกหัด

1. Implement exactly-once producer สำหรับ payment events
2. สร้าง custom partitioner ที่ route orders ตาม customer tier
3. Implement Avro schema ที่ backward compatible กับ V1
4. Deploy Outbox pattern สำหรับ order service และทดสอบ at-least-once delivery
5. สร้าง consumer lag alerting ที่ alert เมื่อ lag > 5000 messages

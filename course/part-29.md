# Part 29: Data Pipeline & ETL with Microservices

## บทนำ

ใน Microservices architecture แต่ละ service มี database เป็นของตัวเอง แต่บ่อยครั้งเราต้องการรวมข้อมูลจากหลาย service เพื่อวิเคราะห์, รายงาน, หรือ sync ข้อมูลระหว่าง service ในบทนี้เราจะเรียนรู้การสร้าง Data Pipeline, ETL processes, และ Change Data Capture (CDC)

## 1. Change Data Capture (CDC) with Debezium

### 1.1 Debezium Setup with Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: orders_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    command:
      - "postgres"
      - "-c"
      - "wal_level=logical"         # Enable logical replication
      - "-c"
      - "max_wal_senders=10"
      - "-c"
      - "max_replication_slots=10"
    ports:
      - "5432:5432"
  
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
  
  kafka-connect:
    image: debezium/connect:2.5
    depends_on: [kafka, postgres]
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:29092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses
      KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      KEY_CONVERTER_SCHEMAS_ENABLE: 'false'
      VALUE_CONVERTER_SCHEMAS_ENABLE: 'false'
  
  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    depends_on: [kafka]
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:29092
```

### 1.2 Register Debezium Connector

```bash
# Register PostgreSQL CDC connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "orders-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "postgres",
      "database.port": "5432",
      "database.user": "postgres",
      "database.password": "password",
      "database.dbname": "orders_db",
      "database.server.name": "orders",
      "table.include.list": "public.orders,public.order_items",
      "plugin.name": "pgoutput",
      "publication.name": "dbz_publication",
      "slot.name": "dbz_slot",
      "topic.prefix": "cdc",
      "decimal.handling.mode": "double",
      "time.precision.mode": "connect",
      "snapshot.mode": "initial",
      "transforms": "route",
      "transforms.route.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
      "transforms.route.renames": "before:previousState,after:currentState"
    }
  }'
```

### 1.3 CDC Event Processor

```javascript
// services/cdcProcessor.js
const { Kafka } = require('kafkajs');

class CDCEventProcessor {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'cdc-processor',
      brokers: [process.env.KAFKA_BROKER || 'localhost:9092'],
    });
    
    this.consumer = this.kafka.consumer({ groupId: 'data-pipeline-group' });
    this.handlers = new Map();
  }
  
  // Register handlers per table
  onInsert(table, handler) {
    if (!this.handlers.has(table)) {
      this.handlers.set(table, { insert: [], update: [], delete: [] });
    }
    this.handlers.get(table).insert.push(handler);
    return this;
  }
  
  onUpdate(table, handler) {
    if (!this.handlers.has(table)) {
      this.handlers.set(table, { insert: [], update: [], delete: [] });
    }
    this.handlers.get(table).update.push(handler);
    return this;
  }
  
  onDelete(table, handler) {
    if (!this.handlers.has(table)) {
      this.handlers.set(table, { insert: [], update: [], delete: [] });
    }
    this.handlers.get(table).delete.push(handler);
    return this;
  }
  
  async start() {
    await this.consumer.connect();
    await this.consumer.subscribe({
      topics: ['cdc.public.orders', 'cdc.public.order_items'],
      fromBeginning: false,
    });
    
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        try {
          const event = JSON.parse(message.value.toString());
          await this.processEvent(topic, event, message.offset);
        } catch (error) {
          console.error('CDC processing error:', error, { topic, offset: message.offset });
          // Send to dead letter queue
          await this.sendToDeadLetterQueue(topic, message, error);
        }
      },
    });
  }
  
  async processEvent(topic, event, offset) {
    // Extract table name from topic: cdc.public.orders → orders
    const tableName = topic.split('.').pop();
    
    const op = event.op; // 'c'=create, 'u'=update, 'd'=delete, 'r'=read(snapshot)
    const currentState = event.currentState;
    const previousState = event.previousState;
    
    console.log(`CDC Event: table=${tableName} op=${op} id=${currentState?.id || previousState?.id}`);
    
    const tableHandlers = this.handlers.get(tableName);
    if (!tableHandlers) return;
    
    const context = {
      topic,
      offset,
      timestamp: new Date(event.ts_ms),
      transactionId: event.transaction?.id,
    };
    
    if (op === 'c' || op === 'r') {
      for (const handler of tableHandlers.insert) {
        await handler(currentState, context);
      }
    } else if (op === 'u') {
      for (const handler of tableHandlers.update) {
        await handler(currentState, previousState, context);
      }
    } else if (op === 'd') {
      for (const handler of tableHandlers.delete) {
        await handler(previousState, context);
      }
    }
  }
  
  async sendToDeadLetterQueue(originalTopic, message, error) {
    const producer = this.kafka.producer();
    await producer.connect();
    
    await producer.send({
      topic: `${originalTopic}.dlq`,
      messages: [{
        key: message.key,
        value: message.value,
        headers: {
          'original-topic': originalTopic,
          'error-message': error.message,
          'failed-at': new Date().toISOString(),
        },
      }],
    });
    
    await producer.disconnect();
  }
}

module.exports = CDCEventProcessor;
```

## 2. ETL Pipeline in Node.js

### 2.1 ETL Framework

```javascript
// pipeline/etl.js
class ETLPipeline {
  constructor(name) {
    this.name = name;
    this.extractors = [];
    this.transformers = [];
    this.loaders = [];
    this.errorHandlers = [];
    this.metrics = {
      extracted: 0,
      transformed: 0,
      loaded: 0,
      errors: 0,
      startTime: null,
      endTime: null,
    };
  }
  
  extract(extractor) {
    this.extractors.push(extractor);
    return this;
  }
  
  transform(transformer) {
    this.transformers.push(transformer);
    return this;
  }
  
  load(loader) {
    this.loaders.push(loader);
    return this;
  }
  
  onError(handler) {
    this.errorHandlers.push(handler);
    return this;
  }
  
  async* extractData() {
    for (const extractor of this.extractors) {
      yield* extractor.extract();
    }
  }
  
  async transformRecord(record) {
    let transformed = record;
    for (const transformer of this.transformers) {
      transformed = await transformer.transform(transformed);
      if (transformed === null || transformed === undefined) return null;
    }
    return transformed;
  }
  
  async run(options = {}) {
    const { batchSize = 100, dryRun = false } = options;
    this.metrics.startTime = Date.now();
    
    console.log(`Starting ETL pipeline: ${this.name}`);
    
    let batch = [];
    
    try {
      for await (const record of this.extractData()) {
        this.metrics.extracted++;
        
        try {
          const transformed = await this.transformRecord(record);
          if (transformed === null) continue;
          
          this.metrics.transformed++;
          batch.push(transformed);
          
          if (batch.length >= batchSize) {
            if (!dryRun) {
              await this.flushBatch(batch);
            }
            batch = [];
          }
        } catch (transformError) {
          this.metrics.errors++;
          for (const handler of this.errorHandlers) {
            await handler(transformError, record, 'transform');
          }
        }
      }
      
      // Flush remaining batch
      if (batch.length > 0 && !dryRun) {
        await this.flushBatch(batch);
      }
      
    } finally {
      this.metrics.endTime = Date.now();
      const duration = (this.metrics.endTime - this.metrics.startTime) / 1000;
      
      console.log(`ETL pipeline completed: ${this.name}`, {
        extracted: this.metrics.extracted,
        transformed: this.metrics.transformed,
        loaded: this.metrics.loaded,
        errors: this.metrics.errors,
        durationSeconds: duration,
        recordsPerSecond: Math.floor(this.metrics.loaded / duration),
      });
    }
    
    return this.metrics;
  }
  
  async flushBatch(records) {
    for (const loader of this.loaders) {
      await loader.load(records);
    }
    this.metrics.loaded += records.length;
  }
}

module.exports = ETLPipeline;
```

### 2.2 Extractors

```javascript
// pipeline/extractors/postgresExtractor.js
const { Pool } = require('pg');

class PostgresExtractor {
  constructor(config) {
    this.pool = new Pool(config.connection);
    this.query = config.query;
    this.params = config.params || [];
    this.chunkSize = config.chunkSize || 1000;
    this.orderBy = config.orderBy || 'id';
  }
  
  async* extract() {
    let offset = 0;
    let hasMore = true;
    
    while (hasMore) {
      const result = await this.pool.query(
        `${this.query} ORDER BY ${this.orderBy} LIMIT $${this.params.length + 1} OFFSET $${this.params.length + 2}`,
        [...this.params, this.chunkSize, offset]
      );
      
      for (const row of result.rows) {
        yield row;
      }
      
      hasMore = result.rows.length === this.chunkSize;
      offset += result.rows.length;
      
      if (offset % 10000 === 0) {
        console.log(`Extracted ${offset} records so far...`);
      }
    }
  }
}

// pipeline/extractors/incrementalExtractor.js
class IncrementalExtractor {
  constructor(config) {
    this.pool = new Pool(config.connection);
    this.table = config.table;
    this.timestampColumn = config.timestampColumn || 'updated_at';
    this.lastRunFile = config.lastRunFile || `.last_run_${config.table}`;
    this.chunkSize = config.chunkSize || 1000;
  }
  
  getLastRunTime() {
    try {
      const data = require('fs').readFileSync(this.lastRunFile, 'utf8');
      return new Date(data.trim());
    } catch {
      return new Date(0); // Start from epoch if no last run
    }
  }
  
  saveLastRunTime(time) {
    require('fs').writeFileSync(this.lastRunFile, time.toISOString());
  }
  
  async* extract() {
    const lastRun = this.getLastRunTime();
    const currentRun = new Date();
    let offset = 0;
    let hasMore = true;
    
    console.log(`Incremental extract from ${lastRun.toISOString()} to ${currentRun.toISOString()}`);
    
    while (hasMore) {
      const result = await this.pool.query(
        `SELECT * FROM ${this.table}
         WHERE ${this.timestampColumn} > $1 AND ${this.timestampColumn} <= $2
         ORDER BY ${this.timestampColumn}, id
         LIMIT $3 OFFSET $4`,
        [lastRun, currentRun, this.chunkSize, offset]
      );
      
      for (const row of result.rows) {
        yield row;
      }
      
      hasMore = result.rows.length === this.chunkSize;
      offset += result.rows.length;
    }
    
    this.saveLastRunTime(currentRun);
  }
}

module.exports = { PostgresExtractor, IncrementalExtractor };
```

### 2.3 Transformers

```javascript
// pipeline/transformers/orderTransformer.js
class OrderTransformer {
  constructor(config = {}) {
    this.currencyRates = config.currencyRates || { THB: 1, USD: 35, EUR: 38 };
  }
  
  async transform(order) {
    // Skip cancelled orders
    if (order.status === 'CANCELLED' && !order.has_payment) {
      return null; // null = skip this record
    }
    
    return {
      // Flatten nested data
      order_id: order.id,
      order_number: order.order_number,
      customer_id: order.customer_id,
      customer_email: order.customer?.email,
      customer_tier: order.customer?.tier || 'STANDARD',
      
      // Normalize currency to THB
      amount_thb: this.convertToTHB(order.total_amount, order.currency),
      original_amount: order.total_amount,
      original_currency: order.currency,
      
      // Categorize order
      order_category: this.categorizeOrder(order),
      
      // Date dimensions
      order_date: order.created_at,
      order_year: new Date(order.created_at).getFullYear(),
      order_month: new Date(order.created_at).getMonth() + 1,
      order_quarter: Math.ceil((new Date(order.created_at).getMonth() + 1) / 3),
      order_day_of_week: new Date(order.created_at).getDay(),
      
      // Status
      status: order.status,
      is_completed: order.status === 'DELIVERED',
      days_to_deliver: this.calculateDeliveryDays(order),
      
      // Metadata
      etl_timestamp: new Date().toISOString(),
      source_system: 'orders_service',
    };
  }
  
  convertToTHB(amount, currency) {
    const rate = this.currencyRates[currency] || 1;
    return Math.round(amount * rate * 100) / 100;
  }
  
  categorizeOrder(order) {
    const amount = this.convertToTHB(order.total_amount, order.currency);
    if (amount >= 10000) return 'HIGH_VALUE';
    if (amount >= 1000) return 'MEDIUM_VALUE';
    return 'LOW_VALUE';
  }
  
  calculateDeliveryDays(order) {
    if (!order.delivered_at || !order.created_at) return null;
    const diff = new Date(order.delivered_at) - new Date(order.created_at);
    return Math.floor(diff / (1000 * 60 * 60 * 24));
  }
}

// pipeline/transformers/validationTransformer.js
class ValidationTransformer {
  constructor(rules) {
    this.rules = rules;
    this.rejectedCount = 0;
  }
  
  async transform(record) {
    const errors = [];
    
    for (const [field, rule] of Object.entries(this.rules)) {
      const value = record[field];
      
      if (rule.required && (value === null || value === undefined)) {
        errors.push(`Missing required field: ${field}`);
        continue;
      }
      
      if (value !== null && value !== undefined) {
        if (rule.type === 'number' && typeof value !== 'number') {
          errors.push(`Field ${field} must be a number, got ${typeof value}`);
        }
        if (rule.min !== undefined && value < rule.min) {
          errors.push(`Field ${field} must be >= ${rule.min}`);
        }
        if (rule.max !== undefined && value > rule.max) {
          errors.push(`Field ${field} must be <= ${rule.max}`);
        }
        if (rule.enum && !rule.enum.includes(value)) {
          errors.push(`Field ${field} must be one of: ${rule.enum.join(', ')}`);
        }
      }
    }
    
    if (errors.length > 0) {
      this.rejectedCount++;
      console.warn(`Validation failed for record ${record.id}:`, errors);
      return null; // Skip invalid records
    }
    
    return record;
  }
}

module.exports = { OrderTransformer, ValidationTransformer };
```

### 2.4 Loaders

```javascript
// pipeline/loaders/postgresLoader.js
const { Pool } = require('pg');

class PostgresLoader {
  constructor(config) {
    this.pool = new Pool(config.connection);
    this.table = config.table;
    this.conflictColumn = config.conflictColumn || 'id';
    this.updateOnConflict = config.updateOnConflict !== false;
  }
  
  async load(records) {
    if (records.length === 0) return;
    
    const columns = Object.keys(records[0]);
    const placeholders = records.map((_, rowIndex) => 
      `(${columns.map((_, colIndex) => `$${rowIndex * columns.length + colIndex + 1}`).join(', ')})`
    ).join(', ');
    
    const values = records.flatMap(record => columns.map(col => record[col]));
    
    let query;
    if (this.updateOnConflict) {
      const updateSet = columns
        .filter(col => col !== this.conflictColumn)
        .map(col => `${col} = EXCLUDED.${col}`)
        .join(', ');
      
      query = `
        INSERT INTO ${this.table} (${columns.join(', ')})
        VALUES ${placeholders}
        ON CONFLICT (${this.conflictColumn}) DO UPDATE SET ${updateSet}
      `;
    } else {
      query = `
        INSERT INTO ${this.table} (${columns.join(', ')})
        VALUES ${placeholders}
        ON CONFLICT (${this.conflictColumn}) DO NOTHING
      `;
    }
    
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');
      await client.query(query, values);
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}

// pipeline/loaders/elasticsearchLoader.js
const { Client } = require('@elastic/elasticsearch');

class ElasticsearchLoader {
  constructor(config) {
    this.client = new Client({ node: config.url || 'http://localhost:9200' });
    this.index = config.index;
    this.idField = config.idField || 'id';
  }
  
  async load(records) {
    if (records.length === 0) return;
    
    const operations = records.flatMap(doc => [
      { index: { _index: this.index, _id: doc[this.idField] } },
      doc,
    ]);
    
    const result = await this.client.bulk({ refresh: false, operations });
    
    if (result.errors) {
      const failed = result.items.filter(item => item.index?.error);
      console.error(`Elasticsearch bulk load errors: ${failed.length}/${records.length}`);
      
      for (const item of failed.slice(0, 5)) {
        console.error('ES error:', item.index.error);
      }
    }
    
    return result;
  }
}

module.exports = { PostgresLoader, ElasticsearchLoader };
```

## 3. Complete ETL Pipeline Example

### 3.1 Orders to Data Warehouse Pipeline

```javascript
// pipelines/ordersDataWarehouse.js
const ETLPipeline = require('../pipeline/etl');
const { IncrementalExtractor } = require('../pipeline/extractors/postgresExtractor');
const { OrderTransformer, ValidationTransformer } = require('../pipeline/transformers/orderTransformer');
const { PostgresLoader } = require('../pipeline/loaders/postgresLoader');
const { ElasticsearchLoader } = require('../pipeline/loaders/elasticsearchLoader');

async function runOrdersETL() {
  const pipeline = new ETLPipeline('orders-to-warehouse');
  
  // Extract: Incremental from source DB
  pipeline.extract(new IncrementalExtractor({
    connection: { connectionString: process.env.SOURCE_DB_URL },
    table: 'orders',
    timestampColumn: 'updated_at',
    lastRunFile: '/data/last_run_orders.txt',
  }));
  
  // Transform: Validate then normalize
  pipeline.transform(new ValidationTransformer({
    id: { required: true },
    customer_id: { required: true },
    total_amount: { required: true, type: 'number', min: 0 },
    status: { required: true, enum: ['DRAFT', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'] },
  }));
  
  pipeline.transform(new OrderTransformer({
    currencyRates: { THB: 1, USD: 35.5, EUR: 38.2 },
  }));
  
  // Load: Multiple destinations
  pipeline.load(new PostgresLoader({
    connection: { connectionString: process.env.WAREHOUSE_DB_URL },
    table: 'fact_orders',
    conflictColumn: 'order_id',
  }));
  
  pipeline.load(new ElasticsearchLoader({
    url: process.env.ELASTICSEARCH_URL,
    index: 'orders',
    idField: 'order_id',
  }));
  
  // Error handling
  pipeline.onError(async (error, record, stage) => {
    console.error(`ETL Error in ${stage} stage:`, error, { recordId: record?.id });
    
    // Save failed records for later reprocessing
    await db.query(
      'INSERT INTO etl_failed_records (table_name, record_id, error_message, failed_at) VALUES ($1, $2, $3, NOW())',
      ['orders', record?.id, error.message]
    );
  });
  
  const metrics = await pipeline.run({ batchSize: 500 });
  return metrics;
}

module.exports = { runOrdersETL };
```

## 4. Stream Processing with Kafka Streams (Node.js)

### 4.1 Real-time Aggregation Pipeline

```javascript
// streams/orderAggregator.js
const { Kafka } = require('kafkajs');
const Redis = require('ioredis');

class OrderStreamAggregator {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'order-aggregator',
      brokers: [process.env.KAFKA_BROKER],
    });
    this.redis = new Redis(process.env.REDIS_URL);
    this.consumer = this.kafka.consumer({ groupId: 'stream-aggregator' });
    this.producer = this.kafka.producer();
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
        await this.processEvent(event);
      },
    });
  }
  
  async processEvent(event) {
    const { type, payload, timestamp } = event;
    
    if (type === 'ORDER_COMPLETED') {
      await this.updateSalesAggregations(payload, timestamp);
    }
    
    if (type === 'ORDER_CREATED') {
      await this.updateFunnelMetrics(payload, timestamp);
    }
  }
  
  async updateSalesAggregations(order, timestamp) {
    const date = new Date(timestamp);
    const dateKey = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
    const hourKey = `${dateKey}T${String(date.getHours()).padStart(2, '0')}`;
    
    const pipeline = this.redis.pipeline();
    
    // Daily aggregation
    pipeline.hincrbyfloat(`sales:daily:${dateKey}`, 'revenue', order.total_amount);
    pipeline.hincrby(`sales:daily:${dateKey}`, 'count', 1);
    pipeline.expire(`sales:daily:${dateKey}`, 90 * 24 * 60 * 60); // 90 days TTL
    
    // Hourly aggregation  
    pipeline.hincrbyfloat(`sales:hourly:${hourKey}`, 'revenue', order.total_amount);
    pipeline.hincrby(`sales:hourly:${hourKey}`, 'count', 1);
    pipeline.expire(`sales:hourly:${hourKey}`, 7 * 24 * 60 * 60); // 7 days TTL
    
    // Customer stats
    pipeline.hincrbyfloat(`customer:stats:${order.customer_id}`, 'total_spent', order.total_amount);
    pipeline.hincrby(`customer:stats:${order.customer_id}`, 'order_count', 1);
    
    // Category aggregation
    for (const item of order.items) {
      pipeline.hincrbyfloat(`sales:category:${dateKey}:${item.category_id}`, 'revenue', item.subtotal);
      pipeline.hincrby(`sales:category:${dateKey}:${item.category_id}`, 'quantity', item.quantity);
    }
    
    await pipeline.exec();
    
    // Publish real-time dashboard event
    await this.producer.send({
      topic: 'dashboard.updates',
      messages: [{
        key: dateKey,
        value: JSON.stringify({
          type: 'SALES_UPDATED',
          date: dateKey,
          orderId: order.id,
          amount: order.total_amount,
          timestamp: new Date().toISOString(),
        }),
      }],
    });
  }
  
  async updateFunnelMetrics(order, timestamp) {
    const dateKey = new Date(timestamp).toISOString().split('T')[0];
    
    await this.redis.hincrby(`funnel:${dateKey}`, 'orders_created', 1);
    
    // Track source attribution
    if (order.utm_source) {
      await this.redis.hincrby(`funnel:source:${dateKey}:${order.utm_source}`, 'orders', 1);
    }
  }
  
  async getMetrics(dateKey) {
    const [daily, categories] = await Promise.all([
      this.redis.hgetall(`sales:daily:${dateKey}`),
      this.redis.keys(`sales:category:${dateKey}:*`),
    ]);
    
    const categoryData = await Promise.all(
      categories.map(async key => {
        const data = await this.redis.hgetall(key);
        return { category: key.split(':').pop(), ...data };
      })
    );
    
    return {
      date: dateKey,
      revenue: parseFloat(daily?.revenue || 0),
      orderCount: parseInt(daily?.count || 0),
      avgOrderValue: daily?.count > 0 
        ? parseFloat(daily.revenue) / parseInt(daily.count) 
        : 0,
      byCategory: categoryData,
    };
  }
}

module.exports = OrderStreamAggregator;
```

## 5. Data Quality Checks

### 5.1 Data Quality Framework

```javascript
// pipeline/dataQuality.js
class DataQualityChecker {
  constructor(rules) {
    this.rules = rules;
    this.results = [];
  }
  
  async check(dataset, options = {}) {
    const { sampleSize = 1000, failOnError = false } = options;
    
    for (const rule of this.rules) {
      const startTime = Date.now();
      
      try {
        const result = await rule.check(dataset);
        const passed = result.failureCount === 0 || result.failureRate < (rule.threshold || 0);
        
        this.results.push({
          rule: rule.name,
          passed,
          total: result.totalCount,
          failures: result.failureCount,
          failureRate: result.failureRate,
          examples: result.failureExamples?.slice(0, 3),
          duration: Date.now() - startTime,
        });
        
        if (!passed && failOnError) {
          throw new Error(`Data quality check failed: ${rule.name} (${(result.failureRate * 100).toFixed(1)}% failures)`);
        }
        
      } catch (error) {
        if (error.message.includes('Data quality check failed')) throw error;
        
        this.results.push({
          rule: rule.name,
          passed: false,
          error: error.message,
          duration: Date.now() - startTime,
        });
      }
    }
    
    return this.getReport();
  }
  
  getReport() {
    const passed = this.results.filter(r => r.passed).length;
    const failed = this.results.filter(r => !r.passed).length;
    
    return {
      summary: {
        total: this.results.length,
        passed,
        failed,
        score: (passed / this.results.length * 100).toFixed(1) + '%',
      },
      details: this.results,
    };
  }
}

// Data quality rules
const nullCheckRule = (field, threshold = 0) => ({
  name: `null_check_${field}`,
  threshold,
  async check(records) {
    const nullCount = records.filter(r => r[field] === null || r[field] === undefined).length;
    return {
      totalCount: records.length,
      failureCount: nullCount,
      failureRate: nullCount / records.length,
      failureExamples: records.filter(r => !r[field]).slice(0, 5).map(r => r.id),
    };
  }
});

const rangeCheckRule = (field, min, max, threshold = 0) => ({
  name: `range_check_${field}`,
  threshold,
  async check(records) {
    const outOfRange = records.filter(r => {
      const v = r[field];
      return v !== null && v !== undefined && (v < min || v > max);
    });
    return {
      totalCount: records.length,
      failureCount: outOfRange.length,
      failureRate: outOfRange.length / records.length,
      failureExamples: outOfRange.slice(0, 5).map(r => ({ id: r.id, value: r[field] })),
    };
  }
});

const uniquenessRule = (field, threshold = 0) => ({
  name: `uniqueness_${field}`,
  threshold,
  async check(records) {
    const values = records.map(r => r[field]).filter(v => v !== null);
    const uniqueValues = new Set(values);
    const duplicates = values.length - uniqueValues.size;
    return {
      totalCount: records.length,
      failureCount: duplicates,
      failureRate: duplicates / records.length,
    };
  }
});

const referentialIntegrityRule = (field, referencedIds) => ({
  name: `referential_integrity_${field}`,
  async check(records) {
    const idSet = new Set(referencedIds);
    const orphans = records.filter(r => r[field] && !idSet.has(r[field]));
    return {
      totalCount: records.length,
      failureCount: orphans.length,
      failureRate: orphans.length / records.length,
      failureExamples: orphans.slice(0, 5).map(r => ({ id: r.id, [field]: r[field] })),
    };
  }
});

// Usage
async function validateOrderData(orders) {
  const checker = new DataQualityChecker([
    nullCheckRule('id'),
    nullCheckRule('customer_id'),
    nullCheckRule('total_amount'),
    rangeCheckRule('total_amount', 0, 10000000), // 0 - 10M THB
    uniquenessRule('id'),
    {
      name: 'status_validity',
      threshold: 0,
      async check(records) {
        const validStatuses = ['DRAFT', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'];
        const invalid = records.filter(r => !validStatuses.includes(r.status));
        return {
          totalCount: records.length,
          failureCount: invalid.length,
          failureRate: invalid.length / records.length,
          failureExamples: invalid.slice(0, 5).map(r => ({ id: r.id, status: r.status })),
        };
      }
    },
  ]);
  
  const report = await checker.check(orders);
  console.log('Data Quality Report:', JSON.stringify(report, null, 2));
  
  return report;
}

module.exports = { DataQualityChecker, nullCheckRule, rangeCheckRule, uniquenessRule };
```

## 6. Pipeline Monitoring

### 6.1 Pipeline Metrics Exporter

```javascript
// monitoring/pipelineMetrics.js
const prometheus = require('prom-client');

class PipelineMetricsCollector {
  constructor() {
    this.register = new prometheus.Registry();
    
    this.pipelineRuns = new prometheus.Counter({
      name: 'etl_pipeline_runs_total',
      help: 'Total number of ETL pipeline runs',
      labelNames: ['pipeline_name', 'status'],
      registers: [this.register],
    });
    
    this.recordsProcessed = new prometheus.Counter({
      name: 'etl_records_processed_total',
      help: 'Total records processed',
      labelNames: ['pipeline_name', 'stage'],
      registers: [this.register],
    });
    
    this.processingDuration = new prometheus.Histogram({
      name: 'etl_pipeline_duration_seconds',
      help: 'ETL pipeline duration in seconds',
      labelNames: ['pipeline_name'],
      buckets: [10, 30, 60, 120, 300, 600, 1800, 3600],
      registers: [this.register],
    });
    
    this.errorRate = new prometheus.Gauge({
      name: 'etl_error_rate',
      help: 'Current error rate for ETL pipeline',
      labelNames: ['pipeline_name'],
      registers: [this.register],
    });
    
    this.dataQualityScore = new prometheus.Gauge({
      name: 'etl_data_quality_score',
      help: 'Data quality score (0-100)',
      labelNames: ['pipeline_name', 'rule'],
      registers: [this.register],
    });
    
    this.lagSeconds = new prometheus.Gauge({
      name: 'etl_pipeline_lag_seconds',
      help: 'How far behind real-time the pipeline is',
      labelNames: ['pipeline_name'],
      registers: [this.register],
    });
  }
  
  recordPipelineRun(pipelineName, status, metrics) {
    this.pipelineRuns.inc({ pipeline_name: pipelineName, status });
    
    if (metrics.extracted) {
      this.recordsProcessed.inc(
        { pipeline_name: pipelineName, stage: 'extract' },
        metrics.extracted
      );
    }
    
    if (metrics.loaded) {
      this.recordsProcessed.inc(
        { pipeline_name: pipelineName, stage: 'load' },
        metrics.loaded
      );
    }
    
    if (metrics.errors && metrics.extracted > 0) {
      this.errorRate.set(
        { pipeline_name: pipelineName },
        metrics.errors / metrics.extracted
      );
    }
    
    if (metrics.startTime && metrics.endTime) {
      this.processingDuration.observe(
        { pipeline_name: pipelineName },
        (metrics.endTime - metrics.startTime) / 1000
      );
    }
  }
  
  recordDataQuality(pipelineName, report) {
    for (const detail of report.details) {
      const score = detail.passed ? 100 : (1 - (detail.failureRate || 1)) * 100;
      this.dataQualityScore.set(
        { pipeline_name: pipelineName, rule: detail.rule },
        score
      );
    }
  }
  
  async getMetrics() {
    return this.register.metrics();
  }
}

module.exports = new PipelineMetricsCollector();
```

## 7. Scheduled Pipeline with Cron

```javascript
// scheduler/pipelineScheduler.js
const cron = require('node-cron');
const { runOrdersETL } = require('../pipelines/ordersDataWarehouse');
const metrics = require('../monitoring/pipelineMetrics');

class PipelineScheduler {
  constructor() {
    this.jobs = new Map();
    this.runningJobs = new Set();
  }
  
  schedule(name, cronExpression, pipeline, options = {}) {
    if (this.jobs.has(name)) {
      throw new Error(`Pipeline ${name} already scheduled`);
    }
    
    const job = cron.schedule(cronExpression, async () => {
      if (this.runningJobs.has(name)) {
        console.warn(`Pipeline ${name} is already running, skipping this run`);
        return;
      }
      
      this.runningJobs.add(name);
      console.log(`Starting scheduled pipeline: ${name}`);
      
      try {
        const result = await pipeline();
        metrics.recordPipelineRun(name, 'success', result);
        console.log(`Pipeline ${name} completed successfully`, result);
      } catch (error) {
        metrics.recordPipelineRun(name, 'failed', {});
        console.error(`Pipeline ${name} failed:`, error);
        
        if (options.alertOnFailure) {
          await this.sendAlert(name, error);
        }
      } finally {
        this.runningJobs.delete(name);
      }
    }, {
      scheduled: false,
      timezone: options.timezone || 'Asia/Bangkok',
    });
    
    this.jobs.set(name, job);
    return this;
  }
  
  start(name) {
    const job = this.jobs.get(name);
    if (!job) throw new Error(`Pipeline ${name} not found`);
    job.start();
    console.log(`Started pipeline scheduler: ${name}`);
    return this;
  }
  
  startAll() {
    for (const [name, job] of this.jobs) {
      job.start();
      console.log(`Started pipeline scheduler: ${name}`);
    }
    return this;
  }
  
  stop(name) {
    const job = this.jobs.get(name);
    if (job) job.stop();
    return this;
  }
  
  async sendAlert(pipelineName, error) {
    // Implement Slack/PagerDuty notification
    console.error(`ALERT: Pipeline ${pipelineName} failed: ${error.message}`);
  }
}

// Setup schedules
const scheduler = new PipelineScheduler();

// Orders ETL every 15 minutes
scheduler.schedule(
  'orders-etl',
  '*/15 * * * *',
  runOrdersETL,
  { alertOnFailure: true, timezone: 'Asia/Bangkok' }
);

// Daily full sync at 2 AM
scheduler.schedule(
  'orders-full-sync',
  '0 2 * * *',
  () => runOrdersETL({ fullSync: true }),
  { alertOnFailure: true }
);

scheduler.startAll();

module.exports = scheduler;
```

## 8. Dead Letter Queue Reprocessing

```javascript
// pipeline/dlqReprocessor.js
const { Kafka } = require('kafkajs');

class DLQReprocessor {
  constructor() {
    this.kafka = new Kafka({
      clientId: 'dlq-reprocessor',
      brokers: [process.env.KAFKA_BROKER],
    });
  }
  
  async reprocess(dlqTopic, options = {}) {
    const { maxRetries = 3, batchSize = 10, dryRun = false } = options;
    const consumer = this.kafka.consumer({ groupId: 'dlq-reprocessor' });
    const producer = this.kafka.producer();
    
    await consumer.connect();
    await producer.connect();
    
    await consumer.subscribe({ topic: dlqTopic, fromBeginning: true });
    
    const processed = [];
    const failed = [];
    
    await consumer.run({
      eachMessage: async ({ message }) => {
        const originalTopic = message.headers['original-topic']?.toString();
        const failedAt = message.headers['failed-at']?.toString();
        const errorMessage = message.headers['error-message']?.toString();
        
        const failedAtDate = new Date(failedAt);
        const ageHours = (Date.now() - failedAtDate) / (1000 * 60 * 60);
        
        // Skip if too old (> 24 hours)
        if (ageHours > 24) {
          console.log(`Skipping old DLQ message (${ageHours.toFixed(1)}h old): ${originalTopic}`);
          return;
        }
        
        console.log(`Reprocessing DLQ message: topic=${originalTopic} error=${errorMessage}`);
        
        if (!dryRun) {
          try {
            await producer.send({
              topic: originalTopic,
              messages: [{
                key: message.key,
                value: message.value,
                headers: {
                  'retry-count': String(parseInt(message.headers['retry-count'] || '0') + 1),
                  'original-error': errorMessage,
                  'reprocessed-at': new Date().toISOString(),
                },
              }],
            });
            processed.push({ originalTopic, messageKey: message.key?.toString() });
          } catch (error) {
            failed.push({ originalTopic, error: error.message });
          }
        }
      },
    });
    
    await consumer.disconnect();
    await producer.disconnect();
    
    return { processed: processed.length, failed: failed.length, dryRun };
  }
}

module.exports = DLQReprocessor;
```

## 9. Pipeline Docker Compose

```yaml
# docker-compose.pipeline.yml
version: '3.8'
services:
  etl-service:
    build: .
    environment:
      SOURCE_DB_URL: postgresql://user:pass@postgres:5432/orders_db
      WAREHOUSE_DB_URL: postgresql://user:pass@warehouse-db:5432/warehouse
      KAFKA_BROKER: kafka:29092
      REDIS_URL: redis://redis:6379
      ELASTICSEARCH_URL: http://elasticsearch:9200
    volumes:
      - etl-state:/data  # Store last run timestamps
    depends_on:
      - postgres
      - kafka
      - redis
      - elasticsearch
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 30s
      timeout: 10s
      retries: 3
  
  pipeline-monitor:
    build:
      context: .
      dockerfile: Dockerfile.monitor
    ports:
      - "9090:9090"  # Prometheus metrics endpoint
    environment:
      METRICS_PORT: 9090
    depends_on:
      - etl-service

volumes:
  etl-state:
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CDC with Debezium** - Capture changes จาก PostgreSQL WAL logs โดยไม่ต้องแก้ application code
2. **ETL Framework** - Composable pipeline ด้วย Extractor/Transformer/Loader pattern
3. **Incremental Extraction** - ดึงเฉพาะข้อมูลที่เปลี่ยนแปลงตั้งแต่ครั้งล่าสุด
4. **Stream Processing** - Real-time aggregation ด้วย Kafka consumer + Redis
5. **Data Quality** - Automated checks สำหรับ null, range, uniqueness, referential integrity
6. **Dead Letter Queue** - Handle failed records และ retry mechanism
7. **Pipeline Monitoring** - Prometheus metrics สำหรับ tracking pipeline health
8. **Scheduling** - Cron-based pipeline scheduling ด้วย concurrency protection

## แบบฝึกหัด

1. สร้าง CDC pipeline จาก MySQL ไปยัง Elasticsearch สำหรับ product search
2. Implement ETL pipeline ที่ดึงข้อมูล API, transform, และ load ไปยัง PostgreSQL
3. สร้าง real-time dashboard ด้วย Kafka Streams ที่แสดง top products
4. เพิ่ม data lineage tracking ให้กับ ETL pipeline
5. Implement DLQ monitoring และ alerting เมื่อ failure rate สูงกว่า threshold

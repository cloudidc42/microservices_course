# Part 63: Microservices Migration Strategies

## บทนำ

การ Migrate จาก Monolith ไปสู่ Microservices เป็นกระบวนการที่ต้องทำอย่างระมัดระวัง ไม่สามารถทำได้ในครั้งเดียว บทนี้จะครอบคลุม Patterns ที่ใช้จริงในองค์กรต่างๆ เช่น Strangler Fig Pattern, Anti-Corruption Layer, Branch by Abstraction, Parallel Run Pattern และกลยุทธ์การ Migrate ฐานข้อมูล

## 1. Strangler Fig Pattern

Strangler Fig Pattern ค่อยๆ แทนที่ Monolith ทีละส่วน โดยเริ่มจาก Feature ที่เป็น High Value หรือ High Change Rate

### 1.1 API Gateway เป็น Traffic Router

```typescript
// src/gateway/strangler-fig-proxy.ts
import express, { Request, Response } from 'express';
import { createProxyMiddleware, Options } from 'http-proxy-middleware';
import { FeatureFlagService } from '../feature-flags/feature-flag-service';
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';

interface RouteConfig {
  path: string;
  method?: string;
  target: 'monolith' | 'microservice';
  microserviceUrl?: string;
  featureFlag?: string;
  trafficPercentage?: number;  // 0-100 สำหรับ Progressive Migration
}

export class StranglerFigProxy {
  private routes: RouteConfig[] = [];

  constructor(
    private monolithUrl: string,
    private featureFlags: FeatureFlagService,
    private metrics: MetricsCollector
  ) {}

  addRoute(config: RouteConfig): void {
    this.routes.push(config);
  }

  createMiddleware() {
    return async (req: Request, res: Response, next: express.NextFunction) => {
      const route = this.findMatchingRoute(req);

      if (!route) {
        return next();
      }

      const target = await this.determineTarget(req, route);
      const targetUrl = target === 'microservice' && route.microserviceUrl
        ? route.microserviceUrl
        : this.monolithUrl;

      logger.info('Routing request', {
        path: req.path,
        method: req.method,
        target,
        targetUrl,
      });

      this.metrics.increment('migration.request', {
        path: route.path,
        target,
        method: req.method,
      });

      createProxyMiddleware({
        target: targetUrl,
        changeOrigin: true,
        pathRewrite: target === 'microservice' ? this.getPathRewrite(route) : undefined,
        on: {
          error: (err, req, res) => {
            logger.error('Proxy error', { err, path: req.url });
            this.metrics.increment('migration.proxy_error', { target });
          },
        },
      })(req, res, next);
    };
  }

  private findMatchingRoute(req: Request): RouteConfig | null {
    return this.routes.find(route => {
      const pathMatch = req.path.startsWith(route.path);
      const methodMatch = !route.method || route.method === req.method;
      return pathMatch && methodMatch;
    }) ?? null;
  }

  private async determineTarget(
    req: Request,
    route: RouteConfig
  ): Promise<'monolith' | 'microservice'> {
    // ถ้าไม่มี Microservice URL ยังคงส่งไป Monolith
    if (!route.microserviceUrl) {
      return 'monolith';
    }

    // ตรวจสอบ Feature Flag
    if (route.featureFlag) {
      const userId = (req as any).user?.id ?? req.ip ?? 'anonymous';
      const flagEnabled = await this.featureFlags.isEnabled(
        route.featureFlag,
        { userId }
      );

      if (!flagEnabled) {
        return 'monolith';
      }
    }

    // Traffic Percentage Splitting
    if (route.trafficPercentage !== undefined && route.trafficPercentage < 100) {
      const random = Math.random() * 100;
      return random < route.trafficPercentage ? 'microservice' : 'monolith';
    }

    return route.target;
  }

  private getPathRewrite(route: RouteConfig): Record<string, string> | undefined {
    if (!route.microserviceUrl) return undefined;
    // เช่น /api/orders/* -> /* ใน Order Microservice
    return { [`^${route.path}`]: '' };
  }
}

// ตัวอย่าง Migration Plan สำหรับ Thai E-commerce Monolith
function setupStranglerFig(
  featureFlags: FeatureFlagService,
  metrics: MetricsCollector
): StranglerFigProxy {
  const proxy = new StranglerFigProxy(
    'http://monolith.internal:8080',
    featureFlags,
    metrics
  );

  // Phase 1: Product Catalog (Low Risk, High Value)
  proxy.addRoute({
    path: '/api/products',
    target: 'microservice',
    microserviceUrl: 'http://product-service.internal:3001',
    featureFlag: 'use_product_microservice',
    trafficPercentage: 100, // Migration เสร็จแล้ว
  });

  // Phase 2: User Profiles (กำลัง Migrate อยู่)
  proxy.addRoute({
    path: '/api/users',
    target: 'microservice',
    microserviceUrl: 'http://user-service.internal:3002',
    featureFlag: 'use_user_microservice',
    trafficPercentage: 50, // 50% Traffic ไปที่ Microservice
  });

  // Phase 3: Orders (เริ่ม Migrate)
  proxy.addRoute({
    path: '/api/orders',
    target: 'microservice',
    microserviceUrl: 'http://order-service.internal:3003',
    featureFlag: 'use_order_microservice',
    trafficPercentage: 10, // 10% Traffic ไปทดสอบ
  });

  // Phase 4: Payment (ยังอยู่ใน Monolith)
  proxy.addRoute({
    path: '/api/payments',
    target: 'monolith',
    // ยังไม่ Migrate เนื่องจาก High Risk
  });

  return proxy;
}
```

## 2. Anti-Corruption Layer (ACL)

ACL แปลง Interface ของ Monolith เก่าให้เข้ากับ Domain Model ใหม่

### 2.1 Anti-Corruption Layer Implementation

```typescript
// src/acl/monolith-adapter.ts

// Domain Models (ใหม่)
interface NewOrderModel {
  id: string;
  customerId: string;
  items: Array<{
    productId: string;
    quantity: number;
    unitPrice: number;
    subtotal: number;
  }>;
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  totalAmount: number;
  currency: string;
  shippingAddress: Address;
  createdAt: Date;
  updatedAt: Date;
}

interface Address {
  street: string;
  district: string;
  province: string;
  postalCode: string;
  country: string;
}

// Legacy Monolith Models (เก่า)
interface LegacyOrder {
  order_id: string;
  cust_id: string;
  order_date: string;
  total_price: number;
  status_code: number; // 1=pending, 2=confirmed, 3=shipped, 4=delivered, 5=cancelled
  items: string; // JSON string ของ items
  ship_addr: string;
  ship_city: string;
  ship_zip: string;
}

// Anti-Corruption Layer
export class OrderAntiCorruptionLayer {
  private statusMap: Record<number, NewOrderModel['status']> = {
    1: 'pending',
    2: 'confirmed',
    3: 'shipped',
    4: 'delivered',
    5: 'cancelled',
  };

  private reverseStatusMap: Record<string, number> = {
    pending: 1,
    confirmed: 2,
    shipped: 3,
    delivered: 4,
    cancelled: 5,
  };

  translateToNewModel(legacy: LegacyOrder): NewOrderModel {
    let items: any[] = [];
    
    try {
      const rawItems = JSON.parse(legacy.items);
      items = rawItems.map((item: any) => ({
        productId: item.prod_id,
        quantity: parseInt(item.qty),
        unitPrice: parseFloat(item.price),
        subtotal: parseInt(item.qty) * parseFloat(item.price),
      }));
    } catch (error) {
      items = [];
    }

    // แปลง Address จาก Legacy Format
    const addressParts = legacy.ship_addr?.split(',') ?? [];
    const address: Address = {
      street: addressParts[0]?.trim() ?? '',
      district: addressParts[1]?.trim() ?? '',
      province: legacy.ship_city ?? '',
      postalCode: legacy.ship_zip ?? '',
      country: 'TH',
    };

    return {
      id: legacy.order_id,
      customerId: legacy.cust_id,
      items,
      status: this.statusMap[legacy.status_code] ?? 'pending',
      totalAmount: legacy.total_price,
      currency: 'THB',
      shippingAddress: address,
      createdAt: new Date(legacy.order_date),
      updatedAt: new Date(legacy.order_date),
    };
  }

  translateToLegacyModel(order: NewOrderModel): Partial<LegacyOrder> {
    return {
      order_id: order.id,
      cust_id: order.customerId,
      order_date: order.createdAt.toISOString(),
      total_price: order.totalAmount,
      status_code: this.reverseStatusMap[order.status] ?? 1,
      items: JSON.stringify(
        order.items.map(item => ({
          prod_id: item.productId,
          qty: item.quantity.toString(),
          price: item.unitPrice.toString(),
        }))
      ),
      ship_addr: `${order.shippingAddress.street}, ${order.shippingAddress.district}`,
      ship_city: order.shippingAddress.province,
      ship_zip: order.shippingAddress.postalCode,
    };
  }
}

// ACL Service ที่ Wrap Legacy API
export class LegacyOrderServiceACL {
  private acl: OrderAntiCorruptionLayer;

  constructor(
    private legacyApiUrl: string,
    private httpClient: any
  ) {
    this.acl = new OrderAntiCorruptionLayer();
  }

  async getOrder(orderId: string): Promise<NewOrderModel> {
    // เรียก Legacy API
    const response = await this.httpClient.get(
      `${this.legacyApiUrl}/orders/${orderId}`
    );
    
    const legacyOrder: LegacyOrder = response.data;
    
    // แปลงเป็น New Domain Model
    return this.acl.translateToNewModel(legacyOrder);
  }

  async createOrder(order: Omit<NewOrderModel, 'id' | 'createdAt' | 'updatedAt'>): Promise<NewOrderModel> {
    // แปลงเป็น Legacy Format
    const legacyData = this.acl.translateToLegacyModel({
      ...order,
      id: '',
      createdAt: new Date(),
      updatedAt: new Date(),
    });

    // ส่งไป Legacy API
    const response = await this.httpClient.post(
      `${this.legacyApiUrl}/orders`,
      legacyData
    );
    
    return this.acl.translateToNewModel(response.data);
  }

  async updateOrderStatus(
    orderId: string,
    status: NewOrderModel['status']
  ): Promise<void> {
    const legacyStatusCode = this.acl['reverseStatusMap'][status];
    
    await this.httpClient.patch(
      `${this.legacyApiUrl}/orders/${orderId}`,
      { status_code: legacyStatusCode }
    );
  }
}
```

## 3. Branch by Abstraction

Branch by Abstraction ช่วยให้ Refactor Code ใน Monolith ก่อน Extract ออกเป็น Microservice

### 3.1 Abstract Interface

```typescript
// src/abstractions/notification-service.ts

// Step 1: สร้าง Abstract Interface
export interface NotificationService {
  sendSMS(phoneNumber: string, message: string): Promise<void>;
  sendEmail(email: string, subject: string, body: string): Promise<void>;
  sendPushNotification(userId: string, title: string, body: string): Promise<void>;
  sendLineNotification(lineUserId: string, message: string): Promise<void>;
}

// Step 2: Legacy Implementation (Monolith Code)
export class LegacyNotificationService implements NotificationService {
  async sendSMS(phoneNumber: string, message: string): Promise<void> {
    // Legacy SMS implementation
    console.log(`Legacy SMS to ${phoneNumber}: ${message}`);
    // await legacyDb.insertNotification(...)
  }

  async sendEmail(email: string, subject: string, body: string): Promise<void> {
    // Legacy Email implementation
    console.log(`Legacy Email to ${email}: ${subject}`);
  }

  async sendPushNotification(userId: string, title: string, body: string): Promise<void> {
    // Legacy Push implementation
    console.log(`Legacy Push to ${userId}: ${title}`);
  }

  async sendLineNotification(lineUserId: string, message: string): Promise<void> {
    // Legacy Line implementation  
    console.log(`Legacy Line to ${lineUserId}: ${message}`);
  }
}

// Step 3: New Microservice Implementation
export class NotificationMicroserviceClient implements NotificationService {
  constructor(
    private baseUrl: string,
    private apiKey: string
  ) {}

  async sendSMS(phoneNumber: string, message: string): Promise<void> {
    await fetch(`${this.baseUrl}/sms`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
      },
      body: JSON.stringify({ phoneNumber, message }),
    });
  }

  async sendEmail(email: string, subject: string, body: string): Promise<void> {
    await fetch(`${this.baseUrl}/email`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
      },
      body: JSON.stringify({ email, subject, body }),
    });
  }

  async sendPushNotification(userId: string, title: string, body: string): Promise<void> {
    await fetch(`${this.baseUrl}/push`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
      },
      body: JSON.stringify({ userId, title, body }),
    });
  }

  async sendLineNotification(lineUserId: string, message: string): Promise<void> {
    await fetch(`${this.baseUrl}/line`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'X-API-Key': this.apiKey,
      },
      body: JSON.stringify({ lineUserId, message }),
    });
  }
}

// Step 4: Factory ที่เลือก Implementation ตาม Feature Flag
export class NotificationServiceFactory {
  static create(
    featureFlags: any,
    config: {
      microserviceUrl?: string;
      apiKey?: string;
    }
  ): NotificationService {
    const useMicroservice = process.env.USE_NOTIFICATION_MICROSERVICE === 'true';
    
    if (useMicroservice && config.microserviceUrl && config.apiKey) {
      return new NotificationMicroserviceClient(
        config.microserviceUrl,
        config.apiKey
      );
    }
    
    return new LegacyNotificationService();
  }
}
```

## 4. Parallel Run Pattern

Parallel Run เรียกทั้ง Old และ New Implementation พร้อมกัน แล้วเปรียบเทียบผลลัพธ์

### 4.1 Parallel Run Framework

```typescript
// src/migration/parallel-run.ts
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';

interface ParallelRunResult<T> {
  primaryResult: T;
  secondaryResult?: T;
  matched: boolean;
  primaryDurationMs: number;
  secondaryDurationMs?: number;
  discrepancies?: string[];
}

interface ParallelRunConfig {
  enabled: boolean;
  recordDiscrepancies: boolean;
  alertOnDiscrepancy: boolean;
  secondaryTimeoutMs: number;
}

export class ParallelRunner<T> {
  constructor(
    private name: string,
    private primaryFn: () => Promise<T>,
    private secondaryFn: () => Promise<T>,
    private compareFn: (primary: T, secondary: T) => { matched: boolean; discrepancies?: string[] },
    private config: ParallelRunConfig,
    private metrics: MetricsCollector
  ) {}

  async run(): Promise<T> {
    const primaryStart = Date.now();
    let primaryResult: T;
    
    try {
      primaryResult = await this.primaryFn();
    } catch (error) {
      this.metrics.increment('parallel_run.primary_error', { name: this.name });
      throw error;
    }
    
    const primaryDurationMs = Date.now() - primaryStart;
    
    // ถ้า Parallel Run ไม่เปิดใช้งาน Return Primary Result ทันที
    if (!this.config.enabled) {
      return primaryResult;
    }

    // รัน Secondary ใน Background (ไม่ Block Primary)
    this.runSecondary(primaryResult, primaryDurationMs).catch(error => {
      logger.error('Parallel run secondary error', { name: this.name, error });
    });

    return primaryResult;
  }

  private async runSecondary(
    primaryResult: T,
    primaryDurationMs: number
  ): Promise<void> {
    const secondaryStart = Date.now();
    
    try {
      const secondaryResult = await Promise.race([
        this.secondaryFn(),
        new Promise<never>((_, reject) =>
          setTimeout(
            () => reject(new Error('Secondary timeout')),
            this.config.secondaryTimeoutMs
          )
        ),
      ]);
      
      const secondaryDurationMs = Date.now() - secondaryStart;
      
      const comparison = this.compareFn(primaryResult, secondaryResult);
      
      const result: ParallelRunResult<T> = {
        primaryResult,
        secondaryResult,
        matched: comparison.matched,
        primaryDurationMs,
        secondaryDurationMs,
        discrepancies: comparison.discrepancies,
      };

      this.metrics.increment('parallel_run.comparison', {
        name: this.name,
        matched: comparison.matched.toString(),
      });

      this.metrics.histogram('parallel_run.latency_diff', 
        secondaryDurationMs - primaryDurationMs,
        { name: this.name }
      );

      if (!comparison.matched) {
        if (this.config.recordDiscrepancies) {
          logger.warn('Parallel run discrepancy detected', {
            name: this.name,
            discrepancies: comparison.discrepancies,
          });
        }

        if (this.config.alertOnDiscrepancy) {
          // ส่ง Alert
          logger.error('CRITICAL: Parallel run discrepancy', {
            name: this.name,
            discrepancies: comparison.discrepancies,
          });
        }
      } else {
        logger.debug('Parallel run matched', {
          name: this.name,
          primaryDurationMs,
          secondaryDurationMs,
        });
      }
      
    } catch (error) {
      this.metrics.increment('parallel_run.secondary_error', { name: this.name });
      logger.error('Secondary function failed in parallel run', {
        name: this.name,
        error,
      });
    }
  }
}

// ตัวอย่าง: Parallel Run สำหรับ Order Price Calculation
interface OrderTotal {
  subtotal: number;
  discount: number;
  shipping: number;
  tax: number;
  total: number;
}

class OrderPricingParallelRun {
  private runner: ParallelRunner<OrderTotal>;

  constructor(
    private legacyPricingService: any,
    private newPricingService: any,
    metrics: MetricsCollector
  ) {
    this.runner = new ParallelRunner(
      'order_pricing',
      () => this.legacyPricingService.calculate(),
      () => this.newPricingService.calculate(),
      (primary, secondary) => {
        const discrepancies: string[] = [];
        
        const tolerance = 0.01; // 1 สตางค์ tolerance
        
        if (Math.abs(primary.subtotal - secondary.subtotal) > tolerance) {
          discrepancies.push(
            `Subtotal mismatch: ${primary.subtotal} vs ${secondary.subtotal}`
          );
        }
        
        if (Math.abs(primary.total - secondary.total) > tolerance) {
          discrepancies.push(
            `Total mismatch: ${primary.total} vs ${secondary.total}`
          );
        }
        
        return {
          matched: discrepancies.length === 0,
          discrepancies,
        };
      },
      {
        enabled: true,
        recordDiscrepancies: true,
        alertOnDiscrepancy: true,
        secondaryTimeoutMs: 5000,
      },
      metrics
    );
  }

  async calculateOrderTotal(orderId: string): Promise<OrderTotal> {
    return this.runner.run();
  }
}
```

## 5. Database Decomposition Strategies

### 5.1 Dual-Write Pattern สำหรับ Database Migration

```typescript
// src/migration/dual-write.ts
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';

interface WriteResult {
  success: boolean;
  error?: Error;
  durationMs: number;
}

export class DualWriteService<T> {
  constructor(
    private primaryWriter: (data: T) => Promise<void>,
    private secondaryWriter: (data: T) => Promise<void>,
    private metrics: MetricsCollector,
    private options: {
      requireBothSuccess: boolean;
      logDiscrepancies: boolean;
    } = { requireBothSuccess: false, logDiscrepancies: true }
  ) {}

  async write(data: T): Promise<void> {
    const [primaryResult, secondaryResult] = await Promise.allSettled([
      this.writeWithMetrics('primary', () => this.primaryWriter(data)),
      this.writeWithMetrics('secondary', () => this.secondaryWriter(data)),
    ]);

    const primarySuccess = primaryResult.status === 'fulfilled';
    const secondarySuccess = secondaryResult.status === 'fulfilled';

    this.metrics.increment('dual_write.result', {
      primary: primarySuccess ? 'success' : 'failure',
      secondary: secondarySuccess ? 'success' : 'failure',
    });

    if (!primarySuccess) {
      const error = (primaryResult as PromiseRejectedResult).reason;
      logger.error('Primary write failed', { error });
      throw error;
    }

    if (!secondarySuccess) {
      const error = (secondaryResult as PromiseRejectedResult).reason;
      
      if (this.options.requireBothSuccess) {
        logger.error('Secondary write failed (required)', { error });
        // TODO: Rollback primary write
        throw error;
      }
      
      logger.warn('Secondary write failed (non-critical)', { error });
    }
  }

  private async writeWithMetrics(
    target: string,
    fn: () => Promise<void>
  ): Promise<WriteResult> {
    const start = Date.now();
    
    try {
      await fn();
      return { success: true, durationMs: Date.now() - start };
    } catch (error) {
      return {
        success: false,
        error: error as Error,
        durationMs: Date.now() - start,
      };
    }
  }
}

// ตัวอย่าง: Migrate Customer Data จาก MySQL Monolith ไปยัง PostgreSQL Microservice
class CustomerDataMigration {
  private dualWrite: DualWriteService<Customer>;

  constructor(
    private legacyDb: any,
    private newDb: any,
    metrics: MetricsCollector
  ) {
    this.dualWrite = new DualWriteService<Customer>(
      // Primary: Legacy MySQL
      async (customer) => {
        await legacyDb.query(
          'INSERT INTO customers (id, name, email, phone, created_at) VALUES (?, ?, ?, ?, ?)',
          [customer.id, customer.name, customer.email, customer.phone, customer.createdAt]
        );
      },
      // Secondary: New PostgreSQL Microservice
      async (customer) => {
        await newDb.query(
          `INSERT INTO customers (id, full_name, email_address, phone_number, created_at)
           VALUES ($1, $2, $3, $4, $5)
           ON CONFLICT (id) DO UPDATE SET
             full_name = EXCLUDED.full_name,
             email_address = EXCLUDED.email_address,
             phone_number = EXCLUDED.phone_number`,
          [customer.id, customer.name, customer.email, customer.phone, customer.createdAt]
        );
      },
      metrics,
      {
        requireBothSuccess: false, // ถ้า Secondary ล้มเหลว ยังคง Continue
        logDiscrepancies: true,
      }
    );
  }

  async createCustomer(customer: Customer): Promise<Customer> {
    await this.dualWrite.write(customer);
    return customer;
  }
}

interface Customer {
  id: string;
  name: string;
  email: string;
  phone: string;
  createdAt: Date;
}
```

## 6. Data Migration Pipeline

### 6.1 Incremental Data Migration

```typescript
// src/migration/data-migration-pipeline.ts
import { Pool } from 'pg';
import { logger } from '../utils/logger';
import { MetricsCollector } from '../monitoring/metrics';

interface MigrationCheckpoint {
  lastProcessedId: string;
  processedCount: number;
  errorCount: number;
  startedAt: Date;
  lastRunAt: Date;
}

interface MigrationConfig {
  batchSize: number;
  delayBetweenBatchesMs: number;
  maxRetries: number;
  checkpointTable: string;
}

export class DataMigrationPipeline<TSource, TTarget> {
  constructor(
    private sourceDb: Pool,
    private targetDb: Pool,
    private config: MigrationConfig,
    private metrics: MetricsCollector,
    private transformer: (source: TSource) => TTarget,
    private migrationName: string
  ) {}

  async run(): Promise<void> {
    logger.info(`Starting migration: ${this.migrationName}`);
    
    const checkpoint = await this.loadCheckpoint();
    let processedCount = checkpoint?.processedCount ?? 0;
    let lastId = checkpoint?.lastProcessedId ?? '0';
    
    while (true) {
      const batch = await this.fetchBatch(lastId);
      
      if (batch.length === 0) {
        logger.info(`Migration complete: ${this.migrationName}`, {
          totalProcessed: processedCount,
        });
        break;
      }

      await this.processBatch(batch);
      
      processedCount += batch.length;
      lastId = (batch[batch.length - 1] as any).id;
      
      await this.saveCheckpoint({
        lastProcessedId: lastId,
        processedCount,
        errorCount: 0,
        startedAt: checkpoint?.startedAt ?? new Date(),
        lastRunAt: new Date(),
      });

      this.metrics.increment('migration.batch_processed', {
        migration: this.migrationName,
        batchSize: batch.length.toString(),
      });

      logger.info(`Migration progress: ${this.migrationName}`, {
        processed: processedCount,
        lastId,
      });

      if (batch.length < this.config.batchSize) {
        logger.info('Reached end of data');
        break;
      }

      await this.sleep(this.config.delayBetweenBatchesMs);
    }
  }

  private async fetchBatch(lastId: string): Promise<TSource[]> {
    const result = await this.sourceDb.query(
      `SELECT * FROM source_table WHERE id > $1 ORDER BY id LIMIT $2`,
      [lastId, this.config.batchSize]
    );
    return result.rows as TSource[];
  }

  private async processBatch(batch: TSource[]): Promise<void> {
    const transformed = batch.map(item => this.transformer(item));
    
    // Upsert ข้อมูลใน Batch
    const client = await this.targetDb.connect();
    
    try {
      await client.query('BEGIN');
      
      for (const item of transformed) {
        await this.upsertItem(client, item);
      }
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private async upsertItem(client: any, item: TTarget): Promise<void> {
    // Generic upsert - ต้อง Override ใน Subclass
    const keys = Object.keys(item as any);
    const values = Object.values(item as any);
    const placeholders = keys.map((_, i) => `$${i + 1}`).join(', ');
    const updateSet = keys
      .filter(k => k !== 'id')
      .map((k, i) => `${k} = $${i + 2}`)
      .join(', ');

    await client.query(
      `INSERT INTO target_table (${keys.join(', ')}) VALUES (${placeholders})
       ON CONFLICT (id) DO UPDATE SET ${updateSet}`,
      values
    );
  }

  private async loadCheckpoint(): Promise<MigrationCheckpoint | null> {
    try {
      const result = await this.sourceDb.query(
        `SELECT * FROM ${this.config.checkpointTable} WHERE migration_name = $1`,
        [this.migrationName]
      );
      return result.rows[0] ?? null;
    } catch {
      return null;
    }
  }

  private async saveCheckpoint(checkpoint: MigrationCheckpoint): Promise<void> {
    await this.sourceDb.query(
      `INSERT INTO ${this.config.checkpointTable} 
         (migration_name, last_processed_id, processed_count, error_count, started_at, last_run_at)
       VALUES ($1, $2, $3, $4, $5, $6)
       ON CONFLICT (migration_name) DO UPDATE SET
         last_processed_id = $2,
         processed_count = $3,
         error_count = $4,
         last_run_at = $6`,
      [
        this.migrationName,
        checkpoint.lastProcessedId,
        checkpoint.processedCount,
        checkpoint.errorCount,
        checkpoint.startedAt,
        checkpoint.lastRunAt,
      ]
    );
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## 7. Migration Verification

### 7.1 Data Consistency Verifier

```typescript
// src/migration/consistency-verifier.ts
import { Pool } from 'pg';
import { logger } from '../utils/logger';

interface VerificationResult {
  tableName: string;
  sourceCount: number;
  targetCount: number;
  countMatch: boolean;
  sampleMismatches: any[];
  checksumMatch?: boolean;
}

export class MigrationVerifier {
  constructor(
    private sourceDb: Pool,
    private targetDb: Pool
  ) {}

  async verifyMigration(
    sourceTable: string,
    targetTable: string,
    keyColumn: string,
    sampleSize: number = 100
  ): Promise<VerificationResult> {
    logger.info(`Verifying migration: ${sourceTable} -> ${targetTable}`);

    // ตรวจสอบจำนวน Records
    const [sourceCount, targetCount] = await Promise.all([
      this.getCount(this.sourceDb, sourceTable),
      this.getCount(this.targetDb, targetTable),
    ]);

    // Spot Check ตัวอย่าง Records
    const sampleIds = await this.getSampleIds(sourceTable, keyColumn, sampleSize);
    const mismatches = await this.checkSampleRecords(
      sourceTable,
      targetTable,
      keyColumn,
      sampleIds
    );

    const result: VerificationResult = {
      tableName: sourceTable,
      sourceCount,
      targetCount,
      countMatch: sourceCount === targetCount,
      sampleMismatches: mismatches,
    };

    if (!result.countMatch) {
      logger.error('Count mismatch detected', {
        table: sourceTable,
        sourceCount,
        targetCount,
        difference: sourceCount - targetCount,
      });
    }

    if (mismatches.length > 0) {
      logger.error('Data mismatches detected', {
        table: sourceTable,
        mismatchCount: mismatches.length,
        examples: mismatches.slice(0, 3),
      });
    }

    return result;
  }

  private async getCount(db: Pool, tableName: string): Promise<number> {
    const result = await db.query(`SELECT COUNT(*) as count FROM ${tableName}`);
    return parseInt(result.rows[0].count);
  }

  private async getSampleIds(
    tableName: string,
    keyColumn: string,
    sampleSize: number
  ): Promise<string[]> {
    const result = await this.sourceDb.query(
      `SELECT ${keyColumn} FROM ${tableName} ORDER BY RANDOM() LIMIT $1`,
      [sampleSize]
    );
    return result.rows.map(row => row[keyColumn]);
  }

  private async checkSampleRecords(
    sourceTable: string,
    targetTable: string,
    keyColumn: string,
    ids: string[]
  ): Promise<any[]> {
    const mismatches: any[] = [];
    
    for (const id of ids) {
      const [sourceRecord, targetRecord] = await Promise.all([
        this.sourceDb.query(
          `SELECT * FROM ${sourceTable} WHERE ${keyColumn} = $1`,
          [id]
        ),
        this.targetDb.query(
          `SELECT * FROM ${targetTable} WHERE ${keyColumn} = $1`,
          [id]
        ),
      ]);

      if (sourceRecord.rows.length === 0 && targetRecord.rows.length === 0) {
        continue;
      }

      if (sourceRecord.rows.length === 0 || targetRecord.rows.length === 0) {
        mismatches.push({
          id,
          issue: 'Record exists in one database but not the other',
          inSource: sourceRecord.rows.length > 0,
          inTarget: targetRecord.rows.length > 0,
        });
        continue;
      }

      // เปรียบเทียบ Fields ที่สำคัญ
      const source = sourceRecord.rows[0];
      const target = targetRecord.rows[0];
      
      const differences: string[] = [];
      
      // ตรวจสอบ Fields หลัก
      for (const field of Object.keys(source)) {
        if (JSON.stringify(source[field]) !== JSON.stringify(target[field])) {
          differences.push(`${field}: "${source[field]}" vs "${target[field]}"`);
        }
      }
      
      if (differences.length > 0) {
        mismatches.push({ id, differences });
      }
    }
    
    return mismatches;
  }
}
```

## สรุป

| Pattern | เหมาะกับ | ความเสี่ยง | ระยะเวลา |
|---------|---------|----------|---------|
| Strangler Fig | Extract Feature-by-Feature | ต่ำ | ยาว (3-12 เดือน) |
| Anti-Corruption Layer | Domain Model ต่างกันมาก | ต่ำ | ปานกลาง |
| Branch by Abstraction | Refactor ก่อน Extract | ต่ำ | ปานกลาง |
| Parallel Run | Validate ความถูกต้อง | ต่ำมาก | สั้น (สำหรับ Validation) |
| Dual Write | Database Migration | ปานกลาง | ปานกลาง |
| Data Pipeline | Batch Data Migration | ปานกลาง | ขึ้นกับขนาดข้อมูล |

กุญแจสำคัญในการ Migrate สำเร็จคือการทำ Incremental Migration ไม่ควร Big Bang ระบบ E-commerce ไทยที่มี Transaction สูงควรเริ่มจาก Service ที่ไม่ได้มี Strong Coupling กับส่วนอื่นก่อน เช่น Product Catalog หรือ Notification Service แล้วค่อยๆ ย้าย Order และ Payment ซึ่งมี Dependency ซับซ้อน

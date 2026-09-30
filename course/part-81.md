# Part 81: Microservices Anti-Patterns — ข้อผิดพลาดที่พบบ่อยและวิธีแก้ไข

## บทนำ

ในการพัฒนา Microservices หลายทีมมักตกหลุมพรางเดิมซ้ำๆ ซึ่งนำไปสู่ระบบที่ยากต่อการดูแลรักษา มีประสิทธิภาพต่ำ และเปราะบาง บทนี้จะสำรวจ Anti-Patterns ที่พบบ่อยที่สุด พร้อมแนวทางการระบุปัญหาและกลยุทธ์การแก้ไข

---

## Anti-Pattern 1: Distributed Monolith (Monolith in Disguise)

### ปัญหาคืออะไร?

Distributed Monolith เกิดขึ้นเมื่อคุณแยก service ออกจากกัน แต่ยังคง deploy และ scale ไปพร้อมกันทั้งหมด เหมือนกับ Monolith เดิม แต่มีความซับซ้อนของ distributed system เพิ่มเข้ามา

```
สัญญาณเตือน (Warning Signs):
┌─────────────────────────────────────────────────────────────────┐
│  1. ต้อง deploy ทุก service พร้อมกัน                             │
│  2. Service A fail แล้ว Service B-Z ก็ fail ตาม               │
│  3. Database schema เปลี่ยนแล้วทุก service ต้อง update          │
│  4. ไม่สามารถ scale service ใด service หนึ่งได้อิสระ             │
│  5. Circular dependencies ระหว่าง services                      │
└─────────────────────────────────────────────────────────────────┘
```

### ตัวอย่างโค้ดที่ผิด (Anti-Pattern)

```typescript
// ❌ BAD: OrderService ที่ต้องเรียก UserService, ProductService, 
//         InventoryService, PricingService พร้อมกันทุกครั้ง
// services/order/src/order.service.ts

import { Injectable } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';

@Injectable()
export class OrderService {
  constructor(private readonly httpService: HttpService) {}

  async createOrder(userId: string, items: OrderItem[]): Promise<Order> {
    // เรียก UserService เพื่อ validate user
    const user = await this.httpService
      .get(`http://user-service/users/${userId}`)
      .toPromise();

    // เรียก ProductService เพื่อดึงข้อมูล product
    const products = await Promise.all(
      items.map(item =>
        this.httpService
          .get(`http://product-service/products/${item.productId}`)
          .toPromise()
      )
    );

    // เรียก InventoryService เพื่อ check stock
    const inventory = await this.httpService
      .post('http://inventory-service/check', { items })
      .toPromise();

    // เรียก PricingService เพื่อคำนวณราคา
    const pricing = await this.httpService
      .post('http://pricing-service/calculate', { items, userId })
      .toPromise();

    // เรียก PaymentService เพื่อสร้าง payment intent
    const payment = await this.httpService
      .post('http://payment-service/create-intent', {
        amount: pricing.data.total,
        userId,
      })
      .toPromise();

    // ถ้า service ใด service หนึ่ง down -> ทั้งหมด fail
    return this.saveOrder({ user, products, inventory, pricing, payment });
  }
}
```

### วิธีแก้ไข: Event-Driven Architecture + Local Cache

```typescript
// ✅ GOOD: ใช้ Event-Driven + Local Data Denormalization
// services/order/src/order.service.ts

import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { Order } from './entities/order.entity';
import { OrderItem } from './entities/order-item.entity';
import { LocalProductCache } from './local-product-cache.service';
import { LocalUserCache } from './local-user-cache.service';

@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepo: Repository<Order>,
    private readonly eventEmitter: EventEmitter2,
    private readonly productCache: LocalProductCache,
    private readonly userCache: LocalUserCache,
  ) {}

  async createOrder(
    userId: string, 
    items: CreateOrderItemDto[]
  ): Promise<Order> {
    // ใช้ข้อมูล local ที่ sync มาแล้ว - ไม่ต้องเรียก service อื่น
    const user = await this.userCache.get(userId);
    if (!user) {
      throw new NotFoundException(`User ${userId} not found`);
    }

    const orderItems: OrderItem[] = [];
    let totalAmount = 0;

    for (const item of items) {
      // ใช้ locally cached product data
      const product = await this.productCache.get(item.productId);
      if (!product) {
        throw new NotFoundException(`Product ${item.productId} not found`);
      }

      if (product.stockQuantity < item.quantity) {
        throw new BadRequestException(
          `Insufficient stock for product ${item.productId}`
        );
      }

      const orderItem = this.orderRepo.manager.create(OrderItem, {
        productId: item.productId,
        productName: product.name, // denormalized data
        productSku: product.sku,
        unitPrice: product.price,
        quantity: item.quantity,
        subtotal: product.price * item.quantity,
      });

      orderItems.push(orderItem);
      totalAmount += orderItem.subtotal;
    }

    const order = this.orderRepo.create({
      userId,
      userName: user.name, // denormalized
      userEmail: user.email,
      items: orderItems,
      totalAmount,
      status: 'PENDING',
    });

    const savedOrder = await this.orderRepo.save(order);

    // Emit event แล้วให้ services อื่นจัดการต่อเอง
    await this.eventEmitter.emitAsync('order.created', {
      orderId: savedOrder.id,
      userId,
      items: orderItems.map(i => ({
        productId: i.productId,
        quantity: i.quantity,
      })),
      totalAmount,
    });

    return savedOrder;
  }
}
```

```typescript
// services/order/src/listeners/product-cache.listener.ts
// Sync product data จาก events

import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { LocalProductCache } from '../local-product-cache.service';

@Injectable()
export class ProductCacheListener {
  constructor(private readonly productCache: LocalProductCache) {}

  @OnEvent('product.updated')
  async handleProductUpdated(payload: ProductUpdatedEvent): Promise<void> {
    await this.productCache.update(payload.productId, {
      name: payload.name,
      price: payload.price,
      sku: payload.sku,
      stockQuantity: payload.stockQuantity,
    });
  }

  @OnEvent('product.deleted')
  async handleProductDeleted(payload: ProductDeletedEvent): Promise<void> {
    await this.productCache.delete(payload.productId);
  }
}
```

---

## Anti-Pattern 2: Chatty Services (Microservices กับ N+1 Problem)

### ปัญหาคืออะไร?

Service เรียก API อื่นมากเกินไปในแต่ละ request ทำให้ latency สูงและ network traffic มาก

```
ตัวอย่าง Chatty Pattern:
┌──────────────┐
│   Client     │
└──────┬───────┘
       │ 1 request
       ▼
┌──────────────┐
│  API Gateway │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌─────────────┐
│  Order       │────►│  Product    │ ← เรียก 1 ครั้ง
│  Service     │────►│  Service    │ ← เรียก 10 ครั้ง
└──────────────┘     └─────────────┘
       │              ← รวม 50 network calls ต่อ 1 user request
       │
       ▼ ← latency 500ms+
```

### ตัวอย่าง Anti-Pattern

```typescript
// ❌ BAD: N+1 Problem ใน Microservices
async getOrderHistory(userId: string): Promise<OrderWithDetails[]> {
  const orders = await this.orderRepo.find({ where: { userId } });
  // orders.length = 50
  
  const ordersWithDetails = await Promise.all(
    orders.map(async (order) => {
      const items = await Promise.all(
        order.itemIds.map(itemId =>
          // เรียก ProductService 50 * average_items ครั้ง = ~200 calls
          this.productClient.get(`/products/${itemId}`)
        )
      );
      
      // เรียก UserService อีก 50 ครั้ง
      const user = await this.userClient.get(`/users/${order.userId}`);
      
      return { ...order, items, user };
    })
  );
  
  return ordersWithDetails;
}
```

### วิธีแก้ไข: Batch Requests + DataLoader Pattern

```typescript
// ✅ GOOD: ใช้ DataLoader Pattern เพื่อ batch requests
// services/order/src/loaders/product.loader.ts

import DataLoader from 'dataloader';
import { Injectable, Scope } from '@nestjs/common';
import { ProductServiceClient } from '../clients/product-service.client';

@Injectable({ scope: Scope.REQUEST })
export class ProductLoader {
  private loader: DataLoader<string, Product>;

  constructor(private readonly productClient: ProductServiceClient) {
    this.loader = new DataLoader(
      async (productIds: readonly string[]) => {
        // เรียก batch endpoint 1 ครั้งแทน N ครั้ง
        const products = await this.productClient.getBatch(
          Array.from(productIds)
        );
        
        // Map กลับให้ตรงลำดับ productIds
        return productIds.map(
          id => products.find(p => p.id === id) || null
        );
      },
      {
        // Cache ใน request scope
        cache: true,
        // Batch window 10ms
        batchScheduleFn: (callback) => setTimeout(callback, 10),
        maxBatchSize: 100,
      }
    );
  }

  async load(productId: string): Promise<Product | null> {
    return this.loader.load(productId);
  }

  async loadMany(productIds: string[]): Promise<(Product | null)[]> {
    return this.loader.loadMany(productIds);
  }
}
```

```typescript
// services/order/src/order.service.ts - ใช้ DataLoader
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepo: Repository<Order>,
    private readonly productLoader: ProductLoader,
  ) {}

  async getOrderHistory(userId: string): Promise<OrderWithDetails[]> {
    const orders = await this.orderRepo.find({ 
      where: { userId },
      relations: ['items'],
    });

    // DataLoader จะ batch product calls โดยอัตโนมัติ
    const ordersWithDetails = await Promise.all(
      orders.map(async (order) => {
        const products = await this.productLoader.loadMany(
          order.items.map(item => item.productId)
        );
        
        return {
          ...order,
          items: order.items.map((item, index) => ({
            ...item,
            product: products[index],
          })),
        };
      })
    );

    return ordersWithDetails;
  }
}
```

```typescript
// services/product/src/controllers/product.controller.ts
// เพิ่ม Batch endpoint ใน Product Service

@Controller('products')
export class ProductController {
  constructor(private readonly productService: ProductService) {}

  @Get('batch')
  async getBatch(@Query('ids') ids: string): Promise<Product[]> {
    const productIds = ids.split(',').filter(Boolean);
    
    if (productIds.length > 100) {
      throw new BadRequestException('Maximum 100 products per batch request');
    }
    
    return this.productService.findByIds(productIds);
  }
}
```

---

## Anti-Pattern 3: Shared Database (Database-per-Service ที่ไม่ทำ)

### ปัญหาคืออะไร?

หลาย services share ฐานข้อมูลเดียวกัน ทำให้ services มี coupling กันสูงและไม่สามารถ scale หรือ deploy อิสระได้

```
Anti-Pattern: Shared Database
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Order   │  │  User    │  │ Product  │  │Inventory │   │
│  │ Service  │  │ Service  │  │ Service  │  │ Service  │   │
│  └─────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│        │            │             │              │          │
│        └────────────┴─────────────┴──────────────┘         │
│                              │                              │
│                    ┌─────────▼─────────┐                   │
│                    │   Shared MySQL    │                   │
│                    │   (monolith DB)   │                   │
│                    └───────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### วิธีแก้ไข: Database per Service + Event Sourcing

```yaml
# docker-compose.yml สำหรับ Database per Service pattern

version: '3.8'

services:
  # Order Service - PostgreSQL
  order-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: order_svc
      POSTGRES_PASSWORD: ${ORDER_DB_PASSWORD}
    volumes:
      - order-db-data:/var/lib/postgresql/data
    networks:
      - order-net
    # ไม่ให้ service อื่น access ได้
    
  order-service:
    build: ./services/order
    environment:
      DATABASE_URL: postgresql://order_svc:${ORDER_DB_PASSWORD}@order-db:5432/orders
    networks:
      - order-net
      - event-bus-net  # เชื่อมต่อผ่าน event bus เท่านั้น

  # User Service - MySQL
  user-db:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: users
      MYSQL_USER: user_svc
      MYSQL_PASSWORD: ${USER_DB_PASSWORD}
    volumes:
      - user-db-data:/var/lib/mysql
    networks:
      - user-net

  user-service:
    build: ./services/user
    environment:
      DATABASE_URL: mysql://user_svc:${USER_DB_PASSWORD}@user-db:3306/users
    networks:
      - user-net
      - event-bus-net

  # Product Service - MongoDB
  product-db:
    image: mongo:6
    environment:
      MONGO_INITDB_DATABASE: products
    volumes:
      - product-db-data:/data/db
    networks:
      - product-net

  product-service:
    build: ./services/product
    environment:
      MONGODB_URI: mongodb://product-db:27017/products
    networks:
      - product-net
      - event-bus-net

  # Message Bus (Kafka)
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    networks:
      - event-bus-net
    # ...

networks:
  order-net:
    internal: true  # isolate จาก services อื่น
  user-net:
    internal: true
  product-net:
    internal: true
  event-bus-net:  # shared สำหรับ events เท่านั้น

volumes:
  order-db-data:
  user-db-data:
  product-db-data:
```

```typescript
// ✅ GOOD: Migration Strategy - Strangler Fig Pattern
// tools/db-migration/src/strangler-fig.ts

interface MigrationConfig {
  legacyDbUrl: string;
  newServiceDbUrl: string;
  tableName: string;
  batchSize: number;
}

export class StranglerFigMigration {
  private readonly logger = new Logger(StranglerFigMigration.name);

  async migrate(config: MigrationConfig): Promise<void> {
    const legacyDb = await createConnection(config.legacyDbUrl);
    const newDb = await createConnection(config.newServiceDbUrl);

    let offset = 0;
    let migrated = 0;

    this.logger.log(`Starting migration of ${config.tableName}`);

    while (true) {
      // ดึงข้อมูล batch จาก legacy DB
      const records = await legacyDb.query(
        `SELECT * FROM ${config.tableName} 
         ORDER BY created_at 
         LIMIT ${config.batchSize} 
         OFFSET ${offset}`
      );

      if (records.length === 0) break;

      // Insert ลง new DB
      await newDb.transaction(async (trx) => {
        await trx
          .createQueryBuilder()
          .insert()
          .into(config.tableName)
          .values(records)
          .orIgnore() // ข้าม duplicate
          .execute();
      });

      migrated += records.length;
      offset += config.batchSize;

      this.logger.log(
        `Migrated ${migrated} records from ${config.tableName}`
      );

      // throttle เพื่อไม่ให้ lock DB
      await sleep(100);
    }

    this.logger.log(`Migration complete: ${migrated} records`);
  }
}
```

---

## Anti-Pattern 4: Too Fine-Grained Services (Nanoservices)

### ปัญหาคืออะไร?

แยก service เล็กเกินไปจนแต่ละ service ทำงานได้แค่สิ่งเดียวที่ง่ายมาก ทำให้มี overhead มากกว่าประโยชน์

```
❌ BAD: Nanoservices
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │Validate  │  │ Format   │  │ Persist  │                   │
│  │  Name    │  │  Phone   │  │  User    │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │ Hash     │  │  Send    │  │  Log     │                   │
│  │Password  │  │  Email   │  │  Event   │                   │
│  └──────────┘  └──────────┘  └──────────┘                   │
│                                                              │
│  ← ทุกอย่างนี้ควรเป็น UserService service เดียว              │
└──────────────────────────────────────────────────────────────┘
```

### Framework สำหรับ Deciding Service Boundaries

```
Bounded Context Analysis Template:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ถามคำถามเหล่านี้ก่อนแยก Service:

1. มี Team เป็นของตัวเองได้ไหม? (2-pizza rule: 6-10 คน)
   ถ้าไม่มี → อาจเล็กเกินไป

2. มี Business Capability ที่ชัดเจนไหม?
   ตัวอย่าง: "User Management", "Order Processing"
   ไม่ใช่: "Validate Phone Number", "Hash Password"

3. มี Data Domain ของตัวเองไหม?
   ถ้าต้องการ access DB ของ service อื่นตลอด → merge กัน

4. Rate of Change แตกต่างกันไหม?
   ถ้าเปลี่ยนพร้อมกันเสมอ → อาจควร merge

5. มี Independent Deployment value ไหม?
   ถ้า deploy ได้แต่ไม่มีประโยชน์ → ไม่จำเป็นต้องแยก
```

### Service Sizing Guidelines

```typescript
// ✅ GOOD: การกำหนด Service Boundaries ที่ถูกต้อง
// ใช้ Domain-Driven Design Bounded Contexts

// UserService ควรรวมทุกอย่างที่เกี่ยวกับ User Domain
// services/user/src/

export class UserService {
  // Registration (ไม่ใช่ validate-name-service แยก)
  async register(dto: RegisterUserDto): Promise<User> {
    // validate ทุกอย่างใน service เดียวกัน
    this.validateRegistrationData(dto);
    
    const hashedPassword = await bcrypt.hash(dto.password, 12);
    const formattedPhone = this.formatPhoneNumber(dto.phone);
    
    const user = await this.userRepo.save({
      name: dto.name.trim(),
      email: dto.email.toLowerCase(),
      phone: formattedPhone,
      password: hashedPassword,
    });
    
    // emit event สำหรับ services อื่น
    await this.eventEmitter.emit('user.registered', { userId: user.id });
    
    return user;
  }

  // Authentication - อยู่ใน User Domain
  async authenticate(email: string, password: string): Promise<AuthResult> {
    const user = await this.userRepo.findOne({ where: { email } });
    if (!user || !(await bcrypt.compare(password, user.password))) {
      throw new UnauthorizedException('Invalid credentials');
    }
    
    return {
      user,
      token: this.jwtService.sign({ sub: user.id }),
    };
  }

  // Profile management - ยังอยู่ใน User Domain
  async updateProfile(userId: string, dto: UpdateProfileDto): Promise<User> {
    // ...
  }

  // Private helpers - ไม่จำเป็นต้องเป็น service แยก
  private validateRegistrationData(dto: RegisterUserDto): void {
    if (!this.isValidEmail(dto.email)) {
      throw new BadRequestException('Invalid email format');
    }
    if (!this.isValidPhone(dto.phone)) {
      throw new BadRequestException('Invalid phone format');
    }
  }

  private formatPhoneNumber(phone: string): string {
    return phone.replace(/\D/g, '').replace(/^0/, '+66');
  }
}
```

---

## Anti-Pattern 5: Synchronous Chain of Calls

### ปัญหาคืออะไร?

```
Synchronous Chain:
Client → A → B → C → D → Response

ถ้า D ใช้เวลา 200ms:
- A + B + C + D = ~300ms total latency
- ถ้า D fail → ทุกอย่าง fail
- ถ้า D slow → ทุกอย่าง slow (Cascading Slowdown)
```

### วิธีแก้ไข: Saga Pattern

```typescript
// ✅ GOOD: Choreography-based Saga Pattern
// services/order/src/sagas/order-creation.saga.ts

import { Injectable } from '@nestjs/common';
import { OnEvent } from '@nestjs/event-emitter';
import { InjectRepository } from '@nestjs/typeorm';
import { Order } from '../entities/order.entity';

@Injectable()
export class OrderCreationSaga {
  constructor(
    @InjectRepository(Order)
    private readonly orderRepo: Repository<Order>,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  // Step 1: Order สร้างขึ้น
  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent): Promise<void> {
    // emit event ให้ Inventory Service ไป reserve items
    await this.eventEmitter.emit('inventory.reserve.requested', {
      orderId: event.orderId,
      items: event.items,
      correlationId: event.correlationId,
    });
  }

  // Step 2: Inventory reserved หรือ failed
  @OnEvent('inventory.reserved')
  async handleInventoryReserved(event: InventoryReservedEvent): Promise<void> {
    await this.orderRepo.update(event.orderId, { 
      status: 'INVENTORY_RESERVED' 
    });
    
    // emit event ให้ Payment Service ไป charge
    await this.eventEmitter.emit('payment.charge.requested', {
      orderId: event.orderId,
      amount: event.totalAmount,
      userId: event.userId,
      correlationId: event.correlationId,
    });
  }

  @OnEvent('inventory.reserve.failed')
  async handleInventoryFailed(event: InventoryFailedEvent): Promise<void> {
    // Compensating transaction - cancel order
    await this.orderRepo.update(event.orderId, { 
      status: 'CANCELLED',
      cancelReason: 'INVENTORY_UNAVAILABLE',
    });
    
    await this.eventEmitter.emit('order.cancelled', {
      orderId: event.orderId,
      reason: 'INVENTORY_UNAVAILABLE',
    });
  }

  // Step 3: Payment charged
  @OnEvent('payment.charged')
  async handlePaymentCharged(event: PaymentChargedEvent): Promise<void> {
    await this.orderRepo.update(event.orderId, { 
      status: 'PAYMENT_CONFIRMED',
      paymentId: event.paymentId,
    });
    
    // Saga complete - emit final success event
    await this.eventEmitter.emit('order.confirmed', {
      orderId: event.orderId,
    });
  }

  @OnEvent('payment.charge.failed')
  async handlePaymentFailed(event: PaymentFailedEvent): Promise<void> {
    // Compensating: release inventory reservation
    await this.eventEmitter.emit('inventory.release.requested', {
      orderId: event.orderId,
      items: event.items,
    });
    
    await this.orderRepo.update(event.orderId, { 
      status: 'CANCELLED',
      cancelReason: 'PAYMENT_FAILED',
    });
  }
}
```

---

## Anti-Pattern 6: Missing Circuit Breaker

### ตัวอย่างการใช้ Circuit Breaker ที่ถูกต้อง

```typescript
// ✅ GOOD: Circuit Breaker implementation
// services/order/src/clients/product.client.ts

import CircuitBreaker from 'opossum';
import { Injectable, Logger } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';

@Injectable()
export class ProductServiceClient {
  private readonly logger = new Logger(ProductServiceClient.name);
  private readonly circuitBreaker: CircuitBreaker;

  constructor(private readonly httpService: HttpService) {
    this.circuitBreaker = new CircuitBreaker(
      this.callProductService.bind(this),
      {
        timeout: 3000,          // 3 seconds
        errorThresholdPercentage: 50, // open ถ้า error > 50%
        resetTimeout: 30000,    // ลอง close หลังจาก 30s
        volumeThreshold: 5,     // ต้องมี 5 calls ก่อนจะ open
      }
    );

    this.circuitBreaker.fallback(this.getFallbackProduct.bind(this));

    this.circuitBreaker.on('open', () => {
      this.logger.warn('Circuit breaker OPENED for ProductService');
    });

    this.circuitBreaker.on('halfOpen', () => {
      this.logger.log('Circuit breaker HALF-OPEN for ProductService');
    });

    this.circuitBreaker.on('close', () => {
      this.logger.log('Circuit breaker CLOSED for ProductService');
    });
  }

  private async callProductService(productId: string): Promise<Product> {
    const response = await this.httpService
      .get<Product>(`http://product-service/products/${productId}`)
      .toPromise();
    return response.data;
  }

  private async getFallbackProduct(productId: string): Promise<Product | null> {
    // Return cached data หรือ null
    this.logger.warn(`Using fallback for product ${productId}`);
    return null;
  }

  async getProduct(productId: string): Promise<Product | null> {
    return this.circuitBreaker.fire(productId) as Promise<Product | null>;
  }
}
```

---

## Anti-Pattern 7: No API Versioning

### ปัญหาและวิธีแก้ไข

```typescript
// ❌ BAD: ไม่มี API versioning
@Controller('users')
export class UserController {
  @Get(':id')
  getUser(@Param('id') id: string) {
    // เปลี่ยน response format ตามใจชอบ → ทำให้ clients break
  }
}

// ✅ GOOD: URL versioning + backward compatibility
@Controller({ path: 'users', version: '1' })
export class UserV1Controller {
  @Get(':id')
  async getUser(@Param('id') id: string): Promise<UserV1Response> {
    const user = await this.userService.findById(id);
    return {
      id: user.id,
      name: user.name,
      email: user.email,
    };
  }
}

@Controller({ path: 'users', version: '2' })
export class UserV2Controller {
  @Get(':id')
  async getUser(@Param('id') id: string): Promise<UserV2Response> {
    const user = await this.userService.findById(id);
    return {
      id: user.id,
      firstName: user.firstName,  // แยก name เป็น firstName/lastName
      lastName: user.lastName,
      email: user.email,
      phone: user.phone,           // เพิ่ม field ใหม่
    };
  }
}

// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableVersioning({
    type: VersioningType.URI,
    defaultVersion: '1', // backward compatible
  });
}
```

---

## กลยุทธ์การ Refactor: Strangler Fig Pattern

```
Strangler Fig Migration Strategy:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Phase 1: Identify & Extract (สัปดาห์ 1-4)
┌─────────────────────────────────────────┐
│  Legacy Monolith                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐  │
│  │ Orders  │ │  Users  │ │Products │  │
│  └─────────┘ └─────────┘ └─────────┘  │
└─────────────────────────────────────────┘

Phase 2: Route Traffic (สัปดาห์ 5-8)
┌──────────┐     ┌─────────────────────────────┐
│  Client  │────►│  API Gateway / Facade       │
└──────────┘     └──┬──────────────────────┬───┘
                    │                      │
           ┌────────▼────────┐    ┌────────▼────────┐
           │  New User       │    │  Legacy Monolith │
           │  Microservice   │    │  (Orders+Products│
           └─────────────────┘    └─────────────────┘

Phase 3: Complete Migration (สัปดาห์ 9-16)
┌──────────┐     ┌─────────────────────────────┐
│  Client  │────►│  API Gateway                │
└──────────┘     └──┬──────────┬───────────────┘
                    │          │
         ┌──────────▼──┐  ┌───▼──────────┐
         │ User Service│  │Order+Product │
         │  (new)      │  │ Services     │
         └─────────────┘  └──────────────┘
```

```typescript
// tools/migration/src/strangler-facade.ts
// Facade ที่ route traffic ไปยัง service ใหม่หรือ legacy

import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { UserMicroserviceClient } from '../clients/user-microservice.client';
import { LegacyMonolithClient } from '../clients/legacy-monolith.client';

@Injectable()
export class UserFacade {
  constructor(
    private readonly configService: ConfigService,
    private readonly newService: UserMicroserviceClient,
    private readonly legacyService: LegacyMonolithClient,
  ) {}

  async getUser(userId: string): Promise<User> {
    const featureFlag = this.configService.get<boolean>(
      'features.useNewUserService'
    );

    if (featureFlag) {
      try {
        return await this.newService.getUser(userId);
      } catch (error) {
        // Fallback to legacy if new service fails
        this.logger.error('New user service failed, falling back', error);
        return await this.legacyService.getUser(userId);
      }
    }

    return await this.legacyService.getUser(userId);
  }
}
```

---

## Checklist: Anti-Pattern Identification

```
┌─────────────────────────────────────────────────────────────────────┐
│           MICROSERVICES ANTI-PATTERN CHECKLIST                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Distributed Monolith                                               │
│  [ ] Services ต้อง deploy พร้อมกันไหม?                              │
│  [ ] มี circular dependencies ระหว่าง services ไหม?                │
│  [ ] Service fail หนึ่งแล้วทุกอย่าง fail ไหม?                       │
│                                                                     │
│  Chatty Services                                                    │
│  [ ] มีการเรียก API มากกว่า 5 ครั้งต่อ 1 request ไหม?              │
│  [ ] มี N+1 query pattern ไหม?                                      │
│  [ ] ใช้ batch endpoints ไหม?                                       │
│                                                                     │
│  Shared Database                                                    │
│  [ ] แต่ละ service มี database ของตัวเองไหม?                        │
│  [ ] มี service ที่ access DB ของ service อื่นโดยตรงไหม?            │
│  [ ] มี schema ที่ share กันระหว่าง services ไหม?                   │
│                                                                     │
│  Over-decomposition                                                 │
│  [ ] แต่ละ service มี team ดูแลได้ (2-pizza rule) ไหม?              │
│  [ ] แต่ละ service deploy อิสระได้ไหม?                              │
│  [ ] มี service ที่ทำงานเพียง 1-2 functions ไหม?                    │
│                                                                     │
│  Missing Resilience                                                 │
│  [ ] ทุก external call มี circuit breaker ไหม?                     │
│  [ ] มี retry logic ไหม?                                            │
│  [ ] มี timeout ทุก request ไหม?                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## การวัดผล: Metrics สำหรับตรวจจับ Anti-Patterns

```typescript
// services/shared/src/metrics/anti-pattern-detector.ts

import { Injectable } from '@nestjs/common';
import { InjectMetric } from '@willsoto/nestjs-prometheus';
import { Counter, Histogram } from 'prom-client';

@Injectable()
export class AntiPatternMetrics {
  constructor(
    @InjectMetric('service_calls_total')
    private readonly callsCounter: Counter,
    
    @InjectMetric('service_call_duration_ms')
    private readonly callDuration: Histogram,
    
    @InjectMetric('circuit_breaker_state')
    private readonly circuitBreakerState: Counter,
  ) {}

  // ตรวจจับ Chatty Service: calls per request > threshold
  recordOutgoingCall(
    targetService: string,
    operation: string,
    durationMs: number,
    success: boolean,
  ): void {
    this.callsCounter.inc({
      target_service: targetService,
      operation,
      status: success ? 'success' : 'error',
    });

    this.callDuration.observe(
      { target_service: targetService, operation },
      durationMs
    );
  }
}
```

```yaml
# prometheus/rules/anti-pattern-alerts.yml

groups:
  - name: microservices-anti-patterns
    rules:
      # Alert: Chatty Service - มากกว่า 10 calls ต่อ request
      - alert: ChattyServiceDetected
        expr: |
          rate(service_calls_total[5m]) / 
          rate(http_requests_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Chatty service detected: {{ $labels.service }}"
          description: "Service is making more than 10 downstream calls per request"

      # Alert: Cascade Failure - หลาย services fail พร้อมกัน
      - alert: CascadeFailureRisk
        expr: |
          count(
            up{job=~".*-service"} == 0
          ) > 2
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Multiple services down - possible cascade failure"
```

---

## สรุป

Anti-patterns ใน Microservices เป็นเรื่องที่พบได้บ่อย โดยเฉพาะในทีมที่เพิ่งเริ่มต้น ประเด็นสำคัญที่ต้องจำ:

| Anti-Pattern | สัญญาณเตือน | วิธีแก้ไข |
|---|---|---|
| Distributed Monolith | Deploy พร้อมกันทุก service | Event-Driven Architecture |
| Chatty Services | N+1 API calls | DataLoader + Batch endpoints |
| Shared Database | Services share DB | Database per Service + Events |
| Nanoservices | Service ทำงานแค่ 1-2 functions | Domain-Driven Boundaries |
| Sync Chain | Cascade failures | Saga Pattern + Async messaging |
| No Circuit Breaker | Cascade failures | opossum / resilience4j |
| No Versioning | API changes break clients | URL/Header versioning |

กุญแจสำคัญในการหลีกเลี่ยง Anti-patterns คือ:
1. **ทำ Domain Analysis ก่อน** — ใช้ DDD Bounded Contexts
2. **เริ่มจาก Monolith** — แล้วค่อยแยกตาม business needs ที่ชัดเจน
3. **ใช้ Fitness Functions** — วัด coupling และ cohesion อย่างต่อเนื่อง
4. **Design for Failure** — circuit breakers, retries, timeouts ทุก external call
5. **Event-Driven First** — ลด synchronous coupling ระหว่าง services

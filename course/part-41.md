# Part 41: Domain-Driven Design (DDD) สำหรับ Microservices

## บทนำ

Domain-Driven Design (DDD) คือแนวทางการออกแบบซอฟต์แวร์ที่เน้นการสร้าง Model จาก Business Domain เป็นหลัก ในบริบทของ Microservices DDD ช่วยกำหนดขอบเขตของแต่ละ Service ได้อย่างชัดเจนผ่าน Bounded Contexts และสร้าง Domain Model ที่สะท้อน Business Rules อย่างแม่นยำ

### สิ่งที่จะได้เรียนรู้

- Bounded Contexts และการแบ่ง Microservices
- Aggregates, Entities, Value Objects
- Domain Events และ Event Sourcing
- Repository Pattern ด้วย TypeScript + PostgreSQL
- Application Services และ Domain Services
- Anti-Corruption Layer (ACL)
- CQRS (Command Query Responsibility Segregation)

---

## 1. Bounded Contexts

Bounded Context คือขอบเขตที่ชัดเจนของ Domain Model ภายใน Context หนึ่งๆ คำเดียวกันอาจมีความหมายต่างกันใน Context ต่างกัน

### ตัวอย่าง: ระบบ E-Commerce

```
┌─────────────────────────────────────────────────────────┐
│                    E-Commerce System                     │
│                                                          │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │  Order Context   │    │ Catalog Context  │           │
│  │                  │    │                  │           │
│  │  - Order         │    │  - Product       │           │
│  │  - OrderItem     │    │  - Category      │           │
│  │  - Customer      │    │  - Inventory     │           │
│  └──────────────────┘    └──────────────────┘           │
│                                                          │
│  ┌──────────────────┐    ┌──────────────────┐           │
│  │ Payment Context  │    │Shipping Context  │           │
│  │                  │    │                  │           │
│  │  - Payment       │    │  - Shipment      │           │
│  │  - Invoice       │    │  - Carrier       │           │
│  │  - Customer      │    │  - Address       │           │
│  └──────────────────┘    └──────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

### Context Map

```typescript
// context-map.ts
// แสดงความสัมพันธ์ระหว่าง Bounded Contexts

/**
 * Order Context → Catalog Context: Customer/Supplier
 * Order Context → Payment Context: Partnership
 * Order Context → Shipping Context: Customer/Supplier
 * Payment Context → Order Context: Conformist
 */

// Shared Kernel ระหว่าง Order และ Payment
export namespace SharedKernel {
  export interface Money {
    amount: number;
    currency: string;
  }

  export interface Address {
    street: string;
    city: string;
    province: string;
    postalCode: string;
    country: string;
  }
}
```

---

## 2. Value Objects

Value Objects ไม่มี Identity ของตัวเอง ถูกกำหนดด้วยค่า (value) และต้องเป็น Immutable

```typescript
// src/domain/value-objects/money.vo.ts
export class Money {
  private readonly _amount: number;
  private readonly _currency: string;

  private constructor(amount: number, currency: string) {
    if (amount < 0) {
      throw new Error('Amount cannot be negative');
    }
    if (!currency || currency.length !== 3) {
      throw new Error('Currency must be a 3-letter ISO code');
    }
    this._amount = Math.round(amount * 100) / 100; // 2 decimal places
    this._currency = currency.toUpperCase();
  }

  static of(amount: number, currency: string): Money {
    return new Money(amount, currency);
  }

  static zero(currency: string): Money {
    return new Money(0, currency);
  }

  get amount(): number {
    return this._amount;
  }

  get currency(): string {
    return this._currency;
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    const result = this._amount - other._amount;
    if (result < 0) {
      throw new Error('Insufficient funds');
    }
    return new Money(result, this._currency);
  }

  multiply(factor: number): Money {
    return new Money(this._amount * factor, this._currency);
  }

  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  isLessThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount < other._amount;
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new Error(
        `Currency mismatch: ${this._currency} vs ${other._currency}`
      );
    }
  }

  toString(): string {
    return `${this._currency} ${this._amount.toFixed(2)}`;
  }

  toJSON(): { amount: number; currency: string } {
    return { amount: this._amount, currency: this._currency };
  }
}
```

```typescript
// src/domain/value-objects/email.vo.ts
export class Email {
  private readonly _value: string;

  private constructor(value: string) {
    const emailRegex = /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/;
    if (!emailRegex.test(value)) {
      throw new Error(`Invalid email address: ${value}`);
    }
    this._value = value.toLowerCase();
  }

  static of(value: string): Email {
    return new Email(value);
  }

  get value(): string {
    return this._value;
  }

  equals(other: Email): boolean {
    return this._value === other._value;
  }

  toString(): string {
    return this._value;
  }
}
```

```typescript
// src/domain/value-objects/order-id.vo.ts
import { v4 as uuidv4 } from 'uuid';

export class OrderId {
  private readonly _value: string;

  private constructor(value: string) {
    if (!value || value.trim().length === 0) {
      throw new Error('OrderId cannot be empty');
    }
    // Validate UUID format
    const uuidRegex =
      /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    if (!uuidRegex.test(value)) {
      throw new Error(`Invalid OrderId format: ${value}`);
    }
    this._value = value;
  }

  static generate(): OrderId {
    return new OrderId(uuidv4());
  }

  static of(value: string): OrderId {
    return new OrderId(value);
  }

  get value(): string {
    return this._value;
  }

  equals(other: OrderId): boolean {
    return this._value === other._value;
  }

  toString(): string {
    return this._value;
  }
}
```

```typescript
// src/domain/value-objects/address.vo.ts
export interface AddressProps {
  street: string;
  city: string;
  province: string;
  postalCode: string;
  country: string;
}

export class Address {
  private readonly _street: string;
  private readonly _city: string;
  private readonly _province: string;
  private readonly _postalCode: string;
  private readonly _country: string;

  private constructor(props: AddressProps) {
    if (!props.street?.trim()) throw new Error('Street is required');
    if (!props.city?.trim()) throw new Error('City is required');
    if (!props.province?.trim()) throw new Error('Province is required');
    if (!props.postalCode?.trim()) throw new Error('Postal code is required');
    if (!props.country?.trim()) throw new Error('Country is required');

    this._street = props.street.trim();
    this._city = props.city.trim();
    this._province = props.province.trim();
    this._postalCode = props.postalCode.trim();
    this._country = props.country.trim().toUpperCase();
  }

  static of(props: AddressProps): Address {
    return new Address(props);
  }

  get street(): string { return this._street; }
  get city(): string { return this._city; }
  get province(): string { return this._province; }
  get postalCode(): string { return this._postalCode; }
  get country(): string { return this._country; }

  equals(other: Address): boolean {
    return (
      this._street === other._street &&
      this._city === other._city &&
      this._province === other._province &&
      this._postalCode === other._postalCode &&
      this._country === other._country
    );
  }

  toString(): string {
    return `${this._street}, ${this._city}, ${this._province} ${this._postalCode}, ${this._country}`;
  }

  toJSON(): AddressProps {
    return {
      street: this._street,
      city: this._city,
      province: this._province,
      postalCode: this._postalCode,
      country: this._country,
    };
  }
}
```

---

## 3. Entities

Entities มี Identity ของตัวเอง และสามารถเปลี่ยนแปลงสถานะได้

```typescript
// src/domain/entities/order-item.entity.ts
import { Money } from '../value-objects/money.vo';

export class OrderItem {
  private readonly _id: string;
  private readonly _productId: string;
  private readonly _productName: string;
  private _quantity: number;
  private readonly _unitPrice: Money;

  constructor(
    id: string,
    productId: string,
    productName: string,
    quantity: number,
    unitPrice: Money
  ) {
    if (!id) throw new Error('OrderItem id is required');
    if (!productId) throw new Error('Product id is required');
    if (!productName) throw new Error('Product name is required');
    if (quantity <= 0) throw new Error('Quantity must be positive');

    this._id = id;
    this._productId = productId;
    this._productName = productName;
    this._quantity = quantity;
    this._unitPrice = unitPrice;
  }

  get id(): string { return this._id; }
  get productId(): string { return this._productId; }
  get productName(): string { return this._productName; }
  get quantity(): number { return this._quantity; }
  get unitPrice(): Money { return this._unitPrice; }

  get totalPrice(): Money {
    return this._unitPrice.multiply(this._quantity);
  }

  updateQuantity(newQuantity: number): void {
    if (newQuantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    this._quantity = newQuantity;
  }

  equals(other: OrderItem): boolean {
    return this._id === other._id;
  }
}
```

---

## 4. Aggregates

Aggregate คือกลุ่มของ Entities และ Value Objects ที่ถูกจัดการเป็นหน่วยเดียวกัน Aggregate Root เป็น Entry point เดียว

```typescript
// src/domain/aggregates/order.aggregate.ts
import { v4 as uuidv4 } from 'uuid';
import { OrderId } from '../value-objects/order-id.vo';
import { Money } from '../value-objects/money.vo';
import { Address } from '../value-objects/address.vo';
import { OrderItem } from '../entities/order-item.entity';
import { DomainEvent } from '../events/domain-event';
import { OrderCreatedEvent } from '../events/order-created.event';
import { OrderItemAddedEvent } from '../events/order-item-added.event';
import { OrderConfirmedEvent } from '../events/order-confirmed.event';
import { OrderCancelledEvent } from '../events/order-cancelled.event';

export enum OrderStatus {
  DRAFT = 'DRAFT',
  CONFIRMED = 'CONFIRMED',
  PAID = 'PAID',
  SHIPPED = 'SHIPPED',
  DELIVERED = 'DELIVERED',
  CANCELLED = 'CANCELLED',
}

export interface OrderProps {
  id: OrderId;
  customerId: string;
  shippingAddress: Address;
  status: OrderStatus;
  items: OrderItem[];
  currency: string;
  createdAt: Date;
  updatedAt: Date;
}

export class Order {
  private readonly _id: OrderId;
  private readonly _customerId: string;
  private _shippingAddress: Address;
  private _status: OrderStatus;
  private _items: OrderItem[];
  private readonly _currency: string;
  private readonly _createdAt: Date;
  private _updatedAt: Date;
  private _domainEvents: DomainEvent[] = [];

  private constructor(props: OrderProps) {
    this._id = props.id;
    this._customerId = props.customerId;
    this._shippingAddress = props.shippingAddress;
    this._status = props.status;
    this._items = [...props.items];
    this._currency = props.currency;
    this._createdAt = props.createdAt;
    this._updatedAt = props.updatedAt;
  }

  // Factory method
  static create(
    customerId: string,
    shippingAddress: Address,
    currency: string = 'THB'
  ): Order {
    const orderId = OrderId.generate();
    const now = new Date();

    const order = new Order({
      id: orderId,
      customerId,
      shippingAddress,
      status: OrderStatus.DRAFT,
      items: [],
      currency,
      createdAt: now,
      updatedAt: now,
    });

    order.addDomainEvent(
      new OrderCreatedEvent({
        orderId: orderId.value,
        customerId,
        currency,
        occurredAt: now,
      })
    );

    return order;
  }

  // Reconstitute from persistence
  static reconstitute(props: OrderProps): Order {
    return new Order(props);
  }

  // Getters
  get id(): OrderId { return this._id; }
  get customerId(): string { return this._customerId; }
  get shippingAddress(): Address { return this._shippingAddress; }
  get status(): OrderStatus { return this._status; }
  get items(): ReadonlyArray<OrderItem> { return [...this._items]; }
  get currency(): string { return this._currency; }
  get createdAt(): Date { return this._createdAt; }
  get updatedAt(): Date { return this._updatedAt; }

  get totalAmount(): Money {
    return this._items.reduce(
      (total, item) => total.add(item.totalPrice),
      Money.zero(this._currency)
    );
  }

  get itemCount(): number {
    return this._items.reduce((count, item) => count + item.quantity, 0);
  }

  // Business methods
  addItem(
    productId: string,
    productName: string,
    quantity: number,
    unitPrice: Money
  ): void {
    this.assertNotCancelledOrDelivered();
    this.assertStatus(OrderStatus.DRAFT, 'Can only add items to DRAFT orders');

    // Check if item already exists
    const existingItem = this._items.find(
      (item) => item.productId === productId
    );

    if (existingItem) {
      existingItem.updateQuantity(existingItem.quantity + quantity);
    } else {
      const itemId = uuidv4();
      const newItem = new OrderItem(
        itemId,
        productId,
        productName,
        quantity,
        unitPrice
      );
      this._items.push(newItem);

      this.addDomainEvent(
        new OrderItemAddedEvent({
          orderId: this._id.value,
          productId,
          productName,
          quantity,
          unitPrice: unitPrice.toJSON(),
          occurredAt: new Date(),
        })
      );
    }

    this._updatedAt = new Date();
  }

  removeItem(productId: string): void {
    this.assertStatus(OrderStatus.DRAFT, 'Can only remove items from DRAFT orders');

    const index = this._items.findIndex((item) => item.productId === productId);
    if (index === -1) {
      throw new Error(`Item with productId ${productId} not found`);
    }

    this._items.splice(index, 1);
    this._updatedAt = new Date();
  }

  updateShippingAddress(address: Address): void {
    if (
      this._status === OrderStatus.SHIPPED ||
      this._status === OrderStatus.DELIVERED
    ) {
      throw new Error('Cannot update address for shipped or delivered orders');
    }
    this._shippingAddress = address;
    this._updatedAt = new Date();
  }

  confirm(): void {
    this.assertStatus(OrderStatus.DRAFT, 'Can only confirm DRAFT orders');

    if (this._items.length === 0) {
      throw new Error('Cannot confirm an empty order');
    }

    this._status = OrderStatus.CONFIRMED;
    this._updatedAt = new Date();

    this.addDomainEvent(
      new OrderConfirmedEvent({
        orderId: this._id.value,
        customerId: this._customerId,
        totalAmount: this.totalAmount.toJSON(),
        occurredAt: new Date(),
      })
    );
  }

  markAsPaid(): void {
    this.assertStatus(OrderStatus.CONFIRMED, 'Can only mark CONFIRMED orders as paid');
    this._status = OrderStatus.PAID;
    this._updatedAt = new Date();
  }

  ship(): void {
    this.assertStatus(OrderStatus.PAID, 'Can only ship PAID orders');
    this._status = OrderStatus.SHIPPED;
    this._updatedAt = new Date();
  }

  deliver(): void {
    this.assertStatus(OrderStatus.SHIPPED, 'Can only deliver SHIPPED orders');
    this._status = OrderStatus.DELIVERED;
    this._updatedAt = new Date();
  }

  cancel(reason: string): void {
    if (
      this._status === OrderStatus.SHIPPED ||
      this._status === OrderStatus.DELIVERED
    ) {
      throw new Error('Cannot cancel shipped or delivered orders');
    }

    this._status = OrderStatus.CANCELLED;
    this._updatedAt = new Date();

    this.addDomainEvent(
      new OrderCancelledEvent({
        orderId: this._id.value,
        customerId: this._customerId,
        reason,
        occurredAt: new Date(),
      })
    );
  }

  // Domain Events
  get domainEvents(): ReadonlyArray<DomainEvent> {
    return [...this._domainEvents];
  }

  clearDomainEvents(): void {
    this._domainEvents = [];
  }

  private addDomainEvent(event: DomainEvent): void {
    this._domainEvents.push(event);
  }

  // Guard methods
  private assertStatus(expectedStatus: OrderStatus, message: string): void {
    if (this._status !== expectedStatus) {
      throw new Error(
        `${message}. Current status: ${this._status}, Expected: ${expectedStatus}`
      );
    }
  }

  private assertNotCancelledOrDelivered(): void {
    if (
      this._status === OrderStatus.CANCELLED ||
      this._status === OrderStatus.DELIVERED
    ) {
      throw new Error(
        `Cannot modify ${this._status.toLowerCase()} order`
      );
    }
  }
}
```

---

## 5. Domain Events

Domain Events แทนเหตุการณ์สำคัญที่เกิดขึ้นใน Domain

```typescript
// src/domain/events/domain-event.ts
export abstract class DomainEvent {
  readonly eventId: string;
  readonly occurredAt: Date;
  abstract readonly eventType: string;

  constructor(occurredAt: Date = new Date()) {
    this.eventId = require('uuid').v4();
    this.occurredAt = occurredAt;
  }
}
```

```typescript
// src/domain/events/order-created.event.ts
import { DomainEvent } from './domain-event';

interface OrderCreatedEventProps {
  orderId: string;
  customerId: string;
  currency: string;
  occurredAt: Date;
}

export class OrderCreatedEvent extends DomainEvent {
  readonly eventType = 'ORDER_CREATED';
  readonly orderId: string;
  readonly customerId: string;
  readonly currency: string;

  constructor(props: OrderCreatedEventProps) {
    super(props.occurredAt);
    this.orderId = props.orderId;
    this.customerId = props.customerId;
    this.currency = props.currency;
  }
}
```

```typescript
// src/domain/events/order-confirmed.event.ts
import { DomainEvent } from './domain-event';

interface OrderConfirmedEventProps {
  orderId: string;
  customerId: string;
  totalAmount: { amount: number; currency: string };
  occurredAt: Date;
}

export class OrderConfirmedEvent extends DomainEvent {
  readonly eventType = 'ORDER_CONFIRMED';
  readonly orderId: string;
  readonly customerId: string;
  readonly totalAmount: { amount: number; currency: string };

  constructor(props: OrderConfirmedEventProps) {
    super(props.occurredAt);
    this.orderId = props.orderId;
    this.customerId = props.customerId;
    this.totalAmount = props.totalAmount;
  }
}
```

```typescript
// src/domain/events/order-cancelled.event.ts
import { DomainEvent } from './domain-event';

interface OrderCancelledEventProps {
  orderId: string;
  customerId: string;
  reason: string;
  occurredAt: Date;
}

export class OrderCancelledEvent extends DomainEvent {
  readonly eventType = 'ORDER_CANCELLED';
  readonly orderId: string;
  readonly customerId: string;
  readonly reason: string;

  constructor(props: OrderCancelledEventProps) {
    super(props.occurredAt);
    this.orderId = props.orderId;
    this.customerId = props.customerId;
    this.reason = props.reason;
  }
}
```

---

## 6. Repository Pattern

Repository จัดการการ Persist และ Retrieve Aggregates

```typescript
// src/domain/repositories/order.repository.ts
import { Order } from '../aggregates/order.aggregate';
import { OrderId } from '../value-objects/order-id.vo';

export interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  findByCustomerId(customerId: string): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: OrderId): Promise<void>;
  nextIdentity(): Promise<OrderId>;
}
```

```typescript
// src/infrastructure/repositories/postgres-order.repository.ts
import { Pool, PoolClient } from 'pg';
import { Order, OrderStatus } from '../../domain/aggregates/order.aggregate';
import { OrderRepository } from '../../domain/repositories/order.repository';
import { OrderId } from '../../domain/value-objects/order-id.vo';
import { Money } from '../../domain/value-objects/money.vo';
import { Address } from '../../domain/value-objects/address.vo';
import { OrderItem } from '../../domain/entities/order-item.entity';
import { DomainEventPublisher } from '../events/domain-event-publisher';

export class PostgresOrderRepository implements OrderRepository {
  constructor(
    private readonly pool: Pool,
    private readonly eventPublisher: DomainEventPublisher
  ) {}

  async findById(id: OrderId): Promise<Order | null> {
    const client = await this.pool.connect();
    try {
      const orderResult = await client.query(
        `SELECT o.*, 
                oi.id as item_id,
                oi.product_id,
                oi.product_name,
                oi.quantity,
                oi.unit_price_amount,
                oi.unit_price_currency
         FROM orders o
         LEFT JOIN order_items oi ON oi.order_id = o.id
         WHERE o.id = $1 AND o.deleted_at IS NULL`,
        [id.value]
      );

      if (orderResult.rows.length === 0) {
        return null;
      }

      return this.mapRowsToOrder(orderResult.rows);
    } finally {
      client.release();
    }
  }

  async findByCustomerId(customerId: string): Promise<Order[]> {
    const client = await this.pool.connect();
    try {
      const result = await client.query(
        `SELECT o.*, 
                oi.id as item_id,
                oi.product_id,
                oi.product_name,
                oi.quantity,
                oi.unit_price_amount,
                oi.unit_price_currency
         FROM orders o
         LEFT JOIN order_items oi ON oi.order_id = o.id
         WHERE o.customer_id = $1 AND o.deleted_at IS NULL
         ORDER BY o.created_at DESC`,
        [customerId]
      );

      return this.groupRowsByOrderId(result.rows).map(this.mapRowsToOrder);
    } finally {
      client.release();
    }
  }

  async save(order: Order): Promise<void> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      // Upsert order
      await client.query(
        `INSERT INTO orders (
          id, customer_id, status, currency,
          shipping_street, shipping_city, shipping_province,
          shipping_postal_code, shipping_country,
          created_at, updated_at
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
        ON CONFLICT (id) DO UPDATE SET
          status = EXCLUDED.status,
          shipping_street = EXCLUDED.shipping_street,
          shipping_city = EXCLUDED.shipping_city,
          shipping_province = EXCLUDED.shipping_province,
          shipping_postal_code = EXCLUDED.shipping_postal_code,
          shipping_country = EXCLUDED.shipping_country,
          updated_at = EXCLUDED.updated_at`,
        [
          order.id.value,
          order.customerId,
          order.status,
          order.currency,
          order.shippingAddress.street,
          order.shippingAddress.city,
          order.shippingAddress.province,
          order.shippingAddress.postalCode,
          order.shippingAddress.country,
          order.createdAt,
          order.updatedAt,
        ]
      );

      // Delete existing items and reinsert (simplest approach for small orders)
      await client.query('DELETE FROM order_items WHERE order_id = $1', [
        order.id.value,
      ]);

      for (const item of order.items) {
        await client.query(
          `INSERT INTO order_items (
            id, order_id, product_id, product_name,
            quantity, unit_price_amount, unit_price_currency
          ) VALUES ($1, $2, $3, $4, $5, $6, $7)`,
          [
            item.id,
            order.id.value,
            item.productId,
            item.productName,
            item.quantity,
            item.unitPrice.amount,
            item.unitPrice.currency,
          ]
        );
      }

      await client.query('COMMIT');

      // Publish domain events after successful commit
      const events = order.domainEvents;
      order.clearDomainEvents();

      for (const event of events) {
        await this.eventPublisher.publish(event);
      }
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async delete(id: OrderId): Promise<void> {
    await this.pool.query(
      'UPDATE orders SET deleted_at = NOW() WHERE id = $1',
      [id.value]
    );
  }

  async nextIdentity(): Promise<OrderId> {
    return OrderId.generate();
  }

  private mapRowsToOrder(rows: any[]): Order {
    const firstRow = rows[0];

    const items = rows
      .filter((row) => row.item_id)
      .map(
        (row) =>
          new OrderItem(
            row.item_id,
            row.product_id,
            row.product_name,
            row.quantity,
            Money.of(parseFloat(row.unit_price_amount), row.unit_price_currency)
          )
      );

    return Order.reconstitute({
      id: OrderId.of(firstRow.id),
      customerId: firstRow.customer_id,
      shippingAddress: Address.of({
        street: firstRow.shipping_street,
        city: firstRow.shipping_city,
        province: firstRow.shipping_province,
        postalCode: firstRow.shipping_postal_code,
        country: firstRow.shipping_country,
      }),
      status: firstRow.status as OrderStatus,
      items,
      currency: firstRow.currency,
      createdAt: firstRow.created_at,
      updatedAt: firstRow.updated_at,
    });
  }

  private groupRowsByOrderId(rows: any[]): any[][] {
    const groups = new Map<string, any[]>();
    for (const row of rows) {
      const orderId = row.id;
      if (!groups.has(orderId)) {
        groups.set(orderId, []);
      }
      groups.get(orderId)!.push(row);
    }
    return Array.from(groups.values());
  }
}
```

---

## 7. Application Services (Use Cases)

Application Service จัดการ Use Cases และประสานงานระหว่าง Domain Objects

```typescript
// src/application/use-cases/create-order.use-case.ts
import { OrderRepository } from '../../domain/repositories/order.repository';
import { Order } from '../../domain/aggregates/order.aggregate';
import { Address } from '../../domain/value-objects/address.vo';
import { Money } from '../../domain/value-objects/money.vo';

export interface CreateOrderCommand {
  customerId: string;
  shippingAddress: {
    street: string;
    city: string;
    province: string;
    postalCode: string;
    country: string;
  };
  items: Array<{
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: {
      amount: number;
      currency: string;
    };
  }>;
}

export interface CreateOrderResult {
  orderId: string;
  totalAmount: {
    amount: number;
    currency: string;
  };
  status: string;
}

export class CreateOrderUseCase {
  constructor(private readonly orderRepository: OrderRepository) {}

  async execute(command: CreateOrderCommand): Promise<CreateOrderResult> {
    // Validate command
    if (!command.customerId) {
      throw new Error('Customer ID is required');
    }
    if (!command.items || command.items.length === 0) {
      throw new Error('Order must have at least one item');
    }

    // Create Address Value Object
    const shippingAddress = Address.of(command.shippingAddress);

    // Determine currency from first item
    const currency = command.items[0].unitPrice.currency;

    // Create Order Aggregate
    const order = Order.create(command.customerId, shippingAddress, currency);

    // Add items
    for (const item of command.items) {
      const unitPrice = Money.of(item.unitPrice.amount, item.unitPrice.currency);
      order.addItem(
        item.productId,
        item.productName,
        item.quantity,
        unitPrice
      );
    }

    // Confirm the order
    order.confirm();

    // Persist
    await this.orderRepository.save(order);

    return {
      orderId: order.id.value,
      totalAmount: order.totalAmount.toJSON(),
      status: order.status,
    };
  }
}
```

```typescript
// src/application/use-cases/cancel-order.use-case.ts
import { OrderRepository } from '../../domain/repositories/order.repository';
import { OrderId } from '../../domain/value-objects/order-id.vo';

export interface CancelOrderCommand {
  orderId: string;
  customerId: string;
  reason: string;
}

export class CancelOrderUseCase {
  constructor(private readonly orderRepository: OrderRepository) {}

  async execute(command: CancelOrderCommand): Promise<void> {
    const orderId = OrderId.of(command.orderId);
    const order = await this.orderRepository.findById(orderId);

    if (!order) {
      throw new Error(`Order ${command.orderId} not found`);
    }

    // Authorization check
    if (order.customerId !== command.customerId) {
      throw new Error('You can only cancel your own orders');
    }

    order.cancel(command.reason);
    await this.orderRepository.save(order);
  }
}
```

---

## 8. Domain Services

Domain Service จัดการ Business Logic ที่ไม่ควรอยู่ใน Entity ใด Entity หนึ่ง

```typescript
// src/domain/services/pricing.domain-service.ts
import { Money } from '../value-objects/money.vo';
import { OrderItem } from '../entities/order-item.entity';

export interface DiscountPolicy {
  code: string;
  type: 'PERCENTAGE' | 'FIXED';
  value: number;
  minimumOrderAmount?: number;
}

export class PricingDomainService {
  calculateDiscount(
    items: ReadonlyArray<OrderItem>,
    policy: DiscountPolicy,
    currency: string
  ): Money {
    const subtotal = items.reduce(
      (sum, item) => sum.add(item.totalPrice),
      Money.zero(currency)
    );

    // Check minimum order amount
    if (policy.minimumOrderAmount) {
      const minimum = Money.of(policy.minimumOrderAmount, currency);
      if (subtotal.isLessThan(minimum)) {
        return Money.zero(currency);
      }
    }

    if (policy.type === 'PERCENTAGE') {
      return subtotal.multiply(policy.value / 100);
    } else {
      const discount = Money.of(policy.value, currency);
      // Discount cannot exceed order total
      if (discount.isGreaterThan(subtotal)) {
        return subtotal;
      }
      return discount;
    }
  }

  calculateShipping(
    items: ReadonlyArray<OrderItem>,
    destination: string,
    currency: string
  ): Money {
    const totalWeight = items.length * 0.5; // Simplified: 0.5 kg per item
    const baseRate = Money.of(50, currency); // 50 THB base

    if (destination === 'TH') {
      // Domestic shipping
      if (totalWeight <= 1) return baseRate;
      return baseRate.add(Money.of((totalWeight - 1) * 20, currency));
    } else {
      // International shipping
      return Money.of(500 + totalWeight * 100, currency);
    }
  }
}
```

---

## 9. CQRS Pattern

```typescript
// src/application/commands/command-bus.ts
export interface Command {
  readonly commandType: string;
}

export interface CommandHandler<TCommand extends Command, TResult = void> {
  handle(command: TCommand): Promise<TResult>;
}

export class CommandBus {
  private handlers = new Map<string, CommandHandler<any, any>>();

  register<TCommand extends Command>(
    commandType: string,
    handler: CommandHandler<TCommand, any>
  ): void {
    this.handlers.set(commandType, handler);
  }

  async execute<TResult>(command: Command): Promise<TResult> {
    const handler = this.handlers.get(command.commandType);
    if (!handler) {
      throw new Error(`No handler registered for command: ${command.commandType}`);
    }
    return handler.handle(command);
  }
}
```

```typescript
// src/application/queries/query-bus.ts
export interface Query {
  readonly queryType: string;
}

export interface QueryHandler<TQuery extends Query, TResult> {
  handle(query: TQuery): Promise<TResult>;
}

export class QueryBus {
  private handlers = new Map<string, QueryHandler<any, any>>();

  register<TQuery extends Query, TResult>(
    queryType: string,
    handler: QueryHandler<TQuery, TResult>
  ): void {
    this.handlers.set(queryType, handler);
  }

  async execute<TResult>(query: Query): Promise<TResult> {
    const handler = this.handlers.get(query.queryType);
    if (!handler) {
      throw new Error(`No handler registered for query: ${query.queryType}`);
    }
    return handler.handle(query);
  }
}
```

```typescript
// src/application/queries/get-order-by-id.query.ts
import { Pool } from 'pg';
import { Query, QueryHandler } from './query-bus';

export interface GetOrderByIdQuery extends Query {
  queryType: 'GET_ORDER_BY_ID';
  orderId: string;
  customerId?: string;
}

export interface OrderReadModel {
  id: string;
  customerId: string;
  status: string;
  currency: string;
  totalAmount: number;
  itemCount: number;
  shippingAddress: {
    street: string;
    city: string;
    province: string;
    postalCode: string;
    country: string;
  };
  items: Array<{
    id: string;
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
    totalPrice: number;
  }>;
  createdAt: string;
  updatedAt: string;
}

export class GetOrderByIdQueryHandler
  implements QueryHandler<GetOrderByIdQuery, OrderReadModel | null>
{
  constructor(private readonly pool: Pool) {}

  async handle(query: GetOrderByIdQuery): Promise<OrderReadModel | null> {
    const result = await this.pool.query(
      `SELECT 
        o.id,
        o.customer_id,
        o.status,
        o.currency,
        o.shipping_street,
        o.shipping_city,
        o.shipping_province,
        o.shipping_postal_code,
        o.shipping_country,
        o.created_at,
        o.updated_at,
        COALESCE(json_agg(
          json_build_object(
            'id', oi.id,
            'productId', oi.product_id,
            'productName', oi.product_name,
            'quantity', oi.quantity,
            'unitPrice', oi.unit_price_amount,
            'totalPrice', oi.quantity * oi.unit_price_amount
          )
        ) FILTER (WHERE oi.id IS NOT NULL), '[]') as items,
        SUM(COALESCE(oi.quantity * oi.unit_price_amount, 0)) as total_amount,
        SUM(COALESCE(oi.quantity, 0)) as item_count
       FROM orders o
       LEFT JOIN order_items oi ON oi.order_id = o.id
       WHERE o.id = $1 AND o.deleted_at IS NULL
       GROUP BY o.id`,
      [query.orderId]
    );

    if (result.rows.length === 0) return null;

    const row = result.rows[0];

    // Authorization
    if (query.customerId && row.customer_id !== query.customerId) {
      return null;
    }

    return {
      id: row.id,
      customerId: row.customer_id,
      status: row.status,
      currency: row.currency,
      totalAmount: parseFloat(row.total_amount || 0),
      itemCount: parseInt(row.item_count || 0),
      shippingAddress: {
        street: row.shipping_street,
        city: row.shipping_city,
        province: row.shipping_province,
        postalCode: row.shipping_postal_code,
        country: row.shipping_country,
      },
      items: row.items,
      createdAt: row.created_at.toISOString(),
      updatedAt: row.updated_at.toISOString(),
    };
  }
}
```

---

## 10. Domain Event Publisher

```typescript
// src/infrastructure/events/domain-event-publisher.ts
import { DomainEvent } from '../../domain/events/domain-event';
import { Pool } from 'pg';

interface EventHandler {
  (event: DomainEvent): Promise<void>;
}

export class DomainEventPublisher {
  private handlers = new Map<string, EventHandler[]>();

  subscribe(eventType: string, handler: EventHandler): void {
    if (!this.handlers.has(eventType)) {
      this.handlers.set(eventType, []);
    }
    this.handlers.get(eventType)!.push(handler);
  }

  async publish(event: DomainEvent): Promise<void> {
    const eventHandlers = this.handlers.get(event.eventType) || [];
    await Promise.all(eventHandlers.map((handler) => handler(event)));
  }
}
```

```typescript
// src/infrastructure/events/outbox-event-publisher.ts
// Outbox Pattern สำหรับ Reliable Event Publishing
import { Pool } from 'pg';
import { DomainEvent } from '../../domain/events/domain-event';

export class OutboxEventPublisher {
  constructor(private readonly pool: Pool) {}

  async saveToOutbox(event: DomainEvent, client: any): Promise<void> {
    await client.query(
      `INSERT INTO outbox_events (
        id, event_type, payload, created_at, status
      ) VALUES ($1, $2, $3, NOW(), 'PENDING')`,
      [event.eventId, event.eventType, JSON.stringify(event)]
    );
  }

  async processOutboxEvents(
    publisher: (event: any) => Promise<void>
  ): Promise<void> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      // Lock rows for processing
      const result = await client.query(
        `SELECT * FROM outbox_events 
         WHERE status = 'PENDING' 
         ORDER BY created_at ASC 
         LIMIT 10
         FOR UPDATE SKIP LOCKED`
      );

      for (const row of result.rows) {
        try {
          await publisher(JSON.parse(row.payload));

          await client.query(
            `UPDATE outbox_events 
             SET status = 'PROCESSED', processed_at = NOW() 
             WHERE id = $1`,
            [row.id]
          );
        } catch (error) {
          await client.query(
            `UPDATE outbox_events 
             SET status = 'FAILED', error_message = $2
             WHERE id = $1`,
            [row.id, (error as Error).message]
          );
        }
      }

      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
}
```

---

## 11. Anti-Corruption Layer (ACL)

ACL แปลง Model จาก Context อื่นให้เป็น Model ใน Context ของเรา

```typescript
// src/infrastructure/acl/catalog-service.acl.ts
// แปลง Product จาก Catalog Service เป็น Product ใน Order Context

interface CatalogProductDto {
  productId: string;
  title: string;
  price: {
    value: number;
    currency: string;
  };
  stockQuantity: number;
  isActive: boolean;
}

interface OrderProduct {
  productId: string;
  productName: string;
  unitPrice: {
    amount: number;
    currency: string;
  };
  isAvailable: boolean;
}

export class CatalogServiceACL {
  constructor(private readonly catalogServiceUrl: string) {}

  async getProduct(productId: string): Promise<OrderProduct | null> {
    const response = await fetch(
      `${this.catalogServiceUrl}/products/${productId}`
    );

    if (response.status === 404) return null;
    if (!response.ok) {
      throw new Error(`Catalog service error: ${response.status}`);
    }

    const catalogProduct: CatalogProductDto = await response.json();
    return this.translate(catalogProduct);
  }

  private translate(catalogProduct: CatalogProductDto): OrderProduct {
    return {
      productId: catalogProduct.productId,
      productName: catalogProduct.title, // Map 'title' to 'productName'
      unitPrice: {
        amount: catalogProduct.price.value, // Map 'value' to 'amount'
        currency: catalogProduct.price.currency,
      },
      isAvailable: catalogProduct.isActive && catalogProduct.stockQuantity > 0,
    };
  }
}
```

---

## 12. Database Schema

```sql
-- migrations/001_create_orders_tables.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  customer_id VARCHAR(255) NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
  currency CHAR(3) NOT NULL DEFAULT 'THB',
  shipping_street VARCHAR(500) NOT NULL,
  shipping_city VARCHAR(200) NOT NULL,
  shipping_province VARCHAR(200) NOT NULL,
  shipping_postal_code VARCHAR(20) NOT NULL,
  shipping_country CHAR(2) NOT NULL DEFAULT 'TH',
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  deleted_at TIMESTAMP WITH TIME ZONE,
  
  CONSTRAINT orders_status_check CHECK (
    status IN ('DRAFT', 'CONFIRMED', 'PAID', 'SHIPPED', 'DELIVERED', 'CANCELLED')
  ),
  CONSTRAINT orders_currency_check CHECK (
    currency ~ '^[A-Z]{3}$'
  )
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id VARCHAR(255) NOT NULL,
  product_name VARCHAR(500) NOT NULL,
  quantity INTEGER NOT NULL CHECK (quantity > 0),
  unit_price_amount DECIMAL(12, 2) NOT NULL CHECK (unit_price_amount >= 0),
  unit_price_currency CHAR(3) NOT NULL DEFAULT 'THB',
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

CREATE TABLE outbox_events (
  id UUID PRIMARY KEY,
  event_type VARCHAR(100) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  processed_at TIMESTAMP WITH TIME ZONE,
  error_message TEXT,
  
  CONSTRAINT outbox_status_check CHECK (
    status IN ('PENDING', 'PROCESSED', 'FAILED')
  )
);

CREATE INDEX idx_outbox_status ON outbox_events(status, created_at);
```

---

## 13. HTTP Controller

```typescript
// src/interfaces/http/order.controller.ts
import express, { Request, Response, NextFunction } from 'express';
import { CommandBus } from '../../application/commands/command-bus';
import { QueryBus } from '../../application/queries/query-bus';
import { CreateOrderUseCase } from '../../application/use-cases/create-order.use-case';
import { CancelOrderUseCase } from '../../application/use-cases/cancel-order.use-case';
import { GetOrderByIdQueryHandler } from '../../application/queries/get-order-by-id.query';

const router = express.Router();

// POST /orders - Create order
router.post('/', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const { customerId, shippingAddress, items } = req.body;

    if (!customerId || !shippingAddress || !items) {
      return res.status(400).json({
        error: 'customerId, shippingAddress, and items are required',
      });
    }

    const useCase: CreateOrderUseCase = req.app.locals.createOrderUseCase;
    const result = await useCase.execute({
      customerId,
      shippingAddress,
      items,
    });

    res.status(201).json(result);
  } catch (error) {
    next(error);
  }
});

// GET /orders/:id
router.get('/:id', async (req: Request, res: Response, next: NextFunction) => {
  try {
    const queryHandler: GetOrderByIdQueryHandler =
      req.app.locals.getOrderByIdQueryHandler;

    const order = await queryHandler.handle({
      queryType: 'GET_ORDER_BY_ID',
      orderId: req.params.id,
      customerId: req.headers['x-customer-id'] as string,
    });

    if (!order) {
      return res.status(404).json({ error: 'Order not found' });
    }

    res.json(order);
  } catch (error) {
    next(error);
  }
});

// POST /orders/:id/cancel
router.post(
  '/:id/cancel',
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { reason } = req.body;
      const customerId = req.headers['x-customer-id'] as string;

      const useCase: CancelOrderUseCase = req.app.locals.cancelOrderUseCase;
      await useCase.execute({
        orderId: req.params.id,
        customerId,
        reason: reason || 'Customer requested cancellation',
      });

      res.json({ message: 'Order cancelled successfully' });
    } catch (error) {
      next(error);
    }
  }
);

export default router;
```

---

## 14. Dependency Injection Container

```typescript
// src/infrastructure/container.ts
import { Pool } from 'pg';
import { PostgresOrderRepository } from './repositories/postgres-order.repository';
import { DomainEventPublisher } from './events/domain-event-publisher';
import { CreateOrderUseCase } from '../application/use-cases/create-order.use-case';
import { CancelOrderUseCase } from '../application/use-cases/cancel-order.use-case';
import { GetOrderByIdQueryHandler } from '../application/queries/get-order-by-id.query';
import { OutboxEventPublisher } from './events/outbox-event-publisher';

export function createContainer(pool: Pool) {
  const eventPublisher = new DomainEventPublisher();
  const outboxPublisher = new OutboxEventPublisher(pool);
  const orderRepository = new PostgresOrderRepository(pool, eventPublisher);

  const createOrderUseCase = new CreateOrderUseCase(orderRepository);
  const cancelOrderUseCase = new CancelOrderUseCase(orderRepository);
  const getOrderByIdQueryHandler = new GetOrderByIdQueryHandler(pool);

  return {
    eventPublisher,
    outboxPublisher,
    orderRepository,
    createOrderUseCase,
    cancelOrderUseCase,
    getOrderByIdQueryHandler,
  };
}
```

---

## 15. Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  order-service:
    build:
      context: ./order-service
      dockerfile: Dockerfile
    ports:
      - "3001:3000"
    environment:
      NODE_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: orders_db
      DB_USER: orders_user
      DB_PASSWORD: orders_pass
      RABBITMQ_URL: amqp://rabbitmq:5672
      CATALOG_SERVICE_URL: http://catalog-service:3000
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  catalog-service:
    build:
      context: ./catalog-service
      dockerfile: Dockerfile
    ports:
      - "3002:3000"
    environment:
      NODE_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: catalog_db
      DB_USER: catalog_user
      DB_PASSWORD: catalog_pass
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres_pass
      POSTGRES_MULTIPLE_DATABASES: orders_db,catalog_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/create-multiple-databases.sh:/docker-entrypoint-initdb.d/create-multiple-databases.sh
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin_pass
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  postgres_data:
  rabbitmq_data:
```

---

## 16. Tests

```typescript
// src/domain/__tests__/order.aggregate.test.ts
import { Order, OrderStatus } from '../aggregates/order.aggregate';
import { Address } from '../value-objects/address.vo';
import { Money } from '../value-objects/money.vo';
import { OrderCreatedEvent } from '../events/order-created.event';
import { OrderConfirmedEvent } from '../events/order-confirmed.event';

describe('Order Aggregate', () => {
  const validAddress = Address.of({
    street: '123 Main St',
    city: 'Bangkok',
    province: 'Bangkok',
    postalCode: '10110',
    country: 'TH',
  });

  const customerId = 'customer-123';

  describe('Order.create()', () => {
    it('should create a DRAFT order with OrderCreatedEvent', () => {
      const order = Order.create(customerId, validAddress, 'THB');

      expect(order.status).toBe(OrderStatus.DRAFT);
      expect(order.customerId).toBe(customerId);
      expect(order.items).toHaveLength(0);
      expect(order.domainEvents).toHaveLength(1);
      expect(order.domainEvents[0]).toBeInstanceOf(OrderCreatedEvent);
    });
  });

  describe('addItem()', () => {
    it('should add item to DRAFT order', () => {
      const order = Order.create(customerId, validAddress, 'THB');
      const price = Money.of(100, 'THB');

      order.addItem('prod-1', 'Product 1', 2, price);

      expect(order.items).toHaveLength(1);
      expect(order.items[0].quantity).toBe(2);
      expect(order.totalAmount.amount).toBe(200);
    });

    it('should aggregate quantity for same product', () => {
      const order = Order.create(customerId, validAddress, 'THB');
      const price = Money.of(100, 'THB');

      order.addItem('prod-1', 'Product 1', 2, price);
      order.addItem('prod-1', 'Product 1', 3, price);

      expect(order.items).toHaveLength(1);
      expect(order.items[0].quantity).toBe(5);
    });

    it('should throw error when adding to non-DRAFT order', () => {
      const order = Order.create(customerId, validAddress, 'THB');
      order.addItem('prod-1', 'Product 1', 1, Money.of(100, 'THB'));
      order.confirm();

      expect(() =>
        order.addItem('prod-2', 'Product 2', 1, Money.of(200, 'THB'))
      ).toThrow('Can only add items to DRAFT orders');
    });
  });

  describe('confirm()', () => {
    it('should confirm a DRAFT order with items', () => {
      const order = Order.create(customerId, validAddress, 'THB');
      order.addItem('prod-1', 'Product 1', 1, Money.of(100, 'THB'));
      order.clearDomainEvents();

      order.confirm();

      expect(order.status).toBe(OrderStatus.CONFIRMED);
      expect(order.domainEvents).toHaveLength(1);
      expect(order.domainEvents[0]).toBeInstanceOf(OrderConfirmedEvent);
    });

    it('should throw error when confirming empty order', () => {
      const order = Order.create(customerId, validAddress, 'THB');
      expect(() => order.confirm()).toThrow('Cannot confirm an empty order');
    });
  });

  describe('cancel()', () => {
    it('should cancel a CONFIRMED order', () => {
      const order = Order.create(customerId, validAddress, 'THB');
      order.addItem('prod-1', 'Product 1', 1, Money.of(100, 'THB'));
      order.confirm();

      order.cancel('Changed my mind');

      expect(order.status).toBe(OrderStatus.CANCELLED);
    });

    it('should not cancel a SHIPPED order', () => {
      const order = Order.create(customerId, validAddress, 'THB');
      order.addItem('prod-1', 'Product 1', 1, Money.of(100, 'THB'));
      order.confirm();
      order.markAsPaid();
      order.ship();

      expect(() => order.cancel('Too late')).toThrow(
        'Cannot cancel shipped or delivered orders'
      );
    });
  });
});
```

---

## สรุป

| แนวคิด | คำอธิบาย | ตัวอย่างในโค้ด |
|--------|----------|---------------|
| **Bounded Context** | ขอบเขตของ Domain Model ใน Context หนึ่งๆ | Order, Catalog, Payment Contexts |
| **Value Object** | Object ที่กำหนดด้วยค่า ไม่มี Identity | `Money`, `Email`, `Address` |
| **Entity** | Object ที่มี Identity สามารถเปลี่ยนสถานะได้ | `OrderItem` |
| **Aggregate** | กลุ่ม Entities ที่จัดการเป็นหน่วยเดียว | `Order` (Aggregate Root) |
| **Domain Event** | เหตุการณ์สำคัญใน Business Domain | `OrderCreatedEvent`, `OrderConfirmedEvent` |
| **Repository** | Interface สำหรับ Persist/Retrieve Aggregates | `OrderRepository` |
| **Domain Service** | Business Logic ที่ไม่ควรอยู่ใน Entity ใดๆ | `PricingDomainService` |
| **Application Service** | จัดการ Use Cases, ประสานงาน Domain Objects | `CreateOrderUseCase` |
| **CQRS** | แยก Command และ Query ออกจากกัน | `CommandBus`, `QueryBus` |
| **ACL** | แปลง Model จาก External Context | `CatalogServiceACL` |
| **Outbox Pattern** | Reliable Event Publishing พร้อมกับ DB Transaction | `OutboxEventPublisher` |

# Part 66: Advanced TypeScript Patterns สำหรับ Microservices

## บทนำ

ใน Part นี้เราจะเรียนรู้ TypeScript patterns ขั้นสูงที่ช่วยให้โค้ด Microservices มีความแข็งแกร่ง ปลอดภัย และบำรุงรักษาได้ง่ายขึ้น โดยเน้นที่ patterns ที่ใช้งานจริงใน production

---

## 1. Branded Types

Branded types ช่วยป้องกัน type confusion ระหว่าง primitive types ที่มี semantic ต่างกัน เช่น UserId และ OrderId ซึ่งทั้งคู่เป็น string แต่ไม่ควรนำมาใช้แทนกัน

### ปัญหาที่แก้ไข

```typescript
// ❌ ไม่มี branding - compiler ไม่แจ้ง error
function getUser(id: string) { /* ... */ }
function getOrder(id: string) { /* ... */ }

const userId = "user-123";
const orderId = "order-456";

getUser(orderId);  // ไม่มี error แต่ผิด logic!
getOrder(userId);  // ไม่มี error แต่ผิด logic!
```

### การสร้าง Branded Types

```typescript
// branded-types.ts
declare const __brand: unique symbol;

type Brand<T, TBrand> = T & { readonly [__brand]: TBrand };

// Define branded types
export type UserId = Brand<string, "UserId">;
export type OrderId = Brand<string, "OrderId">;
export type ProductId = Brand<string, "ProductId">;
export type Email = Brand<string, "Email">;
export type Amount = Brand<number, "Amount">;
export type Timestamp = Brand<number, "Timestamp">;

// Constructor functions พร้อม validation
export function createUserId(id: string): UserId {
  if (!id.startsWith("user-")) {
    throw new Error(`Invalid UserId format: ${id}`);
  }
  return id as UserId;
}

export function createOrderId(id: string): OrderId {
  if (!id.startsWith("order-")) {
    throw new Error(`Invalid OrderId format: ${id}`);
  }
  return id as OrderId;
}

export function createEmail(email: string): Email {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(email)) {
    throw new Error(`Invalid email format: ${email}`);
  }
  return email as Email;
}

export function createAmount(value: number): Amount {
  if (value < 0) {
    throw new Error(`Amount cannot be negative: ${value}`);
  }
  return value as Amount;
}

// ✅ Type-safe functions
function getUser(id: UserId) { /* ... */ }
function getOrder(id: OrderId) { /* ... */ }

const userId = createUserId("user-123");
const orderId = createOrderId("order-456");

getUser(userId);   // ✅ ถูกต้อง
getOrder(orderId); // ✅ ถูกต้อง
// getUser(orderId); // ❌ TypeScript error!
// getOrder(userId); // ❌ TypeScript error!
```

### Branded Types ใน Domain Model

```typescript
// domain/user.ts
import { UserId, Email, createUserId, createEmail } from "./branded-types";

interface UserProps {
  id: UserId;
  email: Email;
  name: string;
  createdAt: Date;
}

export class User {
  private readonly props: UserProps;

  private constructor(props: UserProps) {
    this.props = props;
  }

  static create(params: {
    id: string;
    email: string;
    name: string;
  }): User {
    return new User({
      id: createUserId(params.id),
      email: createEmail(params.email),
      name: params.name,
      createdAt: new Date(),
    });
  }

  get id(): UserId { return this.props.id; }
  get email(): Email { return this.props.email; }
  get name(): string { return this.props.name; }
}

// domain/order.ts
import { OrderId, UserId, ProductId, Amount } from "./branded-types";

interface OrderLineItem {
  productId: ProductId;
  quantity: number;
  unitPrice: Amount;
}

interface OrderProps {
  id: OrderId;
  userId: UserId;
  items: OrderLineItem[];
  totalAmount: Amount;
  status: "pending" | "confirmed" | "shipped" | "delivered";
}

export class Order {
  private readonly props: OrderProps;

  private constructor(props: OrderProps) {
    this.props = props;
  }

  static create(params: {
    id: string;
    userId: string;
    items: Array<{ productId: string; quantity: number; unitPrice: number }>;
  }): Order {
    const items = params.items.map((item) => ({
      productId: item.productId as ProductId,
      quantity: item.quantity,
      unitPrice: item.unitPrice as Amount,
    }));

    const totalAmount = items.reduce(
      (sum, item) => (sum + item.unitPrice * item.quantity) as Amount,
      0 as Amount
    );

    return new Order({
      id: params.id as OrderId,
      userId: params.userId as UserId,
      items,
      totalAmount,
      status: "pending",
    });
  }

  get id(): OrderId { return this.props.id; }
  get userId(): UserId { return this.props.userId; }
  get totalAmount(): Amount { return this.props.totalAmount; }
}
```

---

## 2. Result Type Pattern

Result type ช่วยจัดการ error handling แบบ explicit โดยไม่ต้องใช้ try/catch และทำให้ flow ของ error ชัดเจน

### การสร้าง Result Type

```typescript
// result.ts
export type Result<T, E = Error> = Ok<T, E> | Err<T, E>;

export class Ok<T, E> {
  readonly _tag = "Ok" as const;
  constructor(readonly value: T) {}

  isOk(): this is Ok<T, E> { return true; }
  isErr(): this is Err<T, E> { return false; }

  map<U>(fn: (value: T) => U): Result<U, E> {
    return new Ok(fn(this.value));
  }

  mapErr<F>(fn: (error: E) => F): Result<T, F> {
    return new Ok(this.value);
  }

  flatMap<U>(fn: (value: T) => Result<U, E>): Result<U, E> {
    return fn(this.value);
  }

  getOrElse(_defaultValue: T): T {
    return this.value;
  }

  getOrThrow(): T {
    return this.value;
  }

  match<U>(patterns: { Ok: (value: T) => U; Err: (error: E) => U }): U {
    return patterns.Ok(this.value);
  }
}

export class Err<T, E> {
  readonly _tag = "Err" as const;
  constructor(readonly error: E) {}

  isOk(): this is Ok<T, E> { return false; }
  isErr(): this is Err<T, E> { return true; }

  map<U>(_fn: (value: T) => U): Result<U, E> {
    return new Err(this.error);
  }

  mapErr<F>(fn: (error: E) => F): Result<T, F> {
    return new Err(fn(this.error));
  }

  flatMap<U>(_fn: (value: T) => Result<U, E>): Result<U, E> {
    return new Err(this.error);
  }

  getOrElse(defaultValue: T): T {
    return defaultValue;
  }

  getOrThrow(): never {
    throw this.error;
  }

  match<U>(patterns: { Ok: (value: T) => U; Err: (error: E) => U }): U {
    return patterns.Err(this.error);
  }
}

// Helper functions
export const ok = <T>(value: T): Ok<T, never> => new Ok(value);
export const err = <E>(error: E): Err<never, E> => new Err(error);

// Combine multiple results
export function combine<T extends Result<unknown, unknown>[]>(
  results: [...T]
): Result<
  { [K in keyof T]: T[K] extends Result<infer V, unknown> ? V : never },
  T[number] extends Result<unknown, infer E> ? E : never
> {
  const values: unknown[] = [];
  for (const result of results) {
    if (result.isErr()) {
      return result as any;
    }
    values.push((result as Ok<unknown, unknown>).value);
  }
  return ok(values as any);
}
```

### การใช้งาน Result Type ใน Service

```typescript
// services/user.service.ts
import { Result, ok, err } from "../result";
import { User } from "../domain/user";
import { UserId, Email } from "../domain/branded-types";

// Domain errors
type UserNotFoundError = { _tag: "UserNotFoundError"; id: string };
type EmailAlreadyExistsError = { _tag: "EmailAlreadyExistsError"; email: string };
type ValidationError = { _tag: "ValidationError"; message: string };
type DatabaseError = { _tag: "DatabaseError"; cause: Error };

type UserServiceError =
  | UserNotFoundError
  | EmailAlreadyExistsError
  | ValidationError
  | DatabaseError;

interface UserRepository {
  findById(id: UserId): Promise<User | null>;
  findByEmail(email: Email): Promise<User | null>;
  save(user: User): Promise<void>;
}

export class UserService {
  constructor(private readonly userRepo: UserRepository) {}

  async createUser(params: {
    id: string;
    email: string;
    name: string;
  }): Promise<Result<User, UserServiceError>> {
    // Validate
    if (!params.name || params.name.trim().length < 2) {
      return err<ValidationError>({
        _tag: "ValidationError",
        message: "Name must be at least 2 characters",
      });
    }

    // Check email uniqueness
    let existingUser: User | null;
    try {
      existingUser = await this.userRepo.findByEmail(params.email as Email);
    } catch (e) {
      return err<DatabaseError>({
        _tag: "DatabaseError",
        cause: e as Error,
      });
    }

    if (existingUser) {
      return err<EmailAlreadyExistsError>({
        _tag: "EmailAlreadyExistsError",
        email: params.email,
      });
    }

    // Create and save
    let user: User;
    try {
      user = User.create(params);
      await this.userRepo.save(user);
    } catch (e) {
      return err<DatabaseError>({
        _tag: "DatabaseError",
        cause: e as Error,
      });
    }

    return ok(user);
  }

  async getUserById(id: string): Promise<Result<User, UserServiceError>> {
    let user: User | null;
    try {
      user = await this.userRepo.findById(id as UserId);
    } catch (e) {
      return err<DatabaseError>({
        _tag: "DatabaseError",
        cause: e as Error,
      });
    }

    if (!user) {
      return err<UserNotFoundError>({
        _tag: "UserNotFoundError",
        id,
      });
    }

    return ok(user);
  }
}

// controllers/user.controller.ts
import { Request, Response } from "express";
import { UserService } from "../services/user.service";

export class UserController {
  constructor(private readonly userService: UserService) {}

  async createUser(req: Request, res: Response): Promise<void> {
    const result = await this.userService.createUser(req.body);

    result.match({
      Ok: (user) => {
        res.status(201).json({
          id: user.id,
          email: user.email,
          name: user.name,
        });
      },
      Err: (error) => {
        switch (error._tag) {
          case "ValidationError":
            res.status(400).json({ error: error.message });
            break;
          case "EmailAlreadyExistsError":
            res.status(409).json({ error: `Email ${error.email} already exists` });
            break;
          case "DatabaseError":
            res.status(500).json({ error: "Internal server error" });
            break;
          default:
            res.status(500).json({ error: "Unknown error" });
        }
      },
    });
  }
}
```

---

## 3. Option/Maybe Monad

Option/Maybe monad ช่วยจัดการค่า nullable อย่างปลอดภัยโดยไม่ต้องตรวจสอบ null/undefined ทุกครั้ง

```typescript
// option.ts
export type Option<T> = Some<T> | None;

export class Some<T> {
  readonly _tag = "Some" as const;
  constructor(readonly value: T) {}

  isSome(): this is Some<T> { return true; }
  isNone(): this is None { return false; }

  map<U>(fn: (value: T) => U): Option<U> {
    return new Some(fn(this.value));
  }

  flatMap<U>(fn: (value: T) => Option<U>): Option<U> {
    return fn(this.value);
  }

  filter(predicate: (value: T) => boolean): Option<T> {
    return predicate(this.value) ? this : none;
  }

  getOrElse(_defaultValue: T): T {
    return this.value;
  }

  getOrNull(): T | null {
    return this.value;
  }

  toResult<E>(error: E): import("./result").Result<T, E> {
    return new (require("./result").Ok)(this.value);
  }

  match<U>(patterns: { Some: (value: T) => U; None: () => U }): U {
    return patterns.Some(this.value);
  }
}

export class None {
  readonly _tag = "None" as const;

  isSome(): this is Some<never> { return false; }
  isNone(): this is None { return true; }

  map<U>(_fn: (value: never) => U): Option<U> {
    return none;
  }

  flatMap<U>(_fn: (value: never) => Option<U>): Option<U> {
    return none;
  }

  filter(_predicate: (value: never) => boolean): Option<never> {
    return none;
  }

  getOrElse<T>(defaultValue: T): T {
    return defaultValue;
  }

  getOrNull(): null {
    return null;
  }

  toResult<T, E>(error: E): import("./result").Result<T, E> {
    return new (require("./result").Err)(error);
  }

  match<U>(patterns: { Some: (value: never) => U; None: () => U }): U {
    return patterns.None();
  }
}

export const none = new None();
export const some = <T>(value: T): Some<T> => new Some(value);
export const fromNullable = <T>(value: T | null | undefined): Option<T> =>
  value == null ? none : some(value);

// การใช้งาน
interface Config {
  database: {
    host?: string;
    port?: number;
    maxConnections?: number;
  };
}

function getDbMaxConnections(config: Config): number {
  return fromNullable(config.database)
    .flatMap((db) => fromNullable(db.maxConnections))
    .filter((n) => n > 0 && n <= 100)
    .getOrElse(10); // default value
}

// Repository ที่ใช้ Option
class ProductRepository {
  async findById(id: string): Promise<Option<Product>> {
    const row = await db.query("SELECT * FROM products WHERE id = $1", [id]);
    return fromNullable(row.rows[0]).map((row) => Product.fromRow(row));
  }

  async findBySlug(slug: string): Promise<Option<Product>> {
    const row = await db.query("SELECT * FROM products WHERE slug = $1", [slug]);
    return fromNullable(row.rows[0]).map((row) => Product.fromRow(row));
  }
}

// Service ที่ใช้ Option pipeline
class ProductService {
  async getProductWithCategory(productId: string): Promise<Option<ProductWithCategory>> {
    const productOpt = await this.productRepo.findById(productId);

    return productOpt.flatMap(async (product) => {
      const categoryOpt = await this.categoryRepo.findById(product.categoryId);
      return categoryOpt.map((category) => ({
        ...product,
        category,
      }));
    });
  }
}
```

---

## 4. Dependency Injection ด้วย tsyringe

```typescript
// tsconfig.json (เพิ่ม)
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

```bash
npm install tsyringe reflect-metadata
```

```typescript
// container.ts
import "reflect-metadata";
import { container } from "tsyringe";

// Tokens สำหรับ interfaces
export const TOKENS = {
  UserRepository: Symbol("UserRepository"),
  OrderRepository: Symbol("OrderRepository"),
  EmailService: Symbol("EmailService"),
  Logger: Symbol("Logger"),
  Config: Symbol("Config"),
  EventBus: Symbol("EventBus"),
  Cache: Symbol("Cache"),
} as const;

// interfaces/user-repository.interface.ts
export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

// interfaces/email-service.interface.ts
export interface IEmailService {
  sendWelcomeEmail(email: string, name: string): Promise<void>;
  sendPasswordResetEmail(email: string, token: string): Promise<void>;
}

// interfaces/logger.interface.ts
export interface ILogger {
  info(message: string, context?: Record<string, unknown>): void;
  error(message: string, error?: Error, context?: Record<string, unknown>): void;
  warn(message: string, context?: Record<string, unknown>): void;
  debug(message: string, context?: Record<string, unknown>): void;
}

// implementations/postgres-user.repository.ts
import { injectable } from "tsyringe";
import { Pool } from "pg";
import { IUserRepository } from "../interfaces";

@injectable()
export class PostgresUserRepository implements IUserRepository {
  constructor(private readonly pool: Pool) {}

  async findById(id: string): Promise<User | null> {
    const result = await this.pool.query(
      "SELECT * FROM users WHERE id = $1 AND deleted_at IS NULL",
      [id]
    );
    return result.rows[0] ? User.fromRow(result.rows[0]) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const result = await this.pool.query(
      "SELECT * FROM users WHERE email = $1 AND deleted_at IS NULL",
      [email]
    );
    return result.rows[0] ? User.fromRow(result.rows[0]) : null;
  }

  async save(user: User): Promise<void> {
    await this.pool.query(
      `INSERT INTO users (id, email, name, created_at)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (id) DO UPDATE SET
         email = EXCLUDED.email,
         name = EXCLUDED.name`,
      [user.id, user.email, user.name, user.createdAt]
    );
  }

  async delete(id: string): Promise<void> {
    await this.pool.query(
      "UPDATE users SET deleted_at = NOW() WHERE id = $1",
      [id]
    );
  }
}

// services/user.service.ts
import { injectable, inject } from "tsyringe";
import { TOKENS } from "../container";
import { IUserRepository, IEmailService, ILogger } from "../interfaces";

@injectable()
export class UserService {
  constructor(
    @inject(TOKENS.UserRepository)
    private readonly userRepo: IUserRepository,

    @inject(TOKENS.EmailService)
    private readonly emailService: IEmailService,

    @inject(TOKENS.Logger)
    private readonly logger: ILogger
  ) {}

  async createUser(params: CreateUserParams): Promise<Result<User>> {
    this.logger.info("Creating user", { email: params.email });

    try {
      const existing = await this.userRepo.findByEmail(params.email);
      if (existing) {
        return err({ code: "EMAIL_EXISTS", email: params.email });
      }

      const user = User.create(params);
      await this.userRepo.save(user);
      await this.emailService.sendWelcomeEmail(user.email, user.name);

      this.logger.info("User created successfully", { userId: user.id });
      return ok(user);
    } catch (error) {
      this.logger.error("Failed to create user", error as Error);
      throw error;
    }
  }
}

// bootstrap/container.setup.ts
import { container } from "tsyringe";
import { Pool } from "pg";
import { TOKENS } from "../container";
import { PostgresUserRepository } from "../implementations";
import { SendGridEmailService } from "../implementations";
import { WinstonLogger } from "../implementations";

export function setupContainer(config: AppConfig): void {
  // Register infrastructure
  const pool = new Pool({
    host: config.database.host,
    port: config.database.port,
    database: config.database.name,
    user: config.database.user,
    password: config.database.password,
    max: config.database.maxConnections,
  });

  container.registerInstance(Pool, pool);

  // Register repositories
  container.register(TOKENS.UserRepository, {
    useClass: PostgresUserRepository,
  });

  // Register services
  container.register(TOKENS.EmailService, {
    useClass: SendGridEmailService,
  });

  // Register logger as singleton
  container.registerSingleton(TOKENS.Logger, WinstonLogger);
}

// app.ts
import "reflect-metadata";
import { container } from "tsyringe";
import { setupContainer } from "./bootstrap/container.setup";
import { UserService } from "./services/user.service";

setupContainer(config);

const userService = container.resolve(UserService);
```

---

## 5. Decorators สำหรับ Cross-cutting Concerns

```typescript
// decorators/log.decorator.ts
import { ILogger } from "../interfaces";

export function Log(logger?: ILogger) {
  return function (
    target: unknown,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: unknown[]) {
      const log = logger || (this as any).logger;
      const className = (target as any).constructor?.name || "Unknown";

      log?.info(`${className}.${propertyKey} called`, {
        args: args.map((a) => JSON.stringify(a)).join(", "),
      });

      const start = Date.now();
      try {
        const result = await originalMethod.apply(this, args);
        const duration = Date.now() - start;

        log?.info(`${className}.${propertyKey} completed`, {
          duration: `${duration}ms`,
        });

        return result;
      } catch (error) {
        const duration = Date.now() - start;
        log?.error(`${className}.${propertyKey} failed`, error as Error, {
          duration: `${duration}ms`,
        });
        throw error;
      }
    };

    return descriptor;
  };
}

// decorators/cache.decorator.ts
export function Cache(ttlSeconds: number = 60) {
  const cache = new Map<string, { value: unknown; expiresAt: number }>();

  return function (
    _target: unknown,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: unknown[]) {
      const cacheKey = `${propertyKey}:${JSON.stringify(args)}`;
      const now = Date.now();

      const cached = cache.get(cacheKey);
      if (cached && cached.expiresAt > now) {
        return cached.value;
      }

      const result = await originalMethod.apply(this, args);
      cache.set(cacheKey, {
        value: result,
        expiresAt: now + ttlSeconds * 1000,
      });

      return result;
    };

    return descriptor;
  };
}

// decorators/retry.decorator.ts
export interface RetryOptions {
  maxAttempts: number;
  delayMs: number;
  backoffMultiplier?: number;
  retryOn?: (error: Error) => boolean;
}

export function Retry(options: RetryOptions) {
  return function (
    _target: unknown,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: unknown[]) {
      let lastError: Error;
      let delay = options.delayMs;

      for (let attempt = 1; attempt <= options.maxAttempts; attempt++) {
        try {
          return await originalMethod.apply(this, args);
        } catch (error) {
          lastError = error as Error;

          const shouldRetry = options.retryOn
            ? options.retryOn(lastError)
            : true;

          if (!shouldRetry || attempt === options.maxAttempts) {
            throw lastError;
          }

          await new Promise((resolve) => setTimeout(resolve, delay));
          delay *= options.backoffMultiplier ?? 2;
        }
      }

      throw lastError!;
    };

    return descriptor;
  };
}

// decorators/validate.decorator.ts
import Joi from "joi";

export function Validate(schema: Joi.ObjectSchema) {
  return function (
    _target: unknown,
    _propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    const originalMethod = descriptor.value;

    descriptor.value = async function (...args: unknown[]) {
      const params = args[0];
      const { error } = schema.validate(params);

      if (error) {
        throw new ValidationError(error.details.map((d) => d.message).join(", "));
      }

      return originalMethod.apply(this, args);
    };

    return descriptor;
  };
}

// การใช้งาน decorators ร่วมกัน
const createUserSchema = Joi.object({
  email: Joi.string().email().required(),
  name: Joi.string().min(2).max(100).required(),
  password: Joi.string().min(8).required(),
});

@injectable()
export class UserService {
  @Log()
  @Validate(createUserSchema)
  @Retry({ maxAttempts: 3, delayMs: 100, backoffMultiplier: 2 })
  async createUser(params: CreateUserParams): Promise<User> {
    // ...
  }

  @Log()
  @Cache(300) // cache 5 minutes
  async getUserProfile(userId: string): Promise<UserProfile> {
    // ...
  }
}
```

---

## 6. Type-safe Event System

```typescript
// events/event-bus.ts

// Define event types
interface EventMap {
  "user.created": { userId: string; email: string; name: string };
  "user.updated": { userId: string; changes: Partial<User> };
  "user.deleted": { userId: string };
  "order.created": { orderId: string; userId: string; amount: number };
  "order.shipped": { orderId: string; trackingNumber: string };
  "payment.completed": { paymentId: string; orderId: string; amount: number };
  "payment.failed": { paymentId: string; orderId: string; reason: string };
}

type EventName = keyof EventMap;
type EventPayload<T extends EventName> = EventMap[T];
type EventHandler<T extends EventName> = (payload: EventPayload<T>) => Promise<void>;

interface DomainEvent<T extends EventName> {
  id: string;
  name: T;
  payload: EventPayload<T>;
  timestamp: Date;
  version: number;
}

export class TypedEventBus {
  private handlers: Map<string, EventHandler<any>[]> = new Map();

  on<T extends EventName>(event: T, handler: EventHandler<T>): () => void {
    const handlers = this.handlers.get(event) ?? [];
    handlers.push(handler);
    this.handlers.set(event, handlers);

    // Return unsubscribe function
    return () => {
      const current = this.handlers.get(event) ?? [];
      this.handlers.set(
        event,
        current.filter((h) => h !== handler)
      );
    };
  }

  async emit<T extends EventName>(
    event: T,
    payload: EventPayload<T>
  ): Promise<void> {
    const domainEvent: DomainEvent<T> = {
      id: crypto.randomUUID(),
      name: event,
      payload,
      timestamp: new Date(),
      version: 1,
    };

    const handlers = this.handlers.get(event) ?? [];
    await Promise.all(handlers.map((h) => h(domainEvent.payload)));
  }

  once<T extends EventName>(event: T, handler: EventHandler<T>): void {
    const unsubscribe = this.on(event, async (payload) => {
      unsubscribe();
      await handler(payload);
    });
  }
}

// Strongly-typed event handlers
const eventBus = new TypedEventBus();

// ✅ Type-safe - TypeScript รู้ว่า payload มี userId, email, name
eventBus.on("user.created", async (payload) => {
  console.log(`New user: ${payload.email}`);
  await sendWelcomeEmail(payload.email, payload.name);
});

// ✅ Type-safe - TypeScript รู้ว่า payload มี orderId, userId, amount
eventBus.on("order.created", async (payload) => {
  await processPayment(payload.orderId, payload.amount);
});

// emit ก็ type-safe เช่นกัน
await eventBus.emit("user.created", {
  userId: "user-123",
  email: "john@example.com",
  name: "John Doe",
});

// ❌ TypeScript error - missing required field
// await eventBus.emit("user.created", { userId: "123" });

// Event Sourcing integration
export class EventSourcedRepository<T extends AggregateRoot> {
  constructor(
    private readonly eventBus: TypedEventBus,
    private readonly eventStore: EventStore
  ) {}

  async save(aggregate: T): Promise<void> {
    const events = aggregate.getUncommittedEvents();

    // Store events
    await this.eventStore.appendEvents(aggregate.id, events);

    // Publish to event bus
    for (const event of events) {
      await this.eventBus.emit(event.name as EventName, event.payload);
    }

    aggregate.clearUncommittedEvents();
  }
}

// Outbox Pattern สำหรับ reliable event delivery
interface OutboxMessage {
  id: string;
  eventName: string;
  payload: unknown;
  createdAt: Date;
  processedAt?: Date;
  attempts: number;
}

export class OutboxEventPublisher {
  constructor(
    private readonly db: Pool,
    private readonly messageBroker: MessageBroker
  ) {}

  async publishPending(): Promise<void> {
    const messages = await this.db.query<OutboxMessage>(
      `SELECT * FROM outbox_messages
       WHERE processed_at IS NULL
         AND attempts < 3
       ORDER BY created_at ASC
       LIMIT 100`
    );

    for (const message of messages.rows) {
      try {
        await this.messageBroker.publish(message.eventName, message.payload);
        await this.db.query(
          "UPDATE outbox_messages SET processed_at = NOW() WHERE id = $1",
          [message.id]
        );
      } catch (error) {
        await this.db.query(
          "UPDATE outbox_messages SET attempts = attempts + 1 WHERE id = $1",
          [message.id]
        );
      }
    }
  }
}
```

---

## 7. Advanced Generic Patterns

```typescript
// patterns/repository.ts

// Generic repository interface
interface Repository<T, ID> {
  findById(id: ID): Promise<T | null>;
  findAll(options?: FindOptions): Promise<PaginatedResult<T>>;
  save(entity: T): Promise<void>;
  delete(id: ID): Promise<void>;
}

interface FindOptions {
  page?: number;
  limit?: number;
  sortBy?: string;
  sortOrder?: "ASC" | "DESC";
  filters?: Record<string, unknown>;
}

interface PaginatedResult<T> {
  items: T[];
  total: number;
  page: number;
  totalPages: number;
}

// Base repository implementation
abstract class BaseRepository<T, ID> implements Repository<T, ID> {
  constructor(
    protected readonly pool: Pool,
    protected readonly tableName: string
  ) {}

  abstract mapRowToEntity(row: Record<string, unknown>): T;
  abstract mapEntityToRow(entity: T): Record<string, unknown>;

  async findById(id: ID): Promise<T | null> {
    const result = await this.pool.query(
      `SELECT * FROM ${this.tableName} WHERE id = $1 AND deleted_at IS NULL`,
      [id]
    );
    return result.rows[0] ? this.mapRowToEntity(result.rows[0]) : null;
  }

  async findAll(options: FindOptions = {}): Promise<PaginatedResult<T>> {
    const {
      page = 1,
      limit = 20,
      sortBy = "created_at",
      sortOrder = "DESC",
    } = options;

    const offset = (page - 1) * limit;

    const [dataResult, countResult] = await Promise.all([
      this.pool.query(
        `SELECT * FROM ${this.tableName}
         WHERE deleted_at IS NULL
         ORDER BY ${sortBy} ${sortOrder}
         LIMIT $1 OFFSET $2`,
        [limit, offset]
      ),
      this.pool.query(
        `SELECT COUNT(*) FROM ${this.tableName} WHERE deleted_at IS NULL`
      ),
    ]);

    const total = parseInt(countResult.rows[0].count);

    return {
      items: dataResult.rows.map((row) => this.mapRowToEntity(row)),
      total,
      page,
      totalPages: Math.ceil(total / limit),
    };
  }

  async save(entity: T): Promise<void> {
    const row = this.mapEntityToRow(entity);
    const columns = Object.keys(row);
    const values = Object.values(row);
    const placeholders = columns.map((_, i) => `$${i + 1}`).join(", ");
    const updateSet = columns
      .filter((c) => c !== "id")
      .map((c, i) => `${c} = $${i + 2}`)
      .join(", ");

    await this.pool.query(
      `INSERT INTO ${this.tableName} (${columns.join(", ")})
       VALUES (${placeholders})
       ON CONFLICT (id) DO UPDATE SET ${updateSet}`,
      values
    );
  }

  async delete(id: ID): Promise<void> {
    await this.pool.query(
      `UPDATE ${this.tableName} SET deleted_at = NOW() WHERE id = $1`,
      [id]
    );
  }
}

// Concrete implementation
class UserRepository extends BaseRepository<User, UserId> {
  constructor(pool: Pool) {
    super(pool, "users");
  }

  mapRowToEntity(row: Record<string, unknown>): User {
    return User.create({
      id: row.id as string,
      email: row.email as string,
      name: row.name as string,
    });
  }

  mapEntityToRow(user: User): Record<string, unknown> {
    return {
      id: user.id,
      email: user.email,
      name: user.name,
      created_at: user.createdAt,
    };
  }

  // Additional user-specific methods
  async findByEmail(email: Email): Promise<User | null> {
    const result = await this.pool.query(
      "SELECT * FROM users WHERE email = $1 AND deleted_at IS NULL",
      [email]
    );
    return result.rows[0] ? this.mapRowToEntity(result.rows[0]) : null;
  }
}
```

---

## 8. Type-safe Configuration

```typescript
// config/schema.ts
import Joi from "joi";

const configSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid("development", "staging", "production", "test")
    .default("development"),

  // Server
  PORT: Joi.number().port().default(3000),
  HOST: Joi.string().hostname().default("0.0.0.0"),

  // Database
  DATABASE_URL: Joi.string().uri().required(),
  DATABASE_MAX_CONNECTIONS: Joi.number().min(1).max(100).default(10),
  DATABASE_IDLE_TIMEOUT_MS: Joi.number().default(30000),

  // Redis
  REDIS_URL: Joi.string().uri().required(),
  REDIS_MAX_RETRIES: Joi.number().default(3),

  // JWT
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_EXPIRES_IN: Joi.string().default("1h"),
  JWT_REFRESH_EXPIRES_IN: Joi.string().default("7d"),

  // External services
  SENDGRID_API_KEY: Joi.string().optional(),
  STRIPE_SECRET_KEY: Joi.string().optional(),

  // Feature flags
  FEATURE_NEW_CHECKOUT: Joi.boolean().default(false),
  FEATURE_AI_RECOMMENDATIONS: Joi.boolean().default(false),
});

interface AppConfig {
  env: "development" | "staging" | "production" | "test";
  server: { port: number; host: string };
  database: {
    url: string;
    maxConnections: number;
    idleTimeoutMs: number;
  };
  redis: { url: string; maxRetries: number };
  jwt: {
    secret: string;
    expiresIn: string;
    refreshExpiresIn: string;
  };
  sendgrid?: { apiKey: string };
  stripe?: { secretKey: string };
  features: {
    newCheckout: boolean;
    aiRecommendations: boolean;
  };
}

export function loadConfig(): AppConfig {
  const { error, value } = configSchema.validate(process.env, {
    allowUnknown: true,
    stripUnknown: true,
  });

  if (error) {
    throw new Error(`Configuration validation failed: ${error.message}`);
  }

  return {
    env: value.NODE_ENV,
    server: {
      port: value.PORT,
      host: value.HOST,
    },
    database: {
      url: value.DATABASE_URL,
      maxConnections: value.DATABASE_MAX_CONNECTIONS,
      idleTimeoutMs: value.DATABASE_IDLE_TIMEOUT_MS,
    },
    redis: {
      url: value.REDIS_URL,
      maxRetries: value.REDIS_MAX_RETRIES,
    },
    jwt: {
      secret: value.JWT_SECRET,
      expiresIn: value.JWT_EXPIRES_IN,
      refreshExpiresIn: value.JWT_REFRESH_EXPIRES_IN,
    },
    sendgrid: value.SENDGRID_API_KEY
      ? { apiKey: value.SENDGRID_API_KEY }
      : undefined,
    stripe: value.STRIPE_SECRET_KEY
      ? { secretKey: value.STRIPE_SECRET_KEY }
      : undefined,
    features: {
      newCheckout: value.FEATURE_NEW_CHECKOUT,
      aiRecommendations: value.FEATURE_AI_RECOMMENDATIONS,
    },
  };
}
```

---

## 9. Middleware Pipeline Pattern

```typescript
// middleware/pipeline.ts
type Middleware<T> = (context: T, next: () => Promise<void>) => Promise<void>;

export class Pipeline<T> {
  private middlewares: Middleware<T>[] = [];

  use(middleware: Middleware<T>): this {
    this.middlewares.push(middleware);
    return this;
  }

  async execute(context: T): Promise<void> {
    const run = async (index: number): Promise<void> => {
      if (index >= this.middlewares.length) return;
      await this.middlewares[index](context, () => run(index + 1));
    };
    await run(0);
  }
}

// HTTP Request context
interface RequestContext {
  req: Request;
  res: Response;
  user?: AuthenticatedUser;
  correlationId: string;
  startTime: number;
}

// Middleware implementations
const authMiddleware: Middleware<RequestContext> = async (ctx, next) => {
  const token = ctx.req.headers.authorization?.replace("Bearer ", "");
  if (!token) {
    ctx.res.status(401).json({ error: "Unauthorized" });
    return;
  }

  try {
    ctx.user = await verifyToken(token);
    await next();
  } catch {
    ctx.res.status(401).json({ error: "Invalid token" });
  }
};

const rateLimitMiddleware: Middleware<RequestContext> = async (ctx, next) => {
  const key = `rate:${ctx.user?.id ?? ctx.req.ip}`;
  const count = await redis.incr(key);

  if (count === 1) {
    await redis.expire(key, 60);
  }

  if (count > 100) {
    ctx.res.status(429).json({ error: "Too Many Requests" });
    return;
  }

  await next();
};

const loggingMiddleware: Middleware<RequestContext> = async (ctx, next) => {
  logger.info("Request started", {
    correlationId: ctx.correlationId,
    method: ctx.req.method,
    path: ctx.req.path,
    userId: ctx.user?.id,
  });

  await next();

  const duration = Date.now() - ctx.startTime;
  logger.info("Request completed", {
    correlationId: ctx.correlationId,
    statusCode: ctx.res.statusCode,
    duration: `${duration}ms`,
  });
};

// Build pipeline
const requestPipeline = new Pipeline<RequestContext>()
  .use(loggingMiddleware)
  .use(authMiddleware)
  .use(rateLimitMiddleware);
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้ TypeScript patterns ขั้นสูงที่สำคัญสำหรับการพัฒนา Microservices:

1. **Branded Types** — ป้องกัน type confusion ระหว่าง primitive types ที่มี semantic ต่างกัน
2. **Result Type Pattern** — จัดการ error handling แบบ explicit และ composable
3. **Option/Maybe Monad** — จัดการค่า nullable อย่างปลอดภัย
4. **Dependency Injection** — ทำให้โค้ด testable และ loosely coupled ด้วย tsyringe
5. **Decorators** — ใส่ cross-cutting concerns (logging, caching, retry) โดยไม่ปะปนกับ business logic
6. **Type-safe Event System** — ป้องกัน event name/payload mismatch ด้วย TypeScript generics
7. **Generic Repository** — สร้าง reusable data access layer ที่ type-safe
8. **Type-safe Configuration** — ตรวจสอบ environment variables ตั้งแต่ startup

Patterns เหล่านี้ทำงานร่วมกันได้ดีและช่วยให้โค้ดมีความแข็งแกร่ง บำรุงรักษาได้ง่าย และลด runtime errors ลงได้อย่างมาก

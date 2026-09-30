# Part 83: API Design Best Practices

## บทนำ

API คือ "contract" ระหว่าง services และ clients การออกแบบ API ที่ดีนั้นไม่ใช่แค่เรื่อง technical แต่เป็นเรื่องของ developer experience, backward compatibility, และ business agility บทนี้จะให้ guidance ที่ครอบคลุมทุกด้านของ API design

---

## REST vs GraphQL vs gRPC: Decision Guide

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    API PROTOCOL DECISION MATRIX                         │
├─────────────────┬─────────────────┬─────────────────┬───────────────────┤
│                 │      REST       │    GraphQL      │       gRPC        │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Use case        │ Public APIs,    │ Complex queries,│ Internal service  │
│                 │ CRUD resources  │ mobile apps     │ communication     │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Performance     │ Good            │ Can be slow     │ Excellent         │
│                 │                 │ (N+1 risk)      │ (binary, HTTP/2)  │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Learning curve  │ Low             │ Medium          │ High              │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Caching         │ Easy (HTTP)     │ Complex         │ Manual            │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Type safety     │ Manual/OpenAPI  │ Built-in        │ Built-in (proto)  │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Streaming       │ SSE/WS          │ Subscriptions   │ Native            │
├─────────────────┼─────────────────┼─────────────────┼───────────────────┤
│ Browser support │ Native          │ Native          │ gRPC-web needed   │
└─────────────────┴─────────────────┴─────────────────┴───────────────────┘

เมื่อไหร่ควรใช้ REST?
✓ Public-facing API (third-party developers)
✓ Simple CRUD operations
✓ ต้องการ HTTP caching
✓ Team คุ้นเคยกับ REST

เมื่อไหร่ควรใช้ GraphQL?
✓ Mobile apps ที่ต้องการ bandwidth optimization
✓ Complex queries ที่ต้องการ flexibility
✓ Rapid iteration ของ frontend
✓ Aggregation layer (BFF pattern)

เมื่อไหร่ควรใช้ gRPC?
✓ Internal service-to-service communication
✓ High performance ต้องการ low latency
✓ Streaming data (real-time)
✓ Polyglot environment
```

---

## REST API Design Best Practices

### Resource Naming Conventions

```
Resource Naming Rules:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✅ DO:
GET    /api/v1/orders              - list all orders
POST   /api/v1/orders              - create order
GET    /api/v1/orders/{orderId}    - get specific order
PUT    /api/v1/orders/{orderId}    - replace order
PATCH  /api/v1/orders/{orderId}    - partial update
DELETE /api/v1/orders/{orderId}    - delete order

GET    /api/v1/orders/{orderId}/items          - get order items (sub-resource)
POST   /api/v1/orders/{orderId}/items          - add item to order
DELETE /api/v1/orders/{orderId}/items/{itemId} - remove item

❌ DON'T:
GET  /api/v1/getOrders         - ใช้ HTTP method แทน verb ใน URL
POST /api/v1/createOrder       - ไม่ควรมี verb ใน URL
GET  /api/v1/order             - ใช้ plural
GET  /api/v1/Orders            - lowercase เสมอ
GET  /api/v1/order_items       - ใช้ kebab-case ไม่ใช่ snake_case
```

```typescript
// ✅ GOOD: Well-designed REST Controller
// services/order/src/controllers/order.controller.ts

import {
  Controller,
  Get,
  Post,
  Put,
  Patch,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
  UseGuards,
  Request,
  VERSION_NEUTRAL,
} from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
} from '@nestjs/swagger';

@ApiTags('orders')
@ApiBearerAuth()
@Controller({ path: 'orders', version: '1' })
export class OrderV1Controller {
  constructor(private readonly orderService: OrderService) {}

  @Get()
  @ApiOperation({ summary: 'List all orders for authenticated user' })
  @ApiResponse({ status: 200, type: PaginatedOrdersResponse })
  async listOrders(
    @Request() req: AuthenticatedRequest,
    @Query() query: ListOrdersQueryDto,
  ): Promise<PaginatedOrdersResponse> {
    return this.orderService.listOrders(req.user.id, query);
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: 'Create a new order' })
  @ApiResponse({ status: 201, type: OrderResponse })
  @ApiResponse({ status: 400, type: ErrorResponse })
  @ApiResponse({ status: 422, type: ValidationErrorResponse })
  async createOrder(
    @Request() req: AuthenticatedRequest,
    @Body() dto: CreateOrderDto,
  ): Promise<OrderResponse> {
    return this.orderService.createOrder(req.user.id, dto);
  }

  @Get(':orderId')
  @ApiOperation({ summary: 'Get a specific order' })
  @ApiResponse({ status: 200, type: OrderResponse })
  @ApiResponse({ status: 404, type: ErrorResponse })
  async getOrder(
    @Request() req: AuthenticatedRequest,
    @Param('orderId') orderId: string,
  ): Promise<OrderResponse> {
    return this.orderService.getOrder(orderId, req.user.id);
  }

  @Patch(':orderId')
  @ApiOperation({ summary: 'Partially update an order' })
  async updateOrder(
    @Request() req: AuthenticatedRequest,
    @Param('orderId') orderId: string,
    @Body() dto: UpdateOrderDto,
  ): Promise<OrderResponse> {
    return this.orderService.updateOrder(orderId, req.user.id, dto);
  }

  @Delete(':orderId')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Cancel an order' })
  @ApiResponse({ status: 204 })
  async cancelOrder(
    @Request() req: AuthenticatedRequest,
    @Param('orderId') orderId: string,
  ): Promise<void> {
    await this.orderService.cancelOrder(orderId, req.user.id);
  }

  // Sub-resource: Order Items
  @Get(':orderId/items')
  async listOrderItems(
    @Param('orderId') orderId: string,
  ): Promise<OrderItem[]> {
    return this.orderService.listOrderItems(orderId);
  }
}
```

---

## API Versioning Strategies

### Strategy 1: URL Path Versioning (แนะนำสำหรับ REST)

```typescript
// main.ts
import { VersioningType } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  app.enableVersioning({
    type: VersioningType.URI,
    prefix: 'v',
    defaultVersion: '1',
  });
  
  await app.listen(3000);
}

// URL examples:
// GET /api/v1/orders
// GET /api/v2/orders
```

### Strategy 2: Header Versioning

```typescript
app.enableVersioning({
  type: VersioningType.HEADER,
  header: 'API-Version',
  defaultVersion: '1',
});

// Request headers:
// API-Version: 2
```

### Deprecation Strategy

```typescript
// services/order/src/middlewares/deprecation.middleware.ts

import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class DeprecationMiddleware implements NestMiddleware {
  private readonly deprecatedVersions: Map<string, DeprecationInfo> = new Map([
    ['v1', {
      sunset: new Date('2025-06-01'),
      alternative: '/api/v2',
      migrationGuide: 'https://docs.company.com/api/v1-to-v2-migration',
    }],
  ]);

  use(req: Request, res: Response, next: NextFunction): void {
    const version = this.extractVersion(req.path);
    const deprecationInfo = this.deprecatedVersions.get(version);

    if (deprecationInfo) {
      // Add Deprecation headers (RFC 8594)
      res.setHeader('Deprecation', deprecationInfo.sunset.toUTCString());
      res.setHeader('Sunset', deprecationInfo.sunset.toUTCString());
      res.setHeader('Link', `<${deprecationInfo.alternative}>; rel="successor-version"`);
      
      if (deprecationInfo.migrationGuide) {
        res.setHeader(
          'Link',
          `${res.getHeader('Link')}, <${deprecationInfo.migrationGuide}>; rel="deprecation"`
        );
      }
    }

    next();
  }

  private extractVersion(path: string): string {
    const match = path.match(/^\/api\/(v\d+)\//);
    return match ? match[1] : '';
  }
}
```

---

## Error Response Format

### Standard Error Format (RFC 7807 Problem Details)

```typescript
// shared/src/errors/problem-details.ts

export interface ProblemDetails {
  type: string;         // URI ที่ identify ประเภทของ error
  title: string;        // Short human-readable summary
  status: number;       // HTTP status code
  detail: string;       // Human-readable explanation
  instance?: string;    // URI ที่ identify specific occurrence
  
  // Extension fields
  errors?: ValidationError[];
  traceId?: string;
  requestId?: string;
}

export interface ValidationError {
  field: string;
  message: string;
  rejectedValue?: unknown;
  code: string;
}
```

```typescript
// shared/src/filters/http-exception.filter.ts

import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Response, Request } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const requestId = request.headers['x-request-id'] as string || uuidv4();
    const traceId = request.headers['x-trace-id'] as string;

    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      const problemDetails: ProblemDetails = {
        type: this.getErrorType(status),
        title: this.getErrorTitle(status),
        status,
        detail: typeof exceptionResponse === 'string' 
          ? exceptionResponse 
          : (exceptionResponse as any).message || 'An error occurred',
        instance: request.url,
        requestId,
        traceId,
        ...(this.isValidationError(exceptionResponse) && {
          errors: this.extractValidationErrors(exceptionResponse),
        }),
      };

      this.logger.warn(`HTTP ${status} on ${request.method} ${request.url}`, {
        requestId,
        error: problemDetails.detail,
      });

      response
        .status(status)
        .contentType('application/problem+json')
        .json(problemDetails);
    } else {
      this.logger.error('Unhandled exception', exception as Error, {
        requestId,
        method: request.method,
        url: request.url,
      });

      response
        .status(HttpStatus.INTERNAL_SERVER_ERROR)
        .contentType('application/problem+json')
        .json({
          type: 'https://docs.company.com/errors/internal-server-error',
          title: 'Internal Server Error',
          status: 500,
          detail: 'An unexpected error occurred. Please try again later.',
          instance: request.url,
          requestId,
          traceId,
        } as ProblemDetails);
    }
  }

  private getErrorType(status: number): string {
    const types: Record<number, string> = {
      400: 'https://docs.company.com/errors/bad-request',
      401: 'https://docs.company.com/errors/unauthorized',
      403: 'https://docs.company.com/errors/forbidden',
      404: 'https://docs.company.com/errors/not-found',
      409: 'https://docs.company.com/errors/conflict',
      422: 'https://docs.company.com/errors/unprocessable-entity',
      429: 'https://docs.company.com/errors/too-many-requests',
      500: 'https://docs.company.com/errors/internal-server-error',
    };
    return types[status] || `https://docs.company.com/errors/${status}`;
  }

  private getErrorTitle(status: number): string {
    const titles: Record<number, string> = {
      400: 'Bad Request',
      401: 'Unauthorized',
      403: 'Forbidden',
      404: 'Not Found',
      409: 'Conflict',
      422: 'Unprocessable Entity',
      429: 'Too Many Requests',
      500: 'Internal Server Error',
    };
    return titles[status] || 'Error';
  }

  private isValidationError(response: unknown): boolean {
    return (
      typeof response === 'object' &&
      response !== null &&
      'message' in response &&
      Array.isArray((response as any).message)
    );
  }

  private extractValidationErrors(response: unknown): ValidationError[] {
    const messages = (response as any).message as string[];
    return messages.map(msg => ({
      field: msg.split(' ')[0],
      message: msg,
      code: 'VALIDATION_ERROR',
    }));
  }
}
```

```json
// ตัวอย่าง Error Response:
{
  "type": "https://docs.company.com/errors/unprocessable-entity",
  "title": "Unprocessable Entity",
  "status": 422,
  "detail": "Request validation failed",
  "instance": "/api/v1/orders",
  "requestId": "req-123e4567-e89b-12d3-a456-426614174000",
  "traceId": "abc123def456",
  "errors": [
    {
      "field": "items[0].quantity",
      "message": "items[0].quantity must be a positive integer",
      "rejectedValue": -1,
      "code": "MIN_VALUE"
    },
    {
      "field": "shippingAddress.zipCode",
      "message": "shippingAddress.zipCode must match Thai postal code format",
      "rejectedValue": "ABC",
      "code": "PATTERN_MISMATCH"
    }
  ]
}
```

---

## Pagination Patterns

### Cursor-based Pagination (แนะนำสำหรับ large datasets)

```typescript
// shared/src/pagination/cursor-pagination.ts

export interface CursorPaginationParams {
  cursor?: string;    // Base64 encoded cursor
  limit: number;      // max 100
  direction?: 'forward' | 'backward';
}

export interface CursorPaginatedResponse<T> {
  data: T[];
  pagination: {
    hasNextPage: boolean;
    hasPreviousPage: boolean;
    startCursor: string | null;
    endCursor: string | null;
    totalCount?: number;  // optional, expensive to compute
  };
}

@Injectable()
export class CursorPaginationService {
  encodeCursor(value: string | number | Date): string {
    return Buffer.from(String(value)).toString('base64url');
  }

  decodeCursor(cursor: string): string {
    return Buffer.from(cursor, 'base64url').toString('utf-8');
  }

  async paginate<T extends { id: string; createdAt: Date }>(
    queryBuilder: SelectQueryBuilder<T>,
    params: CursorPaginationParams,
  ): Promise<CursorPaginatedResponse<T>> {
    const limit = Math.min(params.limit || 20, 100);

    if (params.cursor) {
      const decodedCursor = this.decodeCursor(params.cursor);
      
      if (params.direction === 'backward') {
        queryBuilder.andWhere('entity.id > :cursor', { cursor: decodedCursor });
      } else {
        queryBuilder.andWhere('entity.id < :cursor', { cursor: decodedCursor });
      }
    }

    queryBuilder
      .orderBy('entity.createdAt', 'DESC')
      .addOrderBy('entity.id', 'DESC')
      .limit(limit + 1); // fetch one extra to check hasNextPage

    const items = await queryBuilder.getMany();
    const hasNextPage = items.length > limit;
    
    if (hasNextPage) {
      items.pop(); // remove the extra item
    }

    const startCursor = items.length > 0 
      ? this.encodeCursor(items[0].id)
      : null;
    
    const endCursor = items.length > 0 
      ? this.encodeCursor(items[items.length - 1].id)
      : null;

    return {
      data: items,
      pagination: {
        hasNextPage,
        hasPreviousPage: !!params.cursor,
        startCursor,
        endCursor,
      },
    };
  }
}
```

### Offset-based Pagination (สำหรับ simple cases)

```typescript
// shared/src/pagination/offset-pagination.ts

export interface OffsetPaginationParams {
  page?: number;    // 1-based
  limit?: number;   // max 100
}

export interface OffsetPaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    totalCount: number;
    totalPages: number;
    hasNextPage: boolean;
    hasPreviousPage: boolean;
  };
}

@Injectable()
export class OffsetPaginationService {
  async paginate<T>(
    queryBuilder: SelectQueryBuilder<T>,
    params: OffsetPaginationParams,
  ): Promise<OffsetPaginatedResponse<T>> {
    const page = Math.max(1, params.page || 1);
    const limit = Math.min(params.limit || 20, 100);
    const offset = (page - 1) * limit;

    const [data, totalCount] = await queryBuilder
      .skip(offset)
      .take(limit)
      .getManyAndCount();

    const totalPages = Math.ceil(totalCount / limit);

    return {
      data,
      pagination: {
        page,
        limit,
        totalCount,
        totalPages,
        hasNextPage: page < totalPages,
        hasPreviousPage: page > 1,
      },
    };
  }
}
```

```json
// Response format สำหรับ Cursor Pagination:
{
  "data": [
    { "id": "order-1", "status": "COMPLETED" },
    { "id": "order-2", "status": "PENDING" }
  ],
  "pagination": {
    "hasNextPage": true,
    "hasPreviousPage": false,
    "startCursor": "b3JkZXItMQ==",
    "endCursor": "b3JkZXItMjA=="
  }
}
```

---

## Idempotency Keys

### การ implement Idempotency สำหรับ POST requests

```typescript
// shared/src/idempotency/idempotency.service.ts

import { Injectable, ConflictException } from '@nestjs/common';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

export interface IdempotencyRecord {
  requestId: string;
  response: unknown;
  statusCode: number;
  createdAt: Date;
  expiresAt: Date;
}

@Injectable()
export class IdempotencyService {
  private readonly TTL_SECONDS = 86400; // 24 hours

  constructor(
    @InjectRedis() private readonly redis: Redis,
  ) {}

  private getKey(idempotencyKey: string): string {
    return `idempotency:${idempotencyKey}`;
  }

  async getStoredResponse(
    idempotencyKey: string
  ): Promise<IdempotencyRecord | null> {
    const stored = await this.redis.get(this.getKey(idempotencyKey));
    if (!stored) return null;
    return JSON.parse(stored);
  }

  async storeResponse(
    idempotencyKey: string,
    statusCode: number,
    response: unknown,
  ): Promise<void> {
    const record: IdempotencyRecord = {
      requestId: idempotencyKey,
      response,
      statusCode,
      createdAt: new Date(),
      expiresAt: new Date(Date.now() + this.TTL_SECONDS * 1000),
    };

    await this.redis.setex(
      this.getKey(idempotencyKey),
      this.TTL_SECONDS,
      JSON.stringify(record),
    );
  }

  async acquireLock(idempotencyKey: string): Promise<boolean> {
    // Set lock with NX (only if not exists)
    const result = await this.redis.set(
      `${this.getKey(idempotencyKey)}:lock`,
      '1',
      'EX',
      30,   // lock expires in 30 seconds
      'NX', // only set if not exists
    );
    return result === 'OK';
  }

  async releaseLock(idempotencyKey: string): Promise<void> {
    await this.redis.del(`${this.getKey(idempotencyKey)}:lock`);
  }
}
```

```typescript
// shared/src/idempotency/idempotency.interceptor.ts

import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  ConflictException,
  HttpException,
  Logger,
} from '@nestjs/common';
import { Observable, from } from 'rxjs';
import { switchMap, tap } from 'rxjs/operators';
import { Request, Response } from 'express';

@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  private readonly logger = new Logger(IdempotencyInterceptor.name);

  constructor(
    private readonly idempotencyService: IdempotencyService,
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<Request>();
    const response = context.switchToHttp().getResponse<Response>();
    
    const idempotencyKey = request.headers['idempotency-key'] as string;
    
    // Only apply to POST, PUT, PATCH
    if (!['POST', 'PUT', 'PATCH'].includes(request.method) || !idempotencyKey) {
      return next.handle();
    }

    return from(this.handleIdempotency(idempotencyKey, response, next));
  }

  private async handleIdempotency(
    idempotencyKey: string,
    response: Response,
    next: CallHandler,
  ): Promise<unknown> {
    // Check if we already have a response
    const stored = await this.idempotencyService.getStoredResponse(idempotencyKey);
    if (stored) {
      this.logger.debug(`Returning cached response for key: ${idempotencyKey}`);
      response.setHeader('Idempotency-Key-Replay', 'true');
      response.status(stored.statusCode);
      return stored.response;
    }

    // Try to acquire lock (prevent concurrent requests with same key)
    const locked = await this.idempotencyService.acquireLock(idempotencyKey);
    if (!locked) {
      throw new ConflictException(
        'A request with this idempotency key is currently being processed'
      );
    }

    try {
      const result = await next.handle().toPromise();
      const statusCode = response.statusCode || 200;
      
      await this.idempotencyService.storeResponse(
        idempotencyKey,
        statusCode,
        result,
      );
      
      return result;
    } finally {
      await this.idempotencyService.releaseLock(idempotencyKey);
    }
  }
}
```

```typescript
// ใช้ Idempotency Interceptor ใน controller
@Controller({ path: 'orders', version: '1' })
@UseInterceptors(IdempotencyInterceptor)
export class OrderController {
  @Post()
  @HttpCode(HttpStatus.CREATED)
  async createOrder(@Body() dto: CreateOrderDto): Promise<Order> {
    // Client ส่ง header: Idempotency-Key: unique-uuid-per-request
    // ถ้า retry ด้วย key เดิม จะได้ response เดิมกลับมา
    return this.orderService.createOrder(dto);
  }
}
```

---

## Webhook Design

### Webhook Payload Structure

```typescript
// shared/src/webhooks/webhook.types.ts

export interface WebhookPayload<T = unknown> {
  // Event identification
  id: string;           // Unique webhook event ID
  type: string;         // Event type: "order.created", "payment.completed"
  version: string;      // Payload schema version: "1.0"
  
  // Timing
  createdAt: string;    // ISO 8601
  
  // Source
  source: {
    service: string;    // "order-service"
    environment: string;// "production"
  };
  
  // Payload
  data: T;
  
  // Retry info
  attempt: number;      // 1 = first attempt
  maxAttempts: number;
}

// ตัวอย่าง Order Created Webhook
export interface OrderCreatedWebhookPayload {
  orderId: string;
  userId: string;
  totalAmount: number;
  currency: string;
  items: Array<{
    productId: string;
    quantity: number;
    unitPrice: number;
  }>;
  shippingAddress: Address;
}
```

```typescript
// services/order/src/webhooks/webhook.service.ts

@Injectable()
export class WebhookService {
  private readonly logger = new Logger(WebhookService.name);

  constructor(
    @InjectRepository(WebhookSubscription)
    private readonly subscriptionRepo: Repository<WebhookSubscription>,
    private readonly httpService: HttpService,
    @InjectQueue('webhook-delivery')
    private readonly webhookQueue: Queue,
  ) {}

  async deliver(
    eventType: string,
    data: unknown,
  ): Promise<void> {
    // Find all active subscriptions for this event type
    const subscriptions = await this.subscriptionRepo.find({
      where: {
        events: Like(`%${eventType}%`),
        isActive: true,
      },
    });

    for (const subscription of subscriptions) {
      const payload: WebhookPayload = {
        id: uuidv4(),
        type: eventType,
        version: '1.0',
        createdAt: new Date().toISOString(),
        source: {
          service: 'order-service',
          environment: process.env.NODE_ENV || 'production',
        },
        data,
        attempt: 1,
        maxAttempts: 5,
      };

      // Queue the delivery with retry logic
      await this.webhookQueue.add(
        'deliver',
        {
          subscriptionId: subscription.id,
          webhookUrl: subscription.url,
          signingSecret: subscription.signingSecret,
          payload,
        },
        {
          attempts: 5,
          backoff: {
            type: 'exponential',
            delay: 1000, // 1s, 2s, 4s, 8s, 16s
          },
          removeOnComplete: 100,
          removeOnFail: 200,
        }
      );
    }
  }

  // HMAC signature สำหรับ webhook security
  private signPayload(payload: WebhookPayload, secret: string): string {
    const timestamp = Math.floor(Date.now() / 1000);
    const signedContent = `${timestamp}.${JSON.stringify(payload)}`;
    
    return `t=${timestamp},v1=${
      crypto
        .createHmac('sha256', secret)
        .update(signedContent)
        .digest('hex')
    }`;
  }

  async sendWebhook(
    url: string,
    payload: WebhookPayload,
    signingSecret: string,
  ): Promise<void> {
    const signature = this.signPayload(payload, signingSecret);
    
    await this.httpService.post(url, payload, {
      headers: {
        'Content-Type': 'application/json',
        'X-Webhook-Signature': signature,
        'X-Webhook-ID': payload.id,
        'X-Webhook-Timestamp': new Date().toISOString(),
      },
      timeout: 30000, // 30 second timeout
    }).toPromise();
  }
}
```

```typescript
// ตัวอย่าง Webhook Verification ฝั่ง consumer:
// services/delivery/src/webhooks/order-webhook.handler.ts

@Controller('webhooks')
export class OrderWebhookHandler {
  @Post('orders')
  async handleOrderEvent(
    @Body() payload: WebhookPayload,
    @Headers('x-webhook-signature') signature: string,
    @RawBody() rawBody: Buffer,
  ): Promise<void> {
    // Verify signature
    const isValid = this.verifySignature(rawBody, signature);
    if (!isValid) {
      throw new UnauthorizedException('Invalid webhook signature');
    }

    // Process idempotently using webhook ID
    const alreadyProcessed = await this.redis.get(`webhook:${payload.id}`);
    if (alreadyProcessed) {
      return; // Already processed
    }

    await this.processWebhook(payload);
    
    // Mark as processed (expire after 24 hours)
    await this.redis.setex(`webhook:${payload.id}`, 86400, '1');
  }

  private verifySignature(body: Buffer, signature: string): boolean {
    const [timestampPart, signaturePart] = signature.split(',');
    const timestamp = timestampPart.replace('t=', '');
    const receivedSig = signaturePart.replace('v1=', '');

    // Reject webhooks older than 5 minutes
    const webhookAge = Math.abs(
      Date.now() / 1000 - parseInt(timestamp)
    );
    if (webhookAge > 300) {
      return false;
    }

    const expectedSig = crypto
      .createHmac('sha256', process.env.WEBHOOK_SECRET)
      .update(`${timestamp}.${body.toString('utf-8')}`)
      .digest('hex');

    return crypto.timingSafeEqual(
      Buffer.from(receivedSig),
      Buffer.from(expectedSig)
    );
  }
}
```

---

## OpenAPI Documentation Best Practices

```typescript
// main.ts - OpenAPI setup

import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('Order Service API')
    .setDescription(`
      ## Overview
      The Order Service manages the complete order lifecycle from creation to delivery.
      
      ## Authentication
      All endpoints require Bearer token authentication.
      Use the \`/auth/login\` endpoint to obtain a token.
      
      ## Versioning
      This API uses URL versioning: \`/api/v1/...\`, \`/api/v2/...\`
      
      ## Rate Limiting
      - Standard: 100 requests/minute
      - Premium: 1000 requests/minute
      
      ## Idempotency
      POST endpoints support idempotency via the \`Idempotency-Key\` header.
    `)
    .setVersion('1.0')
    .addBearerAuth()
    .addServer('https://api.company.com', 'Production')
    .addServer('https://api-staging.company.com', 'Staging')
    .addTag('orders', 'Order management')
    .addTag('order-items', 'Order item management')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  
  SwaggerModule.setup('api/docs', app, document, {
    swaggerOptions: {
      persistAuthorization: true,
      tagsSorter: 'alpha',
      operationsSorter: 'alpha',
    },
  });
}
```

---

## Rate Limiting

```typescript
// shared/src/rate-limit/rate-limit.guard.ts

import { Injectable, ExecutionContext, TooManyRequestsException } from '@nestjs/common';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(
    @InjectRedis() private readonly redis: Redis,
    private readonly reflector: Reflector,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();
    
    const rateLimit = this.reflector.getAllAndOverride<RateLimitConfig>(
      'rateLimit',
      [context.getHandler(), context.getClass()]
    ) || { limit: 100, window: 60 };

    const key = `rate:${request.user?.id || request.ip}:${request.path}`;
    
    const [count] = await this.redis
      .multi()
      .incr(key)
      .expire(key, rateLimit.window)
      .exec();

    const currentCount = count[1] as number;
    const remaining = Math.max(0, rateLimit.limit - currentCount);
    
    response.setHeader('X-RateLimit-Limit', rateLimit.limit);
    response.setHeader('X-RateLimit-Remaining', remaining);
    response.setHeader(
      'X-RateLimit-Reset',
      Math.floor(Date.now() / 1000) + rateLimit.window
    );

    if (currentCount > rateLimit.limit) {
      response.setHeader(
        'Retry-After',
        rateLimit.window.toString()
      );
      throw new TooManyRequestsException(
        `Rate limit exceeded. Try again in ${rateLimit.window} seconds.`
      );
    }

    return true;
  }
}
```

---

## สรุป

การออกแบบ API ที่ดีต้องพิจารณาหลายมิติ:

| หัวข้อ | Best Practice | เหตุผล |
|--------|---------------|--------|
| Protocol | REST (public), gRPC (internal) | Performance + compatibility |
| Resource Naming | Plural, lowercase, kebab-case | Consistency |
| Versioning | URL path versioning | ชัดเจน, explicit |
| Error Format | RFC 7807 Problem Details | Standard, machine-readable |
| Pagination | Cursor-based for large datasets | Performance, consistency |
| Idempotency | Idempotency-Key header | Safe retries |
| Webhooks | HMAC signature + retry queue | Security + reliability |
| Documentation | OpenAPI 3.0 | Developer experience |

จำไว้ว่า API คือ "product" — ออกแบบโดยคำนึงถึง developer experience ของ consumer เสมอ และ backward compatibility คือ non-negotiable เมื่อ API อยู่ใน production แล้ว

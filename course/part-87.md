# Part 87: gRPC ขั้นสูง

## บทนำ

gRPC เป็น high-performance RPC framework ที่สร้างโดย Google ใช้ Protocol Buffers เป็น interface definition language และ HTTP/2 เป็น transport layer บทนี้ครอบคลุม gRPC แบบขั้นสูงตั้งแต่ streaming patterns, interceptors, load balancing จนถึง gRPC-web สำหรับ browser

---

## Protocol Buffers (Proto3) Best Practices

### Service Definition

```protobuf
// proto/order/v1/order.proto

syntax = "proto3";

package order.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/field_mask.proto";
import "validate/validate.proto";  // protoc-gen-validate

option go_package = "github.com/company/proto/order/v1;orderv1";
option java_package = "com.company.proto.order.v1";
option java_multiple_files = true;

// OrderService handles all order operations
service OrderService {
  // Unary RPC - Create new order
  rpc CreateOrder(CreateOrderRequest) returns (Order) {}
  
  // Unary RPC - Get order by ID
  rpc GetOrder(GetOrderRequest) returns (Order) {}
  
  // Unary RPC - Update order fields
  rpc UpdateOrder(UpdateOrderRequest) returns (Order) {}
  
  // Server streaming - Watch order status changes
  rpc WatchOrder(WatchOrderRequest) returns (stream OrderEvent) {}
  
  // Client streaming - Batch create orders
  rpc BatchCreateOrders(stream CreateOrderRequest) returns (BatchCreateOrdersResponse) {}
  
  // Bidirectional streaming - Real-time order processing
  rpc ProcessOrders(stream CreateOrderRequest) returns (stream Order) {}
  
  // List orders with pagination
  rpc ListOrders(ListOrdersRequest) returns (ListOrdersResponse) {}
}

// ────────────────────────────── Messages ──────────────────────────────

message Order {
  string id = 1;
  string user_id = 2;
  OrderStatus status = 3;
  repeated OrderItem items = 4;
  Money total_amount = 5;
  Address shipping_address = 6;
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
  
  // Metadata
  map<string, string> metadata = 9;
}

message OrderItem {
  string product_id = 1;
  string product_name = 2;
  int32 quantity = 3 [(validate.rules).int32.gt = 0];
  Money unit_price = 4;
  Money subtotal = 5;
}

message Money {
  string currency_code = 1 [(validate.rules).string = {
    pattern: "^[A-Z]{3}$"  // ISO 4217
  }];
  int64 units = 2;      // ส่วนเต็ม (baht)
  int32 nanos = 3;      // ส่วนทศนิยม (-999999999 ถึง +999999999)
}

message Address {
  string street = 1 [(validate.rules).string.min_len = 1];
  string city = 2 [(validate.rules).string.min_len = 1];
  string province = 3;
  string postal_code = 4 [(validate.rules).string = {
    pattern: "^[0-9]{5}$"  // Thai postal code
  }];
  string country_code = 5 [(validate.rules).string = {
    len: 2
  }];
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;  // proto3 ต้องมี default 0
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_PROCESSING = 3;
  ORDER_STATUS_SHIPPED = 4;
  ORDER_STATUS_DELIVERED = 5;
  ORDER_STATUS_CANCELLED = 6;
  ORDER_STATUS_REFUNDED = 7;
}

// ────────────────────────────── Requests ──────────────────────────────

message CreateOrderRequest {
  string user_id = 1 [(validate.rules).string.uuid = true];
  repeated OrderItemInput items = 2 [(validate.rules).repeated.min_items = 1];
  Address shipping_address = 3 [(validate.rules).message.required = true];
  string idempotency_key = 4 [(validate.rules).string.min_len = 1];
}

message OrderItemInput {
  string product_id = 1 [(validate.rules).string.uuid = true];
  int32 quantity = 2 [(validate.rules).int32 = {gt: 0, lte: 100}];
}

message GetOrderRequest {
  string order_id = 1 [(validate.rules).string.uuid = true];
  google.protobuf.FieldMask field_mask = 2;  // Partial response
}

message UpdateOrderRequest {
  string order_id = 1 [(validate.rules).string.uuid = true];
  Order order = 2;
  google.protobuf.FieldMask update_mask = 3;  // Only update specified fields
}

message WatchOrderRequest {
  string order_id = 1 [(validate.rules).string.uuid = true];
  repeated OrderStatus watch_statuses = 2;  // Filter specific status changes
}

message OrderEvent {
  string order_id = 1;
  OrderStatus old_status = 2;
  OrderStatus new_status = 3;
  google.protobuf.Timestamp event_time = 4;
  string message = 5;
}

message ListOrdersRequest {
  string user_id = 1;
  OrderStatus status_filter = 2;
  int32 page_size = 3 [(validate.rules).int32 = {gt: 0, lte: 100}];
  string page_token = 4;
}

message ListOrdersResponse {
  repeated Order orders = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message BatchCreateOrdersResponse {
  repeated Order orders = 1;
  repeated BatchCreateError errors = 2;
  int32 success_count = 3;
  int32 error_count = 4;
}

message BatchCreateError {
  int32 index = 1;
  string error_code = 2;
  string message = 3;
}
```

---

## NestJS gRPC Implementation

### Server Setup

```typescript
// services/order/src/main.ts

import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { join } from 'path';
import { AppModule } from './app.module';

async function bootstrap() {
  // HTTP server สำหรับ REST API
  const app = await NestFactory.create(AppModule);

  // gRPC microservice
  app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.GRPC,
    options: {
      package: 'order.v1',
      protoPath: join(__dirname, '../proto/order/v1/order.proto'),
      url: '0.0.0.0:50051',
      loader: {
        keepCase: true,
        longs: String,
        enums: String,
        defaults: true,
        oneofs: true,
        includeDirs: [join(__dirname, '../proto')],
      },
      channelOptions: {
        'grpc.max_receive_message_length': 1024 * 1024 * 10, // 10MB
        'grpc.max_send_message_length': 1024 * 1024 * 10,
        'grpc.keepalive_time_ms': 30000,
        'grpc.keepalive_timeout_ms': 5000,
        'grpc.keepalive_permit_without_calls': 1,
      },
    },
  });

  await app.startAllMicroservices();
  await app.listen(3000);
  
  console.log('gRPC server listening on :50051');
  console.log('HTTP server listening on :3000');
}

bootstrap();
```

```typescript
// services/order/src/order.controller.ts

import { Controller, Logger } from '@nestjs/common';
import { GrpcMethod, GrpcStreamMethod } from '@nestjs/microservices';
import { Subject, Observable } from 'rxjs';
import {
  Order,
  CreateOrderRequest,
  GetOrderRequest,
  WatchOrderRequest,
  OrderEvent,
  ListOrdersRequest,
  ListOrdersResponse,
} from './generated/order/v1/order';

@Controller()
export class OrderGrpcController {
  private readonly logger = new Logger(OrderGrpcController.name);

  constructor(private readonly orderService: OrderService) {}

  // ─── Unary RPC ───────────────────────────────────────────────────────────

  @GrpcMethod('OrderService', 'CreateOrder')
  async createOrder(data: CreateOrderRequest): Promise<Order> {
    this.logger.debug(`Creating order for user: ${data.userId}`);
    return this.orderService.create(data);
  }

  @GrpcMethod('OrderService', 'GetOrder')
  async getOrder(data: GetOrderRequest): Promise<Order> {
    return this.orderService.findById(data.orderId, data.fieldMask);
  }

  // ─── Server Streaming RPC ───────────────────────────────────────────────

  @GrpcStreamMethod('OrderService', 'WatchOrder')
  watchOrder(data: WatchOrderRequest): Observable<OrderEvent> {
    const subject = new Subject<OrderEvent>();

    // Subscribe ไปยัง event stream
    const subscription = this.orderService.watchOrderEvents(
      data.orderId,
      data.watchStatuses,
      (event) => subject.next(event),
      (error) => subject.error(error),
      () => subject.complete(),
    );

    // Cleanup เมื่อ client disconnect
    return new Observable((observer) => {
      const sub = subject.subscribe(observer);
      
      return () => {
        sub.unsubscribe();
        subscription.unsubscribe();
        this.logger.debug(`Client stopped watching order: ${data.orderId}`);
      };
    });
  }

  // ─── Client Streaming RPC ──────────────────────────────────────────────

  @GrpcStreamMethod('OrderService', 'BatchCreateOrders')
  async batchCreateOrders(
    messages: Observable<CreateOrderRequest>,
  ): Promise<BatchCreateOrdersResponse> {
    const orders: Order[] = [];
    const errors: BatchCreateError[] = [];
    let index = 0;

    return new Promise((resolve, reject) => {
      messages.subscribe({
        next: async (request) => {
          try {
            const order = await this.orderService.create(request);
            orders.push(order);
          } catch (error) {
            errors.push({
              index,
              errorCode: error.code || 'UNKNOWN',
              message: error.message,
            });
          }
          index++;
        },
        error: reject,
        complete: () => {
          resolve({
            orders,
            errors,
            successCount: orders.length,
            errorCount: errors.length,
          });
        },
      });
    });
  }

  // ─── Bidirectional Streaming RPC ──────────────────────────────────────

  @GrpcStreamMethod('OrderService', 'ProcessOrders')
  processOrders(
    messages: Observable<CreateOrderRequest>,
  ): Observable<Order> {
    const subject = new Subject<Order>();

    messages.subscribe({
      next: async (request) => {
        try {
          const order = await this.orderService.create(request);
          subject.next(order);
        } catch (error) {
          // gRPC error handling - ส่ง error ใน metadata
          subject.error({
            code: Status.INTERNAL,
            message: error.message,
          });
        }
      },
      error: (error) => subject.error(error),
      complete: () => subject.complete(),
    });

    return subject.asObservable();
  }
}
```

---

## gRPC Error Handling

```typescript
// shared/src/grpc/grpc-exception.filter.ts

import { Catch, RpcExceptionFilter, ArgumentsHost } from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { RpcException } from '@nestjs/microservices';
import { Status } from '@grpc/grpc-js/build/src/constants';

interface GrpcError {
  code: Status;
  message: string;
  details?: string;
  metadata?: Record<string, string>;
}

@Catch()
export class GrpcExceptionFilter implements RpcExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost): Observable<never> {
    const grpcError = this.mapToGrpcError(exception);
    return throwError(() => grpcError);
  }

  private mapToGrpcError(exception: unknown): GrpcError {
    if (exception instanceof RpcException) {
      return exception.getError() as GrpcError;
    }

    if (exception instanceof Error) {
      // Map domain exceptions to gRPC status codes
      const errorMap: Record<string, Status> = {
        NotFoundException: Status.NOT_FOUND,
        UnauthorizedException: Status.UNAUTHENTICATED,
        ForbiddenException: Status.PERMISSION_DENIED,
        BadRequestException: Status.INVALID_ARGUMENT,
        ConflictException: Status.ALREADY_EXISTS,
        TooManyRequestsException: Status.RESOURCE_EXHAUSTED,
        ServiceUnavailableException: Status.UNAVAILABLE,
      };

      const constructorName = exception.constructor.name;
      const statusCode = errorMap[constructorName] || Status.INTERNAL;

      return {
        code: statusCode,
        message: exception.message,
        details: JSON.stringify({
          type: constructorName,
          timestamp: new Date().toISOString(),
        }),
      };
    }

    return {
      code: Status.INTERNAL,
      message: 'Internal server error',
    };
  }
}
```

---

## Interceptors

### Logging Interceptor

```typescript
// shared/src/grpc/interceptors/logging.interceptor.ts

import { Injectable } from '@nestjs/common';
import { InjectPinoLogger, PinoLogger } from 'nestjs-pino';
import { GrpcInterceptor, GrpcCall } from '@grpc/grpc-js';

export const createLoggingInterceptor = (): GrpcInterceptor => {
  return {
    interceptUnary<RequestType, ResponseType>(
      methodDescriptor: any,
      call: GrpcCall<RequestType, ResponseType>,
    ) {
      const method = methodDescriptor.path;
      const startTime = Date.now();
      
      console.log(`gRPC call started: ${method}`);
      
      call.on('error', (error) => {
        console.error(`gRPC call error: ${method}`, {
          duration: Date.now() - startTime,
          error: error.message,
          code: error.code,
        });
      });

      call.on('end', () => {
        console.log(`gRPC call completed: ${method}`, {
          duration: Date.now() - startTime,
        });
      });

      return call;
    },
  };
};
```

### Authentication Interceptor

```typescript
// shared/src/grpc/interceptors/auth.interceptor.ts

import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Metadata } from '@grpc/grpc-js';
import { JwtService } from '@nestjs/jwt';

@Injectable()
export class GrpcAuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const type = context.getType<string>();
    
    if (type !== 'rpc') {
      return true;
    }

    const metadata = context.switchToRpc().getContext<Metadata>();
    const authHeader = metadata.get('authorization')[0] as string;

    if (!authHeader) {
      throw new RpcException({
        code: Status.UNAUTHENTICATED,
        message: 'No authorization token provided',
      });
    }

    const token = authHeader.replace('Bearer ', '');
    
    try {
      const payload = await this.jwtService.verifyAsync(token);
      
      // Attach user to metadata สำหรับ downstream use
      metadata.set('user-id', payload.sub);
      metadata.set('user-email', payload.email);
      
      return true;
    } catch (error) {
      throw new RpcException({
        code: Status.UNAUTHENTICATED,
        message: 'Invalid or expired token',
      });
    }
  }
}
```

### Retry Interceptor (Client-side)

```typescript
// shared/src/grpc/interceptors/retry.interceptor.ts

import {
  CallOptions,
  ClientInterceptor,
  InterceptingCall,
  Listener,
  Metadata,
  Requester,
  StatusObject,
} from '@grpc/grpc-js';
import { Status } from '@grpc/grpc-js/build/src/constants';

export function createRetryInterceptor(
  maxRetries = 3,
  retryableStatusCodes = [
    Status.UNAVAILABLE,
    Status.INTERNAL,
    Status.RESOURCE_EXHAUSTED,
  ]
): ClientInterceptor {
  return (options: CallOptions, nextCall: any) => {
    let retryCount = 0;
    let metadata: Metadata;
    let requestMessage: any;

    const requester: Requester = {
      start: (meta: Metadata, listener: Listener, next: any) => {
        metadata = meta;
        next(meta, {
          onReceiveStatus: (status: StatusObject, nextListener: any) => {
            if (
              retryCount < maxRetries &&
              retryableStatusCodes.includes(status.code)
            ) {
              retryCount++;
              const delay = Math.pow(2, retryCount) * 100; // exponential backoff
              
              setTimeout(() => {
                const newCall = nextCall(options);
                newCall.start(metadata, listener);
                if (requestMessage) {
                  newCall.sendMessage(requestMessage);
                }
                newCall.halfClose();
              }, delay);
              
              return; // Don't call nextListener yet
            }
            
            nextListener.onReceiveStatus(status);
          },
        });
      },
      sendMessage: (message: any, next: any) => {
        requestMessage = message;
        next(message);
      },
      halfClose: (next: any) => next(),
    };

    return new InterceptingCall(nextCall(options), requester);
  };
}
```

---

## Health Checking Protocol

```typescript
// services/order/src/health/grpc-health.controller.ts
// Implements gRPC Health Checking Protocol (grpc.health.v1)

import { Controller } from '@nestjs/common';
import { GrpcMethod } from '@nestjs/microservices';
import { InjectDataSource } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';
import { InjectRedis } from '@liaoliaots/nestjs-redis';
import Redis from 'ioredis';

enum ServingStatus {
  UNKNOWN = 0,
  SERVING = 1,
  NOT_SERVING = 2,
  SERVICE_UNKNOWN = 3,
}

@Controller()
export class HealthController {
  constructor(
    @InjectDataSource()
    private readonly dataSource: DataSource,
    @InjectRedis()
    private readonly redis: Redis,
  ) {}

  @GrpcMethod('Health', 'Check')
  async check(data: { service: string }) {
    const checks = {
      database: await this.checkDatabase(),
      redis: await this.checkRedis(),
    };

    const isHealthy = Object.values(checks).every(Boolean);

    return {
      status: isHealthy ? ServingStatus.SERVING : ServingStatus.NOT_SERVING,
    };
  }

  @GrpcMethod('Health', 'Watch')
  watch(data: { service: string }) {
    // Stream health status changes
    // Implementation ขึ้นอยู่กับ use case
  }

  private async checkDatabase(): Promise<boolean> {
    try {
      await this.dataSource.query('SELECT 1');
      return true;
    } catch {
      return false;
    }
  }

  private async checkRedis(): Promise<boolean> {
    try {
      await this.redis.ping();
      return true;
    } catch {
      return false;
    }
  }
}
```

```yaml
# kubernetes/probes/order-service.yaml
# ใช้ grpc_health_probe สำหรับ Kubernetes health checks

spec:
  containers:
  - name: order-service
    image: company/order-service:latest
    
    livenessProbe:
      exec:
        command:
        - /bin/grpc_health_probe
        - -addr=:50051
        - -service=order.v1.OrderService
        - -rpc-timeout=5s
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    
    readinessProbe:
      exec:
        command:
        - /bin/grpc_health_probe
        - -addr=:50051
        - -service=order.v1.OrderService
      initialDelaySeconds: 5
      periodSeconds: 5
```

---

## gRPC Load Balancing

```typescript
// services/api-gateway/src/grpc/load-balancing.service.ts
// Client-side load balancing ด้วย round-robin

import { Injectable, OnModuleInit } from '@nestjs/common';
import * as grpc from '@grpc/grpc-js';
import { credentials, loadPackageDefinition } from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

@Injectable()
export class OrderServiceClient implements OnModuleInit {
  private client: any;

  async onModuleInit(): Promise<void> {
    const packageDef = await protoLoader.load(
      'proto/order/v1/order.proto',
      {
        keepCase: true,
        longs: String,
        enums: String,
      }
    );

    const proto = loadPackageDefinition(packageDef) as any;

    // DNS load balancing - Kubernetes discovers all pods
    this.client = new proto.order.v1.OrderService(
      'dns:///order-service.production.svc.cluster.local:50051',
      credentials.createInsecure(),
      {
        // Client-side round-robin load balancing
        'grpc.service_config': JSON.stringify({
          loadBalancingPolicy: 'round_robin',
          methodConfig: [
            {
              name: [{ service: 'order.v1.OrderService' }],
              retryPolicy: {
                maxAttempts: 3,
                initialBackoff: '0.1s',
                maxBackoff: '1s',
                backoffMultiplier: 2,
                retryableStatusCodes: ['UNAVAILABLE', 'RESOURCE_EXHAUSTED'],
              },
              timeout: '5s',
            },
          ],
        }),
        'grpc.keepalive_time_ms': 30000,
        'grpc.keepalive_timeout_ms': 5000,
      }
    );
  }
}
```

---

## gRPC-web สำหรับ Browser

```typescript
// frontend/src/grpc/order-client.ts
// gRPC-web client สำหรับ React/Vue frontend

import {
  OrderServiceClient,
  CreateOrderRequest,
  OrderItem,
  Address,
} from './generated/order/v1/order_grpc_web_pb';
import { OrderServicePromiseClient } from './generated/order/v1/order_grpc_web_pb';
import { AuthInterceptor } from './interceptors/auth.interceptor';

// สร้าง gRPC-web client
const authInterceptor = new AuthInterceptor(getAuthToken);

export const orderClient = new OrderServicePromiseClient(
  process.env.REACT_APP_GRPC_GATEWAY_URL || 'http://localhost:8080',
  null,
  {
    unaryInterceptors: [authInterceptor],
    streamInterceptors: [authInterceptor],
  }
);

// Unary call
export async function createOrder(params: {
  userId: string;
  items: Array<{ productId: string; quantity: number }>;
  shippingAddress: {
    street: string;
    city: string;
    postalCode: string;
  };
}): Promise<Order> {
  const request = new CreateOrderRequest();
  request.setUserId(params.userId);
  request.setIdempotencyKey(crypto.randomUUID());

  for (const item of params.items) {
    const orderItem = new OrderItem();
    orderItem.setProductId(item.productId);
    orderItem.setQuantity(item.quantity);
    request.addItems(orderItem);
  }

  const address = new Address();
  address.setStreet(params.shippingAddress.street);
  address.setCity(params.shippingAddress.city);
  address.setPostalCode(params.shippingAddress.postalCode);
  address.setCountryCode('TH');
  request.setShippingAddress(address);

  const order = await orderClient.createOrder(request, {});
  return order.toObject();
}

// Server streaming - Watch order status
export function watchOrder(
  orderId: string,
  onEvent: (event: OrderEvent) => void,
  onError: (error: Error) => void,
  onComplete: () => void,
): () => void {
  const request = new WatchOrderRequest();
  request.setOrderId(orderId);

  const stream = orderClient.watchOrder(request, {});

  stream.on('data', (event) => {
    onEvent(event.toObject());
  });

  stream.on('error', onError);
  stream.on('end', onComplete);

  // Return cancel function
  return () => stream.cancel();
}
```

```yaml
# kubernetes/envoy/grpc-web-proxy.yaml
# Envoy Proxy สำหรับ gRPC-web transcoding

apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy-grpc-gateway
  namespace: production
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: envoy
        image: envoyproxy/envoy:v1.28.0
        ports:
        - containerPort: 8080  # HTTP/gRPC-web port
        - containerPort: 9901  # Admin port
        volumeMounts:
        - name: envoy-config
          mountPath: /etc/envoy
      volumes:
      - name: envoy-config
        configMap:
          name: envoy-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
      - name: listener_0
        address:
          socket_address:
            protocol: TCP
            address: 0.0.0.0
            port_value: 8080
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              codec_type: AUTO
              route_config:
                name: local_route
                virtual_hosts:
                - name: backend
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/order.v1.OrderService"
                    route:
                      cluster: order_service_grpc
              http_filters:
              - name: envoy.filters.http.grpc_web  # gRPC-web support
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
              - name: envoy.filters.http.cors
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
              - name: envoy.filters.http.router
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
      
      clusters:
      - name: order_service_grpc
        connect_timeout: 5s
        type: STRICT_DNS
        http2_protocol_options: {}  # Force HTTP/2 for gRPC
        load_assignment:
          cluster_name: order_service_grpc
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: order-service.production.svc.cluster.local
                    port_value: 50051
```

---

## gRPC Reflection สำหรับ Development

```typescript
// services/order/src/main.ts - เพิ่ม reflection ใน development

import { ReflectionService } from '@grpc/reflection';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  if (process.env.NODE_ENV !== 'production') {
    // Enable gRPC reflection (ทำให้ grpcurl/grpc-ui ทำงานได้)
    app.connectMicroservice<MicroserviceOptions>({
      transport: Transport.GRPC,
      options: {
        package: ['order.v1', 'grpc.reflection.v1alpha'],
        protoPath: [
          join(__dirname, '../proto/order/v1/order.proto'),
          join(__dirname, '../proto/grpc/reflection.proto'),
        ],
        url: '0.0.0.0:50051',
      },
    });
  }
}
```

```bash
# Testing gRPC endpoints ด้วย grpcurl

# List services
grpcurl -plaintext localhost:50051 list

# List methods ใน service
grpcurl -plaintext localhost:50051 list order.v1.OrderService

# Describe message type
grpcurl -plaintext localhost:50051 describe order.v1.CreateOrderRequest

# Call unary RPC
grpcurl -plaintext \
  -H "Authorization: Bearer <token>" \
  -d '{
    "user_id": "user-123",
    "items": [{"product_id": "prod-456", "quantity": 2}],
    "shipping_address": {
      "street": "123 Sukhumvit Rd",
      "city": "Bangkok",
      "postal_code": "10110",
      "country_code": "TH"
    },
    "idempotency_key": "order-req-uuid-here"
  }' \
  localhost:50051 \
  order.v1.OrderService/CreateOrder

# Stream RPC
grpcurl -plaintext \
  -H "Authorization: Bearer <token>" \
  -d '{"order_id": "order-789"}' \
  localhost:50051 \
  order.v1.OrderService/WatchOrder
```

---

## Proto Schema Evolution (Backward Compatibility)

```protobuf
// proto/order/v1/order.proto - Best practices สำหรับ backward compatibility

message Order {
  // Field numbers ห้ามเปลี่ยน หรือ reuse
  string id = 1;
  string user_id = 2;
  OrderStatus status = 3;
  
  // ถ้าต้องการลบ field ให้ใช้ reserved แทนการลบ
  reserved 4;  // was: string old_field = 4;
  reserved "old_field";
  
  // เพิ่ม fields ใหม่ได้โดยใช้ field number ใหม่ (สูงกว่าเดิม)
  repeated OrderItem items = 5;
  Money total_amount = 6;
  
  // New fields in v2 - clients เก่าจะ ignore
  string tracking_number = 10;
  DeliveryInfo delivery_info = 11;
}

// ถ้าเปลี่ยน Enum ให้เพิ่มค่าใหม่ ไม่ลบค่าเดิม
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  // ห้ามลบ values เดิม
  // ORDER_STATUS_DEPRECATED = 3;  // ← ถ้าต้องการ deprecate
  ORDER_STATUS_PROCESSING = 4;  // เพิ่มใหม่ได้
}
```

---

## สรุป

gRPC เป็นเครื่องมือที่ทรงพลังสำหรับ internal microservices communication:

| Feature | Best Practice |
|---------|---------------|
| Proto definition | ใช้ validation rules, reserved fields |
| Streaming | Unary default, streaming เมื่อจำเป็น |
| Error handling | Map domain errors to gRPC status codes |
| Interceptors | Auth, logging, retry ใน interceptor layer |
| Health check | Implement gRPC health checking protocol |
| Load balancing | Client-side LB + K8s DNS |
| gRPC-web | Envoy proxy สำหรับ browser clients |
| Reflection | เปิดเฉพาะ non-production |
| Schema evolution | ไม่เปลี่ยน field numbers, ใช้ reserved |

ข้อควรระวัง:
1. **gRPC ไม่เหมาะกับ public API** เนื่องจาก browser support ยังจำกัด
2. **Backward compatibility** — ต้องระวังเรื่อง proto schema changes มาก
3. **Debugging** ยากกว่า REST เนื่องจาก binary format — ใช้ grpcurl/grpcui
4. **ไม่มี built-in caching** ต้องทำเองถ้าต้องการ

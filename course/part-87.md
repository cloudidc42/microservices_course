# Part 87: gRPC in Microservices

## บทนำ

gRPC (gRPC Remote Procedure Call) เป็น high-performance RPC framework จาก Google ที่ใช้ Protocol Buffers เป็น interface definition language และ HTTP/2 เป็น transport layer ใน Microservices gRPC เหมาะสำหรับ internal service-to-service communication ที่ต้องการ performance สูงและ type safety

---

## 1. gRPC vs REST Comparison

### 1.1 Feature Comparison

```typescript
/**
 * gRPC vs REST Comparison:
 * 
 * Feature          | gRPC                    | REST
 * ------------------|-------------------------|------------------
 * Protocol         | HTTP/2                  | HTTP/1.1 or HTTP/2
 * Message Format   | Protocol Buffers (binary)| JSON (text)
 * Schema           | .proto files            | OpenAPI (optional)
 * Streaming        | Bidirectional           | Limited (SSE, WebSocket)
 * Code Generation  | Built-in                | Optional (OpenAPI generator)
 * Browser Support  | Via gRPC-Web proxy      | Native
 * Latency          | ~20-30% better          | Baseline
 * Payload Size     | 5-10x smaller           | Baseline
 * Type Safety      | Strong (compile-time)   | Depends on tooling
 * Error Handling   | Status codes            | HTTP status codes
 * Load Balancing   | L7 (connection-level)   | L7 (request-level)
 */

// ตัวอย่าง: REST ทำแบบนี้
fetch('/api/users/123')
  .then(r => r.json())
  .then(user => console.log(user));

// ตัวอย่าง: gRPC ทำแบบนี้ (strongly typed)
const user = await userClient.getUser({ id: '123' });
console.log(user.name, user.email);
```

---

## 2. Protocol Buffers Schema Design

### 2.1 Proto File Design Best Practices

```protobuf
// proto/user.proto
syntax = "proto3";

package user.v1;

option go_package = "github.com/example/user/v1";
option java_package = "com.example.user.v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/field_mask.proto";

// User message definition
message User {
  string id = 1;                                    // Field numbers ไม่เปลี่ยน
  string name = 2;
  string email = 3;
  UserRole role = 4;
  bool is_active = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
  
  // Nested message
  Address address = 8;
  
  // Optional field (proto3 ต้องใช้ optional keyword)
  optional string phone_number = 9;
  
  // Reserved fields (อย่าลบ field เก่า ให้ reserve แทน)
  reserved 10, 11;
  reserved "old_field_name";
}

message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;  // ต้องมี 0 value เสมอ
  USER_ROLE_USER = 1;
  USER_ROLE_ADMIN = 2;
  USER_ROLE_MODERATOR = 3;
}

// Request/Response messages
message GetUserRequest {
  string user_id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  UserRole role = 3;
  optional Address address = 4;
}

message UpdateUserRequest {
  string user_id = 1;
  string name = 2;
  optional string phone_number = 3;
  optional Address address = 4;
  
  // Field mask: ระบุว่า field ไหนที่จะ update
  google.protobuf.FieldMask update_mask = 5;
}

message ListUsersRequest {
  int32 page_size = 1;         // สูงสุดไม่เกิน 100
  string page_token = 2;       // cursor for pagination
  string filter = 3;           // "role=ADMIN AND is_active=true"
  string order_by = 4;         // "created_at DESC"
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;  // ว่างหมายความว่าไม่มีหน้าถัดไป
  int32 total_size = 3;        // จำนวนทั้งหมด (ถ้ารู้)
}

// UserService definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (User);
  
  // Unary with Empty response
  rpc DeleteUser(GetUserRequest) returns (google.protobuf.Empty);
  
  // Unary
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  
  // Server streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateUsersResponse);
  
  // Bidirectional streaming
  rpc SyncUsers(stream SyncUserRequest) returns (stream SyncUserResponse);
}

message BatchCreateUsersResponse {
  repeated User created_users = 1;
  repeated string failed_emails = 2;
  int32 success_count = 3;
  int32 failure_count = 4;
}

message SyncUserRequest {
  oneof operation {
    CreateUserRequest create = 1;
    UpdateUserRequest update = 2;
    GetUserRequest delete = 3;
  }
}

message SyncUserResponse {
  string operation_id = 1;
  bool success = 2;
  optional string error_message = 3;
  optional User user = 4;
}
```

### 2.2 Proto Compilation

```json
// package.json scripts
{
  "scripts": {
    "proto:compile": "grpc_tools_node_protoc --js_out=import_style=commonjs,binary:./src/generated --grpc_out=grpc_js:./src/generated --proto_path=./proto ./proto/**/*.proto",
    "proto:compile:ts": "grpc_tools_node_protoc_plugin --plugin=protoc-gen-ts=./node_modules/.bin/protoc-gen-ts --ts_out=grpc_js:./src/generated --grpc_out=grpc_js:./src/generated --proto_path=./proto ./proto/**/*.proto"
  }
}
```

```yaml
# buf.yaml - Modern proto tooling
version: v1
deps:
  - buf.build/googleapis/googleapis
breaking:
  use:
    - FILE
lint:
  use:
    - STANDARD
  except:
    - PACKAGE_VERSION_SUFFIX
```

---

## 3. gRPC Server Implementation

```typescript
// src/grpc/user-server.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

const PROTO_PATH = path.join(__dirname, '../../proto/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
  includeDirs: ['./proto'],
});

const userProto = grpc.loadPackageDefinition(packageDefinition) as any;

// Better: use generated types with ts-proto
import {
  UserServiceServer,
  UserServiceService,
  GetUserRequest,
  CreateUserRequest,
  ListUsersRequest,
  User,
  UserRole,
} from './generated/user';
import { ServerUnaryCall, sendUnaryData, ServerWritableStream, ServerReadableStream, ServerDuplexStream } from '@grpc/grpc-js';
import { Empty } from 'google-protobuf/google/protobuf/empty_pb';

export class UserGrpcServer implements UserServiceServer {
  [name: string]: any;
  
  constructor(private userService: UserService) {}

  // Unary RPC
  async getUser(
    call: ServerUnaryCall<GetUserRequest, User>,
    callback: sendUnaryData<User>
  ): Promise<void> {
    const { user_id } = call.request;
    
    try {
      const user = await this.userService.findById(user_id);
      
      if (!user) {
        callback({
          code: grpc.status.NOT_FOUND,
          message: `User ${user_id} not found`,
        });
        return;
      }
      
      callback(null, {
        id: user.id,
        name: user.name,
        email: user.email,
        role: UserRole[user.role as keyof typeof UserRole],
        is_active: user.isActive,
        created_at: {
          seconds: Math.floor(user.createdAt.getTime() / 1000),
          nanos: (user.createdAt.getTime() % 1000) * 1e6,
        },
        updated_at: {
          seconds: Math.floor(user.updatedAt.getTime() / 1000),
          nanos: (user.updatedAt.getTime() % 1000) * 1e6,
        },
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: `Internal error: ${(error as Error).message}`,
      });
    }
  }

  // Server Streaming RPC
  async listUsers(
    call: ServerWritableStream<ListUsersRequest, User>
  ): Promise<void> {
    const { page_size, filter } = call.request;
    
    try {
      const cursor = this.userService.streamUsers({
        limit: page_size || 50,
        filter,
      });
      
      for await (const user of cursor) {
        // ตรวจสอบว่า client ยกเลิกแล้วหรือไม่
        if (call.cancelled) break;
        
        call.write({
          id: user.id,
          name: user.name,
          email: user.email,
          role: UserRole[user.role as keyof typeof UserRole],
          is_active: user.isActive,
        });
      }
      
      call.end();
    } catch (error) {
      call.destroy(error as Error);
    }
  }

  // Client Streaming RPC
  async batchCreateUsers(
    call: ServerReadableStream<CreateUserRequest, any>,
    callback: sendUnaryData<any>
  ): Promise<void> {
    const createdUsers: User[] = [];
    const failedEmails: string[] = [];
    
    try {
      for await (const request of call) {
        try {
          const user = await this.userService.create({
            name: request.name,
            email: request.email,
          });
          createdUsers.push(user as any);
        } catch (error) {
          failedEmails.push(request.email);
        }
      }
      
      callback(null, {
        created_users: createdUsers,
        failed_emails: failedEmails,
        success_count: createdUsers.length,
        failure_count: failedEmails.length,
      });
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: (error as Error).message,
      });
    }
  }

  // Bidirectional Streaming RPC
  async syncUsers(
    call: ServerDuplexStream<any, any>
  ): Promise<void> {
    call.on('data', async (request: any) => {
      const operationId = crypto.randomUUID();
      
      try {
        let user: any;
        
        if (request.create) {
          user = await this.userService.create(request.create);
          call.write({ operation_id: operationId, success: true, user });
        } else if (request.update) {
          user = await this.userService.update(request.update.user_id, request.update);
          call.write({ operation_id: operationId, success: true, user });
        } else if (request.delete) {
          await this.userService.delete(request.delete.user_id);
          call.write({ operation_id: operationId, success: true });
        }
      } catch (error) {
        call.write({
          operation_id: operationId,
          success: false,
          error_message: (error as Error).message,
        });
      }
    });
    
    call.on('end', () => {
      call.end();
    });
    
    call.on('error', (error) => {
      console.error('Sync stream error:', error);
    });
  }
}

// Server startup
export function startGrpcServer(service: UserGrpcServer): grpc.Server {
  const server = new grpc.Server({
    'grpc.max_receive_message_length': 10 * 1024 * 1024, // 10MB
    'grpc.max_send_message_length': 10 * 1024 * 1024,
    'grpc.keepalive_time_ms': 30000,
    'grpc.keepalive_timeout_ms': 5000,
  });
  
  server.addService(UserServiceService, service);
  
  const port = process.env.GRPC_PORT || '50051';
  const credentials = process.env.TLS_CERT_PATH
    ? grpc.ServerCredentials.createSsl(
        Buffer.from(require('fs').readFileSync(process.env.TLS_CA_CERT!)),
        [{
          cert_chain: Buffer.from(require('fs').readFileSync(process.env.TLS_CERT_PATH!)),
          private_key: Buffer.from(require('fs').readFileSync(process.env.TLS_KEY_PATH!)),
        }],
        true
      )
    : grpc.ServerCredentials.createInsecure();
  
  server.bindAsync(`0.0.0.0:${port}`, credentials, (error, port) => {
    if (error) {
      console.error('Failed to bind gRPC server:', error);
      process.exit(1);
    }
    console.log(`gRPC server listening on port ${port}`);
    server.start();
  });
  
  return server;
}
```

---

## 4. gRPC Client Implementation

```typescript
// src/grpc/user-client.ts
import * as grpc from '@grpc/grpc-js';
import { UserServiceClient } from './generated/user';

export class UserGrpcClient {
  private client: UserServiceClient;
  
  constructor(address: string) {
    this.client = new UserServiceClient(
      address,
      process.env.TLS_ENABLED === 'true'
        ? grpc.credentials.createSsl()
        : grpc.credentials.createInsecure(),
      {
        'grpc.keepalive_time_ms': 30000,
        'grpc.keepalive_permit_without_calls': 1,
        'grpc.service_config': JSON.stringify({
          loadBalancingConfig: [{ round_robin: {} }],
          methodConfig: [{
            name: [{ service: 'user.v1.UserService' }],
            retryPolicy: {
              maxAttempts: 3,
              initialBackoff: '0.1s',
              maxBackoff: '1s',
              backoffMultiplier: 2,
              retryableStatusCodes: ['UNAVAILABLE', 'RESOURCE_EXHAUSTED'],
            },
            timeout: '5s',
          }],
        }),
      }
    );
  }

  async getUser(userId: string): Promise<User> {
    return new Promise((resolve, reject) => {
      const metadata = new grpc.Metadata();
      metadata.add('x-correlation-id', getCorrelationId());
      metadata.add('authorization', `Bearer ${getAuthToken()}`);
      
      this.client.getUser(
        { user_id: userId },
        metadata,
        (error, response) => {
          if (error) {
            if (error.code === grpc.status.NOT_FOUND) {
              reject(new NotFoundError(`User ${userId} not found`));
            } else {
              reject(new Error(`gRPC error: ${error.message}`));
            }
            return;
          }
          resolve(response!);
        }
      );
    });
  }

  // Server streaming - returns async iterator
  async *listUsers(filter?: string): AsyncGenerator<User> {
    const metadata = new grpc.Metadata();
    metadata.add('authorization', `Bearer ${getAuthToken()}`);
    
    const call = this.client.listUsers(
      { filter, page_size: 100 },
      metadata
    );
    
    try {
      for await (const user of call) {
        yield user;
      }
    } finally {
      call.cancel();
    }
  }

  // Client streaming
  async batchCreate(users: Array<{ name: string; email: string }>): Promise<any> {
    return new Promise((resolve, reject) => {
      const call = this.client.batchCreateUsers((error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
      
      // Write all messages
      for (const user of users) {
        call.write({ name: user.name, email: user.email });
      }
      
      call.end();
    });
  }

  async close(): Promise<void> {
    this.client.close();
  }
}
```

---

## 5. gRPC Interceptors

### 5.1 Server Interceptors (Authentication, Logging, Tracing)

```typescript
// src/grpc/interceptors/auth.interceptor.ts
import * as grpc from '@grpc/grpc-js';

export function createAuthInterceptor(jwtService: JWTService) {
  return function authInterceptor(
    options: grpc.InterceptorOptions,
    nextCall: grpc.NextCall
  ): grpc.InterceptingCall {
    return new grpc.InterceptingCall(nextCall(options), {
      start: function (metadata, listener, next) {
        const token = metadata.get('authorization')[0]?.toString().split(' ')[1];
        
        if (!token) {
          const call = nextCall(options);
          call.start(metadata, {
            onReceiveMessage: (message, next) => next(message),
            onReceiveStatus: (status, next) => {
              next({
                code: grpc.status.UNAUTHENTICATED,
                details: 'Authentication required',
                metadata: new grpc.Metadata(),
              });
            },
          });
          return;
        }
        
        try {
          const decoded = jwtService.verify(token);
          metadata.add('x-user-id', decoded.userId);
          metadata.add('x-user-role', decoded.role);
          next(metadata, listener);
        } catch (error) {
          next(metadata, {
            onReceiveMessage: (message, next) => next(message),
            onReceiveStatus: (status, next) => {
              next({
                code: grpc.status.UNAUTHENTICATED,
                details: 'Invalid token',
                metadata: new grpc.Metadata(),
              });
            },
          });
        }
      },
    });
  };
}

// Logging Interceptor
export function createLoggingInterceptor() {
  return function loggingInterceptor(
    options: grpc.InterceptorOptions,
    nextCall: grpc.NextCall
  ): grpc.InterceptingCall {
    const startTime = Date.now();
    const method = options.method_definition.path;
    
    return new grpc.InterceptingCall(nextCall(options), {
      start: function (metadata, listener, next) {
        const correlationId = metadata.get('x-correlation-id')[0]?.toString() || 
                             crypto.randomUUID();
        
        console.info({
          message: 'gRPC request started',
          method,
          correlationId,
        });
        
        next(metadata, {
          onReceiveMessage: (message, next) => next(message),
          onReceiveStatus: (status, next) => {
            console.info({
              message: 'gRPC request completed',
              method,
              status: status.code,
              duration: Date.now() - startTime,
              correlationId,
            });
            next(status);
          },
        });
      },
    });
  };
}

// Apply interceptors to server
server.addService(UserServiceService, userServiceImpl, {
  interceptors: [
    createLoggingInterceptor(),
    createAuthInterceptor(jwtService),
  ],
});
```

---

## 6. gRPC-Web for Browser Clients

```typescript
// web/src/grpc-web-client.ts
import { GrpcWebFetchTransport } from "@protobuf-ts/grpcweb-transport";
import { UserServiceClient } from "./generated/user.client";

const transport = new GrpcWebFetchTransport({
  baseUrl: process.env.REACT_APP_GRPC_WEB_URL || 'http://localhost:8080',
  fetchInit: {
    credentials: 'include',
  },
});

const userClient = new UserServiceClient(transport);

// React Hook
export function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    const call = userClient.getUser({
      userId,
    });
    
    call.response.then(response => {
      setUser(response.response);
      setLoading(false);
    }).catch(err => {
      setError(err);
      setLoading(false);
    });
    
    return () => call.cancel();
  }, [userId]);
  
  return { user, loading, error };
}

// Server streaming hook
export function useUserStream(filter?: string) {
  const [users, setUsers] = useState<User[]>([]);
  
  useEffect(() => {
    const call = userClient.listUsers({ filter });
    
    (async () => {
      try {
        for await (const user of call.responses) {
          setUsers(prev => [...prev, user]);
        }
      } catch (error) {
        if ((error as any).code !== 'CANCELLED') {
          console.error(error);
        }
      }
    })();
    
    return () => call.cancel();
  }, [filter]);
  
  return users;
}
```

```yaml
# Envoy proxy config สำหรับ gRPC-Web
# envoy.yaml
static_resources:
  listeners:
    - name: listener_0
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 8080
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: AUTO
                stat_prefix: ingress_http
                route_config:
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/user.v1.UserService" }
                          route:
                            cluster: user_grpc_service
                      cors:
                        allow_origin_string_match:
                          - prefix: "*"
                        allow_methods: GET, PUT, DELETE, POST, OPTIONS
                        allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout,authorization
                        max_age: "1728000"
                        expose_headers: grpc-status,grpc-message
                http_filters:
                  - name: envoy.filters.http.grpc_web
                  - name: envoy.filters.http.cors
                  - name: envoy.filters.http.router

  clusters:
    - name: user_grpc_service
      connect_timeout: 0.25s
      type: LOGICAL_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}
      load_assignment:
        cluster_name: user_grpc_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: user-service
                      port_value: 50051
```

---

## 7. gRPC Error Handling

```typescript
// src/grpc/error-handler.ts
import * as grpc from '@grpc/grpc-js';

// Mapping application errors to gRPC status codes
export class GrpcErrorHandler {
  static toGrpcError(error: Error): grpc.ServiceError {
    if (error instanceof NotFoundError) {
      return {
        code: grpc.status.NOT_FOUND,
        details: error.message,
        metadata: new grpc.Metadata(),
        name: 'ServiceError',
        message: error.message,
      };
    }
    
    if (error instanceof ValidationError) {
      return {
        code: grpc.status.INVALID_ARGUMENT,
        details: error.message,
        metadata: new grpc.Metadata(),
        name: 'ServiceError',
        message: error.message,
      };
    }
    
    if (error instanceof AuthenticationError) {
      return {
        code: grpc.status.UNAUTHENTICATED,
        details: error.message,
        metadata: new grpc.Metadata(),
        name: 'ServiceError',
        message: error.message,
      };
    }
    
    if (error instanceof AuthorizationError) {
      return {
        code: grpc.status.PERMISSION_DENIED,
        details: error.message,
        metadata: new grpc.Metadata(),
        name: 'ServiceError',
        message: error.message,
      };
    }
    
    if (error instanceof ConflictError) {
      return {
        code: grpc.status.ALREADY_EXISTS,
        details: error.message,
        metadata: new grpc.Metadata(),
        name: 'ServiceError',
        message: error.message,
      };
    }
    
    if (error instanceof RateLimitError) {
      const metadata = new grpc.Metadata();
      metadata.add('retry-after', '60');
      return {
        code: grpc.status.RESOURCE_EXHAUSTED,
        details: error.message,
        metadata,
        name: 'ServiceError',
        message: error.message,
      };
    }
    
    // Default: internal error
    return {
      code: grpc.status.INTERNAL,
      details: 'An internal error occurred',
      metadata: new grpc.Metadata(),
      name: 'ServiceError',
      message: error.message,
    };
  }
}

// Client-side error handling
export function handleGrpcError(error: grpc.ServiceError): never {
  const message = error.details || error.message;
  
  switch (error.code) {
    case grpc.status.NOT_FOUND:
      throw new NotFoundError(message);
    case grpc.status.INVALID_ARGUMENT:
      throw new ValidationError(message);
    case grpc.status.UNAUTHENTICATED:
      throw new AuthenticationError(message);
    case grpc.status.PERMISSION_DENIED:
      throw new AuthorizationError(message);
    case grpc.status.ALREADY_EXISTS:
      throw new ConflictError(message);
    case grpc.status.RESOURCE_EXHAUSTED:
      throw new RateLimitError(message);
    case grpc.status.UNAVAILABLE:
      throw new ServiceUnavailableError(message);
    default:
      throw new Error(`gRPC error ${error.code}: ${message}`);
  }
}
```

---

## 8. gRPC Health Checking Protocol

```protobuf
// proto/grpc/health/v1/health.proto (standard)
syntax = "proto3";

package grpc.health.v1;

message HealthCheckRequest {
  string service = 1;  // ว่างหมายถึงตรวจสอบ server โดยรวม
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

```typescript
// src/grpc/health.service.ts
import * as grpc from '@grpc/grpc-js';
import { 
  HealthImplementation,
  ServingStatusMap 
} from 'grpc-health-check';

export class HealthService {
  private health: HealthImplementation;
  private services: Map<string, ServingStatusMap[keyof ServingStatusMap]> = new Map();

  constructor() {
    this.health = new HealthImplementation({});
  }

  setStatus(service: string, status: 'SERVING' | 'NOT_SERVING'): void {
    this.services.set(service, status === 'SERVING' 
      ? HealthImplementation.status.SERVING 
      : HealthImplementation.status.NOT_SERVING);
    
    this.health.setStatus(service, this.services.get(service)!);
  }

  addToServer(server: grpc.Server): void {
    // Add health service to gRPC server
    this.health.addToServer(server);
    
    // Set initial status
    this.setStatus('', 'SERVING');
    this.setStatus('user.v1.UserService', 'SERVING');
  }

  async checkDependencies(): Promise<void> {
    try {
      await this.dbService.ping();
      await this.redisService.ping();
      this.setStatus('user.v1.UserService', 'SERVING');
    } catch (error) {
      console.error('Health check failed:', error);
      this.setStatus('user.v1.UserService', 'NOT_SERVING');
    }
  }
}

// Kubernetes liveness/readiness probe
// kubectl -n prod exec pod -- grpc_health_probe -addr=:50051
```

---

## 9. gRPC Reflection for Debugging

```typescript
// src/grpc/reflection.ts
import { addReflection } from 'grpc-server-reflection';
import * as grpc from '@grpc/grpc-js';

export function addReflectionToServer(
  server: grpc.Server,
  protoFiles: string[]
): void {
  if (process.env.GRPC_REFLECTION_ENABLED !== 'true') {
    return;
  }
  
  addReflection(server, protoFiles);
  
  console.info('gRPC reflection enabled for debugging');
}

// Usage สำหรับ debug ด้วย grpcurl:
// grpcurl -plaintext localhost:50051 list
// grpcurl -plaintext localhost:50051 list user.v1.UserService
// grpcurl -plaintext -d '{"user_id":"123"}' localhost:50051 user.v1.UserService/GetUser
```

---

## 10. gRPC Load Balancing Strategies

```typescript
// src/grpc/load-balancer.ts

// Service mesh approach (Kubernetes + Istio/Linkerd)
// DNS resolution: user-service.default.svc.cluster.local:50051

// Client-side load balancing
const client = new UserServiceClient(
  'dns:///user-service:50051',  // DNS round-robin
  grpc.credentials.createInsecure(),
  {
    'grpc.service_config': JSON.stringify({
      loadBalancingConfig: [
        { round_robin: {} },
        // หรือ
        // { grpclb: {} }
        // { pick_first: {} }
      ],
    }),
    // Keepalive สำหรับ detect dead connections
    'grpc.keepalive_time_ms': 30000,
    'grpc.keepalive_timeout_ms': 5000,
    'grpc.keepalive_permit_without_calls': 1,
  }
);

// gRPC over Kubernetes Service
// user-service headless service allows client to discover all pods
// kubectl create service clusterip user-service-headless --clusterip=None
```

```yaml
# kubernetes/user-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service-headless
  namespace: production
spec:
  clusterIP: None  # Headless service
  selector:
    app: user-service
  ports:
    - name: grpc
      port: 50051
      targetPort: 50051
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: user-service
          image: user-service:latest
          ports:
            - containerPort: 50051
              name: grpc
          readinessProbe:
            exec:
              command: ["/bin/grpc_health_probe", "-addr=:50051"]
            initialDelaySeconds: 5
          livenessProbe:
            exec:
              command: ["/bin/grpc_health_probe", "-addr=:50051"]
            initialDelaySeconds: 10
```

---

## สรุปท้ายบท

| หัวข้อ | gRPC | REST |
|--------|------|------|
| Serialization | Protobuf (binary, ~5-10x smaller) | JSON (text) |
| Latency | ~20-30% faster | Baseline |
| Type Safety | Compile-time | Runtime (with tools) |
| Streaming | Native bidirectional | Limited |
| Browser Support | Via proxy (gRPC-Web) | Native |
| Learning Curve | สูงกว่า | ต่ำกว่า |
| Tooling | ดี (grpcurl, Postman) | ดีมาก |
| Caching | ยาก | ง่าย (HTTP cache headers) |

### เมื่อไหร่ควรใช้ gRPC

1. **Internal service-to-service** communication ที่ต้องการ performance สูง
2. **Real-time streaming** data (server push, bidirectional)
3. **Polyglot microservices** ที่ใช้หลายภาษา (code generation จาก proto)
4. **Strongly-typed API** ที่ต้องการ compile-time safety
5. **Mobile/IoT** clients ที่ bandwidth มีจำกัด

### เมื่อไหร่ควรใช้ REST

1. **Public APIs** ที่ต้องการ browser compatibility
2. **Simple CRUD operations** ที่ไม่ต้องการ streaming
3. **Team ที่ familiar กับ REST** มากกว่า
4. **Caching** เป็น requirement หลัก

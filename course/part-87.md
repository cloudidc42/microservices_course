# Part 87: gRPC in Microservices

## บทนำ

gRPC (Google Remote Procedure Call) เป็น framework สำหรับ RPC ที่ใช้ Protocol Buffers เป็น interface definition language และ HTTP/2 เป็น transport layer บทนี้จะครอบคลุมการใช้งาน gRPC ใน TypeScript Microservices ตั้งแต่พื้นฐานจนถึง advanced patterns

---

## 1. gRPC vs REST vs GraphQL เปรียบเทียบ

### 1.1 ตารางเปรียบเทียบ

| Feature | gRPC | REST | GraphQL |
|---------|------|------|---------|
| Protocol | HTTP/2 | HTTP/1.1 หรือ HTTP/2 | HTTP/1.1 หรือ HTTP/2 |
| Data Format | Protocol Buffers (binary) | JSON/XML (text) | JSON (text) |
| Type Safety | Strong (protobuf schema) | Weak (OpenAPI optional) | Strong (GraphQL schema) |
| Streaming | รองรับ 4 รูปแบบ | Limited (SSE, WebSocket) | Subscriptions |
| Code Generation | อัตโนมัติจาก .proto | ต้องทำเอง หรือ OpenAPI | Code gen จาก schema |
| Browser Support | ต้องการ gRPC-Web proxy | Native | Native |
| Performance | สูงมาก (binary + multiplexing) | ปานกลาง | ปานกลาง |
| Learning Curve | สูง | ต่ำ | ปานกลาง |
| Use Case | Internal microservices | Public APIs | Flexible data fetching |
| Contract-First | บังคับ | Optional | บังคับ |

### 1.2 เมื่อไหรควรใช้อะไร

**ใช้ gRPC เมื่อ:**
- Internal microservices communication
- ต้องการ performance สูงสุด
- ต้องการ streaming (เช่น real-time data)
- ทีมต้องการ strong type safety

**ใช้ REST เมื่อ:**
- Public APIs ที่ต้องการ simplicity
- Browser clients โดยตรง
- ทีมที่ไม่คุ้นเคยกับ protobuf
- Simple CRUD operations

**ใช้ GraphQL เมื่อ:**
- Client ต้องการ query data แบบยืดหยุ่น
- Multiple clients ที่ต้องการ data ต่างกัน
- Aggregation หลาย services

---

## 2. Protocol Buffers: Proto Files

### 2.1 user.proto

```protobuf
// proto/user.proto
syntax = "proto3";

package user.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

option go_package = "github.com/mycompany/proto/user/v1";

// Enums
enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_CUSTOMER = 1;
  USER_ROLE_ADMIN = 2;
  USER_ROLE_MANAGER = 3;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}

// Messages
message User {
  string id = 1;
  string email = 2;
  string name = 3;
  UserRole role = 4;
  UserStatus status = 5;
  google.protobuf.Timestamp created_at = 6;
  google.protobuf.Timestamp updated_at = 7;
  map<string, string> metadata = 8;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  string password = 3;
  UserRole role = 4;
}

message CreateUserResponse {
  User user = 1;
}

message GetUserRequest {
  string id = 1;
}

message GetUserResponse {
  User user = 1;
}

message UpdateUserRequest {
  string id = 1;
  optional string name = 2;
  optional string email = 3;
  optional UserRole role = 4;
  optional UserStatus status = 5;
}

message UpdateUserResponse {
  User user = 1;
}

message DeleteUserRequest {
  string id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  string filter = 3;
  string order_by = 4;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message WatchUsersRequest {
  repeated string user_ids = 1;
  repeated string event_types = 2;
}

message UserEvent {
  string event_type = 1;
  User user = 2;
  google.protobuf.Timestamp event_time = 3;
}

// Service definition
service UserService {
  // Unary RPCs
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);

  // Server streaming
  rpc WatchUsers(WatchUsersRequest) returns (stream UserEvent);
}
```

### 2.2 order.proto

```protobuf
// proto/order.proto
syntax = "proto3";

package order.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/wrappers.proto";

option go_package = "github.com/mycompany/proto/order/v1";

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_PROCESSING = 3;
  ORDER_STATUS_SHIPPED = 4;
  ORDER_STATUS_DELIVERED = 5;
  ORDER_STATUS_CANCELLED = 6;
  ORDER_STATUS_REFUNDED = 7;
}

message OrderItem {
  string product_id = 1;
  string product_name = 2;
  int32 quantity = 3;
  double unit_price = 4;
  double subtotal = 5;
}

message ShippingAddress {
  string street = 1;
  string city = 2;
  string state = 3;
  string country = 4;
  string postal_code = 5;
}

message Order {
  string id = 1;
  string customer_id = 2;
  repeated OrderItem items = 3;
  double total_amount = 4;
  OrderStatus status = 5;
  ShippingAddress shipping_address = 6;
  google.protobuf.Timestamp created_at = 7;
  google.protobuf.Timestamp updated_at = 8;
  string tracking_number = 9;
}

message CreateOrderRequest {
  string customer_id = 1;
  repeated OrderItem items = 2;
  ShippingAddress shipping_address = 3;
}

message CreateOrderResponse {
  Order order = 1;
}

message GetOrderRequest {
  string id = 1;
}

message GetOrderResponse {
  Order order = 1;
}

message StreamOrderStatusRequest {
  string order_id = 1;
}

message OrderStatusUpdate {
  string order_id = 1;
  OrderStatus status = 2;
  string message = 3;
  google.protobuf.Timestamp timestamp = 4;
}

message BatchCreateOrderRequest {
  repeated CreateOrderRequest orders = 1;
}

message BatchCreateOrderResponse {
  repeated Order orders = 1;
  int32 success_count = 2;
  int32 failure_count = 3;
}

// สำหรับ bidirectional streaming
message OrderChatMessage {
  string sender = 1;
  string content = 2;
  string order_id = 3;
  google.protobuf.Timestamp timestamp = 4;
}

service OrderService {
  // Unary
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);

  // Server streaming - ส่ง status updates ไปยัง client
  rpc StreamOrderStatus(StreamOrderStatusRequest) returns (stream OrderStatusUpdate);

  // Client streaming - batch upload orders
  rpc BatchCreateOrder(stream CreateOrderRequest) returns (BatchCreateOrderResponse);

  // Bidirectional streaming - chat support
  rpc OrderChat(stream OrderChatMessage) returns (stream OrderChatMessage);
}
```

---

## 3. gRPC Server: TypeScript Implementation

### 3.1 Generate Types จาก Proto

```bash
# ติดตั้ง tools
npm install @grpc/grpc-js @grpc/proto-loader
npm install --save-dev grpc-tools grpc_tools_node_protoc_ts
npm install --save-dev @types/node

# Generate TypeScript types
npx grpc_tools_node_protoc \
  --js_out=import_style=commonjs,binary:./generated \
  --grpc_out=grpc_js:./generated \
  --ts_out=./generated \
  --proto_path=./proto \
  ./proto/user.proto \
  ./proto/order.proto
```

### 3.2 UserService Server Implementation

```typescript
// grpc/user-service-server.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

// Load proto definition
const PROTO_PATH = path.join(__dirname, '../proto/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
  includeDirs: [path.join(__dirname, '../proto')],
});

const proto = grpc.loadPackageDefinition(packageDefinition) as any;
const UserServiceProto = proto.user.v1;

// Types (normally generated from proto)
interface User {
  id: string;
  email: string;
  name: string;
  role: string;
  status: string;
  created_at: { seconds: number; nanos: number };
  updated_at: { seconds: number; nanos: number };
  metadata: Record<string, string>;
}

interface CreateUserRequest {
  email: string;
  name: string;
  password: string;
  role: string;
}

// In-memory database (จำลองสำหรับตัวอย่าง)
class UserRepository {
  private users: Map<string, User> = new Map();
  private emailIndex: Map<string, string> = new Map();

  async create(data: CreateUserRequest): Promise<User> {
    if (this.emailIndex.has(data.email)) {
      throw new Error(`User with email ${data.email} already exists`);
    }

    const now = { seconds: Math.floor(Date.now() / 1000), nanos: 0 };
    const user: User = {
      id: `user-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
      email: data.email,
      name: data.name,
      role: data.role || 'USER_ROLE_CUSTOMER',
      status: 'USER_STATUS_ACTIVE',
      created_at: now,
      updated_at: now,
      metadata: {},
    };

    this.users.set(user.id, user);
    this.emailIndex.set(user.email, user.id);
    return user;
  }

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) || null;
  }

  async update(id: string, updates: Partial<User>): Promise<User | null> {
    const user = this.users.get(id);
    if (!user) return null;

    const updated = {
      ...user,
      ...updates,
      updated_at: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
    };
    this.users.set(id, updated);
    return updated;
  }

  async delete(id: string): Promise<boolean> {
    const user = this.users.get(id);
    if (!user) return false;
    this.emailIndex.delete(user.email);
    return this.users.delete(id);
  }

  async list(pageSize = 10, pageToken?: string): Promise<{ users: User[]; nextToken?: string }> {
    const allUsers = Array.from(this.users.values());
    const startIndex = pageToken ? parseInt(Buffer.from(pageToken, 'base64').toString()) : 0;
    const page = allUsers.slice(startIndex, startIndex + pageSize);
    const nextIndex = startIndex + page.length;
    const nextToken = nextIndex < allUsers.length
      ? Buffer.from(nextIndex.toString()).toString('base64')
      : undefined;

    return { users: page, nextToken };
  }
}

// gRPC Service Implementation
class UserServiceServer {
  private repository: UserRepository;
  private watchStreams: Map<string, grpc.ServerWritableStream<any, any>[]> = new Map();

  constructor() {
    this.repository = new UserRepository();
  }

  // Unary RPC: CreateUser
  async createUser(
    call: grpc.ServerUnaryCall<CreateUserRequest, any>,
    callback: grpc.sendUnaryData<any>
  ): Promise<void> {
    try {
      const request = call.request;

      // Validation
      if (!request.email || !request.name || !request.password) {
        return callback({
          code: grpc.status.INVALID_ARGUMENT,
          message: 'email, name, and password are required',
        });
      }

      const user = await this.repository.create(request);
      callback(null, { user });
    } catch (error: any) {
      if (error.message.includes('already exists')) {
        callback({
          code: grpc.status.ALREADY_EXISTS,
          message: error.message,
        });
      } else {
        callback({
          code: grpc.status.INTERNAL,
          message: `Internal error: ${error.message}`,
        });
      }
    }
  }

  // Unary RPC: GetUser
  async getUser(
    call: grpc.ServerUnaryCall<{ id: string }, any>,
    callback: grpc.sendUnaryData<any>
  ): Promise<void> {
    try {
      const user = await this.repository.findById(call.request.id);

      if (!user) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: `User ${call.request.id} not found`,
        });
      }

      callback(null, { user });
    } catch (error: any) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message,
      });
    }
  }

  // Unary RPC: UpdateUser
  async updateUser(
    call: grpc.ServerUnaryCall<any, any>,
    callback: grpc.sendUnaryData<any>
  ): Promise<void> {
    try {
      const { id, ...updates } = call.request;
      const user = await this.repository.update(id, updates);

      if (!user) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: `User ${id} not found`,
        });
      }

      // แจ้ง watchers
      this.notifyWatchers(id, 'USER_UPDATED', user);

      callback(null, { user });
    } catch (error: any) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message,
      });
    }
  }

  // Unary RPC: DeleteUser
  async deleteUser(
    call: grpc.ServerUnaryCall<{ id: string }, any>,
    callback: grpc.sendUnaryData<any>
  ): Promise<void> {
    try {
      const deleted = await this.repository.delete(call.request.id);

      if (!deleted) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: `User ${call.request.id} not found`,
        });
      }

      callback(null, {});
    } catch (error: any) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message,
      });
    }
  }

  // Unary RPC: ListUsers
  async listUsers(
    call: grpc.ServerUnaryCall<any, any>,
    callback: grpc.sendUnaryData<any>
  ): Promise<void> {
    try {
      const { page_size, page_token } = call.request;
      const { users, nextToken } = await this.repository.list(
        page_size || 10,
        page_token || undefined
      );

      callback(null, {
        users,
        next_page_token: nextToken || '',
        total_count: users.length,
      });
    } catch (error: any) {
      callback({
        code: grpc.status.INTERNAL,
        message: error.message,
      });
    }
  }

  // Server Streaming RPC: WatchUsers
  watchUsers(call: grpc.ServerWritableStream<any, any>): void {
    const { user_ids } = call.request;

    // ลงทะเบียน watcher สำหรับแต่ละ user
    for (const userId of user_ids) {
      if (!this.watchStreams.has(userId)) {
        this.watchStreams.set(userId, []);
      }
      this.watchStreams.get(userId)!.push(call);
    }

    // Cleanup เมื่อ client disconnect
    call.on('cancelled', () => {
      for (const userId of user_ids) {
        const streams = this.watchStreams.get(userId) || [];
        const index = streams.indexOf(call);
        if (index !== -1) {
          streams.splice(index, 1);
        }
      }
      console.log('Watch stream cancelled');
    });

    call.on('error', (error) => {
      console.error('Watch stream error:', error);
    });

    console.log(`Watching users: ${user_ids.join(', ')}`);
  }

  private notifyWatchers(userId: string, eventType: string, user: User): void {
    const streams = this.watchStreams.get(userId) || [];
    const event = {
      event_type: eventType,
      user,
      event_time: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
    };

    for (const stream of streams) {
      try {
        stream.write(event);
      } catch (error) {
        console.error('Error writing to watch stream:', error);
      }
    }
  }
}

// Start server
function createServer(): grpc.Server {
  const server = new grpc.Server({
    'grpc.max_receive_message_length': 10 * 1024 * 1024, // 10MB
    'grpc.max_send_message_length': 10 * 1024 * 1024,    // 10MB
    'grpc.keepalive_time_ms': 10000,
    'grpc.keepalive_timeout_ms': 5000,
  });

  const userService = new UserServiceServer();

  server.addService(UserServiceProto.UserService.service, {
    createUser: userService.createUser.bind(userService),
    getUser: userService.getUser.bind(userService),
    updateUser: userService.updateUser.bind(userService),
    deleteUser: userService.deleteUser.bind(userService),
    listUsers: userService.listUsers.bind(userService),
    watchUsers: userService.watchUsers.bind(userService),
  });

  return server;
}

const server = createServer();
server.bindAsync(
  '0.0.0.0:50051',
  grpc.ServerCredentials.createInsecure(),
  (error, port) => {
    if (error) {
      console.error('Failed to bind server:', error);
      process.exit(1);
    }
    server.start();
    console.log(`gRPC server running on port ${port}`);
  }
);
```

---

## 4. gRPC Client: Typed Client Wrapper พร้อม Connection Pooling

```typescript
// grpc/user-service-client.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

const PROTO_PATH = path.join(__dirname, '../proto/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const proto = grpc.loadPackageDefinition(packageDefinition) as any;

// Connection Pool
class GrpcConnectionPool {
  private clients: grpc.Client[] = [];
  private currentIndex = 0;

  constructor(
    private readonly ServiceClass: any,
    private readonly address: string,
    private readonly credentials: grpc.ChannelCredentials,
    private readonly poolSize: number = 5,
    private readonly options?: grpc.ClientOptions
  ) {
    this.initialize();
  }

  private initialize(): void {
    for (let i = 0; i < this.poolSize; i++) {
      const client = new this.ServiceClass(
        this.address,
        this.credentials,
        {
          ...this.options,
          'grpc.use_local_subchannel_pool': 1,
        }
      );
      this.clients.push(client);
    }
    console.log(`gRPC connection pool initialized: ${this.poolSize} connections`);
  }

  // Round-robin load balancing
  getClient(): grpc.Client {
    const client = this.clients[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.poolSize;
    return client;
  }

  close(): void {
    for (const client of this.clients) {
      client.close();
    }
    this.clients = [];
  }
}

// Typed UserService Client
interface CreateUserInput {
  email: string;
  name: string;
  password: string;
  role?: string;
}

interface User {
  id: string;
  email: string;
  name: string;
  role: string;
  status: string;
}

interface ListUsersOptions {
  pageSize?: number;
  pageToken?: string;
  filter?: string;
  orderBy?: string;
}

class UserServiceClient {
  private pool: GrpcConnectionPool;

  constructor(address: string = 'localhost:50051', poolSize = 5) {
    this.pool = new GrpcConnectionPool(
      proto.user.v1.UserService,
      address,
      grpc.credentials.createInsecure(),
      poolSize,
      {
        'grpc.max_receive_message_length': 10 * 1024 * 1024,
        'grpc.max_send_message_length': 10 * 1024 * 1024,
        'grpc.keepalive_time_ms': 10000,
        'grpc.keepalive_timeout_ms': 5000,
        'grpc.keepalive_permit_without_calls': 1,
      }
    );
  }

  private getDeadline(timeoutMs = 5000): grpc.Deadline {
    return new Date(Date.now() + timeoutMs);
  }

  private promisify<TRequest, TResponse>(
    method: (
      request: TRequest,
      metadata: grpc.Metadata,
      options: grpc.CallOptions,
      callback: (error: grpc.ServiceError | null, response: TResponse) => void
    ) => grpc.ClientUnaryCall
  ) {
    return (request: TRequest, timeoutMs?: number): Promise<TResponse> => {
      return new Promise((resolve, reject) => {
        const metadata = new grpc.Metadata();
        metadata.set('x-request-id', `req-${Date.now()}`);

        const client = this.pool.getClient() as any;
        const deadline = this.getDeadline(timeoutMs);

        method.call(
          client,
          request,
          metadata,
          { deadline },
          (error: grpc.ServiceError | null, response: TResponse) => {
            if (error) {
              reject(this.transformError(error));
            } else {
              resolve(response);
            }
          }
        );
      });
    };
  }

  private transformError(error: grpc.ServiceError): Error {
    const transformed = new Error(error.message);
    (transformed as any).code = error.code;
    (transformed as any).grpcStatus = grpc.status[error.code];
    return transformed;
  }

  // Unary RPCs
  async createUser(input: CreateUserInput): Promise<User> {
    const client = this.pool.getClient() as any;
    return new Promise((resolve, reject) => {
      client.createUser(
        input,
        new grpc.Metadata(),
        { deadline: this.getDeadline() },
        (error: grpc.ServiceError | null, response: any) => {
          if (error) reject(this.transformError(error));
          else resolve(response.user);
        }
      );
    });
  }

  async getUser(id: string): Promise<User | null> {
    try {
      const client = this.pool.getClient() as any;
      return new Promise((resolve, reject) => {
        client.getUser(
          { id },
          new grpc.Metadata(),
          { deadline: this.getDeadline() },
          (error: grpc.ServiceError | null, response: any) => {
            if (error) {
              if (error.code === grpc.status.NOT_FOUND) resolve(null);
              else reject(this.transformError(error));
            } else {
              resolve(response.user);
            }
          }
        );
      });
    } catch (error) {
      throw error;
    }
  }

  async updateUser(id: string, updates: Partial<Omit<User, 'id'>>): Promise<User> {
    const client = this.pool.getClient() as any;
    return new Promise((resolve, reject) => {
      client.updateUser(
        { id, ...updates },
        new grpc.Metadata(),
        { deadline: this.getDeadline() },
        (error: grpc.ServiceError | null, response: any) => {
          if (error) reject(this.transformError(error));
          else resolve(response.user);
        }
      );
    });
  }

  async deleteUser(id: string): Promise<void> {
    const client = this.pool.getClient() as any;
    return new Promise((resolve, reject) => {
      client.deleteUser(
        { id },
        new grpc.Metadata(),
        { deadline: this.getDeadline() },
        (error: grpc.ServiceError | null) => {
          if (error) reject(this.transformError(error));
          else resolve();
        }
      );
    });
  }

  async listUsers(options: ListUsersOptions = {}): Promise<{
    users: User[];
    nextPageToken: string;
    totalCount: number;
  }> {
    const client = this.pool.getClient() as any;
    return new Promise((resolve, reject) => {
      client.listUsers(
        {
          page_size: options.pageSize || 10,
          page_token: options.pageToken || '',
          filter: options.filter || '',
          order_by: options.orderBy || '',
        },
        new grpc.Metadata(),
        { deadline: this.getDeadline() },
        (error: grpc.ServiceError | null, response: any) => {
          if (error) reject(this.transformError(error));
          else resolve({
            users: response.users,
            nextPageToken: response.next_page_token,
            totalCount: response.total_count,
          });
        }
      );
    });
  }

  // Server Streaming: Watch User changes
  watchUsers(
    userIds: string[],
    onEvent: (event: { eventType: string; user: User }) => void,
    onError?: (error: Error) => void,
    onEnd?: () => void
  ): () => void {
    const client = this.pool.getClient() as any;
    const metadata = new grpc.Metadata();

    const stream = client.watchUsers({
      user_ids: userIds,
      event_types: ['USER_CREATED', 'USER_UPDATED', 'USER_DELETED'],
    }, metadata);

    stream.on('data', (event: any) => {
      onEvent({
        eventType: event.event_type,
        user: event.user,
      });
    });

    stream.on('error', (error: Error) => {
      onError?.(error);
    });

    stream.on('end', () => {
      onEnd?.();
    });

    // Return cancel function
    return () => stream.cancel();
  }

  close(): void {
    this.pool.close();
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const client = new UserServiceClient('localhost:50051', 5);

  try {
    // สร้าง user
    const user = await client.createUser({
      email: 'john@example.com',
      name: 'John Doe',
      password: 'secret123',
      role: 'USER_ROLE_CUSTOMER',
    });
    console.log('Created user:', user);

    // Get user
    const found = await client.getUser(user.id);
    console.log('Found user:', found);

    // Watch user changes
    const cancelWatch = client.watchUsers(
      [user.id],
      (event) => console.log('User event:', event),
      (error) => console.error('Watch error:', error)
    );

    // Update user
    await client.updateUser(user.id, { name: 'John Updated' });

    // รอ 1 วินาทีแล้ว cancel
    await new Promise((resolve) => setTimeout(resolve, 1000));
    cancelWatch();

    // List users
    const { users, totalCount } = await client.listUsers({ pageSize: 5 });
    console.log(`Listed ${users.length} of ${totalCount} users`);
  } finally {
    client.close();
  }
}

main().catch(console.error);
```

---

## 5. Order Service: Streaming Implementations

### 5.1 Server Streaming - Order Status

```typescript
// grpc/order-server-streaming.ts
import * as grpc from '@grpc/grpc-js';

interface OrderStatusUpdate {
  order_id: string;
  status: string;
  message: string;
  timestamp: { seconds: number; nanos: number };
}

// Server: ส่ง order status updates แบบ stream
class OrderServiceStreamingServer {
  // Server Streaming RPC
  streamOrderStatus(call: grpc.ServerWritableStream<any, OrderStatusUpdate>): void {
    const orderId = call.request.order_id;
    console.log(`Starting order status stream for: ${orderId}`);

    // จำลองการส่ง status updates
    const statuses = [
      { status: 'ORDER_STATUS_CONFIRMED', message: 'Order confirmed' },
      { status: 'ORDER_STATUS_PROCESSING', message: 'Payment processing' },
      { status: 'ORDER_STATUS_SHIPPED', message: 'Package shipped' },
      { status: 'ORDER_STATUS_DELIVERED', message: 'Package delivered' },
    ];

    let index = 0;

    const sendUpdate = () => {
      if (call.cancelled || index >= statuses.length) {
        if (!call.cancelled) {
          call.end();
          console.log(`Order ${orderId} stream completed`);
        }
        return;
      }

      const update: OrderStatusUpdate = {
        order_id: orderId,
        ...statuses[index],
        timestamp: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
      };

      const canContinue = call.write(update);
      index++;

      if (canContinue) {
        setTimeout(sendUpdate, 2000); // ส่งทุก 2 วินาที
      } else {
        // รอ drain event ก่อนส่งต่อ
        call.once('drain', () => setTimeout(sendUpdate, 2000));
      }
    };

    call.on('cancelled', () => {
      console.log(`Order ${orderId} stream cancelled by client`);
    });

    // เริ่มส่ง updates
    sendUpdate();
  }
}
```

### 5.2 Client Streaming - Batch Upload

```typescript
// grpc/order-client-streaming.ts
import * as grpc from '@grpc/grpc-js';

interface CreateOrderRequest {
  customer_id: string;
  items: Array<{ product_id: string; quantity: number; unit_price: number }>;
}

interface BatchCreateOrderResponse {
  orders: any[];
  success_count: number;
  failure_count: number;
}

// Server: รับ batch orders จาก client stream
class OrderBatchServer {
  // Client Streaming RPC
  batchCreateOrder(
    call: grpc.ServerReadableStream<CreateOrderRequest, BatchCreateOrderResponse>,
    callback: grpc.sendUnaryData<BatchCreateOrderResponse>
  ): void {
    const orders: any[] = [];
    let successCount = 0;
    let failureCount = 0;

    call.on('data', async (request: CreateOrderRequest) => {
      try {
        // ประมวลผลแต่ละ order
        const order = {
          id: `order-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
          customer_id: request.customer_id,
          items: request.items,
          total_amount: request.items.reduce(
            (sum, item) => sum + item.quantity * item.unit_price, 0
          ),
          status: 'ORDER_STATUS_CONFIRMED',
          created_at: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
        };

        orders.push(order);
        successCount++;
      } catch (error) {
        failureCount++;
        console.error('Failed to process order:', error);
      }
    });

    call.on('end', () => {
      console.log(`Batch processing complete: ${successCount} success, ${failureCount} failed`);
      callback(null, {
        orders,
        success_count: successCount,
        failure_count: failureCount,
      });
    });

    call.on('error', (error) => {
      console.error('Client stream error:', error);
      callback({
        code: grpc.status.INTERNAL,
        message: error.message,
      });
    });
  }
}

// Client: ส่ง batch orders
class OrderBatchClient {
  private client: any;

  constructor(address: string) {
    // client initialization (simplified)
  }

  async batchCreateOrders(orders: CreateOrderRequest[]): Promise<BatchCreateOrderResponse> {
    return new Promise((resolve, reject) => {
      const call = this.client.batchCreateOrder(
        (error: grpc.ServiceError | null, response: BatchCreateOrderResponse) => {
          if (error) reject(error);
          else resolve(response);
        }
      );

      // ส่ง orders ทีละตัว
      for (const order of orders) {
        call.write(order);
      }

      // บอก server ว่าส่งเสร็จแล้ว
      call.end();
    });
  }
}
```

### 5.3 Bidirectional Streaming - Chat Service

```typescript
// grpc/order-bidi-streaming.ts
import * as grpc from '@grpc/grpc-js';

interface ChatMessage {
  sender: string;
  content: string;
  order_id: string;
  timestamp: { seconds: number; nanos: number };
}

// Server: Bidirectional chat
class OrderChatServer {
  private chatRooms: Map<string, grpc.ServerDuplexStream<ChatMessage, ChatMessage>[]> = new Map();

  orderChat(call: grpc.ServerDuplexStream<ChatMessage, ChatMessage>): void {
    let orderId: string | null = null;

    call.on('data', (message: ChatMessage) => {
      if (!orderId) {
        orderId = message.order_id;
        // เข้าร่วม chat room
        if (!this.chatRooms.has(orderId)) {
          this.chatRooms.set(orderId, []);
        }
        this.chatRooms.get(orderId)!.push(call);
        console.log(`${message.sender} joined chat for order ${orderId}`);
      }

      console.log(`[${message.order_id}] ${message.sender}: ${message.content}`);

      // Broadcast ไปยัง participants ทั้งหมดใน room
      const room = this.chatRooms.get(message.order_id) || [];
      const broadcastMessage: ChatMessage = {
        ...message,
        timestamp: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
      };

      for (const stream of room) {
        if (stream !== call && !stream.cancelled) {
          try {
            stream.write(broadcastMessage);
          } catch (error) {
            console.error('Error broadcasting message:', error);
          }
        }
      }
    });

    call.on('end', () => {
      if (orderId) {
        const room = this.chatRooms.get(orderId) || [];
        const index = room.indexOf(call);
        if (index !== -1) room.splice(index, 1);
        if (room.length === 0) this.chatRooms.delete(orderId);
      }
      call.end();
      console.log('Chat stream ended');
    });

    call.on('cancelled', () => {
      if (orderId) {
        const room = this.chatRooms.get(orderId) || [];
        const index = room.indexOf(call);
        if (index !== -1) room.splice(index, 1);
      }
    });
  }
}

// Client: Bidirectional chat client
class OrderChatClient {
  private client: any;

  async joinChat(
    orderId: string,
    senderName: string,
    onMessage: (message: ChatMessage) => void
  ): Promise<{
    sendMessage: (content: string) => void;
    leave: () => void;
  }> {
    const metadata = new grpc.Metadata();
    const call = this.client.orderChat(metadata);

    call.on('data', (message: ChatMessage) => {
      onMessage(message);
    });

    call.on('error', (error: Error) => {
      console.error('Chat error:', error);
    });

    call.on('end', () => {
      console.log('Chat ended');
    });

    // ส่งข้อความแรกเพื่อระบุ order และ sender
    call.write({
      sender: senderName,
      content: `${senderName} joined the chat`,
      order_id: orderId,
      timestamp: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
    });

    return {
      sendMessage: (content: string) => {
        call.write({
          sender: senderName,
          content,
          order_id: orderId,
          timestamp: { seconds: Math.floor(Date.now() / 1000), nanos: 0 },
        });
      },
      leave: () => {
        call.end();
      },
    };
  }
}
```

---

## 6. gRPC Interceptors

### 6.1 Auth, Logging, Tracing, Deadline Interceptors

```typescript
// grpc/interceptors.ts
import * as grpc from '@grpc/grpc-js';

// =========================================================
// 1. Auth Interceptor
// =========================================================

function createAuthInterceptor(
  validateToken: (token: string) => Promise<boolean>
): grpc.ServerInterceptor {
  return async (call: any, callback: any) => {
    const metadata = call.metadata;
    const authHeader = metadata.get('authorization')[0] as string;

    if (!authHeader) {
      return callback({
        code: grpc.status.UNAUTHENTICATED,
        message: 'Missing authorization header',
      });
    }

    const token = authHeader.replace('Bearer ', '');

    try {
      const isValid = await validateToken(token);
      if (!isValid) {
        return callback({
          code: grpc.status.UNAUTHENTICATED,
          message: 'Invalid token',
        });
      }

      // เพิ่ม user info ไปยัง metadata
      call.metadata.set('x-authenticated', 'true');
      call.call.handler.call(call, callback);
    } catch (error) {
      callback({
        code: grpc.status.INTERNAL,
        message: 'Auth validation failed',
      });
    }
  };
}

// =========================================================
// 2. Logging Interceptor
// =========================================================

type InterceptorNext = (...args: any[]) => void;

function loggingInterceptor(
  methodDefinition: grpc.MethodDefinition<any, any>,
  call: any
): grpc.ServerInterceptingCall {
  const startTime = Date.now();
  const callPath = methodDefinition.path;

  return new grpc.ServerInterceptingCall(call, {
    start: (metadata, listener, next) => {
      console.log(`[gRPC] START ${callPath}`, {
        metadata: Object.fromEntries(
          Object.entries(metadata.getMap()).map(([k, v]) => [k, String(v)])
        ),
      });

      const newListener = {
        onReceiveMessage: (message: any, next: InterceptorNext) => {
          console.log(`[gRPC] RECEIVED ${callPath}:`, message);
          next(message);
        },
        onReceiveHalfClose: (next: InterceptorNext) => {
          next();
        },
        onCancel: () => {
          console.log(`[gRPC] CANCELLED ${callPath}`);
        },
      };

      next(metadata, newListener);
    },
    sendMessage: (message, next) => {
      console.log(`[gRPC] SENDING ${callPath}:`, message);
      next(message);
    },
    sendStatus: (status, next) => {
      const duration = Date.now() - startTime;
      console.log(`[gRPC] COMPLETE ${callPath}:`, {
        code: status.code,
        details: status.details,
        durationMs: duration,
      });
      next(status);
    },
  });
}

// =========================================================
// 3. Tracing Interceptor (OpenTelemetry)
// =========================================================

import { trace, context, SpanStatusCode, propagation } from '@opentelemetry/api';

function tracingInterceptor(
  methodDefinition: grpc.MethodDefinition<any, any>,
  call: any
): grpc.ServerInterceptingCall {
  const tracer = trace.getTracer('grpc-server');

  return new grpc.ServerInterceptingCall(call, {
    start: (metadata, listener, next) => {
      // Extract trace context จาก metadata
      const traceContext: Record<string, string> = {};
      for (const [key, values] of Object.entries(metadata.getMap())) {
        traceContext[key] = Array.isArray(values) ? values[0] : String(values);
      }

      const parentContext = propagation.extract(context.active(), traceContext);
      const span = tracer.startSpan(
        methodDefinition.path,
        { attributes: { 'rpc.system': 'grpc', 'rpc.method': methodDefinition.path } },
        parentContext
      );

      const spanContext = context.with(trace.setSpan(context.active(), span), () => context.active());

      const newListener = {
        onReceiveMessage: (message: any, next: InterceptorNext) => {
          span.addEvent('message_received');
          next(message);
        },
        onReceiveHalfClose: (next: InterceptorNext) => {
          next();
        },
        onCancel: () => {
          span.setStatus({ code: SpanStatusCode.ERROR, message: 'Cancelled' });
          span.end();
        },
      };

      next(metadata, newListener);
    },
    sendStatus: (status, next) => {
      const currentSpan = trace.getSpan(context.active());
      if (currentSpan) {
        if (status.code !== grpc.status.OK) {
          currentSpan.setStatus({
            code: SpanStatusCode.ERROR,
            message: status.details,
          });
        }
        currentSpan.setAttribute('rpc.grpc.status_code', status.code);
        currentSpan.end();
      }
      next(status);
    },
  });
}

// =========================================================
// 4. Deadline Interceptor
// =========================================================

function deadlineInterceptor(
  methodDefinition: grpc.MethodDefinition<any, any>,
  call: any
): grpc.ServerInterceptingCall {
  const defaultDeadlineMs = 30000; // 30 seconds

  return new grpc.ServerInterceptingCall(call, {
    start: (metadata, listener, next) => {
      // ตรวจสอบ deadline
      const deadline = call.deadline;

      if (deadline) {
        const remainingMs = new Date(deadline).getTime() - Date.now();
        if (remainingMs <= 0) {
          call.sendStatus({
            code: grpc.status.DEADLINE_EXCEEDED,
            details: 'Deadline exceeded before processing started',
          });
          return;
        }

        if (remainingMs < 100) {
          console.warn(`Very short deadline: ${remainingMs}ms for ${methodDefinition.path}`);
        }
      }

      next(metadata, listener);
    },
  });
}

// =========================================================
// 5. ประกอบ Interceptors เข้าด้วยกัน
// =========================================================

function createServerWithInterceptors(): grpc.Server {
  const server = new grpc.Server({
    interceptors: [
      deadlineInterceptor,
      tracingInterceptor,
      loggingInterceptor,
    ],
  });

  return server;
}
```

---

## 7. gRPC-Web: Envoy Proxy Config & Browser Client

### 7.1 Envoy Proxy Configuration

```yaml
# envoy/envoy.yaml
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address:
      protocol: TCP
      address: 0.0.0.0
      port_value: 9901

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
                codec_type: auto
                stat_prefix: ingress_http
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          route:
                            cluster: grpc_service
                            timeout: 0s
                            max_stream_duration:
                              grpc_timeout_header_max: 0s
                      cors:
                        allow_origin_string_match:
                          - prefix: "*"
                        allow_methods: GET, PUT, DELETE, POST, OPTIONS
                        allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,custom-header-1,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout,authorization
                        max_age: "1728000"
                        expose_headers: custom-header-1,grpc-status,grpc-message
                http_filters:
                  - name: envoy.filters.http.grpc_web
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.grpc_web.v3.GrpcWeb
                  - name: envoy.filters.http.cors
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.cors.v3.CorsPolicy
                  - name: envoy.filters.http.router
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
    - name: grpc_service
      connect_timeout: 0.25s
      type: logical_dns
      http2_protocol_options: {}
      lb_policy: round_robin
      load_assignment:
        cluster_name: grpc_service
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address:
                      address: grpc-server
                      port_value: 50051
```

### 7.2 Browser TypeScript Client (gRPC-Web)

```typescript
// browser/grpc-web-client.ts
// ใช้ @improbable-eng/grpc-web หรือ grpc-web package

import { grpc } from '@improbable-eng/grpc-web';
import { UserServiceClient } from '../generated/user_pb_service';
import {
  CreateUserRequest,
  GetUserRequest,
  ListUsersRequest,
  WatchUsersRequest,
} from '../generated/user_pb';

class BrowserUserServiceClient {
  private client: UserServiceClient;

  constructor(host: string = 'http://localhost:8080') {
    this.client = new UserServiceClient(host, {
      debug: process.env.NODE_ENV === 'development',
    });
  }

  async createUser(input: {
    email: string;
    name: string;
    password: string;
    role?: string;
  }): Promise<{
    id: string;
    email: string;
    name: string;
  }> {
    const request = new CreateUserRequest();
    request.setEmail(input.email);
    request.setName(input.name);
    request.setPassword(input.password);
    if (input.role) request.setRole(input.role as any);

    return new Promise((resolve, reject) => {
      this.client.createUser(request, (error, response) => {
        if (error) reject(new Error(error.message));
        else {
          const user = response!.getUser()!;
          resolve({
            id: user.getId(),
            email: user.getEmail(),
            name: user.getName(),
          });
        }
      });
    });
  }

  async getUser(id: string): Promise<any> {
    const request = new GetUserRequest();
    request.setId(id);

    return new Promise((resolve, reject) => {
      this.client.getUser(request, (error, response) => {
        if (error) reject(new Error(error.message));
        else resolve(response!.getUser()!.toObject());
      });
    });
  }

  // Server streaming ผ่าน gRPC-Web
  watchUsers(
    userIds: string[],
    onEvent: (event: any) => void,
    onError?: (error: Error) => void,
    onComplete?: () => void
  ): () => void {
    const request = new WatchUsersRequest();
    request.setUserIdsList(userIds);

    const stream = this.client.watchUsers(request);

    stream.on('data', (event) => {
      onEvent({
        type: event.getEventType(),
        user: event.getUser()?.toObject(),
        timestamp: event.getEventTime()?.toObject(),
      });
    });

    stream.on('status', (status) => {
      if (status.code !== 0) {
        onError?.(new Error(status.details));
      }
    });

    stream.on('end', () => {
      onComplete?.();
    });

    // Return cancel function
    return () => stream.cancel();
  }
}

// React Hook สำหรับใช้งาน gRPC-Web
function useUserService() {
  const client = new BrowserUserServiceClient('http://localhost:8080');

  const createUser = async (input: Parameters<typeof client.createUser>[0]) => {
    try {
      return await client.createUser(input);
    } catch (error) {
      console.error('Failed to create user:', error);
      throw error;
    }
  };

  const watchUser = (userId: string, onUpdate: (user: any) => void) => {
    const cancel = client.watchUsers([userId], onUpdate);
    return cancel; // Cleanup function สำหรับ useEffect
  };

  return { createUser, watchUser };
}
```

---

## 8. gRPC Health Checking

```typescript
// grpc/health-check.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import path from 'path';

// Load health check proto
const HEALTH_PROTO_PATH = path.join(
  __dirname,
  '../node_modules/@grpc/grpc-js/build/src/generated/grpc/health/v1/health.proto'
);

interface HealthCheckRequest {
  service: string;
}

interface HealthCheckResponse {
  status: string;
}

type ServingStatus = 'SERVING' | 'NOT_SERVING' | 'SERVICE_UNKNOWN' | 'UNKNOWN';

class HealthCheckServer {
  private serviceStatuses: Map<string, ServingStatus> = new Map();
  private watchStreams: Map<string, grpc.ServerWritableStream<any, any>[]> = new Map();

  constructor() {
    // Default: server itself is SERVING
    this.serviceStatuses.set('', 'SERVING');
  }

  setServingStatus(service: string, status: ServingStatus): void {
    const previousStatus = this.serviceStatuses.get(service);
    this.serviceStatuses.set(service, status);

    // Notify watchers ถ้า status เปลี่ยน
    if (previousStatus !== status) {
      this.notifyWatchers(service, status);
    }

    console.log(`Health status updated: ${service || 'server'} = ${status}`);
  }

  check(
    call: grpc.ServerUnaryCall<HealthCheckRequest, HealthCheckResponse>,
    callback: grpc.sendUnaryData<HealthCheckResponse>
  ): void {
    const { service } = call.request;
    const status = this.serviceStatuses.get(service) || 'SERVICE_UNKNOWN';

    callback(null, { status });
  }

  watch(call: grpc.ServerWritableStream<HealthCheckRequest, HealthCheckResponse>): void {
    const { service } = call.request;

    // ส่ง current status ทันที
    const currentStatus = this.serviceStatuses.get(service) || 'SERVICE_UNKNOWN';
    call.write({ status: currentStatus });

    // ลงทะเบียน watcher
    if (!this.watchStreams.has(service)) {
      this.watchStreams.set(service, []);
    }
    this.watchStreams.get(service)!.push(call);

    call.on('cancelled', () => {
      const streams = this.watchStreams.get(service) || [];
      const index = streams.indexOf(call);
      if (index !== -1) streams.splice(index, 1);
    });
  }

  private notifyWatchers(service: string, status: ServingStatus): void {
    const streams = this.watchStreams.get(service) || [];
    for (const stream of streams) {
      if (!stream.cancelled) {
        try {
          stream.write({ status });
        } catch (error) {
          console.error('Error notifying health watcher:', error);
        }
      }
    }
  }

  getServiceDefinition() {
    return {
      check: this.check.bind(this),
      watch: this.watch.bind(this),
    };
  }
}

// ใช้งาน Health Check
function addHealthCheckToServer(server: grpc.Server): HealthCheckServer {
  const healthChecker = new HealthCheckServer();

  // Load health proto
  const packageDef = protoLoader.loadSync(
    require.resolve('@grpc/grpc-js/build/src/generated/grpc/health/v1/health.proto')
  );
  const healthProto = grpc.loadPackageDefinition(packageDef) as any;

  server.addService(
    healthProto.grpc.health.v1.Health.service,
    healthChecker.getServiceDefinition()
  );

  return healthChecker;
}
```

---

## 9. gRPC Reflection Server

```typescript
// grpc/reflection-server.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import { ReflectionService } from '@grpc/reflection';
import path from 'path';

function addReflectionToServer(server: grpc.Server, protoFiles: string[]): void {
  const packageDefinitions = protoFiles.map((protoFile) =>
    protoLoader.loadSync(protoFile, {
      keepCase: true,
      longs: String,
      enums: String,
      defaults: true,
      oneofs: true,
    })
  );

  const reflection = new ReflectionService(packageDefinitions[0]);
  reflection.addToServer(server);

  console.log('gRPC reflection server added');
  console.log('Test with: grpcurl -plaintext localhost:50051 list');
}

// Docker Compose สำหรับ grpcurl testing
const dockerCompose = `
# docker-compose.grpcurl.yml
version: '3.8'

services:
  grpcurl:
    image: fullstorydev/grpcurl:latest
    network_mode: host
    command: >
      -plaintext
      localhost:50051
      list

  grpcui:
    image: fullstorydev/grpcui:latest
    network_mode: host
    command: >
      -plaintext
      localhost:50051
    ports:
      - "8080:8080"
`;

export { addReflectionToServer, dockerCompose };
```

---

## 10. Load Balancing

### 10.1 Round-robin และ Pick First Policies

```typescript
// grpc/load-balancing.ts
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';

// =========================================================
// 1. Round Robin Load Balancing
// =========================================================

function createRoundRobinClient(
  ServiceClass: any,
  endpoints: string[],
  credentials: grpc.ChannelCredentials
): any {
  // ใช้ dns:// สำหรับ round-robin
  const target = `dns:///service.example.com:50051`;

  return new ServiceClass(target, credentials, {
    'grpc.service_config': JSON.stringify({
      loadBalancingConfig: [{ round_robin: {} }],
      methodConfig: [
        {
          name: [{ service: 'user.v1.UserService' }],
          retryPolicy: {
            maxAttempts: 3,
            initialBackoff: '0.1s',
            maxBackoff: '1s',
            backoffMultiplier: 2,
            retryableStatusCodes: ['UNAVAILABLE', 'RESOURCE_EXHAUSTED'],
          },
          timeout: '5s',
          waitForReady: true,
        },
      ],
    }),
  });
}

// =========================================================
// 2. Pick First (default) - เชื่อมต่อกับ server แรก
// =========================================================

function createPickFirstClient(
  ServiceClass: any,
  address: string,
  credentials: grpc.ChannelCredentials
): any {
  return new ServiceClass(address, credentials, {
    'grpc.service_config': JSON.stringify({
      loadBalancingConfig: [{ pick_first: {} }],
    }),
  });
}

// =========================================================
// 3. Custom Client-side Load Balancer
// =========================================================

class ClientSideLoadBalancer<T extends grpc.Client> {
  private clients: T[];
  private currentIndex = 0;
  private healthyClients: Set<number>;

  constructor(
    ServiceClass: new (address: string, credentials: grpc.ChannelCredentials, options?: object) => T,
    endpoints: string[],
    credentials: grpc.ChannelCredentials,
    options?: object
  ) {
    this.clients = endpoints.map((endpoint) => new ServiceClass(endpoint, credentials, options));
    this.healthyClients = new Set(endpoints.map((_, i) => i));

    // Health check loop
    this.startHealthChecks();
  }

  private startHealthChecks(): void {
    setInterval(() => {
      this.clients.forEach((client, index) => {
        const deadline = new Date(Date.now() + 1000);
        // ตรวจสอบ channel state
        const state = client.getChannel().getConnectivityState(false);

        if (state === grpc.connectivityState.READY) {
          this.healthyClients.add(index);
        } else if (
          state === grpc.connectivityState.TRANSIENT_FAILURE ||
          state === grpc.connectivityState.SHUTDOWN
        ) {
          this.healthyClients.delete(index);
          console.warn(`Client ${index} is unhealthy, state: ${state}`);
        }
      });
    }, 5000);
  }

  getClient(): T {
    const healthyIndices = Array.from(this.healthyClients);

    if (healthyIndices.length === 0) {
      throw new Error('No healthy gRPC servers available');
    }

    // Round-robin ใน healthy clients
    const healthyIndex = this.currentIndex % healthyIndices.length;
    const clientIndex = healthyIndices[healthyIndex];
    this.currentIndex = (this.currentIndex + 1) % healthyIndices.length;

    return this.clients[clientIndex];
  }

  closeAll(): void {
    this.clients.forEach((client) => client.close());
  }
}

// ตัวอย่างการใช้งาน
async function main() {
  const PROTO_PATH = path.join(__dirname, '../proto/user.proto');
  const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
    keepCase: true, longs: String, enums: String, defaults: true, oneofs: true,
  });
  const proto = grpc.loadPackageDefinition(packageDefinition) as any;

  // สร้าง load balancer
  const lb = new ClientSideLoadBalancer(
    proto.user.v1.UserService,
    ['localhost:50051', 'localhost:50052', 'localhost:50053'],
    grpc.credentials.createInsecure()
  );

  // ใช้งาน client
  const client = lb.getClient();

  client.getUser(
    { id: 'user-001' },
    new grpc.Metadata(),
    { deadline: new Date(Date.now() + 5000) },
    (error: any, response: any) => {
      if (error) console.error('Error:', error);
      else console.log('User:', response?.user);
    }
  );
}

main().catch(console.error);
```

---

## สรุป

| หัวข้อ | เนื้อหาสำคัญ |
|--------|-------------|
| gRPC vs REST vs GraphQL | gRPC เหมาะกับ internal services ที่ต้องการ performance และ type safety |
| Protocol Buffers | Schema-first approach ด้วย .proto files สำหรับ user และ order services |
| gRPC Server | TypeScript server implementation พร้อม CRUD operations และ server streaming |
| gRPC Client | Connection pooling, typed client wrapper, และ graceful error handling |
| Unary RPC | Request/Response pattern พื้นฐาน |
| Server Streaming | ส่ง order status updates แบบ real-time ไปยัง client |
| Client Streaming | Batch upload orders จาก client ไปยัง server |
| Bidirectional Streaming | Chat service สำหรับ order support |
| Interceptors | Auth, Logging, Tracing, Deadline interceptors |
| gRPC-Web | Envoy proxy configuration สำหรับ browser clients |
| Health Checking | grpc.health.v1.Health implementation สำหรับ Kubernetes |
| Reflection | Server reflection สำหรับ debugging ด้วย grpcurl |
| Load Balancing | Round-robin, pick_first, และ custom load balancer |

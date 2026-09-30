# Part 83: API Design Best Practices

## บทนำ

การออกแบบ API ที่ดีเป็นรากฐานสำคัญของระบบ Microservices ที่ประสบความสำเร็จ ในบทนี้เราจะเรียนรู้หลักการและแนวปฏิบัติที่ดีที่สุดสำหรับการออกแบบ RESTful API รวมถึงการใช้ OpenAPI 3.0 สำหรับเขียน specification การจัดการ versioning และ patterns ต่างๆ ที่ใช้ในการสร้าง API ระดับ production

---

## 1. RESTful API Design Principles

### 1.1 หลักการพื้นฐานของ REST

REST (Representational State Transfer) มีหลักการสำคัญ 6 ประการ:

1. **Client-Server**: แยก client กับ server ออกจากกัน
2. **Stateless**: ทุก request ต้องมีข้อมูลครบถ้วนในตัวเอง
3. **Cacheable**: response ต้องระบุว่า cacheable หรือไม่
4. **Uniform Interface**: interface ที่สม่ำเสมอ
5. **Layered System**: ระบบสามารถมี layer ได้
6. **Code on Demand** (optional): server สามารถส่ง code ให้ client ได้

### 1.2 Resource Naming Conventions

```typescript
// ✅ Good: ใช้ nouns (คำนาม) ไม่ใช่ verbs (คำกริยา)
GET    /api/v1/users           // ดึงรายการ users
GET    /api/v1/users/:id       // ดึง user คนเดียว
POST   /api/v1/users           // สร้าง user ใหม่
PUT    /api/v1/users/:id       // อัปเดต user ทั้งหมด
PATCH  /api/v1/users/:id       // อัปเดต user บางส่วน
DELETE /api/v1/users/:id       // ลบ user

// ✅ Good: Nested resources
GET    /api/v1/users/:userId/orders
POST   /api/v1/users/:userId/orders
GET    /api/v1/users/:userId/orders/:orderId

// ❌ Bad: ใช้ verbs ใน URL
GET    /api/v1/getUsers
POST   /api/v1/createUser
PUT    /api/v1/updateUser/:id
DELETE /api/v1/deleteUser/:id

// ❌ Bad: ใช้ singular nouns
GET    /api/v1/user
POST   /api/v1/order
```

### 1.3 HTTP Methods และ Status Codes

```typescript
// HTTP Status Codes ที่สำคัญ
const HTTP_STATUS = {
  // 2xx Success
  OK: 200,                    // GET, PUT, PATCH, DELETE สำเร็จ
  CREATED: 201,               // POST สร้างสำเร็จ
  ACCEPTED: 202,              // Async operation accepted
  NO_CONTENT: 204,            // DELETE สำเร็จ ไม่มี body

  // 3xx Redirection
  MOVED_PERMANENTLY: 301,     // URL เปลี่ยนถาวร
  NOT_MODIFIED: 304,          // Cache ยังใช้ได้

  // 4xx Client Errors
  BAD_REQUEST: 400,           // Request ไม่ถูกต้อง
  UNAUTHORIZED: 401,          // ยังไม่ได้ authenticate
  FORBIDDEN: 403,             // ไม่มีสิทธิ์
  NOT_FOUND: 404,             // ไม่พบ resource
  METHOD_NOT_ALLOWED: 405,    // HTTP method ไม่รองรับ
  CONFLICT: 409,              // ข้อมูลขัดแย้ง
  GONE: 410,                  // Resource ถูกลบถาวร
  UNPROCESSABLE_ENTITY: 422,  // Validation failed
  TOO_MANY_REQUESTS: 429,     // Rate limit exceeded

  // 5xx Server Errors
  INTERNAL_SERVER_ERROR: 500,
  BAD_GATEWAY: 502,
  SERVICE_UNAVAILABLE: 503,
  GATEWAY_TIMEOUT: 504,
} as const;

// Express.js handler example
import express, { Request, Response, NextFunction } from 'express';

const router = express.Router();

// GET /users/:id
router.get('/users/:id', async (req: Request, res: Response) => {
  const { id } = req.params;
  
  const user = await userService.findById(id);
  
  if (!user) {
    return res.status(HTTP_STATUS.NOT_FOUND).json({
      error: {
        code: 'USER_NOT_FOUND',
        message: `User with id ${id} not found`,
        timestamp: new Date().toISOString(),
      }
    });
  }
  
  return res.status(HTTP_STATUS.OK).json({
    data: user,
    meta: {
      timestamp: new Date().toISOString(),
    }
  });
});

// POST /users
router.post('/users', async (req: Request, res: Response) => {
  const { name, email } = req.body;
  
  const existingUser = await userService.findByEmail(email);
  if (existingUser) {
    return res.status(HTTP_STATUS.CONFLICT).json({
      error: {
        code: 'EMAIL_ALREADY_EXISTS',
        message: 'A user with this email already exists',
        field: 'email',
      }
    });
  }
  
  const user = await userService.create({ name, email });
  
  return res.status(HTTP_STATUS.CREATED).json({
    data: user,
    meta: {
      timestamp: new Date().toISOString(),
    }
  });
});
```

---

## 2. OpenAPI 3.0 Specification

### 2.1 โครงสร้างพื้นฐานของ OpenAPI

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: User Service API
  description: |
    API สำหรับจัดการข้อมูล Users ในระบบ Microservices
    
    ## Authentication
    ใช้ Bearer token ใน Authorization header
    
    ## Rate Limiting
    - 1000 requests ต่อ minute สำหรับ authenticated users
    - 100 requests ต่อ minute สำหรับ anonymous
  version: 1.0.0
  contact:
    name: API Support
    email: api-support@example.com
    url: https://developers.example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging
  - url: http://localhost:3000/v1
    description: Local Development

tags:
  - name: Users
    description: User management operations
  - name: Orders
    description: Order management operations

paths:
  /users:
    get:
      tags: [Users]
      summary: List users
      description: ดึงรายการ users พร้อม pagination
      operationId: listUsers
      security:
        - BearerAuth: []
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
        - name: sort
          in: query
          schema:
            type: string
            enum: [name, email, createdAt]
            default: createdAt
        - name: order
          in: query
          schema:
            type: string
            enum: [asc, desc]
            default: desc
        - name: search
          in: query
          description: ค้นหาด้วย name หรือ email
          schema:
            type: string
            maxLength: 100
      responses:
        '200':
          description: รายการ users
          headers:
            X-Total-Count:
              description: จำนวน users ทั้งหมด
              schema:
                type: integer
            X-Rate-Limit-Remaining:
              description: จำนวน requests ที่เหลือ
              schema:
                type: integer
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'
              examples:
                success:
                  $ref: '#/components/examples/UserListExample'
        '401':
          $ref: '#/components/responses/UnauthorizedError'
        '429':
          $ref: '#/components/responses/TooManyRequestsError'

    post:
      tags: [Users]
      summary: Create user
      operationId: createUser
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
            examples:
              basic:
                summary: Basic user creation
                value:
                  name: John Doe
                  email: john@example.com
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '400':
          $ref: '#/components/responses/BadRequestError'
        '409':
          $ref: '#/components/responses/ConflictError'

  /users/{userId}:
    parameters:
      - $ref: '#/components/parameters/UserIdParam'
    
    get:
      tags: [Users]
      summary: Get user by ID
      operationId: getUserById
      security:
        - BearerAuth: []
      responses:
        '200':
          description: User data
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'
        '404':
          $ref: '#/components/responses/NotFoundError'

    patch:
      tags: [Users]
      summary: Update user
      operationId: updateUser
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateUserRequest'
      responses:
        '200':
          description: User updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserResponse'

    delete:
      tags: [Users]
      summary: Delete user
      operationId: deleteUser
      security:
        - BearerAuth: []
      responses:
        '204':
          description: User deleted
        '404':
          $ref: '#/components/responses/NotFoundError'

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
          example: 550e8400-e29b-41d4-a716-446655440000
        name:
          type: string
          minLength: 1
          maxLength: 100
          example: John Doe
        email:
          type: string
          format: email
          example: john@example.com
        role:
          type: string
          enum: [admin, user, moderator]
          default: user
        isActive:
          type: boolean
          default: true
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time
      required: [id, name, email, role, isActive, createdAt, updatedAt]

    CreateUserRequest:
      type: object
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        email:
          type: string
          format: email
        role:
          type: string
          enum: [admin, user, moderator]
          default: user
      required: [name, email]

    UpdateUserRequest:
      type: object
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        role:
          type: string
          enum: [admin, user, moderator]
      minProperties: 1

    UserResponse:
      type: object
      properties:
        data:
          $ref: '#/components/schemas/User'
        meta:
          $ref: '#/components/schemas/ResponseMeta'

    UserListResponse:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/User'
        meta:
          $ref: '#/components/schemas/PaginationMeta'

    ResponseMeta:
      type: object
      properties:
        timestamp:
          type: string
          format: date-time
        requestId:
          type: string
          format: uuid

    PaginationMeta:
      allOf:
        - $ref: '#/components/schemas/ResponseMeta'
        - type: object
          properties:
            pagination:
              type: object
              properties:
                page:
                  type: integer
                  minimum: 1
                limit:
                  type: integer
                  minimum: 1
                  maximum: 100
                total:
                  type: integer
                  minimum: 0
                totalPages:
                  type: integer
                  minimum: 0
                hasNext:
                  type: boolean
                hasPrev:
                  type: boolean

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
            message:
              type: string
            details:
              type: array
              items:
                type: object
                properties:
                  field:
                    type: string
                  message:
                    type: string
            timestamp:
              type: string
              format: date-time
            requestId:
              type: string
              format: uuid
          required: [code, message, timestamp]

  parameters:
    UserIdParam:
      name: userId
      in: path
      required: true
      schema:
        type: string
        format: uuid
    PageParam:
      name: page
      in: query
      schema:
        type: integer
        minimum: 1
        default: 1
    LimitParam:
      name: limit
      in: query
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

  responses:
    UnauthorizedError:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            error:
              code: UNAUTHORIZED
              message: Authentication token is missing or invalid
              timestamp: '2024-01-01T00:00:00Z'
    
    NotFoundError:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    BadRequestError:
      description: Invalid request data
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    ConflictError:
      description: Resource conflict
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

    TooManyRequestsError:
      description: Rate limit exceeded
      headers:
        Retry-After:
          description: Seconds until rate limit resets
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'

  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  examples:
    UserListExample:
      summary: Example user list response
      value:
        data:
          - id: 550e8400-e29b-41d4-a716-446655440000
            name: John Doe
            email: john@example.com
            role: user
            isActive: true
            createdAt: '2024-01-01T00:00:00Z'
            updatedAt: '2024-01-01T00:00:00Z'
        meta:
          timestamp: '2024-01-01T00:00:00Z'
          requestId: 660e8400-e29b-41d4-a716-446655440001
          pagination:
            page: 1
            limit: 20
            total: 100
            totalPages: 5
            hasNext: true
            hasPrev: false
```

### 2.2 TypeScript Types จาก OpenAPI

```typescript
// tools/generate-types.ts
import { execSync } from 'child_process';
import fs from 'fs';

// Generate TypeScript types from OpenAPI spec
execSync('npx openapi-typescript openapi.yaml --output src/types/api.ts');

// Generated types (ตัวอย่าง)
export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'user' | 'moderator';
  isActive: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface CreateUserRequest {
  name: string;
  email: string;
  role?: 'admin' | 'user' | 'moderator';
}

export interface UserListResponse {
  data: User[];
  meta: PaginationMeta;
}

export interface PaginationMeta {
  timestamp: string;
  requestId: string;
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}
```

---

## 3. API Versioning Strategies

### 3.1 URL Path Versioning (แนะนำ)

```typescript
// src/app.ts
import express from 'express';
import { routerV1 } from './routes/v1';
import { routerV2 } from './routes/v2';

const app = express();

// Version 1 - stable
app.use('/api/v1', routerV1);

// Version 2 - new features
app.use('/api/v2', routerV2);

// Redirect root to latest
app.use('/api', (req, res) => {
  res.redirect('/api/v2' + req.path);
});
```

### 3.2 Header-Based Versioning

```typescript
// src/middleware/version.ts
import { Request, Response, NextFunction } from 'express';

export const versionMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const version = req.headers['api-version'] || 
                  req.headers['accept']?.match(/version=(\d+)/)?.[1] ||
                  '1';
  
  req.apiVersion = parseInt(version as string);
  next();
};

// Usage
app.use(versionMiddleware);
app.use('/api/users', (req, res) => {
  switch (req.apiVersion) {
    case 2:
      return usersV2Handler(req, res);
    default:
      return usersV1Handler(req, res);
  }
});

// src/routes/users.ts - Versioned handlers
export const usersV1Handler = async (req: Request, res: Response) => {
  const users = await userService.findAll();
  // V1: ส่ง array ตรงๆ
  return res.json(users);
};

export const usersV2Handler = async (req: Request, res: Response) => {
  const users = await userService.findAll();
  // V2: ส่งพร้อม metadata
  return res.json({
    data: users,
    meta: { total: users.length, timestamp: new Date().toISOString() }
  });
};
```

### 3.3 Deprecation Headers

```typescript
// src/middleware/deprecation.ts
import { Request, Response, NextFunction } from 'express';

interface DeprecationInfo {
  deprecatedAt: string;
  sunsetAt: string;
  link?: string;
  message?: string;
}

const DEPRECATED_VERSIONS: Record<string, DeprecationInfo> = {
  'v1': {
    deprecatedAt: '2024-01-01',
    sunsetAt: '2024-07-01',
    link: 'https://docs.example.com/migration/v1-to-v2',
    message: 'Please migrate to API v2',
  }
};

export const deprecationMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const version = req.path.match(/^\/v(\d+)/)?.[0]?.slice(1);
  
  if (version && DEPRECATED_VERSIONS[version]) {
    const info = DEPRECATED_VERSIONS[version];
    
    // ตาม RFC 8594
    res.setHeader('Deprecation', info.deprecatedAt);
    res.setHeader('Sunset', info.sunsetAt);
    
    if (info.link) {
      res.setHeader('Link', `<${info.link}>; rel="successor-version"`);
    }
    
    // Warning header
    res.setHeader(
      'Warning',
      `299 - "Deprecated API ${version}: ${info.message}"`
    );
  }
  
  next();
};
```

---

## 4. Pagination Patterns

### 4.1 Offset Pagination

```typescript
// src/services/pagination.ts
import { Repository, FindManyOptions } from 'typeorm';

interface OffsetPaginationOptions {
  page: number;
  limit: number;
  sort?: string;
  order?: 'asc' | 'desc';
}

interface PaginatedResult<T> {
  data: T[];
  meta: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

export async function paginateOffset<T>(
  repository: Repository<T>,
  options: OffsetPaginationOptions,
  where?: FindManyOptions<T>['where']
): Promise<PaginatedResult<T>> {
  const { page, limit, sort = 'createdAt', order = 'desc' } = options;
  const offset = (page - 1) * limit;

  const [data, total] = await repository.findAndCount({
    where,
    take: limit,
    skip: offset,
    order: { [sort]: order.toUpperCase() } as any,
  });

  const totalPages = Math.ceil(total / limit);

  return {
    data,
    meta: {
      page,
      limit,
      total,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1,
    },
  };
}
```

### 4.2 Cursor Pagination (สำหรับ Real-time data)

```typescript
// src/services/cursor-pagination.ts
import { DataSource } from 'typeorm';
import { encode, decode } from 'base64url';

interface CursorPaginationOptions {
  limit: number;
  cursor?: string;
  direction?: 'next' | 'prev';
}

interface CursorInfo {
  id: string;
  createdAt: string;
}

interface CursorPaginatedResult<T> {
  data: T[];
  meta: {
    limit: number;
    nextCursor?: string;
    prevCursor?: string;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

export function encodeCursor(cursor: CursorInfo): string {
  return encode(JSON.stringify(cursor));
}

export function decodeCursor(cursor: string): CursorInfo {
  return JSON.parse(decode(cursor));
}

export async function paginateCursor<T extends { id: string; createdAt: Date }>(
  dataSource: DataSource,
  tableName: string,
  options: CursorPaginationOptions
): Promise<CursorPaginatedResult<T>> {
  const { limit, cursor, direction = 'next' } = options;
  
  let query = dataSource
    .createQueryBuilder<T>(tableName, 't')
    .orderBy('t.createdAt', direction === 'next' ? 'DESC' : 'ASC')
    .addOrderBy('t.id', direction === 'next' ? 'DESC' : 'ASC')
    .take(limit + 1); // ดึงเพิ่ม 1 เพื่อเช็ค hasNext

  if (cursor) {
    const cursorInfo = decodeCursor(cursor);
    
    if (direction === 'next') {
      query = query.where(
        '(t.createdAt, t.id) < (:createdAt, :id)',
        { createdAt: cursorInfo.createdAt, id: cursorInfo.id }
      );
    } else {
      query = query.where(
        '(t.createdAt, t.id) > (:createdAt, :id)',
        { createdAt: cursorInfo.createdAt, id: cursorInfo.id }
      );
    }
  }

  const items = await query.getMany();
  const hasMore = items.length > limit;
  
  if (hasMore) items.pop();
  
  if (direction === 'prev') items.reverse();

  const nextCursor = hasMore && direction === 'next' && items.length > 0
    ? encodeCursor({
        id: items[items.length - 1].id,
        createdAt: items[items.length - 1].createdAt.toISOString(),
      })
    : undefined;

  const prevCursor = cursor && items.length > 0
    ? encodeCursor({
        id: items[0].id,
        createdAt: items[0].createdAt.toISOString(),
      })
    : undefined;

  return {
    data: items,
    meta: {
      limit,
      nextCursor,
      prevCursor,
      hasNext: hasMore && direction === 'next',
      hasPrev: !!cursor,
    },
  };
}
```

### 4.3 Keyset Pagination (สำหรับ Large Datasets)

```typescript
// src/services/keyset-pagination.ts
interface KeysetOptions {
  limit: number;
  lastId?: string;
  lastCreatedAt?: string;
}

export async function paginateKeyset(
  db: any,
  options: KeysetOptions
) {
  const { limit, lastId, lastCreatedAt } = options;
  
  let query = `
    SELECT * FROM users
    WHERE 1=1
  `;
  
  const params: any[] = [];
  
  // Keyset condition
  if (lastId && lastCreatedAt) {
    query += `
      AND (created_at, id) < ($1, $2)
    `;
    params.push(lastCreatedAt, lastId);
  }
  
  query += `
    ORDER BY created_at DESC, id DESC
    LIMIT $${params.length + 1}
  `;
  params.push(limit + 1);
  
  const rows = await db.query(query, params);
  const hasMore = rows.length > limit;
  
  if (hasMore) rows.pop();
  
  const lastRow = rows[rows.length - 1];
  
  return {
    data: rows,
    meta: {
      hasNext: hasMore,
      lastId: lastRow?.id,
      lastCreatedAt: lastRow?.createdAt,
    },
  };
}
```

---

## 5. Filtering and Sorting Patterns

### 5.1 Advanced Filtering

```typescript
// src/utils/query-builder.ts
import { SelectQueryBuilder } from 'typeorm';

type FilterOperator = 
  | 'eq' | 'ne'           // Equal, Not Equal
  | 'gt' | 'gte'          // Greater Than, Greater or Equal
  | 'lt' | 'lte'          // Less Than, Less or Equal
  | 'in' | 'nin'          // In, Not In
  | 'like' | 'ilike'      // Like, Case-insensitive Like
  | 'between'             // Between
  | 'isNull' | 'isNotNull'; // Null checks

interface FilterCondition {
  field: string;
  operator: FilterOperator;
  value: any;
}

interface SortCondition {
  field: string;
  direction: 'ASC' | 'DESC';
}

export function applyFilters<T>(
  queryBuilder: SelectQueryBuilder<T>,
  filters: FilterCondition[],
  alias: string = 't'
): SelectQueryBuilder<T> {
  filters.forEach((filter, index) => {
    const paramName = `param_${index}`;
    const fieldPath = `${alias}.${filter.field}`;
    
    switch (filter.operator) {
      case 'eq':
        queryBuilder.andWhere(`${fieldPath} = :${paramName}`, { [paramName]: filter.value });
        break;
      case 'ne':
        queryBuilder.andWhere(`${fieldPath} != :${paramName}`, { [paramName]: filter.value });
        break;
      case 'gt':
        queryBuilder.andWhere(`${fieldPath} > :${paramName}`, { [paramName]: filter.value });
        break;
      case 'gte':
        queryBuilder.andWhere(`${fieldPath} >= :${paramName}`, { [paramName]: filter.value });
        break;
      case 'lt':
        queryBuilder.andWhere(`${fieldPath} < :${paramName}`, { [paramName]: filter.value });
        break;
      case 'lte':
        queryBuilder.andWhere(`${fieldPath} <= :${paramName}`, { [paramName]: filter.value });
        break;
      case 'in':
        queryBuilder.andWhere(`${fieldPath} IN (:...${paramName})`, { [paramName]: filter.value });
        break;
      case 'nin':
        queryBuilder.andWhere(`${fieldPath} NOT IN (:...${paramName})`, { [paramName]: filter.value });
        break;
      case 'like':
        queryBuilder.andWhere(`${fieldPath} LIKE :${paramName}`, { [paramName]: `%${filter.value}%` });
        break;
      case 'ilike':
        queryBuilder.andWhere(`${fieldPath} ILIKE :${paramName}`, { [paramName]: `%${filter.value}%` });
        break;
      case 'between':
        queryBuilder.andWhere(
          `${fieldPath} BETWEEN :${paramName}_start AND :${paramName}_end`,
          { [`${paramName}_start`]: filter.value[0], [`${paramName}_end`]: filter.value[1] }
        );
        break;
      case 'isNull':
        queryBuilder.andWhere(`${fieldPath} IS NULL`);
        break;
      case 'isNotNull':
        queryBuilder.andWhere(`${fieldPath} IS NOT NULL`);
        break;
    }
  });
  
  return queryBuilder;
}

// src/utils/parse-filters.ts
export function parseQueryFilters(query: Record<string, string>): FilterCondition[] {
  const filters: FilterCondition[] = [];
  const filterPattern = /^filter\[(.+)\]\[(.+)\]$/;
  
  for (const [key, value] of Object.entries(query)) {
    const match = key.match(filterPattern);
    if (!match) continue;
    
    const [, field, operator] = match;
    
    let parsedValue: any = value;
    
    // Parse special values
    if (['in', 'nin'].includes(operator)) {
      parsedValue = value.split(',');
    } else if (operator === 'between') {
      parsedValue = value.split(',').map(v => new Date(v));
    } else if (!isNaN(Number(value))) {
      parsedValue = Number(value);
    } else if (value === 'true' || value === 'false') {
      parsedValue = value === 'true';
    }
    
    filters.push({
      field,
      operator: operator as FilterOperator,
      value: parsedValue,
    });
  }
  
  return filters;
}

// ตัวอย่างการใช้งาน:
// GET /users?filter[role][eq]=admin&filter[createdAt][gte]=2024-01-01&filter[name][ilike]=john
```

---

## 6. HATEOAS Implementation

```typescript
// src/utils/hateoas.ts
interface Link {
  href: string;
  rel: string;
  method?: string;
  title?: string;
}

interface HATEOASResponse<T> {
  data: T;
  _links: Record<string, Link>;
  _embedded?: Record<string, any>;
}

export function createHATEOASResponse<T>(
  data: T,
  links: Record<string, Link>
): HATEOASResponse<T> {
  return { data, _links: links };
}

// src/controllers/users.ts
export class UsersController {
  async getUser(req: Request, res: Response) {
    const user = await userService.findById(req.params.id);
    const baseUrl = `${req.protocol}://${req.get('host')}/api/v1`;
    
    const response = createHATEOASResponse(user, {
      self: {
        href: `${baseUrl}/users/${user.id}`,
        rel: 'self',
        method: 'GET',
      },
      update: {
        href: `${baseUrl}/users/${user.id}`,
        rel: 'update',
        method: 'PATCH',
        title: 'Update user',
      },
      delete: {
        href: `${baseUrl}/users/${user.id}`,
        rel: 'delete',
        method: 'DELETE',
        title: 'Delete user',
      },
      orders: {
        href: `${baseUrl}/users/${user.id}/orders`,
        rel: 'orders',
        method: 'GET',
        title: "User's orders",
      },
    });
    
    return res.json(response);
  }

  async listUsers(req: Request, res: Response) {
    const { page = 1, limit = 20 } = req.query;
    const result = await userService.findAll({ page: Number(page), limit: Number(limit) });
    const baseUrl = `${req.protocol}://${req.get('host')}/api/v1`;
    
    const links: Record<string, Link> = {
      self: {
        href: `${baseUrl}/users?page=${page}&limit=${limit}`,
        rel: 'self',
        method: 'GET',
      },
    };
    
    if (result.meta.hasNext) {
      links.next = {
        href: `${baseUrl}/users?page=${Number(page) + 1}&limit=${limit}`,
        rel: 'next',
        method: 'GET',
      };
    }
    
    if (result.meta.hasPrev) {
      links.prev = {
        href: `${baseUrl}/users?page=${Number(page) - 1}&limit=${limit}`,
        rel: 'prev',
        method: 'GET',
      };
    }
    
    links.first = {
      href: `${baseUrl}/users?page=1&limit=${limit}`,
      rel: 'first',
      method: 'GET',
    };
    
    links.last = {
      href: `${baseUrl}/users?page=${result.meta.totalPages}&limit=${limit}`,
      rel: 'last',
      method: 'GET',
    };
    
    return res.json({
      data: result.data,
      _links: links,
      _meta: result.meta,
    });
  }
}
```

---

## 7. API Response Formats and Error Codes

### 7.1 Standardized Response Format

```typescript
// src/utils/response.ts
interface ApiSuccess<T> {
  success: true;
  data: T;
  meta?: {
    timestamp: string;
    requestId: string;
    version: string;
  };
}

interface ApiError {
  success: false;
  error: {
    code: string;
    message: string;
    details?: ErrorDetail[];
    timestamp: string;
    requestId: string;
    documentation?: string;
  };
}

interface ErrorDetail {
  field?: string;
  code: string;
  message: string;
}

type ApiResponse<T> = ApiSuccess<T> | ApiError;

export class ResponseBuilder {
  static success<T>(data: T, requestId: string): ApiSuccess<T> {
    return {
      success: true,
      data,
      meta: {
        timestamp: new Date().toISOString(),
        requestId,
        version: process.env.API_VERSION || '1.0.0',
      },
    };
  }

  static error(
    code: string,
    message: string,
    requestId: string,
    details?: ErrorDetail[]
  ): ApiError {
    return {
      success: false,
      error: {
        code,
        message,
        details,
        timestamp: new Date().toISOString(),
        requestId,
        documentation: `https://docs.example.com/errors/${code}`,
      },
    };
  }
}

// Error codes ที่ใช้บ่อย
export const ERROR_CODES = {
  // Authentication
  AUTH_TOKEN_MISSING: 'AUTH_TOKEN_MISSING',
  AUTH_TOKEN_INVALID: 'AUTH_TOKEN_INVALID',
  AUTH_TOKEN_EXPIRED: 'AUTH_TOKEN_EXPIRED',
  
  // Authorization
  PERMISSION_DENIED: 'PERMISSION_DENIED',
  INSUFFICIENT_SCOPE: 'INSUFFICIENT_SCOPE',
  
  // Validation
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  INVALID_FIELD: 'INVALID_FIELD',
  REQUIRED_FIELD_MISSING: 'REQUIRED_FIELD_MISSING',
  
  // Resource
  RESOURCE_NOT_FOUND: 'RESOURCE_NOT_FOUND',
  RESOURCE_ALREADY_EXISTS: 'RESOURCE_ALREADY_EXISTS',
  RESOURCE_CONFLICT: 'RESOURCE_CONFLICT',
  RESOURCE_DELETED: 'RESOURCE_DELETED',
  
  // Rate Limiting
  RATE_LIMIT_EXCEEDED: 'RATE_LIMIT_EXCEEDED',
  QUOTA_EXCEEDED: 'QUOTA_EXCEEDED',
  
  // Server
  INTERNAL_ERROR: 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE: 'SERVICE_UNAVAILABLE',
  DATABASE_ERROR: 'DATABASE_ERROR',
} as const;
```

### 7.2 Global Error Handler

```typescript
// src/middleware/error-handler.ts
import { Request, Response, NextFunction } from 'express';
import { ResponseBuilder, ERROR_CODES } from '../utils/response';

export class AppError extends Error {
  constructor(
    public statusCode: number,
    public code: string,
    message: string,
    public details?: any[]
  ) {
    super(message);
    this.name = 'AppError';
  }
}

export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const requestId = req.headers['x-request-id'] as string || 'unknown';

  if (err instanceof AppError) {
    return res.status(err.statusCode).json(
      ResponseBuilder.error(err.code, err.message, requestId, err.details)
    );
  }

  // Zod validation error
  if (err.name === 'ZodError') {
    const details = (err as any).errors.map((e: any) => ({
      field: e.path.join('.'),
      code: 'INVALID_FIELD',
      message: e.message,
    }));
    
    return res.status(422).json(
      ResponseBuilder.error(
        ERROR_CODES.VALIDATION_ERROR,
        'Validation failed',
        requestId,
        details
      )
    );
  }

  // Default: Internal server error
  console.error('Unhandled error:', err);
  
  return res.status(500).json(
    ResponseBuilder.error(
      ERROR_CODES.INTERNAL_ERROR,
      'An unexpected error occurred',
      requestId
    )
  );
};
```

---

## 8. Idempotency Keys

```typescript
// src/middleware/idempotency.ts
import { Request, Response, NextFunction } from 'express';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

interface CachedResponse {
  statusCode: number;
  body: any;
  headers: Record<string, string>;
  createdAt: string;
}

export const idempotencyMiddleware = (ttlSeconds: number = 86400) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    // Only apply to POST, PUT, PATCH
    if (!['POST', 'PUT', 'PATCH'].includes(req.method)) {
      return next();
    }
    
    const idempotencyKey = req.headers['idempotency-key'] as string;
    if (!idempotencyKey) return next();
    
    // Validate key format (UUID v4)
    const uuidV4Pattern = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
    if (!uuidV4Pattern.test(idempotencyKey)) {
      return res.status(400).json({
        error: {
          code: 'INVALID_IDEMPOTENCY_KEY',
          message: 'Idempotency-Key must be a valid UUID v4',
        }
      });
    }
    
    const cacheKey = `idempotency:${req.path}:${idempotencyKey}`;
    
    // Check for existing response
    const cached = await redis.get(cacheKey);
    if (cached) {
      const cachedResponse: CachedResponse = JSON.parse(cached);
      
      res.setHeader('Idempotency-Replayed', 'true');
      res.setHeader('Idempotency-Key', idempotencyKey);
      
      Object.entries(cachedResponse.headers).forEach(([key, value]) => {
        res.setHeader(key, value);
      });
      
      return res.status(cachedResponse.statusCode).json(cachedResponse.body);
    }
    
    // Check for in-flight request (concurrent requests with same key)
    const inFlight = await redis.get(`${cacheKey}:in-flight`);
    if (inFlight) {
      return res.status(409).json({
        error: {
          code: 'IDEMPOTENCY_KEY_IN_USE',
          message: 'A request with this idempotency key is already being processed',
        }
      });
    }
    
    // Mark as in-flight
    await redis.setex(`${cacheKey}:in-flight`, 30, '1');
    
    // Intercept response
    const originalJson = res.json.bind(res);
    res.json = (body: any) => {
      const response: CachedResponse = {
        statusCode: res.statusCode,
        body,
        headers: {
          'Content-Type': 'application/json',
        },
        createdAt: new Date().toISOString(),
      };
      
      // Cache successful responses only (2xx)
      if (res.statusCode >= 200 && res.statusCode < 300) {
        redis.setex(cacheKey, ttlSeconds, JSON.stringify(response));
      }
      
      // Remove in-flight marker
      redis.del(`${cacheKey}:in-flight`);
      
      res.setHeader('Idempotency-Key', idempotencyKey);
      return originalJson(body);
    };
    
    next();
  };
};

// ใช้งาน:
// POST /orders
// Headers: Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

---

## 9. API Deprecation Strategy

### 9.1 Deprecation Notice System

```typescript
// src/services/deprecation.ts
interface DeprecatedEndpoint {
  path: string;
  method: string;
  deprecatedAt: Date;
  sunsetAt: Date;
  successor?: string;
  migrationGuide?: string;
  reason: string;
}

export const DEPRECATED_ENDPOINTS: DeprecatedEndpoint[] = [
  {
    path: '/api/v1/users',
    method: 'GET',
    deprecatedAt: new Date('2024-01-01'),
    sunsetAt: new Date('2024-07-01'),
    successor: '/api/v2/users',
    migrationGuide: 'https://docs.example.com/migration/users-v1-to-v2',
    reason: 'V2 includes cursor-based pagination and better performance',
  },
  {
    path: '/api/v1/users/:id/profile',
    method: 'GET',
    deprecatedAt: new Date('2024-02-01'),
    sunsetAt: new Date('2024-08-01'),
    successor: '/api/v2/users/:id',
    reason: 'Profile data merged into user object in v2',
  },
];

export class DeprecationService {
  findDeprecated(method: string, path: string): DeprecatedEndpoint | null {
    return DEPRECATED_ENDPOINTS.find(endpoint => {
      const pathPattern = endpoint.path.replace(/:[^/]+/g, '[^/]+');
      const regex = new RegExp(`^${pathPattern}$`);
      return endpoint.method === method && regex.test(path);
    }) || null;
  }
  
  isExpired(endpoint: DeprecatedEndpoint): boolean {
    return new Date() > endpoint.sunsetAt;
  }
  
  getDaysUntilSunset(endpoint: DeprecatedEndpoint): number {
    const diff = endpoint.sunsetAt.getTime() - Date.now();
    return Math.ceil(diff / (1000 * 60 * 60 * 24));
  }
}

// Middleware
export const deprecationMiddleware = (service: DeprecationService) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const deprecated = service.findDeprecated(req.method, req.path);
    
    if (!deprecated) return next();
    
    if (service.isExpired(deprecated)) {
      return res.status(410).json({
        error: {
          code: 'ENDPOINT_GONE',
          message: `This endpoint was removed on ${deprecated.sunsetAt.toISOString()}`,
          successor: deprecated.successor,
          migrationGuide: deprecated.migrationGuide,
        }
      });
    }
    
    const daysLeft = service.getDaysUntilSunset(deprecated);
    
    res.setHeader('Deprecation', deprecated.deprecatedAt.toUTCString());
    res.setHeader('Sunset', deprecated.sunsetAt.toUTCString());
    res.setHeader('X-Deprecation-Days-Remaining', daysLeft.toString());
    
    if (deprecated.successor) {
      res.setHeader('Link', `<${deprecated.successor}>; rel="successor-version"`);
    }
    
    next();
  };
};
```

---

## 10. API Mocking with Prism

### 10.1 Setup Prism สำหรับ Mock Server

```yaml
# docker-compose.yml
version: '3.8'
services:
  prism:
    image: stoplight/prism:4
    command: mock -h 0.0.0.0 /api/openapi.yaml
    volumes:
      - ./openapi.yaml:/api/openapi.yaml:ro
    ports:
      - "4010:4010"
    environment:
      - LOG_LEVEL=info
  
  # Prism as proxy (validates real requests)
  prism-proxy:
    image: stoplight/prism:4
    command: proxy /api/openapi.yaml http://api:3000 -h 0.0.0.0
    volumes:
      - ./openapi.yaml:/api/openapi.yaml:ro
    ports:
      - "4011:4010"
```

### 10.2 Test Suite with Prism

```typescript
// tests/contract/prism-test.ts
import axios from 'axios';
import { spawn, ChildProcess } from 'child_process';

const PRISM_PORT = 4010;
const PRISM_URL = `http://localhost:${PRISM_PORT}`;

let prismProcess: ChildProcess;

beforeAll(async () => {
  prismProcess = spawn('npx', [
    '@stoplight/prism-cli',
    'mock',
    '-p', PRISM_PORT.toString(),
    '-h', '0.0.0.0',
    'openapi.yaml'
  ]);
  
  // Wait for Prism to start
  await new Promise(resolve => setTimeout(resolve, 2000));
});

afterAll(() => {
  prismProcess.kill();
});

describe('Users API Contract Tests', () => {
  test('GET /users returns valid response', async () => {
    const response = await axios.get(`${PRISM_URL}/api/v1/users`, {
      headers: { Authorization: 'Bearer mock-token' }
    });
    
    expect(response.status).toBe(200);
    expect(response.data).toHaveProperty('data');
    expect(Array.isArray(response.data.data)).toBe(true);
  });

  test('POST /users creates user', async () => {
    const response = await axios.post(
      `${PRISM_URL}/api/v1/users`,
      { name: 'Test User', email: 'test@example.com' },
      { headers: { Authorization: 'Bearer mock-token' } }
    );
    
    expect(response.status).toBe(201);
    expect(response.data.data).toHaveProperty('id');
  });

  test('GET /users/:id returns 404 for unknown id', async () => {
    try {
      await axios.get(
        `${PRISM_URL}/api/v1/users/00000000-0000-0000-0000-000000000000`,
        { headers: { Authorization: 'Bearer mock-token' } }
      );
    } catch (error: any) {
      expect(error.response.status).toBe(404);
    }
  });
});
```

### 10.3 Consumer-Driven Contract Testing

```typescript
// tests/contract/pact-test.ts
import { Pact, Interaction } from '@pact-foundation/pact';
import axios from 'axios';
import path from 'path';

const provider = new Pact({
  consumer: 'OrderService',
  provider: 'UserService',
  port: 4011,
  dir: path.resolve(process.cwd(), 'pacts'),
  log: path.resolve(process.cwd(), 'pacts/logs'),
});

describe('UserService Contract', () => {
  beforeAll(() => provider.setup());
  afterEach(() => provider.verify());
  afterAll(() => provider.finalize());

  describe('GET /users/:id', () => {
    const userId = '550e8400-e29b-41d4-a716-446655440000';
    
    beforeEach(() => {
      const interaction: Interaction = {
        state: 'User exists',
        uponReceiving: 'A request for user',
        withRequest: {
          method: 'GET',
          path: `/api/v1/users/${userId}`,
          headers: {
            Authorization: 'Bearer token',
          },
        },
        willRespondWith: {
          status: 200,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            data: {
              id: userId,
              name: 'John Doe',
              email: 'john@example.com',
              role: 'user',
            },
          },
        },
      };
      
      return provider.addInteraction(interaction);
    });

    test('returns user data', async () => {
      const response = await axios.get(
        `http://localhost:4011/api/v1/users/${userId}`,
        { headers: { Authorization: 'Bearer token' } }
      );
      
      expect(response.data.data.id).toBe(userId);
      expect(response.data.data.name).toBe('John Doe');
    });
  });
});
```

---

## สรุปท้ายบท

| หัวข้อ | แนวปฏิบัติที่ดีที่สุด | เครื่องมือ |
|--------|---------------------|-----------|
| RESTful Design | ใช้ nouns, HTTP methods ถูกต้อง, Status codes ถูกต้อง | - |
| OpenAPI Specification | เขียน spec ก่อน code, ใช้ $ref สำหรับ reuse | swagger-ui, redoc |
| Versioning | URL path versioning แนะนำสำหรับ public APIs | - |
| Pagination | Cursor สำหรับ real-time, Offset สำหรับ simple cases | - |
| Filtering | Standard filter syntax, sanitize inputs | - |
| HATEOAS | ใส่ _links ใน response | - |
| Error Handling | Standard error format, machine-readable codes | - |
| Idempotency | UUID keys, Redis caching | Redis |
| Deprecation | RFC 8594 headers, sunset dates | - |
| API Mocking | Prism สำหรับ mock และ validation | Stoplight Prism |

### คำแนะนำสำหรับ Production

1. **เสมอใช้ HTTPS** - ไม่มี HTTP ใน production
2. **Rate Limiting** - ทุก endpoint ต้องมี rate limit
3. **Request Validation** - validate ที่ edge ก่อนถึง business logic
4. **Response Compression** - ใช้ gzip/brotli compression
5. **API Gateway** - ใช้สำหรับ centralized authentication, rate limiting, logging
6. **Monitoring** - ติดตาม latency, error rates, traffic patterns
7. **Documentation** - อัปเดต OpenAPI spec ทุกครั้งที่เปลี่ยน API

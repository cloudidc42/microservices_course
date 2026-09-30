# Part 08: REST API Design Best Practices
## ออกแบบ REST API ที่ดี สำหรับ Microservices

> **ระดับ:** ⭐⭐ พื้นฐาน | **เวลาเรียน:** 4-5 ชั่วโมง | **Prerequisites:** Part 07

---

## 🎯 สิ่งที่จะได้เรียนรู้

- REST API Principles และ Richardson Maturity Model
- HTTP Methods, Status Codes, Headers
- URL Design Best Practices
- Request/Response Format Standards
- API Versioning Strategies
- Error Response Standards
- OpenAPI/Swagger Documentation
- Workshop: Design E-Commerce APIs

---

## 1. REST Principles

### 1.1 REST คืออะไร?

REST (Representational State Transfer) เป็น Architectural Style ที่กำหนดข้อตกลงสำหรับการออกแบบ API

**6 Constraints ของ REST:**
```
1. Client-Server    - แยก UI จาก Business Logic
2. Stateless        - Server ไม่เก็บ client state
3. Cacheable        - Response ระบุว่า cacheable ได้หรือไม่
4. Uniform Interface - API มีรูปแบบสม่ำเสมอ
5. Layered System   - Client ไม่รู้ว่า server ทำงานกี่ layer
6. Code on Demand   - (Optional) Server ส่ง executable code ได้
```

### 1.2 Richardson Maturity Model

```
Level 0: The Swamp of POX (Plain Old XML/JSON)
  POST /api
  Body: { "action": "getUser", "userId": 1 }
  ← ทุก request ไปที่ URL เดียว

Level 1: Resources
  GET /users/1
  GET /orders/1
  ← แยก URLs ตาม Resources

Level 2: HTTP Verbs
  GET /users/1      ← ดึงข้อมูล
  POST /users       ← สร้างใหม่
  PUT /users/1      ← อัปเดต
  DELETE /users/1   ← ลบ
  ← ใช้ HTTP Methods ถูกต้อง

Level 3: Hypermedia (HATEOAS)
  GET /users/1
  Response: {
    "id": 1,
    "name": "สมชาย",
    "_links": {
      "self": "/users/1",
      "orders": "/users/1/orders",
      "update": "/users/1"
    }
  }
  ← Response มี links สำหรับ next actions
  (Production Microservices ส่วนใหญ่อยู่ Level 2)
```

---

## 2. URL Design

### 2.1 Resource Naming

```
✅ Good URL Design:

# Collections (Plural Nouns)
GET    /users              # Get all users
POST   /users              # Create user

# Single Resource
GET    /users/123          # Get user 123
PUT    /users/123          # Update user 123
PATCH  /users/123          # Partial update
DELETE /users/123          # Delete user 123

# Nested Resources (Relationships)
GET    /users/123/orders   # Get orders of user 123
POST   /users/123/orders   # Create order for user 123
GET    /users/123/orders/456  # Get specific order

# Actions (เมื่อ REST ไม่ fit)
POST   /users/123/activate    # Activate user
POST   /orders/456/cancel     # Cancel order
POST   /payments/789/refund   # Refund payment
```

```
❌ Bad URL Design:

GET /getUser             ← ไม่ใช้ Verb ใน URL
GET /user                ← ควร Plural
POST /user/create        ← Method ซ้ำซ้อน
GET /getUserById/123     ← ไม่ RESTful
DELETE /user/delete/123  ← Verb ซ้ำซ้อน
GET /User/123            ← ไม่ lowercase
GET /users/123/get-order-items  ← ซับซ้อนเกิน
```

### 2.2 Query Parameters

```
# Filtering
GET /products?category=laptop&price_min=10000&price_max=50000
GET /orders?status=pending&user_id=123

# Searching
GET /products?search=MacBook
GET /users?q=สมชาย

# Sorting
GET /products?sort=price&order=asc
GET /orders?sort=created_at&order=desc

# Pagination
GET /products?page=2&limit=20
GET /products?offset=20&limit=20
GET /products?cursor=eyJpZCI6MTB9  # Cursor-based (better for large datasets)

# Field Selection (Sparse Fieldsets)
GET /users?fields=id,name,email
GET /orders?include=items,user

# Multiple values
GET /products?category=laptop,phone  # comma-separated
GET /products?category[]=laptop&category[]=phone  # repeated params
```

### 2.3 API Versioning

```
# Option 1: URI Versioning (Most common, most visible)
GET /v1/users/123
GET /v2/users/123

# Option 2: Query Parameter
GET /users/123?version=2
GET /users/123?api-version=2024-01-15

# Option 3: Header Versioning (Cleanest URLs)
GET /users/123
Accept: application/vnd.myapp.v2+json
# หรือ
X-API-Version: 2

# Option 4: Content Type Negotiation
Accept: application/vnd.myapp+json;version=2

# สำหรับ Microservices: URI Versioning แนะนำ
# เพราะ clear, easy to route in API Gateway
```

---

## 3. HTTP Methods

### 3.1 Method Semantics

```
GET     - Read resource (Safe + Idempotent)
POST    - Create resource or action (Not safe, Not idempotent)
PUT     - Replace entire resource (Idempotent)
PATCH   - Partial update (Not necessarily idempotent)
DELETE  - Remove resource (Idempotent)
HEAD    - Like GET but without body (headers only)
OPTIONS - Discover allowed methods (CORS preflight)
```

### 3.2 Safe vs Idempotent

```
Safe: ไม่เปลี่ยน state ของ server
  GET, HEAD, OPTIONS

Idempotent: เรียกกี่ครั้งก็ได้ผลเหมือนกัน
  GET, PUT, DELETE, HEAD, OPTIONS

  PUT /users/123 (body: {name: "สมหมาย"})
  → เรียก 10 ครั้ง = ผลเหมือนกัน (name = สมหมาย)

Not Idempotent:
  POST /orders (สร้าง order ใหม่ทุกครั้ง)
  → เรียก 10 ครั้ง = สร้าง 10 orders!
```

### 3.3 PUT vs PATCH

```javascript
// PUT - Replace entire resource
// Client ต้องส่งทุก field
PUT /users/123
{
  "name": "ชื่อใหม่",
  "email": "old@example.com",   // ต้องส่งด้วย แม้ไม่ได้เปลี่ยน
  "role": "customer"             // ต้องส่งด้วย
}

// PATCH - Partial update
// Client ส่งเฉพาะ field ที่ต้องการเปลี่ยน
PATCH /users/123
{
  "name": "ชื่อใหม่"  // เปลี่ยนเฉพาะ name
}
```

---

## 4. HTTP Status Codes

### 4.1 Complete Status Code Guide

```
2xx - Success
  200 OK              - Request successful (GET, PUT, PATCH)
  201 Created         - Resource created (POST)
  202 Accepted        - Request accepted (async processing)
  204 No Content      - Success, no body (DELETE)
  206 Partial Content - Partial GET (pagination/range)

3xx - Redirection
  301 Moved Permanently  - Resource moved permanently
  302 Found              - Temporary redirect
  304 Not Modified       - Client cache still valid

4xx - Client Error
  400 Bad Request        - Invalid syntax/request
  401 Unauthorized       - Not authenticated
  403 Forbidden          - Authenticated but not authorized
  404 Not Found          - Resource doesn't exist
  405 Method Not Allowed - HTTP method not supported
  408 Request Timeout    - Client too slow
  409 Conflict           - Resource conflict (duplicate email)
  410 Gone               - Resource permanently deleted
  412 Precondition Failed- Conditional update failed
  422 Unprocessable Entity - Validation failed
  429 Too Many Requests  - Rate limited

5xx - Server Error
  500 Internal Server Error - Unexpected server error
  502 Bad Gateway           - Upstream service error
  503 Service Unavailable   - Server overloaded/maintenance
  504 Gateway Timeout       - Upstream timeout
```

### 4.2 Status Code Usage Examples

```javascript
// 200 OK
app.get('/users/:id', async (req, res) => {
  const user = await getUser(req.params.id);
  res.status(200).json({ data: user });  // หรือแค่ res.json()
});

// 201 Created
app.post('/users', async (req, res) => {
  const user = await createUser(req.body);
  res.status(201)
    .header('Location', `/users/${user.id}`)  // Location header!
    .json({ data: user });
});

// 202 Accepted (Async processing)
app.post('/reports/generate', async (req, res) => {
  const job = await queueReportGeneration(req.body);
  res.status(202).json({
    message: 'Report generation started',
    jobId: job.id,
    statusUrl: `/jobs/${job.id}`,
  });
});

// 204 No Content
app.delete('/users/:id', async (req, res) => {
  await deleteUser(req.params.id);
  res.status(204).send();  // No body!
});

// 400 Bad Request
app.post('/users', async (req, res) => {
  if (!req.body.email) {
    return res.status(400).json({
      error: 'VALIDATION_ERROR',
      message: 'Email is required',
    });
  }
});

// 401 Unauthorized
app.get('/profile', authenticate, async (req, res) => {
  // authenticate middleware sets 401 if no token
});

// 409 Conflict
app.post('/users', async (req, res) => {
  const exists = await emailExists(req.body.email);
  if (exists) {
    return res.status(409).json({
      error: 'CONFLICT',
      message: 'Email already registered',
    });
  }
});

// 429 Too Many Requests
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100,
  handler: (req, res) => {
    res.status(429).json({
      error: 'RATE_LIMIT_EXCEEDED',
      message: 'Too many requests',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000),
    });
  }
}));
```

---

## 5. Response Format

### 5.1 Consistent Response Structure

```javascript
// ✅ Consistent API Response Format

// Success (single resource)
{
  "success": true,
  "data": {
    "id": 1,
    "name": "สมชาย",
    "email": "somchai@example.com"
  }
}

// Success (collection)
{
  "success": true,
  "data": [
    { "id": 1, "name": "สมชาย" },
    { "id": 2, "name": "สมหญิง" }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "totalPages": 10,
    "hasNext": true,
    "hasPrev": false
  }
}

// Error Response
{
  "success": false,
  "error": "VALIDATION_ERROR",
  "message": "Invalid input data",
  "details": [
    { "field": "email", "message": "Invalid email format" },
    { "field": "name", "message": "Name is required" }
  ],
  "requestId": "req-abc123",
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

### 5.2 Response Helper

```javascript
// src/utils/response.js
class ApiResponse {
  static success(res, data, statusCode = 200, message = null) {
    const response = { success: true };
    if (message) response.message = message;
    response.data = data;
    return res.status(statusCode).json(response);
  }
  
  static paginated(res, data, pagination) {
    return res.json({
      success: true,
      data,
      pagination: {
        page: pagination.page,
        limit: pagination.limit,
        total: pagination.total,
        totalPages: Math.ceil(pagination.total / pagination.limit),
        hasNext: pagination.page < Math.ceil(pagination.total / pagination.limit),
        hasPrev: pagination.page > 1,
      },
    });
  }
  
  static created(res, data, location = null) {
    if (location) res.setHeader('Location', location);
    return res.status(201).json({ success: true, data });
  }
  
  static noContent(res) {
    return res.status(204).send();
  }
  
  static error(res, statusCode, code, message, details = null) {
    const response = {
      success: false,
      error: code,
      message,
      timestamp: new Date().toISOString(),
    };
    if (details) response.details = details;
    return res.status(statusCode).json(response);
  }
}

module.exports = ApiResponse;
```

---

## 6. HTTP Headers

### 6.1 Important Request Headers

```
Authorization: Bearer <jwt-token>
Content-Type: application/json
Accept: application/json
X-Request-ID: uuid-for-tracing
X-Idempotency-Key: unique-key-for-POST
If-None-Match: "etag-value"         # Conditional GET
If-Modified-Since: date             # Conditional GET
X-API-Version: 2                    # Version header
Accept-Language: th, en-US          # Language preference
```

### 6.2 Important Response Headers

```
Content-Type: application/json; charset=utf-8
X-Request-ID: uuid-from-request
X-Rate-Limit-Limit: 100
X-Rate-Limit-Remaining: 95
X-Rate-Limit-Reset: 1705320000
Location: /users/123                # After POST
ETag: "abc123"                      # For caching
Cache-Control: max-age=300          # Cache duration
Last-Modified: Mon, 15 Jan 2024 ... # For caching
```

### 6.3 Implementing Important Headers

```javascript
// Request ID Middleware
const { v4: uuidv4 } = require('uuid');

function requestId(req, res, next) {
  // Use incoming request ID or generate new one
  const requestId = req.headers['x-request-id'] || uuidv4();
  req.requestId = requestId;
  res.setHeader('X-Request-ID', requestId);
  next();
}

// ETag for caching
const etag = require('etag');

app.get('/products/:id', async (req, res) => {
  const product = await getProduct(req.params.id);
  
  const productETag = etag(JSON.stringify(product));
  
  // Check If-None-Match header
  if (req.headers['if-none-match'] === productETag) {
    return res.status(304).send();  // Not Modified
  }
  
  res.setHeader('ETag', productETag);
  res.setHeader('Cache-Control', 'max-age=300');
  res.json({ data: product });
});

// Idempotency Key (สำหรับ POST requests)
const processedRequests = new Map();

function idempotencyCheck(req, res, next) {
  const key = req.headers['x-idempotency-key'];
  
  if (!key) return next();  // Optional header
  
  if (processedRequests.has(key)) {
    const cached = processedRequests.get(key);
    return res.status(cached.status).json(cached.body);
  }
  
  // Override res.json to cache response
  const originalJson = res.json.bind(res);
  res.json = (body) => {
    processedRequests.set(key, { status: res.statusCode, body });
    return originalJson(body);
  };
  
  next();
}
```

---

## 7. Pagination Patterns

### 7.1 Offset-based Pagination

```javascript
// GET /products?page=2&limit=10
app.get('/products', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const offset = (page - 1) * limit;
  
  const [products, total] = await Promise.all([
    db.query('SELECT * FROM products LIMIT $1 OFFSET $2', [limit, offset]),
    db.query('SELECT COUNT(*) FROM products')
  ]);
  
  const totalCount = parseInt(total.rows[0].count);
  const totalPages = Math.ceil(totalCount / limit);
  
  res.json({
    data: products.rows,
    pagination: {
      page,
      limit,
      total: totalCount,
      totalPages,
      hasNext: page < totalPages,
      hasPrev: page > 1,
      links: {
        self: `/products?page=${page}&limit=${limit}`,
        first: `/products?page=1&limit=${limit}`,
        last: `/products?page=${totalPages}&limit=${limit}`,
        next: page < totalPages ? `/products?page=${page + 1}&limit=${limit}` : null,
        prev: page > 1 ? `/products?page=${page - 1}&limit=${limit}` : null,
      }
    }
  });
});
```

### 7.2 Cursor-based Pagination (ดีกว่าสำหรับ Real-time data)

```javascript
// GET /orders?cursor=eyJpZCI6MTB9&limit=10
app.get('/orders', async (req, res) => {
  const limit = parseInt(req.query.limit) || 10;
  let cursor = null;
  
  if (req.query.cursor) {
    cursor = JSON.parse(Buffer.from(req.query.cursor, 'base64').toString());
  }
  
  const query = cursor
    ? 'SELECT * FROM orders WHERE id > $1 ORDER BY id LIMIT $2'
    : 'SELECT * FROM orders ORDER BY id LIMIT $1';
  
  const params = cursor ? [cursor.id, limit + 1] : [limit + 1];
  const result = await db.query(query, params);
  
  const hasMore = result.rows.length > limit;
  const orders = hasMore ? result.rows.slice(0, -1) : result.rows;
  
  const nextCursor = hasMore 
    ? Buffer.from(JSON.stringify({ id: orders[orders.length - 1].id })).toString('base64')
    : null;
  
  res.json({
    data: orders,
    pagination: {
      limit,
      hasMore,
      nextCursor,
      nextUrl: nextCursor ? `/orders?cursor=${nextCursor}&limit=${limit}` : null,
    }
  });
});
```

---

## 8. OpenAPI/Swagger Documentation

### 8.1 Complete API Spec Example

```yaml
# openapi.yaml
openapi: '3.0.3'
info:
  title: E-Commerce Microservices API
  version: '1.0.0'
  description: |
    ## Overview
    REST API สำหรับระบบ E-Commerce Microservices
    
    ## Authentication
    ใช้ JWT Bearer token สำหรับ authenticated endpoints
    
    ## Rate Limiting
    - 100 requests per 15 minutes สำหรับ public endpoints
    - 1000 requests per 15 minutes สำหรับ authenticated endpoints

servers:
  - url: http://localhost:8080/api
    description: Local Development
  - url: https://api.myshop.com/v1
    description: Production

security:
  - bearerAuth: []

tags:
  - name: Auth
    description: Authentication operations
  - name: Users
    description: User management
  - name: Products
    description: Product catalog

paths:
  /auth/login:
    post:
      tags: [Auth]
      summary: Login
      security: []  # Public endpoint
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email:
                  type: string
                  format: email
                  example: user@example.com
                password:
                  type: string
                  minLength: 8
                  example: Password123
      responses:
        200:
          description: Login successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: object
                    properties:
                      user:
                        $ref: '#/components/schemas/User'
                      accessToken:
                        type: string
                      refreshToken:
                        type: string
        401:
          $ref: '#/components/responses/Unauthorized'

  /users:
    get:
      tags: [Users]
      summary: List users
      parameters:
        - in: query
          name: page
          schema:
            type: integer
            default: 1
        - in: query
          name: limit
          schema:
            type: integer
            default: 10
            maximum: 100
        - in: query
          name: search
          schema:
            type: string
      responses:
        200:
          description: List of users
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/User'
                  pagination:
                    $ref: '#/components/schemas/Pagination'

    post:
      tags: [Users]
      summary: Create user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        201:
          description: User created
          headers:
            Location:
              schema:
                type: string
              description: URL of created resource
          content:
            application/json:
              schema:
                type: object
                properties:
                  success:
                    type: boolean
                  data:
                    $ref: '#/components/schemas/User'
        409:
          $ref: '#/components/responses/Conflict'

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
          example: 1
        email:
          type: string
          format: email
          example: user@example.com
        name:
          type: string
          example: สมชาย ใจดี
        role:
          type: string
          enum: [customer, admin, staff]
        isActive:
          type: boolean
        createdAt:
          type: string
          format: date-time

    CreateUserRequest:
      type: object
      required: [email, name, password]
      properties:
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 2
          maxLength: 100
        password:
          type: string
          minLength: 8
          description: Must contain uppercase, lowercase, and number
        role:
          type: string
          enum: [customer, admin, staff]
          default: customer

    Pagination:
      type: object
      properties:
        page:
          type: integer
        limit:
          type: integer
        total:
          type: integer
        totalPages:
          type: integer
        hasNext:
          type: boolean
        hasPrev:
          type: boolean

    Error:
      type: object
      properties:
        success:
          type: boolean
          example: false
        error:
          type: string
        message:
          type: string
        requestId:
          type: string

  responses:
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            success: false
            error: NOT_FOUND
            message: User not found

    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    Conflict:
      description: Resource conflict
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
```

---

## 9. Workshop: Design E-Commerce APIs

### 9.1 Complete API Design

```javascript
// routes/products.routes.js - Complete Product API

/**
 * @swagger
 * /products:
 *   get:
 *     tags: [Products]
 *     summary: Search and filter products
 *     parameters:
 *       - in: query
 *         name: search
 *         schema:
 *           type: string
 *       - in: query
 *         name: category
 *         schema:
 *           type: string
 *       - in: query
 *         name: price_min
 *         schema:
 *           type: number
 *       - in: query
 *         name: price_max
 *         schema:
 *           type: number
 *       - in: query
 *         name: sort
 *         schema:
 *           type: string
 *           enum: [price, name, created_at]
 *       - in: query
 *         name: order
 *         schema:
 *           type: string
 *           enum: [asc, desc]
 */
router.get('/',
  validate(schemas.getProducts, 'query'),
  productsController.getProducts
);

/**
 * @swagger
 * /products/{id}/reviews:
 *   get:
 *     tags: [Products]
 *     summary: Get product reviews
 */
router.get('/:id/reviews',
  productsController.getProductReviews
);

/**
 * @swagger
 * /products/{id}/reviews:
 *   post:
 *     tags: [Products]
 *     summary: Add review to product
 *     security:
 *       - bearerAuth: []
 */
router.post('/:id/reviews',
  authenticate,
  validate(schemas.createReview),
  productsController.addReview
);
```

### 9.2 API Design สำหรับ Order Service

```javascript
// Order Service APIs Design

// ── Orders ──
POST   /orders                    # สร้าง order
GET    /orders/:id                # ดู order
GET    /orders?userId=123         # Orders ของ user
PATCH  /orders/:id/status         # อัปเดต status
DELETE /orders/:id                # Cancel order (soft delete)

// ── Order Items ──
GET    /orders/:id/items          # Items ใน order
POST   /orders/:id/items          # เพิ่ม item
DELETE /orders/:id/items/:itemId  # ลบ item

// ── Order Actions ──
POST   /orders/:id/confirm        # Confirm order
POST   /orders/:id/cancel         # Cancel order  
POST   /orders/:id/refund         # Request refund

// Response Examples:

// POST /orders
{
  "success": true,
  "data": {
    "id": "ord_123",
    "userId": 1,
    "status": "pending",
    "items": [
      {
        "productId": 1,
        "name": "MacBook Pro",
        "quantity": 1,
        "price": 75000,
        "subtotal": 75000
      }
    ],
    "subtotal": 75000,
    "shipping": 0,
    "discount": 0,
    "total": 75000,
    "currency": "THB",
    "createdAt": "2024-01-15T10:00:00Z"
  }
}

// PATCH /orders/ord_123/status
// Request
{
  "status": "confirmed",
  "note": "Payment received"
}

// Response
{
  "success": true,
  "data": {
    "id": "ord_123",
    "status": "confirmed",
    "statusHistory": [
      { "status": "pending", "timestamp": "2024-01-15T10:00:00Z" },
      { "status": "confirmed", "timestamp": "2024-01-15T10:05:00Z", "note": "Payment received" }
    ]
  }
}
```

---

## 10. API Testing

```bash
# test-api.sh - Comprehensive API Tests
BASE_URL="http://localhost:8080/api"

echo "🔍 Testing E-Commerce API..."

# Test: Create User
echo "--- Create User ---"
USER_RESPONSE=$(curl -s -X POST "$BASE_URL/users" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ทดสอบ ระบบ",
    "email": "test@example.com",
    "password": "TestPass123"
  }')
echo $USER_RESPONSE | jq
USER_ID=$(echo $USER_RESPONSE | jq -r '.data.id')

# Test: Login
echo "--- Login ---"
LOGIN_RESPONSE=$(curl -s -X POST "$BASE_URL/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "TestPass123"}')
TOKEN=$(echo $LOGIN_RESPONSE | jq -r '.data.accessToken')
echo "Token: ${TOKEN:0:50}..."

# Test: Get Profile
echo "--- Get Profile ---"
curl -s "$BASE_URL/profile" \
  -H "Authorization: Bearer $TOKEN" | jq

# Test: Create Product
echo "--- Create Product ---"
PRODUCT_RESPONSE=$(curl -s -X POST "$BASE_URL/products" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name": "Test Product", "price": 100, "stock": 10}')
PRODUCT_ID=$(echo $PRODUCT_RESPONSE | jq -r '.data.id')

# Test: Create Order
echo "--- Create Order ---"
curl -s -X POST "$BASE_URL/orders" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"items\": [{\"productId\": $PRODUCT_ID, \"quantity\": 1}]
  }" | jq

# Test: Rate Limiting
echo "--- Test Rate Limiting ---"
for i in {1..110}; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$BASE_URL/products")
  if [ "$STATUS" = "429" ]; then
    echo "Rate limit hit at request $i"
    break
  fi
done

echo "✅ API Tests Complete!"
```

---

## 11. สรุป

### REST API Design Checklist

```
URLs:
✅ Plural nouns (/users ไม่ใช่ /user)
✅ Lowercase และ hyphens (/user-profiles)
✅ Nested resources สำหรับ relationships (/users/:id/orders)
✅ Actions เป็น sub-resources (/orders/:id/cancel)
✅ Version ใน URL (/v1/users)

HTTP Methods:
✅ GET สำหรับ Read
✅ POST สำหรับ Create
✅ PUT สำหรับ Full Replace
✅ PATCH สำหรับ Partial Update
✅ DELETE สำหรับ Remove

Status Codes:
✅ 200/201/204 สำหรับ Success
✅ 400 สำหรับ Validation errors
✅ 401/403 สำหรับ Auth errors
✅ 404 สำหรับ Not found
✅ 409 สำหรับ Conflicts
✅ 500 สำหรับ Server errors

Response Format:
✅ Consistent structure {success, data, pagination, error}
✅ Pagination ในทุก collection
✅ Error details ใน 4xx responses
✅ Request ID สำหรับ tracing

Documentation:
✅ OpenAPI spec
✅ Request/Response examples
✅ Error documentation
```

---

**ต่อไป:** [Part 09 - Service-to-Service Communication →](part-09.md)

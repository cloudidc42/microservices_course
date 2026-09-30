# Part 28: API Design Best Practices & Versioning

## บทนำ

การออกแบบ API ที่ดีเป็นพื้นฐานสำคัญของ Microservices ที่ประสบความสำเร็จ ในบทนี้เราจะเรียนรู้หลักการออกแบบ REST API ระดับโลก รวมถึงการทำ Versioning, OpenAPI Specification, และการรักษา Backward Compatibility

## 1. REST API Design Principles

### 1.1 Resource Naming Conventions

```
# ✅ Good - Nouns, plural, lowercase, hyphenated
GET    /api/v1/user-profiles
GET    /api/v1/order-items/{orderId}/line-items
POST   /api/v1/payment-methods
DELETE /api/v1/shopping-carts/{cartId}/items/{itemId}

# ❌ Bad - Verbs, inconsistent casing
GET    /api/getUsers
POST   /api/CreateOrder
GET    /api/user_Profile
DELETE /api/deleteCart
```

### 1.2 HTTP Methods & Status Codes

```javascript
// controllers/productController.js
const express = require('express');
const router = express.Router();

// GET /products - List with pagination
router.get('/products', async (req, res) => {
  const { page = 1, limit = 20, sort = 'createdAt', order = 'desc', q } = req.query;
  
  try {
    const result = await productService.findAll({ page, limit, sort, order, q });
    
    res.status(200).json({
      data: result.items,
      meta: {
        total: result.total,
        page: parseInt(page),
        limit: parseInt(limit),
        totalPages: Math.ceil(result.total / limit),
      },
      links: {
        self: `/api/v1/products?page=${page}&limit=${limit}`,
        first: `/api/v1/products?page=1&limit=${limit}`,
        last: `/api/v1/products?page=${Math.ceil(result.total / limit)}&limit=${limit}`,
        next: page < Math.ceil(result.total / limit)
          ? `/api/v1/products?page=${parseInt(page) + 1}&limit=${limit}`
          : null,
        prev: page > 1
          ? `/api/v1/products?page=${parseInt(page) - 1}&limit=${limit}`
          : null,
      }
    });
  } catch (error) {
    handleError(res, error);
  }
});

// GET /products/:id - Single resource
router.get('/products/:id', async (req, res) => {
  try {
    const product = await productService.findById(req.params.id);
    if (!product) {
      return res.status(404).json({
        error: {
          code: 'PRODUCT_NOT_FOUND',
          message: `Product with id '${req.params.id}' not found`,
          timestamp: new Date().toISOString(),
          path: req.path,
        }
      });
    }
    res.status(200).json({ data: product });
  } catch (error) {
    handleError(res, error);
  }
});

// POST /products - Create
router.post('/products', authenticate, authorize('products:create'), async (req, res) => {
  try {
    const product = await productService.create(req.body);
    res.status(201)
      .header('Location', `/api/v1/products/${product.id}`)
      .json({ data: product });
  } catch (error) {
    handleError(res, error);
  }
});

// PUT /products/:id - Full Replace
router.put('/products/:id', authenticate, authorize('products:update'), async (req, res) => {
  try {
    const product = await productService.replace(req.params.id, req.body);
    res.status(200).json({ data: product });
  } catch (error) {
    handleError(res, error);
  }
});

// PATCH /products/:id - Partial Update
router.patch('/products/:id', authenticate, authorize('products:update'), async (req, res) => {
  try {
    const product = await productService.update(req.params.id, req.body);
    res.status(200).json({ data: product });
  } catch (error) {
    handleError(res, error);
  }
});

// DELETE /products/:id - Delete
router.delete('/products/:id', authenticate, authorize('products:delete'), async (req, res) => {
  try {
    await productService.delete(req.params.id);
    res.status(204).send(); // No Content
  } catch (error) {
    handleError(res, error);
  }
});

// Standard error handler
function handleError(res, error) {
  console.error(error);
  
  if (error.name === 'ValidationError') {
    return res.status(400).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Request validation failed',
        details: error.details,
        timestamp: new Date().toISOString(),
      }
    });
  }
  
  if (error.name === 'ConflictError') {
    return res.status(409).json({
      error: {
        code: 'CONFLICT',
        message: error.message,
        timestamp: new Date().toISOString(),
      }
    });
  }
  
  res.status(500).json({
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'An unexpected error occurred',
      timestamp: new Date().toISOString(),
    }
  });
}

module.exports = router;
```

### 1.3 HTTP Status Codes Reference

```javascript
// utils/httpStatus.js
const HTTP_STATUS = {
  // 2xx Success
  OK: 200,                    // GET, PUT, PATCH success
  CREATED: 201,               // POST success (resource created)
  ACCEPTED: 202,              // Async operation accepted
  NO_CONTENT: 204,            // DELETE success, no body
  
  // 3xx Redirection
  MOVED_PERMANENTLY: 301,     // Resource moved, update URL
  NOT_MODIFIED: 304,          // Conditional GET, cached version ok
  
  // 4xx Client Errors
  BAD_REQUEST: 400,           // Invalid input/syntax
  UNAUTHORIZED: 401,          // Missing/invalid auth
  FORBIDDEN: 403,             // Authenticated but not authorized
  NOT_FOUND: 404,             // Resource not found
  METHOD_NOT_ALLOWED: 405,    // HTTP method not supported
  CONFLICT: 409,              // Duplicate/version conflict
  GONE: 410,                  // Resource permanently deleted
  UNPROCESSABLE_ENTITY: 422,  // Semantic validation error
  TOO_MANY_REQUESTS: 429,     // Rate limit exceeded
  
  // 5xx Server Errors
  INTERNAL_SERVER_ERROR: 500,
  BAD_GATEWAY: 502,
  SERVICE_UNAVAILABLE: 503,
  GATEWAY_TIMEOUT: 504,
};

module.exports = HTTP_STATUS;
```

## 2. API Versioning Strategies

### 2.1 URL Path Versioning (Recommended for breaking changes)

```javascript
// app.js
const express = require('express');
const app = express();

// Version 1 routes
const v1Routes = require('./routes/v1');
app.use('/api/v1', v1Routes);

// Version 2 routes
const v2Routes = require('./routes/v2');
app.use('/api/v2', v2Routes);

// routes/v1/index.js
const router = require('express').Router();
router.use('/users', require('./users'));
router.use('/products', require('./products'));
module.exports = router;

// routes/v2/index.js - Breaking changes from v1
const router = require('express').Router();
router.use('/users', require('./users'));    // changed response format
router.use('/products', require('./products')); // new fields
router.use('/recommendations', require('./recommendations')); // new endpoint
module.exports = router;
```

### 2.2 Header Versioning

```javascript
// middleware/apiVersion.js
function apiVersionMiddleware(req, res, next) {
  const version = req.headers['api-version'] || 
                  req.headers['accept']?.match(/version=(\d+)/)?.[1] ||
                  '1';
  
  req.apiVersion = parseInt(version);
  res.setHeader('API-Version', req.apiVersion);
  
  if (req.apiVersion < 1 || req.apiVersion > 3) {
    return res.status(400).json({
      error: {
        code: 'INVALID_API_VERSION',
        message: 'API version must be between 1 and 3',
        supportedVersions: [1, 2, 3],
      }
    });
  }
  
  next();
}

// routes/users.js - Version-aware route handler
router.get('/users/:id', apiVersionMiddleware, async (req, res) => {
  const user = await userService.findById(req.params.id);
  
  // Return different shapes based on version
  if (req.apiVersion === 1) {
    return res.json({
      id: user.id,
      name: user.name,
      email: user.email,
    });
  }
  
  if (req.apiVersion >= 2) {
    return res.json({
      id: user.id,
      firstName: user.firstName,
      lastName: user.lastName,
      email: user.email,
      avatar: user.avatarUrl,
      createdAt: user.createdAt,
    });
  }
});
```

### 2.3 Query Parameter Versioning

```javascript
// GET /api/users?version=2
router.get('/users', async (req, res) => {
  const version = parseInt(req.query.version) || 1;
  const users = await userService.findAll();
  
  const serialized = users.map(user => serializeUser(user, version));
  res.json({ data: serialized });
});

function serializeUser(user, version) {
  if (version === 1) {
    return { id: user.id, name: user.name, email: user.email };
  }
  return {
    id: user.id,
    firstName: user.firstName,
    lastName: user.lastName,
    email: user.email,
    profile: user.profile,
    createdAt: user.createdAt,
  };
}
```

## 3. OpenAPI 3.0 Specification

### 3.1 Complete OpenAPI Spec

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: Microservices E-Commerce API
  description: |
    # E-Commerce Microservices API
    
    This API provides endpoints for managing products, orders, and users
    in our e-commerce platform.
    
    ## Authentication
    
    Most endpoints require a JWT Bearer token. Obtain one via `POST /auth/login`.
    
    ## Rate Limiting
    
    - Authenticated: 1000 requests/hour
    - Unauthenticated: 100 requests/hour
    
    Rate limit headers are included in every response:
    - `X-RateLimit-Limit`: Your limit
    - `X-RateLimit-Remaining`: Requests remaining
    - `X-RateLimit-Reset`: Unix timestamp when limit resets
  version: 2.1.0
  contact:
    name: API Support
    email: api-support@example.com
    url: https://docs.example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.example.com/v2
    description: Production
  - url: https://staging-api.example.com/v2
    description: Staging
  - url: http://localhost:3000/api/v2
    description: Development

tags:
  - name: products
    description: Product catalog management
  - name: orders
    description: Order management
  - name: users
    description: User management
  - name: auth
    description: Authentication & authorization

security:
  - BearerAuth: []

paths:
  /products:
    get:
      tags: [products]
      summary: List products
      description: Returns a paginated list of products with optional filtering
      operationId: listProducts
      security: []  # Public endpoint
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
        - $ref: '#/components/parameters/SortParam'
        - name: category
          in: query
          description: Filter by category slug
          schema:
            type: string
            example: electronics
        - name: minPrice
          in: query
          schema:
            type: number
            minimum: 0
        - name: maxPrice
          in: query
          schema:
            type: number
            minimum: 0
        - name: inStock
          in: query
          schema:
            type: boolean
      responses:
        '200':
          description: Successful response
          headers:
            X-Total-Count:
              schema:
                type: integer
              description: Total number of products
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductListResponse'
              examples:
                default:
                  $ref: '#/components/examples/ProductList'
        '400':
          $ref: '#/components/responses/BadRequest'
        '429':
          $ref: '#/components/responses/TooManyRequests'
    
    post:
      tags: [products]
      summary: Create product
      operationId: createProduct
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateProductRequest'
            examples:
              basic:
                $ref: '#/components/examples/CreateProduct'
      responses:
        '201':
          description: Product created
          headers:
            Location:
              schema:
                type: string
              description: URL of the created product
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '400':
          $ref: '#/components/responses/BadRequest'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
        '409':
          $ref: '#/components/responses/Conflict'
  
  /products/{productId}:
    parameters:
      - name: productId
        in: path
        required: true
        schema:
          type: string
          format: uuid
    
    get:
      tags: [products]
      summary: Get product by ID
      operationId: getProduct
      security: []
      responses:
        '200':
          description: Product found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '404':
          $ref: '#/components/responses/NotFound'
    
    patch:
      tags: [products]
      summary: Update product
      operationId: updateProduct
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateProductRequest'
      responses:
        '200':
          description: Product updated
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProductResponse'
        '404':
          $ref: '#/components/responses/NotFound'
    
    delete:
      tags: [products]
      summary: Delete product
      operationId: deleteProduct
      responses:
        '204':
          description: Product deleted
        '404':
          $ref: '#/components/responses/NotFound'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  
  parameters:
    PageParam:
      name: page
      in: query
      description: Page number (1-based)
      schema:
        type: integer
        minimum: 1
        default: 1
    
    LimitParam:
      name: limit
      in: query
      description: Items per page
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
    
    SortParam:
      name: sort
      in: query
      description: Sort field
      schema:
        type: string
        enum: [name, price, createdAt, updatedAt]
        default: createdAt
  
  schemas:
    Product:
      type: object
      required: [id, name, price, sku, category]
      properties:
        id:
          type: string
          format: uuid
          readOnly: true
          example: 550e8400-e29b-41d4-a716-446655440000
        sku:
          type: string
          pattern: '^[A-Z]{2}-[0-9]{6}$'
          example: EL-123456
        name:
          type: string
          minLength: 1
          maxLength: 200
          example: MacBook Pro 14"
        description:
          type: string
          maxLength: 5000
        price:
          type: number
          format: float
          minimum: 0
          example: 59900.00
        compareAtPrice:
          type: number
          format: float
          minimum: 0
          nullable: true
          description: Original price for showing discounts
        currency:
          type: string
          default: THB
          enum: [THB, USD, EUR]
        category:
          $ref: '#/components/schemas/Category'
        images:
          type: array
          items:
            $ref: '#/components/schemas/ProductImage'
        inventory:
          $ref: '#/components/schemas/Inventory'
        attributes:
          type: object
          additionalProperties:
            type: string
          example:
            color: Space Gray
            storage: 512GB
        tags:
          type: array
          items:
            type: string
          example: [laptop, apple, m3]
        isActive:
          type: boolean
          default: true
        createdAt:
          type: string
          format: date-time
          readOnly: true
        updatedAt:
          type: string
          format: date-time
          readOnly: true
    
    CreateProductRequest:
      type: object
      required: [sku, name, price, categoryId]
      properties:
        sku:
          type: string
          pattern: '^[A-Z]{2}-[0-9]{6}$'
        name:
          type: string
          minLength: 1
          maxLength: 200
        description:
          type: string
          maxLength: 5000
        price:
          type: number
          minimum: 0
        compareAtPrice:
          type: number
          minimum: 0
          nullable: true
        categoryId:
          type: string
          format: uuid
        inventory:
          type: object
          properties:
            quantity:
              type: integer
              minimum: 0
        attributes:
          type: object
          additionalProperties:
            type: string
    
    UpdateProductRequest:
      type: object
      minProperties: 1
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 200
        description:
          type: string
        price:
          type: number
          minimum: 0
        isActive:
          type: boolean
    
    ProductResponse:
      type: object
      properties:
        data:
          $ref: '#/components/schemas/Product'
    
    ProductListResponse:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/Product'
        meta:
          $ref: '#/components/schemas/PaginationMeta'
        links:
          $ref: '#/components/schemas/PaginationLinks'
    
    Category:
      type: object
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        slug:
          type: string
    
    ProductImage:
      type: object
      properties:
        id:
          type: string
          format: uuid
        url:
          type: string
          format: uri
        alt:
          type: string
        position:
          type: integer
          minimum: 0
    
    Inventory:
      type: object
      properties:
        quantity:
          type: integer
          minimum: 0
        reserved:
          type: integer
          minimum: 0
        available:
          type: integer
          minimum: 0
          readOnly: true
    
    PaginationMeta:
      type: object
      properties:
        total:
          type: integer
        page:
          type: integer
        limit:
          type: integer
        totalPages:
          type: integer
    
    PaginationLinks:
      type: object
      properties:
        self:
          type: string
        first:
          type: string
        last:
          type: string
        next:
          type: string
          nullable: true
        prev:
          type: string
          nullable: true
    
    Error:
      type: object
      required: [error]
      properties:
        error:
          type: object
          required: [code, message]
          properties:
            code:
              type: string
              example: VALIDATION_ERROR
            message:
              type: string
              example: Request validation failed
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
            path:
              type: string
            requestId:
              type: string
  
  responses:
    BadRequest:
      description: Invalid request
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    Unauthorized:
      description: Authentication required
      headers:
        WWW-Authenticate:
          schema:
            type: string
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    Forbidden:
      description: Insufficient permissions
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
    
    NotFound:
      description: Resource not found
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
    
    TooManyRequests:
      description: Rate limit exceeded
      headers:
        Retry-After:
          schema:
            type: integer
          description: Seconds until rate limit resets
        X-RateLimit-Limit:
          schema:
            type: integer
        X-RateLimit-Remaining:
          schema:
            type: integer
        X-RateLimit-Reset:
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
  
  examples:
    ProductList:
      summary: List of products
      value:
        data:
          - id: 550e8400-e29b-41d4-a716-446655440000
            sku: EL-123456
            name: MacBook Pro 14"
            price: 59900.00
            category:
              id: cat-001
              name: Laptops
              slug: laptops
        meta:
          total: 150
          page: 1
          limit: 20
          totalPages: 8
        links:
          self: /api/v2/products?page=1&limit=20
          first: /api/v2/products?page=1&limit=20
          last: /api/v2/products?page=8&limit=20
          next: /api/v2/products?page=2&limit=20
          prev: null
    
    CreateProduct:
      summary: Create a product
      value:
        sku: EL-123456
        name: MacBook Pro 14"
        price: 59900.00
        categoryId: 550e8400-e29b-41d4-a716-000000000001
        inventory:
          quantity: 50
```

## 4. Request/Response Design Patterns

### 4.1 Envelope Pattern

```javascript
// utils/responseBuilder.js
class ResponseBuilder {
  static success(data, meta = {}) {
    return { data, meta };
  }
  
  static paginated(items, total, page, limit) {
    const totalPages = Math.ceil(total / limit);
    return {
      data: items,
      meta: {
        total,
        page,
        limit,
        totalPages,
        hasNext: page < totalPages,
        hasPrev: page > 1,
      }
    };
  }
  
  static error(code, message, details = null, statusCode = 500) {
    const body = {
      error: {
        code,
        message,
        timestamp: new Date().toISOString(),
      }
    };
    if (details) body.error.details = details;
    return { statusCode, body };
  }
  
  static created(data, location) {
    return { data, links: { self: location } };
  }
  
  static accepted(message, jobId, statusUrl) {
    return {
      message,
      job: {
        id: jobId,
        statusUrl,
        estimatedCompletionTime: new Date(Date.now() + 30000).toISOString(),
      }
    };
  }
}

module.exports = ResponseBuilder;
```

### 4.2 HATEOAS (Hypermedia As The Engine Of Application State)

```javascript
// utils/hypermedia.js
class HypermediaBuilder {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
  }
  
  buildOrderLinks(order, userRole) {
    const links = {
      self: { href: `${this.baseUrl}/orders/${order.id}`, method: 'GET' },
      collection: { href: `${this.baseUrl}/orders`, method: 'GET' },
    };
    
    // Add state-appropriate actions
    if (order.status === 'DRAFT') {
      links.submit = {
        href: `${this.baseUrl}/orders/${order.id}/submit`,
        method: 'POST',
        title: 'Submit order for processing',
      };
      links.addItem = {
        href: `${this.baseUrl}/orders/${order.id}/items`,
        method: 'POST',
        title: 'Add item to order',
      };
      links.delete = {
        href: `${this.baseUrl}/orders/${order.id}`,
        method: 'DELETE',
        title: 'Delete draft order',
      };
    }
    
    if (order.status === 'CONFIRMED') {
      links.cancel = {
        href: `${this.baseUrl}/orders/${order.id}/cancel`,
        method: 'POST',
        title: 'Cancel order',
      };
      links.payment = {
        href: `${this.baseUrl}/orders/${order.id}/payment`,
        method: 'GET',
        title: 'View payment details',
      };
    }
    
    if (order.status === 'SHIPPED') {
      links.track = {
        href: `${this.baseUrl}/orders/${order.id}/tracking`,
        method: 'GET',
        title: 'Track shipment',
      };
    }
    
    if (['DELIVERED', 'SHIPPED'].includes(order.status) && !order.reviewed) {
      links.review = {
        href: `${this.baseUrl}/orders/${order.id}/review`,
        method: 'POST',
        title: 'Leave a review',
      };
    }
    
    // Admin-only links
    if (userRole === 'ADMIN') {
      links.refund = {
        href: `${this.baseUrl}/orders/${order.id}/refund`,
        method: 'POST',
        title: 'Process refund',
      };
    }
    
    return links;
  }
}

// Usage in route handler
router.get('/orders/:id', authenticate, async (req, res) => {
  const order = await orderService.findById(req.params.id);
  const hypermedia = new HypermediaBuilder('https://api.example.com/v2');
  
  res.json({
    data: order,
    _links: hypermedia.buildOrderLinks(order, req.user.role),
  });
});
```

## 5. API Deprecation Strategy

### 5.1 Sunset Headers and Deprecation Notices

```javascript
// middleware/deprecation.js
const DEPRECATED_ENDPOINTS = new Map([
  ['/api/v1/users', {
    deprecatedAt: '2024-01-01',
    sunsetAt: '2024-07-01',
    successor: '/api/v2/users',
    reason: 'Response format changed to include firstName/lastName instead of name',
  }],
  ['/api/v1/products/search', {
    deprecatedAt: '2024-03-01',
    sunsetAt: '2024-09-01',
    successor: '/api/v2/products?q=',
    reason: 'Merged into products list endpoint with q parameter',
  }],
]);

function deprecationMiddleware(req, res, next) {
  const endpoint = `${req.baseUrl}${req.path}`;
  const deprecation = DEPRECATED_ENDPOINTS.get(endpoint);
  
  if (!deprecation) return next();
  
  const sunsetDate = new Date(deprecation.sunsetAt);
  
  // After sunset date, block the endpoint
  if (new Date() > sunsetDate) {
    return res.status(410).json({
      error: {
        code: 'ENDPOINT_GONE',
        message: `This endpoint was sunset on ${deprecation.sunsetAt}`,
        successor: deprecation.successor,
        migrationGuide: `https://docs.example.com/migration/${deprecation.sunsetAt}`,
      }
    });
  }
  
  // Before sunset, add deprecation headers
  res.setHeader('Deprecation', `date="${deprecation.deprecatedAt}"`);
  res.setHeader('Sunset', sunsetDate.toUTCString());
  res.setHeader('Link', `<${deprecation.successor}>; rel="successor-version"`);
  res.setHeader('Warning', `299 - "Deprecated API: ${deprecation.reason}. Use ${deprecation.successor}"`);
  
  // Log deprecation usage for monitoring
  console.warn({
    type: 'DEPRECATED_API_USAGE',
    endpoint,
    user: req.user?.id,
    sunsetAt: deprecation.sunsetAt,
    daysUntilSunset: Math.floor((sunsetDate - new Date()) / (1000 * 60 * 60 * 24)),
  });
  
  next();
}

module.exports = deprecationMiddleware;
```

### 5.2 Version Lifecycle Manager

```javascript
// services/versionLifecycle.js
class VersionLifecycleManager {
  constructor() {
    this.versions = new Map([
      ['v1', {
        status: 'deprecated',
        deprecatedAt: '2024-01-01',
        sunsetAt: '2024-12-31',
        features: ['basic-crud', 'auth'],
      }],
      ['v2', {
        status: 'stable',
        releasedAt: '2024-01-01',
        features: ['basic-crud', 'auth', 'filtering', 'sorting', 'hateoas'],
      }],
      ['v3', {
        status: 'preview',
        previewSince: '2024-06-01',
        features: ['basic-crud', 'auth', 'filtering', 'sorting', 'hateoas', 'graphql', 'webhooks'],
      }],
    ]);
  }
  
  getVersionInfo(version) {
    return this.versions.get(version);
  }
  
  isSupported(version) {
    const info = this.versions.get(version);
    if (!info) return false;
    if (info.status === 'sunset') return false;
    return true;
  }
  
  getDaysUntilSunset(version) {
    const info = this.versions.get(version);
    if (!info?.sunsetAt) return null;
    const diff = new Date(info.sunsetAt) - new Date();
    return Math.floor(diff / (1000 * 60 * 60 * 24));
  }
  
  addVersionHeaders(res, version) {
    const info = this.versions.get(version);
    if (!info) return;
    
    res.setHeader('API-Version', version);
    res.setHeader('API-Status', info.status);
    
    if (info.status === 'deprecated') {
      res.setHeader('Deprecation', `date="${info.deprecatedAt}"`);
      res.setHeader('Sunset', new Date(info.sunsetAt).toUTCString());
    }
    
    if (info.status === 'preview') {
      res.setHeader('Warning', '199 - "Preview API: Subject to breaking changes"');
    }
  }
}

module.exports = new VersionLifecycleManager();
```

## 6. Backward Compatibility Rules

### 6.1 Non-Breaking Changes (Safe)

```javascript
// ✅ Adding new optional fields to response
// Before v1
{ id: '123', name: 'John' }

// After v1 (backward compatible - client ignores unknown fields)
{ id: '123', name: 'John', avatar: 'https://...', createdAt: '...' }

// ✅ Adding new optional request parameters
// Before
GET /products

// After (still accepts requests without category)
GET /products?category=electronics

// ✅ Adding new endpoints
// Before: GET /orders
// After: GET /orders + POST /orders/bulk (new)

// ✅ Making required field optional
// Before: { name: required }
// After: { name: optional } - existing clients still send name, no break

// ✅ Expanding enum values
// Before: status: ['ACTIVE', 'INACTIVE']
// After: status: ['ACTIVE', 'INACTIVE', 'SUSPENDED'] - old clients get new values they ignore
```

### 6.2 Breaking Changes (Require Version Bump)

```javascript
// ❌ Removing fields from response
// Before
{ id: '123', name: 'John', legacyField: '...' }
// After - BREAKING: client code that reads legacyField will fail
{ id: '123', name: 'John' }

// ❌ Renaming fields
// Before
{ userId: '123', userName: 'John' }
// After - BREAKING: client reads userId, gets undefined
{ id: '123', name: 'John' }

// ❌ Changing field types
// Before: { count: 5 }  (number)
// After: { count: '5' }  (string) - BREAKING

// ❌ Adding required request fields
// Before: POST /orders { items: [...] }
// After: POST /orders { items: [...], shippingAddress: required } - BREAKING

// ❌ Changing URL structure
// Before: GET /users/:userId/orders
// After: GET /orders?userId=:userId - BREAKING
```

### 6.3 API Compatibility Testing

```javascript
// tests/apiCompatibility.test.js
const { expect } = require('chai');
const v1Schema = require('./schemas/v1');
const v2Schema = require('./schemas/v2');

describe('API Backward Compatibility', () => {
  describe('v2 is backward compatible with v1', () => {
    it('v2 response includes all v1 fields', async () => {
      const v1Fields = Object.keys(v1Schema.ProductResponse.properties);
      const v2Fields = Object.keys(v2Schema.ProductResponse.properties);
      
      v1Fields.forEach(field => {
        expect(v2Fields, `v2 should include v1 field: ${field}`).to.include(field);
      });
    });
    
    it('v2 does not change field types from v1', () => {
      const v1Fields = v1Schema.ProductResponse.properties;
      const v2Fields = v2Schema.ProductResponse.properties;
      
      Object.entries(v1Fields).forEach(([field, schema]) => {
        if (v2Fields[field]) {
          expect(v2Fields[field].type, `Field type mismatch: ${field}`).to.equal(schema.type);
        }
      });
    });
    
    it('v2 does not add required fields to request body', () => {
      const v1Required = v1Schema.CreateProductRequest.required || [];
      const v2Required = v2Schema.CreateProductRequest.required || [];
      
      v2Required.forEach(field => {
        expect(v1Required, `v2 added new required field: ${field}`).to.include(field);
      });
    });
  });
});
```

## 7. SDK Generation from OpenAPI

### 7.1 Auto-generating TypeScript Client

```bash
# Install OpenAPI generator
npm install -g @openapitools/openapi-generator-cli

# Generate TypeScript client
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./sdk/typescript \
  --additional-properties=supportsES6=true,npmName=@example/api-client,npmVersion=1.0.0

# Generate Python client
openapi-generator-cli generate \
  -i openapi.yaml \
  -g python \
  -o ./sdk/python \
  --additional-properties=packageName=example_api_client
```

### 7.2 Custom SDK Wrapper

```typescript
// sdk/typescript/src/ExampleApiClient.ts
import { ProductsApi, OrdersApi, Configuration } from './generated';
import axios, { AxiosInstance } from 'axios';

interface ClientOptions {
  baseURL?: string;
  apiKey?: string;
  timeout?: number;
  retries?: number;
}

export class ExampleApiClient {
  private axiosInstance: AxiosInstance;
  products: ProductsApi;
  orders: OrdersApi;
  
  constructor(options: ClientOptions = {}) {
    this.axiosInstance = axios.create({
      baseURL: options.baseURL || 'https://api.example.com/v2',
      timeout: options.timeout || 10000,
      headers: {
        'Content-Type': 'application/json',
        ...(options.apiKey && { Authorization: `Bearer ${options.apiKey}` }),
      },
    });
    
    // Add retry logic
    if (options.retries) {
      this.addRetryInterceptor(options.retries);
    }
    
    const config = new Configuration({
      basePath: options.baseURL || 'https://api.example.com/v2',
    });
    
    this.products = new ProductsApi(config, undefined, this.axiosInstance);
    this.orders = new OrdersApi(config, undefined, this.axiosInstance);
  }
  
  private addRetryInterceptor(retries: number): void {
    this.axiosInstance.interceptors.response.use(
      response => response,
      async error => {
        const config = error.config;
        config._retryCount = config._retryCount || 0;
        
        if (config._retryCount >= retries) {
          return Promise.reject(error);
        }
        
        // Only retry on network errors or 5xx
        if (!error.response || error.response.status >= 500) {
          config._retryCount += 1;
          const delay = Math.pow(2, config._retryCount) * 1000;
          await new Promise(resolve => setTimeout(resolve, delay));
          return this.axiosInstance(config);
        }
        
        return Promise.reject(error);
      }
    );
  }
  
  setAuthToken(token: string): void {
    this.axiosInstance.defaults.headers.common.Authorization = `Bearer ${token}`;
  }
}

// Usage example
const client = new ExampleApiClient({
  apiKey: 'your-jwt-token',
  retries: 3,
});

const products = await client.products.listProducts({ page: 1, limit: 10 });
console.log(products.data);
```

## 8. API Changelog Management

### 8.1 CHANGELOG.md Format

```markdown
# API Changelog

All notable changes to the API will be documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]
### Added
- `GET /products/:id/recommendations` - AI-powered product recommendations

## [2.1.0] - 2024-06-01
### Added
- `POST /orders/bulk` - Create multiple orders in one request
- `GET /users/:id/preferences` - Get user preferences
- `PATCH /users/:id/preferences` - Update user preferences
- Filtering on `GET /products`: `minRating`, `maxRating` parameters

### Changed
- `GET /products` now supports cursor-based pagination via `cursor` parameter
  in addition to offset-based pagination. Cursor pagination recommended for
  real-time data.

### Deprecated
- `POST /products/search` - Use `GET /products?q=` instead.
  Will be removed in v3.0.0 (sunset: 2025-01-01).

## [2.0.0] - 2024-01-01
### Added
- Full HATEOAS support via `_links` in all responses
- Rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`
- Request ID via `X-Request-ID` header for tracing
- Webhook support via `/webhooks` endpoints

### Changed
- **BREAKING**: `GET /users/:id` response: `name` split into `firstName` + `lastName`
- **BREAKING**: `GET /products` pagination: `offset`/`count` → `page`/`limit`
- Error response shape: `{ error }` wrapper (was flat object)

### Removed
- **BREAKING**: `GET /v1/users/search` removed (replaced by `GET /v2/users?q=`)
- Removed `X-API-Key` header auth (use Bearer token)

## [1.2.0] - 2023-10-01
### Added
- `DELETE /orders/:id` - Cancel orders in DRAFT status
- Sorting support: `?sort=price&order=asc`
```

### 8.2 Automated Changelog from Git

```bash
#!/bin/bash
# scripts/generate-changelog.sh

VERSION=$1
PREV_TAG=$(git describe --abbrev=0 --tags HEAD^)
CURRENT_DATE=$(date +%Y-%m-%d)

echo "## [$VERSION] - $CURRENT_DATE"
echo ""

# Added features
echo "### Added"
git log $PREV_TAG..HEAD --grep="feat:" --pretty=format:"- %s" | sed 's/feat: //'
echo ""

# Breaking changes
echo "### Changed (Breaking)"
git log $PREV_TAG..HEAD --grep="BREAKING CHANGE" --pretty=format:"- **BREAKING**: %b"
echo ""

# Non-breaking changes
echo "### Changed"
git log $PREV_TAG..HEAD --grep="refactor:\|perf:" --pretty=format:"- %s"
echo ""

# Fixes
echo "### Fixed"
git log $PREV_TAG..HEAD --grep="fix:" --pretty=format:"- %s" | sed 's/fix: //'
echo ""

# Deprecated
echo "### Deprecated"
git log $PREV_TAG..HEAD --grep="deprecate:" --pretty=format:"- %s"
```

## 9. API Documentation Portal

### 9.1 Swagger UI Setup

```javascript
// server.js
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');
const path = require('path');

const app = express();
const swaggerDocument = YAML.load(path.join(__dirname, 'openapi.yaml'));

// Customize Swagger UI
const swaggerOptions = {
  customCss: `
    .swagger-ui .topbar { display: none }
    .swagger-ui .info .title { color: #1a1a2e }
  `,
  customSiteTitle: 'E-Commerce API Docs',
  customfavIcon: '/favicon.ico',
  swaggerOptions: {
    docExpansion: 'none',
    filter: true,
    showExtensions: true,
    tryItOutEnabled: true,
    persistAuthorization: true,
    displayRequestDuration: true,
  },
};

app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument, swaggerOptions));

// Serve raw OpenAPI spec
app.get('/openapi.json', (req, res) => res.json(swaggerDocument));
app.get('/openapi.yaml', (req, res) => {
  res.type('text/yaml');
  res.sendFile(path.join(__dirname, 'openapi.yaml'));
});
```

### 9.2 Redoc (Alternative Documentation)

```html
<!-- public/api-docs.html -->
<!DOCTYPE html>
<html>
<head>
  <title>API Documentation</title>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link href="https://fonts.googleapis.com/css?family=Montserrat:300,400,700|Roboto:300,400,700" rel="stylesheet">
  <style>
    body { margin: 0; padding: 0; }
  </style>
</head>
<body>
  <redoc spec-url='/openapi.yaml'
         expand-responses="200,201"
         hide-download-button="false"
         scroll-y-offset="0"
         suppress-warnings="false">
  </redoc>
  <script src="https://cdn.jsdelivr.net/npm/redoc@latest/bundles/redoc.standalone.js"></script>
</body>
</html>
```

## 10. Request Validation Middleware

```javascript
// middleware/validateRequest.js
const Ajv = require('ajv');
const addFormats = require('ajv-formats');
const YAML = require('yamljs');
const path = require('path');

const ajv = new Ajv({ allErrors: true, coerceTypes: true });
addFormats(ajv);

const openApiSpec = YAML.load(path.join(__dirname, '../openapi.yaml'));

// Extract and compile schemas from OpenAPI spec
const schemas = {};
Object.entries(openApiSpec.components.schemas).forEach(([name, schema]) => {
  // Replace $ref with actual schemas
  schemas[name] = resolveRefs(schema, openApiSpec.components.schemas);
  ajv.addSchema(schemas[name], `#/components/schemas/${name}`);
});

function validateRequestBody(schemaName) {
  const validate = ajv.compile(schemas[schemaName]);
  
  return (req, res, next) => {
    const valid = validate(req.body);
    if (!valid) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Request body validation failed',
          details: validate.errors.map(err => ({
            field: err.instancePath.replace('/', '') || err.params.missingProperty,
            message: err.message,
            value: err.data,
          })),
          timestamp: new Date().toISOString(),
        }
      });
    }
    next();
  };
}

function validateQueryParams(rules) {
  return (req, res, next) => {
    const errors = [];
    
    Object.entries(rules).forEach(([param, rule]) => {
      const value = req.query[param];
      
      if (rule.required && !value) {
        errors.push({ field: param, message: 'Required query parameter missing' });
        return;
      }
      
      if (value && rule.type === 'integer') {
        const num = parseInt(value);
        if (isNaN(num)) {
          errors.push({ field: param, message: 'Must be an integer' });
        } else {
          if (rule.minimum !== undefined && num < rule.minimum) {
            errors.push({ field: param, message: `Must be >= ${rule.minimum}` });
          }
          if (rule.maximum !== undefined && num > rule.maximum) {
            errors.push({ field: param, message: `Must be <= ${rule.maximum}` });
          }
          req.query[param] = num;
        }
      }
      
      if (value && rule.enum && !rule.enum.includes(value)) {
        errors.push({
          field: param,
          message: `Must be one of: ${rule.enum.join(', ')}`,
        });
      }
    });
    
    if (errors.length > 0) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Query parameter validation failed',
          details: errors,
        }
      });
    }
    
    next();
  };
}

// Usage
router.get('/products', 
  validateQueryParams({
    page: { type: 'integer', minimum: 1, default: 1 },
    limit: { type: 'integer', minimum: 1, maximum: 100, default: 20 },
    sort: { enum: ['name', 'price', 'createdAt', 'updatedAt'] },
  }),
  async (req, res) => { /* handler */ }
);

router.post('/products',
  authenticate,
  validateRequestBody('CreateProductRequest'),
  async (req, res) => { /* handler */ }
);

module.exports = { validateRequestBody, validateQueryParams };
```

## 11. Rate Limiting Strategy

```javascript
// middleware/rateLimiter.js
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

class RateLimiter {
  constructor(options = {}) {
    this.defaultConfig = {
      windowMs: 60 * 60 * 1000, // 1 hour
      max: 1000,
      keyPrefix: 'rl:',
    };
  }
  
  getConfig(req) {
    if (!req.user) {
      return { windowMs: 60 * 60 * 1000, max: 100 }; // Unauthenticated
    }
    
    const tierLimits = {
      FREE: { windowMs: 60 * 60 * 1000, max: 500 },
      PRO: { windowMs: 60 * 60 * 1000, max: 5000 },
      ENTERPRISE: { windowMs: 60 * 60 * 1000, max: 50000 },
    };
    
    return tierLimits[req.user.tier] || tierLimits.FREE;
  }
  
  middleware() {
    return async (req, res, next) => {
      const config = this.getConfig(req);
      const key = `${this.defaultConfig.keyPrefix}${req.user?.id || req.ip}`;
      const windowStart = Math.floor(Date.now() / config.windowMs);
      const windowKey = `${key}:${windowStart}`;
      
      const pipeline = redis.pipeline();
      pipeline.incr(windowKey);
      pipeline.pttl(windowKey);
      
      const [[, count], [, pttl]] = await pipeline.exec();
      
      if (pttl === -1) {
        await redis.pexpire(windowKey, config.windowMs);
      }
      
      const remaining = Math.max(0, config.max - count);
      const resetTime = Math.floor((Date.now() + (pttl > 0 ? pttl : config.windowMs)) / 1000);
      
      res.setHeader('X-RateLimit-Limit', config.max);
      res.setHeader('X-RateLimit-Remaining', remaining);
      res.setHeader('X-RateLimit-Reset', resetTime);
      
      if (count > config.max) {
        const retryAfter = Math.ceil((pttl > 0 ? pttl : config.windowMs) / 1000);
        res.setHeader('Retry-After', retryAfter);
        
        return res.status(429).json({
          error: {
            code: 'RATE_LIMIT_EXCEEDED',
            message: `Too many requests. Retry after ${retryAfter} seconds.`,
            limit: config.max,
            remaining: 0,
            resetAt: new Date(resetTime * 1000).toISOString(),
          }
        });
      }
      
      next();
    };
  }
}

module.exports = new RateLimiter();
```

## 12. API Health & Readiness

```javascript
// routes/health.js
const router = require('express').Router();
const db = require('../db');
const redis = require('../redis');

// Liveness probe - Is the process alive?
router.get('/health/live', (req, res) => {
  res.status(200).json({
    status: 'alive',
    version: process.env.APP_VERSION || '1.0.0',
    uptime: process.uptime(),
    timestamp: new Date().toISOString(),
  });
});

// Readiness probe - Is the app ready to serve traffic?
router.get('/health/ready', async (req, res) => {
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkDependencies(),
  ]);
  
  const results = {
    database: checks[0].status === 'fulfilled' ? checks[0].value : { ok: false, error: checks[0].reason?.message },
    cache: checks[1].status === 'fulfilled' ? checks[1].value : { ok: false, error: checks[1].reason?.message },
    dependencies: checks[2].status === 'fulfilled' ? checks[2].value : { ok: false, error: checks[2].reason?.message },
  };
  
  const allHealthy = Object.values(results).every(r => r.ok);
  
  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ready' : 'not_ready',
    checks: results,
    timestamp: new Date().toISOString(),
  });
});

async function checkDatabase() {
  const start = Date.now();
  await db.query('SELECT 1');
  return { ok: true, latency: Date.now() - start };
}

async function checkRedis() {
  const start = Date.now();
  await redis.ping();
  return { ok: true, latency: Date.now() - start };
}

async function checkDependencies() {
  // Check external service health
  const checks = await Promise.allSettled([
    axios.get(`${process.env.PAYMENT_SERVICE_URL}/health/live`, { timeout: 3000 }),
    axios.get(`${process.env.INVENTORY_SERVICE_URL}/health/live`, { timeout: 3000 }),
  ]);
  
  return {
    ok: checks.every(c => c.status === 'fulfilled'),
    services: {
      paymentService: checks[0].status === 'fulfilled' ? 'up' : 'down',
      inventoryService: checks[1].status === 'fulfilled' ? 'up' : 'down',
    }
  };
}

module.exports = router;
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **REST API Design** - Naming conventions, HTTP methods, status codes ที่ถูกต้อง
2. **API Versioning** - URL path, header, query parameter versioning strategies
3. **OpenAPI 3.0** - เขียน spec ที่สมบูรณ์ครอบคลุม schemas, responses, examples
4. **Response Patterns** - Envelope pattern, HATEOAS links, pagination metadata
5. **Deprecation Strategy** - Sunset headers, version lifecycle management
6. **Backward Compatibility** - Rules สำหรับ breaking/non-breaking changes
7. **SDK Generation** - Auto-generate TypeScript/Python clients จาก OpenAPI spec
8. **Changelog Management** - CHANGELOG.md format และ automated generation

## แบบฝึกหัด

1. สร้าง OpenAPI spec สำหรับ Blog API ครอบคลุม posts, comments, tags
2. Implement API versioning ใน Express.js ที่รองรับ v1 และ v2
3. เพิ่ม HATEOAS links ใน Order API ตาม state machine
4. สร้าง deprecation middleware และทดสอบ sunset header
5. Generate TypeScript SDK จาก OpenAPI spec และทดสอบ type safety

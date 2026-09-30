# Part 42: GraphQL Federation ขั้นสูง

## บทนำ

GraphQL Federation ช่วยให้เราสามารถแบ่ง GraphQL Schema ออกเป็น Subgraphs หลายๆ อัน แต่ละ Subgraph ดูแล Domain ของตัวเอง และ Apollo Router จะรวม Schema ทั้งหมดเข้าด้วยกันเป็น Supergraph เดียว ทำให้ Client สามารถ Query ข้ามหลาย Service ได้อย่าง Transparent

### สิ่งที่จะได้เรียนรู้

- Apollo Federation 2.0 Architecture
- Subgraph Services ด้วย Apollo Server
- Apollo Router Configuration
- Federation Directives: @key, @external, @requires, @provides, @shareable
- Subscriptions ผ่าน WebSocket
- DataLoader สำหรับแก้ปัญหา N+1
- Authorization ที่ Gateway
- Error Handling และ Monitoring

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Apollo Router                               │
│                    (Supergraph / Gateway)                            │
│                                                                      │
│  schema.graphql (Composed from all subgraphs)                       │
│  Authorization Middleware                                           │
│  Rate Limiting                                                      │
└────────────────────────────────────────────────────────────────────┘
           │                │                 │               │
           ▼                ▼                 ▼               ▼
    ┌──────────┐    ┌──────────────┐  ┌────────────┐  ┌──────────────┐
    │  Users   │    │   Products   │  │   Orders   │  │   Reviews    │
    │ Subgraph │    │  Subgraph    │  │  Subgraph  │  │  Subgraph   │
    └──────────┘    └──────────────┘  └────────────┘  └──────────────┘
         │                │                 │               │
         ▼                ▼                 ▼               ▼
    ┌──────────┐    ┌──────────────┐  ┌────────────┐  ┌──────────────┐
    │ Users DB │    │  Products DB │  │  Orders DB │  │  Reviews DB  │
    └──────────┘    └──────────────┘  └────────────┘  └──────────────┘
```

---

## 2. Subgraph: Users Service

```typescript
// users-service/src/schema.ts
import { buildSubgraphSchema } from '@apollo/subgraph';
import { gql } from 'graphql-tag';
import { UserResolver } from './resolvers/user.resolver';

export const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key", "@shareable"])

  type User @key(fields: "id") {
    id: ID!
    email: String!
    name: String!
    role: UserRole!
    createdAt: String!
    profile: UserProfile
  }

  type UserProfile @shareable {
    avatar: String
    bio: String
    phoneNumber: String
  }

  enum UserRole {
    ADMIN
    CUSTOMER
    VENDOR
  }

  type Query {
    me: User
    user(id: ID!): User
    users(page: Int, limit: Int): UserConnection!
  }

  type UserConnection {
    edges: [UserEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type UserEdge {
    node: User!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  type Mutation {
    createUser(input: CreateUserInput!): CreateUserPayload!
    updateUser(id: ID!, input: UpdateUserInput!): UpdateUserPayload!
    deleteUser(id: ID!): DeleteUserPayload!
  }

  input CreateUserInput {
    email: String!
    name: String!
    password: String!
    role: UserRole = CUSTOMER
  }

  input UpdateUserInput {
    name: String
    profile: UpdateProfileInput
  }

  input UpdateProfileInput {
    avatar: String
    bio: String
    phoneNumber: String
  }

  type CreateUserPayload {
    user: User
    errors: [UserError!]
  }

  type UpdateUserPayload {
    user: User
    errors: [UserError!]
  }

  type DeleteUserPayload {
    success: Boolean!
    errors: [UserError!]
  }

  type UserError {
    field: String
    message: String!
    code: String!
  }
`;

export function createSubgraphSchema() {
  return buildSubgraphSchema({
    typeDefs,
    resolvers: UserResolver,
  });
}
```

```typescript
// users-service/src/resolvers/user.resolver.ts
import { Pool } from 'pg';
import bcrypt from 'bcrypt';
import { UserDataLoader } from '../dataloaders/user.dataloader';

export const UserResolver = {
  Query: {
    me: async (_: any, __: any, context: any) => {
      if (!context.user) {
        throw new Error('Not authenticated');
      }
      return context.dataloaders.user.load(context.user.id);
    },

    user: async (_: any, { id }: { id: string }, context: any) => {
      if (!context.user || context.user.role !== 'ADMIN') {
        throw new Error('Not authorized');
      }
      return context.dataloaders.user.load(id);
    },

    users: async (
      _: any,
      { page = 1, limit = 20 }: { page: number; limit: number },
      context: any
    ) => {
      const pool: Pool = context.pool;
      const offset = (page - 1) * limit;

      const [countResult, usersResult] = await Promise.all([
        pool.query('SELECT COUNT(*) FROM users WHERE deleted_at IS NULL'),
        pool.query(
          `SELECT * FROM users WHERE deleted_at IS NULL 
           ORDER BY created_at DESC LIMIT $1 OFFSET $2`,
          [limit, offset]
        ),
      ]);

      const totalCount = parseInt(countResult.rows[0].count);
      const users = usersResult.rows;

      return {
        edges: users.map((user, index) => ({
          node: mapUserRow(user),
          cursor: Buffer.from(`${offset + index + 1}`).toString('base64'),
        })),
        pageInfo: {
          hasNextPage: offset + limit < totalCount,
          hasPreviousPage: page > 1,
          startCursor:
            users.length > 0
              ? Buffer.from(`${offset + 1}`).toString('base64')
              : null,
          endCursor:
            users.length > 0
              ? Buffer.from(`${offset + users.length}`).toString('base64')
              : null,
        },
        totalCount,
      };
    },
  },

  Mutation: {
    createUser: async (
      _: any,
      { input }: { input: any },
      context: any
    ) => {
      const pool: Pool = context.pool;
      const errors: any[] = [];

      // Validate email uniqueness
      const existing = await pool.query(
        'SELECT id FROM users WHERE email = $1',
        [input.email.toLowerCase()]
      );

      if (existing.rows.length > 0) {
        errors.push({
          field: 'email',
          message: 'Email already in use',
          code: 'EMAIL_TAKEN',
        });
        return { user: null, errors };
      }

      const hashedPassword = await bcrypt.hash(input.password, 12);

      const result = await pool.query(
        `INSERT INTO users (email, name, password_hash, role)
         VALUES ($1, $2, $3, $4)
         RETURNING *`,
        [
          input.email.toLowerCase(),
          input.name,
          hashedPassword,
          input.role || 'CUSTOMER',
        ]
      );

      return {
        user: mapUserRow(result.rows[0]),
        errors: [],
      };
    },

    updateUser: async (
      _: any,
      { id, input }: { id: string; input: any },
      context: any
    ) => {
      const pool: Pool = context.pool;

      if (!context.user || (context.user.id !== id && context.user.role !== 'ADMIN')) {
        throw new Error('Not authorized');
      }

      const result = await pool.query(
        `UPDATE users SET name = COALESCE($2, name), updated_at = NOW()
         WHERE id = $1 RETURNING *`,
        [id, input.name]
      );

      if (result.rows.length === 0) {
        return {
          user: null,
          errors: [{ message: 'User not found', code: 'NOT_FOUND' }],
        };
      }

      return { user: mapUserRow(result.rows[0]), errors: [] };
    },

    deleteUser: async (
      _: any,
      { id }: { id: string },
      context: any
    ) => {
      if (!context.user || context.user.role !== 'ADMIN') {
        throw new Error('Not authorized');
      }

      const pool: Pool = context.pool;
      const result = await pool.query(
        'UPDATE users SET deleted_at = NOW() WHERE id = $1',
        [id]
      );

      return {
        success: result.rowCount! > 0,
        errors: [],
      };
    },
  },

  // Federation reference resolver
  User: {
    __resolveReference: async (reference: { id: string }, context: any) => {
      return context.dataloaders.user.load(reference.id);
    },
  },
};

function mapUserRow(row: any) {
  return {
    id: row.id,
    email: row.email,
    name: row.name,
    role: row.role,
    createdAt: row.created_at.toISOString(),
    profile: row.profile_avatar || row.profile_bio
      ? {
          avatar: row.profile_avatar,
          bio: row.profile_bio,
          phoneNumber: row.profile_phone,
        }
      : null,
  };
}
```

---

## 3. Subgraph: Products Service

```typescript
// products-service/src/schema.ts
import { gql } from 'graphql-tag';

export const typeDefs = gql`
  extend schema
    @link(
      url: "https://specs.apollo.dev/federation/v2.0"
      import: ["@key", "@shareable", "@provides"]
    )

  type Product @key(fields: "id") {
    id: ID!
    sku: String!
    name: String!
    description: String
    price: Money!
    category: Category!
    inventory: Inventory!
    images: [ProductImage!]!
    vendor: Vendor!
    isActive: Boolean!
    createdAt: String!
  }

  type Money @shareable {
    amount: Float!
    currency: String!
    formatted: String!
  }

  type Inventory {
    available: Int!
    reserved: Int!
    total: Int!
    inStock: Boolean!
  }

  type ProductImage {
    url: String!
    alt: String
    isPrimary: Boolean!
  }

  type Category @key(fields: "id") {
    id: ID!
    name: String!
    slug: String!
    parentId: ID
  }

  type Vendor @key(fields: "id") {
    id: ID!
    name: String!
    @provides(fields: "name")
  }

  type Query {
    product(id: ID!): Product
    products(
      filter: ProductFilter
      sort: ProductSort
      page: Int
      limit: Int
    ): ProductConnection!
    searchProducts(query: String!, limit: Int): [Product!]!
  }

  input ProductFilter {
    categoryId: ID
    vendorId: ID
    minPrice: Float
    maxPrice: Float
    inStockOnly: Boolean
  }

  input ProductSort {
    field: ProductSortField!
    order: SortOrder!
  }

  enum ProductSortField {
    PRICE
    NAME
    CREATED_AT
    POPULARITY
  }

  enum SortOrder {
    ASC
    DESC
  }

  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type ProductEdge {
    node: Product!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  type Mutation {
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    updateInventory(productId: ID!, delta: Int!): Inventory!
  }

  input CreateProductInput {
    sku: String!
    name: String!
    description: String
    price: MoneyInput!
    categoryId: ID!
    vendorId: ID!
    initialInventory: Int!
  }

  input UpdateProductInput {
    name: String
    description: String
    price: MoneyInput
    isActive: Boolean
  }

  input MoneyInput {
    amount: Float!
    currency: String!
  }
`;
```

```typescript
// products-service/src/resolvers/product.resolver.ts
import DataLoader from 'dataloader';
import { Pool } from 'pg';

export const ProductResolver = {
  Query: {
    product: async (_: any, { id }: { id: string }, context: any) => {
      return context.dataloaders.product.load(id);
    },

    products: async (
      _: any,
      {
        filter,
        sort,
        page = 1,
        limit = 20,
      }: {
        filter?: any;
        sort?: any;
        page: number;
        limit: number;
      },
      context: any
    ) => {
      const pool: Pool = context.pool;
      const conditions: string[] = ['p.deleted_at IS NULL'];
      const params: any[] = [];

      if (filter?.categoryId) {
        params.push(filter.categoryId);
        conditions.push(`p.category_id = $${params.length}`);
      }

      if (filter?.vendorId) {
        params.push(filter.vendorId);
        conditions.push(`p.vendor_id = $${params.length}`);
      }

      if (filter?.minPrice !== undefined) {
        params.push(filter.minPrice);
        conditions.push(`p.price_amount >= $${params.length}`);
      }

      if (filter?.maxPrice !== undefined) {
        params.push(filter.maxPrice);
        conditions.push(`p.price_amount <= $${params.length}`);
      }

      if (filter?.inStockOnly) {
        conditions.push(`i.available > 0`);
      }

      const whereClause =
        conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';

      const sortField = sort?.field === 'PRICE' ? 'p.price_amount'
        : sort?.field === 'NAME' ? 'p.name'
        : 'p.created_at';
      const sortOrder = sort?.order || 'DESC';
      const offset = (page - 1) * limit;

      params.push(limit, offset);

      const [countResult, productsResult] = await Promise.all([
        pool.query(
          `SELECT COUNT(*) FROM products p 
           LEFT JOIN inventory i ON i.product_id = p.id
           ${whereClause}`,
          params.slice(0, -2)
        ),
        pool.query(
          `SELECT p.*, 
                  i.available, i.reserved,
                  c.name as category_name, c.slug as category_slug
           FROM products p
           LEFT JOIN inventory i ON i.product_id = p.id
           LEFT JOIN categories c ON c.id = p.category_id
           ${whereClause}
           ORDER BY ${sortField} ${sortOrder}
           LIMIT $${params.length - 1} OFFSET $${params.length}`,
          params
        ),
      ]);

      const totalCount = parseInt(countResult.rows[0].count);

      return {
        edges: productsResult.rows.map((row, index) => ({
          node: mapProductRow(row),
          cursor: Buffer.from(`${offset + index + 1}`).toString('base64'),
        })),
        pageInfo: {
          hasNextPage: offset + limit < totalCount,
          hasPreviousPage: page > 1,
          startCursor: null,
          endCursor: null,
        },
        totalCount,
      };
    },

    searchProducts: async (
      _: any,
      { query, limit = 10 }: { query: string; limit: number },
      context: any
    ) => {
      const pool: Pool = context.pool;
      const result = await pool.query(
        `SELECT p.*, i.available, i.reserved
         FROM products p
         LEFT JOIN inventory i ON i.product_id = p.id
         WHERE p.deleted_at IS NULL
           AND (
             p.name ILIKE $1
             OR p.description ILIKE $1
             OR p.sku ILIKE $1
           )
         ORDER BY ts_rank(to_tsvector('english', p.name), plainto_tsquery($2)) DESC
         LIMIT $3`,
        [`%${query}%`, query, limit]
      );

      return result.rows.map(mapProductRow);
    },
  },

  Mutation: {
    updateInventory: async (
      _: any,
      { productId, delta }: { productId: string; delta: number },
      context: any
    ) => {
      const pool: Pool = context.pool;
      const result = await pool.query(
        `UPDATE inventory 
         SET available = available + $2, updated_at = NOW()
         WHERE product_id = $1
         RETURNING *`,
        [productId, delta]
      );

      if (result.rows.length === 0) {
        throw new Error(`Inventory not found for product: ${productId}`);
      }

      const row = result.rows[0];
      return {
        available: row.available,
        reserved: row.reserved,
        total: row.available + row.reserved,
        inStock: row.available > 0,
      };
    },
  },

  Product: {
    __resolveReference: async (reference: { id: string }, context: any) => {
      return context.dataloaders.product.load(reference.id);
    },

    inventory: async (product: any, _: any, context: any) => {
      return context.dataloaders.inventory.load(product.id);
    },

    vendor: (product: any) => {
      // Return stub with just the id — the Vendor subgraph provides the rest
      return { __typename: 'Vendor', id: product.vendorId };
    },
  },

  Category: {
    __resolveReference: async (reference: { id: string }, context: any) => {
      return context.dataloaders.category.load(reference.id);
    },
  },
};

function mapProductRow(row: any) {
  return {
    id: row.id,
    sku: row.sku,
    name: row.name,
    description: row.description,
    price: {
      amount: parseFloat(row.price_amount),
      currency: row.price_currency,
      formatted: `${row.price_currency} ${parseFloat(row.price_amount).toFixed(2)}`,
    },
    category: {
      id: row.category_id,
      name: row.category_name,
      slug: row.category_slug,
    },
    inventory: row.available !== undefined
      ? {
          available: row.available,
          reserved: row.reserved,
          total: row.available + row.reserved,
          inStock: row.available > 0,
        }
      : null,
    vendorId: row.vendor_id,
    images: [],
    isActive: row.is_active,
    createdAt: row.created_at.toISOString(),
  };
}
```

---

## 4. DataLoader สำหรับแก้ปัญหา N+1

```typescript
// products-service/src/dataloaders/index.ts
import DataLoader from 'dataloader';
import { Pool } from 'pg';

export function createDataLoaders(pool: Pool) {
  // Product DataLoader — batch load by IDs
  const productLoader = new DataLoader<string, any>(
    async (ids: readonly string[]) => {
      const result = await pool.query(
        `SELECT p.*, i.available, i.reserved
         FROM products p
         LEFT JOIN inventory i ON i.product_id = p.id
         WHERE p.id = ANY($1)`,
        [Array.from(ids)]
      );

      // Map to original order
      const productMap = new Map(result.rows.map((row) => [row.id, row]));
      return ids.map((id) => productMap.get(id) || null);
    },
    {
      maxBatchSize: 100,
      cache: true,
    }
  );

  // Inventory DataLoader
  const inventoryLoader = new DataLoader<string, any>(
    async (productIds: readonly string[]) => {
      const result = await pool.query(
        'SELECT * FROM inventory WHERE product_id = ANY($1)',
        [Array.from(productIds)]
      );

      const inventoryMap = new Map(
        result.rows.map((row) => [row.product_id, row])
      );

      return productIds.map((id) => {
        const inv = inventoryMap.get(id);
        if (!inv) return null;
        return {
          available: inv.available,
          reserved: inv.reserved,
          total: inv.available + inv.reserved,
          inStock: inv.available > 0,
        };
      });
    }
  );

  // Category DataLoader
  const categoryLoader = new DataLoader<string, any>(
    async (ids: readonly string[]) => {
      const result = await pool.query(
        'SELECT * FROM categories WHERE id = ANY($1)',
        [Array.from(ids)]
      );

      const categoryMap = new Map(result.rows.map((row) => [row.id, row]));
      return ids.map((id) => categoryMap.get(id) || null);
    }
  );

  // User DataLoader (from users service context)
  const userLoader = new DataLoader<string, any>(
    async (ids: readonly string[]) => {
      const result = await pool.query(
        'SELECT * FROM users WHERE id = ANY($1)',
        [Array.from(ids)]
      );

      const userMap = new Map(result.rows.map((row) => [row.id, row]));
      return ids.map((id) => userMap.get(id) || null);
    }
  );

  return {
    product: productLoader,
    inventory: inventoryLoader,
    category: categoryLoader,
    user: userLoader,
  };
}
```

---

## 5. Subgraph: Orders Service พร้อม @key และ @external

```typescript
// orders-service/src/schema.ts
import { gql } from 'graphql-tag';

export const typeDefs = gql`
  extend schema
    @link(
      url: "https://specs.apollo.dev/federation/v2.0"
      import: ["@key", "@external", "@requires", "@shareable"]
    )

  # Reference types from other subgraphs
  type User @key(fields: "id", resolvable: false) {
    id: ID!
  }

  type Product @key(fields: "id") {
    id: ID!
    name: String! @external
    price: Money! @external
    orders: [OrderItem!]!
  }

  type Money @shareable {
    amount: Float!
    currency: String!
    formatted: String!
  }

  type Order @key(fields: "id") {
    id: ID!
    customer: User!
    status: OrderStatus!
    items: [OrderItem!]!
    totalAmount: Money!
    shippingAddress: Address!
    trackingNumber: String
    createdAt: String!
    updatedAt: String!
  }

  type OrderItem {
    id: ID!
    product: Product!
    quantity: Int!
    unitPrice: Money!
    totalPrice: Money!
  }

  type Address @shareable {
    street: String!
    city: String!
    province: String!
    postalCode: String!
    country: String!
  }

  enum OrderStatus {
    DRAFT
    CONFIRMED
    PAID
    SHIPPED
    DELIVERED
    CANCELLED
  }

  type Query {
    order(id: ID!): Order
    myOrders(status: OrderStatus, page: Int, limit: Int): OrderConnection!
  }

  type OrderConnection {
    edges: [OrderEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type OrderEdge {
    node: Order!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  type Subscription {
    orderStatusChanged(orderId: ID!): Order!
    myOrderUpdates: Order!
  }

  type Mutation {
    createOrder(input: CreateOrderInput!): CreateOrderPayload!
    cancelOrder(id: ID!, reason: String): CancelOrderPayload!
  }

  input CreateOrderInput {
    shippingAddress: AddressInput!
    items: [OrderItemInput!]!
  }

  input AddressInput {
    street: String!
    city: String!
    province: String!
    postalCode: String!
    country: String!
  }

  input OrderItemInput {
    productId: ID!
    quantity: Int!
  }

  type CreateOrderPayload {
    order: Order
    errors: [OrderError!]
  }

  type CancelOrderPayload {
    order: Order
    errors: [OrderError!]
  }

  type OrderError {
    field: String
    message: String!
    code: String!
  }
`;
```

---

## 6. Subscriptions ผ่าน WebSocket

```typescript
// orders-service/src/server.ts
import { ApolloServer } from '@apollo/server';
import { expressMiddleware } from '@apollo/server/express4';
import { ApolloServerPluginDrainHttpServer } from '@apollo/server/plugin/drainHttpServer';
import { makeExecutableSchema } from '@graphql-tools/schema';
import { WebSocketServer } from 'ws';
import { useServer } from 'graphql-ws/lib/use/ws';
import { PubSub } from 'graphql-subscriptions';
import express from 'express';
import http from 'http';
import cors from 'cors';
import { buildSubgraphSchema } from '@apollo/subgraph';
import { typeDefs } from './schema';
import { createResolvers } from './resolvers';
import { createDataLoaders } from './dataloaders';
import { Pool } from 'pg';

const pubsub = new PubSub();

export async function startServer(pool: Pool) {
  const app = express();
  const httpServer = http.createServer(app);

  // WebSocket server for subscriptions
  const wsServer = new WebSocketServer({
    server: httpServer,
    path: '/graphql',
  });

  const schema = buildSubgraphSchema({
    typeDefs,
    resolvers: createResolvers(pubsub),
  });

  // Cleanup function for WebSocket server
  const serverCleanup = useServer(
    {
      schema,
      context: async (ctx) => {
        // Extract auth token from WebSocket connection params
        const token = ctx.connectionParams?.authorization as string;
        const user = token ? await verifyToken(token) : null;

        return {
          user,
          pool,
          pubsub,
          dataloaders: createDataLoaders(pool),
        };
      },
    },
    wsServer
  );

  const server = new ApolloServer({
    schema,
    plugins: [
      ApolloServerPluginDrainHttpServer({ httpServer }),
      {
        async serverWillStart() {
          return {
            async drainServer() {
              await serverCleanup.dispose();
            },
          };
        },
      },
    ],
  });

  await server.start();

  app.use(
    '/graphql',
    cors<cors.CorsRequest>({
      origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
      credentials: true,
    }),
    express.json(),
    expressMiddleware(server, {
      context: async ({ req }) => {
        const token = req.headers.authorization?.replace('Bearer ', '');
        const user = token ? await verifyToken(token) : null;

        return {
          user,
          pool,
          pubsub,
          dataloaders: createDataLoaders(pool),
        };
      },
    })
  );

  const PORT = process.env.PORT || 4003;
  await new Promise<void>((resolve) =>
    httpServer.listen({ port: PORT }, resolve)
  );

  console.log(`Orders Subgraph ready at http://localhost:${PORT}/graphql`);
  console.log(`Subscriptions ready at ws://localhost:${PORT}/graphql`);

  return { server, httpServer };
}

async function verifyToken(token: string): Promise<any> {
  // Implement JWT verification
  const jwt = require('jsonwebtoken');
  try {
    return jwt.verify(token, process.env.JWT_SECRET);
  } catch {
    return null;
  }
}
```

```typescript
// orders-service/src/resolvers/subscription.resolver.ts
import { PubSub } from 'graphql-subscriptions';
import { withFilter } from 'graphql-subscriptions';

export const EVENTS = {
  ORDER_STATUS_CHANGED: 'ORDER_STATUS_CHANGED',
  ORDER_UPDATED: 'ORDER_UPDATED',
};

export function createSubscriptionResolvers(pubsub: PubSub) {
  return {
    Subscription: {
      orderStatusChanged: {
        subscribe: withFilter(
          () => pubsub.asyncIterator([EVENTS.ORDER_STATUS_CHANGED]),
          (payload, variables, context) => {
            // Only send to the subscriber who requested this orderId
            return payload.orderStatusChanged.id === variables.orderId;
          }
        ),
      },

      myOrderUpdates: {
        subscribe: withFilter(
          () => pubsub.asyncIterator([EVENTS.ORDER_UPDATED]),
          (payload, _, context) => {
            // Only send updates for orders belonging to this user
            return (
              payload.myOrderUpdates.customerId === context.user?.id
            );
          }
        ),
      },
    },
  };
}
```

---

## 7. Reviews Subgraph — @requires และ @provides

```typescript
// reviews-service/src/schema.ts
import { gql } from 'graphql-tag';

export const typeDefs = gql`
  extend schema
    @link(
      url: "https://specs.apollo.dev/federation/v2.0"
      import: ["@key", "@external", "@requires", "@provides", "@shareable"]
    )

  type Product @key(fields: "id") {
    id: ID!
    name: String! @external
    # Extending Product with reviews field
    reviews(limit: Int): [Review!]!
    averageRating: Float!
    reviewCount: Int!
  }

  type User @key(fields: "id") {
    id: ID!
    name: String! @external
    email: String! @external
    # Extending User with reviews field
    reviews: [Review!]!
    # @requires เพื่อใช้ fields จาก User subgraph
    reviewSummary: ReviewSummary! @requires(fields: "name email")
  }

  type Review @key(fields: "id") {
    id: ID!
    product: Product!
    author: User!
    rating: Int!
    title: String!
    body: String
    verified: Boolean!
    helpful: Int!
    createdAt: String!
  }

  type ReviewSummary {
    authorName: String!
    authorEmail: String!
    totalReviews: Int!
    averageRatingGiven: Float!
  }

  type Query {
    review(id: ID!): Review
    productReviews(productId: ID!, limit: Int, offset: Int): ReviewConnection!
  }

  type ReviewConnection {
    edges: [ReviewEdge!]!
    totalCount: Int!
    averageRating: Float!
  }

  type ReviewEdge {
    node: Review!
  }

  type Mutation {
    createReview(input: CreateReviewInput!): Review!
    markReviewHelpful(reviewId: ID!): Review!
  }

  input CreateReviewInput {
    productId: ID!
    rating: Int!
    title: String!
    body: String
  }
`;
```

```typescript
// reviews-service/src/resolvers/review.resolver.ts
export const ReviewResolver = {
  Query: {
    productReviews: async (
      _: any,
      {
        productId,
        limit = 10,
        offset = 0,
      }: { productId: string; limit: number; offset: number },
      context: any
    ) => {
      const { pool } = context;

      const [countResult, reviewsResult, avgResult] = await Promise.all([
        pool.query(
          'SELECT COUNT(*) FROM reviews WHERE product_id = $1',
          [productId]
        ),
        pool.query(
          `SELECT * FROM reviews WHERE product_id = $1
           ORDER BY created_at DESC LIMIT $2 OFFSET $3`,
          [productId, limit, offset]
        ),
        pool.query(
          'SELECT AVG(rating) FROM reviews WHERE product_id = $1',
          [productId]
        ),
      ]);

      return {
        edges: reviewsResult.rows.map((row) => ({
          node: mapReviewRow(row),
        })),
        totalCount: parseInt(countResult.rows[0].count),
        averageRating: parseFloat(avgResult.rows[0].avg || 0),
      };
    },
  },

  Mutation: {
    createReview: async (_: any, { input }: { input: any }, context: any) => {
      if (!context.user) throw new Error('Authentication required');

      const { pool, pubsub } = context;

      // Check if user already reviewed this product
      const existing = await pool.query(
        'SELECT id FROM reviews WHERE product_id = $1 AND user_id = $2',
        [input.productId, context.user.id]
      );

      if (existing.rows.length > 0) {
        throw new Error('You have already reviewed this product');
      }

      const result = await pool.query(
        `INSERT INTO reviews (product_id, user_id, rating, title, body, verified)
         VALUES ($1, $2, $3, $4, $5, false)
         RETURNING *`,
        [
          input.productId,
          context.user.id,
          input.rating,
          input.title,
          input.body,
        ]
      );

      const review = mapReviewRow(result.rows[0]);

      // Publish event for real-time updates
      pubsub.publish('REVIEW_CREATED', { reviewCreated: review });

      return review;
    },
  },

  Product: {
    __resolveReference: async (reference: { id: string }, context: any) => {
      return { id: reference.id };
    },

    reviews: async (product: { id: string }, { limit = 5 }: any, context: any) => {
      return context.dataloaders.productReviews.load({
        productId: product.id,
        limit,
      });
    },

    averageRating: async (product: { id: string }, _: any, context: any) => {
      const result = await context.pool.query(
        'SELECT AVG(rating) FROM reviews WHERE product_id = $1',
        [product.id]
      );
      return parseFloat(result.rows[0].avg || 0);
    },

    reviewCount: async (product: { id: string }, _: any, context: any) => {
      const result = await context.pool.query(
        'SELECT COUNT(*) FROM reviews WHERE product_id = $1',
        [product.id]
      );
      return parseInt(result.rows[0].count);
    },
  },

  User: {
    __resolveReference: async (reference: { id: string }, context: any) => {
      return reference;
    },

    reviews: async (user: { id: string }, _: any, context: any) => {
      const result = await context.pool.query(
        'SELECT * FROM reviews WHERE user_id = $1 ORDER BY created_at DESC',
        [user.id]
      );
      return result.rows.map(mapReviewRow);
    },

    // @requires(fields: "name email") — user.name and user.email come from Users subgraph
    reviewSummary: async (
      user: { id: string; name: string; email: string },
      _: any,
      context: any
    ) => {
      const result = await context.pool.query(
        `SELECT COUNT(*) as total, AVG(rating) as avg_rating
         FROM reviews WHERE user_id = $1`,
        [user.id]
      );

      return {
        authorName: user.name,
        authorEmail: user.email,
        totalReviews: parseInt(result.rows[0].total),
        averageRatingGiven: parseFloat(result.rows[0].avg_rating || 0),
      };
    },
  },
};

function mapReviewRow(row: any) {
  return {
    id: row.id,
    product: { __typename: 'Product', id: row.product_id },
    author: { __typename: 'User', id: row.user_id },
    rating: row.rating,
    title: row.title,
    body: row.body,
    verified: row.verified,
    helpful: row.helpful || 0,
    createdAt: row.created_at.toISOString(),
  };
}
```

---

## 8. Apollo Router Configuration

```yaml
# router.yaml
supergraph:
  # Path to the composed schema
  path: ./supergraph-schema.graphql

homepage:
  enabled: false

sandbox:
  enabled: true

health-check:
  enabled: true
  listen: 127.0.0.1:8088

cors:
  origins:
    - http://localhost:3000
    - https://www.myshop.com
  allow_headers:
    - Authorization
    - Content-Type
    - Apollo-Require-Preflight
  methods:
    - GET
    - POST
    - OPTIONS

limits:
  # Maximum depth of incoming queries
  max_depth: 15
  # Maximum complexity score
  max_complexity: 2000

traffic_shaping:
  router:
    # Enable request deduplication
    deduplicate_query: true
  all:
    timeout: 30s
    retry:
      enabled: true
      min_per_sec: 10
      retry_on_http_statuses:
        - 500
        - 503

authentication:
  router:
    jwt:
      jwks:
        - url: http://auth-service:4000/.well-known/jwks.json
          poll_interval: 60s
      header_name: Authorization
      header_value_prefix: Bearer

authorization:
  preview_directives:
    enabled: true
  require_authentication: false

coprocessor:
  url: http://auth-coprocessor:9000/
  router:
    request:
      headers: true
      body: false
      context: true
    response:
      headers: true
      body: false
      context: true

# Subgraph services
override_subgraph_url:
  users: http://users-service:4001/graphql
  products: http://products-service:4002/graphql
  orders: http://orders-service:4003/graphql
  reviews: http://reviews-service:4004/graphql

telemetry:
  exporters:
    tracing:
      otlp:
        endpoint: http://jaeger:4318/v1/traces
        protocol: http
    metrics:
      prometheus:
        enabled: true
        listen: 0.0.0.0:9090
        path: /metrics

  instrumentation:
    spans:
      router:
        request:
          attributes:
            http.method: true
            http.status_code: true
        response:
          attributes:
            http.status_code: true
    events:
      router:
        request: info
        response: info
        error: error
      supergraph:
        request: info
        response: info
        error: error
```

---

## 9. Supergraph Composition

```yaml
# supergraph-config.yaml
federation_version: =2.4.0

subgraphs:
  users:
    routing_url: http://users-service:4001/graphql
    schema:
      subgraph_url: http://users-service:4001/graphql

  products:
    routing_url: http://products-service:4002/graphql
    schema:
      subgraph_url: http://products-service:4002/graphql

  orders:
    routing_url: http://orders-service:4003/graphql
    schema:
      subgraph_url: http://orders-service:4003/graphql

  reviews:
    routing_url: http://reviews-service:4004/graphql
    schema:
      subgraph_url: http://reviews-service:4004/graphql
```

```bash
#!/bin/bash
# compose-schema.sh

# Install rover CLI if not present
if ! command -v rover &> /dev/null; then
  curl -sSL https://rover.apollo.dev/nix/latest | sh
fi

# Compose the supergraph schema
rover supergraph compose \
  --config ./supergraph-config.yaml \
  --output ./supergraph-schema.graphql

echo "Supergraph schema composed successfully"
```

---

## 10. Authorization ที่ Gateway ด้วย Coprocessor

```typescript
// auth-coprocessor/src/index.ts
import express from 'express';
import jwt from 'jsonwebtoken';

const app = express();
app.use(express.json());

interface CoprocessorRequest {
  version: number;
  stage: string;
  control: string;
  headers: Record<string, string[]>;
  body?: string;
  context: Record<string, any>;
  sdl?: string;
}

app.post('/', async (req, res) => {
  const payload: CoprocessorRequest = req.body;

  if (payload.stage === 'RouterRequest') {
    return handleRouterRequest(payload, res);
  }

  if (payload.stage === 'RouterResponse') {
    return handleRouterResponse(payload, res);
  }

  res.json(payload);
});

async function handleRouterRequest(
  payload: CoprocessorRequest,
  res: express.Response
) {
  const authHeader = payload.headers['authorization']?.[0];
  const token = authHeader?.replace('Bearer ', '');

  if (!token) {
    // Allow anonymous access — subgraphs handle auth per operation
    return res.json({
      ...payload,
      context: {
        ...payload.context,
        'x-user-id': null,
        'x-user-role': 'ANONYMOUS',
      },
    });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as any;

    // Add user info to context (passed to all subgraphs)
    res.json({
      ...payload,
      headers: {
        ...payload.headers,
        'x-user-id': [decoded.sub],
        'x-user-role': [decoded.role],
        'x-user-email': [decoded.email],
      },
      context: {
        ...payload.context,
        'x-user-id': decoded.sub,
        'x-user-role': decoded.role,
        'x-user-email': decoded.email,
      },
    });
  } catch (error) {
    // Invalid token
    res.status(401).json({
      control: {
        break: 401,
      },
      body: JSON.stringify({
        errors: [
          {
            message: 'Invalid or expired token',
            extensions: {
              code: 'UNAUTHENTICATED',
            },
          },
        ],
      }),
    });
  }
}

async function handleRouterResponse(
  payload: CoprocessorRequest,
  res: express.Response
) {
  // Add security headers
  res.json({
    ...payload,
    headers: {
      ...payload.headers,
      'x-content-type-options': ['nosniff'],
      'x-frame-options': ['DENY'],
      'strict-transport-security': ['max-age=31536000; includeSubDomains'],
    },
  });
}

app.listen(9000, () => {
  console.log('Auth coprocessor running on port 9000');
});
```

---

## 11. Docker Compose สำหรับ Federation

```yaml
# docker-compose.yml
version: '3.9'

services:
  apollo-router:
    image: ghcr.io/apollographql/router:v1.43.2
    ports:
      - "4000:4000"
      - "9090:9090"
    volumes:
      - ./router.yaml:/dist/config/router.yaml
      - ./supergraph-schema.graphql:/dist/config/supergraph-schema.graphql
    command:
      - --config
      - /dist/config/router.yaml
      - --supergraph
      - /dist/config/supergraph-schema.graphql
      - --dev
    environment:
      APOLLO_TELEMETRY_DISABLED: "true"
      RUST_LOG: "info"
    depends_on:
      - users-service
      - products-service
      - orders-service
      - reviews-service

  auth-coprocessor:
    build:
      context: ./auth-coprocessor
    environment:
      JWT_SECRET: ${JWT_SECRET}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/health"]
      interval: 30s

  users-service:
    build:
      context: ./users-service
    ports:
      - "4001:4001"
    environment:
      PORT: 4001
      DATABASE_URL: postgresql://users_user:users_pass@postgres:5432/users_db
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy

  products-service:
    build:
      context: ./products-service
    ports:
      - "4002:4002"
    environment:
      PORT: 4002
      DATABASE_URL: postgresql://products_user:products_pass@postgres:5432/products_db
    depends_on:
      postgres:
        condition: service_healthy

  orders-service:
    build:
      context: ./orders-service
    ports:
      - "4003:4003"
    environment:
      PORT: 4003
      DATABASE_URL: postgresql://orders_user:orders_pass@postgres:5432/orders_db
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      postgres:
        condition: service_healthy

  reviews-service:
    build:
      context: ./reviews-service
    ports:
      - "4004:4004"
    environment:
      PORT: 4004
      DATABASE_URL: postgresql://reviews_user:reviews_pass@postgres:5432/reviews_db
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  jaeger:
    image: jaegertracing/all-in-one:1.55
    ports:
      - "16686:16686"
      - "4318:4318"

volumes:
  postgres_data:
```

---

## 12. ตัวอย่าง Query ข้าม Subgraphs

```graphql
# Query ที่รวม Users, Products, Orders, Reviews ใน Request เดียว
query GetUserDashboard($userId: ID!) {
  # From Users Subgraph
  user(id: $userId) {
    id
    name
    email
    role
    
    # From Reviews Subgraph (@requires name, email)
    reviewSummary {
      totalReviews
      averageRatingGiven
    }
    
    reviews {
      id
      rating
      title
      # From Products Subgraph
      product {
        id
        name
        price {
          amount
          currency
          formatted
        }
      }
    }
  }
}

# Subscription สำหรับ Real-time Order Updates
subscription WatchOrderStatus($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    id
    status
    trackingNumber
    updatedAt
    items {
      quantity
      product {
        name
      }
    }
  }
}
```

---

## สรุป

| หัวข้อ | เทคนิค | ประโยชน์ |
|--------|---------|---------|
| **Federation Architecture** | Apollo Federation 2.0 | รวม Multiple GraphQL APIs เป็น Single Supergraph |
| **@key directive** | กำหนด Primary Key ของ Entity | ให้ Router เข้าใจวิธี Join ข้าม Subgraphs |
| **@external** | ระบุ Field จาก Subgraph อื่น | ใช้ข้อมูลข้ามบริการโดยไม่ต้อง Duplicate |
| **@requires** | ต้องการ Fields อื่นก่อน Resolve | เข้าถึง Data จาก Subgraph อื่นใน Resolver |
| **@provides** | บอก Router ว่า Field มีอยู่ | ลด Round-trip ไปยัง Subgraph อื่น |
| **@shareable** | ใช้ Type เดียวกันข้าม Subgraphs | ลด Schema Duplication |
| **DataLoader** | Batching + Caching | แก้ปัญหา N+1 Query |
| **Subscriptions** | graphql-ws + WebSocket | Real-time updates ผ่าน Gateway |
| **Coprocessor** | Middleware ที่ Gateway | Auth, Rate Limiting, Header manipulation |
| **Router Config** | YAML-based configuration | Traffic shaping, Retry, Timeout |

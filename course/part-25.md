# Part 25: GraphQL Federation for Microservices

## ภาพรวม

ใน Part นี้เราจะเรียนรู้ GraphQL Federation:
- **GraphQL Basics** สำหรับ Microservices
- **Apollo Federation** - Single Graph จากหลาย services
- **Schema Composition** - Subgraphs และ Supergraph
- **Entity References** - Join data ข้ามบริการ
- **Subscriptions** - Real-time data
- **Performance** - DataLoader, persisted queries

---

## 1. GraphQL vs REST สำหรับ Microservices

```
REST Problems:
- Multiple roundtrips (N+1 requests)
- Over-fetching (ได้ข้อมูลมากเกินต้องการ)
- Under-fetching (ต้องส่ง request หลายครั้ง)
- API versioning ซับซ้อน

GraphQL Benefits:
- Single request, get exactly what you need
- Strongly typed schema
- Self-documenting
- Real-time subscriptions

GraphQL Federation:
- แต่ละ microservice มี subgraph ของตัวเอง
- Gateway รวม subgraphs เป็น single graph
- Frontend query ผ่าน gateway เพียง endpoint เดียว
```

---

## 2. Setup Apollo Federation

### 2.1 Gateway

```javascript
// gateway/index.js
const { ApolloGateway, IntrospectAndCompose } = require('@apollo/gateway');
const { ApolloServer } = require('@apollo/server');
const { expressMiddleware } = require('@apollo/server/express4');
const express = require('express');

const app = express();

const gateway = new ApolloGateway({
  supergraphSdl: new IntrospectAndCompose({
    subgraphs: [
      { name: 'users', url: 'http://user-service:4001/graphql' },
      { name: 'products', url: 'http://product-service:4002/graphql' },
      { name: 'orders', url: 'http://order-service:4003/graphql' },
      { name: 'reviews', url: 'http://review-service:4004/graphql' }
    ],
    // Polling interval for schema changes
    pollIntervalInMs: 15000
  }),

  buildService({ name, url }) {
    return new RemoteGraphQLDataSource({
      url,
      willSendRequest({ request, context }) {
        // Forward auth headers to subgraphs
        request.http.headers.set(
          'Authorization',
          context.authHeader || ''
        );
        request.http.headers.set(
          'X-User-Id',
          context.userId || ''
        );
        request.http.headers.set(
          'X-Tenant-Id',
          context.tenantId || ''
        );
      }
    });
  }
});

const server = new ApolloServer({
  gateway,
  context: ({ req }) => {
    const token = req.headers.authorization || '';
    const user = verifyToken(token);
    
    return {
      authHeader: token,
      userId: user?.id,
      tenantId: user?.tenantId
    };
  },
  plugins: [
    // Caching
    ApolloServerPluginResponseCache({
      sessionId: (requestContext) => {
        return requestContext.request.http?.headers.get('authorization') || null;
      }
    })
  ]
});

await server.start();
app.use('/graphql', expressMiddleware(server));
app.listen(4000);
```

### 2.2 User Service Subgraph

```javascript
// user-service/graphql/schema.js
const { buildSubgraphSchema } = require('@apollo/subgraph');
const { gql } = require('graphql-tag');

const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0", import: ["@key", "@shareable"])

  type Query {
    me: User
    user(id: ID!): User
    users(filter: UserFilter, pagination: PaginationInput): UserConnection!
  }

  type Mutation {
    createUser(input: CreateUserInput!): CreateUserResult!
    updateUser(id: ID!, input: UpdateUserInput!): User!
    deleteUser(id: ID!): Boolean!
  }

  # Entity definition - can be referenced by other subgraphs
  type User @key(fields: "id") {
    id: ID!
    email: String!
    name: String!
    avatar: String
    role: UserRole!
    status: UserStatus!
    createdAt: DateTime!
    
    # These fields are resolved in THIS subgraph
    profile: UserProfile
    preferences: UserPreferences
  }

  type UserProfile {
    bio: String
    phone: String
    address: Address
  }

  type Address {
    street: String!
    city: String!
    postalCode: String!
    country: String!
  }

  type UserPreferences {
    language: String!
    timezone: String!
    notifications: NotificationSettings!
  }

  type NotificationSettings {
    email: Boolean!
    push: Boolean!
    sms: Boolean!
  }

  type UserConnection {
    nodes: [User!]!
    totalCount: Int!
    pageInfo: PageInfo!
  }

  type PageInfo {
    hasNextPage: Boolean!
    endCursor: String
  }

  input UserFilter {
    status: UserStatus
    role: UserRole
    search: String
  }

  input PaginationInput {
    first: Int
    after: String
  }

  input CreateUserInput {
    email: String!
    password: String!
    name: String!
    role: UserRole
  }

  type CreateUserResult {
    user: User!
    token: String!
  }

  input UpdateUserInput {
    name: String
    avatar: String
  }

  enum UserRole {
    ADMIN
    MANAGER
    USER
  }

  enum UserStatus {
    ACTIVE
    INACTIVE
    SUSPENDED
  }

  scalar DateTime
`;

const resolvers = {
  Query: {
    me: async (_, __, { userId, dataSources }) => {
      if (!userId) throw new AuthenticationError('Not authenticated');
      return dataSources.userAPI.getUserById(userId);
    },

    user: async (_, { id }, { dataSources, userId }) => {
      // Only admins can view other users
      const requestingUser = await dataSources.userAPI.getUserById(userId);
      if (requestingUser.role !== 'ADMIN' && id !== userId) {
        throw new ForbiddenError('Access denied');
      }
      return dataSources.userAPI.getUserById(id);
    },

    users: async (_, { filter, pagination }, { dataSources }) => {
      return dataSources.userAPI.getUsers(filter, pagination);
    }
  },

  Mutation: {
    createUser: async (_, { input }, { dataSources }) => {
      return dataSources.userAPI.createUser(input);
    },

    updateUser: async (_, { id, input }, { userId, dataSources }) => {
      if (id !== userId) throw new ForbiddenError('Can only update your own profile');
      return dataSources.userAPI.updateUser(id, input);
    }
  },

  // Reference resolver - ให้ subgraphs อื่น fetch User entity ได้
  User: {
    __resolveReference: async (reference, { dataSources }) => {
      return dataSources.userAPI.getUserById(reference.id);
    },

    profile: async (user, _, { dataSources }) => {
      return dataSources.userAPI.getUserProfile(user.id);
    }
  }
};

module.exports = buildSubgraphSchema({ typeDefs, resolvers });
```

### 2.3 Order Service Subgraph (Entity Extension)

```javascript
// order-service/graphql/schema.js
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0",
          import: ["@key", "@external", "@requires", "@provides"])

  # Reference User type from user-service (external entity)
  type User @key(fields: "id") {
    id: ID! @external
    name: String! @external
    
    # Extend User with orders field (resolved here)
    orders(filter: OrderFilter): OrderConnection!
    orderStats: OrderStats!
  }

  type Query {
    order(id: ID!): Order
    orders(filter: OrderFilter, pagination: PaginationInput): OrderConnection!
  }

  type Mutation {
    createOrder(input: CreateOrderInput!): Order!
    cancelOrder(id: ID!, reason: String!): Order!
  }

  type Order @key(fields: "id") {
    id: ID!
    status: OrderStatus!
    items: [OrderItem!]!
    totalAmount: Float!
    currency: String!
    createdAt: DateTime!
    
    # Reference to User (resolved in user-service)
    user: User!
    
    # Reference to Products (resolved in product-service)
    shippingAddress: Address!
  }

  type OrderItem {
    quantity: Int!
    unitPrice: Float!
    lineTotal: Float!
    
    # Reference to Product
    product: Product!
  }

  # Reference Product from product-service
  type Product @key(fields: "id") {
    id: ID! @external
  }

  type OrderStats {
    totalOrders: Int!
    totalSpent: Float!
    completedOrders: Int!
  }

  type OrderConnection {
    nodes: [Order!]!
    totalCount: Int!
    pageInfo: PageInfo!
  }

  input CreateOrderInput {
    items: [OrderItemInput!]!
    shippingAddress: AddressInput!
    paymentMethodId: String!
  }

  input OrderItemInput {
    productId: ID!
    quantity: Int!
  }

  input OrderFilter {
    status: OrderStatus
    dateFrom: DateTime
    dateTo: DateTime
  }

  enum OrderStatus {
    DRAFT
    SUBMITTED
    CONFIRMED
    PAID
    SHIPPED
    DELIVERED
    CANCELLED
  }
`;

const resolvers = {
  Query: {
    order: (_, { id }, { dataSources }) => dataSources.orderAPI.getOrder(id),
    orders: (_, { filter, pagination }, { dataSources, userId }) =>
      dataSources.orderAPI.getOrders({ ...filter, userId }, pagination)
  },

  Mutation: {
    createOrder: async (_, { input }, { userId, dataSources }) => {
      return dataSources.orderAPI.createOrder({ ...input, userId });
    },
    cancelOrder: (_, { id, reason }, { dataSources }) =>
      dataSources.orderAPI.cancelOrder(id, reason)
  },

  Order: {
    __resolveReference: (ref, { dataSources }) =>
      dataSources.orderAPI.getOrder(ref.id),

    // Join with user-service
    user: (order) => ({ __typename: 'User', id: order.userId }),

    // Join with product-service
    items: async (order, _, { dataSources }) => {
      const items = await dataSources.orderAPI.getOrderItems(order.id);
      return items.map(item => ({
        ...item,
        product: { __typename: 'Product', id: item.productId }
      }));
    }
  },

  // Extend User type with orders
  User: {
    orders: (user, { filter }, { dataSources }) =>
      dataSources.orderAPI.getOrders({ ...filter, userId: user.id }),

    orderStats: (user, _, { dataSources }) =>
      dataSources.orderAPI.getOrderStats(user.id)
  }
};
```

---

## 3. Product Service Subgraph

```javascript
// product-service/graphql/schema.js
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0",
          import: ["@key", "@shareable"])

  type Query {
    product(id: ID!): Product
    products(filter: ProductFilter, pagination: PaginationInput): ProductConnection!
    searchProducts(query: String!, filters: SearchFilters): [Product!]!
    featuredProducts: [Product!]!
  }

  type Product @key(fields: "id") {
    id: ID!
    name: String!
    description: String
    price: Float!
    compareAtPrice: Float
    images: [ProductImage!]!
    category: Category!
    tags: [String!]!
    inStock: Boolean!
    stockCount: Int!
    rating: Float
    reviewCount: Int
    createdAt: DateTime!
  }

  type ProductImage {
    url: String!
    alt: String
    isPrimary: Boolean!
  }

  type Category @key(fields: "id") @shareable {
    id: ID!
    name: String!
    slug: String!
    parent: Category
    children: [Category!]!
  }

  type ProductConnection {
    nodes: [Product!]!
    totalCount: Int!
    pageInfo: PageInfo!
    facets: ProductFacets!
  }

  type ProductFacets {
    categories: [FacetValue!]!
    priceRange: PriceRange!
    tags: [FacetValue!]!
  }

  type FacetValue {
    value: String!
    count: Int!
  }

  type PriceRange {
    min: Float!
    max: Float!
  }

  input ProductFilter {
    categoryId: ID
    minPrice: Float
    maxPrice: Float
    inStock: Boolean
    tags: [String!]
  }

  input SearchFilters {
    categoryId: ID
    minPrice: Float
    maxPrice: Float
    inStock: Boolean
  }
`;

const resolvers = {
  Query: {
    product: (_, { id }, { dataSources }) =>
      dataSources.productAPI.getProduct(id),

    products: (_, { filter, pagination }, { dataSources }) =>
      dataSources.productAPI.getProducts(filter, pagination),

    searchProducts: (_, { query, filters }, { dataSources }) =>
      dataSources.productAPI.searchProducts(query, filters),

    featuredProducts: (_, __, { dataSources }) =>
      dataSources.productAPI.getFeaturedProducts()
  },

  Product: {
    __resolveReference: (ref, { dataSources }) =>
      dataSources.productAPI.getProduct(ref.id)
  }
};
```

---

## 4. Review Service - Cross-Service References

```javascript
// review-service/graphql/schema.js
const typeDefs = gql`
  extend schema
    @link(url: "https://specs.apollo.dev/federation/v2.0",
          import: ["@key", "@external"])

  # Reference Product from product-service
  type Product @key(fields: "id") {
    id: ID! @external
    reviews(pagination: PaginationInput): ReviewConnection!
    averageRating: Float!
    ratingDistribution: [RatingCount!]!
  }

  # Reference User from user-service
  type User @key(fields: "id") {
    id: ID! @external
    reviews: [Review!]!
  }

  type Query {
    review(id: ID!): Review
  }

  type Mutation {
    createReview(input: CreateReviewInput!): Review!
    updateReview(id: ID!, input: UpdateReviewInput!): Review!
    deleteReview(id: ID!): Boolean!
    markHelpful(reviewId: ID!): Review!
  }

  type Review @key(fields: "id") {
    id: ID!
    rating: Int!
    title: String!
    body: String!
    verified: Boolean!
    helpfulCount: Int!
    images: [String!]!
    createdAt: DateTime!
    
    # References
    product: Product!
    user: User!
  }

  type ReviewConnection {
    nodes: [Review!]!
    totalCount: Int!
    averageRating: Float!
    pageInfo: PageInfo!
  }

  type RatingCount {
    rating: Int!
    count: Int!
  }

  input CreateReviewInput {
    productId: ID!
    rating: Int!
    title: String!
    body: String!
    images: [String!]
  }

  input UpdateReviewInput {
    rating: Int
    title: String
    body: String
  }
`;

const resolvers = {
  Product: {
    reviews: (product, { pagination }, { dataSources }) =>
      dataSources.reviewAPI.getProductReviews(product.id, pagination),

    averageRating: (product, _, { dataSources }) =>
      dataSources.reviewAPI.getAverageRating(product.id),

    ratingDistribution: (product, _, { dataSources }) =>
      dataSources.reviewAPI.getRatingDistribution(product.id)
  },

  User: {
    reviews: (user, _, { dataSources }) =>
      dataSources.reviewAPI.getUserReviews(user.id)
  },

  Review: {
    __resolveReference: (ref, { dataSources }) =>
      dataSources.reviewAPI.getReview(ref.id),

    product: (review) => ({ __typename: 'Product', id: review.productId }),
    user: (review) => ({ __typename: 'User', id: review.userId })
  }
};
```

---

## 5. DataLoader สำหรับ N+1 Prevention

```javascript
// data-loaders/user-loader.js
const DataLoader = require('dataloader');

class UserDataLoader {
  constructor(userAPI) {
    this.batchGetUsers = new DataLoader(
      async (ids) => {
        const users = await userAPI.getUsersByIds(ids);
        const userMap = users.reduce((map, user) => {
          map[user.id] = user;
          return map;
        }, {});
        return ids.map(id => userMap[id] || null);
      },
      {
        cache: true,      // Cache within request
        maxBatchSize: 100 // Max batch size
      }
    );
  }

  async getUserById(id) {
    return this.batchGetUsers.load(id);
  }
}

// Request-scoped DataLoaders
function createDataLoaders(apis) {
  return {
    userLoader: new UserDataLoader(apis.userAPI),
    productLoader: new DataLoader(
      async (ids) => {
        const products = await apis.productAPI.getProductsByIds(ids);
        const map = products.reduce((m, p) => ({ ...m, [p.id]: p }), {});
        return ids.map(id => map[id] || null);
      }
    )
  };
}

// Context factory - create fresh loaders per request
function createContext(apis) {
  return ({ req }) => {
    const loaders = createDataLoaders(apis);
    const user = verifyToken(req.headers.authorization);
    
    return {
      userId: user?.id,
      tenantId: user?.tenantId,
      loaders,
      dataSources: apis
    };
  };
}
```

---

## 6. GraphQL Subscriptions

```javascript
// subscriptions/order-subscriptions.js
const { PubSub, withFilter } = require('graphql-subscriptions');
const { RedisPubSub } = require('graphql-redis-subscriptions');

// Use Redis for multi-instance support
const pubsub = new RedisPubSub({
  publisher: new Redis(process.env.REDIS_URL),
  subscriber: new Redis(process.env.REDIS_URL)
});

const subscriptionTypeDefs = gql`
  type Subscription {
    orderUpdated(orderId: ID!): OrderUpdate!
    newOrderForAdmin: Order!
    inventoryAlert: InventoryAlert!
  }

  type OrderUpdate {
    orderId: ID!
    previousStatus: OrderStatus!
    newStatus: OrderStatus!
    order: Order!
    updatedAt: DateTime!
  }

  type InventoryAlert {
    productId: ID!
    productName: String!
    currentStock: Int!
    threshold: Int!
  }
`;

const subscriptionResolvers = {
  Subscription: {
    orderUpdated: {
      subscribe: withFilter(
        () => pubsub.asyncIterator('ORDER_UPDATED'),
        (payload, variables, context) => {
          // Only send to order owner or admin
          return payload.orderUpdated.orderId === variables.orderId &&
            (context.userId === payload.orderUpdated.userId ||
             context.role === 'ADMIN');
        }
      )
    },

    newOrderForAdmin: {
      subscribe: withFilter(
        () => pubsub.asyncIterator('NEW_ORDER'),
        (_, __, context) => context.role === 'ADMIN'
      )
    },

    inventoryAlert: {
      subscribe: withFilter(
        () => pubsub.asyncIterator('INVENTORY_ALERT'),
        (_, __, context) => ['ADMIN', 'MANAGER'].includes(context.role)
      )
    }
  }
};

// Publish when order status changes
async function publishOrderUpdate(previousOrder, updatedOrder) {
  await pubsub.publish('ORDER_UPDATED', {
    orderUpdated: {
      orderId: updatedOrder.id,
      userId: updatedOrder.userId,
      previousStatus: previousOrder.status,
      newStatus: updatedOrder.status,
      order: updatedOrder,
      updatedAt: new Date().toISOString()
    }
  });
}
```

---

## 7. Client Query Examples

```graphql
# Client queries ที่ใช้ Federation ได้อย่างไร้รอยต่อ

# Query 1: Get my profile with orders and reviews
query MyProfile {
  me {
    id
    name
    email
    avatar
    orders(filter: { status: DELIVERED }) {
      nodes {
        id
        totalAmount
        createdAt
        items {
          quantity
          product {
            id
            name
            price
            images { url }
          }
        }
      }
    }
    reviews {
      rating
      title
      product {
        name
      }
    }
  }
}

# Query 2: Product detail page
query ProductDetail($id: ID!) {
  product(id: $id) {
    id
    name
    description
    price
    inStock
    stockCount
    images { url alt isPrimary }
    reviews(pagination: { first: 10 }) {
      nodes {
        rating
        title
        body
        verified
        createdAt
        user {
          name
          avatar
        }
      }
      averageRating
      totalCount
    }
    category {
      name
      parent { name }
    }
  }
}

# Subscription: Watch order status
subscription WatchOrder($orderId: ID!) {
  orderUpdated(orderId: $orderId) {
    previousStatus
    newStatus
    order {
      status
      items {
        product { name }
        quantity
      }
    }
  }
}
```

---

## สรุป

```
GraphQL Federation Architecture:

Client
  ↓
Apollo Gateway (port 4000)
  ↓ Query Planning ↓ Parallel fetching
  ├── User Subgraph    (port 4001)
  ├── Product Subgraph (port 4002)
  ├── Order Subgraph   (port 4003)
  └── Review Subgraph  (port 4004)

Entity References (cross-service joins):
- User entity owned by user-service
- Extended by order-service (orders field)
- Extended by review-service (reviews field)
- Product entity owned by product-service
- Extended by review-service (reviews field)

Performance:
- DataLoader batches identical IDs
- Gateway executes parallel queries
- Response caching per query + user
- Persisted queries reduce bandwidth
```

**Next:** Part 26 - Real-time Features: WebSockets and Server-Sent Events

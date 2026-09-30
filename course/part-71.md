# Part 71: Microservices Testing Strategy

## บทนำ

Testing strategy ที่ดีสำหรับ microservices ต้องครอบคลุมทุกระดับ ตั้งแต่ unit tests ไปจนถึง end-to-end tests โดยเน้นที่ความเร็ว ความน่าเชื่อถือ และ coverage ที่มีความหมาย ใน Part นี้เราจะเรียนรู้วิธีสร้าง comprehensive testing strategy

---

## 1. Test Pyramid สำหรับ Microservices

```
                    /\
                   /  \
                  / E2E \          <-- น้อย แต่ครอบคลุม critical paths
                 /  Tests \
                /___________\
               /             \
              / Integration   \    <-- ปานกลาง
             /   Tests         \
            /___________________ \
           /                      \
          /    Unit Tests           \   <-- มาก เร็ว ถูก
         /____________________________\
```

```typescript
// jest.config.ts
import type { Config } from "jest";

const config: Config = {
  projects: [
    {
      displayName: "unit",
      testMatch: ["**/*.unit.test.ts"],
      transform: {
        "^.+\\.tsx?$": ["ts-jest", { tsconfig: "tsconfig.test.json" }],
      },
      testEnvironment: "node",
      collectCoverageFrom: ["src/**/*.ts", "!src/**/*.d.ts"],
      coverageThreshold: {
        global: {
          branches: 80,
          functions: 80,
          lines: 80,
          statements: 80,
        },
      },
    },
    {
      displayName: "integration",
      testMatch: ["**/*.integration.test.ts"],
      transform: {
        "^.+\\.tsx?$": ["ts-jest", {}],
      },
      testEnvironment: "node",
      globalSetup: "./test/setup/global-setup.ts",
      globalTeardown: "./test/setup/global-teardown.ts",
      setupFilesAfterFramework: ["./test/setup/jest-setup.ts"],
      testTimeout: 30000,
    },
    {
      displayName: "contract",
      testMatch: ["**/*.contract.test.ts"],
      transform: {
        "^.+\\.tsx?$": ["ts-jest", {}],
      },
      testEnvironment: "node",
      testTimeout: 60000,
    },
  ],
};

export default config;
```

---

## 2. Unit Tests

### Testing Domain Logic

```typescript
// domain/order.unit.test.ts
import { Order } from "../src/domain/order";
import { OrderItem } from "../src/domain/order-item";

describe("Order", () => {
  describe("create", () => {
    it("should create order with correct total amount", () => {
      const order = Order.create({
        id: "order-1",
        userId: "user-1",
        items: [
          { productId: "prod-1", quantity: 2, unitPrice: 100 },
          { productId: "prod-2", quantity: 1, unitPrice: 50 },
        ],
      });

      expect(order.totalAmount).toBe(250); // 2*100 + 1*50
    });

    it("should start with pending status", () => {
      const order = Order.create({
        id: "order-1",
        userId: "user-1",
        items: [{ productId: "prod-1", quantity: 1, unitPrice: 100 }],
      });

      expect(order.status).toBe("pending");
    });

    it("should throw when items is empty", () => {
      expect(() =>
        Order.create({ id: "order-1", userId: "user-1", items: [] })
      ).toThrow("Order must have at least one item");
    });

    it("should throw when quantity is zero or negative", () => {
      expect(() =>
        Order.create({
          id: "order-1",
          userId: "user-1",
          items: [{ productId: "prod-1", quantity: 0, unitPrice: 100 }],
        })
      ).toThrow("Quantity must be greater than 0");
    });
  });

  describe("confirm", () => {
    it("should change status to confirmed", () => {
      const order = Order.create({
        id: "order-1",
        userId: "user-1",
        items: [{ productId: "prod-1", quantity: 1, unitPrice: 100 }],
      });

      order.confirm();
      expect(order.status).toBe("confirmed");
    });

    it("should not allow confirming already confirmed order", () => {
      const order = Order.create({
        id: "order-1",
        userId: "user-1",
        items: [{ productId: "prod-1", quantity: 1, unitPrice: 100 }],
      });

      order.confirm();
      expect(() => order.confirm()).toThrow(
        "Cannot confirm order with status: confirmed"
      );
    });

    it("should emit OrderConfirmed event", () => {
      const order = Order.create({
        id: "order-1",
        userId: "user-1",
        items: [{ productId: "prod-1", quantity: 1, unitPrice: 100 }],
      });

      order.confirm();
      const events = order.getUncommittedEvents();
      expect(events).toHaveLength(1);
      expect(events[0].name).toBe("order.confirmed");
      expect(events[0].payload.orderId).toBe("order-1");
    });
  });
});

// services/user.service.unit.test.ts
import { UserService } from "../src/services/user.service";
import { IUserRepository } from "../src/interfaces";
import { IEmailService } from "../src/interfaces";

// Mock repository
const mockUserRepo: jest.Mocked<IUserRepository> = {
  findById: jest.fn(),
  findByEmail: jest.fn(),
  save: jest.fn(),
  delete: jest.fn(),
};

const mockEmailService: jest.Mocked<IEmailService> = {
  sendWelcomeEmail: jest.fn(),
  sendPasswordResetEmail: jest.fn(),
};

describe("UserService", () => {
  let userService: UserService;

  beforeEach(() => {
    jest.clearAllMocks();
    userService = new UserService(mockUserRepo, mockEmailService);
  });

  describe("createUser", () => {
    it("should create user successfully", async () => {
      mockUserRepo.findByEmail.mockResolvedValue(null);
      mockUserRepo.save.mockResolvedValue(undefined);
      mockEmailService.sendWelcomeEmail.mockResolvedValue(undefined);

      const result = await userService.createUser({
        id: "user-1",
        email: "test@example.com",
        name: "Test User",
      });

      expect(result.isOk()).toBe(true);
      if (result.isOk()) {
        expect(result.value.email).toBe("test@example.com");
      }

      expect(mockUserRepo.save).toHaveBeenCalledTimes(1);
      expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith(
        "test@example.com",
        "Test User"
      );
    });

    it("should return error when email already exists", async () => {
      mockUserRepo.findByEmail.mockResolvedValue({
        id: "existing-user",
        email: "test@example.com",
      } as any);

      const result = await userService.createUser({
        id: "user-1",
        email: "test@example.com",
        name: "Test User",
      });

      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error._tag).toBe("EmailAlreadyExistsError");
      }
      expect(mockUserRepo.save).not.toHaveBeenCalled();
    });

    it("should return error when name is too short", async () => {
      const result = await userService.createUser({
        id: "user-1",
        email: "test@example.com",
        name: "A",
      });

      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error._tag).toBe("ValidationError");
      }
    });

    it("should return DatabaseError when repository throws", async () => {
      mockUserRepo.findByEmail.mockResolvedValue(null);
      mockUserRepo.save.mockRejectedValue(new Error("Connection refused"));

      const result = await userService.createUser({
        id: "user-1",
        email: "test@example.com",
        name: "Test User",
      });

      expect(result.isErr()).toBe(true);
      if (result.isErr()) {
        expect(result.error._tag).toBe("DatabaseError");
      }
    });
  });
});
```

### Property-Based Testing

```typescript
// domain/pricing.property.test.ts
import fc from "fast-check";
import { calculateDiscount, applyTax } from "../src/domain/pricing";

describe("Pricing - Property Tests", () => {
  test("discount should never exceed original price", () => {
    fc.assert(
      fc.property(
        fc.float({ min: 0, max: 10000 }),
        fc.float({ min: 0, max: 1 }),
        (price, discountRate) => {
          const discount = calculateDiscount(price, discountRate);
          return discount <= price;
        }
      )
    );
  });

  test("tax should increase the price", () => {
    fc.assert(
      fc.property(
        fc.float({ min: 0.01, max: 10000 }),
        fc.float({ min: 0, max: 1 }),
        (price, taxRate) => {
          const taxedPrice = applyTax(price, taxRate);
          return taxedPrice >= price;
        }
      )
    );
  });

  test("total calculation should be associative", () => {
    fc.assert(
      fc.property(
        fc.array(fc.float({ min: 0, max: 100 }), { minLength: 1 }),
        (prices) => {
          const sum1 = prices.reduce((a, b) => a + b, 0);
          const sum2 = [...prices].reverse().reduce((a, b) => a + b, 0);
          return Math.abs(sum1 - sum2) < 0.0001;
        }
      )
    );
  });
});
```

---

## 3. Integration Tests ด้วย TestContainers

```typescript
// test/setup/global-setup.ts
import { GenericContainer, Network, StartedNetwork } from "testcontainers";
import { StartedPostgreSqlContainer, PostgreSqlContainer } from "@testcontainers/postgresql";
import { StartedRedisContainer, RedisContainer } from "@testcontainers/redis";

declare global {
  var __DB_CONTAINER__: StartedPostgreSqlContainer;
  var __REDIS_CONTAINER__: StartedRedisContainer;
  var __NETWORK__: StartedNetwork;
}

export default async function globalSetup() {
  const network = await new Network().start();
  global.__NETWORK__ = network;

  // Start PostgreSQL
  const dbContainer = await new PostgreSqlContainer("postgres:15-alpine")
    .withNetwork(network)
    .withNetworkAliases("postgres")
    .withDatabase("testdb")
    .withUsername("test")
    .withPassword("test")
    .withCommand(["postgres", "-c", "log_statement=all"])
    .start();

  global.__DB_CONTAINER__ = dbContainer;
  process.env.DATABASE_URL = dbContainer.getConnectionUri();

  // Start Redis
  const redisContainer = await new RedisContainer("redis:7-alpine")
    .withNetwork(network)
    .withNetworkAliases("redis")
    .start();

  global.__REDIS_CONTAINER__ = redisContainer;
  process.env.REDIS_URL = `redis://${redisContainer.getHost()}:${redisContainer.getPort()}`;

  // Run migrations
  const { MigrationRunner } = await import("../../src/database/migration-runner");
  const { Pool } = await import("pg");
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  const runner = new MigrationRunner(pool, "./src/database/migrations");
  await runner.migrate();
  await pool.end();

  console.log("Test containers started");
}

// test/setup/global-teardown.ts
export default async function globalTeardown() {
  await global.__DB_CONTAINER__?.stop();
  await global.__REDIS_CONTAINER__?.stop();
  await global.__NETWORK__?.stop();
  console.log("Test containers stopped");
}

// repositories/user.repository.integration.test.ts
import { Pool } from "pg";
import { PostgresUserRepository } from "../../src/repositories/postgres-user.repository";

describe("PostgresUserRepository (Integration)", () => {
  let pool: Pool;
  let repo: PostgresUserRepository;

  beforeAll(async () => {
    pool = new Pool({ connectionString: process.env.DATABASE_URL });
    repo = new PostgresUserRepository(pool);
  });

  afterAll(async () => {
    await pool.end();
  });

  afterEach(async () => {
    // Clean up between tests
    await pool.query("DELETE FROM users WHERE email LIKE 'test-%@test.com'");
  });

  describe("save and findById", () => {
    it("should persist and retrieve a user", async () => {
      const user = User.create({
        id: `user-${Date.now()}`,
        email: `test-${Date.now()}@test.com`,
        name: "Test User",
      });

      await repo.save(user);
      const found = await repo.findById(user.id);

      expect(found).not.toBeNull();
      expect(found?.email).toBe(user.email);
      expect(found?.name).toBe(user.name);
    });

    it("should return null for non-existent user", async () => {
      const found = await repo.findById("non-existent-id" as any);
      expect(found).toBeNull();
    });

    it("should update existing user", async () => {
      const user = User.create({
        id: `user-${Date.now()}`,
        email: `test-update-${Date.now()}@test.com`,
        name: "Original Name",
      });

      await repo.save(user);

      const updated = User.create({
        id: user.id,
        email: user.email,
        name: "Updated Name",
      });
      await repo.save(updated);

      const found = await repo.findById(user.id);
      expect(found?.name).toBe("Updated Name");
    });

    it("should handle concurrent saves without data corruption", async () => {
      const userId = `user-concurrent-${Date.now()}`;
      const baseUser = User.create({
        id: userId,
        email: `test-concurrent-${Date.now()}@test.com`,
        name: "Initial",
      });

      await repo.save(baseUser);

      // Concurrent updates
      await Promise.all([
        repo.save(User.create({ id: userId, email: baseUser.email, name: "Update 1" })),
        repo.save(User.create({ id: userId, email: baseUser.email, name: "Update 2" })),
      ]);

      const found = await repo.findById(userId as any);
      expect(found).not.toBeNull();
      // At least one of the updates should have succeeded
      expect(["Update 1", "Update 2"]).toContain(found?.name);
    });
  });
});
```

---

## 4. Consumer-Driven Contract Testing

```typescript
// contracts/user-service-consumer.contract.test.ts
import { Pact, Matchers } from "@pact-foundation/pact";
import path from "path";
import axios from "axios";

const { like, eachLike, term } = Matchers;

describe("UserService API Contract (Consumer)", () => {
  const provider = new Pact({
    consumer: "OrderService",
    provider: "UserService",
    port: 4000,
    log: path.resolve("./logs", "pact.log"),
    dir: path.resolve("./pacts"),
    logLevel: "WARN",
  });

  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());

  describe("GET /users/:id", () => {
    const userId = "user-123";

    beforeEach(() => {
      return provider.addInteraction({
        state: `user with id ${userId} exists`,
        uponReceiving: "a request to get a user by id",
        withRequest: {
          method: "GET",
          path: `/users/${userId}`,
          headers: {
            Authorization: like("Bearer token"),
            Accept: "application/json",
          },
        },
        willRespondWith: {
          status: 200,
          headers: {
            "Content-Type": "application/json",
          },
          body: {
            id: like(userId),
            email: term({
              generate: "john@example.com",
              matcher: "^[^@\\s]+@[^@\\s]+\\.[^@\\s]+$",
            }),
            name: like("John Doe"),
            createdAt: like("2024-01-01T00:00:00.000Z"),
          },
        },
      });
    });

    it("should return user data", async () => {
      const response = await axios.get(
        `http://localhost:4000/users/${userId}`,
        {
          headers: {
            Authorization: "Bearer test-token",
            Accept: "application/json",
          },
        }
      );

      expect(response.status).toBe(200);
      expect(response.data).toHaveProperty("id");
      expect(response.data).toHaveProperty("email");
      expect(response.data).toHaveProperty("name");
    });
  });

  describe("POST /users", () => {
    beforeEach(() => {
      return provider.addInteraction({
        state: "no user exists with email test@example.com",
        uponReceiving: "a request to create a user",
        withRequest: {
          method: "POST",
          path: "/users",
          headers: {
            "Content-Type": "application/json",
          },
          body: {
            email: "test@example.com",
            name: "Test User",
            password: like("securepassword123"),
          },
        },
        willRespondWith: {
          status: 201,
          headers: {
            "Content-Type": "application/json",
          },
          body: {
            id: like("user-456"),
            email: "test@example.com",
            name: "Test User",
          },
        },
      });
    });

    it("should create a new user", async () => {
      const response = await axios.post("http://localhost:4000/users", {
        email: "test@example.com",
        name: "Test User",
        password: "securepassword123",
      });

      expect(response.status).toBe(201);
      expect(response.data.email).toBe("test@example.com");
    });
  });
});

// Provider verification
// user-service/verify-pacts.ts
import { Verifier } from "@pact-foundation/pact";
import path from "path";

async function verifyPacts() {
  const opts = {
    provider: "UserService",
    providerBaseUrl: `http://localhost:${process.env.PORT ?? 3000}`,
    pactBrokerUrl: process.env.PACT_BROKER_URL,
    pactBrokerToken: process.env.PACT_BROKER_TOKEN,
    publishVerificationResult: process.env.CI === "true",
    providerVersion: process.env.GIT_COMMIT,
    providerVersionBranch: process.env.GIT_BRANCH,
    // Or use local pact files during development
    pactUrls: process.env.CI
      ? undefined
      : [path.resolve("./pacts/OrderService-UserService.json")],
    stateHandlers: {
      [`user with id user-123 exists`]: async () => {
        await testDb.query(`
          INSERT INTO users (id, email, name, created_at)
          VALUES ('user-123', 'john@example.com', 'John Doe', NOW())
          ON CONFLICT (id) DO NOTHING
        `);
      },
      "no user exists with email test@example.com": async () => {
        await testDb.query(
          "DELETE FROM users WHERE email = 'test@example.com'"
        );
      },
    },
  };

  const verifier = new Verifier(opts);
  await verifier.verifyProvider();
  console.log("Pact verification successful!");
}

verifyPacts().catch(console.error);
```

---

## 5. API Compatibility Testing

```typescript
// tests/api-compatibility.test.ts
import request from "supertest";
import { app } from "../../src/app";

describe("API Compatibility Tests", () => {
  describe("User API v1", () => {
    it("GET /api/v1/users/:id should return expected schema", async () => {
      const response = await request(app)
        .get("/api/v1/users/user-123")
        .set("Authorization", "Bearer valid-token")
        .expect(200);

      // Verify response schema
      expect(response.body).toMatchObject({
        id: expect.any(String),
        email: expect.stringMatching(/^[^\s@]+@[^\s@]+\.[^\s@]+$/),
        name: expect.any(String),
        createdAt: expect.any(String),
      });

      // Verify no extra sensitive fields
      expect(response.body).not.toHaveProperty("passwordHash");
      expect(response.body).not.toHaveProperty("internalId");
    });

    it("GET /api/v1/users should support pagination", async () => {
      const response = await request(app)
        .get("/api/v1/users?page=1&limit=10")
        .set("Authorization", "Bearer valid-token")
        .expect(200);

      expect(response.body).toMatchObject({
        items: expect.any(Array),
        total: expect.any(Number),
        page: 1,
        totalPages: expect.any(Number),
      });
    });

    it("POST /api/v1/users should return 201 with Location header", async () => {
      const response = await request(app)
        .post("/api/v1/users")
        .send({
          email: `test-${Date.now()}@example.com`,
          name: "Test User",
          password: "SecurePass123!",
        })
        .expect(201);

      expect(response.headers.location).toMatch(/\/api\/v1\/users\/.+/);
      expect(response.body).toHaveProperty("id");
    });

    it("should maintain backward compatibility for deprecated fields", async () => {
      const response = await request(app)
        .get("/api/v1/users/user-123")
        .set("Authorization", "Bearer valid-token")
        .expect(200);

      // These fields were in v1 and must remain for backward compatibility
      expect(response.body).toHaveProperty("id");
      expect(response.body).toHaveProperty("email");
      expect(response.body).toHaveProperty("name");
    });
  });
});

// Snapshot testing สำหรับ API responses
describe("User API Snapshot Tests", () => {
  it("GET /users/:id response structure should not change", async () => {
    const response = await request(app)
      .get("/api/v1/users/user-snapshot-test")
      .set("Authorization", "Bearer valid-token")
      .expect(200);

    // Normalize timestamps for consistent snapshots
    const normalized = {
      ...response.body,
      createdAt: "[DATE]",
      updatedAt: "[DATE]",
    };

    expect(normalized).toMatchSnapshot();
  });
});
```

---

## 6. Database Testing Patterns

```typescript
// test/database/transaction-test.helper.ts
import { Pool, PoolClient } from "pg";

/**
 * Run a test in a transaction that's always rolled back.
 * Ensures test isolation without needing to clean up manually.
 */
export function withinTransaction(
  pool: Pool,
  testFn: (client: PoolClient) => Promise<void>
): () => Promise<void> {
  return async () => {
    const client = await pool.connect();
    try {
      await client.query("BEGIN");
      await testFn(client);
    } finally {
      await client.query("ROLLBACK");
      client.release();
    }
  };
}

// Usage
describe("OrderRepository (within transactions)", () => {
  let pool: Pool;

  beforeAll(() => {
    pool = new Pool({ connectionString: process.env.DATABASE_URL });
  });

  afterAll(() => pool.end());

  it(
    "should save and retrieve order",
    withinTransaction(pool, async (client) => {
      // All queries use the same client/transaction
      await client.query(
        "INSERT INTO orders (id, user_id, total, status) VALUES ($1, $2, $3, $4)",
        ["order-test-1", "user-1", 100, "pending"]
      );

      const result = await client.query(
        "SELECT * FROM orders WHERE id = $1",
        ["order-test-1"]
      );

      expect(result.rows).toHaveLength(1);
      expect(result.rows[0].total).toBe("100");
      // Transaction rolls back after test — no cleanup needed!
    })
  );
});

// Database fixture helpers
class TestFixtures {
  constructor(private readonly pool: Pool) {}

  async createUser(overrides: Partial<UserFixture> = {}): Promise<UserFixture> {
    const id = `user-${Date.now()}-${Math.random().toString(36).slice(2)}`;
    const user = {
      id,
      email: `test-${id}@example.com`,
      name: "Test User",
      password_hash: "hashed_password",
      ...overrides,
    };

    await this.pool.query(
      `INSERT INTO users (id, email, name, password_hash)
       VALUES ($1, $2, $3, $4)`,
      [user.id, user.email, user.name, user.password_hash]
    );

    return user;
  }

  async createOrder(
    userId: string,
    overrides: Partial<OrderFixture> = {}
  ): Promise<OrderFixture> {
    const id = `order-${Date.now()}-${Math.random().toString(36).slice(2)}`;
    const order = {
      id,
      userId,
      total: 100,
      status: "pending",
      ...overrides,
    };

    await this.pool.query(
      `INSERT INTO orders (id, user_id, total, status)
       VALUES ($1, $2, $3, $4)`,
      [order.id, order.userId, order.total, order.status]
    );

    return order;
  }

  async cleanAll(): Promise<void> {
    await this.pool.query(
      "DELETE FROM order_items WHERE order_id LIKE 'order-%'"
    );
    await this.pool.query("DELETE FROM orders WHERE id LIKE 'order-%'");
    await this.pool.query("DELETE FROM users WHERE id LIKE 'user-%'");
  }
}
```

---

## 7. Mock vs Stub vs Spy

```typescript
// testing-doubles/examples.test.ts
import { jest } from "@jest/globals";

// --- STUB: ส่งคืนค่า fixed โดยไม่ track calls ---
const stubEmailService = {
  sendWelcomeEmail: async (_email: string, _name: string) => {
    // Do nothing — stub
  },
  sendPasswordResetEmail: async (_email: string, _token: string) => {
    // Do nothing — stub
  },
};

// --- MOCK: เหมือน Stub แต่ track calls ด้วย ---
const mockEmailService = {
  sendWelcomeEmail: jest.fn().mockResolvedValue(undefined),
  sendPasswordResetEmail: jest.fn().mockResolvedValue(undefined),
};

// --- SPY: wrap real implementation แต่ track calls ---
class RealEmailService {
  async sendWelcomeEmail(email: string, name: string): Promise<void> {
    console.log(`Sending welcome email to ${email}`);
    // Real implementation
  }
}
const realService = new RealEmailService();
const spySendWelcome = jest.spyOn(realService, "sendWelcomeEmail");

// Tests
describe("Mock vs Stub vs Spy comparison", () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it("stub: just provides test data, no verification", async () => {
    const service = new UserService(mockUserRepo, stubEmailService);
    await service.createUser({ id: "1", email: "a@b.com", name: "Test" });
    // Stub doesn't track calls — can't verify email was sent
  });

  it("mock: verify interactions", async () => {
    const service = new UserService(mockUserRepo, mockEmailService);
    mockUserRepo.findByEmail.mockResolvedValue(null);
    mockUserRepo.save.mockResolvedValue(undefined);

    await service.createUser({ id: "1", email: "a@b.com", name: "Test" });

    // Verify mock was called correctly
    expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledTimes(1);
    expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith(
      "a@b.com",
      "Test"
    );
  });

  it("spy: verify real implementation was called", async () => {
    await realService.sendWelcomeEmail("a@b.com", "Test");

    expect(spySendWelcome).toHaveBeenCalledTimes(1);
    expect(spySendWelcome).toHaveBeenCalledWith("a@b.com", "Test");
    // Real method was actually called
  });

  it("mock with specific return values", async () => {
    mockUserRepo.findById
      .mockResolvedValueOnce(null)      // First call returns null
      .mockResolvedValueOnce({ id: "1", email: "a@b.com" } as any); // Second returns user

    const first = await mockUserRepo.findById("user-1" as any);
    const second = await mockUserRepo.findById("user-1" as any);

    expect(first).toBeNull();
    expect(second).not.toBeNull();
  });

  it("mock to simulate failures", async () => {
    mockUserRepo.save.mockRejectedValue(new Error("Connection refused"));

    await expect(
      userService.createUser({ id: "1", email: "a@b.com", name: "Test" })
    ).rejects.toThrow("Connection refused");
  });
});

// Test with fake implementations (more realistic than mocks)
class FakeUserRepository implements IUserRepository {
  private readonly users = new Map<string, User>();

  async findById(id: UserId): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async findByEmail(email: Email): Promise<User | null> {
    return [...this.users.values()].find((u) => u.email === email) ?? null;
  }

  async save(user: User): Promise<void> {
    this.users.set(user.id, user);
  }

  async delete(id: UserId): Promise<void> {
    this.users.delete(id);
  }

  // Helper for tests
  getUserCount(): number {
    return this.users.size;
  }

  addUser(user: User): void {
    this.users.set(user.id, user);
  }
}

describe("UserService with Fake Repository", () => {
  let fakeRepo: FakeUserRepository;
  let service: UserService;

  beforeEach(() => {
    fakeRepo = new FakeUserRepository();
    service = new UserService(fakeRepo, stubEmailService);
  });

  it("should persist user in fake repo", async () => {
    await service.createUser({ id: "user-1", email: "a@b.com", name: "Test" });
    expect(fakeRepo.getUserCount()).toBe(1);
  });
});
```

---

## 8. Load Testing และ Performance Testing

```typescript
// tests/performance/order-service.perf.test.ts
import autocannon from "autocannon";

describe("Order Service Performance", () => {
  const BASE_URL = process.env.TEST_SERVICE_URL ?? "http://localhost:3000";

  it("GET /orders should handle 100 RPS with p99 < 200ms", async () => {
    const result = await autocannon({
      url: `${BASE_URL}/api/v1/orders`,
      connections: 10,
      duration: 10,
      headers: {
        authorization: "Bearer test-token",
      },
    });

    console.log(autocannon.printResult(result));

    expect(result.errors).toBe(0);
    expect(result.timeouts).toBe(0);
    expect(result.latency.p99).toBeLessThan(200);
    expect(result.requests.average).toBeGreaterThan(100);
  });

  it("POST /orders should handle 50 RPS with p99 < 500ms", async () => {
    const result = await autocannon({
      url: `${BASE_URL}/api/v1/orders`,
      method: "POST",
      connections: 5,
      duration: 10,
      headers: {
        "content-type": "application/json",
        authorization: "Bearer test-token",
      },
      body: JSON.stringify({
        items: [{ productId: "prod-1", quantity: 1 }],
      }),
    });

    expect(result.errors).toBe(0);
    expect(result.latency.p99).toBeLessThan(500);
  });
});
```

---

## 9. CI/CD Integration

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:unit -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

  contract-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run consumer contract tests
        run: npm run test:contract:consumer

      - name: Publish pacts to broker
        run: npm run pact:publish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          GIT_COMMIT: ${{ github.sha }}
          GIT_BRANCH: ${{ github.ref_name }}

  verify-pacts:
    runs-on: ubuntu-latest
    needs: [contract-tests]
    steps:
      - uses: actions/checkout@v4

      - name: Start service
        run: npm run start:test &

      - name: Verify pacts
        run: npm run pact:verify
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          GIT_COMMIT: ${{ github.sha }}
          GIT_BRANCH: ${{ github.ref_name }}
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Microservices Testing Strategy ที่ครอบคลุม:

1. **Test Pyramid** — Unit (มาก, เร็ว, ถูก) → Integration (ปานกลาง) → E2E (น้อย, ช้า, แพง)
2. **Unit Tests** — ทดสอบ domain logic และ service layer ด้วย mocks/stubs/fakes
3. **Property-Based Testing** — ทดสอบ properties ที่ควรเป็นจริงเสมอด้วย fast-check
4. **Integration Tests** — TestContainers สำหรับ realistic test environments
5. **Consumer-Driven Contract Testing** — Pact framework ป้องกัน API breaking changes
6. **API Compatibility Testing** — รับประกัน backward compatibility
7. **Database Testing** — Transaction rollback, fixtures, test isolation
8. **Mock vs Stub vs Spy** — เลือกใช้ test doubles ให้เหมาะสม
9. **Load Testing** — autocannon ทดสอบ performance targets

กุญแจสำคัญ: tests ควรเร็ว reliable และ meaningful — ทดสอบ behavior ไม่ใช่ implementation details

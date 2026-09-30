# Part 70: Data Consistency Patterns

## บทนำ

ใน distributed systems การ maintain data consistency เป็นเรื่องที่ท้าทาย เพราะเราต้องเลือกระหว่าง consistency, availability, และ partition tolerance (CAP theorem) ใน Part นี้เราจะเรียนรู้ patterns ต่างๆ สำหรับจัดการ data consistency ใน microservices

---

## 1. Eventual Consistency Strategies

Eventual consistency หมายความว่าระบบจะ consistent ในที่สุด แต่ไม่ได้ consistent ทันที

```typescript
// eventual-consistency/event-driven-sync.ts
import { Pool } from "pg";
import { Redis } from "ioredis";

// State machine สำหรับ order consistency
type OrderStatus =
  | "pending"
  | "inventory_reserved"
  | "payment_processing"
  | "payment_completed"
  | "fulfillment_started"
  | "shipped"
  | "delivered"
  | "failed"
  | "cancelled";

interface OrderStateTransition {
  fromStatus: OrderStatus;
  toStatus: OrderStatus;
  event: string;
  compensation?: string;
}

const orderStateTransitions: OrderStateTransition[] = [
  { fromStatus: "pending", toStatus: "inventory_reserved", event: "INVENTORY_RESERVED" },
  { fromStatus: "inventory_reserved", toStatus: "payment_processing", event: "PAYMENT_INITIATED" },
  { fromStatus: "payment_processing", toStatus: "payment_completed", event: "PAYMENT_COMPLETED" },
  { fromStatus: "payment_processing", toStatus: "failed", event: "PAYMENT_FAILED", compensation: "RELEASE_INVENTORY" },
  { fromStatus: "payment_completed", toStatus: "fulfillment_started", event: "FULFILLMENT_STARTED" },
  { fromStatus: "fulfillment_started", toStatus: "shipped", event: "ORDER_SHIPPED" },
  { fromStatus: "shipped", toStatus: "delivered", event: "ORDER_DELIVERED" },
];

class OrderStateMachine {
  private readonly validTransitions: Map<string, OrderStateTransition>;

  constructor() {
    this.validTransitions = new Map();
    for (const transition of orderStateTransitions) {
      this.validTransitions.set(
        `${transition.fromStatus}:${transition.event}`,
        transition
      );
    }
  }

  isValidTransition(currentStatus: OrderStatus, event: string): boolean {
    return this.validTransitions.has(`${currentStatus}:${event}`);
  }

  getNextStatus(currentStatus: OrderStatus, event: string): OrderStatus | null {
    const transition = this.validTransitions.get(`${currentStatus}:${event}`);
    return transition?.toStatus ?? null;
  }

  getCompensation(currentStatus: OrderStatus, event: string): string | undefined {
    const transition = this.validTransitions.get(`${currentStatus}:${event}`);
    return transition?.compensation;
  }
}

// Event store สำหรับ order events
class OrderEventStore {
  constructor(private readonly db: Pool) {}

  async appendEvent(
    orderId: string,
    event: string,
    payload: Record<string, unknown>
  ): Promise<void> {
    await this.db.query(
      `INSERT INTO order_events (order_id, event_type, payload, created_at)
       VALUES ($1, $2, $3, NOW())`,
      [orderId, event, JSON.stringify(payload)]
    );
  }

  async getEvents(orderId: string): Promise<OrderEvent[]> {
    const result = await this.db.query(
      `SELECT * FROM order_events WHERE order_id = $1 ORDER BY id ASC`,
      [orderId]
    );
    return result.rows;
  }

  // Rebuild state from events (Event Sourcing)
  async rebuildState(orderId: string): Promise<Order> {
    const events = await this.getEvents(orderId);
    return events.reduce((order, event) => applyEvent(order, event), createEmptyOrder(orderId));
  }
}

// Idempotent event processing
class IdempotentEventProcessor {
  constructor(
    private readonly redis: Redis,
    private readonly db: Pool
  ) {}

  async process(
    eventId: string,
    handler: () => Promise<void>
  ): Promise<boolean> {
    const lockKey = `processed:${eventId}`;

    // Check if already processed
    const alreadyProcessed = await this.redis.get(lockKey);
    if (alreadyProcessed) {
      console.log(`Event ${eventId} already processed, skipping`);
      return false;
    }

    // Process the event
    await handler();

    // Mark as processed (with TTL for cleanup)
    await this.redis.set(lockKey, "1", "EX", 86400); // 24 hours TTL

    return true;
  }
}
```

---

## 2. Conflict Resolution Strategies

### Last Write Wins (LWW)

```typescript
// conflict-resolution/lww.ts

interface VersionedDocument<T> {
  data: T;
  timestamp: number;  // Unix timestamp milliseconds
  nodeId: string;
  version: number;
}

class LWWRegister<T> {
  private current: VersionedDocument<T> | null = null;

  write(value: T, nodeId: string): VersionedDocument<T> {
    const doc: VersionedDocument<T> = {
      data: value,
      timestamp: Date.now(),
      nodeId,
      version: (this.current?.version ?? 0) + 1,
    };

    // Last write wins: higher timestamp wins
    if (!this.current || doc.timestamp > this.current.timestamp) {
      this.current = doc;
    } else if (
      doc.timestamp === this.current.timestamp &&
      doc.nodeId > this.current.nodeId  // Tiebreak by node ID
    ) {
      this.current = doc;
    }

    return doc;
  }

  read(): VersionedDocument<T> | null {
    return this.current;
  }

  merge(remote: VersionedDocument<T>): void {
    if (!this.current) {
      this.current = remote;
      return;
    }

    if (remote.timestamp > this.current.timestamp) {
      this.current = remote;
    } else if (
      remote.timestamp === this.current.timestamp &&
      remote.nodeId > this.current.nodeId
    ) {
      this.current = remote;
    }
  }
}

// ตัวอย่าง: User profile sync across regions
class UserProfileService {
  private readonly registers: Map<string, LWWRegister<UserProfile>>;

  constructor() {
    this.registers = new Map();
  }

  updateProfile(userId: string, profile: UserProfile, nodeId: string): void {
    let register = this.registers.get(userId);
    if (!register) {
      register = new LWWRegister<UserProfile>();
      this.registers.set(userId, register);
    }
    register.write(profile, nodeId);
  }

  getProfile(userId: string): UserProfile | null {
    return this.registers.get(userId)?.read()?.data ?? null;
  }

  // Merge updates from another node/region
  mergeFromRemote(userId: string, remoteDoc: VersionedDocument<UserProfile>): void {
    let register = this.registers.get(userId);
    if (!register) {
      register = new LWWRegister<UserProfile>();
      this.registers.set(userId, register);
    }
    register.merge(remoteDoc);
  }
}
```

### Vector Clocks

```typescript
// conflict-resolution/vector-clock.ts

type VectorClock = Map<string, number>;

function incrementClock(clock: VectorClock, nodeId: string): VectorClock {
  const newClock = new Map(clock);
  newClock.set(nodeId, (clock.get(nodeId) ?? 0) + 1);
  return newClock;
}

function mergeClock(a: VectorClock, b: VectorClock): VectorClock {
  const merged = new Map(a);
  for (const [nodeId, counter] of b) {
    merged.set(nodeId, Math.max(merged.get(nodeId) ?? 0, counter));
  }
  return merged;
}

type ClockComparison = "before" | "after" | "concurrent" | "equal";

function compareClock(a: VectorClock, b: VectorClock): ClockComparison {
  let aLessB = false;
  let bLessA = false;

  const allNodes = new Set([...a.keys(), ...b.keys()]);

  for (const node of allNodes) {
    const aVal = a.get(node) ?? 0;
    const bVal = b.get(node) ?? 0;
    if (aVal < bVal) aLessB = true;
    if (aVal > bVal) bLessA = true;
  }

  if (!aLessB && !bLessA) return "equal";
  if (aLessB && !bLessA) return "before";
  if (!aLessB && bLessA) return "after";
  return "concurrent"; // Conflict!
}

interface VectorClockEntry<T> {
  value: T;
  clock: VectorClock;
  nodeId: string;
}

class VectorClockStore<T> {
  private entries: VectorClockEntry<T>[] = [];
  private localClock: VectorClock = new Map();

  write(value: T, nodeId: string): VectorClockEntry<T> {
    this.localClock = incrementClock(this.localClock, nodeId);

    const entry: VectorClockEntry<T> = {
      value,
      clock: new Map(this.localClock),
      nodeId,
    };

    this.entries.push(entry);
    this.pruneSuperseded();
    return entry;
  }

  merge(remoteEntry: VectorClockEntry<T>): "accepted" | "rejected" | "conflict" {
    for (const existing of this.entries) {
      const comparison = compareClock(remoteEntry.clock, existing.clock);

      if (comparison === "before" || comparison === "equal") {
        return "rejected"; // We already have a newer version
      }
    }

    this.entries.push(remoteEntry);
    this.localClock = mergeClock(this.localClock, remoteEntry.clock);
    this.pruneSuperseded();

    if (this.entries.length > 1) {
      return "conflict"; // Multiple concurrent versions
    }

    return "accepted";
  }

  read(): T | T[] {
    if (this.entries.length === 0) throw new Error("No value");
    if (this.entries.length === 1) return this.entries[0].value;

    // Return all concurrent versions for application-level resolution
    return this.entries.map((e) => e.value);
  }

  hasConflict(): boolean {
    return this.entries.length > 1;
  }

  resolve(resolvedValue: T, nodeId: string): void {
    if (!this.hasConflict()) return;

    // Merge all clocks
    const mergedClock = this.entries.reduce(
      (acc, entry) => mergeClock(acc, entry.clock),
      new Map<string, number>()
    );

    // Increment local counter
    const newClock = incrementClock(mergedClock, nodeId);

    this.entries = [{ value: resolvedValue, clock: newClock, nodeId }];
    this.localClock = newClock;
  }

  private pruneSuperseded(): void {
    this.entries = this.entries.filter((entry) => {
      const isSuperseded = this.entries.some((other) => {
        if (other === entry) return false;
        return compareClock(entry.clock, other.clock) === "before";
      });
      return !isSuperseded;
    });
  }
}

// ตัวอย่างการใช้งาน
const store = new VectorClockStore<{ name: string; age: number }>();

// Node A writes
store.write({ name: "Alice", age: 30 }, "node-a");

// Node B writes concurrently (before seeing A's write)
const storeB = new VectorClockStore<{ name: string; age: 30 }>();
storeB.write({ name: "Alice", age: 31 }, "node-b");

// When they sync, detect conflict
const result = store.merge(storeB.read() as any);
if (result === "conflict") {
  const versions = store.read() as Array<{ name: string; age: number }>;
  // Application resolves: use highest age
  const resolved = versions.reduce((a, b) => a.age > b.age ? a : b);
  store.resolve(resolved, "node-c");
}
```

---

## 3. Read-Your-Writes Consistency

```typescript
// consistency/read-your-writes.ts
import { Redis } from "ioredis";
import { Pool } from "pg";

class ReadYourWritesConsistencyManager {
  private readonly TOKEN_TTL_SECONDS = 30;

  constructor(
    private readonly redis: Redis,
    private readonly primary: Pool,
    private readonly replicas: Pool[]
  ) {}

  // After a write, store a consistency token
  async afterWrite(userId: string, writeTimestamp: number): Promise<string> {
    const token = `${userId}:${writeTimestamp}`;
    await this.redis.setex(
      `consistency_token:${userId}`,
      this.TOKEN_TTL_SECONDS,
      writeTimestamp.toString()
    );
    return token;
  }

  // When reading, check if we need to read from primary
  async read<T>(
    userId: string,
    query: (db: Pool) => Promise<T>
  ): Promise<T> {
    const token = await this.redis.get(`consistency_token:${userId}`);

    if (token) {
      // User just wrote — read from primary to get latest data
      console.log(`Reading from primary for user ${userId} (consistency token active)`);
      return query(this.primary);
    }

    // Safe to read from replica
    const replica = this.getReplica();
    return query(replica);
  }

  private getReplica(): Pool {
    return this.replicas[Math.floor(Math.random() * this.replicas.length)];
  }
}

// HTTP middleware สำหรับ read-your-writes
class ConsistencyMiddleware {
  constructor(
    private readonly manager: ReadYourWritesConsistencyManager,
    private readonly redis: Redis
  ) {}

  // Middleware หลัง write operations
  writeMiddleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      await next();

      // After successful write (2xx), set consistency token
      if (res.statusCode >= 200 && res.statusCode < 300) {
        const userId = (req as any).user?.id;
        if (userId) {
          const token = await this.manager.afterWrite(userId, Date.now());
          res.setHeader("X-Consistency-Token", token);
        }
      }
    };
  }

  // Middleware สำหรับ read operations
  readMiddleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      const userId = (req as any).user?.id;
      const consistencyToken = req.headers["x-consistency-token"] as string;

      // Client can send back the token to ensure read-your-writes
      if (userId && consistencyToken) {
        await this.redis.setex(
          `consistency_token:${userId}`,
          30,
          consistencyToken.split(":")[1]
        );
      }

      await next();
    };
  }
}
```

---

## 4. Monotonic Reads

```typescript
// consistency/monotonic-reads.ts

// Monotonic reads: เห็น version ที่ >= ที่เคยเห็น
class MonotonicReadStore<T> {
  private readonly seenVersions: Map<string, number> = new Map();

  constructor(
    private readonly redis: Redis,
    private readonly primary: Pool,
    private readonly replicas: Pool[]
  ) {}

  async read(
    sessionId: string,
    resourceId: string,
    query: (db: Pool) => Promise<{ data: T; version: number }>
  ): Promise<T> {
    const lastSeenVersion = await this.getLastSeenVersion(sessionId, resourceId);

    // Try replicas first
    for (const replica of this.replicas) {
      try {
        const result = await query(replica);

        if (result.version >= lastSeenVersion) {
          // This replica is up-to-date enough
          await this.updateLastSeenVersion(sessionId, resourceId, result.version);
          return result.data;
        }
      } catch (error) {
        console.warn("Replica read failed, trying next...");
      }
    }

    // Fall back to primary
    const result = await query(this.primary);
    await this.updateLastSeenVersion(sessionId, resourceId, result.version);
    return result.data;
  }

  private async getLastSeenVersion(
    sessionId: string,
    resourceId: string
  ): Promise<number> {
    const key = `monotonic:${sessionId}:${resourceId}`;
    const value = await this.redis.get(key);
    return value ? parseInt(value) : 0;
  }

  private async updateLastSeenVersion(
    sessionId: string,
    resourceId: string,
    version: number
  ): Promise<void> {
    const key = `monotonic:${sessionId}:${resourceId}`;
    const current = await this.getLastSeenVersion(sessionId, resourceId);

    if (version > current) {
      await this.redis.setex(key, 3600, version.toString()); // 1 hour TTL
    }
  }
}
```

---

## 5. CRDTs (Conflict-free Replicated Data Types)

CRDTs คือ data structures ที่ merge ได้โดยอัตโนมัติโดยไม่เกิด conflicts

```typescript
// crdts/grow-only-set.ts (G-Set)
// G-Set: add-only set ที่ merge ได้โดยการ union

class GrowOnlySet<T> {
  private readonly elements: Set<T>;

  constructor(elements?: T[]) {
    this.elements = new Set(elements ?? []);
  }

  add(element: T): GrowOnlySet<T> {
    const newSet = new GrowOnlySet<T>([...this.elements]);
    newSet.elements.add(element);
    return newSet;
  }

  contains(element: T): boolean {
    return this.elements.has(element);
  }

  merge(other: GrowOnlySet<T>): GrowOnlySet<T> {
    // Union of both sets — always safe, no conflicts
    return new GrowOnlySet<T>([...this.elements, ...other.elements]);
  }

  toArray(): T[] {
    return [...this.elements];
  }

  size(): number {
    return this.elements.size;
  }
}

// 2P-Set (Two-Phase Set): supports removes
class TwoPhaseSet<T> {
  private readonly addSet: GrowOnlySet<T>;
  private readonly removeSet: GrowOnlySet<T>;

  constructor(
    addElements?: T[],
    removeElements?: T[]
  ) {
    this.addSet = new GrowOnlySet(addElements);
    this.removeSet = new GrowOnlySet(removeElements);
  }

  add(element: T): TwoPhaseSet<T> {
    return new TwoPhaseSet(
      [...this.addSet.toArray(), element],
      this.removeSet.toArray()
    );
  }

  remove(element: T): TwoPhaseSet<T> {
    if (!this.addSet.contains(element)) {
      throw new Error("Cannot remove element not in add set");
    }
    return new TwoPhaseSet(
      this.addSet.toArray(),
      [...this.removeSet.toArray(), element]
    );
  }

  contains(element: T): boolean {
    // Present if in add set but NOT in remove set
    return this.addSet.contains(element) && !this.removeSet.contains(element);
  }

  merge(other: TwoPhaseSet<T>): TwoPhaseSet<T> {
    return new TwoPhaseSet(
      this.addSet.merge(other.addSet).toArray(),
      this.removeSet.merge(other.removeSet).toArray()
    );
  }
}

// OR-Set (Observed-Remove Set): better semantics for adds and removes
class ORSet<T> {
  // Map from element to set of unique tags
  private readonly entries: Map<T, Set<string>>;
  private readonly tombstones: Map<T, Set<string>>;

  constructor() {
    this.entries = new Map();
    this.tombstones = new Map();
  }

  add(element: T): ORSet<T> {
    const newSet = this.clone();
    const tag = crypto.randomUUID();
    if (!newSet.entries.has(element)) {
      newSet.entries.set(element, new Set());
    }
    newSet.entries.get(element)!.add(tag);
    return newSet;
  }

  remove(element: T): ORSet<T> {
    if (!this.contains(element)) return this;

    const newSet = this.clone();
    const tags = this.entries.get(element) ?? new Set();

    // Tombstone all current tags for this element
    if (!newSet.tombstones.has(element)) {
      newSet.tombstones.set(element, new Set());
    }
    for (const tag of tags) {
      newSet.tombstones.get(element)!.add(tag);
    }

    return newSet;
  }

  contains(element: T): boolean {
    const tags = this.entries.get(element);
    const tombstoned = this.tombstones.get(element) ?? new Set();

    if (!tags || tags.size === 0) return false;

    // Contains if any tag is NOT tombstoned
    return [...tags].some((tag) => !tombstoned.has(tag));
  }

  merge(other: ORSet<T>): ORSet<T> {
    const merged = new ORSet<T>();

    // Merge entries
    for (const [elem, tags] of this.entries) {
      merged.entries.set(elem, new Set(tags));
    }
    for (const [elem, tags] of other.entries) {
      if (!merged.entries.has(elem)) {
        merged.entries.set(elem, new Set());
      }
      for (const tag of tags) {
        merged.entries.get(elem)!.add(tag);
      }
    }

    // Merge tombstones
    for (const [elem, tags] of this.tombstones) {
      merged.tombstones.set(elem, new Set(tags));
    }
    for (const [elem, tags] of other.tombstones) {
      if (!merged.tombstones.has(elem)) {
        merged.tombstones.set(elem, new Set());
      }
      for (const tag of tags) {
        merged.tombstones.get(elem)!.add(tag);
      }
    }

    return merged;
  }

  toArray(): T[] {
    return [...this.entries.keys()].filter((e) => this.contains(e));
  }

  private clone(): ORSet<T> {
    const newSet = new ORSet<T>();
    for (const [k, v] of this.entries) {
      newSet.entries.set(k, new Set(v));
    }
    for (const [k, v] of this.tombstones) {
      newSet.tombstones.set(k, new Set(v));
    }
    return newSet;
  }
}

// G-Counter CRDT: distributed counter
class GCounter {
  private readonly counts: Map<string, number>;

  constructor(private readonly nodeId: string) {
    this.counts = new Map();
    this.counts.set(nodeId, 0);
  }

  increment(amount: number = 1): GCounter {
    const newCounter = new GCounter(this.nodeId);
    for (const [k, v] of this.counts) {
      newCounter.counts.set(k, v);
    }
    newCounter.counts.set(
      this.nodeId,
      (this.counts.get(this.nodeId) ?? 0) + amount
    );
    return newCounter;
  }

  value(): number {
    let total = 0;
    for (const count of this.counts.values()) {
      total += count;
    }
    return total;
  }

  merge(other: GCounter): GCounter {
    const merged = new GCounter(this.nodeId);
    const allNodes = new Set([...this.counts.keys(), ...other.counts.keys()]);

    for (const nodeId of allNodes) {
      merged.counts.set(
        nodeId,
        Math.max(
          this.counts.get(nodeId) ?? 0,
          other.counts.get(nodeId) ?? 0
        )
      );
    }

    return merged;
  }
}

// PN-Counter: increment and decrement
class PNCounter {
  private pCounter: GCounter;
  private nCounter: GCounter;

  constructor(private readonly nodeId: string) {
    this.pCounter = new GCounter(nodeId);
    this.nCounter = new GCounter(nodeId);
  }

  increment(amount: number = 1): PNCounter {
    const newCounter = new PNCounter(this.nodeId);
    newCounter.pCounter = this.pCounter.increment(amount);
    newCounter.nCounter = this.nCounter;
    return newCounter;
  }

  decrement(amount: number = 1): PNCounter {
    const newCounter = new PNCounter(this.nodeId);
    newCounter.pCounter = this.pCounter;
    newCounter.nCounter = this.nCounter.increment(amount);
    return newCounter;
  }

  value(): number {
    return this.pCounter.value() - this.nCounter.value();
  }

  merge(other: PNCounter): PNCounter {
    const merged = new PNCounter(this.nodeId);
    merged.pCounter = this.pCounter.merge(other.pCounter);
    merged.nCounter = this.nCounter.merge(other.nCounter);
    return merged;
  }
}

// LWW-Element-Set: CRDT set with LWW semantics
interface LWWElement<T> {
  value: T;
  timestamp: number;
}

class LWWElementSet<T> {
  private readonly addTimestamps: Map<string, LWWElement<T>>;
  private readonly removeTimestamps: Map<string, number>;

  constructor() {
    this.addTimestamps = new Map();
    this.removeTimestamps = new Map();
  }

  add(key: string, value: T, timestamp: number = Date.now()): void {
    const existing = this.addTimestamps.get(key);
    if (!existing || timestamp > existing.timestamp) {
      this.addTimestamps.set(key, { value, timestamp });
    }
  }

  remove(key: string, timestamp: number = Date.now()): void {
    const existing = this.removeTimestamps.get(key);
    if (!existing || timestamp > existing) {
      this.removeTimestamps.set(key, timestamp);
    }
  }

  contains(key: string): boolean {
    const addEntry = this.addTimestamps.get(key);
    const removeTimestamp = this.removeTimestamps.get(key);

    if (!addEntry) return false;
    if (!removeTimestamp) return true;

    // Add wins on tie (or use remove-wins variant)
    return addEntry.timestamp >= removeTimestamp;
  }

  get(key: string): T | undefined {
    if (!this.contains(key)) return undefined;
    return this.addTimestamps.get(key)?.value;
  }

  merge(other: LWWElementSet<T>): LWWElementSet<T> {
    const merged = new LWWElementSet<T>();

    // Merge add timestamps
    for (const [key, entry] of this.addTimestamps) {
      merged.add(key, entry.value, entry.timestamp);
    }
    for (const [key, entry] of other.addTimestamps) {
      merged.add(key, entry.value, entry.timestamp);
    }

    // Merge remove timestamps
    for (const [key, ts] of this.removeTimestamps) {
      merged.remove(key, ts);
    }
    for (const [key, ts] of other.removeTimestamps) {
      merged.remove(key, ts);
    }

    return merged;
  }
}
```

---

## 6. Gossip Protocol

```typescript
// gossip/gossip-protocol.ts
interface NodeState {
  nodeId: string;
  data: Record<string, unknown>;
  version: number;
  timestamp: number;
}

interface GossipMessage {
  fromNode: string;
  states: NodeState[];
  timestamp: number;
}

class GossipNode {
  private readonly nodeId: string;
  private readonly knownNodes: Set<string>;
  private readonly states: Map<string, NodeState>;
  private gossipInterval?: NodeJS.Timer;

  constructor(nodeId: string, initialNodes: string[] = []) {
    this.nodeId = nodeId;
    this.knownNodes = new Set(initialNodes);
    this.states = new Map();

    // Initialize own state
    this.states.set(nodeId, {
      nodeId,
      data: {},
      version: 0,
      timestamp: Date.now(),
    });
  }

  updateLocalState(key: string, value: unknown): void {
    const currentState = this.states.get(this.nodeId)!;
    this.states.set(this.nodeId, {
      ...currentState,
      data: { ...currentState.data, [key]: value },
      version: currentState.version + 1,
      timestamp: Date.now(),
    });
  }

  // Fan-out gossip: send to random subset of nodes
  createGossipMessage(): GossipMessage {
    return {
      fromNode: this.nodeId,
      states: [...this.states.values()],
      timestamp: Date.now(),
    };
  }

  // Process incoming gossip
  receiveGossip(message: GossipMessage): void {
    let updated = false;

    for (const remoteState of message.states) {
      const localState = this.states.get(remoteState.nodeId);

      if (!localState || remoteState.version > localState.version) {
        this.states.set(remoteState.nodeId, remoteState);
        this.knownNodes.add(remoteState.nodeId);
        updated = true;
      }
    }

    if (updated) {
      this.notifyStateChange();
    }
  }

  getClusterState(): Map<string, NodeState> {
    return new Map(this.states);
  }

  // Get nodes that haven't updated recently (possible failures)
  getStaleNodes(maxAgeMs: number = 30000): string[] {
    const now = Date.now();
    return [...this.states.values()]
      .filter((s) => s.nodeId !== this.nodeId && now - s.timestamp > maxAgeMs)
      .map((s) => s.nodeId);
  }

  startGossiping(intervalMs: number = 1000): void {
    this.gossipInterval = setInterval(async () => {
      await this.gossipToRandomNodes();
    }, intervalMs);
  }

  stopGossiping(): void {
    if (this.gossipInterval) {
      clearInterval(this.gossipInterval);
    }
  }

  private async gossipToRandomNodes(fanout: number = 3): Promise<void> {
    const nodes = [...this.knownNodes].filter((n) => n !== this.nodeId);
    const selected = this.selectRandom(nodes, fanout);

    const message = this.createGossipMessage();

    for (const nodeId of selected) {
      try {
        await this.sendGossip(nodeId, message);
      } catch (error) {
        console.warn(`Failed to gossip to ${nodeId}:`, error);
      }
    }
  }

  private selectRandom<T>(items: T[], count: number): T[] {
    const shuffled = [...items].sort(() => Math.random() - 0.5);
    return shuffled.slice(0, count);
  }

  private async sendGossip(
    targetNodeId: string,
    message: GossipMessage
  ): Promise<void> {
    // In a real system, this would use HTTP, gRPC, or message queue
    console.log(`${this.nodeId} gossiping to ${targetNodeId}`, message);
  }

  private notifyStateChange(): void {
    console.log(`${this.nodeId}: cluster state updated`);
  }
}

// ตัวอย่าง: Service Discovery ด้วย Gossip
class ServiceRegistry {
  private readonly gossipNode: GossipNode;

  constructor(nodeId: string, peers: string[]) {
    this.gossipNode = new GossipNode(nodeId, peers);
    this.gossipNode.startGossiping(2000); // gossip every 2 seconds
  }

  registerService(serviceName: string, endpoint: string, port: number): void {
    this.gossipNode.updateLocalState(`service:${serviceName}`, {
      endpoint,
      port,
      registeredAt: Date.now(),
      healthy: true,
    });
  }

  discoverService(serviceName: string): ServiceEndpoint[] {
    const clusterState = this.gossipNode.getClusterState();
    const endpoints: ServiceEndpoint[] = [];

    for (const [, nodeState] of clusterState) {
      const serviceData = nodeState.data[`service:${serviceName}`] as any;
      if (serviceData?.healthy) {
        endpoints.push({
          endpoint: serviceData.endpoint,
          port: serviceData.port,
          nodeId: nodeState.nodeId,
        });
      }
    }

    return endpoints;
  }

  markUnhealthy(serviceName: string): void {
    this.gossipNode.updateLocalState(`service:${serviceName}`, {
      healthy: false,
      markedAt: Date.now(),
    });
  }
}
```

---

## 7. Change Data Capture (CDC)

```typescript
// cdc/debezium-consumer.ts
import { Kafka } from "kafkajs";

interface DebeziumEvent {
  before: Record<string, unknown> | null;
  after: Record<string, unknown> | null;
  source: {
    version: string;
    connector: string;
    name: string;
    ts_ms: number;
    db: string;
    table: string;
    txId: number;
    lsn: number;
  };
  op: "c" | "u" | "d" | "r";  // create, update, delete, read (snapshot)
  ts_ms: number;
}

class CDCConsumer {
  private readonly kafka: Kafka;

  constructor(brokers: string[]) {
    this.kafka = new Kafka({
      clientId: "cdc-consumer",
      brokers,
    });
  }

  async consume(
    tables: string[],
    handlers: {
      onCreate?: (data: Record<string, unknown>, source: DebeziumEvent["source"]) => Promise<void>;
      onUpdate?: (before: Record<string, unknown>, after: Record<string, unknown>, source: DebeziumEvent["source"]) => Promise<void>;
      onDelete?: (data: Record<string, unknown>, source: DebeziumEvent["source"]) => Promise<void>;
    }
  ): Promise<void> {
    const consumer = this.kafka.consumer({ groupId: "cdc-processor" });
    await consumer.connect();

    const topics = tables.map((t) => `dbserver.public.${t}`);
    await consumer.subscribe({ topics, fromBeginning: false });

    await consumer.run({
      eachMessage: async ({ message }) => {
        if (!message.value) return;

        const event: DebeziumEvent = JSON.parse(message.value.toString());

        try {
          switch (event.op) {
            case "c":
              if (handlers.onCreate && event.after) {
                await handlers.onCreate(event.after, event.source);
              }
              break;
            case "u":
              if (handlers.onUpdate && event.before && event.after) {
                await handlers.onUpdate(event.before, event.after, event.source);
              }
              break;
            case "d":
              if (handlers.onDelete && event.before) {
                await handlers.onDelete(event.before, event.source);
              }
              break;
          }
        } catch (error) {
          console.error("Failed to process CDC event:", error);
          // DLQ handling
          await this.sendToDLQ(message, error as Error);
        }
      },
    });
  }

  private async sendToDLQ(message: any, error: Error): Promise<void> {
    const producer = this.kafka.producer();
    await producer.connect();
    await producer.send({
      topic: "cdc-dlq",
      messages: [
        {
          value: JSON.stringify({
            originalMessage: message,
            error: error.message,
            timestamp: Date.now(),
          }),
        },
      ],
    });
    await producer.disconnect();
  }
}

// Sync inventory service สำหรับ search index
const cdcConsumer = new CDCConsumer(["kafka:9092"]);

await cdcConsumer.consume(["products", "inventory"], {
  onCreate: async (data) => {
    await searchIndex.index("products", data.id as string, {
      name: data.name,
      description: data.description,
      price: data.price,
      inStock: (data.stock_quantity as number) > 0,
    });
  },
  onUpdate: async (_before, after) => {
    await searchIndex.update("products", after.id as string, {
      name: after.name,
      price: after.price,
      inStock: (after.stock_quantity as number) > 0,
    });
  },
  onDelete: async (before) => {
    await searchIndex.delete("products", before.id as string);
  },
});
```

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Data Consistency Patterns ที่สำคัญ:

1. **Eventual Consistency** — ยอมรับ temporary inconsistency แลกกับ availability และ performance สูงขึ้น
2. **Last Write Wins (LWW)** — conflict resolution แบบง่าย เหมาะกับ profile updates
3. **Vector Clocks** — ตรวจจับ concurrent writes และระบุ conflicts อย่างแม่นยำ
4. **Read-Your-Writes** — รับประกันว่า user เห็น writes ของตัวเองทันที
5. **Monotonic Reads** — รับประกันว่า version ไม่ถอยหลัง
6. **CRDTs** — data structures ที่ merge ได้อัตโนมัติ: G-Set, 2P-Set, OR-Set, G-Counter, PN-Counter
7. **Gossip Protocol** — distributed state sharing แบบ decentralized
8. **Change Data Capture (CDC)** — sync data changes ข้าม services ผ่าน event stream

Key insight: ไม่มี consistency model ที่ดีที่สุดสำหรับทุก use case — ต้องเลือกตาม requirements ของ business logic และยอมรับ trade-offs ที่เหมาะสม

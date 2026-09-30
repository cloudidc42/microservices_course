# Part 47: Database Sharding และ Partitioning

## บทนำ

เมื่อข้อมูลมีปริมาณมากจนฐานข้อมูลเดียวไม่เพียงพอ เราต้องการ Database Sharding และ Partitioning ในบทนี้จะเรียนรู้ Horizontal Sharding Strategies, Consistent Hashing Ring, PostgreSQL Table Partitioning, Cross-Shard Queries, Citus Extension, Shard Migration และ Rebalancing

---

## 1. โครงสร้างโปรเจกต์

```
database-sharding/
├── src/
│   ├── sharding/
│   │   ├── consistent-hash.ts
│   │   ├── shard-manager.ts
│   │   ├── shard-router.ts
│   │   └── cross-shard-query.ts
│   ├── partitioning/
│   │   ├── partition-manager.ts
│   │   └── migrations/
│   ├── services/
│   │   ├── user.service.ts
│   │   └── order.service.ts
│   └── config/
│       └── shards.config.ts
├── migrations/
│   ├── 001_create_partitioned_orders.sql
│   ├── 002_create_partitioned_users.sql
│   └── 003_setup_citus.sql
├── docker-compose.yml
└── k8s/
    └── postgres-cluster.yaml
```

---

## 2. Consistent Hashing Ring

```typescript
// src/sharding/consistent-hash.ts
import { createHash } from 'crypto';

interface VirtualNode {
  shardId: string;
  virtualNodeId: string;
  hash: number;
}

interface ShardInfo {
  id: string;
  host: string;
  port: number;
  database: string;
  weight: number; // สำหรับ weighted distribution
  isActive: boolean;
  tags: string[];
}

export class ConsistentHashRing {
  private ring: VirtualNode[] = [];
  private shards: Map<string, ShardInfo> = new Map();
  private readonly virtualNodesPerShard: number;

  constructor(virtualNodesPerShard: number = 150) {
    this.virtualNodesPerShard = virtualNodesPerShard;
  }

  // เพิ่ม shard เข้าไปใน ring
  addShard(shard: ShardInfo): void {
    this.shards.set(shard.id, shard);

    // สร้าง virtual nodes ตาม weight
    const numNodes = Math.floor(this.virtualNodesPerShard * shard.weight);
    
    for (let i = 0; i < numNodes; i++) {
      const virtualNodeId = `${shard.id}:vn${i}`;
      const hash = this.hashKey(virtualNodeId);
      
      this.ring.push({
        shardId: shard.id,
        virtualNodeId,
        hash,
      });
    }

    // เรียงลำดับ ring ตาม hash
    this.ring.sort((a, b) => a.hash - b.hash);
  }

  // ลบ shard ออกจาก ring
  removeShard(shardId: string): void {
    this.shards.delete(shardId);
    this.ring = this.ring.filter((node) => node.shardId !== shardId);
  }

  // หา shard สำหรับ key ที่กำหนด
  getShard(key: string): ShardInfo {
    if (this.ring.length === 0) {
      throw new Error('No shards available in the ring');
    }

    const hash = this.hashKey(key);
    
    // หา virtual node แรกที่ hash >= hash ของ key
    let low = 0;
    let high = this.ring.length - 1;
    
    while (low < high) {
      const mid = Math.floor((low + high) / 2);
      if (this.ring[mid].hash < hash) {
        low = mid + 1;
      } else {
        high = mid;
      }
    }
    
    // Wrap around ถ้าเกิน ring
    const nodeIndex = this.ring[low].hash >= hash ? low : 0;
    const shardId = this.ring[nodeIndex].shardId;
    
    const shard = this.shards.get(shardId);
    if (!shard) {
      throw new Error(`Shard ${shardId} not found`);
    }
    
    if (!shard.isActive) {
      // หา shard ถัดไปที่ active
      return this.getNextActiveShard(nodeIndex);
    }
    
    return shard;
  }

  // หา N shards สำหรับ replication
  getReplicaShards(key: string, count: number): ShardInfo[] {
    const hash = this.hashKey(key);
    const results: ShardInfo[] = [];
    const seenShards = new Set<string>();
    
    let startIndex = this.ring.findIndex((node) => node.hash >= hash);
    if (startIndex === -1) startIndex = 0;
    
    let i = startIndex;
    while (results.length < count) {
      const shardId = this.ring[i % this.ring.length].shardId;
      
      if (!seenShards.has(shardId)) {
        const shard = this.shards.get(shardId);
        if (shard?.isActive) {
          results.push(shard);
          seenShards.add(shardId);
        }
      }
      
      i++;
      if (i - startIndex >= this.ring.length) break;
    }
    
    return results;
  }

  // คำนวณการกระจาย key ไปยัง shards (สำหรับ monitoring)
  getDistribution(sampleSize: number = 10000): Map<string, number> {
    const distribution = new Map<string, number>();
    
    for (const shardId of this.shards.keys()) {
      distribution.set(shardId, 0);
    }
    
    for (let i = 0; i < sampleSize; i++) {
      const key = `test-key-${i}`;
      const shard = this.getShard(key);
      distribution.set(shard.id, (distribution.get(shard.id) || 0) + 1);
    }
    
    return distribution;
  }

  // แสดง ring statistics
  getRingStats(): {
    totalVirtualNodes: number;
    shardCount: number;
    distribution: Record<string, { virtualNodes: number; percentage: string }>;
  } {
    const shardVNodeCount = new Map<string, number>();
    
    for (const node of this.ring) {
      shardVNodeCount.set(
        node.shardId,
        (shardVNodeCount.get(node.shardId) || 0) + 1,
      );
    }
    
    const distribution: Record<string, { virtualNodes: number; percentage: string }> = {};
    
    for (const [shardId, count] of shardVNodeCount) {
      distribution[shardId] = {
        virtualNodes: count,
        percentage: ((count / this.ring.length) * 100).toFixed(2) + '%',
      };
    }
    
    return {
      totalVirtualNodes: this.ring.length,
      shardCount: this.shards.size,
      distribution,
    };
  }

  private hashKey(key: string): number {
    const hash = createHash('sha256').update(key).digest('hex');
    // แปลง hex เป็น number (ใช้แค่ 32 bits แรก)
    return parseInt(hash.substring(0, 8), 16);
  }

  private getNextActiveShard(startIndex: number): ShardInfo {
    for (let i = 1; i <= this.ring.length; i++) {
      const index = (startIndex + i) % this.ring.length;
      const shard = this.shards.get(this.ring[index].shardId);
      if (shard?.isActive) return shard;
    }
    throw new Error('No active shards available');
  }
}
```

---

## 3. Shard Manager

```typescript
// src/sharding/shard-manager.ts
import { Pool, PoolClient } from 'pg';
import { ConsistentHashRing } from './consistent-hash';

interface ShardConnection {
  pool: Pool;
  shardId: string;
}

interface ShardConfig {
  id: string;
  host: string;
  port: number;
  database: string;
  username: string;
  password: string;
  maxConnections: number;
  weight: number;
}

export class ShardManager {
  private connections: Map<string, ShardConnection> = new Map();
  private ring: ConsistentHashRing;
  private configs: Map<string, ShardConfig> = new Map();

  constructor(private readonly shardConfigs: ShardConfig[]) {
    this.ring = new ConsistentHashRing(150);
    this.initializeShards();
  }

  private initializeShards(): void {
    for (const config of this.shardConfigs) {
      const pool = new Pool({
        host: config.host,
        port: config.port,
        database: config.database,
        user: config.username,
        password: config.password,
        max: config.maxConnections,
        idleTimeoutMillis: 30000,
        connectionTimeoutMillis: 5000,
        ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: true } : false,
      });

      // Monitor pool events
      pool.on('error', (err) => {
        console.error(`Pool error on shard ${config.id}:`, err);
      });

      pool.on('connect', () => {
        console.debug(`New connection to shard ${config.id}`);
      });

      this.connections.set(config.id, { pool, shardId: config.id });
      this.configs.set(config.id, config);

      this.ring.addShard({
        id: config.id,
        host: config.host,
        port: config.port,
        database: config.database,
        weight: config.weight,
        isActive: true,
        tags: [],
      });
    }
  }

  // ดึง connection สำหรับ sharding key
  async getConnectionForKey(shardKey: string): Promise<PoolClient> {
    const shard = this.ring.getShard(shardKey);
    return this.getConnectionForShard(shard.id);
  }

  // ดึง connection สำหรับ shard ID โดยตรง
  async getConnectionForShard(shardId: string): Promise<PoolClient> {
    const connection = this.connections.get(shardId);
    if (!connection) {
      throw new Error(`Shard ${shardId} not found`);
    }
    return connection.pool.connect();
  }

  // ดึง connections ทั้งหมด (สำหรับ cross-shard queries)
  async getAllConnections(): Promise<{ shardId: string; client: PoolClient }[]> {
    const connections: { shardId: string; client: PoolClient }[] = [];
    
    for (const [shardId, connection] of this.connections) {
      const client = await connection.pool.connect();
      connections.push({ shardId, client });
    }
    
    return connections;
  }

  // Execute query บน shard ที่ถูกต้องตาม key
  async executeOnShard<T>(
    shardKey: string,
    query: string,
    params: unknown[] = [],
  ): Promise<T[]> {
    const client = await this.getConnectionForKey(shardKey);
    
    try {
      const result = await client.query<T>(query, params);
      return result.rows;
    } finally {
      client.release();
    }
  }

  // Execute transaction บน shard
  async executeTransactionOnShard<T>(
    shardKey: string,
    callback: (client: PoolClient) => Promise<T>,
  ): Promise<T> {
    const client = await this.getConnectionForKey(shardKey);
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  // ตรวจสอบสุขภาพของทุก shards
  async healthCheck(): Promise<Map<string, boolean>> {
    const results = new Map<string, boolean>();
    
    await Promise.allSettled(
      Array.from(this.connections.entries()).map(async ([shardId, conn]) => {
        try {
          const client = await conn.pool.connect();
          await client.query('SELECT 1');
          client.release();
          results.set(shardId, true);
        } catch {
          results.set(shardId, false);
        }
      }),
    );
    
    return results;
  }

  // ดึง statistics ของแต่ละ shard pool
  getPoolStats(): Record<string, { total: number; idle: number; waiting: number }> {
    const stats: Record<string, { total: number; idle: number; waiting: number }> = {};
    
    for (const [shardId, conn] of this.connections) {
      stats[shardId] = {
        total: conn.pool.totalCount,
        idle: conn.pool.idleCount,
        waiting: conn.pool.waitingCount,
      };
    }
    
    return stats;
  }

  // ปิด connections ทั้งหมด
  async shutdown(): Promise<void> {
    const closePromises = Array.from(this.connections.values()).map((conn) =>
      conn.pool.end(),
    );
    await Promise.all(closePromises);
  }
}
```

---

## 4. Cross-Shard Queries

```typescript
// src/sharding/cross-shard-query.ts
import { ShardManager } from './shard-manager';
import { PoolClient } from 'pg';

interface CrossShardQueryOptions {
  timeout?: number;
  maxParallelism?: number;
  consistencyLevel?: 'eventual' | 'strong';
}

interface ShardResult<T> {
  shardId: string;
  rows: T[];
  error?: Error;
}

export class CrossShardQueryExecutor {
  constructor(private readonly shardManager: ShardManager) {}

  // Execute query บนทุก shards พร้อมกัน
  async scatter<T>(
    query: string,
    params: unknown[] = [],
    options: CrossShardQueryOptions = {},
  ): Promise<T[]> {
    const { timeout = 30000, maxParallelism = 10 } = options;
    
    const connections = await this.shardManager.getAllConnections();
    
    // ทำงานเป็น batches ตาม maxParallelism
    const results: T[] = [];
    
    for (let i = 0; i < connections.length; i += maxParallelism) {
      const batch = connections.slice(i, i + maxParallelism);
      
      const batchResults = await Promise.allSettled(
        batch.map(async ({ shardId, client }) => {
          try {
            const timeoutPromise = new Promise<never>((_, reject) =>
              setTimeout(() => reject(new Error(`Shard ${shardId} timeout`)), timeout),
            );
            
            const queryPromise = client.query<T>(query, params);
            const result = await Promise.race([queryPromise, timeoutPromise]);
            
            return { shardId, rows: result.rows };
          } finally {
            client.release();
          }
        }),
      );
      
      for (const result of batchResults) {
        if (result.status === 'fulfilled') {
          results.push(...result.value.rows);
        } else {
          console.error('Shard query failed:', result.reason);
        }
      }
    }
    
    return results;
  }

  // Gather และ aggregate ผลจากทุก shards
  async scatterGather<T, R>(
    query: string,
    params: unknown[],
    aggregator: (results: T[]) => R,
    options: CrossShardQueryOptions = {},
  ): Promise<R> {
    const allResults = await this.scatter<T>(query, params, options);
    return aggregator(allResults);
  }

  // COUNT แบบ cross-shard
  async count(
    table: string,
    whereClause: string = '',
    params: unknown[] = [],
  ): Promise<number> {
    const query = `SELECT COUNT(*) as count FROM ${table} ${whereClause ? 'WHERE ' + whereClause : ''}`;
    
    return this.scatterGather<{ count: string }, number>(
      query,
      params,
      (results) => results.reduce((sum, r) => sum + parseInt(r.count), 0),
    );
  }

  // SUM แบบ cross-shard
  async sum(
    table: string,
    column: string,
    whereClause: string = '',
    params: unknown[] = [],
  ): Promise<number> {
    const query = `SELECT SUM(${column}) as total FROM ${table} ${whereClause ? 'WHERE ' + whereClause : ''}`;
    
    return this.scatterGather<{ total: string | null }, number>(
      query,
      params,
      (results) => results.reduce((sum, r) => sum + (parseFloat(r.total || '0') || 0), 0),
    );
  }

  // Paginated cross-shard query (ซับซ้อนกว่า single-shard)
  async paginatedQuery<T extends { id: string; created_at: Date }>(
    query: string,
    params: unknown[],
    page: number,
    pageSize: number,
    orderBy: keyof T = 'created_at' as keyof T,
  ): Promise<{ data: T[]; total: number; page: number; pageSize: number }> {
    // ดึงข้อมูลทั้งหมดก่อน (แนะนำให้ใช้ cursor-based pagination แทน)
    const allResults = await this.scatter<T>(query, params);
    
    // Sort ทั้งหมด
    allResults.sort((a, b) => {
      const aVal = a[orderBy];
      const bVal = b[orderBy];
      
      if (aVal instanceof Date && bVal instanceof Date) {
        return bVal.getTime() - aVal.getTime();
      }
      
      return String(bVal).localeCompare(String(aVal));
    });
    
    const total = allResults.length;
    const start = (page - 1) * pageSize;
    const data = allResults.slice(start, start + pageSize);
    
    return { data, total, page, pageSize };
  }

  // Distributed JOIN (ทำ join ใน application layer)
  async distributedJoin<T, U, R>(
    primaryQuery: string,
    primaryParams: unknown[],
    foreignKeyField: keyof T,
    secondaryQueryBuilder: (ids: string[]) => { query: string; params: unknown[] },
    joinField: keyof U,
    merger: (primary: T, secondary: U | undefined) => R,
  ): Promise<R[]> {
    // ดึงข้อมูล primary
    const primaryResults = await this.scatter<T>(primaryQuery, primaryParams);
    
    // รวม foreign keys
    const foreignKeys = [
      ...new Set(primaryResults.map((r) => String(r[foreignKeyField]))),
    ];
    
    if (foreignKeys.length === 0) return [];
    
    // ดึงข้อมูล secondary
    const { query: secondaryQuery, params: secondaryParams } =
      secondaryQueryBuilder(foreignKeys);
    const secondaryResults = await this.scatter<U>(secondaryQuery, secondaryParams);
    
    // สร้าง lookup map
    const secondaryMap = new Map(
      secondaryResults.map((r) => [String(r[joinField]), r]),
    );
    
    // Merge results
    return primaryResults.map((primary) =>
      merger(primary, secondaryMap.get(String(primary[foreignKeyField]))),
    );
  }
}
```

---

## 5. PostgreSQL Table Partitioning

### 5.1 Range Partitioning สำหรับ Orders

```sql
-- migrations/001_create_partitioned_orders.sql

-- สร้าง parent table ที่ partitioned ตาม created_at (Range Partitioning)
CREATE TABLE orders (
    id          UUID NOT NULL DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL,
    status      VARCHAR(50) NOT NULL DEFAULT 'pending',
    total       DECIMAL(12, 2) NOT NULL,
    currency    CHAR(3) NOT NULL DEFAULT 'THB',
    items       JSONB NOT NULL DEFAULT '[]',
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (id, created_at)  -- partition key ต้องอยู่ใน PK
) PARTITION BY RANGE (created_at);

-- สร้าง partitions รายเดือน
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

CREATE TABLE orders_2024_04 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-05-01');

CREATE TABLE orders_2024_05 PARTITION OF orders
    FOR VALUES FROM ('2024-05-01') TO ('2024-06-01');

CREATE TABLE orders_2024_06 PARTITION OF orders
    FOR VALUES FROM ('2024-06-01') TO ('2024-07-01');

CREATE TABLE orders_2024_07 PARTITION OF orders
    FOR VALUES FROM ('2024-07-01') TO ('2024-08-01');

CREATE TABLE orders_2024_08 PARTITION OF orders
    FOR VALUES FROM ('2024-08-01') TO ('2024-09-01');

CREATE TABLE orders_2024_09 PARTITION OF orders
    FOR VALUES FROM ('2024-09-01') TO ('2024-10-01');

CREATE TABLE orders_2024_10 PARTITION OF orders
    FOR VALUES FROM ('2024-10-01') TO ('2024-11-01');

CREATE TABLE orders_2024_11 PARTITION OF orders
    FOR VALUES FROM ('2024-11-01') TO ('2024-12-01');

CREATE TABLE orders_2024_12 PARTITION OF orders
    FOR VALUES FROM ('2024-12-01') TO ('2025-01-01');

-- Default partition สำหรับข้อมูลที่ไม่ตรงกับ partition ไหน
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- สร้าง indexes บน parent table (จะ propagate ไปยัง partitions อัตโนมัติ)
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);

-- Function สำหรับ auto-create partitions
CREATE OR REPLACE FUNCTION create_monthly_partition(
    p_year INTEGER,
    p_month INTEGER
)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_name := format('orders_%s_%s', p_year, LPAD(p_month::TEXT, 2, '0'));
    start_date := make_date(p_year, p_month, 1);
    end_date := start_date + INTERVAL '1 month';
    
    -- ตรวจสอบว่ามี partition อยู่แล้วหรือยัง
    IF NOT EXISTS (
        SELECT 1 FROM pg_tables 
        WHERE tablename = partition_name
    ) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF orders FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            start_date,
            end_date
        );
        
        RAISE NOTICE 'Created partition: %', partition_name;
    ELSE
        RAISE NOTICE 'Partition already exists: %', partition_name;
    END IF;
END;
$$;

-- Schedule สำหรับ auto-create partitions (ใช้ pg_cron)
SELECT cron.schedule(
    'create-monthly-partitions',
    '0 0 20 * *',  -- ทุกวันที่ 20 ของเดือน เวลาเที่ยงคืน
    $$
    SELECT create_monthly_partition(
        EXTRACT(YEAR FROM NOW() + INTERVAL '1 month')::INTEGER,
        EXTRACT(MONTH FROM NOW() + INTERVAL '1 month')::INTEGER
    );
    $$
);
```

### 5.2 Hash Partitioning สำหรับ Users

```sql
-- migrations/002_create_partitioned_users.sql

-- Hash Partitioning กระจาย data ได้สม่ำเสมอกว่า
CREATE TABLE users (
    id          UUID NOT NULL DEFAULT gen_random_uuid(),
    email       VARCHAR(255) NOT NULL,
    username    VARCHAR(100) NOT NULL,
    password    VARCHAR(255) NOT NULL,
    first_name  VARCHAR(100),
    last_name   VARCHAR(100),
    status      VARCHAR(20) NOT NULL DEFAULT 'active',
    role        VARCHAR(50) NOT NULL DEFAULT 'user',
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (id)
) PARTITION BY HASH (id);

-- สร้าง 8 partitions (power of 2 ดีสำหรับ distribution)
CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 2);
CREATE TABLE users_p3 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 3);
CREATE TABLE users_p4 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 4);
CREATE TABLE users_p5 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 5);
CREATE TABLE users_p6 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 6);
CREATE TABLE users_p7 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 7);

-- Unique constraint บน email (ต้องสร้างบน partitions แยก)
DO $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 0..7 LOOP
        EXECUTE format(
            'CREATE UNIQUE INDEX users_p%s_email_idx ON users_p%s (email)',
            i, i
        );
        EXECUTE format(
            'CREATE INDEX users_p%s_username_idx ON users_p%s (username)',
            i, i
        );
        EXECUTE format(
            'CREATE INDEX users_p%s_status_idx ON users_p%s (status)',
            i, i
        );
    END LOOP;
END;
$$;
```

### 5.3 List Partitioning สำหรับ Transactions ตาม Region

```sql
-- List Partitioning ตาม region/geography
CREATE TABLE transactions (
    id          UUID NOT NULL DEFAULT gen_random_uuid(),
    order_id    UUID NOT NULL,
    user_id     UUID NOT NULL,
    amount      DECIMAL(12, 2) NOT NULL,
    currency    CHAR(3) NOT NULL,
    region      VARCHAR(20) NOT NULL,  -- partition key
    status      VARCHAR(50) NOT NULL DEFAULT 'pending',
    gateway     VARCHAR(50) NOT NULL,
    gateway_ref VARCHAR(255),
    metadata    JSONB,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

-- Partitions ตาม region
CREATE TABLE transactions_asia PARTITION OF transactions
    FOR VALUES IN ('TH', 'SG', 'MY', 'ID', 'PH', 'VN', 'JP', 'KR', 'CN');

CREATE TABLE transactions_europe PARTITION OF transactions
    FOR VALUES IN ('GB', 'DE', 'FR', 'IT', 'ES', 'NL', 'SE', 'NO', 'DK');

CREATE TABLE transactions_americas PARTITION OF transactions
    FOR VALUES IN ('US', 'CA', 'MX', 'BR', 'AR', 'CO');

CREATE TABLE transactions_middle_east PARTITION OF transactions
    FOR VALUES IN ('AE', 'SA', 'QA', 'KW', 'BH', 'OM');

CREATE TABLE transactions_other PARTITION OF transactions DEFAULT;

-- Indexes
CREATE INDEX ON transactions (user_id);
CREATE INDEX ON transactions (order_id);
CREATE INDEX ON transactions (status, created_at DESC);
CREATE INDEX ON transactions (region, created_at DESC);
```

---

## 6. Partition Manager

```typescript
// src/partitioning/partition-manager.ts
import { Pool, PoolClient } from 'pg';
import { format } from 'date-fns';

interface PartitionInfo {
  name: string;
  tableName: string;
  parentTable: string;
  rangeStart?: Date;
  rangeEnd?: Date;
  modulus?: number;
  remainder?: number;
  listValues?: string[];
  size?: string;
  rowCount?: number;
}

export class PartitionManager {
  constructor(private readonly pool: Pool) {}

  // สร้าง partition สำหรับเดือนถัดไป
  async createNextMonthPartition(tableName: string): Promise<void> {
    const nextMonth = new Date();
    nextMonth.setMonth(nextMonth.getMonth() + 1);
    
    const year = nextMonth.getFullYear();
    const month = nextMonth.getMonth() + 1;
    const partitionName = `${tableName}_${year}_${String(month).padStart(2, '0')}`;
    
    const startDate = new Date(year, month - 1, 1);
    const endDate = new Date(year, month, 1);
    
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // ตรวจสอบว่ามี partition อยู่แล้วหรือยัง
      const exists = await client.query(
        `SELECT 1 FROM pg_tables WHERE tablename = $1`,
        [partitionName],
      );
      
      if (exists.rowCount === 0) {
        await client.query(`
          CREATE TABLE ${partitionName} PARTITION OF ${tableName}
          FOR VALUES FROM ($1) TO ($2)
        `, [startDate, endDate]);
        
        console.log(`Created partition: ${partitionName}`);
      }
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  // ดึงข้อมูล partitions ทั้งหมด
  async listPartitions(tableName: string): Promise<PartitionInfo[]> {
    const client = await this.pool.connect();
    
    try {
      const result = await client.query<PartitionInfo>(`
        SELECT 
          c.relname AS name,
          c.relname AS "tableName",
          parent.relname AS "parentTable",
          pg_size_pretty(pg_total_relation_size(c.oid)) AS size,
          pgstattuple(c.oid).tuple_count AS "rowCount"
        FROM pg_class c
        JOIN pg_inherits i ON i.inhrelid = c.oid
        JOIN pg_class parent ON parent.oid = i.inhparent
        WHERE parent.relname = $1
        ORDER BY c.relname
      `, [tableName]);
      
      return result.rows;
    } finally {
      client.release();
    }
  }

  // ย้ายข้อมูลเก่าไปยัง archive table
  async archiveOldPartitions(
    tableName: string,
    beforeDate: Date,
  ): Promise<void> {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // หา partitions ที่เก่ากว่า beforeDate
      const partitions = await this.listPartitions(tableName);
      
      for (const partition of partitions) {
        if (!partition.rangeEnd || partition.rangeEnd <= beforeDate) {
          const archiveName = `archive_${partition.name}`;
          
          // สร้าง archive table และย้ายข้อมูล
          await client.query(`
            CREATE TABLE IF NOT EXISTS ${archiveName} 
            (LIKE ${partition.name} INCLUDING ALL)
          `);
          
          await client.query(`
            INSERT INTO ${archiveName} SELECT * FROM ${partition.name}
          `);
          
          // Detach partition จาก parent
          await client.query(`
            ALTER TABLE ${tableName} DETACH PARTITION ${partition.name}
          `);
          
          console.log(`Archived partition: ${partition.name} -> ${archiveName}`);
        }
      }
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  // ดูขนาดของแต่ละ partition
  async getPartitionSizes(tableName: string): Promise<
    { name: string; size: string; rows: number }[]
  > {
    const client = await this.pool.connect();
    
    try {
      const result = await client.query(`
        SELECT 
          c.relname AS name,
          pg_size_pretty(pg_total_relation_size(c.oid)) AS size,
          COALESCE((SELECT n_live_tup FROM pg_stat_user_tables WHERE relname = c.relname), 0) AS rows
        FROM pg_class c
        JOIN pg_inherits i ON i.inhrelid = c.oid
        JOIN pg_class parent ON parent.oid = i.inhparent
        WHERE parent.relname = $1
        ORDER BY pg_total_relation_size(c.oid) DESC
      `, [tableName]);
      
      return result.rows;
    } finally {
      client.release();
    }
  }
}
```

---

## 7. Citus Extension Setup

```sql
-- migrations/003_setup_citus.sql

-- ติดตั้ง Citus extension
CREATE EXTENSION IF NOT EXISTS citus;

-- ตั้งค่า Citus coordinator และ workers
-- (ทำบน coordinator node)
SELECT citus_set_coordinator_host('coordinator', 5432);

-- เพิ่ม worker nodes
SELECT citus_add_node('worker-1', 5432);
SELECT citus_add_node('worker-2', 5432);
SELECT citus_add_node('worker-3', 5432);
SELECT citus_add_node('worker-4', 5432);

-- ตรวจสอบ nodes
SELECT * FROM citus_get_active_worker_nodes();

-- สร้างตารางแบบ distributed
CREATE TABLE users_distributed (
    id          UUID NOT NULL DEFAULT gen_random_uuid(),
    email       VARCHAR(255) NOT NULL,
    username    VARCHAR(100) NOT NULL,
    tenant_id   UUID NOT NULL,  -- distribution column
    status      VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, tenant_id)
);

-- กระจาย table ตาม tenant_id (สำหรับ multi-tenant app)
SELECT create_distributed_table('users_distributed', 'tenant_id');

-- สร้าง reference table (copy ไปทุก worker)
CREATE TABLE countries (
    code    CHAR(2) PRIMARY KEY,
    name    VARCHAR(100) NOT NULL,
    region  VARCHAR(50)
);

SELECT create_reference_table('countries');

-- Collocated tables (sharded บน key เดียวกัน)
CREATE TABLE orders_distributed (
    id          UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,  -- ต้องเป็น distribution column เดียวกัน
    user_id     UUID NOT NULL,
    total       DECIMAL(12, 2) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, tenant_id)
);

-- Collocate กับ users_distributed (เก็บข้อมูลของ tenant เดียวกันไว้บน shard เดียวกัน)
SELECT create_distributed_table('orders_distributed', 'tenant_id', 
    colocate_with => 'users_distributed');

-- ตรวจสอบ shard placement
SELECT 
    logicalrelid::regclass AS table_name,
    shardid,
    nodename,
    nodeport
FROM pg_dist_placement 
JOIN pg_dist_shard USING (shardid)
ORDER BY table_name, shardid;

-- Rebalance shards เมื่อเพิ่ม worker ใหม่
-- SELECT rebalance_table_shards();

-- ตรวจสอบ distributed query plan
EXPLAIN 
SELECT u.username, COUNT(o.id) as order_count
FROM users_distributed u
JOIN orders_distributed o ON u.id = o.user_id AND u.tenant_id = o.tenant_id
WHERE u.tenant_id = '550e8400-e29b-41d4-a716-446655440000'
GROUP BY u.username;
```

---

## 8. Shard Router

```typescript
// src/sharding/shard-router.ts
import { ShardManager } from './shard-manager';

type ShardStrategy = 'hash' | 'range' | 'directory' | 'geo';

interface RoutingConfig {
  strategy: ShardStrategy;
  // สำหรับ range sharding
  ranges?: Array<{
    min: string | number;
    max: string | number;
    shardId: string;
  }>;
  // สำหรับ directory sharding
  directory?: Map<string, string>;
}

export class ShardRouter {
  private readonly routingConfig: RoutingConfig;

  constructor(
    private readonly shardManager: ShardManager,
    routingConfig: RoutingConfig,
  ) {
    this.routingConfig = routingConfig;
  }

  // Route query ไปยัง shard ที่ถูกต้อง
  async route<T>(
    shardKey: string,
    operation: (shardId: string) => Promise<T>,
  ): Promise<T> {
    const shardId = this.resolveShardId(shardKey);
    return operation(shardId);
  }

  // Route สำหรับ user queries (ใช้ user_id เป็น shard key)
  async routeUserQuery<T>(
    userId: string,
    query: string,
    params: unknown[],
  ): Promise<T[]> {
    const shardId = this.resolveShardId(userId);
    const client = await this.shardManager.getConnectionForShard(shardId);
    
    try {
      const result = await client.query<T>(query, params);
      return result.rows;
    } finally {
      client.release();
    }
  }

  // Route สำหรับ order queries (ใช้ user_id เพื่อ collocate กับ users)
  async routeOrderQuery<T>(
    userId: string,
    query: string,
    params: unknown[],
  ): Promise<T[]> {
    // ใช้ user_id เดียวกันกับ users เพื่อ query บน shard เดียวกัน
    return this.routeUserQuery<T>(userId, query, params);
  }

  private resolveShardId(shardKey: string): string {
    switch (this.routingConfig.strategy) {
      case 'hash':
        return this.hashRoute(shardKey);
      case 'range':
        return this.rangeRoute(shardKey);
      case 'directory':
        return this.directoryRoute(shardKey);
      default:
        throw new Error(`Unknown sharding strategy: ${this.routingConfig.strategy}`);
    }
  }

  private hashRoute(key: string): string {
    // ใช้ ConsistentHashRing ผ่าน ShardManager
    // ส่งคืน shard ID ตาม hash
    return 'shard-1'; // placeholder - ใช้ ConsistentHashRing จริงๆ
  }

  private rangeRoute(key: string): string {
    if (!this.routingConfig.ranges) {
      throw new Error('Range routing requires ranges configuration');
    }
    
    const numericKey = parseInt(key, 10);
    
    for (const range of this.routingConfig.ranges) {
      const min = typeof range.min === 'string' ? parseInt(range.min, 10) : range.min;
      const max = typeof range.max === 'string' ? parseInt(range.max, 10) : range.max;
      
      if (numericKey >= min && numericKey < max) {
        return range.shardId;
      }
    }
    
    throw new Error(`No shard found for key: ${key}`);
  }

  private directoryRoute(key: string): string {
    if (!this.routingConfig.directory) {
      throw new Error('Directory routing requires directory configuration');
    }
    
    const shardId = this.routingConfig.directory.get(key);
    if (!shardId) {
      throw new Error(`No shard mapping found for key: ${key}`);
    }
    
    return shardId;
  }
}
```

---

## 9. Shard Migration

```typescript
// src/sharding/shard-migrator.ts
import { Pool, PoolClient } from 'pg';
import { ConsistentHashRing } from './consistent-hash';

interface MigrationPlan {
  sourceShardId: string;
  targetShardId: string;
  keys: string[];
  estimatedRows: number;
}

export class ShardMigrator {
  private readonly batchSize: number = 1000;
  private readonly sleepBetweenBatches: number = 100; // ms

  constructor(
    private readonly sourcePools: Map<string, Pool>,
    private readonly targetPools: Map<string, Pool>,
    private readonly ring: ConsistentHashRing,
  ) {}

  // วางแผน migration เมื่อเพิ่ม shard ใหม่
  async planMigration(
    tableName: string,
    shardKeyColumn: string,
    newShardId: string,
  ): Promise<MigrationPlan[]> {
    const plans: MigrationPlan[] = [];
    
    // เพิ่ม shard ใหม่เข้า ring (ชั่วคราว)
    this.ring.addShard({
      id: newShardId,
      host: 'new-host',
      port: 5432,
      database: 'db',
      weight: 1,
      isActive: false, // ยังไม่ active
      tags: [],
    });
    
    // หา keys ที่ต้อง migrate
    for (const [sourceShardId, pool] of this.sourcePools) {
      if (sourceShardId === newShardId) continue;
      
      const client = await pool.connect();
      
      try {
        // ดึง keys ทั้งหมดจาก shard
        const keysResult = await client.query<{ shard_key: string }>(
          `SELECT DISTINCT ${shardKeyColumn} as shard_key FROM ${tableName}`,
        );
        
        const keysToMigrate = keysResult.rows
          .map((r) => r.shard_key)
          .filter((key) => {
            const targetShard = this.ring.getShard(key);
            return targetShard.id === newShardId;
          });
        
        if (keysToMigrate.length > 0) {
          // นับจำนวน rows ที่ต้อง migrate
          const countResult = await client.query<{ count: string }>(
            `SELECT COUNT(*) as count FROM ${tableName} 
             WHERE ${shardKeyColumn} = ANY($1)`,
            [keysToMigrate],
          );
          
          plans.push({
            sourceShardId,
            targetShardId: newShardId,
            keys: keysToMigrate,
            estimatedRows: parseInt(countResult.rows[0].count),
          });
        }
      } finally {
        client.release();
      }
    }
    
    return plans;
  }

  // ทำการ migrate data จริง
  async executeMigration(
    plan: MigrationPlan,
    tableName: string,
    shardKeyColumn: string,
    onProgress?: (migrated: number, total: number) => void,
  ): Promise<void> {
    const sourcePool = this.sourcePools.get(plan.sourceShardId);
    const targetPool = this.targetPools.get(plan.targetShardId);
    
    if (!sourcePool || !targetPool) {
      throw new Error('Source or target pool not found');
    }
    
    let migrated = 0;
    let lastId: string | null = null;
    
    // ดึง columns ของตาราง
    const sourceClient = await sourcePool.connect();
    const columns = await this.getTableColumns(sourceClient, tableName);
    sourceClient.release();
    
    while (true) {
      const sourceClient = await sourcePool.connect();
      const targetClient = await targetPool.connect();
      
      try {
        // ดึง batch ของ rows ที่ต้อง migrate (ใช้ cursor-based)
        const batchQuery = lastId
          ? `SELECT * FROM ${tableName} 
             WHERE ${shardKeyColumn} = ANY($1) AND id > $2 
             ORDER BY id LIMIT $3`
          : `SELECT * FROM ${tableName} 
             WHERE ${shardKeyColumn} = ANY($1) 
             ORDER BY id LIMIT $2`;
        
        const batchParams = lastId
          ? [plan.keys, lastId, this.batchSize]
          : [plan.keys, this.batchSize];
        
        const rows = await sourceClient.query(batchQuery, batchParams);
        
        if (rows.rowCount === 0) break;
        
        // Copy rows ไปยัง target shard
        await targetClient.query('BEGIN');
        
        for (const row of rows.rows) {
          const values = columns.map((col) => row[col]);
          const placeholders = columns.map((_, i) => `$${i + 1}`).join(', ');
          
          await targetClient.query(
            `INSERT INTO ${tableName} (${columns.join(', ')})
             VALUES (${placeholders})
             ON CONFLICT DO NOTHING`,
            values,
          );
        }
        
        await targetClient.query('COMMIT');
        
        migrated += rows.rowCount || 0;
        lastId = rows.rows[rows.rowCount! - 1].id;
        
        onProgress?.(migrated, plan.estimatedRows);
        
        // หยุดพักระหว่าง batches เพื่อไม่ให้ load สูงเกินไป
        await this.sleep(this.sleepBetweenBatches);
        
      } catch (error) {
        await targetClient.query('ROLLBACK');
        throw error;
      } finally {
        sourceClient.release();
        targetClient.release();
      }
    }
    
    console.log(`Migration completed: ${migrated} rows migrated from ${plan.sourceShardId} to ${plan.targetShardId}`);
  }

  // ลบข้อมูลเก่าหลัง migrate สำเร็จ
  async cleanupAfterMigration(
    plan: MigrationPlan,
    tableName: string,
    shardKeyColumn: string,
  ): Promise<void> {
    const sourcePool = this.sourcePools.get(plan.sourceShardId);
    if (!sourcePool) throw new Error('Source pool not found');
    
    const client = await sourcePool.connect();
    
    try {
      await client.query('BEGIN');
      
      // ลบ rows ที่ migrate ไปแล้ว
      await client.query(
        `DELETE FROM ${tableName} WHERE ${shardKeyColumn} = ANY($1)`,
        [plan.keys],
      );
      
      await client.query('COMMIT');
      console.log(`Cleanup completed for shard ${plan.sourceShardId}`);
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private async getTableColumns(client: PoolClient, tableName: string): Promise<string[]> {
    const result = await client.query<{ column_name: string }>(
      `SELECT column_name FROM information_schema.columns
       WHERE table_name = $1 ORDER BY ordinal_position`,
      [tableName],
    );
    return result.rows.map((r) => r.column_name);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}
```

---

## 10. Docker Compose สำหรับ PostgreSQL Cluster

```yaml
# docker-compose.yml
version: '3.9'

services:
  # Coordinator node สำหรับ Citus
  citus-coordinator:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_USER: citus
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-cituspassword}
      POSTGRES_DB: citus_db
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - citus_coordinator_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    ports:
      - "5432:5432"
    networks:
      - citus-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U citus"]
      interval: 10s
      timeout: 5s
      retries: 5
    command: >
      postgres
      -c max_connections=500
      -c shared_buffers=2GB
      -c effective_cache_size=6GB
      -c maintenance_work_mem=512MB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=64MB
      -c default_statistics_target=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
      -c work_mem=10485kB
      -c min_wal_size=1GB
      -c max_wal_size=4GB
      -c max_worker_processes=16
      -c max_parallel_workers_per_gather=4
      -c max_parallel_workers=16
      -c max_parallel_maintenance_workers=4

  # Worker nodes
  citus-worker-1:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_USER: citus
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-cituspassword}
      POSTGRES_DB: citus_db
    volumes:
      - citus_worker1_data:/var/lib/postgresql/data
    networks:
      - citus-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U citus"]
      interval: 10s
      timeout: 5s
      retries: 5

  citus-worker-2:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_USER: citus
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-cituspassword}
      POSTGRES_DB: citus_db
    volumes:
      - citus_worker2_data:/var/lib/postgresql/data
    networks:
      - citus-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U citus"]
      interval: 10s
      timeout: 5s
      retries: 5

  citus-worker-3:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_USER: citus
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-cituspassword}
      POSTGRES_DB: citus_db
    volumes:
      - citus_worker3_data:/var/lib/postgresql/data
    networks:
      - citus-network

  citus-worker-4:
    image: citusdata/citus:12.1
    environment:
      POSTGRES_USER: citus
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-cituspassword}
      POSTGRES_DB: citus_db
    volumes:
      - citus_worker4_data:/var/lib/postgresql/data
    networks:
      - citus-network

  # PgBouncer สำหรับ connection pooling
  pgbouncer:
    image: pgbouncer/pgbouncer:1.21.0
    environment:
      DATABASES_HOST: citus-coordinator
      DATABASES_PORT: 5432
      DATABASES_USER: citus
      DATABASES_PASSWORD: ${POSTGRES_PASSWORD:-cituspassword}
      DATABASES_DBNAME: citus_db
      PGBOUNCER_POOL_MODE: transaction
      PGBOUNCER_MAX_CLIENT_CONN: 10000
      PGBOUNCER_DEFAULT_POOL_SIZE: 100
      PGBOUNCER_MIN_POOL_SIZE: 10
      PGBOUNCER_RESERVE_POOL_SIZE: 20
      PGBOUNCER_RESERVE_POOL_TIMEOUT: 5
    volumes:
      - ./pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini:ro
    ports:
      - "6432:5432"
    networks:
      - citus-network
    depends_on:
      citus-coordinator:
        condition: service_healthy

  # Citus setup script
  citus-setup:
    image: citusdata/citus:12.1
    depends_on:
      citus-coordinator:
        condition: service_healthy
      citus-worker-1:
        condition: service_healthy
      citus-worker-2:
        condition: service_healthy
      citus-worker-3:
        condition: service_healthy
      citus-worker-4:
        condition: service_healthy
    networks:
      - citus-network
    command: >
      bash -c "
        psql postgresql://citus:${POSTGRES_PASSWORD:-cituspassword}@citus-coordinator:5432/citus_db << 'EOF'
          SELECT citus_set_coordinator_host('citus-coordinator', 5432);
          SELECT citus_add_node('citus-worker-1', 5432);
          SELECT citus_add_node('citus-worker-2', 5432);
          SELECT citus_add_node('citus-worker-3', 5432);
          SELECT citus_add_node('citus-worker-4', 5432);
          SELECT * FROM citus_get_active_worker_nodes();
      EOF
      "

volumes:
  citus_coordinator_data:
  citus_worker1_data:
  citus_worker2_data:
  citus_worker3_data:
  citus_worker4_data:

networks:
  citus-network:
    driver: bridge
```

---

## 11. Rebalancing Script

```bash
#!/bin/bash
# scripts/rebalance-shards.sh
# Script สำหรับ rebalance shards หลังเพิ่ม worker nodes

set -euo pipefail

COORDINATOR_HOST="${COORDINATOR_HOST:-localhost}"
COORDINATOR_PORT="${COORDINATOR_PORT:-5432}"
COORDINATOR_DB="${COORDINATOR_DB:-citus_db}"
COORDINATOR_USER="${COORDINATOR_USER:-citus}"
PGPASSWORD="${PGPASSWORD:-cituspassword}"

export PGPASSWORD

psql_cmd() {
    psql -h "$COORDINATOR_HOST" -p "$COORDINATOR_PORT" \
         -U "$COORDINATOR_USER" -d "$COORDINATOR_DB" \
         -c "$1"
}

echo "=== Citus Shard Rebalancing ==="

# ตรวจสอบ worker nodes
echo "Active worker nodes:"
psql_cmd "SELECT * FROM citus_get_active_worker_nodes();"

# ดู shard distribution ก่อน rebalance
echo -e "\nShard distribution before rebalance:"
psql_cmd "
    SELECT 
        nodename,
        COUNT(*) as shard_count,
        pg_size_pretty(SUM(shard_size)) as total_size
    FROM citus_shards
    GROUP BY nodename
    ORDER BY shard_count DESC;
"

# ทำ rebalance (drain_only=false จะ rebalance ทั้งหมด)
echo -e "\nStarting rebalance..."
psql_cmd "SELECT rebalance_table_shards(drain_only => false);"

# ดู shard distribution หลัง rebalance
echo -e "\nShard distribution after rebalance:"
psql_cmd "
    SELECT 
        nodename,
        COUNT(*) as shard_count,
        pg_size_pretty(SUM(shard_size)) as total_size
    FROM citus_shards
    GROUP BY nodename
    ORDER BY shard_count DESC;
"

echo "✅ Rebalancing completed!"
```

---

## สรุป

| หัวข้อ | เทคโนโลยี | วัตถุประสงค์ |
|--------|-----------|-------------|
| Consistent Hashing | Virtual nodes (150 per shard) | กระจาย data สม่ำเสมอ, minimize remapping เมื่อเพิ่ม shard |
| Hash Partitioning | PostgreSQL PARTITION BY HASH | กระจาย rows สม่ำเสมอตาม primary key |
| Range Partitioning | PostgreSQL PARTITION BY RANGE | แบ่ง data ตามช่วงเวลา (monthly) สำหรับ time-series |
| List Partitioning | PostgreSQL PARTITION BY LIST | แบ่ง data ตาม categorical value (region) |
| Cross-Shard Queries | Scatter-Gather pattern | Query ทุก shards พร้อมกัน แล้ว aggregate |
| Citus Extension | citusdata/citus | Distributed PostgreSQL สำหรับ multi-tenant apps |
| Shard Migration | Zero-downtime migration | ย้าย data ระหว่าง shards โดยไม่หยุด service |
| Connection Pooling | PgBouncer (transaction mode) | รองรับ connections จำนวนมาก (10,000+) |

> **Best Practice**: ใช้ user_id หรือ tenant_id เป็น shard key เสมอ เพื่อให้ข้อมูลของ user/tenant เดียวกันอยู่บน shard เดียวกัน ลด cross-shard queries

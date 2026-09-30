# Part 58: Performance Optimization สำหรับ Microservices

## บทนำ

Performance Optimization เป็นส่วนสำคัญของการพัฒนา Microservices ในระดับ Production การปรับปรุงประสิทธิภาพต้องอาศัยการวัดผล (measurement) ก่อนเสมอ เพื่อให้รู้ว่า bottleneck อยู่ที่ไหน

## หัวข้อที่ครอบคลุม

1. Profiling Node.js Applications
2. Database Query Optimization
3. Connection Pooling
4. Async Optimization
5. Memory Management
6. HTTP/2 และ Compression
7. CPU Profiling และ Flamegraphs
8. Load Testing Analysis

---

## 1. Node.js Profiling

### 1.1 Built-in Profiler

```typescript
// profiling/performance-monitor.ts
import { performance, PerformanceObserver } from 'perf_hooks';
import { EventEmitter } from 'events';

export class PerformanceMonitor extends EventEmitter {
  private observer: PerformanceObserver;
  private measurements: Map<string, number[]> = new Map();
  
  constructor() {
    super();
    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        const name = entry.name;
        const duration = entry.duration;
        
        const existing = this.measurements.get(name) || [];
        existing.push(duration);
        this.measurements.set(name, existing);
        
        this.emit('measurement', { name, duration });
        
        if (duration > 100) {
          console.warn(`Slow operation detected: ${name} took ${duration.toFixed(2)}ms`);
        }
      }
    });
    
    this.observer.observe({ entryTypes: ['measure', 'function'] });
  }
  
  time(name: string): () => void {
    const markName = `${name}-start-${Date.now()}`;
    performance.mark(markName);
    
    return () => {
      performance.measure(name, markName);
      performance.clearMarks(markName);
    };
  }
  
  async timeAsync<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const end = this.time(name);
    try {
      return await fn();
    } finally {
      end();
    }
  }
  
  getStats(name: string): {
    count: number;
    avg: number;
    p50: number;
    p95: number;
    p99: number;
    max: number;
    min: number;
  } | null {
    const data = this.measurements.get(name);
    if (!data || data.length === 0) return null;
    
    const sorted = [...data].sort((a, b) => a - b);
    const sum = sorted.reduce((a, b) => a + b, 0);
    
    return {
      count: sorted.length,
      avg: sum / sorted.length,
      p50: this.percentile(sorted, 50),
      p95: this.percentile(sorted, 95),
      p99: this.percentile(sorted, 99),
      max: sorted[sorted.length - 1],
      min: sorted[0],
    };
  }
  
  private percentile(sorted: number[], p: number): number {
    const index = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[Math.max(0, index)];
  }
  
  report(): void {
    console.log('\n=== Performance Report ===');
    for (const [name] of this.measurements) {
      const stats = this.getStats(name);
      if (stats) {
        console.log(`\n${name}:`);
        console.log(`  Count: ${stats.count}`);
        console.log(`  Avg: ${stats.avg.toFixed(2)}ms`);
        console.log(`  P50: ${stats.p50.toFixed(2)}ms`);
        console.log(`  P95: ${stats.p95.toFixed(2)}ms`);
        console.log(`  P99: ${stats.p99.toFixed(2)}ms`);
        console.log(`  Max: ${stats.max.toFixed(2)}ms`);
      }
    }
  }
  
  reset(): void {
    this.measurements.clear();
  }
}

// Middleware สำหรับ Express
export function createPerformanceMiddleware(monitor: PerformanceMonitor) {
  return (req: any, res: any, next: any) => {
    const operationName = `HTTP ${req.method} ${req.route?.path || req.path}`;
    const end = monitor.time(operationName);
    
    res.on('finish', () => {
      end();
    });
    
    next();
  };
}
```

### 1.2 Clinic.js Integration

```bash
# ติดตั้ง clinic
npm install -g clinic

# Profile CPU
clinic doctor -- node dist/server.js

# Profile event loop
clinic bubbleprof -- node dist/server.js

# Flamegraph
clinic flame -- node dist/server.js
```

### 1.3 V8 Heap Profiling

```typescript
// profiling/heap-profiler.ts
import v8 from 'v8';
import fs from 'fs';
import path from 'path';

export class HeapProfiler {
  private snapshots: string[] = [];
  
  takeSnapshot(label: string): string {
    const snapshotPath = path.join(
      '/tmp',
      `heap-${label}-${Date.now()}.heapsnapshot`
    );
    
    const snapshot = v8.writeHeapSnapshot(snapshotPath);
    this.snapshots.push(snapshot);
    
    console.log(`Heap snapshot saved: ${snapshot}`);
    return snapshot;
  }
  
  getHeapStats(): v8.HeapInfo {
    return v8.getHeapStatistics();
  }
  
  monitorHeap(thresholdMB: number = 512, intervalMs: number = 30000): void {
    setInterval(() => {
      const stats = this.getHeapStats();
      const usedMB = stats.used_heap_size / 1024 / 1024;
      const totalMB = stats.total_heap_size / 1024 / 1024;
      
      console.log(`Heap: ${usedMB.toFixed(1)}MB / ${totalMB.toFixed(1)}MB`);
      
      if (usedMB > thresholdMB) {
        console.warn(`High heap usage: ${usedMB.toFixed(1)}MB`);
        this.takeSnapshot(`high-memory-${usedMB.toFixed(0)}mb`);
      }
    }, intervalMs);
  }
}
```

---

## 2. Database Query Optimization

### 2.1 Query Analyzer

```typescript
// database/query-analyzer.ts
import { Pool, PoolClient } from 'pg';

interface QueryExplainResult {
  planType: string;
  cost: number;
  actualTime: number;
  rows: number;
  loops: number;
  details: string;
}

export class QueryAnalyzer {
  constructor(private db: Pool) {}
  
  async analyze(query: string, params?: unknown[]): Promise<QueryExplainResult> {
    const explainQuery = `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${query}`;
    const result = await this.db.query(explainQuery, params);
    
    const plan = result.rows[0]['QUERY PLAN'][0];
    const executionTime = plan['Execution Time'];
    const planningTime = plan['Planning Time'];
    
    console.log(`Planning: ${planningTime}ms, Execution: ${executionTime}ms`);
    
    return this.parsePlan(plan.Plan);
  }
  
  private parsePlan(plan: any): QueryExplainResult {
    return {
      planType: plan['Node Type'],
      cost: plan['Total Cost'],
      actualTime: plan['Actual Total Time'],
      rows: plan['Actual Rows'],
      loops: plan['Actual Loops'],
      details: JSON.stringify(plan, null, 2),
    };
  }
  
  async findSlowQueries(thresholdMs: number = 100): Promise<any[]> {
    const result = await this.db.query(`
      SELECT 
        query,
        calls,
        total_exec_time,
        mean_exec_time,
        max_exec_time,
        stddev_exec_time,
        rows
      FROM pg_stat_statements
      WHERE mean_exec_time > $1
      ORDER BY mean_exec_time DESC
      LIMIT 20
    `, [thresholdMs]);
    
    return result.rows;
  }
  
  async findMissingIndexes(): Promise<any[]> {
    const result = await this.db.query(`
      SELECT
        schemaname,
        tablename,
        seq_scan,
        seq_tup_read,
        idx_scan,
        idx_tup_fetch,
        seq_tup_read / NULLIF(seq_scan, 0) as avg_seq_tup_per_scan
      FROM pg_stat_user_tables
      WHERE seq_scan > 100
        AND idx_scan < seq_scan
        AND seq_tup_read / NULLIF(seq_scan, 0) > 100
      ORDER BY seq_tup_read DESC
    `);
    
    return result.rows;
  }
  
  async analyzeIndexUsage(): Promise<any[]> {
    const result = await this.db.query(`
      SELECT
        schemaname,
        tablename,
        indexname,
        idx_scan,
        idx_tup_read,
        idx_tup_fetch,
        pg_size_pretty(pg_relation_size(indexrelid)) as index_size
      FROM pg_stat_user_indexes
      ORDER BY idx_scan ASC
      LIMIT 20
    `);
    
    return result.rows;
  }
}

// Query Optimizer Middleware
export class QueryOptimizer {
  private queryCache = new Map<string, any>();
  private queryStats = new Map<string, {
    count: number;
    totalTime: number;
    errors: number;
  }>();
  
  async execute<T>(
    db: Pool,
    query: string,
    params?: unknown[],
    options?: { cache?: boolean; cacheTtlMs?: number }
  ): Promise<T[]> {
    const cacheKey = `${query}:${JSON.stringify(params)}`;
    
    // Check cache
    if (options?.cache) {
      const cached = this.queryCache.get(cacheKey);
      if (cached && Date.now() - cached.timestamp < (options.cacheTtlMs || 60000)) {
        return cached.data;
      }
    }
    
    const startTime = Date.now();
    try {
      const result = await db.query(query, params);
      const duration = Date.now() - startTime;
      
      // Update stats
      const stat = this.queryStats.get(query) || { count: 0, totalTime: 0, errors: 0 };
      stat.count++;
      stat.totalTime += duration;
      this.queryStats.set(query, stat);
      
      // Warn about slow queries
      if (duration > 500) {
        console.warn(`Slow query (${duration}ms): ${query.slice(0, 100)}`);
      }
      
      // Cache result if requested
      if (options?.cache) {
        this.queryCache.set(cacheKey, {
          data: result.rows,
          timestamp: Date.now(),
        });
      }
      
      return result.rows;
    } catch (error) {
      const stat = this.queryStats.get(query) || { count: 0, totalTime: 0, errors: 0 };
      stat.errors++;
      this.queryStats.set(query, stat);
      throw error;
    }
  }
  
  getSlowQueries(thresholdMs: number = 100): Array<{ query: string; avgMs: number; count: number }> {
    const results: Array<{ query: string; avgMs: number; count: number }> = [];
    
    for (const [query, stats] of this.queryStats) {
      const avgMs = stats.totalTime / stats.count;
      if (avgMs > thresholdMs) {
        results.push({ query, avgMs, count: stats.count });
      }
    }
    
    return results.sort((a, b) => b.avgMs - a.avgMs);
  }
}
```

### 2.2 Efficient Pagination

```typescript
// database/cursor-pagination.ts
import { Pool } from 'pg';

interface CursorPaginationResult<T> {
  data: T[];
  nextCursor: string | null;
  hasMore: boolean;
}

// Cursor-based pagination (ดีกว่า OFFSET สำหรับข้อมูลจำนวนมาก)
export class CursorPaginator<T extends { id: string; created_at: Date }> {
  constructor(
    private db: Pool,
    private tableName: string,
    private selectFields: string = '*'
  ) {}
  
  async getPage(
    cursor: string | null,
    limit: number = 20,
    where?: string,
    params?: unknown[]
  ): Promise<CursorPaginationResult<T>> {
    const conditions: string[] = [];
    const queryParams: unknown[] = [...(params || [])];
    
    if (where) {
      conditions.push(`(${where})`);
    }
    
    if (cursor) {
      const decoded = JSON.parse(Buffer.from(cursor, 'base64url').toString());
      conditions.push(`(created_at, id) < ($${queryParams.length + 1}, $${queryParams.length + 2})`);
      queryParams.push(decoded.created_at, decoded.id);
    }
    
    const whereClause = conditions.length > 0 ? `WHERE ${conditions.join(' AND ')}` : '';
    
    const result = await this.db.query(
      `SELECT ${this.selectFields} FROM ${this.tableName}
       ${whereClause}
       ORDER BY created_at DESC, id DESC
       LIMIT $${queryParams.length + 1}`,
      [...queryParams, limit + 1] // Fetch one extra to check hasMore
    );
    
    const hasMore = result.rows.length > limit;
    const data = result.rows.slice(0, limit) as T[];
    
    let nextCursor: string | null = null;
    if (hasMore && data.length > 0) {
      const lastItem = data[data.length - 1];
      nextCursor = Buffer.from(JSON.stringify({
        created_at: lastItem.created_at,
        id: lastItem.id,
      })).toString('base64url');
    }
    
    return { data, nextCursor, hasMore };
  }
}

// Keyset Pagination สำหรับ sorted data
export async function keysetPaginate(
  db: Pool,
  query: string,
  lastId: string | null,
  limit: number = 20
): Promise<{ rows: any[]; nextLastId: string | null }> {
  let paginatedQuery = query;
  const params: unknown[] = [limit + 1];
  
  if (lastId) {
    paginatedQuery += ` AND id > $2`;
    params.push(lastId);
  }
  
  paginatedQuery += ` ORDER BY id ASC LIMIT $1`;
  
  const result = await db.query(paginatedQuery, params);
  const rows = result.rows.slice(0, limit);
  const nextLastId = result.rows.length > limit ? rows[rows.length - 1].id : null;
  
  return { rows, nextLastId };
}
```

---

## 3. Connection Pooling Optimization

```typescript
// database/optimized-pool.ts
import { Pool, PoolConfig } from 'pg';
import { Counter, Histogram, Gauge, Registry } from 'prom-client';

interface PoolMetrics {
  acquireLatency: Histogram;
  queryLatency: Histogram;
  poolSize: Gauge;
  idleCount: Gauge;
  waitingCount: Gauge;
  acquisitionErrors: Counter;
}

export class OptimizedConnectionPool {
  private pool: Pool;
  private metrics: PoolMetrics;
  
  constructor(config: PoolConfig & { registry: Registry }) {
    // Optimal pool configuration
    const poolConfig: PoolConfig = {
      ...config,
      // Max connections = CPU cores * 2 + disk I/O concurrency
      max: config.max || Math.min(process.env.DB_MAX_CONNECTIONS
        ? parseInt(process.env.DB_MAX_CONNECTIONS)
        : 20, 50),
      // Idle connections
      min: config.min || 2,
      // Release idle connections after
      idleTimeoutMillis: config.idleTimeoutMillis || 30000,
      // Fail fast if connection takes too long
      connectionTimeoutMillis: config.connectionTimeoutMillis || 2000,
      // Statement timeout
      statement_timeout: 30000,
      // Query timeout
      query_timeout: 30000,
    };
    
    this.pool = new Pool(poolConfig);
    this.metrics = this.initMetrics(config.registry);
    this.setupPoolMonitoring();
  }
  
  private initMetrics(registry: Registry): PoolMetrics {
    return {
      acquireLatency: new Histogram({
        name: 'db_connection_acquire_duration_seconds',
        help: 'Time to acquire a database connection',
        buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
        registers: [registry],
      }),
      queryLatency: new Histogram({
        name: 'db_query_duration_seconds',
        help: 'Database query execution time',
        labelNames: ['operation'],
        buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
        registers: [registry],
      }),
      poolSize: new Gauge({
        name: 'db_pool_size',
        help: 'Database connection pool size',
        registers: [registry],
      }),
      idleCount: new Gauge({
        name: 'db_pool_idle_connections',
        help: 'Idle database connections',
        registers: [registry],
      }),
      waitingCount: new Gauge({
        name: 'db_pool_waiting_requests',
        help: 'Requests waiting for a connection',
        registers: [registry],
      }),
      acquisitionErrors: new Counter({
        name: 'db_connection_acquisition_errors_total',
        help: 'Total connection acquisition errors',
        registers: [registry],
      }),
    };
  }
  
  private setupPoolMonitoring(): void {
    setInterval(() => {
      this.metrics.poolSize.set(this.pool.totalCount);
      this.metrics.idleCount.set(this.pool.idleCount);
      this.metrics.waitingCount.set(this.pool.waitingCount);
    }, 5000);
    
    this.pool.on('error', (err) => {
      console.error('Unexpected pool error:', err);
    });
    
    this.pool.on('connect', () => {
      // New connection created
    });
    
    this.pool.on('acquire', () => {
      // Connection acquired from pool
    });
    
    this.pool.on('remove', () => {
      // Connection removed from pool
    });
  }
  
  async query<T = any>(
    text: string,
    values?: unknown[],
    operationName?: string
  ): Promise<{ rows: T[]; rowCount: number }> {
    const end = this.metrics.acquireLatency.startTimer();
    const queryEnd = this.metrics.queryLatency.startTimer({
      operation: operationName || 'query',
    });
    
    try {
      const client = await this.pool.connect();
      end(); // Stop acquire timer
      
      try {
        const result = await client.query(text, values);
        return { rows: result.rows as T[], rowCount: result.rowCount || 0 };
      } finally {
        client.release();
      }
    } catch (err) {
      this.metrics.acquisitionErrors.inc();
      throw err;
    } finally {
      queryEnd();
    }
  }
  
  async transaction<T>(
    fn: (client: any) => Promise<T>
  ): Promise<T> {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
  
  async end(): Promise<void> {
    await this.pool.end();
  }
}
```

---

## 4. Async Optimization

### 4.1 Parallel Execution

```typescript
// optimization/parallel-executor.ts

// แทนที่จะรัน sequential:
// const a = await getUser(id);
// const b = await getOrders(id);
// const c = await getPreferences(id);

// ใช้ parallel execution:
export async function getCompleteUserProfile(userId: string) {
  const [user, orders, preferences, address] = await Promise.all([
    getUser(userId),
    getOrders(userId),
    getPreferences(userId),
    getAddress(userId),
  ]);
  
  return { user, orders, preferences, address };
}

// Limit concurrency เพื่อป้องกัน overload
export class ConcurrencyLimiter {
  private running = 0;
  private queue: Array<() => void> = [];
  
  constructor(private maxConcurrency: number) {}
  
  async run<T>(fn: () => Promise<T>): Promise<T> {
    while (this.running >= this.maxConcurrency) {
      await new Promise<void>(resolve => this.queue.push(resolve));
    }
    
    this.running++;
    
    try {
      return await fn();
    } finally {
      this.running--;
      const next = this.queue.shift();
      if (next) next();
    }
  }
  
  // Process items with limited concurrency
  async processAll<T, R>(
    items: T[],
    processor: (item: T) => Promise<R>
  ): Promise<R[]> {
    return Promise.all(items.map(item => this.run(() => processor(item))));
  }
}

// Batch processor
export class BatchProcessor<T, R> {
  private batch: T[] = [];
  private resolvers: Array<(result: R) => void> = [];
  private rejecters: Array<(err: Error) => void> = [];
  private timer: NodeJS.Timeout | null = null;
  
  constructor(
    private processor: (items: T[]) => Promise<R[]>,
    private maxBatchSize: number = 100,
    private maxDelayMs: number = 10
  ) {}
  
  add(item: T): Promise<R> {
    return new Promise((resolve, reject) => {
      this.batch.push(item);
      this.resolvers.push(resolve);
      this.rejecters.push(reject);
      
      if (this.batch.length >= this.maxBatchSize) {
        this.flush();
      } else if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), this.maxDelayMs);
      }
    });
  }
  
  private async flush(): Promise<void> {
    if (this.timer) {
      clearTimeout(this.timer);
      this.timer = null;
    }
    
    const currentBatch = this.batch.splice(0);
    const currentResolvers = this.resolvers.splice(0);
    const currentRejecters = this.rejecters.splice(0);
    
    if (currentBatch.length === 0) return;
    
    try {
      const results = await this.processor(currentBatch);
      currentResolvers.forEach((resolve, i) => resolve(results[i]));
    } catch (err) {
      currentRejecters.forEach(reject => reject(err as Error));
    }
  }
}

// DataLoader pattern (สำหรับ N+1 problem)
export class DataLoader<K, V> {
  private pendingKeys: K[] = [];
  private pendingResolvers: Map<string, Array<(value: V | undefined) => void>> = new Map();
  private timer: NodeJS.Timeout | null = null;
  
  constructor(
    private batchFn: (keys: K[]) => Promise<Map<string, V>>,
    private maxBatchSize: number = 1000,
    private batchScheduler: (fn: () => void) => void = (fn) =>
      setImmediate(fn)
  ) {}
  
  load(key: K): Promise<V | undefined> {
    const keyStr = JSON.stringify(key);
    
    return new Promise((resolve) => {
      const resolvers = this.pendingResolvers.get(keyStr) || [];
      resolvers.push(resolve);
      this.pendingResolvers.set(keyStr, resolvers);
      
      if (!this.pendingKeys.some(k => JSON.stringify(k) === keyStr)) {
        this.pendingKeys.push(key);
      }
      
      if (this.pendingKeys.length >= this.maxBatchSize) {
        this.dispatch();
      } else if (!this.timer) {
        this.batchScheduler(() => this.dispatch());
        this.timer = setTimeout(() => {}, 0);
      }
    });
  }
  
  private async dispatch(): Promise<void> {
    const keys = this.pendingKeys.splice(0);
    const resolversMap = new Map(this.pendingResolvers);
    this.pendingResolvers.clear();
    this.timer = null;
    
    if (keys.length === 0) return;
    
    try {
      const results = await this.batchFn(keys);
      
      for (const [keyStr, resolvers] of resolversMap) {
        const value = results.get(keyStr);
        resolvers.forEach(resolve => resolve(value));
      }
    } catch (err) {
      for (const resolvers of resolversMap.values()) {
        resolvers.forEach(resolve => resolve(undefined));
      }
    }
  }
}

// Helper functions (placeholders)
async function getUser(id: string) { return null; }
async function getOrders(id: string) { return []; }
async function getPreferences(id: string) { return null; }
async function getAddress(id: string) { return null; }
```

---

## 5. Memory Management

### 5.1 Memory Leak Detection

```typescript
// optimization/memory-monitor.ts
import v8 from 'v8';

interface HeapSnapshot {
  timestamp: Date;
  usedHeapMB: number;
  totalHeapMB: number;
  externalMB: number;
}

export class MemoryMonitor {
  private snapshots: HeapSnapshot[] = [];
  private maxSnapshots: number = 60; // keep 60 snapshots
  
  constructor(private intervalMs: number = 60000) {
    this.startMonitoring();
  }
  
  private startMonitoring(): void {
    setInterval(() => {
      this.takeSnapshot();
      this.analyzeForLeaks();
    }, this.intervalMs);
  }
  
  private takeSnapshot(): void {
    const stats = process.memoryUsage();
    const snapshot: HeapSnapshot = {
      timestamp: new Date(),
      usedHeapMB: stats.heapUsed / 1024 / 1024,
      totalHeapMB: stats.heapTotal / 1024 / 1024,
      externalMB: stats.external / 1024 / 1024,
    };
    
    this.snapshots.push(snapshot);
    
    // Keep only recent snapshots
    if (this.snapshots.length > this.maxSnapshots) {
      this.snapshots.shift();
    }
  }
  
  private analyzeForLeaks(): void {
    if (this.snapshots.length < 10) return;
    
    const recent = this.snapshots.slice(-10);
    const oldest = recent[0].usedHeapMB;
    const newest = recent[recent.length - 1].usedHeapMB;
    
    const growthPercent = ((newest - oldest) / oldest) * 100;
    
    if (growthPercent > 20) {
      console.error(`Potential memory leak: heap grew ${growthPercent.toFixed(1)}% in last 10 samples`);
      console.error(`Heap: ${newest.toFixed(1)}MB (was ${oldest.toFixed(1)}MB)`);
      
      // Trigger GC if available
      if (global.gc) {
        console.log('Forcing GC...');
        global.gc();
      }
    }
  }
  
  getReport(): string {
    if (this.snapshots.length === 0) return 'No data';
    
    const latest = this.snapshots[this.snapshots.length - 1];
    const v8Stats = v8.getHeapStatistics();
    
    return [
      `Current Heap: ${latest.usedHeapMB.toFixed(1)}MB / ${latest.totalHeapMB.toFixed(1)}MB`,
      `External Memory: ${latest.externalMB.toFixed(1)}MB`,
      `Heap Limit: ${(v8Stats.heap_size_limit / 1024 / 1024).toFixed(0)}MB`,
      `Objects: ${v8Stats.number_of_native_contexts}`,
    ].join('\n');
  }
}

// Proper cleanup patterns
export class ResourceManager {
  private resources: Array<{ name: string; cleanup: () => void | Promise<void> }> = [];
  
  register(name: string, cleanup: () => void | Promise<void>): void {
    this.resources.push({ name, cleanup });
  }
  
  async cleanup(): Promise<void> {
    for (const resource of this.resources.reverse()) {
      try {
        await resource.cleanup();
        console.log(`Cleaned up: ${resource.name}`);
      } catch (err) {
        console.error(`Failed to cleanup ${resource.name}:`, err);
      }
    }
  }
}

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, starting graceful shutdown...');
  // cleanup resources
  process.exit(0);
});

process.on('SIGINT', async () => {
  console.log('SIGINT received, starting graceful shutdown...');
  process.exit(0);
});
```

---

## 6. HTTP/2 และ Compression

### 6.1 HTTP/2 Server Setup

```typescript
// server/http2-server.ts
import http2 from 'http2';
import fs from 'fs';
import express from 'express';
import compression from 'compression';
import spdy from 'spdy';

export function createHTTP2Server(app: express.Application): http2.Http2SecureServer {
  const options = {
    key: fs.readFileSync('./certs/server.key'),
    cert: fs.readFileSync('./certs/server.crt'),
    allowHTTP1: true, // Backward compatibility
  };
  
  const server = http2.createSecureServer(options, app as any);
  
  return server;
}

// Alternative: ใช้ SPDY (supports HTTP/2 + PUSH)
export function createSPDYServer(app: express.Application) {
  const options = {
    spdy: {
      protocols: ['h2', 'spdy/3.1', 'http/1.1'],
      plain: false,
      'x-forwarded-for': true,
    },
    key: fs.readFileSync('./certs/server.key'),
    cert: fs.readFileSync('./certs/server.crt'),
  };
  
  return spdy.createServer(options, app);
}

// Compression middleware
export function setupCompression(app: express.Application): void {
  app.use(compression({
    filter: (req, res) => {
      // ไม่ compress server-sent events
      if (req.headers['accept'] === 'text/event-stream') {
        return false;
      }
      return compression.filter(req, res);
    },
    level: 6, // Balance speed vs compression ratio
    threshold: 1024, // Only compress responses > 1KB
    chunkSize: 16 * 1024,
  }));
}
```

### 6.2 Response Streaming

```typescript
// optimization/streaming-response.ts
import express from 'express';
import { Readable, Transform } from 'stream';
import { pipeline } from 'stream/promises';
import { Pool } from 'pg';

// Stream large datasets ไม่ต้อง load ทั้งหมดใน memory
export async function streamLargeDataset(
  req: express.Request,
  res: express.Response,
  db: Pool
): Promise<void> {
  res.setHeader('Content-Type', 'application/json');
  res.setHeader('Transfer-Encoding', 'chunked');
  
  let isFirstRow = true;
  res.write('[');
  
  // ใช้ cursor สำหรับ large result sets
  const client = await db.connect();
  
  try {
    await client.query('BEGIN');
    await client.query(
      'DECLARE myportal CURSOR FOR SELECT * FROM large_table ORDER BY id'
    );
    
    while (true) {
      const result = await client.query('FETCH 100 FROM myportal');
      if (result.rows.length === 0) break;
      
      for (const row of result.rows) {
        if (!isFirstRow) res.write(',');
        res.write(JSON.stringify(row));
        isFirstRow = false;
      }
    }
    
    await client.query('CLOSE myportal');
    await client.query('COMMIT');
  } finally {
    client.release();
  }
  
  res.write(']');
  res.end();
}

// Server-Sent Events streaming
export function createSSEStream(
  req: express.Request,
  res: express.Response
): (data: unknown) => void {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.setHeader('X-Accel-Buffering', 'no'); // Disable nginx buffering
  
  // Keep connection alive
  const keepAlive = setInterval(() => {
    res.write(': keepalive\n\n');
  }, 30000);
  
  req.on('close', () => {
    clearInterval(keepAlive);
  });
  
  return (data: unknown) => {
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  };
}
```

---

## 7. Load Testing และ Benchmarking

### 7.1 k6 Load Test Script

```javascript
// tests/performance/load-test.js
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';

const errorRate = new Rate('error_rate');
const apiLatency = new Trend('api_latency');
const requestCount = new Counter('request_count');

export const options = {
  scenarios: {
    // Ramp up สำหรับ baseline
    ramp_up: {
      executor: 'ramping-vus',
      stages: [
        { duration: '30s', target: 10 },
        { duration: '1m', target: 50 },
        { duration: '2m', target: 100 },
        { duration: '1m', target: 0 },
      ],
    },
    // Spike test
    spike: {
      executor: 'ramping-vus',
      startTime: '5m',
      stages: [
        { duration: '10s', target: 200 },  // Sudden spike
        { duration: '1m', target: 200 },   // Hold
        { duration: '10s', target: 0 },    // Drop
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    error_rate: ['rate<0.01'],  // < 1% error rate
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

export default function () {
  group('Product API', function () {
    // Get products
    const listRes = http.get(`${BASE_URL}/api/v1/products?limit=20`);
    
    check(listRes, {
      'list status 200': (r) => r.status === 200,
      'list has data': (r) => JSON.parse(r.body).data.length > 0,
    });
    
    errorRate.add(listRes.status !== 200);
    apiLatency.add(listRes.timings.duration);
    requestCount.add(1);
    
    sleep(0.1);
    
    // Get single product
    const productId = 'product-001';
    const getRes = http.get(`${BASE_URL}/api/v1/products/${productId}`);
    
    check(getRes, {
      'get status 200': (r) => r.status === 200,
      'get has id': (r) => JSON.parse(r.body).id === productId,
    });
    
    errorRate.add(getRes.status !== 200);
    apiLatency.add(getRes.timings.duration);
    requestCount.add(1);
  });
  
  group('Order API', function () {
    const payload = JSON.stringify({
      customerId: 'customer-001',
      items: [
        { productId: 'product-001', quantity: 2, price: 100 }
      ],
    });
    
    const createRes = http.post(
      `${BASE_URL}/api/v1/orders`,
      payload,
      { headers: { 'Content-Type': 'application/json' } }
    );
    
    check(createRes, {
      'create order 201': (r) => r.status === 201,
      'has order id': (r) => JSON.parse(r.body).orderId !== undefined,
    });
    
    errorRate.add(createRes.status !== 201);
    requestCount.add(1);
  });
  
  sleep(1);
}

export function handleSummary(data) {
  return {
    'summary.json': JSON.stringify(data, null, 2),
    'summary.html': htmlReport(data),
  };
}

function htmlReport(data) {
  const p95 = data.metrics.http_req_duration.values['p(95)'];
  const errorRate = data.metrics.error_rate?.values.rate || 0;
  
  return `
<html>
<body>
<h1>Load Test Results</h1>
<p>P95 Latency: ${p95?.toFixed(2)}ms</p>
<p>Error Rate: ${(errorRate * 100).toFixed(2)}%</p>
<p>Total Requests: ${data.metrics.http_reqs?.values.count}</p>
</body>
</html>
  `;
}
```

---

## 8. Performance Optimization Checklist

### Checklist

```typescript
// optimization/performance-checklist.ts

export const performanceChecklist = {
  database: {
    'Use indexes for frequent queries': true,
    'Avoid SELECT *': true,
    'Use connection pooling': true,
    'Implement query result caching': true,
    'Use cursor-based pagination for large datasets': true,
    'Analyze and optimize slow queries': true,
    'Use read replicas for read-heavy workloads': true,
  },
  
  api: {
    'Enable HTTP/2': true,
    'Use compression (gzip/brotli)': true,
    'Implement response caching': true,
    'Use ETags for conditional requests': true,
    'Minimize payload size': true,
    'Implement pagination': true,
    'Use streaming for large responses': true,
  },
  
  nodejs: {
    'Use --max-old-space-size for production': true,
    'Enable cluster mode': true,
    'Use worker threads for CPU-intensive tasks': true,
    'Avoid blocking the event loop': true,
    'Use async/await properly': true,
    'Monitor memory usage': true,
    'Profile with clinic.js': true,
  },
  
  caching: {
    'Implement multi-level cache (L1/L2)': true,
    'Use cache-aside pattern': true,
    'Set appropriate TTLs': true,
    'Implement cache warming': true,
    'Monitor cache hit rates': true,
    'Prevent cache stampede': true,
  },
  
  network: {
    'Use CDN for static assets': true,
    'Enable keep-alive connections': true,
    'Minimize round trips': true,
    'Batch API requests': true,
    'Use WebSockets for real-time data': true,
    'Implement request deduplication': true,
  }
};
```

---

## สรุป

| หัวข้อ | เครื่องมือ | ผลที่ได้ |
|-------|-----------|---------|
| CPU Profiling | clinic.js, V8 flame graphs | ค้นหา hot spots |
| Memory | HeapProfiler, V8 stats | ค้นหา memory leaks |
| Database | EXPLAIN ANALYZE, pg_stat_statements | Optimize queries |
| Connection | pg Pool optimization | ลด connection overhead |
| Async | Promise.all, DataLoader | ลด latency |
| HTTP | HTTP/2, compression | ลด bandwidth |
| Load Testing | k6 | วัดและยืนยัน improvements |

กฎทอง: **Measure before Optimize** — วัดก่อนเสมอ อย่า optimize สิ่งที่ไม่ใช่ bottleneck

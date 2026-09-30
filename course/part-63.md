# Part 63: Database Migration Strategies

## บทนำ

Database Migration เป็นหนึ่งในส่วนที่ยากที่สุดของ Microservices เพราะต้องทำ
โดยไม่ให้ระบบหยุดทำงาน (Zero-Downtime) บทนี้จะครอบคลุมกลยุทธ์และเครื่องมือ
ที่ใช้จริงใน Production

---

## 1. หลักการ Zero-Downtime Migration

### ปัญหาที่เกิดขึ้นบ่อย

```
ปัญหา Schema Migration แบบ Naive:
1. Stop application
2. Run ALTER TABLE
3. Start application
→ Downtime!

ปัญหาของ Schema ที่เข้ากันไม่ได้:
Old code: SELECT name FROM users
New schema: column "name" ได้เปลี่ยนเป็น "full_name"
→ Error!
```

### Expand-Contract Pattern

```
Phase 1: EXPAND (เพิ่ม column ใหม่)
  - Add column full_name (nullable)
  - Deploy code ที่ write ทั้ง name และ full_name
  - ทำ backfill: full_name = name

Phase 2: CONTRACT (ลบ column เก่า)
  - Deploy code ที่ read/write แค่ full_name
  - Drop column name (หลังจาก verify ว่าทุกอย่างโอเค)
```

---

## 2. node-pg-migrate

### การติดตั้งและตั้งค่า

```bash
npm install node-pg-migrate pg
npm install --save-dev @types/pg

# เพิ่ม scripts ใน package.json
```

```json
{
  "scripts": {
    "migrate:up": "node-pg-migrate up",
    "migrate:down": "node-pg-migrate down",
    "migrate:create": "node-pg-migrate create",
    "migrate:status": "node-pg-migrate status",
    "migrate:redo": "node-pg-migrate redo",
    "migrate:up:prod": "DATABASE_URL=$DATABASE_URL node-pg-migrate up"
  }
}
```

```javascript
// database.json (config file สำหรับ node-pg-migrate)
{
  "development": {
    "host": "localhost",
    "port": 5432,
    "database": "orders_dev",
    "user": "postgres",
    "password": "password"
  },
  "test": {
    "host": "localhost",
    "port": 5432,
    "database": "orders_test",
    "user": "postgres",
    "password": "password"
  },
  "production": {
    "connectionString": {
      "ENV": "DATABASE_URL"
    },
    "ssl": {
      "rejectUnauthorized": false
    }
  }
}
```

### Migration Files

```typescript
// migrations/20240101000001_create_orders_table.ts
import { MigrationBuilder, ColumnDefinitions } from 'node-pg-migrate';

export const shorthands: ColumnDefinitions | undefined = undefined;

export async function up(pgm: MigrationBuilder): Promise<void> {
  pgm.createTable('orders', {
    id: {
      type: 'uuid',
      primaryKey: true,
      default: pgm.func('gen_random_uuid()'),
    },
    customer_id: {
      type: 'uuid',
      notNull: true,
    },
    status: {
      type: 'varchar(50)',
      notNull: true,
      default: 'pending',
      check: "status IN ('pending', 'processing', 'completed', 'cancelled')",
    },
    total_amount: {
      type: 'decimal(10, 2)',
      notNull: true,
    },
    currency: {
      type: 'varchar(3)',
      notNull: true,
      default: 'THB',
    },
    created_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    updated_at: {
      type: 'timestamptz',
      notNull: true,
      default: pgm.func('NOW()'),
    },
    deleted_at: {
      type: 'timestamptz',
    },
  });

  pgm.createIndex('orders', 'customer_id', {
    name: 'idx_orders_customer_id',
  });

  pgm.createIndex('orders', 'status', {
    name: 'idx_orders_status',
  });

  pgm.createIndex('orders', ['created_at', 'status'], {
    name: 'idx_orders_created_at_status',
  });

  // Trigger สำหรับ auto-update updated_at
  pgm.createFunction(
    'update_updated_at_column',
    [],
    {
      returns: 'trigger',
      language: 'plpgsql',
    },
    `
    BEGIN
      NEW.updated_at = NOW();
      RETURN NEW;
    END;
    `
  );

  pgm.createTrigger('orders', 'update_orders_updated_at', {
    when: 'BEFORE',
    operation: 'UPDATE',
    function: 'update_updated_at_column',
    level: 'ROW',
  });
}

export async function down(pgm: MigrationBuilder): Promise<void> {
  pgm.dropTrigger('orders', 'update_orders_updated_at');
  pgm.dropFunction('update_updated_at_column', []);
  pgm.dropTable('orders');
}
```

```typescript
// migrations/20240115000001_add_shipping_address.ts
// Phase 1: EXPAND - เพิ่ม column ใหม่
import { MigrationBuilder } from 'node-pg-migrate';

export async function up(pgm: MigrationBuilder): Promise<void> {
  // เพิ่ม shipping address columns ใหม่ (nullable เพื่อ backward compatibility)
  pgm.addColumns('orders', {
    shipping_address_street: {
      type: 'varchar(255)',
    },
    shipping_address_city: {
      type: 'varchar(100)',
    },
    shipping_address_postal_code: {
      type: 'varchar(20)',
    },
    shipping_address_country: {
      type: 'varchar(100)',
    },
  });

  // สร้าง index สำหรับ city เพื่อ analytics
  pgm.createIndex('orders', 'shipping_address_city', {
    name: 'idx_orders_shipping_city',
    where: 'shipping_address_city IS NOT NULL',
  });
}

export async function down(pgm: MigrationBuilder): Promise<void> {
  pgm.dropIndex('orders', 'shipping_address_city', {
    name: 'idx_orders_shipping_city',
  });
  pgm.dropColumns('orders', [
    'shipping_address_street',
    'shipping_address_city',
    'shipping_address_postal_code',
    'shipping_address_country',
  ]);
}
```

```typescript
// migrations/20240215000001_rename_customer_name.ts
// Zero-downtime rename: ใช้ Expand-Contract Pattern

export async function up(pgm: MigrationBuilder): Promise<void> {
  // Phase 1: Add new column
  pgm.addColumn('customers', {
    full_name: {
      type: 'varchar(255)',
    },
  });

  // Phase 2: Backfill data (ใน migration เดียวกัน)
  pgm.sql('UPDATE customers SET full_name = name WHERE full_name IS NULL');

  // Phase 3: Add NOT NULL constraint หลัง backfill
  // (ทำหลังจาก deploy code ที่ write full_name ด้วย)
}

export async function down(pgm: MigrationBuilder): Promise<void> {
  pgm.dropColumn('customers', 'full_name');
}
```

---

## 3. Blue-Green Database Migration

### Architecture

```yaml
# blue-green-migration.yaml
# Blue = Old version, Green = New version

# Stage 1: เตรียม Green database
apiVersion: batch/v1
kind: Job
metadata:
  name: prepare-green-db
spec:
  template:
    spec:
      containers:
        - name: db-migrator
          image: registry.company.com/db-migrator:latest
          env:
            - name: DATABASE_URL
              value: "postgresql://user:pass@postgres-green:5432/orders"
            - name: MIGRATION_DIRECTION
              value: "up"
          command: ["npm", "run", "migrate:up:prod"]
      restartPolicy: Never
```

```typescript
// src/database/blue-green-migration.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource } from 'typeorm';

interface MigrationPlan {
  phase: 'expand' | 'migrate' | 'contract';
  steps: string[];
  canRollback: boolean;
}

@Injectable()
export class BlueGreenMigrationService {
  private readonly logger = new Logger(BlueGreenMigrationService.name);

  async executeMigration(plan: MigrationPlan): Promise<void> {
    this.logger.log(`Starting ${plan.phase} phase migration`);

    for (const step of plan.steps) {
      this.logger.log(`Executing step: ${step}`);
      await this.executeStep(step);
    }

    this.logger.log(`${plan.phase} phase completed`);
  }

  private async executeStep(step: string): Promise<void> {
    // Execute migration step
    this.logger.debug(`Step: ${step}`);
  }

  // Validate ว่า new version compatible กับ old schema
  async validateCompatibility(
    oldSchemaVersion: string,
    newSchemaVersion: string
  ): Promise<{ compatible: boolean; issues: string[] }> {
    const issues: string[] = [];

    // ตรวจสอบ breaking changes
    // ตัวอย่าง: ลบ column ที่ยังมี old code ใช้อยู่
    const breakingChanges = await this.detectBreakingChanges(
      oldSchemaVersion,
      newSchemaVersion
    );

    issues.push(...breakingChanges);

    return {
      compatible: issues.length === 0,
      issues,
    };
  }

  private async detectBreakingChanges(
    oldVersion: string,
    newVersion: string
  ): Promise<string[]> {
    // Logic ตรวจสอบ breaking changes
    return [];
  }
}
```

---

## 4. Rolling Migration

```typescript
// src/database/rolling-migration.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource } from 'typeorm';

@Injectable()
export class RollingMigrationService {
  private readonly logger = new Logger(RollingMigrationService.name);

  constructor(private readonly dataSource: DataSource) {}

  // ทำ backfill แบบ batch เพื่อไม่ lock ทั้ง table
  async batchBackfill(
    tableName: string,
    updateQuery: string,
    batchSize = 1000,
    delayMs = 100
  ): Promise<{ totalUpdated: number }> {
    let totalUpdated = 0;
    let hasMore = true;

    this.logger.log(`Starting batch backfill on ${tableName}`);

    while (hasMore) {
      const result = await this.dataSource.query(
        `${updateQuery} LIMIT ${batchSize}`
      );

      const updated = result.rowCount || 0;
      totalUpdated += updated;

      this.logger.log(`Updated ${totalUpdated} rows so far...`);

      if (updated < batchSize) {
        hasMore = false;
      } else {
        // Pause เพื่อลด load บน database
        await this.sleep(delayMs);
      }
    }

    this.logger.log(`Backfill complete: ${totalUpdated} rows updated`);
    return { totalUpdated };
  }

  // เพิ่ม Index แบบ CONCURRENTLY (ไม่ lock table)
  async addIndexConcurrently(
    tableName: string,
    indexName: string,
    columns: string[],
    options: { unique?: boolean; where?: string } = {}
  ): Promise<void> {
    const uniqueStr = options.unique ? 'UNIQUE' : '';
    const whereStr = options.where ? `WHERE ${options.where}` : '';
    const columnsStr = columns.join(', ');

    const sql = `
      CREATE ${uniqueStr} INDEX CONCURRENTLY IF NOT EXISTS ${indexName}
      ON ${tableName} (${columnsStr})
      ${whereStr}
    `;

    this.logger.log(`Creating index concurrently: ${indexName}`);

    // CONCURRENTLY ไม่สามารถทำใน transaction ได้
    await this.dataSource.query(sql);

    this.logger.log(`Index ${indexName} created successfully`);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## 5. Migration Pipeline ใน CI/CD

```yaml
# .github/workflows/database-migration.yml
name: Database Migration

on:
  push:
    branches: [main]
    paths:
      - 'migrations/**'
      - 'package.json'

jobs:
  validate-migrations:
    name: Validate Migration Files
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run migrations (up)
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
        run: npm run migrate:up
        
      - name: Verify migration state
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
        run: npm run migrate:status
        
      - name: Test rollback
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
        run: npm run migrate:down -- --count 1
        
      - name: Re-apply after rollback
        env:
          DATABASE_URL: postgresql://test_user:test_password@localhost:5432/test_db
        run: npm run migrate:up
        
  deploy-migrations-staging:
    name: Deploy to Staging
    needs: validate-migrations
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        
      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_STAGING }}" > ~/.kube/config
          
      - name: Create migration job
        run: |
          kubectl apply -f k8s/jobs/migration-job.yaml -n staging
          kubectl wait --for=condition=complete job/db-migration \
            --timeout=300s -n staging
          
      - name: Check migration logs
        run: |
          kubectl logs job/db-migration -n staging
          
  deploy-migrations-production:
    name: Deploy to Production
    needs: deploy-migrations-staging
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        
      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_PROD }}" > ~/.kube/config
          
      - name: Take database backup
        run: |
          kubectl create job --from=cronjob/db-backup manual-backup-$(date +%Y%m%d) \
            -n production
          kubectl wait --for=condition=complete job/manual-backup-$(date +%Y%m%d) \
            --timeout=600s -n production
          
      - name: Run migration
        run: |
          kubectl apply -f k8s/jobs/migration-job.yaml -n production
          kubectl wait --for=condition=complete job/db-migration \
            --timeout=300s -n production
```

---

## 6. Kubernetes Migration Job

```yaml
# k8s/jobs/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production
  labels:
    app: db-migration
    version: "{{ .Values.appVersion }}"
spec:
  backoffLimit: 0  # ไม่ retry ถ้า fail
  ttlSecondsAfterFinished: 3600  # ลบ job หลัง 1 ชั่วโมง
  template:
    metadata:
      labels:
        app: db-migration
    spec:
      restartPolicy: Never
      serviceAccountName: migration-sa
      
      initContainers:
        - name: wait-for-db
          image: postgres:15-alpine
          command: ['sh', '-c']
          args:
            - |
              until pg_isready -h $DATABASE_HOST -p $DATABASE_PORT -U $DATABASE_USER; do
                echo "Waiting for database..."
                sleep 2
              done
              echo "Database is ready!"
          env:
            - name: DATABASE_HOST
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: host
            - name: DATABASE_PORT
              value: "5432"
            - name: DATABASE_USER
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: username
                  
      containers:
        - name: migrator
          image: registry.company.com/order-service:{{ .Values.appVersion }}
          command: ["node", "dist/migrate.js"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: url
            - name: NODE_ENV
              value: production
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

---

## 7. Multi-Tenant Migration

```typescript
// src/database/multi-tenant-migration.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource } from 'typeorm';

interface Tenant {
  id: string;
  schema: string;
  connectionString: string;
}

@Injectable()
export class MultiTenantMigrationService {
  private readonly logger = new Logger(MultiTenantMigrationService.name);

  constructor(private readonly masterDataSource: DataSource) {}

  async runMigrationsForAllTenants(): Promise<void> {
    const tenants = await this.getAllTenants();
    
    this.logger.log(`Running migrations for ${tenants.length} tenants`);
    
    const results = await Promise.allSettled(
      tenants.map(tenant => this.runMigrationForTenant(tenant))
    );
    
    const failed = results.filter(r => r.status === 'rejected');
    if (failed.length > 0) {
      this.logger.error(`${failed.length} tenant migrations failed`);
    }
    
    this.logger.log(`Migrations complete: ${results.length - failed.length}/${results.length} successful`);
  }

  private async runMigrationForTenant(tenant: Tenant): Promise<void> {
    this.logger.log(`Migrating tenant: ${tenant.id}`);
    
    const tenantDataSource = new DataSource({
      type: 'postgres',
      url: tenant.connectionString,
      schema: tenant.schema,
      migrations: ['migrations/*.ts'],
      migrationsRun: false,
    });

    try {
      await tenantDataSource.initialize();
      await tenantDataSource.runMigrations({ transaction: 'each' });
      this.logger.log(`Tenant ${tenant.id} migrated successfully`);
    } finally {
      await tenantDataSource.destroy();
    }
  }

  private async createTenantSchema(tenantId: string): Promise<void> {
    const schemaName = `tenant_${tenantId}`;
    
    await this.masterDataSource.transaction(async (manager) => {
      await manager.query(`CREATE SCHEMA IF NOT EXISTS "${schemaName}"`);
      
      // Copy template schema
      await manager.query(`
        SELECT clone_schema('tenant_template', '${schemaName}')
      `);
    });
  }

  private async getAllTenants(): Promise<Tenant[]> {
    // ดึงรายชื่อ tenants ทั้งหมดจาก master database
    const results = await this.masterDataSource.query(
      'SELECT id, schema_name, connection_string FROM tenants WHERE active = true'
    );
    
    return results.map((r: { id: string; schema_name: string; connection_string: string }) => ({
      id: r.id,
      schema: r.schema_name,
      connectionString: r.connection_string,
    }));
  }
}
```

---

## 8. Rollback Strategies

```typescript
// src/database/migration-rollback.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource } from 'typeorm';

interface MigrationSnapshot {
  version: string;
  takenAt: Date;
  backupLocation: string;
}

@Injectable()
export class MigrationRollbackService {
  private readonly logger = new Logger(MigrationRollbackService.name);

  constructor(private readonly dataSource: DataSource) {}

  // Snapshot-based rollback
  async takeSnapshot(version: string): Promise<MigrationSnapshot> {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const backupFile = `backup_${version}_${timestamp}.sql`;
    
    this.logger.log(`Taking snapshot for version ${version}`);
    
    // Execute pg_dump (ใน production จะ run เป็น Job แยกต่างหาก)
    const snapshot: MigrationSnapshot = {
      version,
      takenAt: new Date(),
      backupLocation: `s3://backups/migrations/${backupFile}`,
    };
    
    return snapshot;
  }

  async rollbackToSnapshot(snapshot: MigrationSnapshot): Promise<void> {
    this.logger.log(`Rolling back to snapshot: ${snapshot.version}`);
    
    // 1. Stop traffic (อาจใช้ feature flag หรือ maintenance mode)
    await this.enableMaintenanceMode();
    
    try {
      // 2. Restore from backup
      await this.restoreFromBackup(snapshot.backupLocation);
      
      // 3. Rollback migration files
      const currentVersion = await this.getCurrentMigrationVersion();
      const stepsToRollback = await this.calculateRollbackSteps(
        snapshot.version,
        currentVersion
      );
      
      for (let i = 0; i < stepsToRollback; i++) {
        await this.dataSource.undoLastMigration({ transaction: 'all' });
      }
      
      this.logger.log(`Rollback to ${snapshot.version} completed`);
    } finally {
      // 4. Re-enable traffic
      await this.disableMaintenanceMode();
    }
  }

  private async enableMaintenanceMode(): Promise<void> {
    this.logger.log('Enabling maintenance mode');
  }

  private async disableMaintenanceMode(): Promise<void> {
    this.logger.log('Disabling maintenance mode');
  }

  private async restoreFromBackup(backupLocation: string): Promise<void> {
    this.logger.log(`Restoring from: ${backupLocation}`);
  }

  private async getCurrentMigrationVersion(): Promise<string> {
    const result = await this.dataSource.query(
      'SELECT id FROM migrations ORDER BY run_on DESC LIMIT 1'
    );
    return result[0]?.id || '0';
  }

  private async calculateRollbackSteps(
    targetVersion: string,
    currentVersion: string
  ): Promise<number> {
    const migrations = await this.dataSource.showMigrations();
    return migrations.length; // simplified
  }
}
```

---

## 9. Database Version Control

```typescript
// src/database/version-control.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { DataSource } from 'typeorm';

interface SchemaVersion {
  version: string;
  appliedAt: Date;
  checksum: string;
  executionTime: number;
  success: boolean;
}

@Injectable()
export class DatabaseVersionControlService {
  private readonly logger = new Logger(DatabaseVersionControlService.name);

  constructor(private readonly dataSource: DataSource) {}

  async getCurrentVersion(): Promise<string> {
    try {
      const result = await this.dataSource.query(
        'SELECT version FROM schema_version ORDER BY applied_at DESC LIMIT 1'
      );
      return result[0]?.version || '0.0.0';
    } catch {
      return '0.0.0';
    }
  }

  async getAppliedMigrations(): Promise<SchemaVersion[]> {
    return this.dataSource.query(
      'SELECT * FROM schema_version ORDER BY applied_at ASC'
    );
  }

  async recordMigration(migration: Omit<SchemaVersion, 'appliedAt'>): Promise<void> {
    await this.dataSource.query(
      `INSERT INTO schema_version (version, applied_at, checksum, execution_time, success)
       VALUES ($1, NOW(), $2, $3, $4)`,
      [migration.version, migration.checksum, migration.executionTime, migration.success]
    );
  }

  async verifyChecksums(): Promise<{ valid: boolean; issues: string[] }> {
    const issues: string[] = [];
    const appliedMigrations = await this.getAppliedMigrations();
    
    for (const migration of appliedMigrations) {
      const currentChecksum = await this.calculateChecksum(migration.version);
      
      if (currentChecksum !== migration.checksum) {
        issues.push(`Migration ${migration.version}: checksum mismatch`);
      }
    }
    
    return { valid: issues.length === 0, issues };
  }

  private async calculateChecksum(version: string): Promise<string> {
    const crypto = await import('crypto');
    const content = `migration_${version}`;
    return crypto.createHash('md5').update(content).digest('hex');
  }
}
```

---

## 10. TypeORM Migration ตัวอย่าง

```typescript
// migrations/1706745600000-CreateOrdersTable.ts
import { MigrationInterface, QueryRunner, Table, Index } from 'typeorm';

export class CreateOrdersTable1706745600000 implements MigrationInterface {
  name = 'CreateOrdersTable1706745600000';

  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: 'orders',
        columns: [
          {
            name: 'id',
            type: 'uuid',
            isPrimary: true,
            generationStrategy: 'uuid',
            default: 'gen_random_uuid()',
          },
          {
            name: 'customer_id',
            type: 'uuid',
            isNullable: false,
          },
          {
            name: 'status',
            type: 'varchar',
            length: '50',
            default: "'pending'",
          },
          {
            name: 'total_amount',
            type: 'decimal',
            precision: 10,
            scale: 2,
          },
          {
            name: 'created_at',
            type: 'timestamptz',
            default: 'NOW()',
          },
          {
            name: 'updated_at',
            type: 'timestamptz',
            default: 'NOW()',
          },
        ],
      }),
      true
    );

    await queryRunner.createIndex(
      'orders',
      new Index({
        name: 'idx_orders_customer_id',
        columnNames: ['customer_id'],
      })
    );
  }

  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropIndex('orders', 'idx_orders_customer_id');
    await queryRunner.dropTable('orders');
  }
}
```

---

## สรุป

| กลยุทธ์ | เมื่อใช้ | ข้อดี | ข้อควรระวัง |
|---------|---------|-------|------------|
| Expand-Contract | Rename/Split columns | Zero-downtime | ต้อง deploy 2 ครั้ง |
| Blue-Green DB | Major schema changes | Quick rollback | ต้องการ 2x storage |
| Rolling Migration | Column backfill | ไม่ lock table | ช้ากว่า direct update |
| Batch Backfill | Large table updates | ลด DB load | ใช้เวลานานกว่า |
| CONCURRENT Index | Add index to live table | No table lock | ใช้เวลานานกว่า |
| Schema per Tenant | Multi-tenant apps | Isolation | Migration complexity |
| Migration Job (K8s) | CI/CD pipeline | Automated | Must handle failures |
| Snapshot Rollback | Critical failures | Fast recovery | Requires backup storage |
| Checksum Verify | Migration integrity | Detect corruption | Extra storage |
| node-pg-migrate | Node.js projects | Simple API | Limited features vs Flyway |

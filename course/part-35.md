# Part 35: Database Migration Strategies for Microservices

## บทนำ

การจัดการ Database Migration ใน Microservices เป็นหนึ่งในความท้าทายที่ยากที่สุด เพราะแต่ละ service มีฐานข้อมูลของตัวเอง และต้องสามารถ deploy ได้อย่างอิสระโดยไม่กระทบ service อื่น บทนี้จะสอนเทคนิคการทำ migration แบบ zero-downtime และ production-ready patterns

## เนื้อหาในบทนี้

- Flyway และ Liquibase migration setup
- Zero-downtime migration ด้วย Expand-Contract Pattern
- Blue-Green Database Migrations
- Rolling Migrations with Backward Compatibility
- Data Backfill Strategies
- Testing Database Migrations
- Rollback Procedures
- Schema Versioning with PostgreSQL

---

## 1. Flyway Migration Setup

### การติดตั้งและโครงสร้างโปรเจค

Flyway เป็น database migration tool ที่นิยมใช้กับ Java/Spring Boot โดยใช้ SQL scripts หรือ Java-based migrations

```
user-service/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/userservice/
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/
│   │           └── migration/
│   │               ├── V1__create_users_table.sql
│   │               ├── V2__add_email_index.sql
│   │               ├── V3__add_profile_columns.sql
│   │               └── R__create_views.sql
│   └── test/
│       └── java/
│           └── com/example/userservice/
│               └── migration/
│                   └── MigrationTest.java
└── pom.xml
```

### pom.xml dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-maven-plugin</artifactId>
            <configuration>
                <url>jdbc:postgresql://localhost:5432/userdb</url>
                <user>postgres</user>
                <password>${db.password}</password>
                <locations>classpath:db/migration</locations>
            </configuration>
        </plugin>
    </plugins>
</build>
```

### application.yml configuration

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:userdb}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:secret}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 0
    validate-on-migrate: true
    out-of-order: false
    table: flyway_schema_history
    schemas: public
    # สำหรับ multi-tenant
    # schemas: tenant1,tenant2

  jpa:
    hibernate:
      ddl-auto: validate  # ห้ามใช้ create หรือ update ใน production
    show-sql: false
```

### Migration Scripts

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    username    VARCHAR(50) NOT NULL UNIQUE,
    email       VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at  TIMESTAMP   NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP   NOT NULL DEFAULT NOW(),
    is_active   BOOLEAN     NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_created_at ON users(created_at);

-- trigger สำหรับ auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

COMMENT ON TABLE users IS 'Core users table - managed by user-service';
COMMENT ON COLUMN users.id IS 'UUID primary key';
COMMENT ON COLUMN users.password_hash IS 'bcrypt hashed password';
```

```sql
-- V2__add_email_index.sql
-- เพิ่ม partial index สำหรับ active users เท่านั้น
CREATE INDEX CONCURRENTLY idx_users_active_email 
ON users(email) 
WHERE is_active = TRUE;
```

```sql
-- V3__add_profile_columns.sql
-- Expand phase: เพิ่ม columns ใหม่ที่ nullable ก่อน
ALTER TABLE users 
    ADD COLUMN IF NOT EXISTS first_name VARCHAR(100),
    ADD COLUMN IF NOT EXISTS last_name  VARCHAR(100),
    ADD COLUMN IF NOT EXISTS phone      VARCHAR(20),
    ADD COLUMN IF NOT EXISTS avatar_url TEXT,
    ADD COLUMN IF NOT EXISTS bio        TEXT,
    ADD COLUMN IF NOT EXISTS timezone   VARCHAR(50) DEFAULT 'UTC';

-- เพิ่ม index สำหรับการ search
CREATE INDEX CONCURRENTLY idx_users_full_name 
ON users(first_name, last_name) 
WHERE first_name IS NOT NULL;
```

---

## 2. Liquibase Migration Setup

Liquibase รองรับ XML, YAML, JSON, และ SQL format มีความยืดหยุ่นกว่า Flyway

```
order-service/
├── src/
│   └── main/
│       └── resources/
│           └── db/
│               └── changelog/
│                   ├── db.changelog-master.xml
│                   ├── changes/
│                   │   ├── 001-create-orders-table.xml
│                   │   ├── 002-add-order-items.xml
│                   │   └── 003-add-indexes.sql
│                   └── rollback/
│                       └── rollback-001.sql
```

### db.changelog-master.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <property name="autoIncrement" value="true" dbms="postgresql"/>

    <include file="changes/001-create-orders-table.xml" 
             relativeToChangelogFile="true"/>
    <include file="changes/002-add-order-items.xml" 
             relativeToChangelogFile="true"/>
    <include file="changes/003-add-indexes.sql" 
             relativeToChangelogFile="true"/>
</databaseChangeLog>
```

### 001-create-orders-table.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="001-create-orders-table" author="dev-team">
        <comment>Create orders table for order-service</comment>
        
        <createTable tableName="orders">
            <column name="id" type="UUID" defaultValueComputed="gen_random_uuid()">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="user_id" type="UUID">
                <constraints nullable="false"/>
            </column>
            <column name="status" type="VARCHAR(20)" defaultValue="PENDING">
                <constraints nullable="false"/>
            </column>
            <column name="total_amount" type="DECIMAL(12,2)">
                <constraints nullable="false"/>
            </column>
            <column name="currency" type="CHAR(3)" defaultValue="THB">
                <constraints nullable="false"/>
            </column>
            <column name="shipping_address" type="JSONB"/>
            <column name="metadata" type="JSONB" defaultValue="{}"/>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="NOW()">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP" defaultValueComputed="NOW()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <!-- Check constraint สำหรับ status -->
        <sql>
            ALTER TABLE orders 
            ADD CONSTRAINT check_order_status 
            CHECK (status IN ('PENDING', 'CONFIRMED', 'PROCESSING', 
                             'SHIPPED', 'DELIVERED', 'CANCELLED', 'REFUNDED'));
        </sql>

        <!-- Index -->
        <createIndex tableName="orders" indexName="idx_orders_user_id">
            <column name="user_id"/>
        </createIndex>

        <createIndex tableName="orders" indexName="idx_orders_status">
            <column name="status"/>
        </createIndex>

        <!-- Rollback definition -->
        <rollback>
            <dropTable tableName="orders" cascadeConstraints="true"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

---

## 3. Expand-Contract Pattern (Zero-Downtime Migration)

นี่คือ pattern ที่สำคัญที่สุดสำหรับการทำ migration แบบ zero-downtime

### หลักการทำงาน

```
Phase 1: EXPAND
  - เพิ่ม columns/tables ใหม่ (nullable)
  - ยังไม่ลบของเก่า
  - Deploy service version ที่ write ทั้ง old และ new schema

Phase 2: MIGRATE (Data Backfill)
  - Copy/transform ข้อมูลจาก old → new format
  - ทำ gradually ไม่ใช่ทีเดียว

Phase 3: CONTRACT
  - Deploy service version ที่ใช้แค่ new schema
  - ลบ old columns/tables

ตัวอย่าง: เปลี่ยน full_name → first_name + last_name
```

### Phase 1: Expand Migration

```sql
-- V10__expand_user_name_columns.sql
-- เพิ่ม columns ใหม่ (nullable เสมอในขั้นนี้)
ALTER TABLE users 
    ADD COLUMN IF NOT EXISTS first_name VARCHAR(100),
    ADD COLUMN IF NOT EXISTS last_name  VARCHAR(100);

-- สร้าง function เพื่อ auto-fill ค่าจาก full_name
-- สำหรับ backward compatibility ระหว่าง migration
CREATE OR REPLACE FUNCTION split_full_name()
RETURNS TRIGGER AS $$
DECLARE
    parts TEXT[];
BEGIN
    IF NEW.first_name IS NULL AND NEW.full_name IS NOT NULL THEN
        parts := string_to_array(trim(NEW.full_name), ' ');
        NEW.first_name := parts[1];
        NEW.last_name  := array_to_string(parts[2:array_length(parts, 1)], ' ');
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER auto_split_name
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION split_full_name();
```

### Phase 1: Service Code (Dual Write)

```java
// UserService.java - Version ที่ write ทั้ง old และ new schema
@Service
@Slf4j
public class UserService {

    @Transactional
    public User createUser(CreateUserRequest request) {
        User user = new User();
        
        // Write to new schema
        user.setFirstName(request.getFirstName());
        user.setLastName(request.getLastName());
        
        // Also write to old schema (backward compatibility)
        // จะถูกลบใน Phase 3
        String fullName = request.getFirstName() + " " + request.getLastName();
        user.setFullName(fullName.trim());
        
        return userRepository.save(user);
    }

    public UserResponse getUser(UUID userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        UserResponse response = new UserResponse();
        
        // Read จาก new schema ถ้ามีข้อมูล ไม่งั้น fallback ไป old schema
        if (user.getFirstName() != null) {
            response.setFirstName(user.getFirstName());
            response.setLastName(user.getLastName());
        } else if (user.getFullName() != null) {
            // Fallback: parse จาก full_name
            String[] parts = user.getFullName().split(" ", 2);
            response.setFirstName(parts[0]);
            response.setLastName(parts.length > 1 ? parts[1] : "");
        }
        
        return response;
    }
}
```

### Phase 2: Data Backfill

```java
// MigrationBackfillJob.java
@Component
@Slf4j
public class MigrationBackfillJob {

    private final JdbcTemplate jdbcTemplate;
    private final MeterRegistry meterRegistry;

    // ทำ backfill แบบ batch เพื่อไม่ให้ lock table นานเกินไป
    @Scheduled(fixedDelay = 5000) // ทุก 5 วินาที
    @ConditionalOnProperty(name = "migration.backfill.enabled", havingValue = "true")
    public void backfillUserNames() {
        int batchSize = 1000;
        int processed = 0;
        
        do {
            processed = backfillBatch(batchSize);
            log.info("Backfilled {} users", processed);
            
            // Track progress
            meterRegistry.counter("migration.backfill.processed", 
                "table", "users").increment(processed);
            
            if (processed > 0) {
                // Short sleep เพื่อลด database load
                try { Thread.sleep(100); } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        } while (processed == batchSize);
        
        log.info("Backfill completed");
    }

    @Transactional
    private int backfillBatch(int batchSize) {
        // Update เฉพาะ rows ที่ยังไม่ได้ migrate
        String sql = """
            UPDATE users 
            SET 
                first_name = SPLIT_PART(TRIM(full_name), ' ', 1),
                last_name = SUBSTRING(TRIM(full_name) FROM POSITION(' ' IN TRIM(full_name)) + 1)
            WHERE first_name IS NULL 
              AND full_name IS NOT NULL
              AND full_name != ''
            LIMIT ?
            """;
        
        return jdbcTemplate.update(sql, batchSize);
    }
}
```

### Phase 2: SQL-based Backfill Script

```sql
-- backfill_user_names.sql
-- สำหรับ backfill ปริมาณมากๆ ใช้ DO block พร้อม pagination

DO $$
DECLARE
    batch_size  INT := 5000;
    total_rows  INT := 0;
    updated     INT;
    last_id     UUID := '00000000-0000-0000-0000-000000000000';
BEGIN
    LOOP
        UPDATE users 
        SET 
            first_name = SPLIT_PART(TRIM(full_name), ' ', 1),
            last_name = CASE 
                WHEN POSITION(' ' IN TRIM(full_name)) > 0 
                THEN TRIM(SUBSTRING(full_name FROM POSITION(' ' IN TRIM(full_name)) + 1))
                ELSE ''
            END
        WHERE id IN (
            SELECT id FROM users 
            WHERE first_name IS NULL 
              AND full_name IS NOT NULL
              AND id > last_id
            ORDER BY id
            LIMIT batch_size
        );
        
        GET DIAGNOSTICS updated = ROW_COUNT;
        total_rows := total_rows + updated;
        
        EXIT WHEN updated = 0;
        
        -- Update last_id สำหรับ batch ต่อไป
        SELECT MAX(id) INTO last_id 
        FROM users 
        WHERE first_name IS NOT NULL
          AND id > last_id;
        
        RAISE NOTICE 'Processed % rows, total: %', updated, total_rows;
        
        -- Short pause เพื่อลด I/O pressure
        PERFORM pg_sleep(0.1);
    END LOOP;
    
    RAISE NOTICE 'Backfill complete. Total rows updated: %', total_rows;
END $$;
```

### Phase 3: Contract Migration

```sql
-- V12__contract_remove_full_name.sql
-- ลบ old column หลังจาก backfill เสร็จสมบูรณ์
-- ก่อน run ต้องตรวจสอบว่าไม่มี rows ที่ first_name IS NULL

DO $$
DECLARE
    null_count INT;
BEGIN
    SELECT COUNT(*) INTO null_count 
    FROM users 
    WHERE first_name IS NULL AND full_name IS NOT NULL;
    
    IF null_count > 0 THEN
        RAISE EXCEPTION 'Cannot contract: % rows still need backfill', null_count;
    END IF;
END $$;

-- ลบ trigger ที่ไม่จำเป็นแล้ว
DROP TRIGGER IF EXISTS auto_split_name ON users;
DROP FUNCTION IF EXISTS split_full_name();

-- ตอนนี้ safe ที่จะ make first_name NOT NULL
ALTER TABLE users ALTER COLUMN first_name SET NOT NULL;

-- ลบ old column
ALTER TABLE users DROP COLUMN IF EXISTS full_name;

-- เพิ่ม index ใหม่สำหรับ search
CREATE INDEX CONCURRENTLY idx_users_name 
ON users(first_name, last_name);
```

---

## 4. Blue-Green Database Migrations

```
Blue Environment (Current Production)
├── App Servers (Blue)
└── Database Blue (PostgreSQL Primary)
    └── Replica →

Green Environment (New Version)
├── App Servers (Green) - New code
└── Database Green (PostgreSQL) - Migrated schema
    
Traffic Switch: Blue → Green (instant)
```

### Blue-Green Migration Script

```bash
#!/bin/bash
# blue-green-db-migration.sh

set -euo pipefail

BLUE_DB_HOST="${BLUE_DB_HOST:-blue-postgres}"
GREEN_DB_HOST="${GREEN_DB_HOST:-green-postgres}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-orderdb}"
DB_USER="${DB_USER:-postgres}"
MIGRATION_TIMEOUT="${MIGRATION_TIMEOUT:-300}"

log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

check_replication_lag() {
    local primary_host="$1"
    local replica_host="$2"
    
    local lag_bytes
    lag_bytes=$(psql "postgresql://${DB_USER}@${replica_host}:${DB_PORT}/${DB_NAME}" \
        -t -c "SELECT pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn())" \
        2>/dev/null || echo "999999999")
    
    echo "${lag_bytes:-999999999}"
}

wait_for_sync() {
    local source_host="$1"
    local target_host="$2"
    local max_wait="$3"
    local waited=0
    
    log "Waiting for ${target_host} to sync with ${source_host}..."
    
    while [ $waited -lt $max_wait ]; do
        local lag
        lag=$(check_replication_lag "$source_host" "$target_host")
        
        if [ "$lag" -lt 1000 ]; then
            log "Sync achieved! Lag: ${lag} bytes"
            return 0
        fi
        
        log "Waiting... lag: ${lag} bytes (${waited}s elapsed)"
        sleep 5
        waited=$((waited + 5))
    done
    
    log "ERROR: Sync timeout after ${max_wait}s"
    return 1
}

# Step 1: Validate blue environment
log "Step 1: Validating blue environment..."
psql "postgresql://${DB_USER}@${BLUE_DB_HOST}:${DB_PORT}/${DB_NAME}" \
    -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public'"

# Step 2: Set up green database with latest data
log "Step 2: Setting up green database..."

# ใช้ pg_dump/restore หรือ logical replication
pg_dump \
    --no-owner \
    --no-privileges \
    --schema-only \
    -h "${BLUE_DB_HOST}" \
    -U "${DB_USER}" \
    "${DB_NAME}" | \
psql \
    -h "${GREEN_DB_HOST}" \
    -U "${DB_USER}" \
    "${DB_NAME}"

# Step 3: Run new migrations on green
log "Step 3: Running migrations on green..."
flyway \
    -url="jdbc:postgresql://${GREEN_DB_HOST}:${DB_PORT}/${DB_NAME}" \
    -user="${DB_USER}" \
    -password="${DB_PASSWORD}" \
    migrate

# Step 4: Wait for data to sync
wait_for_sync "${BLUE_DB_HOST}" "${GREEN_DB_HOST}" "${MIGRATION_TIMEOUT}"

# Step 5: Test green environment
log "Step 5: Testing green environment..."
# Run smoke tests against green
curl -f "http://green-app-server/health" || {
    log "ERROR: Green health check failed"
    exit 1
}

# Step 6: Switch traffic (atomic operation)
log "Step 6: Switching traffic to green..."
# Update load balancer / service discovery
kubectl patch service order-service \
    -p '{"spec":{"selector":{"environment":"green"}}}'

log "Blue-Green migration complete!"
log "Monitor for issues before decommissioning blue environment"
```

---

## 5. Rolling Migration with Backward Compatibility

```java
// DatabaseCompatibilityConfig.java
@Configuration
public class DatabaseCompatibilityConfig {

    // Schema version tracking
    @Bean
    public SchemaVersionChecker schemaVersionChecker(DataSource dataSource) {
        return new SchemaVersionChecker(dataSource);
    }
}

@Component
@Slf4j
public class SchemaVersionChecker implements ApplicationListener<ApplicationReadyEvent> {

    private final JdbcTemplate jdbcTemplate;

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        checkSchemaCompatibility();
    }

    private void checkSchemaCompatibility() {
        try {
            // Check minimum required schema version
            String currentVersion = jdbcTemplate.queryForObject(
                "SELECT version FROM schema_versions ORDER BY applied_at DESC LIMIT 1",
                String.class
            );
            
            String minRequired = "5.0.0";
            if (!isVersionCompatible(currentVersion, minRequired)) {
                throw new SchemaVersionMismatchException(
                    "Schema version " + currentVersion + 
                    " is incompatible with minimum required " + minRequired
                );
            }
            
            log.info("Schema version {} is compatible", currentVersion);
        } catch (EmptyResultDataAccessException e) {
            log.warn("No schema version found, assuming fresh database");
        }
    }

    private boolean isVersionCompatible(String current, String minimum) {
        // Semantic versioning comparison
        String[] currentParts = current.split("\\.");
        String[] minParts = minimum.split("\\.");
        
        for (int i = 0; i < Math.min(currentParts.length, minParts.length); i++) {
            int c = Integer.parseInt(currentParts[i]);
            int m = Integer.parseInt(minParts[i]);
            if (c > m) return true;
            if (c < m) return false;
        }
        return true;
    }
}
```

### Rolling Migration Strategy

```yaml
# kubernetes/rolling-migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v35
  namespace: production
  labels:
    app: user-service
    migration-version: "35"
spec:
  backoffLimit: 3
  template:
    metadata:
      labels:
        app: db-migration
    spec:
      restartPolicy: OnFailure
      initContainers:
        - name: wait-for-db
          image: postgres:15-alpine
          command:
            - sh
            - -c
            - |
              until pg_isready -h $DB_HOST -p $DB_PORT -U $DB_USER; do
                echo "Waiting for database..."
                sleep 2
              done
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
            - name: DB_PORT
              value: "5432"
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username

      containers:
        - name: flyway-migration
          image: flyway/flyway:9.22
          args:
            - migrate
            - -url=jdbc:postgresql://$(DB_HOST):$(DB_PORT)/$(DB_NAME)
            - -user=$(DB_USER)
            - -password=$(DB_PASSWORD)
            - -locations=filesystem:/flyway/sql
            - -outOfOrder=false
            - -validateOnMigrate=true
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: userdb
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          volumeMounts:
            - name: migration-scripts
              mountPath: /flyway/sql
          resources:
            requests:
              memory: 256Mi
              cpu: 100m
            limits:
              memory: 512Mi
              cpu: 500m

      volumes:
        - name: migration-scripts
          configMap:
            name: migration-scripts-v35
```

---

## 6. Schema Versioning with PostgreSQL

```sql
-- schema_versions table สำหรับ tracking
CREATE TABLE IF NOT EXISTS schema_versions (
    id              SERIAL PRIMARY KEY,
    version         VARCHAR(20) NOT NULL,
    description     TEXT,
    script_name     VARCHAR(255) NOT NULL,
    checksum        VARCHAR(64),
    applied_by      VARCHAR(100) DEFAULT current_user,
    applied_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    execution_time  INT, -- milliseconds
    success         BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE UNIQUE INDEX ON schema_versions(version);

-- Views สำหรับ monitoring
CREATE OR REPLACE VIEW v_migration_status AS
SELECT 
    version,
    description,
    applied_at,
    execution_time || 'ms' AS duration,
    CASE WHEN success THEN '✓' ELSE '✗' END AS status,
    applied_by
FROM schema_versions
ORDER BY applied_at DESC;
```

```sql
-- PostgreSQL-specific optimizations สำหรับ migration
-- ใช้ CONCURRENTLY เพื่อสร้าง index โดยไม่ lock table

-- สร้าง index แบบ non-blocking
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_user_date 
ON orders(user_id, created_at DESC)
WHERE status != 'CANCELLED';

-- Partial index สำหรับ hot queries
CREATE INDEX CONCURRENTLY idx_orders_pending
ON orders(created_at)
WHERE status = 'PENDING';

-- ตรวจสอบ progress ของ CONCURRENTLY operation
SELECT 
    phase,
    blocks_done,
    blocks_total,
    ROUND(100.0 * blocks_done / NULLIF(blocks_total, 0), 1) AS progress_pct
FROM pg_stat_progress_create_index;
```

---

## 7. Testing Database Migrations

```java
// MigrationTest.java
@SpringBootTest
@Testcontainers
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class MigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        .withReuse(true);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    @Order(1)
    void shouldApplyAllMigrationsSuccessfully() {
        // Verify all migrations applied
        Integer count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM flyway_schema_history WHERE success = true",
            Integer.class
        );
        assertThat(count).isGreaterThan(0);
    }

    @Test
    @Order(2)
    void shouldHaveCorrectTableStructure() {
        // Verify tables exist
        List<String> tables = jdbcTemplate.queryForList(
            "SELECT table_name FROM information_schema.tables " +
            "WHERE table_schema = 'public' ORDER BY table_name",
            String.class
        );
        
        assertThat(tables).contains("users", "orders", "order_items");
    }

    @Test
    @Order(3)
    void shouldHaveRequiredIndexes() {
        List<String> indexes = jdbcTemplate.queryForList(
            "SELECT indexname FROM pg_indexes " +
            "WHERE schemaname = 'public' ORDER BY indexname",
            String.class
        );
        
        assertThat(indexes).contains(
            "idx_users_email",
            "idx_users_active_email",
            "idx_orders_user_id"
        );
    }

    @Test
    @Order(4)
    void shouldEnforceConstraints() {
        // Test check constraint
        assertThrows(DataIntegrityViolationException.class, () -> {
            jdbcTemplate.execute(
                "INSERT INTO orders(user_id, status, total_amount) " +
                "VALUES(gen_random_uuid(), 'INVALID_STATUS', 100.00)"
            );
        });
    }

    @Test
    @Order(5)
    void backfillMigrationShouldBeIdempotent() {
        // Insert test data
        jdbcTemplate.execute(
            "INSERT INTO users(username, email, password_hash, full_name) " +
            "VALUES('test', 'test@example.com', 'hash', 'John Doe')"
        );
        
        // Run backfill twice
        for (int i = 0; i < 2; i++) {
            jdbcTemplate.update(
                "UPDATE users SET " +
                "first_name = SPLIT_PART(TRIM(full_name), ' ', 1), " +
                "last_name = SPLIT_PART(TRIM(full_name), ' ', 2) " +
                "WHERE first_name IS NULL AND full_name IS NOT NULL"
            );
        }
        
        // Verify result
        Map<String, Object> user = jdbcTemplate.queryForMap(
            "SELECT first_name, last_name FROM users WHERE username = 'test'"
        );
        assertThat(user.get("first_name")).isEqualTo("John");
        assertThat(user.get("last_name")).isEqualTo("Doe");
    }
}
```

### Migration Performance Testing

```python
# migration_perf_test.py
import psycopg2
import time
import statistics
from contextlib import contextmanager

@contextmanager
def timer(description):
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"{description}: {elapsed*1000:.2f}ms")

def test_migration_performance(conn_string: str):
    """Test that migration completes within acceptable time"""
    
    conn = psycopg2.connect(conn_string)
    conn.autocommit = False
    cursor = conn.cursor()
    
    # Generate test data
    print("Generating test data...")
    cursor.execute("""
        INSERT INTO users(username, email, password_hash, full_name)
        SELECT 
            'user' || i,
            'user' || i || '@example.com',
            'hash' || i,
            'User ' || i
        FROM generate_series(1, 100000) i
    """)
    conn.commit()
    
    # Test backfill performance
    execution_times = []
    
    for run in range(3):
        # Reset data
        cursor.execute("UPDATE users SET first_name = NULL, last_name = NULL")
        conn.commit()
        
        with timer(f"Backfill run {run + 1}"):
            start = time.perf_counter()
            
            batch_size = 5000
            while True:
                cursor.execute("""
                    UPDATE users 
                    SET 
                        first_name = SPLIT_PART(TRIM(full_name), ' ', 1),
                        last_name  = SPLIT_PART(TRIM(full_name), ' ', 2)
                    WHERE first_name IS NULL 
                      AND full_name IS NOT NULL
                    LIMIT %s
                """, (batch_size,))
                
                if cursor.rowcount == 0:
                    break
                    
                conn.commit()
            
            elapsed = time.perf_counter() - start
            execution_times.append(elapsed)
    
    print(f"\nPerformance Summary:")
    print(f"  Min: {min(execution_times)*1000:.0f}ms")
    print(f"  Max: {max(execution_times)*1000:.0f}ms")
    print(f"  Avg: {statistics.mean(execution_times)*1000:.0f}ms")
    
    # Assert performance requirement
    assert max(execution_times) < 60, f"Migration too slow: {max(execution_times):.1f}s"
    print("\nPerformance test PASSED!")

if __name__ == "__main__":
    test_migration_performance("postgresql://test:test@localhost:5432/testdb")
```

---

## 8. Rollback Procedures

```sql
-- rollback_v12.sql
-- Rollback script สำหรับ V12 (contract phase)
-- ต้อง run ก่อน deploy version เก่า

BEGIN;

-- Step 1: Re-add full_name column
ALTER TABLE users ADD COLUMN IF NOT EXISTS full_name VARCHAR(200);

-- Step 2: Populate full_name จาก first_name + last_name
UPDATE users 
SET full_name = TRIM(CONCAT(first_name, ' ', last_name))
WHERE full_name IS NULL;

-- Step 3: Re-create trigger
CREATE OR REPLACE FUNCTION sync_full_name()
RETURNS TRIGGER AS $$
BEGIN
    NEW.full_name = TRIM(CONCAT(NEW.first_name, ' ', NEW.last_name));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_full_name_trigger
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION sync_full_name();

-- Step 4: Allow nullable again (for rollback compatibility)
ALTER TABLE users ALTER COLUMN first_name DROP NOT NULL;

-- Step 5: Update flyway history to mark as rolled back
UPDATE flyway_schema_history 
SET success = FALSE
WHERE version = '12';

COMMIT;

-- Verify rollback
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'users' 
  AND column_name IN ('full_name', 'first_name', 'last_name')
ORDER BY column_name;
```

### Automated Rollback with Health Checks

```java
// MigrationRollbackManager.java
@Component
@Slf4j
public class MigrationRollbackManager {

    private final Flyway flyway;
    private final ApplicationContext context;
    private final AlertingService alertingService;

    // ตรวจสอบหลัง migration และ rollback ถ้าจำเป็น
    @EventListener(ApplicationReadyEvent.class)
    public void postMigrationHealthCheck() {
        if (!isMigrationHealthy()) {
            log.error("Post-migration health check FAILED - initiating rollback");
            
            alertingService.sendAlert(
                "CRITICAL: Database migration failed health check",
                "Initiating automatic rollback for service " + getServiceName()
            );
            
            performRollback();
        }
    }

    private boolean isMigrationHealthy() {
        try {
            // Basic connectivity check
            context.getBean(JdbcTemplate.class).queryForObject(
                "SELECT 1", Integer.class
            );
            
            // Schema validation
            flyway.validate();
            
            // Business logic check
            return runSmokeTests();
            
        } catch (Exception e) {
            log.error("Health check failed: {}", e.getMessage());
            return false;
        }
    }

    private boolean runSmokeTests() {
        JdbcTemplate jdbcTemplate = context.getBean(JdbcTemplate.class);
        
        // Test CRUD operations
        try {
            jdbcTemplate.execute(
                "INSERT INTO users(username, email, password_hash, first_name, last_name) " +
                "VALUES('__smoke_test__', '__smoke__@test.com', 'hash', 'Smoke', 'Test')"
            );
            
            Integer count = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM users WHERE username = '__smoke_test__'",
                Integer.class
            );
            
            jdbcTemplate.execute(
                "DELETE FROM users WHERE username = '__smoke_test__'"
            );
            
            return count != null && count == 1;
        } catch (Exception e) {
            log.error("Smoke test failed: {}", e.getMessage());
            return false;
        }
    }

    private void performRollback() {
        try {
            // Flyway undo (requires Teams edition)
            // flyway.undo();
            
            // Alternative: run manual rollback script
            Resource rollbackScript = new ClassPathResource("db/rollback/rollback_latest.sql");
            
            ScriptUtils.executeSqlScript(
                context.getBean(DataSource.class).getConnection(),
                rollbackScript
            );
            
            log.info("Rollback completed successfully");
            
        } catch (Exception e) {
            log.error("Rollback FAILED: {}", e.getMessage());
            alertingService.sendPagerAlert("DATABASE ROLLBACK FAILED - MANUAL INTERVENTION REQUIRED");
        }
    }
}
```

---

## 9. Monitoring Migration Progress

```java
// MigrationMetrics.java
@Component
public class MigrationMetrics {

    private final MeterRegistry meterRegistry;
    private final JdbcTemplate jdbcTemplate;

    @Scheduled(fixedRate = 30000) // ทุก 30 วินาที
    public void recordMigrationMetrics() {
        try {
            // นับจำนวน rows ที่ยังไม่ได้ migrate
            Integer pendingRows = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM users WHERE first_name IS NULL AND full_name IS NOT NULL",
                Integer.class
            );
            
            meterRegistry.gauge(
                "migration.pending_rows",
                Tags.of("table", "users", "type", "backfill"),
                pendingRows != null ? pendingRows : 0
            );
            
            // Progress percentage
            Integer totalRows = jdbcTemplate.queryForObject(
                "SELECT COUNT(*) FROM users",
                Integer.class
            );
            
            if (totalRows != null && totalRows > 0 && pendingRows != null) {
                double progress = 100.0 * (totalRows - pendingRows) / totalRows;
                meterRegistry.gauge(
                    "migration.progress_pct",
                    Tags.of("table", "users"),
                    progress
                );
            }
            
        } catch (Exception e) {
            // Migration table may not exist yet
        }
    }
}
```

---

## สรุป

การทำ Database Migration ใน Microservices ต้องคำนึงถึง:

1. **Expand-Contract Pattern** - เพิ่ม columns ใหม่ก่อน ค่อยๆ migrate ข้อมูล แล้วจึงลบของเก่า
2. **CONCURRENTLY** - ใช้สร้าง index โดยไม่ lock table ใน production
3. **Batch Backfill** - ทำ data migration เป็น batch เล็กๆ เพื่อลด load
4. **Health Checks** - ตรวจสอบ migration สำเร็จก่อน traffic เข้ามา
5. **Rollback Scripts** - เตรียม rollback ไว้เสมอและทดสอบให้แน่ใจว่าใช้ได้
6. **Separate Migration from Deployment** - run migration ก่อน deploy application

### ข้อควรจำ

- ห้ามใช้ `ddl-auto: create` หรือ `update` ใน production
- ทดสอบ migration script กับ production data copy เสมอ
- เก็บ rollback script คู่กับ migration script ทุกครั้ง
- Monitor migration progress ด้วย metrics

---

## แบบฝึกหัด

1. **Expand-Contract Exercise**: สร้าง migration สำหรับ `orders` table ที่ต้องการเปลี่ยน `address` text field เป็น `shipping_address` JSONB ที่มี fields: street, city, province, zipcode ตามขั้นตอน Expand-Contract

2. **Backfill Performance**: เขียน SQL script ที่ทำ backfill แบบ batch โดยมี configurable batch size และ sleep interval พร้อม progress tracking

3. **Migration Testing**: สร้าง Integration Test ที่ใช้ Testcontainers ตรวจสอบว่า migration script idempotent (run หลายครั้งให้ผลเหมือนกัน)

4. **Rollback Automation**: เขียน script ที่ตรวจสอบ migration health หลัง deploy และ rollback อัตโนมัติถ้าพบปัญหา โดยต้องส่ง Slack notification ทุกขั้นตอน

5. **Blue-Green Challenge**: ออกแบบ migration strategy สำหรับการเปลี่ยน primary key จาก `INT` เป็น `UUID` โดยไม่มี downtime สำหรับ table ที่มีข้อมูล 10 ล้าน rows

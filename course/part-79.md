# Part 79: Advanced Kubernetes Patterns

## บทนำ

Kubernetes มี patterns ขั้นสูงมากมายที่ช่วยให้ microservices ทำงานได้อย่างมีประสิทธิภาพ บทนี้จะครอบคลุม Pod Templates, Init Containers, Sidecar Containers, Ephemeral Containers สำหรับ debugging, DaemonSets, StatefulSets สำหรับ databases, Job/CronJob patterns และ Multi-container pod patterns

---

## 1. Pod Templates ขั้นสูง

### 1.1 Production-grade Pod Template

```yaml
# kubernetes/pod-template-production.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: production
  labels:
    app: product-service
    version: v3.2.1
    team: platform
    environment: production
  annotations:
    deployment.kubernetes.io/revision: "5"
    git-commit: "abc123def456"
    build-date: "2024-01-15T10:00:00Z"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: product-service
      version: v3.2.1
  template:
    metadata:
      labels:
        app: product-service
        version: v3.2.1
        team: platform
        environment: production
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
        sidecar.istio.io/inject: "true"
        cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    spec:
      serviceAccountName: product-service-sa
      automountServiceAccountToken: false

      # Security Context at Pod level
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
        sysctls:
          - name: net.ipv4.tcp_keepalive_time
            value: "300"

      # Node selection
      nodeSelector:
        kubernetes.io/os: linux
        node-type: compute
      
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - product-service
              topologyKey: kubernetes.io/hostname
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - product-service
                topologyKey: topology.kubernetes.io/zone

      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "compute"
          effect: "NoSchedule"

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: product-service
        - maxSkew: 2
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: product-service

      # Termination
      terminationGracePeriodSeconds: 60

      # DNS configuration
      dnsConfig:
        options:
          - name: ndots
            value: "2"
          - name: edns0
          - name: single-request-reopen

      # Init containers run before main containers
      initContainers:
        - name: db-migration
          image: registry.company.com/db-migrator:v1.2.0
          command: ["./migrate", "--direction=up", "--version=latest"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: product-service-secrets
                  key: database-url
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL

        - name: config-loader
          image: registry.company.com/config-loader:v1.0.0
          command: ["./load-config"]
          volumeMounts:
            - name: config-volume
              mountPath: /config
          env:
            - name: VAULT_ADDR
              value: "http://vault.vault.svc.cluster.local:8200"
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"

      containers:
        - name: product-service
          image: registry.company.com/product-service:v3.2.1
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: grpc
              containerPort: 9000
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP

          # Security Context at Container level
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE

          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "8080"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: MEMORY_LIMIT
              valueFrom:
                resourceFieldRef:
                  resource: limits.memory
                  divisor: "1Mi"
          
          envFrom:
            - configMapRef:
                name: product-service-config
            - secretRef:
                name: product-service-secrets

          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
              ephemeral-storage: "1Gi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
              ephemeral-storage: "2Gi"

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
              httpHeaders:
                - name: X-Health-Check
                  value: "liveness"
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
            successThreshold: 1

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
            successThreshold: 1

          startupProbe:
            httpGet:
              path: /health/startup
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 30    # 30 * 5s = 150s max startup time

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]

          volumeMounts:
            - name: config-volume
              mountPath: /app/config
              readOnly: true
            - name: tmp-dir
              mountPath: /tmp
            - name: cache-dir
              mountPath: /app/cache

        # Sidecar: Log aggregator
        - name: log-aggregator
          image: fluent/fluent-bit:2.2
          volumeMounts:
            - name: log-volume
              mountPath: /var/log/app
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true

      volumes:
        - name: config-volume
          configMap:
            name: product-service-config
            defaultMode: 0444
        - name: tmp-dir
          emptyDir:
            medium: Memory
            sizeLimit: 100Mi
        - name: cache-dir
          emptyDir:
            sizeLimit: 500Mi
        - name: log-volume
          emptyDir:
            sizeLimit: 1Gi
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-config

      imagePullSecrets:
        - name: registry-credentials
```

---

## 2. Init Containers

### 2.1 Database Migration Init Container

```typescript
// init-containers/db-migrator/src/migrate.ts
import { Pool } from 'pg';
import fs from 'fs/promises';
import path from 'path';
import { createHash } from 'crypto';

interface Migration {
  version: number;
  name: string;
  filename: string;
  checksum: string;
}

async function runMigrations(): Promise<void> {
  const db = new Pool({ connectionString: process.env.DATABASE_URL });

  // Ensure migrations table exists
  await db.query(`
    CREATE TABLE IF NOT EXISTS schema_migrations (
      version BIGINT PRIMARY KEY,
      name VARCHAR(255) NOT NULL,
      checksum VARCHAR(64) NOT NULL,
      applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
    )
  `);

  // Get applied migrations
  const { rows: applied } = await db.query<{ version: number }>(
    'SELECT version FROM schema_migrations ORDER BY version ASC'
  );
  const appliedVersions = new Set(applied.map((r) => r.version));

  // Load migration files
  const migrationsDir = process.env.MIGRATIONS_DIR || './migrations';
  const files = (await fs.readdir(migrationsDir))
    .filter((f) => f.endsWith('.sql'))
    .sort();

  let migrationsRun = 0;

  for (const filename of files) {
    const match = filename.match(/^(\d+)_(.+)\.sql$/);
    if (!match) continue;

    const version = parseInt(match[1]);
    const name = match[2].replace(/_/g, ' ');

    if (appliedVersions.has(version)) continue;

    const sql = await fs.readFile(path.join(migrationsDir, filename), 'utf-8');
    const checksum = createHash('sha256').update(sql).digest('hex');

    const client = await db.connect();
    try {
      await client.query('BEGIN');
      
      // Run migration
      await client.query(sql);
      
      // Record migration
      await client.query(
        'INSERT INTO schema_migrations (version, name, checksum) VALUES ($1, $2, $3)',
        [version, name, checksum]
      );

      await client.query('COMMIT');
      console.log(`✓ Applied migration ${version}: ${name}`);
      migrationsRun++;
    } catch (error: any) {
      await client.query('ROLLBACK');
      console.error(`✗ Failed migration ${version}: ${error.message}`);
      process.exit(1);
    } finally {
      client.release();
    }
  }

  if (migrationsRun === 0) {
    console.log('No new migrations to apply');
  } else {
    console.log(`Applied ${migrationsRun} migration(s)`);
  }

  await db.end();
}

runMigrations().catch((err) => {
  console.error('Migration failed:', err);
  process.exit(1);
});
```

---

## 3. Sidecar Container Patterns

### 3.1 Ambassador Sidecar (Service Proxy)

```yaml
# kubernetes/ambassador-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  template:
    spec:
      containers:
        # Main application
        - name: order-service
          image: registry.company.com/order-service:v2.1.0
          env:
            # Use ambassador sidecar for external calls
            - name: PAYMENT_SERVICE_URL
              value: "http://localhost:9001"  # Ambassador port
            - name: INVENTORY_SERVICE_URL
              value: "http://localhost:9002"
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"

        # Ambassador: payment service proxy
        - name: payment-proxy
          image: envoyproxy/envoy:v1.28
          ports:
            - containerPort: 9001
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
          command: ["envoy", "-c", "/etc/envoy/payment-proxy.yaml"]
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

        # Ambassador: inventory service proxy with circuit breaker
        - name: inventory-proxy
          image: envoyproxy/envoy:v1.28
          ports:
            - containerPort: 9002
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
          command: ["envoy", "-c", "/etc/envoy/inventory-proxy.yaml"]
          resources:
            requests:
              cpu: "100m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"

        # Adapter sidecar: Transform external data format
        - name: data-adapter
          image: registry.company.com/data-adapter:v1.0.0
          ports:
            - containerPort: 9003
          env:
            - name: LEGACY_API_URL
              value: "https://legacy.internal.company.com"
            - name: TRANSFORM_FORMAT
              value: "json-to-protobuf"
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"

      volumes:
        - name: envoy-config
          configMap:
            name: envoy-sidecar-configs
```

### 3.2 Log Collection Sidecar

```yaml
# kubernetes/log-sidecar.yaml
# ConfigMap for Fluent Bit
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: production
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Daemon       Off
        Log_Level    info
        Parsers_File parsers.conf
        HTTP_Server  On
        HTTP_Listen  0.0.0.0
        HTTP_Port    2020

    [INPUT]
        Name              tail
        Path              /var/log/app/*.log
        Parser            json
        Tag               app.*
        Refresh_Interval  5
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        DB                /tmp/flb_input.db

    [FILTER]
        Name    record_modifier
        Match   *
        Record  service product-service
        Record  environment production
        Record  cluster k8s-prod-cluster

    [FILTER]
        Name   grep
        Match  *
        Regex  level (error|warn|info)

    [OUTPUT]
        Name              es
        Match             *
        Host              elasticsearch.logging.svc
        Port              9200
        Logstash_Format   On
        Logstash_Prefix   app-logs
        Retry_Limit       5
        tls               On
        tls.verify        Off
        HTTP_User         ${ELASTIC_USER}
        HTTP_Passwd       ${ELASTIC_PASSWORD}
        Buffer_Size       5MB
        Compress          gzip

  parsers.conf: |
    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ
        Time_Keep   On
        Decode_Field_As escaped_utf8 message do_next
        Decode_Field_As json message
```

---

## 4. Ephemeral Containers สำหรับ Debugging

### 4.1 Debugging Runbook

```bash
#!/bin/bash
# scripts/debug-pod.sh
# ใช้สำหรับ debug production pods โดยไม่ต้อง restart

POD_NAME=$1
NAMESPACE=${2:-production}
DEBUG_IMAGE=${3:-"registry.company.com/debug-tools:latest"}

echo "Adding ephemeral debug container to $POD_NAME in $NAMESPACE"

# Add ephemeral container
kubectl debug -it $POD_NAME \
  --image=$DEBUG_IMAGE \
  --target=product-service \
  --namespace=$NAMESPACE \
  -- /bin/bash

# Alternative: specific tools
kubectl debug $POD_NAME \
  --image=busybox:latest \
  --namespace=$NAMESPACE \
  -- sh -c "
    echo 'Checking network connectivity...'
    wget -qO- http://product-service:8080/health/live
    
    echo 'Checking DNS resolution...'
    nslookup postgres-service.production.svc.cluster.local
    
    echo 'Checking environment variables...'
    env | grep -v 'PASSWORD\|SECRET\|KEY'
    
    echo 'Checking file system...'
    df -h
    
    echo 'Process information...'
    ps aux
  "
```

### 4.2 Debug Container Image

```dockerfile
# debug-tools/Dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    wget \
    netcat-openbsd \
    dnsutils \
    net-tools \
    tcpdump \
    strace \
    htop \
    jq \
    postgresql-client \
    redis-tools \
    python3 \
    python3-pip \
    vim \
    less \
    && rm -rf /var/lib/apt/lists/*

# Install specialized tools
RUN curl -L https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz | \
    tar -xz -C /usr/local/bin

COPY scripts/ /debug-scripts/
RUN chmod +x /debug-scripts/*.sh

WORKDIR /tmp
CMD ["/bin/bash"]
```

---

## 5. DaemonSets

### 5.1 Node Monitoring DaemonSet

```yaml
# kubernetes/monitoring-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitoring-agent
  namespace: monitoring
  labels:
    app: node-monitoring
spec:
  selector:
    matchLabels:
      app: node-monitoring
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # Update one node at a time
  template:
    metadata:
      labels:
        app: node-monitoring
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      serviceAccountName: node-monitoring-sa
      hostNetwork: true      # Access node network
      hostPID: true          # Access node processes
      dnsPolicy: ClusterFirstWithHostNet
      
      tolerations:
        - operator: Exists     # Run on ALL nodes including masters
      
      priorityClassName: system-node-critical

      containers:
        # Node exporter for system metrics
        - name: node-exporter
          image: prom/node-exporter:v1.6.1
          args:
            - "--path.rootfs=/host"
            - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
            - "--collector.netclass.ignored-devices=^(veth|docker|lo|flannel|cni).*$"
            - "--web.listen-address=:9100"
          ports:
            - containerPort: 9100
              hostPort: 9100
              name: metrics
          resources:
            requests:
              cpu: "50m"
              memory: "30Mi"
            limits:
              cpu: "250m"
              memory: "180Mi"
          volumeMounts:
            - name: rootfs
              mountPath: /host
              readOnly: true
              mountPropagation: HostToContainer
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true

        # Log collection agent
        - name: log-collector
          image: registry.company.com/log-collector:v2.0.0
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: LOG_OUTPUT_URL
              value: "https://logs.company.com"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

        # Security scanner
        - name: falco
          image: falcosecurity/falco-no-driver:0.36.2
          securityContext:
            privileged: true
          volumeMounts:
            - name: falco-config
              mountPath: /etc/falco
            - name: dev
              mountPath: /host/dev
            - name: proc
              mountPath: /host/proc
              readOnly: true

      volumes:
        - name: rootfs
          hostPath:
            path: /
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: dev
          hostPath:
            path: /dev
        - name: proc
          hostPath:
            path: /proc
        - name: falco-config
          configMap:
            name: falco-config
```

---

## 6. StatefulSets สำหรับ Databases

### 6.1 PostgreSQL StatefulSet

```yaml
# kubernetes/postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  podManagementPolicy: OrderedReady   # Start/stop in order
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0    # Update all pods
  template:
    metadata:
      labels:
        app: postgres
    spec:
      serviceAccountName: postgres-sa
      securityContext:
        fsGroup: 999  # postgres group
        runAsUser: 999
        runAsGroup: 999

      initContainers:
        - name: postgres-init
          image: registry.company.com/postgres-init:v1.0.0
          command:
            - bash
            - "-c"
            - |
              set -ex
              # Determine if this pod is primary or replica based on index
              [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
              ordinal=${BASH_REMATCH[1]}
              
              if [[ $ordinal -eq 0 ]]; then
                echo "primary" > /etc/postgres/role
              else
                echo "replica" > /etc/postgres/role
                echo "primary_conninfo = 'host=postgres-0.postgres-headless port=5432 user=replicator password=${REPLICATION_PASSWORD}'" >> /etc/postgres/recovery.conf
              fi
          env:
            - name: REPLICATION_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: replication-password
          volumeMounts:
            - name: postgres-config
              mountPath: /etc/postgres

      containers:
        - name: postgres
          image: postgres:16.1
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: superuser
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: password
            - name: POSTGRES_DB
              value: production_db
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POSTGRES_REPLICATION_USER
              value: replicator
            - name: POSTGRES_REPLICATION_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: replication-password
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "8"
              memory: "16Gi"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
            - name: postgres-config
              mountPath: /etc/postgresql/conf.d
            - name: postgres-init-scripts
              mountPath: /docker-entrypoint-initdb.d
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - postgres
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U postgres && psql -U postgres -c "SELECT 1"
            initialDelaySeconds: 5
            periodSeconds: 5

        # Backup sidecar
        - name: pgbackup
          image: registry.company.com/pg-backup:v1.0.0
          env:
            - name: S3_BUCKET
              value: company-postgres-backups
            - name: PGHOST
              value: localhost
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

      volumes:
        - name: postgres-config
          configMap:
            name: postgres-config
        - name: postgres-init-scripts
          configMap:
            name: postgres-init-scripts
        - name: postgres-temp
          emptyDir: {}

  volumeClaimTemplates:
    - metadata:
        name: postgres-data
        annotations:
          volume.beta.kubernetes.io/storage-class: "gp3-encrypted"
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3-encrypted
        resources:
          requests:
            storage: 500Gi
---
# Headless service for stable network identities
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None   # Headless
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgres
---
# Regular service for reads (all replicas)
apiVersion: v1
kind: Service
metadata:
  name: postgres-read
  namespace: production
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgres
---
# Service for primary only
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: production
spec:
  selector:
    app: postgres
    role: primary
  ports:
    - port: 5432
      name: postgres
```

---

## 7. Job และ CronJob Patterns

### 7.1 Batch Processing Job

```yaml
# kubernetes/batch-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: product-data-import-20240115
  namespace: production
  labels:
    app: data-import
    batch-id: "20240115-001"
spec:
  completions: 10        # Total jobs to complete
  parallelism: 3         # Run 3 at a time
  completionMode: Indexed # Each pod gets unique index
  backoffLimit: 3        # Retry failed pods max 3 times
  activeDeadlineSeconds: 3600  # Job timeout: 1 hour
  ttlSecondsAfterFinished: 86400  # Clean up after 24h
  template:
    metadata:
      labels:
        app: data-import
        job-type: product-import
    spec:
      restartPolicy: OnFailure
      serviceAccountName: batch-job-sa

      initContainers:
        - name: wait-for-db
          image: postgres:16-alpine
          command:
            - sh
            - -c
            - |
              until pg_isready -h $POSTGRES_HOST -U $POSTGRES_USER; do
                echo "Waiting for database..."
                sleep 2
              done
          env:
            - name: POSTGRES_HOST
              value: postgres-primary
            - name: POSTGRES_USER
              value: postgres

      containers:
        - name: importer
          image: registry.company.com/data-importer:v1.2.0
          command:
            - "/app/import"
            - "--partition=$(JOB_COMPLETION_INDEX)"
            - "--total-partitions=10"
            - "--batch-size=1000"
          env:
            - name: JOB_COMPLETION_INDEX
              valueFrom:
                fieldRef:
                  fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: database-secret
                  key: url
            - name: S3_SOURCE_PATH
              value: s3://data-lake/products/2024-01-15/
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          volumeMounts:
            - name: tmp-volume
              mountPath: /tmp
      
      volumes:
        - name: tmp-volume
          emptyDir:
            sizeLimit: 5Gi
```

### 7.2 CronJob สำหรับ Scheduled Tasks

```yaml
# kubernetes/cronjob.yaml
# Daily analytics aggregation
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-analytics
  namespace: production
spec:
  schedule: "0 1 * * *"          # 01:00 AM daily (UTC)
  timeZone: "Asia/Bangkok"        # Use Bangkok timezone
  concurrencyPolicy: Forbid       # Don't run if previous still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  startingDeadlineSeconds: 3600   # Allow 1 hour late start
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 7200  # 2 hours max
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: analytics-aggregator
              image: registry.company.com/analytics:v2.0.0
              command: ["python3", "aggregate.py", "--date=yesterday"]
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: analytics-secrets
                      key: database-url
                - name: WAREHOUSE_URL
                  valueFrom:
                    secretKeyRef:
                      name: analytics-secrets
                      key: warehouse-url
              resources:
                requests:
                  cpu: "1"
                  memory: "2Gi"
                limits:
                  cpu: "4"
                  memory: "8Gi"
---
# Hourly cache warming
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cache-warmer
  namespace: production
spec:
  schedule: "*/30 * * * *"   # Every 30 minutes
  concurrencyPolicy: Replace  # Replace running if new one triggers
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: cache-warmer
              image: registry.company.com/cache-warmer:v1.0.0
              env:
                - name: REDIS_URL
                  valueFrom:
                    secretKeyRef:
                      name: redis-secret
                      key: url
                - name: API_URL
                  value: "http://product-service:8080"
              resources:
                requests:
                  cpu: "200m"
                  memory: "256Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
---
# Weekly database cleanup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-cleanup
  namespace: production
spec:
  schedule: "0 2 * * 0"    # 02:00 AM every Sunday
  timeZone: "Asia/Bangkok"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: db-cleanup
              image: registry.company.com/db-maintenance:v1.0.0
              command:
                - "/bin/sh"
                - "-c"
                - |
                  psql $DATABASE_URL -c "
                    DELETE FROM sessions WHERE expires_at < NOW() - INTERVAL '7 days';
                    DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '90 days';
                    DELETE FROM temporary_uploads WHERE created_at < NOW() - INTERVAL '24 hours';
                    VACUUM ANALYZE;
                  "
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: database-secret
                      key: url
```

---

## 8. Multi-container Pod Patterns

### 8.1 Circuit Breaker Sidecar Pattern

```typescript
// sidecars/circuit-breaker/src/proxy.ts
import http from 'http';
import httpProxy from 'http-proxy';

interface CircuitState {
  state: 'CLOSED' | 'OPEN' | 'HALF_OPEN';
  failures: number;
  lastFailureTime: number;
  successCount: number;
}

const FAILURE_THRESHOLD = 5;
const SUCCESS_THRESHOLD = 3;
const RESET_TIMEOUT_MS = 30000;  // 30 seconds

class CircuitBreaker {
  private circuits: Map<string, CircuitState> = new Map();

  private getCircuit(service: string): CircuitState {
    if (!this.circuits.has(service)) {
      this.circuits.set(service, {
        state: 'CLOSED',
        failures: 0,
        lastFailureTime: 0,
        successCount: 0,
      });
    }
    return this.circuits.get(service)!;
  }

  canRequest(service: string): boolean {
    const circuit = this.getCircuit(service);

    if (circuit.state === 'CLOSED') return true;

    if (circuit.state === 'OPEN') {
      // Check if we should try half-open
      if (Date.now() - circuit.lastFailureTime > RESET_TIMEOUT_MS) {
        circuit.state = 'HALF_OPEN';
        circuit.successCount = 0;
        return true;
      }
      return false;
    }

    // HALF_OPEN: Allow limited requests
    return true;
  }

  onSuccess(service: string): void {
    const circuit = this.getCircuit(service);

    if (circuit.state === 'HALF_OPEN') {
      circuit.successCount++;
      if (circuit.successCount >= SUCCESS_THRESHOLD) {
        circuit.state = 'CLOSED';
        circuit.failures = 0;
        console.log(`Circuit CLOSED for ${service}`);
      }
    } else if (circuit.state === 'CLOSED') {
      circuit.failures = Math.max(0, circuit.failures - 1);
    }
  }

  onFailure(service: string): void {
    const circuit = this.getCircuit(service);
    circuit.failures++;
    circuit.lastFailureTime = Date.now();

    if (circuit.state === 'HALF_OPEN' || circuit.failures >= FAILURE_THRESHOLD) {
      circuit.state = 'OPEN';
      console.log(`Circuit OPEN for ${service} after ${circuit.failures} failures`);
    }
  }

  getStatus(): Record<string, CircuitState> {
    return Object.fromEntries(this.circuits);
  }
}

// HTTP Proxy with circuit breaker
const proxy = httpProxy.createProxyServer({});
const circuitBreaker = new CircuitBreaker();

const TARGET_SERVICE = process.env.TARGET_SERVICE || 'http://localhost:8080';
const SERVICE_NAME = process.env.SERVICE_NAME || 'main-service';
const PROXY_PORT = parseInt(process.env.PROXY_PORT || '9000');

const server = http.createServer((req, res) => {
  // Health check endpoint for the proxy itself
  if (req.url === '/proxy/health') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', circuits: circuitBreaker.getStatus() }));
    return;
  }

  if (!circuitBreaker.canRequest(SERVICE_NAME)) {
    res.writeHead(503, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({
      error: 'Service temporarily unavailable (circuit open)',
      service: SERVICE_NAME,
    }));
    return;
  }

  const startTime = Date.now();

  proxy.web(req, res, { target: TARGET_SERVICE }, (err) => {
    circuitBreaker.onFailure(SERVICE_NAME);
    console.error(`Proxy error for ${SERVICE_NAME}:`, err.message);
    
    if (!res.headersSent) {
      res.writeHead(502, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ error: 'Bad gateway' }));
    }
  });

  res.on('finish', () => {
    const latency = Date.now() - startTime;
    if (res.statusCode < 500) {
      circuitBreaker.onSuccess(SERVICE_NAME);
    } else {
      circuitBreaker.onFailure(SERVICE_NAME);
    }
  });
});

server.listen(PROXY_PORT, () => {
  console.log(`Circuit breaker proxy listening on port ${PROXY_PORT}`);
  console.log(`Proxying to: ${TARGET_SERVICE}`);
});
```

### 8.2 Pod Disruption Budget

```yaml
# kubernetes/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: product-service-pdb
  namespace: production
spec:
  minAvailable: 2    # At least 2 pods must be available
  selector:
    matchLabels:
      app: product-service
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  maxUnavailable: 1  # At most 1 pod unavailable (quorum safety)
  selector:
    matchLabels:
      app: postgres
---
# Priority Classes
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-services
value: 1000000
globalDefault: false
description: "Critical services that must not be evicted"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "High priority services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-jobs
value: 1000
globalDefault: false
description: "Batch jobs - can be preempted"
```

---

## 9. Resource Quotas และ LimitRanges

```yaml
# kubernetes/resource-management.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    pods: "200"
    services: "50"
    persistentvolumeclaims: "100"
    services.loadbalancers: "10"
    count/deployments.apps: "50"
    count/jobs.batch: "100"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
    # Default limits for containers
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "8"
        memory: "16Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    
    # Pod-level limits
    - type: Pod
      max:
        cpu: "16"
        memory: "32Gi"
    
    # PVC storage limits
    - type: PersistentVolumeClaim
      max:
        storage: "1Ti"
      min:
        storage: "1Gi"
```

---

## สรุป

บทนี้ครอบคลุม Advanced Kubernetes Patterns อย่างละเอียด:

1. **Pod Templates** - Production-grade configuration: security context, affinity, topology spread
2. **Init Containers** - Database migrations, config loading ก่อน main container start
3. **Sidecar Containers** - Ambassador pattern, log collection, circuit breaker proxy
4. **Ephemeral Containers** - Debug production pods โดยไม่ต้อง restart
5. **DaemonSets** - Node monitoring, log collection, security scanning บน every node
6. **StatefulSets** - PostgreSQL cluster ด้วย replication, stable network identities
7. **Jobs/CronJobs** - Indexed jobs, scheduled maintenance, analytics aggregation
8. **Multi-container Patterns** - Circuit breaker sidecar, PDB สำหรับ HA
9. **Resource Management** - Quotas, LimitRanges, Priority Classes

Key Takeaways:
- ใช้ topologySpreadConstraints แทน podAntiAffinity เพื่อ zone distribution ที่ดีกว่า
- Init containers ช่วยแก้ปัญหา startup dependencies โดยไม่ต้อง polling ใน main app
- Sidecar pattern เป็น foundation ของ service mesh และ observability
- Ephemeral containers เป็น production debugging tool ที่ไม่กระทบ uptime
- StatefulSet headless service ให้ stable DNS names สำหรับ database replication
- PDB เป็น required สำหรับ HA - ป้องกัน voluntary disruption จาก cluster upgrades

# Part 97: Open Source Tools Reference

## บทนำ

บทนี้เป็น Reference Guide ที่ครบครันสำหรับ Open Source Tools ที่ใช้ใน Microservices Ecosystem ตั้งแต่ API Gateway, Service Mesh, Observability, CI/CD, Registry ไปจนถึง Developer Tools ทุก Tool มีตัวอย่าง Config และ Code จริงที่นำไปใช้งานได้เลย

---

## 1. Kong API Gateway

### 1.1 Declarative Configuration (DB-less mode)

```yaml
# kong/kong.yml
_format_version: '3.0'
_transform: true

services:
  - name: upload-service
    url: http://upload-service:3001
    connect_timeout: 10000
    read_timeout: 60000
    write_timeout: 60000
    retries: 3
    routes:
      - name: upload-routes
        paths:
          - /api/v1/upload
        methods:
          - POST
          - PUT
          - DELETE
        strip_path: false
        preserve_host: false
    plugins:
      - name: rate-limiting
        config:
          minute: 30
          hour: 500
          day: 2000
          policy: local
          limit_by: consumer
          fault_tolerant: true
          hide_client_headers: false
      - name: jwt
        config:
          key_claim_name: kid
          claims_to_verify:
            - exp
          maximum_expiration: 86400
      - name: request-size-limiting
        config:
          allowed_payload_size: 1024  # 1GB in MB
          size_unit: megabytes
          require_content_length: false

  - name: search-service
    url: http://search-service:3005
    routes:
      - name: search-routes
        paths:
          - /api/v1/search
        methods:
          - GET
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          second: 10
          minute: 200
          policy: redis
          redis_host: redis
          redis_port: 6379
      - name: response-transformer
        config:
          add:
            headers:
              - 'X-Content-Type-Options:nosniff'
              - 'X-Frame-Options:DENY'
      - name: proxy-cache
        config:
          response_code:
            - 200
          request_method:
            - GET
          content_type:
            - application/json
          cache_ttl: 300
          strategy: memory

  - name: video-service
    url: http://hls-service:3003
    routes:
      - name: video-hls-routes
        paths:
          - /api/v1/videos
        methods:
          - GET
          - POST
          - PATCH
          - DELETE
    plugins:
      - name: cors
        config:
          origins:
            - 'https://myplatform.com'
            - 'https://www.myplatform.com'
          methods:
            - GET
            - POST
            - OPTIONS
          headers:
            - Accept
            - Authorization
            - Content-Type
          exposed_headers:
            - X-Auth-Token
          max_age: 3600
          credentials: true

consumers:
  - username: mobile-app
    custom_id: mobile-app-v1
    jwt_secrets:
      - key: mobile-app-key
        algorithm: RS256
        rsa_public_key: |
          -----BEGIN PUBLIC KEY-----
          MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...
          -----END PUBLIC KEY-----

  - username: web-app
    custom_id: web-app-v1

plugins:
  # Global plugins
  - name: prometheus
    config:
      status_code_metrics: true
      latency_metrics: true
      bandwidth_metrics: true
      upstream_health_metrics: true
  
  - name: correlation-id
    config:
      header_name: X-Request-ID
      generator: uuid
      echo_downstream: true

  - name: zipkin
    config:
      http_endpoint: http://jaeger:9411/api/v2/spans
      sample_ratio: 0.01
      include_credential: true

  - name: ip-restriction
    config:
      allow:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
      deny: []
    enabled: false  # Enable for internal-only endpoints
```

### 1.2 TypeScript Kong Admin API Client

```typescript
// tools/kong-admin/src/kong-admin.client.ts
import axios, { AxiosInstance } from 'axios';

interface KongService {
  id?: string;
  name: string;
  url: string;
  connect_timeout?: number;
  read_timeout?: number;
  retries?: number;
}

interface KongRoute {
  id?: string;
  name: string;
  service: { id: string };
  paths: string[];
  methods: string[];
  strip_path?: boolean;
}

interface KongPlugin {
  id?: string;
  name: string;
  service?: { id: string };
  route?: { id: string };
  config: Record<string, any>;
  enabled?: boolean;
}

interface KongConsumer {
  id?: string;
  username: string;
  custom_id?: string;
}

export class KongAdminClient {
  private http: AxiosInstance;

  constructor(adminUrl: string = 'http://kong:8001') {
    this.http = axios.create({
      baseURL: adminUrl,
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
      timeout: 10000,
    });
  }

  // Services CRUD
  async createService(service: KongService): Promise<KongService> {
    const response = await this.http.post('/services', service);
    return response.data;
  }

  async updateService(id: string, updates: Partial<KongService>): Promise<KongService> {
    const response = await this.http.patch(`/services/${id}`, updates);
    return response.data;
  }

  async deleteService(id: string): Promise<void> {
    await this.http.delete(`/services/${id}`);
  }

  async listServices(): Promise<{ data: KongService[]; total: number }> {
    const response = await this.http.get('/services');
    return response.data;
  }

  // Routes CRUD
  async createRoute(route: KongRoute): Promise<KongRoute> {
    const response = await this.http.post('/routes', route);
    return response.data;
  }

  async updateRoute(id: string, updates: Partial<KongRoute>): Promise<KongRoute> {
    const response = await this.http.patch(`/routes/${id}`, updates);
    return response.data;
  }

  // Plugin management
  async addPlugin(plugin: KongPlugin): Promise<KongPlugin> {
    const response = await this.http.post('/plugins', plugin);
    return response.data;
  }

  async updatePlugin(id: string, updates: Partial<KongPlugin>): Promise<KongPlugin> {
    const response = await this.http.patch(`/plugins/${id}`, updates);
    return response.data;
  }

  async enableRateLimiting(
    serviceId: string,
    limits: { second?: number; minute?: number; hour?: number }
  ): Promise<KongPlugin> {
    return this.addPlugin({
      name: 'rate-limiting',
      service: { id: serviceId },
      config: {
        ...limits,
        policy: 'redis',
        redis_host: process.env.REDIS_HOST || 'redis',
        redis_port: 6379,
        fault_tolerant: true,
      },
    });
  }

  async enableCircuitBreaker(serviceId: string): Promise<KongPlugin> {
    return this.addPlugin({
      name: 'circuit-breaker',
      service: { id: serviceId },
      config: {
        look_back_window: 60,
        percentage_threshold: 50,
        minimum_calls: 10,
        open_circuit_after: 5,
        close_circuit_after: 60,
      },
    });
  }

  // Consumers
  async createConsumer(consumer: KongConsumer): Promise<KongConsumer> {
    const response = await this.http.post('/consumers', consumer);
    return response.data;
  }

  async addJWTCredential(
    consumerId: string,
    credentials: { key: string; algorithm: string; secret?: string }
  ): Promise<object> {
    const response = await this.http.post(
      `/consumers/${consumerId}/jwt`,
      credentials
    );
    return response.data;
  }

  // Health check
  async getNodeStatus(): Promise<object> {
    const response = await this.http.get('/status');
    return response.data;
  }

  async getClusterStatus(): Promise<object> {
    const response = await this.http.get('/clustering/data-planes');
    return response.data;
  }

  // Utility: Setup complete service with routing and plugins
  async setupServiceComplete(config: {
    name: string;
    url: string;
    paths: string[];
    methods: string[];
    rateLimits?: { minute?: number; hour?: number };
    enableJWT?: boolean;
    enableCORS?: boolean;
    corsOrigins?: string[];
  }): Promise<{ service: KongService; route: KongRoute; plugins: KongPlugin[] }> {
    // Create service
    const service = await this.createService({
      name: config.name,
      url: config.url,
      connect_timeout: 10000,
      read_timeout: 30000,
      retries: 3,
    });

    // Create route
    const route = await this.createRoute({
      name: `${config.name}-route`,
      service: { id: service.id! },
      paths: config.paths,
      methods: config.methods,
      strip_path: false,
    });

    const plugins: KongPlugin[] = [];

    // Add rate limiting
    if (config.rateLimits) {
      const plugin = await this.addPlugin({
        name: 'rate-limiting',
        service: { id: service.id! },
        config: {
          ...config.rateLimits,
          policy: 'redis',
          redis_host: 'redis',
        },
      });
      plugins.push(plugin);
    }

    // Add JWT authentication
    if (config.enableJWT) {
      const plugin = await this.addPlugin({
        name: 'jwt',
        service: { id: service.id! },
        config: {
          claims_to_verify: ['exp'],
          key_claim_name: 'kid',
        },
      });
      plugins.push(plugin);
    }

    // Add CORS
    if (config.enableCORS) {
      const plugin = await this.addPlugin({
        name: 'cors',
        service: { id: service.id! },
        config: {
          origins: config.corsOrigins || ['*'],
          methods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
          headers: ['Accept', 'Authorization', 'Content-Type'],
          max_age: 3600,
          credentials: true,
        },
      });
      plugins.push(plugin);
    }

    return { service, route, plugins };
  }
}
```

---

## 2. Traefik

### 2.1 Configuration YAML + Middleware

```yaml
# traefik/traefik.yml
global:
  checkNewVersion: true
  sendAnonymousUsage: false

api:
  dashboard: true
  insecure: false

entryPoints:
  web:
    address: ':80'
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ':443'
    http:
      tls:
        certResolver: letsencrypt
  metrics:
    address: ':8082'

certificatesResolvers:
  letsencrypt:
    acme:
      email: ops@myplatform.com
      storage: /letsencrypt/acme.json
      httpChallenge:
        entryPoint: web

providers:
  docker:
    endpoint: unix:///var/run/docker.sock
    exposedByDefault: false
    network: traefik-network
  
  file:
    directory: /etc/traefik/dynamic
    watch: true

metrics:
  prometheus:
    entryPoint: metrics
    addEntryPointsLabels: true
    addServicesLabels: true
    buckets:
      - 0.1
      - 0.3
      - 1.2
      - 5.0

log:
  level: INFO
  format: json

accessLog:
  format: json
  fields:
    defaultMode: keep
    headers:
      defaultMode: drop
      names:
        Authorization: redact
        X-Request-ID: keep

---
# traefik/dynamic/middlewares.yml
http:
  middlewares:
    # Rate limiting
    api-rate-limit:
      rateLimit:
        burst: 50
        average: 100
        period: 1m
        sourceCriterion:
          ipStrategy:
            depth: 1
    
    # Auth middleware
    jwt-auth:
      forwardAuth:
        address: http://auth-service:3000/validate
        authResponseHeaders:
          - X-User-Id
          - X-User-Role
          - X-Tenant-Id
        trustForwardHeader: true
    
    # Security headers
    security-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        customFrameOptionsValue: DENY
        referrerPolicy: strict-origin-when-cross-origin
        permissionsPolicy: camera=(), microphone=(), geolocation=()
        customResponseHeaders:
          X-Content-Type-Options: nosniff
          X-Permitted-Cross-Domain-Policies: none
    
    # Circuit breaker
    circuit-breaker:
      circuitBreaker:
        expression: "ResponseCodeRatio(500, 600, 0, 600) > 0.30"
        checkPeriod: 10s
        fallbackDuration: 30s
        recoveryDuration: 60s
    
    # Retry
    retry-middleware:
      retry:
        attempts: 3
        initialInterval: 100ms
    
    # Compression
    compress:
      compress:
        excludedContentTypes:
          - text/event-stream
    
    # IP allowlist
    internal-only:
      ipAllowList:
        sourceRange:
          - 10.0.0.0/8
          - 172.16.0.0/12
          - 192.168.0.0/16

  routers:
    upload-service:
      rule: 'Host(`api.myplatform.com`) && PathPrefix(`/api/v1/upload`)'
      entryPoints:
        - websecure
      service: upload-service
      middlewares:
        - jwt-auth
        - api-rate-limit
        - security-headers
        - compress
      tls:
        certResolver: letsencrypt
    
    search-service:
      rule: 'Host(`api.myplatform.com`) && PathPrefix(`/api/v1/search`)'
      entryPoints:
        - websecure
      service: search-service
      middlewares:
        - api-rate-limit
        - security-headers
        - compress
      tls:
        certResolver: letsencrypt

  services:
    upload-service:
      loadBalancer:
        servers:
          - url: http://upload-service:3001
        healthCheck:
          path: /health
          interval: 10s
          timeout: 5s
          scheme: http
        sticky:
          cookie:
            name: upload-session
            secure: true
    
    search-service:
      loadBalancer:
        servers:
          - url: http://search-service:3005
        healthCheck:
          path: /health
          interval: 10s
          timeout: 3s
        passHostHeader: true
```

---

## 3. Istio Service Mesh

### 3.1 Key Resource Examples

```yaml
# istio/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: video-service
  namespace: video-platform
spec:
  hosts:
    - video-service
  http:
    # Canary deployment: 10% to v2
    - name: canary-v2
      match:
        - headers:
            x-canary:
              exact: 'true'
      route:
        - destination:
            host: video-service
            subset: v2
          weight: 100
    
    # A/B testing by user segment
    - name: ab-test
      match:
        - headers:
            x-user-segment:
              exact: 'beta'
      route:
        - destination:
            host: video-service
            subset: v2
    
    # Default traffic split
    - name: primary
      route:
        - destination:
            host: video-service
            subset: v1
          weight: 90
        - destination:
            host: video-service
            subset: v2
          weight: 10
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: 'gateway-error,connect-failure,retriable-4xx'
      fault:
        delay:
          percentage:
            value: 0.1
          fixedDelay: 100ms

---
# istio/destination-rule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: video-service
  namespace: video-platform
spec:
  host: video-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
        connectTimeout: 10s
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
        maxRetries: 3
        idleTimeout: 90s
        h2UpgradePolicy: DEFAULT
    loadBalancer:
      simple: LEAST_CONN
    outlierDetection:
      consecutiveGatewayErrors: 5
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
    tls:
      mode: ISTIO_MUTUAL
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          http:
            http1MaxPendingRequests: 200
    - name: v2
      labels:
        version: v2

---
# istio/authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: video-service-authz
  namespace: video-platform
spec:
  selector:
    matchLabels:
      app: video-service
  rules:
    # Allow authenticated users to read videos
    - from:
        - source:
            principals:
              - cluster.local/ns/video-platform/sa/web-app
              - cluster.local/ns/video-platform/sa/mobile-app
      to:
        - operation:
            methods:
              - GET
            paths:
              - /api/v1/videos*
    
    # Allow upload service to trigger transcoding
    - from:
        - source:
            principals:
              - cluster.local/ns/video-platform/sa/upload-service
      to:
        - operation:
            methods:
              - POST
            paths:
              - /internal/transcode*
    
    # Block everything else
    - {}

---
# istio/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: video-platform
spec:
  mtls:
    mode: STRICT

---
# istio/request-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: video-platform
spec:
  selector:
    matchLabels:
      app: video-service
  jwtRules:
    - issuer: https://auth.myplatform.com
      jwksUri: https://auth.myplatform.com/.well-known/jwks.json
      audiences:
        - myplatform-api
      forwardOriginalToken: true

---
# istio/service-entry.yaml (External service access)
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: elasticsearch-external
  namespace: video-platform
spec:
  hosts:
    - my-elasticsearch.us-east-1.aws.elastic.io
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
  exportTo:
    - 'search-service'
```

---

## 4. Prometheus Monitoring

### 4.1 Recording Rules + Alerting Rules

```yaml
# prometheus/rules/recording-rules.yml
groups:
  - name: microservices.recording
    interval: 30s
    rules:
      # Request rate per service
      - record: job:http_requests:rate5m
        expr: |
          sum by (job, method, status_code) (
            rate(http_requests_total[5m])
          )
      
      # Error rate per service
      - record: job:http_error_rate:rate5m
        expr: |
          sum by (job) (
            rate(http_requests_total{status_code=~"5.."}[5m])
          ) /
          sum by (job) (
            rate(http_requests_total[5m])
          )
      
      # P95 latency per service
      - record: job:http_request_duration_p95:rate5m
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          )
      
      # P99 latency
      - record: job:http_request_duration_p99:rate5m
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          )
      
      # Active connections per service
      - record: job:active_connections:avg
        expr: |
          avg by (job) (nodejs_active_handles_total)

---
# prometheus/rules/alerting-rules.yml
groups:
  - name: microservices.alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: job:http_error_rate:rate5m > 0.05
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: 'High error rate on {{ $labels.job }}'
          description: >
            Service {{ $labels.job }} has error rate {{ $value | humanizePercentage }}
            over the last 5 minutes (threshold: 5%)
          runbook: 'https://wiki.myplatform.com/runbooks/high-error-rate'
          dashboard: 'https://grafana.myplatform.com/d/microservices/{{ $labels.job }}'
      
      # High latency
      - alert: HighLatencyP99
        expr: job:http_request_duration_p99:rate5m > 2.0
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: 'High P99 latency on {{ $labels.job }}'
          description: >
            P99 latency for {{ $labels.job }} is {{ $value | humanizeDuration }}
            (threshold: 2s)
      
      # Service down
      - alert: ServiceDown
        expr: up{job=~".*-service"} == 0
        for: 1m
        labels:
          severity: critical
          team: platform
          pagerduty: 'true'
        annotations:
          summary: 'Service {{ $labels.job }} is down'
          description: 'Service {{ $labels.job }} on {{ $labels.instance }} has been down for 1 minute'
      
      # High memory usage
      - alert: HighMemoryUsage
        expr: |
          (nodejs_heap_used_bytes / nodejs_heap_size_total_bytes) > 0.90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: 'High memory usage on {{ $labels.job }}'
          description: >
            Node.js heap usage is {{ $value | humanizePercentage }} on {{ $labels.job }}
      
      # Kafka consumer lag
      - alert: KafkaConsumerLag
        expr: kafka_consumergroup_lag > 10000
        for: 5m
        labels:
          severity: warning
          team: data
        annotations:
          summary: 'Kafka consumer lag too high'
          description: >
            Consumer group {{ $labels.consumergroup }} has lag of
            {{ $value }} messages on topic {{ $labels.topic }}
      
      # Redis connection failure
      - alert: RedisConnectionFailure
        expr: redis_connected_clients == 0
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: 'Redis has no connected clients'
          description: 'Redis instance {{ $labels.addr }} has lost all connections'
      
      # Database connection pool exhaustion
      - alert: DBConnectionPoolExhausted
        expr: |
          pg_stat_activity_count{state="active"} / pg_settings_max_connections > 0.80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: 'PostgreSQL connection pool near capacity'
          description: >
            Database {{ $labels.datname }} is using
            {{ $value | humanizePercentage }} of max connections
      
      # Disk space
      - alert: LowDiskSpace
        expr: |
          (node_filesystem_avail_bytes{fstype!="tmpfs"} /
           node_filesystem_size_bytes{fstype!="tmpfs"}) < 0.10
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: 'Low disk space on {{ $labels.instance }}'
          description: >
            Filesystem {{ $labels.mountpoint }} on {{ $labels.instance }}
            has only {{ $value | humanizePercentage }} space remaining
```

---

## 5. Grafana: Datasource + Dashboard Provisioning

```yaml
# grafana/provisioning/datasources/datasources.yaml
apiVersion: 1

deleteDatasources:
  - name: Old-Prometheus
    orgId: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      httpMethod: POST
      timeInterval: 15s
      exemplarTraceIdDestinations:
        - name: TraceID
          datasourceUid: jaeger
  
  - name: Jaeger
    type: jaeger
    uid: jaeger
    access: proxy
    url: http://jaeger:16686
    editable: false
    jsonData:
      tracesToLogs:
        datasourceUid: loki
        tags: ['job', 'instance']
  
  - name: Loki
    type: loki
    uid: loki
    access: proxy
    url: http://loki:3100
    editable: false
    jsonData:
      derivedFields:
        - datasourceUid: jaeger
          matcherRegex: 'traceID=(\w+)'
          name: TraceID
          url: '$${__value.raw}'
  
  - name: PostgreSQL
    type: postgres
    url: postgres:5432
    user: grafana_reader
    secureJsonData:
      password: ${GRAFANA_DB_PASSWORD}
    jsonData:
      database: streaming
      sslmode: require
      maxOpenConns: 5
      connMaxLifetime: 14400

---
# grafana/provisioning/dashboards/dashboards.yaml
apiVersion: 1

providers:
  - name: Microservices
    orgId: 1
    folder: Microservices Platform
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /etc/grafana/dashboards/microservices
  
  - name: Infrastructure
    orgId: 1
    folder: Infrastructure
    type: file
    options:
      path: /etc/grafana/dashboards/infrastructure

---
# grafana/dashboards/microservices/overview.json (partial)
# In practice this is a large JSON, showing key panels structure:
```

```typescript
// tools/grafana/src/create-dashboard.ts
import axios from 'axios';

interface GrafanaPanel {
  title: string;
  type: 'graph' | 'stat' | 'table' | 'heatmap' | 'timeseries';
  gridPos: { h: number; w: number; x: number; y: number };
  targets: Array<{
    expr: string;
    legendFormat: string;
    refId: string;
  }>;
  fieldConfig?: {
    defaults: Record<string, any>;
    overrides: any[];
  };
}

interface GrafanaDashboard {
  title: string;
  uid?: string;
  tags: string[];
  timezone: string;
  schemaVersion: number;
  panels: GrafanaPanel[];
  time: { from: string; to: string };
  refresh: string;
  templating: { list: any[] };
}

export class GrafanaDashboardCreator {
  private http = axios.create({
    baseURL: process.env.GRAFANA_URL || 'http://grafana:3000',
    auth: {
      username: process.env.GRAFANA_USER || 'admin',
      password: process.env.GRAFANA_PASSWORD || 'admin',
    },
  });

  async createMicroservicesDashboard(serviceName: string): Promise<string> {
    const dashboard: GrafanaDashboard = {
      title: `${serviceName} - Service Dashboard`,
      uid: `svc-${serviceName.toLowerCase()}`,
      tags: ['microservices', serviceName, 'auto-generated'],
      timezone: 'browser',
      schemaVersion: 38,
      time: { from: 'now-3h', to: 'now' },
      refresh: '30s',
      templating: {
        list: [
          {
            name: 'instance',
            type: 'query',
            query: `label_values(up{job="${serviceName}"}, instance)`,
            refresh: 2,
          },
        ],
      },
      panels: [
        {
          title: 'Request Rate',
          type: 'timeseries',
          gridPos: { h: 8, w: 12, x: 0, y: 0 },
          targets: [
            {
              expr: `sum(rate(http_requests_total{job="${serviceName}"}[5m])) by (method, status_code)`,
              legendFormat: '{{method}} {{status_code}}',
              refId: 'A',
            },
          ],
          fieldConfig: {
            defaults: {
              unit: 'reqps',
              color: { mode: 'palette-classic' },
            },
            overrides: [],
          },
        },
        {
          title: 'Error Rate',
          type: 'stat',
          gridPos: { h: 4, w: 6, x: 12, y: 0 },
          targets: [
            {
              expr: `sum(rate(http_requests_total{job="${serviceName}",status_code=~"5.."}[5m])) / sum(rate(http_requests_total{job="${serviceName}"}[5m]))`,
              legendFormat: 'Error Rate',
              refId: 'A',
            },
          ],
          fieldConfig: {
            defaults: {
              unit: 'percentunit',
              thresholds: {
                steps: [
                  { value: 0, color: 'green' },
                  { value: 0.01, color: 'yellow' },
                  { value: 0.05, color: 'red' },
                ],
              },
            },
            overrides: [],
          },
        },
        {
          title: 'P99 Latency',
          type: 'timeseries',
          gridPos: { h: 8, w: 12, x: 0, y: 8 },
          targets: [
            {
              expr: `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job="${serviceName}"}[5m])) by (le))`,
              legendFormat: 'P99',
              refId: 'A',
            },
            {
              expr: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job="${serviceName}"}[5m])) by (le))`,
              legendFormat: 'P95',
              refId: 'B',
            },
          ],
          fieldConfig: {
            defaults: { unit: 's' },
            overrides: [],
          },
        },
      ],
    };

    const response = await this.http.post('/api/dashboards/db', {
      dashboard,
      overwrite: true,
      folderId: 0,
    });

    return response.data.url;
  }
}
```

---

## 6. ArgoCD GitOps

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: video-platform-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: platform-alerts
    notifications.argoproj.io/subscribe.on-deployed.slack: platform-deploys
spec:
  project: video-platform
  source:
    repoURL: https://github.com/myorg/microservices-platform
    targetRevision: main
    path: apps/video-platform/overlays/production
    kustomize:
      images:
        - upload-service=123456789.dkr.ecr.ap-southeast-1.amazonaws.com/upload-service:1.2.3
        - search-service=123456789.dkr.ecr.ap-southeast-1.amazonaws.com/search-service:1.1.0
  destination:
    server: https://kubernetes.default.svc
    namespace: video-platform
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

---
# argocd/appproject.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: video-platform
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: Video Platform microservices
  sourceRepos:
    - 'https://github.com/myorg/microservices-platform'
    - 'https://charts.helm.sh/stable'
  destinations:
    - namespace: video-platform
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
    - group: rbac.authorization.k8s.io
      kind: ClusterRole
  namespaceResourceBlacklist:
    - group: ''
      kind: ResourceQuota
  roles:
    - name: developers
      description: Developer read-only access
      policies:
        - 'p, proj:video-platform:developers, applications, get, video-platform/*, allow'
        - 'p, proj:video-platform:developers, applications, sync, video-platform/*, deny'
      groups:
        - myorg:developers
    - name: devops
      description: DevOps full access
      policies:
        - 'p, proj:video-platform:devops, applications, *, video-platform/*, allow'
      groups:
        - myorg:devops

---
# argocd/applicationset.yaml (Multi-cluster deployment)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: video-platform-all-clusters
  namespace: argocd
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            environment: production
  template:
    metadata:
      name: 'video-platform-{{name}}'
      namespace: argocd
    spec:
      project: video-platform
      source:
        repoURL: https://github.com/myorg/microservices-platform
        targetRevision: main
        path: 'apps/video-platform/overlays/{{metadata.annotations.region}}'
      destination:
        server: '{{server}}'
        namespace: video-platform
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## 7. Harbor Container Registry

### 7.1 Helm Values + Robot Account TypeScript

```yaml
# harbor/values.yaml (Helm)
expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: harbor-tls
  ingress:
    hosts:
      core: registry.myplatform.com
      notary: notary.myplatform.com
    className: nginx
    annotations:
      nginx.ingress.kubernetes.io/ssl-redirect: 'true'
      nginx.ingress.kubernetes.io/proxy-body-size: '0'
      nginx.ingress.kubernetes.io/proxy-read-timeout: '600'

externalURL: https://registry.myplatform.com

persistence:
  enabled: true
  resourcePolicy: keep
  persistentVolumeClaim:
    registry:
      storageClass: gp3
      accessMode: ReadWriteOnce
      size: 500Gi
    chartmuseum:
      storageClass: gp3
      size: 10Gi
    database:
      storageClass: gp3
      size: 50Gi
    redis:
      storageClass: gp3
      size: 10Gi
    trivy:
      storageClass: gp3
      size: 20Gi

database:
  type: external
  external:
    host: my-rds.cluster.ap-southeast-1.rds.amazonaws.com
    port: 5432
    username: harbor
    password: ${HARBOR_DB_PASSWORD}
    coreDatabase: registry

redis:
  type: external
  external:
    addr: my-redis.cache.amazonaws.com:6379
    password: ''

registry:
  replicas: 2
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 1Gi
      cpu: 500m

core:
  replicas: 2
  resources:
    requests:
      memory: 256Mi
      cpu: 100m

trivy:
  enabled: true
  replicas: 1
  resources:
    requests:
      memory: 512Mi
      cpu: 200m
    limits:
      memory: 1Gi
      cpu: 500m

notary:
  enabled: false  # Enable for content trust

jobservice:
  replicas: 1
  jobLoggers:
    - database
    - stdout

metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    namespace: monitoring
```

```typescript
// tools/harbor/src/harbor.client.ts
import axios, { AxiosInstance } from 'axios';

interface RobotAccount {
  id?: number;
  name: string;
  description?: string;
  duration: number; // days, -1 = no expiry
  permissions: Array<{
    kind: 'project' | 'system';
    namespace: string;
    access: Array<{ resource: string; action: string }>;
  }>;
}

interface RobotAccountSecret {
  id: number;
  name: string;
  secret: string;
  createdAt: string;
  expiresAt: string;
}

interface Project {
  id?: number;
  name: string;
  public: boolean;
  registryId?: number;
  scanOnPush?: boolean;
}

interface VulnerabilityReport {
  severity: 'critical' | 'high' | 'medium' | 'low' | 'none';
  total: number;
  fixable: number;
  summary: Record<string, number>;
}

export class HarborClient {
  private http: AxiosInstance;

  constructor(
    harborUrl: string = 'https://registry.myplatform.com',
    username: string = 'admin',
    password: string
  ) {
    this.http = axios.create({
      baseURL: `${harborUrl}/api/v2.0`,
      auth: { username, password },
      headers: { 'Content-Type': 'application/json' },
    });
  }

  // Project management
  async createProject(project: Project): Promise<void> {
    await this.http.post('/projects', {
      project_name: project.name,
      public: project.public,
      registry_id: project.registryId,
      metadata: {
        auto_scan: project.scanOnPush ? 'true' : 'false',
        prevent_vul: 'true',
        severity: 'high', // Block images with HIGH+ vulnerabilities
        reuse_sys_cve_allowlist: 'true',
      },
    });
  }

  async setProjectQuota(projectId: number, storageGB: number): Promise<void> {
    await this.http.put(`/quotas/${projectId}`, {
      hard: { storage: storageGB * 1024 * 1024 * 1024 },
    });
  }

  // Robot accounts for CI/CD
  async createRobotAccount(robot: RobotAccount): Promise<RobotAccountSecret> {
    const response = await this.http.post('/robots', {
      name: robot.name,
      description: robot.description || '',
      duration: robot.duration,
      level: 'project',
      permissions: robot.permissions.map(p => ({
        kind: p.kind,
        namespace: p.namespace,
        access: p.access,
      })),
    });

    return {
      id: response.data.id,
      name: response.data.name,
      secret: response.data.secret,
      createdAt: response.data.creation_time,
      expiresAt: response.data.expires_at,
    };
  }

  async createCICDRobotAccount(
    projectName: string
  ): Promise<RobotAccountSecret> {
    return this.createRobotAccount({
      name: `ci-${projectName}`,
      description: `CI/CD robot account for ${projectName}`,
      duration: 365, // 1 year
      permissions: [
        {
          kind: 'project',
          namespace: projectName,
          access: [
            { resource: 'repository', action: 'list' },
            { resource: 'repository', action: 'pull' },
            { resource: 'repository', action: 'push' },
            { resource: 'artifact', action: 'read' },
            { resource: 'artifact', action: 'create' },
            { resource: 'artifact', action: 'delete' },
            { resource: 'scan', action: 'create' },
          ],
        },
      ],
    });
  }

  async rotateRobotSecret(robotId: number): Promise<string> {
    const response = await this.http.patch(`/robots/${robotId}`, {
      secret: this.generateSecret(),
    });
    return response.data.secret;
  }

  private generateSecret(): string {
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*';
    return Array.from({ length: 32 }, () =>
      chars[Math.floor(Math.random() * chars.length)]
    ).join('');
  }

  // Vulnerability scanning
  async scanImage(projectName: string, repository: string, tag: string): Promise<void> {
    await this.http.post(
      `/projects/${projectName}/repositories/${encodeURIComponent(repository)}/artifacts/${tag}/scan`
    );
  }

  async getVulnerabilities(
    projectName: string,
    repository: string,
    tag: string
  ): Promise<VulnerabilityReport> {
    const response = await this.http.get(
      `/projects/${projectName}/repositories/${encodeURIComponent(repository)}/artifacts/${tag}/additions/vulnerabilities`
    );

    const report = response.data['application/vnd.security.vulnerability.report; version=1.1'];
    if (!report) {
      return { severity: 'none', total: 0, fixable: 0, summary: {} };
    }

    const summary = report.summary || {};
    const total = Object.values(summary).reduce((a: any, b: any) => a + b, 0) as number;

    return {
      severity: this.getHighestSeverity(summary),
      total,
      fixable: summary.fixable || 0,
      summary,
    };
  }

  private getHighestSeverity(summary: Record<string, number>): VulnerabilityReport['severity'] {
    if (summary.Critical > 0) return 'critical';
    if (summary.High > 0) return 'high';
    if (summary.Medium > 0) return 'medium';
    if (summary.Low > 0) return 'low';
    return 'none';
  }

  // Garbage collection
  async triggerGarbageCollection(): Promise<number> {
    const response = await this.http.post('/system/gc', {
      schedule: { type: 'Manual' },
    });
    return response.data.id;
  }

  async getGCStatus(jobId: number): Promise<string> {
    const response = await this.http.get(`/system/gc/${jobId}`);
    return response.data.job_status;
  }
}
```

---

## 8. Tilt: Local Development

```python
# Tiltfile
# -*- mode: Python -*-

# Load extensions
load('ext://restart_process', 'docker_build_with_restart')
load('ext://namespace', 'namespace_create', 'namespace_inject')
load('ext://dotenv', 'dotenv')

# Load env vars
dotenv('.env.local')

# Create namespace
namespace_create('video-platform')

# Services to develop
SERVICES = {
    'upload-service': {'port': 3001, 'path': './services/upload'},
    'search-service': {'port': 3005, 'path': './services/search'},
    'hls-service': {'port': 3003, 'path': './services/hls'},
    'watch-history-service': {'port': 3004, 'path': './services/watch-history'},
    'analytics-service': {'port': 3007, 'path': './services/analytics'},
    'recommendation-service': {'port': 3006, 'path': './services/recommendation'},
}

# Build each service with live reload
for name, config in SERVICES.items():
    docker_build_with_restart(
        'registry.myplatform.com/video-platform/' + name,
        config['path'],
        dockerfile=config['path'] + '/Dockerfile.dev',
        live_update=[
            sync(config['path'] + '/src', '/app/src'),
            run('cd /app && npm run build:watch', trigger=['./src']),
        ],
        entrypoint=['node', '--watch', 'dist/index.js'],
    )

# Apply Kubernetes configs
k8s_yaml(kustomize('./k8s/overlays/local'))

# Infrastructure
docker_compose('./docker-compose.infra.yml')

# Port forwards for development
for name, config in SERVICES.items():
    k8s_resource(
        name,
        port_forwards=[str(config['port']) + ':' + str(config['port'])],
        resource_deps=['postgres', 'redis', 'kafka'],
        labels=[name.split('-')[0]],
    )

# Infrastructure resources
k8s_resource('postgres', port_forwards=['5432:5432'])
k8s_resource('redis', port_forwards=['6379:6379'])
k8s_resource('kafka', port_forwards=['29092:9092'])
k8s_resource('elasticsearch', port_forwards=['9200:9200'])

# Custom commands
local_resource(
    'run-migrations',
    serve_cmd='npm run db:migrate',
    dir='./services/shared',
    resource_deps=['postgres'],
    labels=['database'],
)

local_resource(
    'seed-data',
    cmd='npm run db:seed',
    dir='./services/shared',
    resource_deps=['run-migrations'],
    labels=['database'],
)

# Run tests on file change
local_resource(
    'run-tests',
    serve_cmd='npm run test:watch',
    dir='./services/upload',
    labels=['testing'],
)
```

---

## 9. Skaffold: Build + Deploy Pipeline

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta5
kind: Config
metadata:
  name: video-platform

build:
  local:
    push: false
    useBuildkit: true
    concurrency: 4
  artifacts:
    - image: upload-service
      context: services/upload
      docker:
        dockerfile: Dockerfile
        buildArgs:
          NODE_ENV: production
      sync:
        manual:
          - src: 'src/**/*.ts'
            dest: /app/src
    
    - image: search-service
      context: services/search
      docker:
        dockerfile: Dockerfile
    
    - image: hls-service
      context: services/hls
      docker:
        dockerfile: Dockerfile

deploy:
  kubectl:
    manifests:
      - k8s/base/*.yaml
  statusCheckDeadlineSeconds: 300

profiles:
  - name: local
    build:
      local:
        push: false
    deploy:
      kubectl:
        manifests:
          - k8s/overlays/local/*.yaml
    portForward:
      - resourceType: service
        resourceName: upload-service
        port: 3001
      - resourceType: service
        resourceName: search-service
        port: 3005
  
  - name: staging
    build:
      googleCloudBuild:
        projectId: myproject-staging
        diskSizeGb: 200
        machineType: N1_HIGHCPU_8
    deploy:
      helm:
        releases:
          - name: video-platform
            chartPath: charts/video-platform
            valuesFiles:
              - charts/video-platform/values.staging.yaml
            setValues:
              image.tag: ${SKAFFOLD_IMAGE_TAG}
  
  - name: production
    build:
      googleCloudBuild:
        projectId: myproject-prod
    deploy:
      helm:
        releases:
          - name: video-platform
            chartPath: charts/video-platform
            valuesFiles:
              - charts/video-platform/values.production.yaml

test:
  - image: upload-service
    structureTests:
      - './tests/structure/upload-service.yaml'
    custom:
      - command: 'npm run test:unit'
        timeoutSeconds: 120
        dependencies:
          paths:
            - 'services/upload/src/**/*.ts'
            - 'services/upload/tests/**/*.ts'
```

---

## 10. Telepresence: Local-to-Cluster Development

```bash
# Telepresence คือ Tool ที่ช่วยให้ Local development service
# สามารถ intercept traffic จาก Kubernetes cluster ได้

# Install telepresence
brew install datawire/blackbird/telepresence

# Connect to cluster
telepresence connect --namespace video-platform

# List all services that can be intercepted
telepresence list

# Intercept upload-service (route traffic to local port 3001)
telepresence intercept upload-service \
  --port 3001:3001 \
  --env-file .env.intercepted \
  --mount /tmp/upload-service-mounts

# Intercept with header matching (only intercept specific requests)
telepresence intercept upload-service \
  --port 3001:3001 \
  --http-header X-Dev-Mode=true \
  --env-file .env.intercepted

# Leave intercept (restore original service)
telepresence leave upload-service

# Disconnect from cluster  
telepresence quit
```

```typescript
// tools/telepresence/src/dev-intercept.ts
// Helper to set up Telepresence intercept programmatically
import { execSync, spawn } from 'child_process';
import { writeFileSync, readFileSync, existsSync } from 'fs';

interface InterceptConfig {
  serviceName: string;
  localPort: number;
  remotePort: number;
  namespace?: string;
  headerMatch?: string;
}

export class TelepresenceHelper {
  async connect(namespace: string = 'video-platform'): Promise<void> {
    console.log(`Connecting to cluster namespace: ${namespace}`);
    execSync(`telepresence connect --namespace ${namespace}`, { stdio: 'inherit' });
  }

  async intercept(config: InterceptConfig): Promise<void> {
    const {
      serviceName,
      localPort,
      remotePort,
      namespace = 'video-platform',
      headerMatch,
    } = config;

    const envFile = `.env.${serviceName}.intercepted`;

    let cmd = `telepresence intercept ${serviceName} `;
    cmd += `--port ${localPort}:${remotePort} `;
    cmd += `--env-file ${envFile} `;
    cmd += `--namespace ${namespace} `;

    if (headerMatch) {
      cmd += `--http-header "${headerMatch}" `;
    }

    console.log(`Starting intercept for ${serviceName}...`);
    console.log(`Command: ${cmd}`);

    execSync(cmd, { stdio: 'inherit' });

    // Load environment variables from cluster
    if (existsSync(envFile)) {
      const envContent = readFileSync(envFile, 'utf-8');
      console.log('\nEnvironment variables from cluster:');
      envContent.split('\n').forEach(line => {
        const [key] = line.split('=');
        if (key) console.log(`  ${key}`);
      });
      console.log(`\nRun: export $(cat ${envFile} | xargs) to load these vars`);
    }
  }

  async leave(serviceName: string): Promise<void> {
    execSync(`telepresence leave ${serviceName}`, { stdio: 'inherit' });
  }

  async listInterceptable(): Promise<string[]> {
    const output = execSync('telepresence list').toString();
    return output
      .split('\n')
      .filter(line => line.includes(':'))
      .map(line => line.split(':')[0].trim());
  }

  async disconnect(): Promise<void> {
    execSync('telepresence quit', { stdio: 'inherit' });
  }

  async status(): Promise<void> {
    execSync('telepresence status', { stdio: 'inherit' });
  }
}

// CLI usage
if (require.main === module) {
  const helper = new TelepresenceHelper();
  const [,, command, ...args] = process.argv;

  switch (command) {
    case 'connect':
      helper.connect(args[0]).catch(console.error);
      break;
    case 'intercept':
      helper.intercept({
        serviceName: args[0],
        localPort: parseInt(args[1] || '3000'),
        remotePort: parseInt(args[2] || '3000'),
        headerMatch: args[3],
      }).catch(console.error);
      break;
    case 'leave':
      helper.leave(args[0]).catch(console.error);
      break;
    case 'list':
      helper.listInterceptable().then(services => {
        console.log('Interceptable services:', services);
      }).catch(console.error);
      break;
    case 'disconnect':
      helper.disconnect().catch(console.error);
      break;
    default:
      console.log('Usage: node dev-intercept.js [connect|intercept|leave|list|disconnect] [args...]');
  }
}
```

---

## สรุป

| Tool | Category | Primary Use Case | Learning Curve |
|------|----------|-----------------|----------------|
| Kong | API Gateway | Rate limiting, Auth, Routing | ปานกลาง |
| Traefik | Reverse Proxy / Gateway | Auto TLS, Docker integration | ต่ำ |
| Istio | Service Mesh | mTLS, Traffic management, Observability | สูง |
| Prometheus | Monitoring | Metrics collection, Alerting | ปานกลาง |
| Grafana | Visualization | Dashboards, Alerting UI | ต่ำ |
| ArgoCD | GitOps | Automated deployment, Drift detection | ปานกลาง |
| Harbor | Registry | Image storage, Vulnerability scan | ต่ำ |
| Tilt | Dev Experience | Local K8s dev with live reload | ต่ำ |
| Skaffold | Build/Deploy | Multi-profile build & deploy | ปานกลาง |
| Telepresence | Dev Experience | Intercept K8s traffic locally | ปานกลาง |

> "The right tool for the right job — don't let tooling complexity outpace your team's ability to maintain it"

---

*ถัดไป: Part 98 - Cost Optimization Strategies*

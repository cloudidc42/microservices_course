# Part 33: Multi-Region Deployment & Global Scale

## บทนำ

เมื่อแอปพลิเคชันเติบโตจนต้องรองรับผู้ใช้ทั่วโลก เราต้องออกแบบ Multi-Region architecture ที่รองรับ High Availability, Data Residency requirements, และ Global Load Balancing ในบทนี้เราจะเรียนรู้การ Deploy Microservices ในหลาย Region อย่างมืออาชีพ

## 1. Multi-Region Architecture Overview

```
                     ┌─────────────────────────────┐
                     │     Global Load Balancer     │
                     │    (Cloudflare / AWS GA)     │
                     └──────────┬──────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                  │
    ┌─────────▼──────┐  ┌──────▼───────┐  ┌──────▼──────────┐
    │  ap-southeast-1│  │ us-east-1    │  │  eu-west-1      │
    │  (Singapore)   │  │ (Virginia)   │  │  (Ireland)      │
    │                │  │              │  │                  │
    │ ┌────────────┐ │  │ ┌──────────┐ │  │ ┌────────────┐  │
    │ │ K8s Cluster│ │  │ │K8s Cluster│ │  │ │K8s Cluster │  │
    │ │ (Primary)  │ │  │ │(Read-only)│ │  │ │(Read-only) │  │
    │ └────────────┘ │  │ └──────────┘ │  │ └────────────┘  │
    │                │  │              │  │                  │
    │ ┌────────────┐ │  │ ┌──────────┐ │  │ ┌────────────┐  │
    │ │ PostgreSQL │◄├──┼►│ Replica  │◄├──┼►│ Replica    │  │
    │ │ (Primary)  │ │  │ │          │ │  │ │            │  │
    │ └────────────┘ │  │ └──────────┘ │  │ └────────────┘  │
    └────────────────┘  └──────────────┘  └────────────────┘
```

## 2. Global DNS & Traffic Routing

### 2.1 Cloudflare Workers for Global Routing

```javascript
// cloudflare-worker/routing.js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

const REGIONS = {
  'AS': 'https://ap.api.example.com',   // Asia
  'NA': 'https://us.api.example.com',   // North America
  'EU': 'https://eu.api.example.com',   // Europe
  'DEFAULT': 'https://ap.api.example.com',
};

async function handleRequest(request) {
  const url = new URL(request.url);
  const country = request.cf.country;
  const region = request.cf.continent;
  
  // Route based on continent
  const targetOrigin = REGIONS[region] || REGIONS.DEFAULT;
  
  // Special data residency rules
  // EU users MUST be served from EU (GDPR)
  const forcedRegion = getDataResidencyRegion(country);
  const origin = forcedRegion || targetOrigin;
  
  // Clone request to new origin
  const newUrl = new URL(url.pathname + url.search, origin);
  const newRequest = new Request(newUrl.toString(), {
    method: request.method,
    headers: request.headers,
    body: request.method !== 'GET' && request.method !== 'HEAD' 
      ? request.body 
      : undefined,
    redirect: 'follow',
  });
  
  // Add routing metadata headers
  newRequest.headers.set('X-CF-Country', country);
  newRequest.headers.set('X-CF-Region', region);
  newRequest.headers.set('X-Routed-To', origin);
  
  try {
    const response = await fetch(newRequest);
    
    // Clone response and add region info
    const newResponse = new Response(response.body, response);
    newResponse.headers.set('X-Served-By', origin);
    newResponse.headers.set('X-Country', country);
    
    return newResponse;
  } catch (error) {
    // Failover to backup region
    const backupOrigin = getBackupRegion(origin);
    const backupUrl = new URL(url.pathname + url.search, backupOrigin);
    return fetch(new Request(backupUrl.toString(), newRequest));
  }
}

function getDataResidencyRegion(country) {
  // EU GDPR countries must stay in EU
  const EU_COUNTRIES = ['DE', 'FR', 'IT', 'ES', 'NL', 'BE', 'AT', 'PL', 'SE', 'FI', 'DK', 'IE', 'PT', 'GR', 'CZ', 'HU', 'RO', 'BG', 'HR', 'SK', 'SI', 'EE', 'LV', 'LT', 'LU', 'MT', 'CY'];
  
  if (EU_COUNTRIES.includes(country)) {
    return 'https://eu.api.example.com';
  }
  
  return null;
}

function getBackupRegion(failedOrigin) {
  const backups = {
    'https://ap.api.example.com': 'https://us.api.example.com',
    'https://us.api.example.com': 'https://eu.api.example.com',
    'https://eu.api.example.com': 'https://ap.api.example.com',
  };
  return backups[failedOrigin] || 'https://ap.api.example.com';
}
```

## 3. Multi-Region Kubernetes with ArgoCD

### 3.1 App of Apps for Multi-Region

```yaml
# gitops/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/microservices
    targetRevision: main
    path: gitops/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# gitops/apps/singapore.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices-singapore
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - service: product-service
            namespace: production
          - service: order-service
            namespace: production
          - service: payment-service
            namespace: production
  template:
    metadata:
      name: '{{service}}-sg'
    spec:
      project: singapore
      source:
        repoURL: https://github.com/org/microservices
        targetRevision: main
        path: 'helm/{{service}}'
        helm:
          valueFiles:
            - values.yaml
            - values-sg.yaml  # Region-specific overrides
      destination:
        server: https://k8s-singapore.example.com
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### 3.2 Region-Specific Helm Values

```yaml
# helm/product-service/values-sg.yaml (Singapore overrides)
replicaCount: 5

image:
  registry: sg.ecr.example.com

config:
  region: ap-southeast-1
  primaryRegion: true  # This region is primary
  
database:
  host: postgres-primary.sg.internal
  readReplica: postgres-replica.sg.internal
  
cache:
  redis:
    host: redis-cluster.sg.internal
    
cdn:
  baseUrl: https://cdn-sg.example.com

ingress:
  host: ap.api.example.com
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

autoscaling:
  minReplicas: 5
  maxReplicas: 50
  targetCPUUtilizationPercentage: 65

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi

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
```

## 4. Global PostgreSQL with CockroachDB

### 4.1 CockroachDB Multi-Region Setup

```yaml
# k8s/cockroachdb/cluster.yaml
apiVersion: crdb.cockroachlabs.com/v1alpha1
kind: CrdbCluster
metadata:
  name: global-db
  namespace: cockroachdb
spec:
  dataStore:
    pvc:
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 500Gi
        storageClassName: gp3
  
  resources:
    requests:
      cpu: 2
      memory: 8Gi
    limits:
      cpu: 4
      memory: 16Gi
  
  nodes: 9  # 3 per region
  
  additionalLabels:
    crdb.io/region: multi
  
  topology:
    regions:
      - region: ap-southeast-1  # Singapore
        nodes: 3
      - region: us-east-1       # Virginia
        nodes: 3
      - region: eu-west-1       # Ireland
        nodes: 3
    
    survivalGoal: REGION  # Survive loss of 1 region
```

### 4.2 Regional Tables & Zone Config

```sql
-- Set table zone configs for optimal latency
ALTER TABLE orders SET (gc.ttlseconds = 86400);

-- Set primary region for this table
ALTER TABLE orders SET LOCALITY REGIONAL BY ROW;

-- Products can be read globally
ALTER TABLE products SET LOCALITY GLOBAL;

-- Audit logs stay in region (compliance)
ALTER TABLE audit_logs SET LOCALITY REGIONAL BY TABLE IN PRIMARY REGION;

-- Multi-region database setup
ALTER DATABASE ecommerce PRIMARY REGION "ap-southeast-1";
ALTER DATABASE ecommerce ADD REGION "us-east-1";
ALTER DATABASE ecommerce ADD REGION "eu-west-1";

-- Survive region failure (requires 3+ regions)
ALTER DATABASE ecommerce SURVIVE REGION FAILURE;
```

## 5. Global State with Redis Global Cluster

### 5.1 Redis Geo-Replication

```javascript
// config/redis-global.js
const Redis = require('ioredis');

class GlobalRedisClient {
  constructor(config) {
    // Connect to nearest cluster
    this.localCluster = new Redis.Cluster(
      config.local.nodes,
      {
        redisOptions: { password: config.local.password },
        scaleReads: 'slave',
      }
    );
    
    // Connect to primary (for writes that need global consistency)
    this.primaryCluster = new Redis.Cluster(
      config.primary.nodes,
      {
        redisOptions: { password: config.primary.password },
      }
    );
    
    this.region = config.region;
  }
  
  // Read from local (fast, may have slight lag)
  async get(key) {
    return this.localCluster.get(key);
  }
  
  // Write to primary (replicated to all regions)
  async set(key, value, options = {}) {
    const args = [key, value];
    if (options.ex) args.push('EX', options.ex);
    if (options.px) args.push('PX', options.px);
    if (options.nx) args.push('NX');
    
    return this.primaryCluster.set(...args);
  }
  
  // Session data - use local for reads, primary for writes
  async getSession(sessionId) {
    return this.get(`session:${sessionId}`);
  }
  
  async setSession(sessionId, data, ttlSeconds = 3600) {
    return this.set(`session:${sessionId}`, JSON.stringify(data), { ex: ttlSeconds });
  }
  
  // Rate limiting - must be consistent globally
  async checkRateLimit(userId, windowMs, maxRequests) {
    const key = `rl:${this.region}:${userId}:${Math.floor(Date.now() / windowMs)}`;
    
    // Use local cluster with Redis Cluster INCR (atomic)
    const pipeline = this.localCluster.pipeline();
    pipeline.incr(key);
    pipeline.pexpire(key, windowMs);
    
    const [[, count]] = await pipeline.exec();
    return { count, allowed: count <= maxRequests };
  }
}

module.exports = GlobalRedisClient;
```

## 6. Global Event Distribution with Kafka MirrorMaker 2

```yaml
# k8s/kafka/mirrormaker2.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaMirrorMaker2
metadata:
  name: global-mirror
  namespace: kafka
spec:
  version: 3.5.1
  replicas: 2
  connectCluster: "us-east-1"
  
  clusters:
    - alias: "ap-southeast-1"
      bootstrapServers: kafka-ap.internal:9092
      config:
        ssl.endpoint.identification.algorithm: https
      authentication:
        type: tls
        certificateAndKey:
          secretName: kafka-ap-cert
          certificate: user.crt
          key: user.key
    
    - alias: "us-east-1"
      bootstrapServers: kafka-us.internal:9092
      authentication:
        type: tls
        certificateAndKey:
          secretName: kafka-us-cert
          certificate: user.crt
          key: user.key
    
    - alias: "eu-west-1"
      bootstrapServers: kafka-eu.internal:9092
      authentication:
        type: tls
        certificateAndKey:
          secretName: kafka-eu-cert
          certificate: user.crt
          key: user.key
  
  mirrors:
    # AP → US replication
    - sourceCluster: "ap-southeast-1"
      targetCluster: "us-east-1"
      sourceConnector:
        config:
          replication.factor: 3
          offset-syncs.topic.replication.factor: 3
          sync.topic.acls.enabled: "false"
          # Only replicate these topics
          topics: "orders.events,inventory.events"
      checkpointConnector:
        config:
          checkpoints.topic.replication.factor: 3
          sync.group.offsets.enabled: "true"
    
    # AP → EU replication
    - sourceCluster: "ap-southeast-1"
      targetCluster: "eu-west-1"
      sourceConnector:
        config:
          replication.factor: 3
          topics: "orders.events,inventory.events"
          # EU data residency: exclude PII topics
          topics.exclude: "user.pii.events"
```

## 7. Application Configuration for Multi-Region

```javascript
// config/multiRegion.js
const os = require('os');

class MultiRegionConfig {
  constructor() {
    this.region = process.env.DEPLOYMENT_REGION || this.detectRegion();
    this.isPrimary = process.env.IS_PRIMARY_REGION === 'true';
  }
  
  detectRegion() {
    // Try to detect from cloud metadata
    const hostname = os.hostname();
    if (hostname.includes('ap-')) return 'ap-southeast-1';
    if (hostname.includes('us-')) return 'us-east-1';
    if (hostname.includes('eu-')) return 'eu-west-1';
    return 'ap-southeast-1';
  }
  
  getDatabaseConfig() {
    const configs = {
      'ap-southeast-1': {
        primary: process.env.DB_PRIMARY_URL,
        readReplica: process.env.DB_REPLICA_URL,
        poolSize: 20,
      },
      'us-east-1': {
        primary: process.env.DB_PRIMARY_URL, // Still write to primary
        readReplica: process.env.DB_REPLICA_URL, // Read from local replica
        poolSize: 15,
      },
      'eu-west-1': {
        primary: process.env.DB_PRIMARY_URL,
        readReplica: process.env.DB_REPLICA_URL,
        poolSize: 15,
        // GDPR: ensure EU data stays in EU
        enableRowLevelSecurity: true,
      },
    };
    
    return configs[this.region] || configs['ap-southeast-1'];
  }
  
  getCacheConfig() {
    return {
      host: process.env.REDIS_HOST,
      port: 6379,
      // Cache TTL varies by region - non-primary regions may have stale data
      defaultTTL: this.isPrimary ? 3600 : 1800,
    };
  }
  
  getFeatureFlags() {
    return {
      // Some features only enabled in specific regions
      newCheckoutFlow: this.region === 'us-east-1', // A/B test in US first
      aiRecommendations: true, // Available everywhere
      advancedAnalytics: this.isPrimary, // Only process in primary
    };
  }
  
  getCircuitBreakerConfig(serviceName) {
    // More aggressive circuit breaking for cross-region calls
    const baseConfig = {
      threshold: 5,
      timeout: 10000,
      resetTimeout: 30000,
    };
    
    const isCrossRegion = this.isCrossRegionCall(serviceName);
    
    return {
      ...baseConfig,
      timeout: isCrossRegion ? 3000 : 10000, // Stricter timeout for cross-region
      threshold: isCrossRegion ? 3 : 5,
    };
  }
  
  isCrossRegionCall(serviceUrl) {
    // Detect if service is in a different region
    const regionPatterns = {
      'ap-southeast-1': /ap\./,
      'us-east-1': /us\./,
      'eu-west-1': /eu\./,
    };
    
    const currentPattern = regionPatterns[this.region];
    return currentPattern && !currentPattern.test(serviceUrl);
  }
}

module.exports = new MultiRegionConfig();
```

## 8. Cross-Region Health Monitoring

```javascript
// monitoring/globalHealthCheck.js
const axios = require('axios');
const { createClient } = require('@datadog/datadog-api-client');

class GlobalHealthMonitor {
  constructor() {
    this.regions = [
      { name: 'ap-southeast-1', url: 'https://ap.api.example.com' },
      { name: 'us-east-1', url: 'https://us.api.example.com' },
      { name: 'eu-west-1', url: 'https://eu.api.example.com' },
    ];
    
    this.alerts = [];
  }
  
  async checkAllRegions() {
    const results = await Promise.allSettled(
      this.regions.map(region => this.checkRegion(region))
    );
    
    const regionStatus = this.regions.map((region, i) => ({
      region: region.name,
      ...( results[i].status === 'fulfilled' 
        ? results[i].value 
        : { healthy: false, error: results[i].reason?.message }
      ),
    }));
    
    await this.handleAlerts(regionStatus);
    
    return {
      timestamp: new Date().toISOString(),
      regions: regionStatus,
      globalHealthy: regionStatus.every(r => r.healthy),
    };
  }
  
  async checkRegion(region) {
    const start = Date.now();
    
    try {
      const response = await axios.get(`${region.url}/health/ready`, {
        timeout: 5000,
        headers: { 'X-Health-Check': 'global-monitor' },
      });
      
      const latency = Date.now() - start;
      
      return {
        healthy: response.status === 200,
        latencyMs: latency,
        checks: response.data.checks,
        statusCode: response.status,
      };
    } catch (error) {
      return {
        healthy: false,
        latencyMs: Date.now() - start,
        error: error.code || error.message,
        statusCode: error.response?.status,
      };
    }
  }
  
  async handleAlerts(regionStatus) {
    for (const status of regionStatus) {
      if (!status.healthy) {
        const alertKey = `region-down:${status.region}`;
        
        if (!this.alerts.includes(alertKey)) {
          this.alerts.push(alertKey);
          await this.sendAlert({
            severity: 'critical',
            title: `Region ${status.region} is unhealthy`,
            body: `Health check failed: ${status.error || 'Unknown error'}`,
            region: status.region,
          });
        }
      } else {
        // Resolve existing alert
        const alertKey = `region-down:${status.region}`;
        const index = this.alerts.indexOf(alertKey);
        if (index > -1) {
          this.alerts.splice(index, 1);
          await this.sendAlert({
            severity: 'info',
            title: `Region ${status.region} has recovered`,
            body: `Health check passing again. Latency: ${status.latencyMs}ms`,
            region: status.region,
          });
        }
      }
    }
  }
  
  async sendAlert(alert) {
    // Send to PagerDuty / Slack
    console.log(`ALERT [${alert.severity.toUpperCase()}]: ${alert.title}`, alert);
    
    if (process.env.PAGERDUTY_KEY) {
      await axios.post('https://events.pagerduty.com/v2/enqueue', {
        routing_key: process.env.PAGERDUTY_KEY,
        event_action: alert.severity === 'critical' ? 'trigger' : 'resolve',
        dedup_key: `region-${alert.region}`,
        payload: {
          summary: alert.title,
          severity: alert.severity,
          source: 'global-health-monitor',
          custom_details: { body: alert.body },
        },
      });
    }
  }
}

// Run every minute
const monitor = new GlobalHealthMonitor();
setInterval(() => {
  monitor.checkAllRegions().then(status => {
    if (!status.globalHealthy) {
      console.error('Global health degraded:', status);
    }
  }).catch(console.error);
}, 60000);
```

## 9. Data Residency & GDPR Compliance

```javascript
// middleware/dataResidency.js
class DataResidencyMiddleware {
  constructor(config) {
    this.rules = config.rules;
  }
  
  middleware() {
    return (req, res, next) => {
      const country = req.headers['x-cf-country'] || req.headers['cf-ipcountry'];
      const region = this.getRegion();
      
      // Check if this region can serve this country's users
      if (!this.canServe(country, region)) {
        return res.status(302).json({
          error: {
            code: 'WRONG_REGION',
            message: 'Please use the regional endpoint',
            redirectTo: `https://${this.getRequiredRegionUrl(country)}/api/v1${req.path}`,
          }
        });
      }
      
      // Mark response with data residency info
      res.setHeader('X-Data-Region', region);
      res.setHeader('X-Data-Jurisdiction', this.getJurisdiction(country));
      
      // Log for compliance
      req.dataResidency = {
        userCountry: country,
        servingRegion: region,
        jurisdiction: this.getJurisdiction(country),
      };
      
      next();
    };
  }
  
  canServe(country, currentRegion) {
    const requiredRegion = this.getRequiredRegion(country);
    return !requiredRegion || requiredRegion === currentRegion;
  }
  
  getRequiredRegion(country) {
    const EU_COUNTRIES = new Set(['AT', 'BE', 'BG', 'CY', 'CZ', 'DE', 'DK', 'EE', 'ES', 'FI', 'FR', 'GR', 'HR', 'HU', 'IE', 'IT', 'LT', 'LU', 'LV', 'MT', 'NL', 'PL', 'PT', 'RO', 'SE', 'SI', 'SK']);
    
    if (EU_COUNTRIES.has(country)) return 'eu-west-1';
    return null; // No restriction for other countries
  }
  
  getRegion() {
    return process.env.DEPLOYMENT_REGION || 'ap-southeast-1';
  }
  
  getJurisdiction(country) {
    const EU_COUNTRIES = new Set(['AT', 'BE', 'BG', 'CY', 'CZ', 'DE', 'DK', 'EE', 'ES', 'FI', 'FR', 'GR', 'HR', 'HU', 'IE', 'IT', 'LT', 'LU', 'LV', 'MT', 'NL', 'PL', 'PT', 'RO', 'SE', 'SI', 'SK']);
    
    if (EU_COUNTRIES.has(country)) return 'GDPR-EU';
    if (country === 'TH') return 'PDPA-TH';
    if (country === 'JP') return 'APPI-JP';
    return 'GENERAL';
  }
  
  getRequiredRegionUrl(country) {
    const requiredRegion = this.getRequiredRegion(country);
    const urls = {
      'eu-west-1': 'eu.api.example.com',
      'ap-southeast-1': 'ap.api.example.com',
      'us-east-1': 'us.api.example.com',
    };
    return urls[requiredRegion] || urls['ap-southeast-1'];
  }
}

module.exports = DataResidencyMiddleware;
```

## 10. Blue-Green Multi-Region Deployment

```yaml
# .github/workflows/multi-region-deploy.yml
name: Multi-Region Deployment

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  IMAGE: ghcr.io/${{ github.repository }}/product-service

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ github.sha }}
            ${{ env.IMAGE }}:latest
  
  deploy-ap-primary:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production-ap
    steps:
      - name: Deploy to AP (Primary)
        run: |
          kubectl set image deployment/product-service \
            app=${{ env.IMAGE }}:${{ github.sha }} \
            -n production
          
          kubectl rollout status deployment/product-service \
            -n production --timeout=10m
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_AP }}
      
      - name: Run smoke tests on AP
        run: |
          sleep 30  # Wait for traffic to shift
          curl -f https://ap.api.example.com/health/ready || exit 1
          npm run test:smoke -- --url=https://ap.api.example.com
  
  deploy-us:
    needs: deploy-ap-primary
    runs-on: ubuntu-latest
    environment: production-us
    steps:
      - name: Deploy to US
        run: |
          kubectl set image deployment/product-service \
            app=${{ env.IMAGE }}:${{ github.sha }} \
            -n production
          kubectl rollout status deployment/product-service -n production --timeout=10m
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_US }}
      
      - name: Smoke test US
        run: curl -f https://us.api.example.com/health/ready || exit 1
  
  deploy-eu:
    needs: deploy-ap-primary
    runs-on: ubuntu-latest
    environment: production-eu
    steps:
      - name: Deploy to EU
        run: |
          kubectl set image deployment/product-service \
            app=${{ env.IMAGE }}:${{ github.sha }} \
            -n production
          kubectl rollout status deployment/product-service -n production --timeout=10m
        env:
          KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_EU }}
      
      - name: Smoke test EU (GDPR compliance check)
        run: |
          # Verify EU endpoint doesn't serve non-EU data
          RESPONSE=$(curl -s -H "CF-IPCountry: DE" https://eu.api.example.com/health/ready)
          echo "$RESPONSE" | jq '.checks.dataResidency.compliant' | grep -q "true"
  
  post-deployment:
    needs: [deploy-us, deploy-eu]
    runs-on: ubuntu-latest
    steps:
      - name: Global health check
        run: |
          for region in ap us eu; do
            curl -f https://${region}.api.example.com/health/ready || exit 1
          done
          echo "All regions healthy!"
      
      - name: Update status page
        run: |
          curl -X POST https://status.example.com/api/incident \
            -d '{"status": "resolved", "message": "Deployment completed successfully"}'
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Multi-Region Architecture** - Active-Active setup กับ primary/replica regions
2. **Global DNS Routing** - Cloudflare Workers สำหรับ geo-based routing และ data residency
3. **Multi-Region K8s** - ArgoCD ApplicationSet สำหรับ deploy ไปหลาย clusters
4. **Global Database** - CockroachDB multi-region ที่ survive region failure
5. **Global Cache** - Redis Geo-Replication สำหรับ session และ rate limiting
6. **Event Distribution** - Kafka MirrorMaker 2 สำหรับ cross-region event replication
7. **Data Residency** - GDPR/PDPA compliance middleware ที่บังคับ routing ถูกต้อง
8. **Global Monitoring** - Health check ทุก region + PagerDuty alerts
9. **Multi-Region Deployment** - Sequential deploy AP→US+EU พร้อม smoke tests

## แบบฝึกหัด

1. Setup 3-region ArgoCD ApplicationSet และ deploy ไปพร้อมกัน
2. Implement data residency middleware ที่ enforce GDPR routing
3. Configure CockroachDB multi-region database ด้วย REGIONAL BY ROW
4. สร้าง Cloudflare Worker ที่ route traffic ตาม continent + data residency
5. Implement global health dashboard ที่ monitor ทุก region พร้อมกัน

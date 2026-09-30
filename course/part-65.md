# Part 65: Microservices Governance

## บทนำ

เมื่อองค์กรมี Microservices จำนวนมาก การจัดการและ Govern Services เหล่านั้นให้มีประสิทธิภาพเป็นสิ่งสำคัญ Governance ครอบคลุมตั้งแต่ Service Catalog, API Versioning, Deprecation Management, ไปจนถึงโมเดลการทำงานของทีมตาม Team Topologies

## 1. Service Catalog

Service Catalog เป็น Central Registry ของ Services ทั้งหมดในองค์กร ช่วยให้ทีมค้นหาและเข้าใจ Services ที่มีอยู่

### 1.1 Service Catalog Schema

```typescript
// src/catalog/service-catalog-schema.ts

export interface ServiceEntry {
  // Identity
  id: string;
  name: string;
  displayName: string;
  description: string;
  
  // Ownership
  owner: {
    teamId: string;
    teamName: string;
    oncallSlack: string;
    oncallEmail: string;
    engineeringManager: string;
  };
  
  // Technical Details
  tech: {
    language: 'typescript' | 'go' | 'python' | 'java';
    framework: string;
    runtime: string;
    repository: string;
    cicdPipeline: string;
    artifactRegistry: string;
  };
  
  // API Information
  api: {
    type: 'rest' | 'grpc' | 'graphql' | 'event';
    version: string;
    specUrl?: string;          // OpenAPI/AsyncAPI Spec URL
    swaggerUrl?: string;
    postmanCollectionUrl?: string;
    changelogUrl?: string;
  };
  
  // Deployment
  deployment: {
    environments: Record<string, EnvironmentInfo>;
    kubernetes: {
      namespace: string;
      deploymentName: string;
    };
  };
  
  // Dependencies
  dependencies: {
    upstream: ServiceDependency[];   // Services ที่เราเรียกใช้
    downstream: ServiceDependency[]; // Services ที่เรียกใช้เรา
    databases: DatabaseDependency[];
    queues: QueueDependency[];
    externalServices: ExternalDependency[];
  };
  
  // SLA & SLO
  sla: {
    availability: number;  // เปอร์เซ็นต์ (เช่น 99.9)
    latency: {
      p50: number;         // milliseconds
      p95: number;
      p99: number;
    };
    errorRate: number;     // เปอร์เซ็นต์สูงสุดที่ยอมรับได้
  };
  
  // Security & Compliance
  security: {
    piiData: boolean;         // มีข้อมูลส่วนบุคคลหรือไม่
    financialData: boolean;   // มีข้อมูลทางการเงินหรือไม่
    authRequired: boolean;
    dataClassification: 'public' | 'internal' | 'confidential' | 'restricted';
  };
  
  // Metadata
  tags: string[];
  tier: 'tier1' | 'tier2' | 'tier3';  // ความสำคัญของ Service
  lifecycle: 'active' | 'deprecated' | 'sunset';
  createdAt: Date;
  updatedAt: Date;
}

interface EnvironmentInfo {
  url: string;
  healthCheckUrl: string;
  dashboardUrl: string;
  logsUrl: string;
  isProduction: boolean;
}

interface ServiceDependency {
  serviceId: string;
  serviceName: string;
  type: 'sync' | 'async';
  criticality: 'critical' | 'non-critical';
  notes?: string;
}

interface DatabaseDependency {
  type: 'postgresql' | 'mysql' | 'mongodb' | 'redis' | 'elasticsearch';
  name: string;
  host: string;
  isReadOnly: boolean;
}

interface QueueDependency {
  type: 'kafka' | 'rabbitmq' | 'sqs';
  topics: string[];
  role: 'producer' | 'consumer' | 'both';
}

interface ExternalDependency {
  name: string;
  url: string;
  type: 'payment_gateway' | 'sms' | 'email' | 'map' | 'other';
  notes?: string;
}
```

### 1.2 Service Catalog API

```typescript
// src/catalog/service-catalog-api.ts
import { Router, Request, Response } from 'express';
import { Pool } from 'pg';
import { ServiceEntry } from './service-catalog-schema';
import { logger } from '../utils/logger';

export class ServiceCatalogAPI {
  constructor(private db: Pool) {}

  createRouter(): Router {
    const router = Router();

    // List Services
    router.get('/services', async (req: Request, res: Response) => {
      const { tier, owner, tag, lifecycle, search } = req.query;
      
      let query = `
        SELECT id, name, display_name, description, owner, tech, api, sla, tags, tier, lifecycle, created_at
        FROM service_catalog
        WHERE 1=1
      `;
      const params: any[] = [];
      let paramIndex = 1;

      if (tier) {
        query += ` AND tier = $${paramIndex++}`;
        params.push(tier);
      }
      
      if (owner) {
        query += ` AND owner->>'teamId' = $${paramIndex++}`;
        params.push(owner);
      }
      
      if (tag) {
        query += ` AND $${paramIndex++} = ANY(tags)`;
        params.push(tag);
      }
      
      if (lifecycle) {
        query += ` AND lifecycle = $${paramIndex++}`;
        params.push(lifecycle);
      }
      
      if (search) {
        query += ` AND (name ILIKE $${paramIndex} OR description ILIKE $${paramIndex})`;
        params.push(`%${search}%`);
        paramIndex++;
      }
      
      query += ' ORDER BY tier, name';
      
      const result = await this.db.query(query, params);
      
      res.json({
        total: result.rows.length,
        services: result.rows,
      });
    });

    // Get Service Details
    router.get('/services/:id', async (req: Request, res: Response) => {
      const result = await this.db.query(
        'SELECT * FROM service_catalog WHERE id = $1',
        [req.params.id]
      );
      
      if (result.rows.length === 0) {
        return res.status(404).json({ error: 'Service not found' });
      }
      
      res.json(result.rows[0]);
    });

    // Register/Update Service
    router.put('/services/:id', async (req: Request, res: Response) => {
      const service: ServiceEntry = req.body;
      
      await this.db.query(
        `INSERT INTO service_catalog (id, name, display_name, description, owner, tech, api, deployment, dependencies, sla, security, tags, tier, lifecycle, updated_at)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, NOW())
         ON CONFLICT (id) DO UPDATE SET
           display_name = EXCLUDED.display_name,
           description = EXCLUDED.description,
           owner = EXCLUDED.owner,
           tech = EXCLUDED.tech,
           api = EXCLUDED.api,
           deployment = EXCLUDED.deployment,
           dependencies = EXCLUDED.dependencies,
           sla = EXCLUDED.sla,
           security = EXCLUDED.security,
           tags = EXCLUDED.tags,
           tier = EXCLUDED.tier,
           lifecycle = EXCLUDED.lifecycle,
           updated_at = NOW()`,
        [
          service.id, service.name, service.displayName, service.description,
          JSON.stringify(service.owner), JSON.stringify(service.tech),
          JSON.stringify(service.api), JSON.stringify(service.deployment),
          JSON.stringify(service.dependencies), JSON.stringify(service.sla),
          JSON.stringify(service.security), service.tags, service.tier,
          service.lifecycle,
        ]
      );
      
      logger.info('Service catalog updated', { serviceId: service.id });
      res.json({ success: true });
    });

    // Get Service Dependencies (Dependency Graph)
    router.get('/services/:id/dependencies', async (req: Request, res: Response) => {
      const { depth = '2' } = req.query;
      const maxDepth = parseInt(depth as string);
      
      const graph = await this.buildDependencyGraph(req.params.id, maxDepth);
      res.json(graph);
    });

    // Service Health Overview
    router.get('/services/health/overview', async (req: Request, res: Response) => {
      // Aggregate Health ของทุก Services
      const services = await this.db.query(
        `SELECT id, name, tier, deployment FROM service_catalog WHERE lifecycle = 'active'`
      );
      
      const healthChecks = await Promise.allSettled(
        services.rows.map(async (service) => {
          const healthUrl = service.deployment?.environments?.production?.healthCheckUrl;
          
          if (!healthUrl) {
            return { id: service.id, name: service.name, status: 'unknown' };
          }
          
          try {
            const response = await fetch(healthUrl, {
              signal: AbortSignal.timeout(3000),
            });
            return {
              id: service.id,
              name: service.name,
              tier: service.tier,
              status: response.ok ? 'healthy' : 'unhealthy',
            };
          } catch {
            return { id: service.id, name: service.name, tier: service.tier, status: 'unreachable' };
          }
        })
      );
      
      const results = healthChecks
        .filter(r => r.status === 'fulfilled')
        .map(r => (r as PromiseFulfilledResult<any>).value);
      
      res.json({
        total: results.length,
        healthy: results.filter(r => r.status === 'healthy').length,
        unhealthy: results.filter(r => r.status === 'unhealthy').length,
        unreachable: results.filter(r => r.status === 'unreachable').length,
        services: results,
      });
    });

    return router;
  }

  private async buildDependencyGraph(
    serviceId: string,
    maxDepth: number
  ): Promise<any> {
    const nodes = new Map<string, any>();
    const edges: any[] = [];
    
    const traverse = async (id: string, depth: number) => {
      if (depth > maxDepth || nodes.has(id)) return;
      
      const result = await this.db.query(
        'SELECT id, name, dependencies, tier FROM service_catalog WHERE id = $1',
        [id]
      );
      
      if (result.rows.length === 0) return;
      
      const service = result.rows[0];
      nodes.set(id, { id: service.id, name: service.name, tier: service.tier });
      
      const deps = service.dependencies?.upstream ?? [];
      
      for (const dep of deps) {
        edges.push({
          from: id,
          to: dep.serviceId,
          type: dep.type,
          criticality: dep.criticality,
        });
        
        await traverse(dep.serviceId, depth + 1);
      }
    };
    
    await traverse(serviceId, 0);
    
    return {
      nodes: Array.from(nodes.values()),
      edges,
    };
  }
}
```

## 2. API Versioning Strategy

### 2.1 Version Management Service

```typescript
// src/versioning/api-version-manager.ts
import { Request, Response, NextFunction, Router } from 'express';
import { logger } from '../utils/logger';

type VersionStrategy = 'url_path' | 'header' | 'query_param' | 'content_type';

interface VersionConfig {
  strategy: VersionStrategy;
  headerName?: string;         // สำหรับ header strategy
  queryParamName?: string;     // สำหรับ query_param strategy
  supportedVersions: string[];
  defaultVersion: string;
  deprecatedVersions: Record<string, {
    sunsetDate: Date;
    migrationGuide: string;
  }>;
}

export class ApiVersionManager {
  constructor(private config: VersionConfig) {}

  extractVersion(req: Request): string | null {
    switch (this.config.strategy) {
      case 'url_path': {
        const match = req.path.match(/^\/v(\d+(?:\.\d+)?)\//);
        return match ? match[1] : null;
      }
      
      case 'header': {
        const headerName = this.config.headerName ?? 'API-Version';
        return req.headers[headerName.toLowerCase()] as string ?? null;
      }
      
      case 'query_param': {
        const paramName = this.config.queryParamName ?? 'api-version';
        return req.query[paramName] as string ?? null;
      }
      
      case 'content_type': {
        // application/vnd.company.v2+json
        const contentType = req.headers['accept'] ?? req.headers['content-type'];
        const match = (contentType as string)?.match(/vnd\.\w+\.v(\d+(?:\.\d+)?)\+/);
        return match ? match[1] : null;
      }
      
      default:
        return null;
    }
  }

  createVersionMiddleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const requestedVersion = this.extractVersion(req) ?? this.config.defaultVersion;
      
      if (!this.config.supportedVersions.includes(requestedVersion)) {
        return res.status(400).json({
          error: 'UNSUPPORTED_API_VERSION',
          message: `API version ${requestedVersion} is not supported`,
          supportedVersions: this.config.supportedVersions,
          defaultVersion: this.config.defaultVersion,
        });
      }
      
      // ตรวจสอบ Deprecated Versions
      const deprecationInfo = this.config.deprecatedVersions[requestedVersion];
      
      if (deprecationInfo) {
        const sunsetDate = deprecationInfo.sunsetDate;
        
        // ตั้ง Deprecation Headers
        res.setHeader('Deprecation', 'true');
        res.setHeader('Sunset', sunsetDate.toUTCString());
        res.setHeader('Link', `<${deprecationInfo.migrationGuide}>; rel="deprecation"`);
        
        logger.warn('Deprecated API version used', {
          version: requestedVersion,
          sunsetDate,
          path: req.path,
        });
      }
      
      // ใส่ Version ใน Request
      (req as any).apiVersion = requestedVersion;
      
      // ตอบกลับด้วย Version ที่ใช้
      res.setHeader('API-Version', requestedVersion);
      
      next();
    };
  }

  createVersionedRouter(): Router {
    const router = Router();
    const versionRouters = new Map<string, Router>();
    
    for (const version of this.config.supportedVersions) {
      versionRouters.set(version, Router());
    }
    
    return router;
  }
}

// ตัวอย่าง: API Versioning สำหรับ Order Service ของไทย
class OrderServiceVersioning {
  private versionManager: ApiVersionManager;

  constructor() {
    this.versionManager = new ApiVersionManager({
      strategy: 'url_path',
      supportedVersions: ['1', '2', '3'],
      defaultVersion: '3',
      deprecatedVersions: {
        '1': {
          sunsetDate: new Date('2025-06-01'),
          migrationGuide: 'https://docs.company.th/api/migration/v1-to-v2',
        },
        '2': {
          sunsetDate: new Date('2025-12-01'),
          migrationGuide: 'https://docs.company.th/api/migration/v2-to-v3',
        },
      },
    });
  }

  setupRoutes(app: any): void {
    app.use('/api', this.versionManager.createVersionMiddleware());
    
    // V1 Routes (Deprecated)
    app.get('/api/v1/orders', this.handleGetOrdersV1.bind(this));
    
    // V2 Routes (Deprecated soon)
    app.get('/api/v2/orders', this.handleGetOrdersV2.bind(this));
    
    // V3 Routes (Current)
    app.get('/api/v3/orders', this.handleGetOrdersV3.bind(this));
  }

  private async handleGetOrdersV1(req: Request, res: Response): Promise<void> {
    // V1 Format (เก่า - ไม่มี Pagination)
    res.json([/* orders */]);
  }

  private async handleGetOrdersV2(req: Request, res: Response): Promise<void> {
    // V2 Format (มี Pagination แต่ Format เก่า)
    res.json({
      orders: [/* orders */],
      total: 0,
      page: 1,
    });
  }

  private async handleGetOrdersV3(req: Request, res: Response): Promise<void> {
    // V3 Format (Current - Cursor-based Pagination)
    res.json({
      data: [/* orders */],
      pagination: {
        cursor: 'abc123',
        hasMore: false,
        total: 0,
      },
      meta: {
        apiVersion: '3',
        requestId: (req as any).id,
      },
    });
  }
}
```

## 3. Deprecation Management

### 3.1 API Deprecation Tracker

```typescript
// src/governance/deprecation-manager.ts
import { Pool } from 'pg';
import { AlertManager } from '../monitoring/alerts';
import { logger } from '../utils/logger';

interface DeprecationRecord {
  id: string;
  serviceId: string;
  apiPath: string;
  apiVersion: string;
  deprecatedAt: Date;
  sunsetDate: Date;
  replacedBy?: string;
  migrationGuide?: string;
  activeConsumers: string[];
  notificationSent: boolean;
}

export class DeprecationManager {
  constructor(
    private db: Pool,
    private alerts: AlertManager
  ) {}

  async registerDeprecation(record: Omit<DeprecationRecord, 'id' | 'notificationSent'>): Promise<void> {
    const id = crypto.randomUUID();
    
    await this.db.query(
      `INSERT INTO api_deprecations 
         (id, service_id, api_path, api_version, deprecated_at, sunset_date, replaced_by, migration_guide, active_consumers, notification_sent)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, false)`,
      [
        id, record.serviceId, record.apiPath, record.apiVersion,
        record.deprecatedAt, record.sunsetDate, record.replacedBy,
        record.migrationGuide, record.activeConsumers,
      ]
    );
    
    logger.info('API deprecation registered', {
      serviceId: record.serviceId,
      apiPath: record.apiPath,
      sunsetDate: record.sunsetDate,
    });
    
    // แจ้งเตือน Consumers ทันที
    await this.notifyConsumers(record);
  }

  async checkSunsetWarnings(): Promise<void> {
    const now = new Date();
    const thirtyDaysFromNow = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);
    
    // ค้นหา API ที่จะถูก Sunset ใน 30 วัน
    const upcoming = await this.db.query(
      `SELECT * FROM api_deprecations 
       WHERE sunset_date <= $1 AND sunset_date > $2
       AND array_length(active_consumers, 1) > 0`,
      [thirtyDaysFromNow, now]
    );

    for (const deprecation of upcoming.rows) {
      const daysUntilSunset = Math.ceil(
        (deprecation.sunset_date.getTime() - now.getTime()) / (1000 * 60 * 60 * 24)
      );
      
      await this.alerts.sendAlert({
        severity: daysUntilSunset <= 7 ? 'critical' : 'warning',
        title: `API Sunset Warning: ${deprecation.api_path}`,
        message: `API ${deprecation.api_path} (v${deprecation.api_version}) will be sunset in ${daysUntilSunset} days`,
        details: {
          sunsetDate: deprecation.sunset_date,
          activeConsumers: deprecation.active_consumers,
          migrationGuide: deprecation.migration_guide,
        },
      });
    }
  }

  async trackUsage(serviceId: string, apiPath: string, consumerId: string): Promise<void> {
    // ติดตามการใช้งาน API ที่ Deprecated
    const deprecation = await this.db.query(
      `SELECT id, active_consumers FROM api_deprecations 
       WHERE service_id = $1 AND api_path = $2`,
      [serviceId, apiPath]
    );
    
    if (deprecation.rows.length === 0) return;
    
    const record = deprecation.rows[0];
    const consumers = record.active_consumers as string[];
    
    if (!consumers.includes(consumerId)) {
      await this.db.query(
        `UPDATE api_deprecations 
         SET active_consumers = array_append(active_consumers, $1)
         WHERE id = $2`,
        [consumerId, record.id]
      );
    }
  }

  async getDeprecationReport(): Promise<any[]> {
    const result = await this.db.query(`
      SELECT 
        d.*,
        s.name as service_name,
        s.owner->>'teamName' as owner_team
      FROM api_deprecations d
      JOIN service_catalog s ON d.service_id = s.id
      WHERE d.sunset_date > NOW()
      ORDER BY d.sunset_date ASC
    `);
    
    return result.rows;
  }

  private async notifyConsumers(record: Omit<DeprecationRecord, 'id' | 'notificationSent'>): Promise<void> {
    for (const consumer of record.activeConsumers) {
      await this.alerts.sendSlackMessage({
        channel: `#team-${consumer}`,
        message: {
          text: `⚠️ API Deprecation Notice`,
          blocks: [
            {
              type: 'section',
              text: {
                type: 'mrkdwn',
                text: `*API Deprecation Notice*\n` +
                      `The API \`${record.apiPath}\` (v${record.apiVersion}) has been deprecated.\n\n` +
                      `*Sunset Date:* ${record.sunsetDate.toLocaleDateString('th-TH')}\n` +
                      `*Replaced By:* ${record.replacedBy ?? 'N/A'}\n` +
                      `*Migration Guide:* ${record.migrationGuide ?? 'N/A'}`,
              },
            },
          ],
        },
      });
    }
  }
}
```

## 4. Service Ownership Model

### 4.1 CODEOWNERS Configuration

```
# .github/CODEOWNERS
# รูปแบบ: pattern @team-or-user

# Payment Services - Team Payment
/services/payment-service/       @company/team-payment
/services/wallet-service/        @company/team-payment
/services/billing-service/       @company/team-payment

# Order Management - Team Orders
/services/order-service/         @company/team-orders
/services/cart-service/          @company/team-orders
/services/fulfillment-service/   @company/team-orders

# User & Auth - Team Platform
/services/user-service/          @company/team-platform
/services/auth-service/          @company/team-platform

# Product & Search - Team Product
/services/product-service/       @company/team-product
/services/search-service/        @company/team-product
/services/recommendation-service/ @company/team-product

# Infrastructure - Team SRE
/infrastructure/                 @company/team-sre
/kubernetes/                     @company/team-sre
/.github/workflows/              @company/team-sre
/monitoring/                     @company/team-sre

# Shared Libraries - All teams must review
/libs/shared-types/              @company/team-platform @company/team-sre
/libs/common-utils/              @company/team-platform
```

### 4.2 Service Ownership Registry

```typescript
// src/governance/service-ownership.ts

interface TeamInfo {
  id: string;
  name: string;
  slackChannel: string;
  oncallRotation: string;
  members: TeamMember[];
  services: string[]; // Service IDs
}

interface TeamMember {
  userId: string;
  name: string;
  role: 'tech_lead' | 'senior_engineer' | 'engineer' | 'engineering_manager';
  email: string;
}

interface ServiceOwnershipPolicy {
  maxServicesPerTeam: number;       // จำกัดจำนวน Services ต่อทีม
  requireOnCallRotation: boolean;   // ต้องมี On-call Rotation
  requireRunbook: boolean;          // ต้องมี Runbook
  minTeamSize: number;              // ขนาดทีมขั้นต่ำสำหรับ Tier-1 Service
}

// Default Policy สำหรับองค์กร
export const defaultOwnershipPolicy: ServiceOwnershipPolicy = {
  maxServicesPerTeam: 10,
  requireOnCallRotation: true,
  requireRunbook: true,
  minTeamSize: 3,
};

export class ServiceOwnershipManager {
  constructor(
    private db: any,
    private policy: ServiceOwnershipPolicy = defaultOwnershipPolicy
  ) {}

  async assignOwnership(serviceId: string, teamId: string): Promise<void> {
    // ตรวจสอบ Policy
    const team = await this.getTeam(teamId);
    
    if (!team) {
      throw new Error(`Team ${teamId} not found`);
    }
    
    if (team.services.length >= this.policy.maxServicesPerTeam) {
      throw new Error(
        `Team ${team.name} already owns ${team.services.length} services. ` +
        `Maximum is ${this.policy.maxServicesPerTeam}`
      );
    }
    
    if (this.policy.requireOnCallRotation && !team.oncallRotation) {
      throw new Error(`Team ${team.name} must have an on-call rotation before owning a service`);
    }

    await this.db.query(
      `UPDATE service_catalog SET owner = jsonb_set(owner, '{teamId}', $1::jsonb) WHERE id = $2`,
      [JSON.stringify(teamId), serviceId]
    );
    
    await this.db.query(
      `UPDATE teams SET services = array_append(services, $1) WHERE id = $2`,
      [serviceId, teamId]
    );
  }

  async getServiceOwner(serviceId: string): Promise<TeamInfo | null> {
    const result = await this.db.query(
      `SELECT t.* FROM teams t
       JOIN service_catalog sc ON sc.owner->>'teamId' = t.id
       WHERE sc.id = $1`,
      [serviceId]
    );
    
    return result.rows[0] ?? null;
  }

  private async getTeam(teamId: string): Promise<TeamInfo | null> {
    const result = await this.db.query(
      'SELECT * FROM teams WHERE id = $1',
      [teamId]
    );
    return result.rows[0] ?? null;
  }
}
```

## 5. Team Topologies

### 5.1 Team Types และ Interaction Modes

```typescript
// src/governance/team-topologies.ts

// ตามหนังสือ "Team Topologies" โดย Matthew Skelton และ Manuel Pais
export type TeamType = 
  | 'stream_aligned'    // ทีมที่ Align กับ Business Domain
  | 'platform'          // ทีมที่ดูแล Internal Platform
  | 'enabling'          // ทีมที่ช่วย Enable Team อื่น
  | 'complicated_subsystem'; // ทีมที่ดูแล Subsystem ที่ซับซ้อน

export type InteractionMode = 
  | 'collaboration'   // ทำงานร่วมกันแบบ Close
  | 'x_as_a_service'  // ให้บริการผ่าน API/Docs
  | 'facilitating';   // ช่วยสอน/Guide

interface TeamTopologyConfig {
  team: {
    id: string;
    name: string;
    type: TeamType;
    domain: string;
    services: string[];
    members: number;
  };
  interactions: Array<{
    withTeam: string;
    mode: InteractionMode;
    purpose: string;
    duration?: 'temporary' | 'permanent';
  }>;
}

// ตัวอย่าง Team Topology ของ Thai E-commerce Platform
export const thaiEcommerceTeamTopology: TeamTopologyConfig[] = [
  {
    team: {
      id: 'team-checkout',
      name: 'Checkout Stream',
      type: 'stream_aligned',
      domain: 'Checkout & Payment',
      services: ['order-service', 'payment-service', 'cart-service'],
      members: 6,
    },
    interactions: [
      {
        withTeam: 'team-platform',
        mode: 'x_as_a_service',
        purpose: 'Use Platform services (Auth, Notification, Config)',
        duration: 'permanent',
      },
      {
        withTeam: 'team-catalog',
        mode: 'x_as_a_service',
        purpose: 'Get product information and inventory',
        duration: 'permanent',
      },
    ],
  },
  {
    team: {
      id: 'team-catalog',
      name: 'Product Catalog Stream',
      type: 'stream_aligned',
      domain: 'Product & Inventory',
      services: ['product-service', 'inventory-service', 'search-service'],
      members: 5,
    },
    interactions: [
      {
        withTeam: 'team-platform',
        mode: 'x_as_a_service',
        purpose: 'Use Platform services',
        duration: 'permanent',
      },
    ],
  },
  {
    team: {
      id: 'team-platform',
      name: 'Platform Engineering',
      type: 'platform',
      domain: 'Developer Platform',
      services: ['auth-service', 'notification-service', 'config-service', 'api-gateway'],
      members: 8,
    },
    interactions: [
      {
        withTeam: 'team-checkout',
        mode: 'x_as_a_service',
        purpose: 'Provide platform capabilities',
        duration: 'permanent',
      },
      {
        withTeam: 'team-catalog',
        mode: 'x_as_a_service',
        purpose: 'Provide platform capabilities',
        duration: 'permanent',
      },
    ],
  },
  {
    team: {
      id: 'team-sre',
      name: 'Site Reliability Engineering',
      type: 'enabling',
      domain: 'Reliability & Operations',
      services: [],
      members: 4,
    },
    interactions: [
      {
        withTeam: 'team-checkout',
        mode: 'facilitating',
        purpose: 'Enable SRE practices, SLO definition, On-call setup',
        duration: 'temporary',
      },
      {
        withTeam: 'team-catalog',
        mode: 'facilitating',
        purpose: 'Enable SRE practices',
        duration: 'temporary',
      },
    ],
  },
];
```

## 6. Inner Source

Inner Source ช่วยให้ทีมต่างๆ ใน Organization สามารถ Contribute Code ให้กันได้เหมือน Open Source

### 6.1 Inner Source Guidelines

```typescript
// src/governance/inner-source-config.ts

interface InnerSourcePolicy {
  // การ Accept Contribution
  contribution: {
    requireCodeReview: boolean;
    minReviewers: number;
    requireOwnerApproval: boolean;
    requireTests: boolean;
    requireDocumentation: boolean;
    autoMergeEnabled: boolean;
  };
  
  // การ Communicate
  communication: {
    requireIssueBeforePR: boolean;
    requireRFCForBreakingChanges: boolean;
    rfcDiscussionPeriodDays: number;
  };
  
  // Recognition
  recognition: {
    trackContributors: boolean;
    contributeToPerformanceReview: boolean;
    publicContributorBoard: boolean;
  };
}

export const defaultInnerSourcePolicy: InnerSourcePolicy = {
  contribution: {
    requireCodeReview: true,
    minReviewers: 2,
    requireOwnerApproval: true,
    requireTests: true,
    requireDocumentation: true,
    autoMergeEnabled: false,
  },
  communication: {
    requireIssueBeforePR: true,
    requireRFCForBreakingChanges: true,
    rfcDiscussionPeriodDays: 7,
  },
  recognition: {
    trackContributors: true,
    contributeToPerformanceReview: true,
    publicContributorBoard: true,
  },
};

// CONTRIBUTING.md Generator
export function generateContributingGuide(
  serviceName: string,
  teamName: string,
  policy: InnerSourcePolicy
): string {
  return `
# Contributing to ${serviceName}

ยินดีต้อนรับทุกคนที่ต้องการ Contribute ให้กับ ${serviceName}!
Service นี้ดูแลโดยทีม **${teamName}**

## ขั้นตอนการ Contribute

### 1. เปิด Issue ก่อน
${policy.communication.requireIssueBeforePR ? 
  'กรุณาเปิด Issue ก่อนสร้าง Pull Request เพื่อหารือเกี่ยวกับ Feature หรือ Bug ที่จะแก้ไข' : 
  'สามารถสร้าง Pull Request ได้โดยตรง'}

### 2. Fork และ Clone Repository
\`\`\`bash
git clone https://github.com/company/${serviceName}.git
cd ${serviceName}
npm install
\`\`\`

### 3. สร้าง Branch ใหม่
\`\`\`bash
git checkout -b feature/your-feature-name
# หรือ
git checkout -b fix/bug-description
\`\`\`

### 4. เขียน Code และ Tests
${policy.contribution.requireTests ? '- **ต้องมี Unit Tests** สำหรับทุก Feature ใหม่\n- Test Coverage ต้องไม่ต่ำกว่า 80%' : ''}
${policy.contribution.requireDocumentation ? '- อัพเดต Documentation ถ้ามีการเปลี่ยนแปลง API' : ''}

### 5. สร้าง Pull Request
- ต้องผ่านการ Review จาก **${policy.contribution.minReviewers} คน**
${policy.contribution.requireOwnerApproval ? '- **ต้องได้รับการ Approve จาก Service Owner** (ทีม ' + teamName + ')' : ''}

## Breaking Changes
${policy.communication.requireRFCForBreakingChanges ?
  `Breaking Changes ต้องผ่านกระบวนการ RFC (Request for Comments) โดยต้องเปิด Discussion ไว้ ${policy.communication.rfcDiscussionPeriodDays} วัน ก่อนที่จะ Merge` :
  'Breaking Changes ต้องระบุใน Commit Message อย่างชัดเจน'}

## Contact
มีคำถามหรือต้องการความช่วยเหลือ ติดต่อทีม ${teamName} ที่ Slack Channel #team-${teamName.toLowerCase().replace(/\s+/g, '-')}
  `.trim();
}
```

## 7. Governance Dashboard

### 7.1 Governance Metrics Collector

```typescript
// src/governance/governance-metrics.ts
import { Pool } from 'pg';

interface GovernanceMetrics {
  services: {
    total: number;
    byTier: Record<string, number>;
    byLifecycle: Record<string, number>;
    withOpenapi: number;
    withRunbooks: number;
    withOnCall: number;
  };
  deprecations: {
    total: number;
    upcoming30Days: number;
    upcoming90Days: number;
    overdue: number;
  };
  teams: {
    total: number;
    avgServicesPerTeam: number;
    withOnCall: number;
    withDocumentation: number;
  };
  compliance: {
    servicesWithSLO: number;
    servicesWithHealthCheck: number;
    servicesWithCI: number;
    overallComplianceScore: number;
  };
}

export class GovernanceMetricsCollector {
  constructor(private db: Pool) {}

  async collect(): Promise<GovernanceMetrics> {
    const [
      serviceStats,
      deprecationStats,
      teamStats,
      complianceStats,
    ] = await Promise.all([
      this.collectServiceStats(),
      this.collectDeprecationStats(),
      this.collectTeamStats(),
      this.collectComplianceStats(),
    ]);

    return {
      services: serviceStats,
      deprecations: deprecationStats,
      teams: teamStats,
      compliance: complianceStats,
    };
  }

  private async collectServiceStats() {
    const total = await this.db.query('SELECT COUNT(*) FROM service_catalog');
    
    const byTier = await this.db.query(
      'SELECT tier, COUNT(*) as count FROM service_catalog GROUP BY tier'
    );
    
    const byLifecycle = await this.db.query(
      'SELECT lifecycle, COUNT(*) as count FROM service_catalog GROUP BY lifecycle'
    );
    
    const withOpenapi = await this.db.query(
      "SELECT COUNT(*) FROM service_catalog WHERE api->>'specUrl' IS NOT NULL"
    );

    return {
      total: parseInt(total.rows[0].count),
      byTier: Object.fromEntries(byTier.rows.map(r => [r.tier, parseInt(r.count)])),
      byLifecycle: Object.fromEntries(byLifecycle.rows.map(r => [r.lifecycle, parseInt(r.count)])),
      withOpenapi: parseInt(withOpenapi.rows[0].count),
      withRunbooks: 0,
      withOnCall: 0,
    };
  }

  private async collectDeprecationStats() {
    const now = new Date();
    const in30Days = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);
    const in90Days = new Date(Date.now() + 90 * 24 * 60 * 60 * 1000);

    const [total, upcoming30, upcoming90, overdue] = await Promise.all([
      this.db.query('SELECT COUNT(*) FROM api_deprecations WHERE sunset_date > NOW()'),
      this.db.query('SELECT COUNT(*) FROM api_deprecations WHERE sunset_date BETWEEN NOW() AND $1', [in30Days]),
      this.db.query('SELECT COUNT(*) FROM api_deprecations WHERE sunset_date BETWEEN NOW() AND $1', [in90Days]),
      this.db.query('SELECT COUNT(*) FROM api_deprecations WHERE sunset_date < NOW()'),
    ]);

    return {
      total: parseInt(total.rows[0].count),
      upcoming30Days: parseInt(upcoming30.rows[0].count),
      upcoming90Days: parseInt(upcoming90.rows[0].count),
      overdue: parseInt(overdue.rows[0].count),
    };
  }

  private async collectTeamStats() {
    const total = await this.db.query('SELECT COUNT(*) FROM teams');
    
    const avgServices = await this.db.query(
      'SELECT AVG(array_length(services, 1)) as avg FROM teams'
    );

    return {
      total: parseInt(total.rows[0].count),
      avgServicesPerTeam: parseFloat(avgServices.rows[0].avg ?? 0),
      withOnCall: 0,
      withDocumentation: 0,
    };
  }

  private async collectComplianceStats() {
    const total = await this.db.query(
      "SELECT COUNT(*) FROM service_catalog WHERE lifecycle = 'active'"
    );
    const totalCount = parseInt(total.rows[0].count);

    const withSLO = await this.db.query(
      "SELECT COUNT(*) FROM service_catalog WHERE lifecycle = 'active' AND sla IS NOT NULL"
    );
    
    const withHealthCheck = await this.db.query(
      `SELECT COUNT(*) FROM service_catalog 
       WHERE lifecycle = 'active' 
       AND deployment->'environments'->'production'->>'healthCheckUrl' IS NOT NULL`
    );

    const sloCount = parseInt(withSLO.rows[0].count);
    const healthCount = parseInt(withHealthCheck.rows[0].count);
    
    const overallScore = totalCount > 0
      ? Math.round(((sloCount + healthCount) / (totalCount * 2)) * 100)
      : 0;

    return {
      servicesWithSLO: sloCount,
      servicesWithHealthCheck: healthCount,
      servicesWithCI: 0,
      overallComplianceScore: overallScore,
    };
  }
}

// API Endpoint สำหรับ Dashboard
export function createGovernanceDashboardRouter(
  metricsCollector: GovernanceMetricsCollector,
  catalogAPI: any,
  deprecationManager: any
) {
  const { Router } = require('express');
  const router = Router();

  router.get('/dashboard/overview', async (req: Request, res: Response) => {
    const metrics = await metricsCollector.collect();
    res.json(metrics);
  });

  router.get('/dashboard/deprecation-report', async (req: Request, res: Response) => {
    const report = await deprecationManager.getDeprecationReport();
    res.json(report);
  });

  router.get('/dashboard/compliance', async (req: Request, res: Response) => {
    const metrics = await metricsCollector.collect();
    res.json({
      score: metrics.compliance.overallComplianceScore,
      details: metrics.compliance,
      recommendations: generateComplianceRecommendations(metrics),
    });
  });

  return router;
}

function generateComplianceRecommendations(metrics: GovernanceMetrics): string[] {
  const recommendations: string[] = [];
  
  const totalActive = metrics.services.byLifecycle['active'] ?? 0;
  
  if (metrics.compliance.servicesWithSLO < totalActive) {
    const missing = totalActive - metrics.compliance.servicesWithSLO;
    recommendations.push(
      `${missing} services ยังไม่มีการกำหนด SLO กรุณาเพิ่ม SLA configuration ใน Service Catalog`
    );
  }
  
  if (metrics.compliance.servicesWithHealthCheck < totalActive) {
    const missing = totalActive - metrics.compliance.servicesWithHealthCheck;
    recommendations.push(
      `${missing} services ยังไม่มี Health Check Endpoint กรุณาเพิ่ม /health endpoint`
    );
  }
  
  if (metrics.deprecations.overdue > 0) {
    recommendations.push(
      `มี ${metrics.deprecations.overdue} API ที่เลย Sunset Date แล้ว กรุณาลบออกหรืออัพเดต Sunset Date`
    );
  }
  
  if (metrics.deprecations.upcoming30Days > 0) {
    recommendations.push(
      `มี ${metrics.deprecations.upcoming30Days} API ที่จะถูก Sunset ใน 30 วัน กรุณาตรวจสอบ Consumers ที่ยังใช้งานอยู่`
    );
  }
  
  return recommendations;
}

interface Request {
  params: any;
  query: any;
  body: any;
}

interface Response {
  json: (data: any) => void;
  status: (code: number) => Response;
}
```

## สรุป

| หัวข้อ Governance | เครื่องมือ/Pattern | ประโยชน์หลัก | ความสำคัญ |
|----------------|----------------|------------|---------|
| Service Catalog | Backstage, Custom API | ค้นหาและเข้าใจ Services ง่าย | สูงมาก |
| API Versioning | URL Path, Headers | Backward Compatibility | สูงมาก |
| Deprecation Management | Sunset Headers, Notifications | Migration ราบรื่น | สูง |
| Dependency Management | SBOM, Renovate Bot | Security & Maintenance | สูง |
| Service Ownership | CODEOWNERS, Registry | Clear Accountability | สูงมาก |
| Team Topologies | Stream-aligned, Platform | Fast Flow, Low Cognitive Load | สูง |
| Inner Source | Contribution Guidelines | Knowledge Sharing | ปานกลาง |
| Governance Dashboard | Metrics, Reports | Visibility & Compliance | ปานกลาง |

Governance ที่ดีช่วยให้องค์กรสามารถ Scale ระบบ Microservices ได้โดยไม่เกิด Chaos สำหรับบริษัท Thai Tech ที่กำลังเติบโต ควรเริ่มจาก Service Catalog และ Service Ownership ก่อน แล้วค่อยๆ เพิ่ม Tooling และ Process เมื่อทีมเติบโตขึ้น อย่าพยายามทำทุกอย่างพร้อมกันเพราะจะทำให้ทีมเสียเวลากับ Process มากกว่า Product

# Part 75: Edge Computing และ CDN Integration

## บทนำ

Edge Computing และ CDN (Content Delivery Network) เป็นส่วนสำคัญในการสร้าง Microservices ที่มีประสิทธิภาพสูงและ latency ต่ำ บทนี้จะครอบคลุม Cloudflare Workers, Edge Functions, CDN Cache Strategies, Geographic Routing, Real User Monitoring และการ Purge Cache อัตโนมัติ

---

## 1. Cloudflare Workers

### 1.1 Worker สำหรับ API Gateway ที่ Edge

```typescript
// workers/api-gateway.ts
import { Router } from 'itty-router';

interface Env {
  ORIGIN_URL: string;
  RATE_LIMIT: KVNamespace;
  CACHE: KVNamespace;
  JWT_SECRET: string;
  API_TOKENS: KVNamespace;
}

const router = Router();

// Middleware: Rate Limiting
async function rateLimitMiddleware(request: Request, env: Env): Promise<Response | null> {
  const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
  const key = `rate:${ip}:${Math.floor(Date.now() / 60000)}`; // per minute

  const current = parseInt((await env.RATE_LIMIT.get(key)) || '0');
  const limit = 100; // 100 requests per minute

  if (current >= limit) {
    return new Response(
      JSON.stringify({ error: 'Rate limit exceeded', retryAfter: 60 }),
      {
        status: 429,
        headers: {
          'Content-Type': 'application/json',
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': '0',
          'Retry-After': '60',
        },
      }
    );
  }

  await env.RATE_LIMIT.put(key, (current + 1).toString(), { expirationTtl: 60 });
  return null; // Continue
}

// Middleware: Auth
async function authMiddleware(request: Request, env: Env): Promise<Response | null> {
  const authHeader = request.headers.get('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return new Response(JSON.stringify({ error: 'Unauthorized' }), {
      status: 401,
      headers: { 'Content-Type': 'application/json' },
    });
  }

  const token = authHeader.substring(7);
  const isValid = await env.API_TOKENS.get(token);
  if (!isValid) {
    return new Response(JSON.stringify({ error: 'Invalid token' }), {
      status: 403,
      headers: { 'Content-Type': 'application/json' },
    });
  }

  return null; // Continue
}

// Routes
router.get('/api/products/:id', async (request: Request, env: Env) => {
  const { id } = (request as any).params;
  const cacheKey = `product:${id}`;
  
  // Try cache first
  const cached = await env.CACHE.get(cacheKey);
  if (cached) {
    return new Response(cached, {
      headers: {
        'Content-Type': 'application/json',
        'X-Cache': 'HIT',
        'Cache-Control': 'public, max-age=300',
      },
    });
  }

  // Fetch from origin
  const originResponse = await fetch(`${env.ORIGIN_URL}/products/${id}`, {
    headers: {
      'X-Forwarded-For': request.headers.get('CF-Connecting-IP') || '',
      'X-Edge-Location': request.cf?.colo || 'unknown',
    },
  });

  const data = await originResponse.text();
  
  // Cache for 5 minutes
  await env.CACHE.put(cacheKey, data, { expirationTtl: 300 });

  return new Response(data, {
    status: originResponse.status,
    headers: {
      'Content-Type': 'application/json',
      'X-Cache': 'MISS',
      'Cache-Control': 'public, max-age=300',
    },
  });
});

router.post('/api/orders', async (request: Request, env: Env) => {
  // POST requests — no cache, forward directly
  const body = await request.text();
  
  const response = await fetch(`${env.ORIGIN_URL}/orders`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Forwarded-For': request.headers.get('CF-Connecting-IP') || '',
      'X-Country-Code': request.cf?.country || 'XX',
    },
    body,
  });

  return response;
});

// Main handler
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Apply middleware
    const rateLimitResponse = await rateLimitMiddleware(request, env);
    if (rateLimitResponse) return rateLimitResponse;

    const url = new URL(request.url);
    if (url.pathname.startsWith('/api/')) {
      const authResponse = await authMiddleware(request, env);
      if (authResponse) return authResponse;
    }

    // Add security headers to all responses
    const response = await router.handle(request, env);
    const newHeaders = new Headers(response.headers);
    newHeaders.set('X-Content-Type-Options', 'nosniff');
    newHeaders.set('X-Frame-Options', 'DENY');
    newHeaders.set('X-XSS-Protection', '1; mode=block');
    newHeaders.set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
    newHeaders.set('X-Edge-Colo', request.cf?.colo || 'unknown');

    return new Response(response.body, {
      status: response.status,
      headers: newHeaders,
    });
  },
};
```

### 1.2 Wrangler Configuration

```toml
# wrangler.toml
name = "api-gateway-worker"
main = "src/worker.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

[vars]
ORIGIN_URL = "https://api.internal.company.com"

[[kv_namespaces]]
binding = "RATE_LIMIT"
id = "your-rate-limit-kv-id"
preview_id = "your-rate-limit-preview-id"

[[kv_namespaces]]
binding = "CACHE"
id = "your-cache-kv-id"
preview_id = "your-cache-preview-id"

[[kv_namespaces]]
binding = "API_TOKENS"
id = "your-tokens-kv-id"
preview_id = "your-tokens-preview-id"

[env.production]
vars = { ENVIRONMENT = "production" }
routes = [
  { pattern = "api.company.com/*", zone_name = "company.com" }
]

[env.staging]
vars = { ENVIRONMENT = "staging" }
routes = [
  { pattern = "api-staging.company.com/*", zone_name = "company.com" }
]

[[rules]]
type = "ESModule"
globs = ["**/*.ts"]

[build]
command = "npm run build"

[triggers]
crons = ["0 0 * * *"]  # Daily cache cleanup
```

---

## 2. Edge Functions ด้วย Vercel/Next.js

### 2.1 Edge Middleware

```typescript
// middleware.ts (Next.js Edge Middleware)
import { NextResponse, NextRequest } from 'next/server';
import { jwtVerify } from 'jose';

export const config = {
  matcher: ['/api/:path*', '/dashboard/:path*'],
  runtime: 'edge',
};

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET!);

// Geographic routing configuration
const GEO_ROUTING: Record<string, string> = {
  TH: 'https://api-th.company.com',
  SG: 'https://api-sg.company.com',
  JP: 'https://api-jp.company.com',
  US: 'https://api-us.company.com',
  EU: 'https://api-eu.company.com',
};

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // A/B Testing at edge
  if (pathname === '/') {
    const abVariant = getABVariant(request);
    const response = NextResponse.next();
    response.cookies.set('ab_variant', abVariant, {
      maxAge: 60 * 60 * 24 * 30, // 30 days
      httpOnly: false,
      secure: true,
      sameSite: 'lax',
    });
    response.headers.set('X-AB-Variant', abVariant);
    return response;
  }

  // Geographic routing
  if (pathname.startsWith('/api/')) {
    const country = request.geo?.country || 'US';
    const originUrl = GEO_ROUTING[country] || GEO_ROUTING['US'];
    
    // Rewrite to closest origin
    const url = new URL(pathname, originUrl);
    url.search = request.nextUrl.search;

    const response = NextResponse.rewrite(url);
    response.headers.set('X-Country', country);
    response.headers.set('X-Origin', originUrl);
    return response;
  }

  // Auth protection
  if (pathname.startsWith('/dashboard/')) {
    const token = request.cookies.get('auth_token')?.value ||
                  request.headers.get('Authorization')?.replace('Bearer ', '');

    if (!token) {
      const loginUrl = new URL('/login', request.url);
      loginUrl.searchParams.set('redirect', pathname);
      return NextResponse.redirect(loginUrl);
    }

    try {
      const { payload } = await jwtVerify(token, SECRET);
      const response = NextResponse.next();
      response.headers.set('X-User-Id', payload.sub as string);
      response.headers.set('X-User-Role', payload.role as string);
      return response;
    } catch {
      const loginUrl = new URL('/login', request.url);
      return NextResponse.redirect(loginUrl);
    }
  }

  return NextResponse.next();
}

function getABVariant(request: NextRequest): string {
  // Use existing cookie or assign new variant
  const existing = request.cookies.get('ab_variant')?.value;
  if (existing) return existing;

  // Deterministic assignment based on IP
  const ip = request.ip || '0.0.0.0';
  const hash = ip.split('.').reduce((acc, part) => acc + parseInt(part), 0);
  return hash % 2 === 0 ? 'control' : 'treatment';
}
```

---

## 3. CDN Cache Strategies

### 3.1 Cache Control Headers Service

```typescript
// src/middleware/cache-control.ts
import { Request, Response, NextFunction } from 'express';

type CacheStrategy =
  | 'no-cache'
  | 'short'
  | 'medium'
  | 'long'
  | 'immutable'
  | 'private'
  | 'custom';

interface CacheConfig {
  strategy: CacheStrategy;
  maxAge?: number;
  sMaxAge?: number;
  staleWhileRevalidate?: number;
  staleIfError?: number;
  private?: boolean;
  varyHeaders?: string[];
}

const CACHE_PRESETS: Record<string, CacheConfig> = {
  'api-public': {
    strategy: 'short',
    maxAge: 60,
    sMaxAge: 300,
    staleWhileRevalidate: 60,
    staleIfError: 3600,
  },
  'api-user': {
    strategy: 'private',
    maxAge: 0,
    private: true,
  },
  'static-assets': {
    strategy: 'immutable',
    maxAge: 31536000,
    sMaxAge: 31536000,
  },
  'html-pages': {
    strategy: 'short',
    maxAge: 0,
    sMaxAge: 60,
    staleWhileRevalidate: 600,
  },
};

export function cacheControl(presetOrConfig: string | CacheConfig) {
  return (req: Request, res: Response, next: NextFunction) => {
    const config: CacheConfig =
      typeof presetOrConfig === 'string'
        ? CACHE_PRESETS[presetOrConfig] || CACHE_PRESETS['api-public']
        : presetOrConfig;

    const directives: string[] = [];

    if (config.private) {
      directives.push('private');
    } else {
      directives.push('public');
    }

    if (config.maxAge !== undefined) {
      directives.push(`max-age=${config.maxAge}`);
    }

    if (config.sMaxAge !== undefined) {
      directives.push(`s-maxage=${config.sMaxAge}`);
    }

    if (config.staleWhileRevalidate !== undefined) {
      directives.push(`stale-while-revalidate=${config.staleWhileRevalidate}`);
    }

    if (config.staleIfError !== undefined) {
      directives.push(`stale-if-error=${config.staleIfError}`);
    }

    if (config.strategy === 'immutable') {
      directives.push('immutable');
    }

    if (config.strategy === 'no-cache') {
      directives.length = 0;
      directives.push('no-cache', 'no-store', 'must-revalidate');
    }

    res.setHeader('Cache-Control', directives.join(', '));

    // Vary headers for cache differentiation
    if (config.varyHeaders && config.varyHeaders.length > 0) {
      res.setHeader('Vary', config.varyHeaders.join(', '));
    }

    // Add surrogate keys for targeted purging
    const surrogateKey = buildSurrogateKey(req);
    if (surrogateKey) {
      res.setHeader('Surrogate-Key', surrogateKey);
      res.setHeader('Cache-Tag', surrogateKey);
    }

    next();
  };
}

function buildSurrogateKey(req: Request): string {
  const keys: string[] = [];

  // Add resource-based keys
  const path = req.path;
  if (path.includes('/products/')) {
    const id = path.split('/products/')[1]?.split('/')[0];
    if (id) keys.push(`product-${id}`);
    keys.push('products');
  }

  if (path.includes('/categories/')) {
    const id = path.split('/categories/')[1]?.split('/')[0];
    if (id) keys.push(`category-${id}`);
  }

  // Add tenant key for multi-tenant apps
  const tenantId = (req as any).tenantId;
  if (tenantId) keys.push(`tenant-${tenantId}`);

  return keys.join(' ');
}
```

### 3.2 Cache Invalidation Service

```typescript
// src/cdn/cache-invalidation.ts
import axios from 'axios';
import crypto from 'crypto';
import { EventEmitter } from 'events';

interface CloudflareCacheConfig {
  zoneId: string;
  apiToken: string;
  baseUrl?: string;
}

interface FastlyCacheConfig {
  serviceId: string;
  apiToken: string;
}

interface PurgeResult {
  provider: string;
  success: boolean;
  purgedItems: string[];
  error?: string;
}

export class CDNCacheInvalidator extends EventEmitter {
  private cfConfig?: CloudflareCacheConfig;
  private fastlyConfig?: FastlyCacheConfig;

  constructor(
    cfConfig?: CloudflareCacheConfig,
    fastlyConfig?: FastlyCacheConfig
  ) {
    super();
    this.cfConfig = cfConfig;
    this.fastlyConfig = fastlyConfig;
  }

  async purgeByTag(tags: string[]): Promise<PurgeResult[]> {
    const results: PurgeResult[] = [];

    if (this.cfConfig) {
      results.push(await this.cloudflarePurgeByTag(tags));
    }

    if (this.fastlyConfig) {
      results.push(await this.fastlyPurgeByTag(tags));
    }

    this.emit('cache:purged', { tags, results });
    return results;
  }

  async purgeByUrl(urls: string[]): Promise<PurgeResult[]> {
    const results: PurgeResult[] = [];

    if (this.cfConfig) {
      results.push(await this.cloudflarePurgeByUrl(urls));
    }

    results.forEach((r) => {
      if (!r.success) {
        this.emit('cache:purge:error', r);
      }
    });

    return results;
  }

  private async cloudflarePurgeByTag(tags: string[]): Promise<PurgeResult> {
    const config = this.cfConfig!;
    try {
      await axios.post(
        `https://api.cloudflare.com/client/v4/zones/${config.zoneId}/purge_cache`,
        { tags },
        {
          headers: {
            Authorization: `Bearer ${config.apiToken}`,
            'Content-Type': 'application/json',
          },
        }
      );

      return { provider: 'cloudflare', success: true, purgedItems: tags };
    } catch (error: any) {
      return {
        provider: 'cloudflare',
        success: false,
        purgedItems: [],
        error: error.message,
      };
    }
  }

  private async cloudflarePurgeByUrl(urls: string[]): Promise<PurgeResult> {
    const config = this.cfConfig!;
    // Cloudflare allows max 30 URLs per request
    const chunks = this.chunkArray(urls, 30);
    
    for (const chunk of chunks) {
      await axios.post(
        `https://api.cloudflare.com/client/v4/zones/${config.zoneId}/purge_cache`,
        { files: chunk },
        {
          headers: {
            Authorization: `Bearer ${config.apiToken}`,
            'Content-Type': 'application/json',
          },
        }
      );
    }

    return { provider: 'cloudflare', success: true, purgedItems: urls };
  }

  private async fastlyPurgeByTag(tags: string[]): Promise<PurgeResult> {
    const config = this.fastlyConfig!;
    const results: string[] = [];

    for (const tag of tags) {
      await axios({
        method: 'PURGE',
        url: `https://api.fastly.com/service/${config.serviceId}/purge/${tag}`,
        headers: {
          'Fastly-Key': config.apiToken,
          'Accept': 'application/json',
        },
      });
      results.push(tag);
    }

    return { provider: 'fastly', success: true, purgedItems: results };
  }

  private chunkArray<T>(arr: T[], size: number): T[][] {
    return Array.from({ length: Math.ceil(arr.length / size) }, (_, i) =>
      arr.slice(i * size, i * size + size)
    );
  }
}

// Event-driven cache invalidation
export class CacheInvalidationHandler {
  constructor(
    private invalidator: CDNCacheInvalidator,
    private eventBus: EventEmitter
  ) {
    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    this.eventBus.on('product:updated', async (data: { id: string; categoryId: string }) => {
      const tags = [`product-${data.id}`, `category-${data.categoryId}`, 'products'];
      await this.invalidator.purgeByTag(tags);
    });

    this.eventBus.on('product:deleted', async (data: { id: string }) => {
      const tags = [`product-${data.id}`, 'products'];
      const urls = [
        `https://www.company.com/products/${data.id}`,
        `https://api.company.com/v1/products/${data.id}`,
      ];
      await Promise.all([
        this.invalidator.purgeByTag(tags),
        this.invalidator.purgeByUrl(urls),
      ]);
    });

    this.eventBus.on('inventory:changed', async (data: { productId: string }) => {
      // Only purge product API cache, not CDN since inventory is not cached long
      await this.invalidator.purgeByTag([`product-${data.productId}`]);
    });

    this.eventBus.on('category:updated', async (data: { id: string }) => {
      const tags = [`category-${data.id}`, 'categories', 'products'];
      await this.invalidator.purgeByTag(tags);
    });
  }
}
```

---

## 4. Geographic Routing

### 4.1 GeoDNS-Based Routing Service

```typescript
// src/routing/geo-router.ts
import geoip from 'geoip-lite';
import { Request, Response, NextFunction } from 'express';

interface RegionConfig {
  name: string;
  countries: string[];
  primaryEndpoint: string;
  fallbackEndpoint: string;
  latencyThresholdMs: number;
}

interface RoutingDecision {
  region: string;
  endpoint: string;
  country: string;
  latency?: number;
  reason: string;
}

const REGIONS: RegionConfig[] = [
  {
    name: 'asia-southeast',
    countries: ['TH', 'SG', 'MY', 'ID', 'PH', 'VN', 'MM', 'KH', 'LA'],
    primaryEndpoint: 'https://api-sg.company.com',
    fallbackEndpoint: 'https://api-us.company.com',
    latencyThresholdMs: 100,
  },
  {
    name: 'asia-east',
    countries: ['JP', 'KR', 'CN', 'HK', 'TW'],
    primaryEndpoint: 'https://api-jp.company.com',
    fallbackEndpoint: 'https://api-sg.company.com',
    latencyThresholdMs: 100,
  },
  {
    name: 'europe',
    countries: ['GB', 'DE', 'FR', 'NL', 'SE', 'NO', 'DK', 'FI', 'PL', 'IT', 'ES'],
    primaryEndpoint: 'https://api-eu.company.com',
    fallbackEndpoint: 'https://api-us.company.com',
    latencyThresholdMs: 150,
  },
  {
    name: 'north-america',
    countries: ['US', 'CA', 'MX'],
    primaryEndpoint: 'https://api-us.company.com',
    fallbackEndpoint: 'https://api-eu.company.com',
    latencyThresholdMs: 100,
  },
];

export class GeoRouter {
  private endpointHealth: Map<string, boolean> = new Map();
  private endpointLatency: Map<string, number> = new Map();

  async route(request: Request): Promise<RoutingDecision> {
    const ip = this.extractIp(request);
    const country = this.getCountry(ip);
    const region = this.findRegion(country);

    if (!region) {
      return {
        region: 'default',
        endpoint: 'https://api-us.company.com',
        country,
        reason: 'no_region_match',
      };
    }

    // Check primary endpoint health
    const primaryHealthy = await this.isEndpointHealthy(region.primaryEndpoint);
    const endpoint = primaryHealthy ? region.primaryEndpoint : region.fallbackEndpoint;

    return {
      region: region.name,
      endpoint,
      country,
      latency: this.endpointLatency.get(endpoint),
      reason: primaryHealthy ? 'primary' : 'fallback',
    };
  }

  private extractIp(request: Request): string {
    return (
      request.headers['cf-connecting-ip'] as string ||
      request.headers['x-forwarded-for']?.toString().split(',')[0]?.trim() ||
      request.socket.remoteAddress ||
      '0.0.0.0'
    );
  }

  private getCountry(ip: string): string {
    const geo = geoip.lookup(ip);
    return geo?.country || 'US';
  }

  private findRegion(country: string): RegionConfig | undefined {
    return REGIONS.find((r) => r.countries.includes(country));
  }

  private async isEndpointHealthy(endpoint: string): Promise<boolean> {
    const cached = this.endpointHealth.get(endpoint);
    if (cached !== undefined) return cached;

    try {
      const start = Date.now();
      const response = await fetch(`${endpoint}/health`, {
        signal: AbortSignal.timeout(5000),
      });
      const latency = Date.now() - start;

      this.endpointLatency.set(endpoint, latency);
      const healthy = response.ok;
      this.endpointHealth.set(endpoint, healthy);

      // Refresh health every 30 seconds
      setTimeout(() => this.endpointHealth.delete(endpoint), 30000);

      return healthy;
    } catch {
      this.endpointHealth.set(endpoint, false);
      setTimeout(() => this.endpointHealth.delete(endpoint), 10000);
      return false;
    }
  }
}

export function geoRoutingMiddleware(geoRouter: GeoRouter) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const decision = await geoRouter.route(req);

    req.headers['X-Routed-Region'] = decision.region;
    req.headers['X-Origin-Endpoint'] = decision.endpoint;
    req.headers['X-Client-Country'] = decision.country;

    res.setHeader('X-Served-By-Region', decision.region);
    res.setHeader('X-Client-Country', decision.country);

    next();
  };
}
```

---

## 5. Latency Optimization

### 5.1 HTTP/2 Server Push และ Preloading

```typescript
// src/optimization/http2-push.ts
import http2 from 'http2';
import fs from 'fs';
import path from 'path';

interface PushMapping {
  trigger: string;        // URL that triggers push
  resources: PushResource[];
}

interface PushResource {
  path: string;
  type: 'script' | 'style' | 'image' | 'font';
  crossOrigin?: boolean;
}

const PUSH_MAPPINGS: PushMapping[] = [
  {
    trigger: '/',
    resources: [
      { path: '/static/css/main.css', type: 'style' },
      { path: '/static/js/app.js', type: 'script' },
      { path: '/static/fonts/inter.woff2', type: 'font', crossOrigin: true },
    ],
  },
  {
    trigger: '/checkout',
    resources: [
      { path: '/static/js/payment.js', type: 'script' },
      { path: '/static/css/checkout.css', type: 'style' },
    ],
  },
];

export class HTTP2PushServer {
  private server: http2.Http2SecureServer;

  constructor(certPath: string, keyPath: string) {
    this.server = http2.createSecureServer({
      cert: fs.readFileSync(certPath),
      key: fs.readFileSync(keyPath),
      allowHTTP1: true,
    });

    this.setupHandlers();
  }

  private setupHandlers(): void {
    this.server.on('stream', (stream, headers) => {
      const url = headers[':path'] || '/';
      const method = headers[':method'] || 'GET';

      // Push resources for GET requests
      if (method === 'GET' && stream.pushAllowed) {
        const mapping = PUSH_MAPPINGS.find((m) => url === m.trigger);
        if (mapping) {
          this.pushResources(stream, mapping.resources);
        }
      }

      this.handleRequest(stream, url, method);
    });
  }

  private pushResources(stream: http2.ServerHttp2Stream, resources: PushResource[]): void {
    for (const resource of resources) {
      const filePath = path.join(process.cwd(), 'public', resource.path);
      if (!fs.existsSync(filePath)) continue;

      const contentType = this.getContentType(resource.type);
      const stats = fs.statSync(filePath);

      stream.pushStream(
        { ':path': resource.path },
        (err, pushStream) => {
          if (err) return;

          pushStream.respond({
            ':status': 200,
            'content-type': contentType,
            'content-length': stats.size.toString(),
            'cache-control': 'public, max-age=31536000, immutable',
          });

          fs.createReadStream(filePath).pipe(pushStream);
        }
      );
    }
  }

  private handleRequest(stream: http2.ServerHttp2Stream, url: string, method: string): void {
    // Handle request normally
    stream.respond({
      ':status': 200,
      'content-type': 'text/html',
    });
    stream.end('<html>...</html>');
  }

  private getContentType(type: PushResource['type']): string {
    const types: Record<string, string> = {
      script: 'application/javascript',
      style: 'text/css',
      image: 'image/webp',
      font: 'font/woff2',
    };
    return types[type] || 'application/octet-stream';
  }

  listen(port: number): void {
    this.server.listen(port, () => {
      console.log(`HTTP/2 server listening on port ${port}`);
    });
  }
}
```

### 5.2 Response Compression Middleware

```typescript
// src/middleware/compression.ts
import { Request, Response, NextFunction } from 'express';
import zlib from 'zlib';
import { Transform } from 'stream';

interface CompressionOptions {
  threshold: number;       // bytes
  level: number;           // zlib compression level
  memLevel: number;
  enableBrotli: boolean;
}

export function advancedCompression(options: Partial<CompressionOptions> = {}) {
  const config: CompressionOptions = {
    threshold: 1024,       // 1KB
    level: 6,
    memLevel: 8,
    enableBrotli: true,
    ...options,
  };

  return (req: Request, res: Response, next: NextFunction) => {
    const acceptEncoding = req.headers['accept-encoding'] || '';
    const originalWrite = res.write.bind(res);
    const originalEnd = res.end.bind(res);
    const chunks: Buffer[] = [];
    let compressor: Transform | null = null;

    // Determine best compression
    if (config.enableBrotli && acceptEncoding.includes('br')) {
      compressor = zlib.createBrotliCompress({
        params: {
          [zlib.constants.BROTLI_PARAM_QUALITY]: config.level,
          [zlib.constants.BROTLI_PARAM_MODE]: zlib.constants.BROTLI_MODE_TEXT,
        },
      });
      res.setHeader('Content-Encoding', 'br');
    } else if (acceptEncoding.includes('gzip')) {
      compressor = zlib.createGzip({
        level: config.level,
        memLevel: config.memLevel,
      });
      res.setHeader('Content-Encoding', 'gzip');
    } else if (acceptEncoding.includes('deflate')) {
      compressor = zlib.createDeflate({ level: config.level });
      res.setHeader('Content-Encoding', 'deflate');
    }

    if (!compressor) {
      return next();
    }

    res.removeHeader('Content-Length');
    res.setHeader('Vary', 'Accept-Encoding');

    (res as any).write = function (
      chunk: any,
      encodingOrCallback?: BufferEncoding | ((err: Error | null) => void),
      callback?: (err: Error | null) => void
    ): boolean {
      if (typeof encodingOrCallback === 'function') {
        chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk));
        return true;
      }
      chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk, encodingOrCallback));
      return true;
    };

    (res as any).end = function (
      chunk?: any,
      encodingOrCallback?: BufferEncoding | (() => void),
      callback?: () => void
    ): Response {
      if (chunk) {
        const encoding = typeof encodingOrCallback === 'string' ? encodingOrCallback : undefined;
        chunks.push(Buffer.isBuffer(chunk) ? chunk : Buffer.from(chunk, encoding));
      }

      const body = Buffer.concat(chunks);
      
      if (body.length < config.threshold) {
        res.removeHeader('Content-Encoding');
        res.setHeader('Content-Length', body.length);
        originalWrite(body);
        originalEnd();
        return res;
      }

      compressor!.on('data', (data: Buffer) => originalWrite(data));
      compressor!.on('end', () => originalEnd());
      compressor!.write(body);
      compressor!.end();
      return res;
    };

    next();
  };
}
```

---

## 6. Edge-Side Rendering (ESR)

### 6.1 Edge Rendering Worker

```typescript
// workers/edge-renderer.ts
import { Router } from 'itty-router';

interface Env {
  API_ORIGIN: string;
  ASSETS: Fetcher;
  CACHE: KVNamespace;
}

const router = Router();

// Edge-rendered product page
router.get('/products/:id', async (request: Request, env: Env) => {
  const { id } = (request as any).params;
  const cacheKey = `rendered:product:${id}:${request.headers.get('Accept-Language') || 'en'}`;

  // Check render cache
  const cachedHtml = await env.CACHE.get(cacheKey);
  if (cachedHtml) {
    return new Response(cachedHtml, {
      headers: {
        'Content-Type': 'text/html; charset=utf-8',
        'X-Edge-Cache': 'HIT',
        'Cache-Control': 'public, max-age=60, stale-while-revalidate=300',
      },
    });
  }

  // Fetch data in parallel
  const [productData, relatedProducts, reviews] = await Promise.all([
    fetchWithTimeout(`${env.API_ORIGIN}/api/products/${id}`, 3000),
    fetchWithTimeout(`${env.API_ORIGIN}/api/products/${id}/related?limit=6`, 3000),
    fetchWithTimeout(`${env.API_ORIGIN}/api/products/${id}/reviews?page=1&limit=5`, 3000),
  ]);

  if (!productData.ok) {
    return new Response('Product not found', { status: 404 });
  }

  const product = await productData.json() as any;
  const related = await relatedProducts.json() as any;
  const reviewData = await reviews.json() as any;

  // Render HTML at edge
  const html = renderProductPage(product, related.items, reviewData.reviews);

  // Cache for 60 seconds
  await env.CACHE.put(cacheKey, html, { expirationTtl: 60 });

  return new Response(html, {
    headers: {
      'Content-Type': 'text/html; charset=utf-8',
      'X-Edge-Cache': 'MISS',
      'Cache-Control': 'public, max-age=60, stale-while-revalidate=300',
    },
  });
});

function renderProductPage(product: any, related: any[], reviews: any[]): string {
  const relatedHtml = related
    .map((p) => `<div class="related-product">
      <img src="${p.image}" alt="${p.title}" loading="lazy" width="200" height="200">
      <h3>${p.title}</h3>
      <span class="price">${p.price}</span>
    </div>`)
    .join('');

  const reviewsHtml = reviews
    .map((r) => `<div class="review">
      <span class="stars">${'★'.repeat(r.rating)}</span>
      <p>${escapeHtml(r.comment)}</p>
      <small>${r.author}</small>
    </div>`)
    .join('');

  return `<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>${escapeHtml(product.title)} - Company Store</title>
  <meta name="description" content="${escapeHtml(product.description?.substring(0, 160))}">
  <link rel="preload" href="/static/css/main.css" as="style">
  <link rel="stylesheet" href="/static/css/main.css">
  <!-- Structured data for SEO -->
  <script type="application/ld+json">
  ${JSON.stringify({
    "@context": "https://schema.org/",
    "@type": "Product",
    "name": product.title,
    "description": product.description,
    "image": product.images,
    "offers": {
      "@type": "Offer",
      "priceCurrency": "THB",
      "price": product.price,
      "availability": product.inStock
        ? "https://schema.org/InStock"
        : "https://schema.org/OutOfStock",
    },
  })}
  </script>
</head>
<body>
  <main>
    <article itemscope itemtype="https://schema.org/Product">
      <h1 itemprop="name">${escapeHtml(product.title)}</h1>
      <img src="${product.image}" alt="${escapeHtml(product.title)}" itemprop="image">
      <p itemprop="description">${escapeHtml(product.description)}</p>
      <span class="price" itemprop="price">${product.price} THB</span>
      <button data-product-id="${product.id}" class="add-to-cart">เพิ่มในตะกร้า</button>
    </article>
    
    <section class="related-products">
      <h2>สินค้าที่เกี่ยวข้อง</h2>
      ${relatedHtml}
    </section>
    
    <section class="reviews">
      <h2>รีวิว (${reviews.length})</h2>
      ${reviewsHtml}
    </section>
  </main>
  <script src="/static/js/app.js" defer></script>
</body>
</html>`;
}

function escapeHtml(str: string = ''): string {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

async function fetchWithTimeout(url: string, timeoutMs: number): Promise<Response> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), timeoutMs);
  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(timeout);
  }
}

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    return router.handle(request, env);
  },
};
```

---

## 7. Purge Cache Automation

### 7.1 Automated Cache Purge Pipeline

```typescript
// src/cdn/automated-purge.ts
import { CDNCacheInvalidator } from './cache-invalidation';
import { Kafka } from 'kafkajs';
import { Redis } from 'ioredis';

interface PurgeEvent {
  eventType: 'product.updated' | 'product.deleted' | 'category.updated' | 'price.changed';
  resourceId: string;
  resourceType: string;
  metadata?: Record<string, any>;
  timestamp: string;
}

interface PurgeBatch {
  tags: string[];
  urls: string[];
  reason: string;
  priority: 'high' | 'medium' | 'low';
}

export class AutomatedCachePurgeService {
  private kafka: Kafka;
  private redis: Redis;
  private pendingPurges: Map<string, PurgeBatch> = new Map();
  private batchInterval: NodeJS.Timer;

  constructor(
    private invalidator: CDNCacheInvalidator,
    kafkaBrokers: string[],
    redis: Redis
  ) {
    this.kafka = new Kafka({
      clientId: 'cache-purge-service',
      brokers: kafkaBrokers,
    });
    this.redis = redis;
    
    // Batch purges every 2 seconds to reduce API calls
    this.batchInterval = setInterval(() => this.flushPurgeBatch(), 2000);
  }

  async startConsuming(): Promise<void> {
    const consumer = this.kafka.consumer({ groupId: 'cache-purge-group' });
    await consumer.connect();
    await consumer.subscribe({
      topics: ['product.events', 'category.events', 'pricing.events'],
      fromBeginning: false,
    });

    await consumer.run({
      eachMessage: async ({ topic, message }) => {
        const event: PurgeEvent = JSON.parse(message.value?.toString() || '{}');
        await this.processPurgeEvent(event);
      },
    });
  }

  private async processPurgeEvent(event: PurgeEvent): Promise<void> {
    const purgeBatch = this.buildPurgeBatch(event);
    
    if (purgeBatch.priority === 'high') {
      // Purge immediately for high priority
      await this.executePurge(purgeBatch);
    } else {
      // Batch low/medium priority purges
      const key = `${event.resourceType}:${event.resourceId}`;
      const existing = this.pendingPurges.get(key);
      
      if (existing) {
        existing.tags = [...new Set([...existing.tags, ...purgeBatch.tags])];
        existing.urls = [...new Set([...existing.urls, ...purgeBatch.urls])];
      } else {
        this.pendingPurges.set(key, purgeBatch);
      }
    }
  }

  private buildPurgeBatch(event: PurgeEvent): PurgeBatch {
    const batch: PurgeBatch = {
      tags: [],
      urls: [],
      reason: event.eventType,
      priority: 'medium',
    };

    switch (event.eventType) {
      case 'product.updated':
        batch.tags = [`product-${event.resourceId}`, 'products'];
        batch.urls = [
          `https://www.company.com/products/${event.resourceId}`,
          `https://api.company.com/v1/products/${event.resourceId}`,
        ];
        batch.priority = 'medium';
        break;

      case 'product.deleted':
        batch.tags = [`product-${event.resourceId}`, 'products'];
        batch.urls = [`https://www.company.com/products/${event.resourceId}`];
        batch.priority = 'high'; // Immediate purge for deleted items
        break;

      case 'price.changed':
        batch.tags = [`product-${event.resourceId}`];
        batch.urls = [
          `https://api.company.com/v1/products/${event.resourceId}`,
          `https://api.company.com/v1/products/${event.resourceId}/price`,
        ];
        batch.priority = 'high'; // Price changes need immediate purge
        break;

      case 'category.updated':
        batch.tags = [
          `category-${event.resourceId}`,
          'categories',
          'navigation',
        ];
        batch.priority = 'low';
        break;
    }

    return batch;
  }

  private async flushPurgeBatch(): Promise<void> {
    if (this.pendingPurges.size === 0) return;

    const allTags = new Set<string>();
    const allUrls = new Set<string>();

    for (const batch of this.pendingPurges.values()) {
      batch.tags.forEach((t) => allTags.add(t));
      batch.urls.forEach((u) => allUrls.add(u));
    }

    this.pendingPurges.clear();

    await this.executePurge({
      tags: Array.from(allTags),
      urls: Array.from(allUrls),
      reason: 'batch_flush',
      priority: 'medium',
    });
  }

  private async executePurge(batch: PurgeBatch): Promise<void> {
    const promises: Promise<any>[] = [];

    if (batch.tags.length > 0) {
      promises.push(this.invalidator.purgeByTag(batch.tags));
    }

    if (batch.urls.length > 0) {
      promises.push(this.invalidator.purgeByUrl(batch.urls));
    }

    const results = await Promise.allSettled(promises);
    
    // Log results to Redis for monitoring
    const logEntry = {
      timestamp: new Date().toISOString(),
      tags: batch.tags,
      urls: batch.urls,
      reason: batch.reason,
      results: results.map((r) => r.status),
    };

    await this.redis.lpush('cache:purge:log', JSON.stringify(logEntry));
    await this.redis.ltrim('cache:purge:log', 0, 999); // Keep last 1000 entries
  }
}
```

---

## 8. Real User Monitoring (RUM)

### 8.1 RUM Collector Service

```typescript
// src/monitoring/rum-collector.ts
import { Router } from 'express';
import { Kafka } from 'kafkajs';
import Joi from 'joi';

interface RUMEvent {
  type: 'navigation' | 'resource' | 'paint' | 'interaction' | 'error';
  url: string;
  userAgent: string;
  country?: string;
  sessionId: string;
  userId?: string;
  metrics: {
    // Navigation Timing
    ttfb?: number;          // Time to First Byte
    fcp?: number;           // First Contentful Paint
    lcp?: number;           // Largest Contentful Paint
    fid?: number;           // First Input Delay
    cls?: number;           // Cumulative Layout Shift
    tbt?: number;           // Total Blocking Time
    tti?: number;           // Time to Interactive
    // Resource Timing
    resourceUrl?: string;
    resourceType?: string;
    duration?: number;
    transferSize?: number;
    // Error
    errorMessage?: string;
    errorStack?: string;
  };
  timestamp: number;
  deviceType: 'mobile' | 'tablet' | 'desktop';
  connectionType?: string;
}

const rumSchema = Joi.object<RUMEvent>({
  type: Joi.string().valid('navigation', 'resource', 'paint', 'interaction', 'error').required(),
  url: Joi.string().uri().required(),
  userAgent: Joi.string().max(500).required(),
  sessionId: Joi.string().max(100).required(),
  userId: Joi.string().max(100).optional(),
  country: Joi.string().max(10).optional(),
  metrics: Joi.object().required(),
  timestamp: Joi.number().required(),
  deviceType: Joi.string().valid('mobile', 'tablet', 'desktop').required(),
  connectionType: Joi.string().optional(),
});

export class RUMCollector {
  private router: Router;
  private kafka: Kafka;

  constructor(kafkaBrokers: string[]) {
    this.router = Router();
    this.kafka = new Kafka({
      clientId: 'rum-collector',
      brokers: kafkaBrokers,
    });
    this.setupRoutes();
  }

  private setupRoutes(): void {
    this.router.post('/rum/events', async (req, res) => {
      const { error, value } = rumSchema.validate(req.body);
      if (error) {
        return res.status(400).json({ error: error.message });
      }

      // Enrich with server-side data
      const enrichedEvent = {
        ...value,
        country: req.headers['cf-ipcountry'] as string || value.country,
        ip: req.headers['cf-connecting-ip'] as string || req.ip,
        receivedAt: Date.now(),
        // Classify performance
        performanceScore: this.calculatePerformanceScore(value.metrics),
        isGoodExperience: this.isGoodExperience(value.metrics),
      };

      await this.publishToKafka(enrichedEvent);
      res.status(202).json({ status: 'accepted' });
    });

    // Batch endpoint for performance
    this.router.post('/rum/events/batch', async (req, res) => {
      const events = req.body.events as RUMEvent[];
      if (!Array.isArray(events) || events.length > 50) {
        return res.status(400).json({ error: 'Invalid batch' });
      }

      const validEvents = events
        .map((e) => rumSchema.validate(e))
        .filter((r) => !r.error)
        .map((r) => r.value!);

      await Promise.all(validEvents.map((e) => this.publishToKafka(e)));
      res.status(202).json({ accepted: validEvents.length, rejected: events.length - validEvents.length });
    });
  }

  private calculatePerformanceScore(metrics: RUMEvent['metrics']): number {
    // Google Web Vitals scoring
    let score = 100;

    if (metrics.lcp) {
      if (metrics.lcp > 4000) score -= 30;
      else if (metrics.lcp > 2500) score -= 15;
    }

    if (metrics.fid) {
      if (metrics.fid > 300) score -= 25;
      else if (metrics.fid > 100) score -= 10;
    }

    if (metrics.cls) {
      if (metrics.cls > 0.25) score -= 25;
      else if (metrics.cls > 0.1) score -= 10;
    }

    if (metrics.ttfb) {
      if (metrics.ttfb > 1800) score -= 20;
      else if (metrics.ttfb > 800) score -= 10;
    }

    return Math.max(0, score);
  }

  private isGoodExperience(metrics: RUMEvent['metrics']): boolean {
    return (
      (metrics.lcp === undefined || metrics.lcp <= 2500) &&
      (metrics.fid === undefined || metrics.fid <= 100) &&
      (metrics.cls === undefined || metrics.cls <= 0.1)
    );
  }

  private async publishToKafka(event: any): Promise<void> {
    const producer = this.kafka.producer();
    await producer.connect();
    await producer.send({
      topic: 'rum.events',
      messages: [{ value: JSON.stringify(event) }],
    });
    await producer.disconnect();
  }

  getRouter(): Router {
    return this.router;
  }
}
```

### 8.2 RUM Analytics Dashboard Script

```typescript
// public/rum-tracker.js (client-side)
(function() {
  'use strict';

  const SESSION_ID = getOrCreateSessionId();
  const COLLECTOR_URL = '/rum/events/batch';
  let pendingEvents = [];
  let flushTimer = null;

  function getOrCreateSessionId() {
    let id = sessionStorage.getItem('rum_session_id');
    if (!id) {
      id = 'sess_' + Math.random().toString(36).substr(2, 9) + Date.now().toString(36);
      sessionStorage.setItem('rum_session_id', id);
    }
    return id;
  }

  function getDeviceType() {
    const ua = navigator.userAgent;
    if (/Mobi|Android/i.test(ua)) return 'mobile';
    if (/Tablet|iPad/i.test(ua)) return 'tablet';
    return 'desktop';
  }

  function queueEvent(event) {
    pendingEvents.push({
      ...event,
      sessionId: SESSION_ID,
      url: location.href,
      userAgent: navigator.userAgent,
      deviceType: getDeviceType(),
      timestamp: Date.now(),
      connectionType: navigator.connection?.effectiveType,
    });

    if (!flushTimer) {
      flushTimer = setTimeout(flush, 3000);
    }

    if (pendingEvents.length >= 10) {
      flush();
    }
  }

  function flush() {
    if (flushTimer) {
      clearTimeout(flushTimer);
      flushTimer = null;
    }

    if (pendingEvents.length === 0) return;

    const events = pendingEvents.splice(0);
    const payload = JSON.stringify({ events });

    if (navigator.sendBeacon) {
      navigator.sendBeacon(COLLECTOR_URL, new Blob([payload], { type: 'application/json' }));
    } else {
      fetch(COLLECTOR_URL, { method: 'POST', body: payload, keepalive: true });
    }
  }

  // Navigation Timing
  window.addEventListener('load', function() {
    requestAnimationFrame(function() {
      const timing = performance.getEntriesByType('navigation')[0];
      if (timing) {
        queueEvent({
          type: 'navigation',
          metrics: {
            ttfb: timing.responseStart - timing.requestStart,
            fcp: performance.getEntriesByName('first-contentful-paint')[0]?.startTime,
            tti: timing.domInteractive,
          },
        });
      }
    });
  });

  // Web Vitals via PerformanceObserver
  if ('PerformanceObserver' in window) {
    // LCP
    new PerformanceObserver(function(list) {
      const entry = list.getEntries().at(-1);
      if (entry) queueEvent({ type: 'paint', metrics: { lcp: entry.startTime } });
    }).observe({ type: 'largest-contentful-paint', buffered: true });

    // FID
    new PerformanceObserver(function(list) {
      for (const entry of list.getEntries()) {
        queueEvent({ type: 'interaction', metrics: { fid: entry.processingStart - entry.startTime } });
      }
    }).observe({ type: 'first-input', buffered: true });

    // CLS
    let clsValue = 0;
    new PerformanceObserver(function(list) {
      for (const entry of list.getEntries()) {
        if (!entry.hadRecentInput) clsValue += entry.value;
      }
      queueEvent({ type: 'paint', metrics: { cls: clsValue } });
    }).observe({ type: 'layout-shift', buffered: true });
  }

  // Flush on page hide
  document.addEventListener('visibilitychange', function() {
    if (document.visibilityState === 'hidden') flush();
  });

  window.addEventListener('pagehide', flush);
})();
```

---

## สรุป

บทนี้ครอบคลุม Edge Computing และ CDN Integration อย่างครบถ้วน:

1. **Cloudflare Workers** - API Gateway ที่ Edge พร้อม Rate Limiting, Auth, และ Caching
2. **Edge Middleware** - Next.js Edge Middleware สำหรับ A/B Testing และ Geographic Routing
3. **CDN Cache Strategies** - Cache-Control headers, Surrogate Keys, และ Stale-While-Revalidate
4. **Cache Invalidation** - Event-driven cache purge ผ่าน Kafka
5. **Geographic Routing** - Routing อัตโนมัติตาม geolocation เพื่อ latency ต่ำสุด
6. **HTTP/2 Push** - Server Push สำหรับ critical assets
7. **Edge-Side Rendering** - Render HTML ที่ edge เพื่อ SEO และ performance
8. **Automated Purge** - Batch cache purge pipeline
9. **Real User Monitoring** - เก็บ Web Vitals จริงจาก users

Key Takeaways:
- Edge functions ช่วยลด latency ได้อย่างมากโดยไม่ต้องเปลี่ยน origin
- Cache invalidation ต้องเป็น event-driven เพื่อความถูกต้องของข้อมูล
- Geographic routing ช่วยให้ users ได้รับบริการจาก region ที่ใกล้ที่สุด
- RUM ให้ข้อมูล performance จริงที่ lab tests ไม่สามารถวัดได้
- Batch purge ลด CDN API calls โดยไม่กระทบ cache freshness

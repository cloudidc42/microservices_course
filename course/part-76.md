# Part 76: Edge Computing and CDN

## บทนำ

Edge Computing นำการประมวลผลเข้าใกล้ผู้ใช้มากขึ้น แทนที่จะส่งทุก request ไปยัง central server บทนี้ครอบคลุม Cloudflare Workers, AWS Lambda@Edge, CDN strategies, และ edge patterns สำหรับ Microservices

---

## 1. Edge Computing Concepts

### แนวคิดหลัก

```typescript
// edge-concepts.ts
// อธิบาย layers ของ computing

interface ComputingTier {
  name: string;
  location: string;
  latencyToUser: string;
  capabilities: string[];
  limitations: string[];
  useCases: string[];
}

const computingTiers: ComputingTier[] = [
  {
    name: 'Central Cloud',
    location: 'Data center (1-3 locations)',
    latencyToUser: '50-200ms',
    capabilities: [
      'Full compute resources',
      'Large storage',
      'Complex processing',
      'All databases',
    ],
    limitations: ['High latency for global users'],
    useCases: ['Business logic', 'Data processing', 'ML training'],
  },
  {
    name: 'Regional Edge',
    location: 'Multiple regions (10-30 locations)',
    latencyToUser: '10-50ms',
    capabilities: [
      'Moderate compute',
      'Caching',
      'Request routing',
    ],
    limitations: ['Limited storage', 'Higher cost per region'],
    useCases: ['API aggregation', 'Regional caching', 'Compliance'],
  },
  {
    name: 'Edge (CDN PoPs)',
    location: 'Global (200+ locations)',
    latencyToUser: '1-10ms',
    capabilities: [
      'Request/response manipulation',
      'Caching',
      'Authentication',
      'A/B testing',
      'Personalization',
    ],
    limitations: [
      'Limited CPU/memory',
      'No persistent storage (usually)',
      'Cold start time',
    ],
    useCases: [
      'Static asset delivery',
      'Authentication',
      'Traffic shaping',
      'Security (WAF)',
    ],
  },
  {
    name: 'IoT Edge',
    location: 'On-device / Local network',
    latencyToUser: '<1ms',
    capabilities: ['Local processing', 'Offline operation'],
    limitations: ['Very limited resources', 'Complex management'],
    useCases: ['IoT sensors', 'Autonomous vehicles', 'Industrial automation'],
  },
];

// Edge Computing Decision Framework
function decideEdgeStrategy(
  requirement: {
    globalUsers: boolean;
    latencyRequirement: number; // ms
    personalizationNeeded: boolean;
    complianceRegion?: string;
    staticContent: boolean;
  }
): string {
  const recommendations: string[] = [];

  if (requirement.staticContent) {
    recommendations.push('Use CDN with aggressive caching for static assets');
  }

  if (requirement.latencyRequirement < 50) {
    recommendations.push('Deploy edge functions for request processing');
  }

  if (requirement.personalizationNeeded) {
    recommendations.push('Use edge workers for personalization at PoP level');
  }

  if (requirement.complianceRegion) {
    recommendations.push(`Route ${requirement.complianceRegion} traffic to regional deployment`);
  }

  if (requirement.globalUsers && requirement.latencyRequirement < 100) {
    recommendations.push('Implement multi-region deployment with global load balancing');
  }

  return recommendations.join('\n');
}
```

---

## 2. Cloudflare Workers with TypeScript

### Setup Cloudflare Worker

```typescript
// workers/order-edge-worker.ts
// Cloudflare Worker สำหรับ Edge processing

export interface Env {
  ORDER_SERVICE_URL: string;
  AUTH_SECRET: string;
  CACHE: KVNamespace;
  ORDER_QUEUE: Queue;
}

interface RequestContext {
  userId: string;
  region: string;
  deviceType: 'mobile' | 'tablet' | 'desktop';
  preferredLanguage: string;
}

// Main worker handler
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);

    // Rate limiting check
    const rateLimitResult = await checkRateLimit(request, env);
    if (!rateLimitResult.allowed) {
      return new Response('Rate limit exceeded', {
        status: 429,
        headers: {
          'Retry-After': String(rateLimitResult.retryAfter),
          'X-RateLimit-Limit': '100',
          'X-RateLimit-Remaining': '0',
        },
      });
    }

    // Authentication at edge
    const authResult = await authenticateRequest(request, env);
    if (!authResult.valid) {
      return new Response('Unauthorized', { status: 401 });
    }

    // Route handling
    if (url.pathname.startsWith('/api/products')) {
      return handleProductsRequest(request, env, authResult.context!);
    }

    if (url.pathname.startsWith('/api/orders')) {
      return handleOrdersRequest(request, env, authResult.context!);
    }

    if (url.pathname === '/health') {
      return new Response(JSON.stringify({ status: 'healthy', edge: true }), {
        headers: { 'Content-Type': 'application/json' },
      });
    }

    return new Response('Not Found', { status: 404 });
  },
};

// Rate limiting using Cloudflare KV
async function checkRateLimit(
  request: Request,
  env: Env
): Promise<{ allowed: boolean; retryAfter?: number }> {
  const ip = request.headers.get('CF-Connecting-IP') || 'unknown';
  const key = `rate-limit:${ip}`;
  const now = Math.floor(Date.now() / 1000);
  const windowStart = Math.floor(now / 60) * 60;

  const count = await env.CACHE.get(`${key}:${windowStart}`);
  const currentCount = count ? parseInt(count) : 0;

  if (currentCount >= 100) {
    return { allowed: false, retryAfter: 60 - (now - windowStart) };
  }

  await env.CACHE.put(
    `${key}:${windowStart}`,
    String(currentCount + 1),
    { expirationTtl: 120 }
  );

  return { allowed: true };
}

// JWT validation at edge
async function authenticateRequest(
  request: Request,
  env: Env
): Promise<{ valid: boolean; context?: RequestContext }> {
  const authHeader = request.headers.get('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return { valid: false };
  }

  const token = authHeader.substring(7);

  try {
    // Verify JWT using Web Crypto API (available in Workers)
    const [headerB64, payloadB64, signatureB64] = token.split('.');
    
    const encoder = new TextEncoder();
    const key = await crypto.subtle.importKey(
      'raw',
      encoder.encode(env.AUTH_SECRET),
      { name: 'HMAC', hash: 'SHA-256' },
      false,
      ['verify']
    );

    const isValid = await crypto.subtle.verify(
      'HMAC',
      key,
      base64ToArrayBuffer(signatureB64),
      encoder.encode(`${headerB64}.${payloadB64}`)
    );

    if (!isValid) return { valid: false };

    const payload = JSON.parse(atob(payloadB64));
    
    // Check expiration
    if (payload.exp < Math.floor(Date.now() / 1000)) {
      return { valid: false };
    }

    const context: RequestContext = {
      userId: payload.sub,
      region: request.cf?.country || 'unknown',
      deviceType: detectDeviceType(request.headers.get('User-Agent') || ''),
      preferredLanguage: request.headers.get('Accept-Language')?.split(',')[0] || 'en',
    };

    return { valid: true, context };
  } catch {
    return { valid: false };
  }
}

function base64ToArrayBuffer(base64: string): ArrayBuffer {
  const binary = atob(base64.replace(/-/g, '+').replace(/_/g, '/'));
  const bytes = new Uint8Array(binary.length);
  for (let i = 0; i < binary.length; i++) {
    bytes[i] = binary.charCodeAt(i);
  }
  return bytes.buffer;
}

function detectDeviceType(userAgent: string): 'mobile' | 'tablet' | 'desktop' {
  if (/Mobile/i.test(userAgent)) return 'mobile';
  if (/Tablet|iPad/i.test(userAgent)) return 'tablet';
  return 'desktop';
}

// Products handler with edge caching
async function handleProductsRequest(
  request: Request,
  env: Env,
  context: RequestContext
): Promise<Response> {
  const url = new URL(request.url);
  const cacheKey = `products:${url.pathname}:${url.search}`;
  
  // Check edge cache
  const cached = await env.CACHE.get(cacheKey, 'text');
  if (cached) {
    return new Response(cached, {
      headers: {
        'Content-Type': 'application/json',
        'X-Cache': 'HIT',
        'X-Edge-Location': 'edge',
      },
    });
  }

  // Fetch from origin
  const originResponse = await fetch(`${env.ORDER_SERVICE_URL}${url.pathname}${url.search}`, {
    headers: {
      'X-User-Id': context.userId,
      'X-Region': context.region,
    },
  });

  if (originResponse.ok) {
    const data = await originResponse.text();
    
    // Cache at edge for 60 seconds
    await env.CACHE.put(cacheKey, data, { expirationTtl: 60 });

    return new Response(data, {
      headers: {
        'Content-Type': 'application/json',
        'X-Cache': 'MISS',
      },
    });
  }

  return originResponse;
}

// Orders handler
async function handleOrdersRequest(
  request: Request,
  env: Env,
  context: RequestContext
): Promise<Response> {
  if (request.method === 'POST') {
    // Queue order creation to prevent overload
    const body = await request.json();
    
    await env.ORDER_QUEUE.send({
      userId: context.userId,
      orderData: body,
      timestamp: Date.now(),
      region: context.region,
    });

    return new Response(
      JSON.stringify({ status: 'QUEUED', message: 'Order is being processed' }),
      {
        status: 202,
        headers: { 'Content-Type': 'application/json' },
      }
    );
  }

  // For GET requests, proxy to origin
  return fetch(`${env.ORDER_SERVICE_URL}${new URL(request.url).pathname}`, {
    headers: {
      Authorization: request.headers.get('Authorization') || '',
      'X-User-Id': context.userId,
    },
  });
}
```

```toml
# wrangler.toml
name = "order-edge-worker"
main = "src/worker.ts"
compatibility_date = "2024-01-01"
compatibility_flags = ["nodejs_compat"]

[env.production]
vars = { ORDER_SERVICE_URL = "https://api.company.com" }

[[kv_namespaces]]
binding = "CACHE"
id = "your-kv-namespace-id"

[[queues.producers]]
binding = "ORDER_QUEUE"
queue = "order-processing-queue"

[[queues.consumers]]
queue = "order-processing-queue"
max_batch_size = 10
max_batch_timeout = 5
max_retries = 3

[triggers]
crons = ["*/5 * * * *"]  # Cache warming every 5 minutes
```

---

## 3. AWS Lambda@Edge

```typescript
// lambda-at-edge/viewer-request.ts
// Lambda@Edge function สำหรับ CloudFront

import { CloudFrontRequestEvent, CloudFrontRequestResult } from 'aws-lambda';

// Viewer Request: รัน ณ Edge ก่อน cache
export const handler = async (
  event: CloudFrontRequestEvent
): Promise<CloudFrontRequestResult> => {
  const request = event.Records[0].cf.request;
  const headers = request.headers;
  
  // A/B Testing at edge
  const abTestResult = performABTest(headers['cookie']?.[0]?.value || '');
  if (abTestResult.variant === 'B') {
    request.uri = request.uri.replace('/api/v1/', '/api/v2/');
  }

  // Geographic routing
  const countryCode = headers['cloudfront-viewer-country']?.[0]?.value;
  if (countryCode === 'TH' || countryCode === 'SG') {
    request.headers['x-route-to'] = [{ key: 'X-Route-To', value: 'asia-pacific' }];
  } else if (countryCode && ['US', 'CA', 'MX'].includes(countryCode)) {
    request.headers['x-route-to'] = [{ key: 'X-Route-To', value: 'us-east' }];
  }

  // Security headers
  const token = extractBearerToken(headers['authorization']?.[0]?.value || '');
  if (token) {
    const isValid = await validateTokenAtEdge(token);
    if (!isValid) {
      return {
        status: '401',
        statusDescription: 'Unauthorized',
        body: JSON.stringify({ error: 'Invalid token' }),
        headers: {
          'content-type': [{ key: 'Content-Type', value: 'application/json' }],
        },
      };
    }
  }

  // Normalize URL
  if (request.uri.endsWith('/') && request.uri !== '/') {
    request.uri = request.uri.slice(0, -1);
  }

  return request;
};

function performABTest(cookieHeader: string): { variant: 'A' | 'B'; userId?: string } {
  const abTestCookie = parseCookie(cookieHeader, 'ab_test');
  
  if (abTestCookie) {
    return { variant: abTestCookie as 'A' | 'B' };
  }

  // Randomly assign 20% to variant B
  const variant = Math.random() < 0.2 ? 'B' : 'A';
  return { variant };
}

function parseCookie(cookieHeader: string, name: string): string | null {
  const cookies = cookieHeader.split(';');
  for (const cookie of cookies) {
    const [key, value] = cookie.trim().split('=');
    if (key === name) return value;
  }
  return null;
}

function extractBearerToken(authHeader: string): string | null {
  if (authHeader.startsWith('Bearer ')) {
    return authHeader.substring(7);
  }
  return null;
}

async function validateTokenAtEdge(token: string): Promise<boolean> {
  // In Lambda@Edge, we can use cached validation
  // Simple signature check without external calls
  try {
    const parts = token.split('.');
    if (parts.length !== 3) return false;
    
    const payload = JSON.parse(
      Buffer.from(parts[1], 'base64').toString()
    );
    
    // Check expiration
    if (payload.exp < Math.floor(Date.now() / 1000)) return false;
    
    return true;
  } catch {
    return false;
  }
}
```

```typescript
// lambda-at-edge/origin-request.ts
// Origin Request: หลัง cache miss ก่อนส่งไป origin

import { CloudFrontRequestEvent, CloudFrontRequestResult } from 'aws-lambda';
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const secretsClient = new SecretsManagerClient({ region: 'us-east-1' });
let cachedApiKey: string | null = null;

export const handler = async (
  event: CloudFrontRequestEvent
): Promise<CloudFrontRequestResult> => {
  const request = event.Records[0].cf.request;

  // Add internal API key (fetch from Secrets Manager, cache locally)
  if (!cachedApiKey) {
    const response = await secretsClient.send(
      new GetSecretValueCommand({ SecretId: 'internal-api-key' })
    );
    cachedApiKey = response.SecretString || '';
  }

  request.headers['x-internal-api-key'] = [
    { key: 'X-Internal-API-Key', value: cachedApiKey },
  ];

  // Add trace ID
  const traceId = generateTraceId();
  request.headers['x-trace-id'] = [
    { key: 'X-Trace-ID', value: traceId },
  ];

  // Cache key customization based on accept-language
  const language = request.headers['accept-language']?.[0]?.value?.split(',')[0] || 'en';
  request.headers['x-language'] = [{ key: 'X-Language', value: language }];

  return request;
};

function generateTraceId(): string {
  return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```

```json
// lambda-at-edge/cloudformation.json - CloudFront Distribution
{
  "AWSTemplateFormatVersion": "2010-09-09",
  "Resources": {
    "CloudFrontDistribution": {
      "Type": "AWS::CloudFront::Distribution",
      "Properties": {
        "DistributionConfig": {
          "DefaultCacheBehavior": {
            "ViewerProtocolPolicy": "redirect-to-https",
            "TargetOriginId": "ApiOrigin",
            "ForwardedValues": {
              "QueryString": true,
              "Headers": ["Authorization", "Accept-Language", "Accept"]
            },
            "LambdaFunctionAssociations": [
              {
                "EventType": "viewer-request",
                "LambdaFunctionARN": { "Ref": "ViewerRequestFunctionVersion" },
                "IncludeBody": false
              },
              {
                "EventType": "origin-request",
                "LambdaFunctionARN": { "Ref": "OriginRequestFunctionVersion" }
              }
            ],
            "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
            "Compress": true
          },
          "Origins": [
            {
              "Id": "ApiOrigin",
              "DomainName": "api.company.com",
              "CustomOriginConfig": {
                "HTTPSPort": 443,
                "OriginProtocolPolicy": "https-only",
                "OriginSSLProtocols": ["TLSv1.2"]
              }
            }
          ],
          "Enabled": true,
          "HttpVersion": "http2and3",
          "PriceClass": "PriceClass_All",
          "ViewerCertificate": {
            "AcmCertificateArn": { "Ref": "Certificate" },
            "SslSupportMethod": "sni-only",
            "MinimumProtocolVersion": "TLSv1.2_2021"
          },
          "WebACLId": { "Ref": "WAFWebACL" }
        }
      }
    }
  }
}
```

---

## 4. CDN Strategy for Microservices

```typescript
// cdn-strategy.ts

interface CachePolicy {
  ttl: number; // seconds
  varyBy: string[];
  cacheControl: string;
  purgeOnUpdate: boolean;
  warmOnDeploy: boolean;
}

const cachePolicies: Record<string, CachePolicy> = {
  staticAssets: {
    ttl: 31536000, // 1 year
    varyBy: [],
    cacheControl: 'public, max-age=31536000, immutable',
    purgeOnUpdate: false,
    warmOnDeploy: false,
  },
  apiProducts: {
    ttl: 300, // 5 minutes
    varyBy: ['Accept-Language', 'Accept-Encoding'],
    cacheControl: 'public, max-age=300, stale-while-revalidate=60',
    purgeOnUpdate: true,
    warmOnDeploy: true,
  },
  userProfile: {
    ttl: 60, // 1 minute
    varyBy: ['Authorization'],
    cacheControl: 'private, max-age=60',
    purgeOnUpdate: true,
    warmOnDeploy: false,
  },
  search: {
    ttl: 30, // 30 seconds
    varyBy: ['Accept-Language'],
    cacheControl: 'public, max-age=30, stale-if-error=300',
    purgeOnUpdate: false,
    warmOnDeploy: false,
  },
  checkout: {
    ttl: 0, // No caching
    varyBy: [],
    cacheControl: 'no-store, no-cache',
    purgeOnUpdate: false,
    warmOnDeploy: false,
  },
};

// Cache warming strategy
class CDNCacheWarmer {
  private cdnApiUrl: string;
  private apiToken: string;

  constructor(cdnApiUrl: string, apiToken: string) {
    this.cdnApiUrl = cdnApiUrl;
    this.apiToken = apiToken;
  }

  async warmCache(urls: string[]): Promise<void> {
    const batchSize = 10;
    
    for (let i = 0; i < urls.length; i += batchSize) {
      const batch = urls.slice(i, i + batchSize);
      
      await Promise.all(
        batch.map(url => this.prefetchUrl(url))
      );
      
      // Rate limiting between batches
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  }

  private async prefetchUrl(url: string): Promise<void> {
    try {
      await fetch(url, {
        headers: {
          'X-Cache-Warm': 'true',
          Authorization: `Bearer ${this.apiToken}`,
        },
      });
      console.log(`Warmed cache: ${url}`);
    } catch (error) {
      console.error(`Failed to warm cache for ${url}:`, error);
    }
  }

  async purgeCache(tags: string[]): Promise<void> {
    const response = await fetch(`${this.cdnApiUrl}/purge`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${this.apiToken}`,
      },
      body: JSON.stringify({ tags }),
    });

    if (!response.ok) {
      throw new Error(`Cache purge failed: ${response.status}`);
    }

    console.log(`Purged cache tags: ${tags.join(', ')}`);
  }
}

// Smart cache key generation
class CacheKeyBuilder {
  private parts: string[] = [];

  static from(request: Request): CacheKeyBuilder {
    return new CacheKeyBuilder().withUrl(request.url);
  }

  withUrl(url: string): this {
    const parsed = new URL(url);
    this.parts.push(`${parsed.pathname}${parsed.search}`);
    return this;
  }

  withHeader(headerName: string, value: string): this {
    this.parts.push(`${headerName}:${value}`);
    return this;
  }

  withVaryHeaders(headers: Headers, varyList: string[]): this {
    for (const header of varyList) {
      const value = headers.get(header);
      if (value) {
        this.withHeader(header.toLowerCase(), value);
      }
    }
    return this;
  }

  withUserId(userId: string): this {
    // Hash user ID สำหรับ security
    this.parts.push(`user:${this.hashUserId(userId)}`);
    return this;
  }

  private hashUserId(userId: string): string {
    // Simple hash (use proper crypto in production)
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = ((hash << 5) - hash) + userId.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash).toString(36);
  }

  build(): string {
    return this.parts.join('|');
  }
}
```

---

## 5. Edge Caching Patterns

```typescript
// edge-caching-patterns.ts

// Stale-While-Revalidate pattern
class StaleWhileRevalidateCache {
  private cache: Map<string, {
    data: unknown;
    fetchedAt: number;
    ttl: number;
    revalidating: boolean;
  }> = new Map();

  async get<T>(
    key: string,
    fetcher: () => Promise<T>,
    options: { ttl: number; staleTime: number }
  ): Promise<T> {
    const cached = this.cache.get(key);
    const now = Date.now();

    if (cached) {
      const age = now - cached.fetchedAt;
      
      if (age < cached.ttl * 1000) {
        // Fresh - return immediately
        return cached.data as T;
      }
      
      if (age < (cached.ttl + options.staleTime) * 1000) {
        // Stale but within stale window - return stale, revalidate in background
        if (!cached.revalidating) {
          cached.revalidating = true;
          this.revalidate(key, fetcher, options.ttl).catch(console.error);
        }
        return cached.data as T;
      }
    }

    // Expired or not cached - fetch fresh
    const data = await fetcher();
    this.cache.set(key, {
      data,
      fetchedAt: now,
      ttl: options.ttl,
      revalidating: false,
    });
    
    return data;
  }

  private async revalidate<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttl: number
  ): Promise<void> {
    try {
      const data = await fetcher();
      this.cache.set(key, {
        data,
        fetchedAt: Date.now(),
        ttl,
        revalidating: false,
      });
    } catch (error) {
      console.error(`Revalidation failed for ${key}:`, error);
      // Keep stale data, clear revalidating flag
      const cached = this.cache.get(key);
      if (cached) {
        cached.revalidating = false;
      }
    }
  }
}

// Cache-Aside Pattern with Cloudflare KV
class EdgeCacheAside<T> {
  private namespace: KVNamespace;
  private defaultTTL: number;

  constructor(namespace: KVNamespace, defaultTTL: number = 300) {
    this.namespace = namespace;
    this.defaultTTL = defaultTTL;
  }

  async get(key: string): Promise<T | null> {
    const cached = await this.namespace.get(key, 'json');
    return cached as T | null;
  }

  async set(key: string, value: T, ttl?: number): Promise<void> {
    await this.namespace.put(
      key,
      JSON.stringify(value),
      { expirationTtl: ttl || this.defaultTTL }
    );
  }

  async getOrFetch(
    key: string,
    fetcher: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    const cached = await this.get(key);
    
    if (cached !== null) {
      return cached;
    }

    const fresh = await fetcher();
    await this.set(key, fresh, ttl);
    return fresh;
  }

  async invalidate(key: string): Promise<void> {
    await this.namespace.delete(key);
  }

  async invalidatePattern(prefix: string): Promise<void> {
    const list = await this.namespace.list({ prefix });
    
    await Promise.all(
      list.keys.map(key => this.namespace.delete(key.name))
    );
  }
}

// Cache Stampede Prevention
class StampedePreventionCache {
  private inFlight: Map<string, Promise<unknown>> = new Map();

  async getOrFetch<T>(
    key: string,
    fetcher: () => Promise<T>,
    cache: EdgeCacheAside<T>
  ): Promise<T> {
    // Check cache first
    const cached = await cache.get(key);
    if (cached !== null) return cached;

    // Check if another request is already fetching
    if (this.inFlight.has(key)) {
      return this.inFlight.get(key) as Promise<T>;
    }

    // Fetch and cache
    const fetchPromise = fetcher().then(async (data) => {
      await cache.set(key, data);
      this.inFlight.delete(key);
      return data;
    }).catch((error) => {
      this.inFlight.delete(key);
      throw error;
    });

    this.inFlight.set(key, fetchPromise);
    return fetchPromise as Promise<T>;
  }
}
```

---

## 6. Geographic Routing

```typescript
// geographic-routing.ts

interface Region {
  code: string;
  name: string;
  countries: string[];
  endpoint: string;
  healthCheckUrl: string;
  primary: boolean;
}

const regions: Region[] = [
  {
    code: 'apac',
    name: 'Asia Pacific',
    countries: ['TH', 'SG', 'MY', 'ID', 'PH', 'VN', 'JP', 'KR', 'CN', 'HK', 'TW', 'AU'],
    endpoint: 'https://api-apac.company.com',
    healthCheckUrl: 'https://api-apac.company.com/health',
    primary: false,
  },
  {
    code: 'eu',
    name: 'Europe',
    countries: ['GB', 'DE', 'FR', 'NL', 'IT', 'ES', 'SE', 'PL', 'NO', 'DK'],
    endpoint: 'https://api-eu.company.com',
    healthCheckUrl: 'https://api-eu.company.com/health',
    primary: false,
  },
  {
    code: 'us',
    name: 'United States',
    countries: ['US', 'CA', 'MX', 'BR', 'AR', 'CL'],
    endpoint: 'https://api-us.company.com',
    healthCheckUrl: 'https://api-us.company.com/health',
    primary: true,
  },
];

class GeographicRouter {
  private regionHealth: Map<string, boolean> = new Map();

  async route(request: Request): Promise<Response> {
    const countryCode = request.headers.get('CF-Connecting-Country') ||
      (request as any).cf?.country ||
      'US';

    const targetRegion = this.findBestRegion(countryCode);
    
    // Check region health
    if (!await this.isRegionHealthy(targetRegion)) {
      // Failover to primary
      const primaryRegion = regions.find(r => r.primary)!;
      return this.forwardToRegion(request, primaryRegion);
    }

    return this.forwardToRegion(request, targetRegion);
  }

  private findBestRegion(countryCode: string): Region {
    // Find region for this country
    const region = regions.find(r => r.countries.includes(countryCode));
    
    if (region) return region;
    
    // Default to primary
    return regions.find(r => r.primary) || regions[0];
  }

  private async isRegionHealthy(region: Region): Promise<boolean> {
    const cached = this.regionHealth.get(region.code);
    if (cached !== undefined) return cached;

    try {
      const response = await fetch(region.healthCheckUrl, {
        signal: AbortSignal.timeout(2000), // 2 second timeout
      });
      const healthy = response.ok;
      this.regionHealth.set(region.code, healthy);
      
      // Cache health status for 10 seconds
      setTimeout(() => this.regionHealth.delete(region.code), 10000);
      
      return healthy;
    } catch {
      this.regionHealth.set(region.code, false);
      setTimeout(() => this.regionHealth.delete(region.code), 5000);
      return false;
    }
  }

  private async forwardToRegion(request: Request, region: Region): Promise<Response> {
    const url = new URL(request.url);
    const targetUrl = `${region.endpoint}${url.pathname}${url.search}`;

    return fetch(targetUrl, {
      method: request.method,
      headers: {
        ...Object.fromEntries(request.headers),
        'X-Forwarded-Region': region.code,
        'X-Forwarded-For': request.headers.get('CF-Connecting-IP') || '',
      },
      body: request.method !== 'GET' && request.method !== 'HEAD' ? request.body : undefined,
    });
  }
}
```

---

## 7. A/B Testing at the Edge

```typescript
// ab-testing-edge.ts

interface Experiment {
  id: string;
  name: string;
  variants: Variant[];
  allocation: number; // % of traffic to include
  startDate: Date;
  endDate?: Date;
}

interface Variant {
  id: string;
  name: string;
  weight: number; // relative weight
  config: Record<string, unknown>;
}

class EdgeABTesting {
  private experiments: Experiment[];

  constructor(experiments: Experiment[]) {
    this.experiments = experiments;
  }

  assignVariant(userId: string, experimentId: string): Variant | null {
    const experiment = this.experiments.find(e => e.id === experimentId);
    if (!experiment) return null;

    // Check if experiment is active
    const now = new Date();
    if (now < experiment.startDate) return null;
    if (experiment.endDate && now > experiment.endDate) return null;

    // Determine if user is in experiment (stable hash)
    const hash = this.stableHash(`${userId}:${experimentId}`);
    const userBucket = hash % 100;

    if (userBucket >= experiment.allocation) return null;

    // Assign variant (stable)
    const variantHash = this.stableHash(`${userId}:${experimentId}:variant`);
    const totalWeight = experiment.variants.reduce((sum, v) => sum + v.weight, 0);
    let cumulativeWeight = 0;
    const bucket = variantHash % totalWeight;

    for (const variant of experiment.variants) {
      cumulativeWeight += variant.weight;
      if (bucket < cumulativeWeight) {
        return variant;
      }
    }

    return experiment.variants[experiment.variants.length - 1];
  }

  private stableHash(input: string): number {
    let hash = 5381;
    for (let i = 0; i < input.length; i++) {
      hash = ((hash << 5) + hash) + input.charCodeAt(i);
      hash = hash & hash; // Convert to 32bit integer
    }
    return Math.abs(hash);
  }

  applyVariantToRequest(
    request: Request,
    variant: Variant | null
  ): Request {
    if (!variant) return request;

    const modifiedHeaders = new Headers(request.headers);
    modifiedHeaders.set('X-AB-Variant', variant.id);
    modifiedHeaders.set('X-AB-Config', JSON.stringify(variant.config));

    return new Request(request.url, {
      method: request.method,
      headers: modifiedHeaders,
      body: request.body,
    });
  }
}

// Cloudflare Worker with A/B Testing
export async function handleWithABTest(
  request: Request,
  env: { EXPERIMENTS: string; CACHE: KVNamespace }
): Promise<Response> {
  const experiments = JSON.parse(env.EXPERIMENTS) as Experiment[];
  const abTester = new EdgeABTesting(experiments);

  // Get user ID from JWT or cookie
  const userId = getUserId(request);
  
  // Assign variant for checkout experiment
  const variant = abTester.assignVariant(userId, 'checkout-redesign-2024');
  
  // Modify request with variant info
  const modifiedRequest = abTester.applyVariantToRequest(request, variant);

  // Forward to origin with variant headers
  const response = await fetch(modifiedRequest);

  // Add experiment tracking headers to response
  const modifiedResponse = new Response(response.body, response);
  if (variant) {
    modifiedResponse.headers.set('X-Experiment-Id', 'checkout-redesign-2024');
    modifiedResponse.headers.set('X-Variant-Id', variant.id);
  }

  return modifiedResponse;
}

function getUserId(request: Request): string {
  const authHeader = request.headers.get('Authorization');
  if (authHeader?.startsWith('Bearer ')) {
    try {
      const payload = JSON.parse(
        atob(authHeader.substring(7).split('.')[1])
      );
      return payload.sub;
    } catch {
      // ignored
    }
  }
  
  // Fallback to session cookie or IP
  return request.headers.get('CF-Connecting-IP') || 'anonymous';
}
```

---

## 8. Edge Authentication

```typescript
// edge-authentication.ts
// Zero-trust authentication ที่ edge

interface TokenValidationResult {
  valid: boolean;
  userId?: string;
  roles?: string[];
  expiresAt?: number;
  reason?: string;
}

class EdgeAuthenticator {
  private publicKeys: Map<string, CryptoKey> = new Map();
  private jwtCache: Map<string, { result: TokenValidationResult; cachedAt: number }> = new Map();
  private cacheTTL = 60000; // 1 minute cache

  async validateJWT(token: string): Promise<TokenValidationResult> {
    // Check cache
    const cacheKey = token.substring(0, 32); // Use first 32 chars as key
    const cached = this.jwtCache.get(cacheKey);
    
    if (cached && Date.now() - cached.cachedAt < this.cacheTTL) {
      return cached.result;
    }

    const result = await this.validateJWTInternal(token);
    
    // Only cache valid tokens
    if (result.valid) {
      this.jwtCache.set(cacheKey, { result, cachedAt: Date.now() });
    }

    return result;
  }

  private async validateJWTInternal(token: string): Promise<TokenValidationResult> {
    try {
      const [headerB64, payloadB64, signatureB64] = token.split('.');
      
      if (!headerB64 || !payloadB64 || !signatureB64) {
        return { valid: false, reason: 'Invalid token format' };
      }

      const header = JSON.parse(atob(headerB64));
      const payload = JSON.parse(atob(payloadB64));

      // Check expiration
      if (payload.exp < Math.floor(Date.now() / 1000)) {
        return { valid: false, reason: 'Token expired' };
      }

      // Check not-before
      if (payload.nbf && payload.nbf > Math.floor(Date.now() / 1000)) {
        return { valid: false, reason: 'Token not yet valid' };
      }

      // Verify signature
      const publicKey = await this.getPublicKey(header.kid);
      if (!publicKey) {
        return { valid: false, reason: 'Unknown key ID' };
      }

      const encoder = new TextEncoder();
      const isValid = await crypto.subtle.verify(
        { name: 'RSASSA-PKCS1-v1_5', hash: 'SHA-256' },
        publicKey,
        base64UrlDecode(signatureB64),
        encoder.encode(`${headerB64}.${payloadB64}`)
      );

      if (!isValid) {
        return { valid: false, reason: 'Invalid signature' };
      }

      return {
        valid: true,
        userId: payload.sub,
        roles: payload.roles || [],
        expiresAt: payload.exp,
      };
    } catch (error) {
      return { valid: false, reason: 'Validation error' };
    }
  }

  private async getPublicKey(kid: string): Promise<CryptoKey | null> {
    if (this.publicKeys.has(kid)) {
      return this.publicKeys.get(kid)!;
    }

    // Fetch JWKS from auth server
    try {
      const response = await fetch('https://auth.company.com/.well-known/jwks.json');
      const jwks = await response.json() as { keys: any[] };
      
      for (const key of jwks.keys) {
        const cryptoKey = await crypto.subtle.importKey(
          'jwk',
          key,
          { name: 'RSASSA-PKCS1-v1_5', hash: 'SHA-256' },
          false,
          ['verify']
        );
        this.publicKeys.set(key.kid, cryptoKey);
      }

      return this.publicKeys.get(kid) || null;
    } catch {
      return null;
    }
  }
}

function base64UrlDecode(input: string): ArrayBuffer {
  const base64 = input.replace(/-/g, '+').replace(/_/g, '/');
  const binaryString = atob(base64);
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes.buffer;
}
```

---

## 9. Real-time Personalization

```typescript
// real-time-personalization.ts
// Personalization ที่ edge โดยไม่ต้องไปถึง origin

interface UserProfile {
  userId: string;
  language: string;
  currency: string;
  country: string;
  tier: 'free' | 'pro' | 'enterprise';
  preferences: {
    theme: 'light' | 'dark';
    density: 'compact' | 'normal' | 'comfortable';
    notifications: boolean;
  };
}

class EdgePersonalization {
  private kvStore: KVNamespace;

  constructor(kvStore: KVNamespace) {
    this.kvStore = kvStore;
  }

  async personalizeResponse(
    response: Response,
    userId: string,
    request: Request
  ): Promise<Response> {
    // Get user profile from edge KV
    const profile = await this.getUserProfile(userId);

    if (!profile) return response;

    const contentType = response.headers.get('Content-Type') || '';
    
    if (contentType.includes('text/html')) {
      return this.personalizeHTML(response, profile);
    }
    
    if (contentType.includes('application/json')) {
      return this.personalizeJSON(response, profile);
    }

    return response;
  }

  private async getUserProfile(userId: string): Promise<UserProfile | null> {
    const cached = await this.kvStore.get(`profile:${userId}`, 'json');
    return cached as UserProfile | null;
  }

  private async personalizeHTML(
    response: Response,
    profile: UserProfile
  ): Promise<Response> {
    // Use HTMLRewriter for Cloudflare Workers
    const rewriter = new HTMLRewriter()
      .on('[data-i18n]', new TranslationHandler(profile.language))
      .on('[data-currency]', new CurrencyHandler(profile.currency))
      .on('html', new ThemeHandler(profile.preferences.theme))
      .on('[data-tier]', new TierHandler(profile.tier));

    return rewriter.transform(response);
  }

  private async personalizeJSON(
    response: Response,
    profile: UserProfile
  ): Promise<Response> {
    const data = await response.json() as Record<string, unknown>;
    
    // Add personalization hints
    const personalized = {
      ...data,
      _personalization: {
        userId: profile.userId,
        currency: profile.currency,
        language: profile.language,
        tier: profile.tier,
      },
    };

    return new Response(JSON.stringify(personalized), {
      status: response.status,
      headers: response.headers,
    });
  }
}

class TranslationHandler {
  private language: string;
  
  constructor(language: string) {
    this.language = language;
  }

  element(element: Element) {
    const key = element.getAttribute('data-i18n');
    if (key) {
      // In a real implementation, load translations from KV
      element.setAttribute('lang', this.language);
    }
  }
}

class CurrencyHandler {
  private currency: string;
  
  constructor(currency: string) {
    this.currency = currency;
  }

  element(element: Element) {
    element.setAttribute('data-currency', this.currency);
  }
}

class ThemeHandler {
  private theme: string;
  
  constructor(theme: string) {
    this.theme = theme;
  }

  element(element: Element) {
    element.setAttribute('data-theme', this.theme);
  }
}

class TierHandler {
  private tier: string;
  
  constructor(tier: string) {
    this.tier = tier;
  }

  element(element: Element) {
    const requiredTier = element.getAttribute('data-tier');
    if (requiredTier && !this.hasTierAccess(requiredTier)) {
      element.setAttribute('style', 'display: none;');
    }
  }

  private hasTierAccess(requiredTier: string): boolean {
    const tiers = ['free', 'pro', 'enterprise'];
    const userTierIndex = tiers.indexOf(this.tier);
    const requiredTierIndex = tiers.indexOf(requiredTier);
    return userTierIndex >= requiredTierIndex;
  }
}
```

---

## 10. Global Latency Optimization

```typescript
// latency-optimization.ts

// HTTP/3 and Connection optimization
const optimizationHeaders = {
  // Enable HTTP/3 (QUIC)
  'Alt-Svc': 'h3=":443"; ma=86400',
  
  // Enable server push hints
  'Link': [
    '</api/critical-data>; rel=preload; as=fetch; crossorigin',
    '</css/critical.css>; rel=preload; as=style',
  ].join(', '),
  
  // Early hints
  'X-Early-Hints': 'enabled',
};

// Compression optimization
class EdgeCompressor {
  async compressResponse(response: Response, request: Request): Promise<Response> {
    const acceptEncoding = request.headers.get('Accept-Encoding') || '';
    const contentType = response.headers.get('Content-Type') || '';

    // ไม่ compress images หรือ already compressed content
    if (
      contentType.includes('image/') ||
      contentType.includes('video/') ||
      contentType.includes('audio/')
    ) {
      return response;
    }

    if (acceptEncoding.includes('br')) {
      return this.brotliCompress(response);
    }

    if (acceptEncoding.includes('gzip')) {
      return this.gzipCompress(response);
    }

    return response;
  }

  private async brotliCompress(response: Response): Promise<Response> {
    // Brotli compression using CompressionStream
    const { readable, writable } = new TransformStream();
    const compressionStream = new CompressionStream('deflate-raw');
    
    response.body?.pipeTo(compressionStream.writable);
    compressionStream.readable.pipeTo(writable);

    return new Response(readable, {
      ...response,
      headers: {
        ...Object.fromEntries(response.headers),
        'Content-Encoding': 'br',
        'Vary': 'Accept-Encoding',
      },
    });
  }

  private async gzipCompress(response: Response): Promise<Response> {
    const { readable, writable } = new TransformStream();
    const compressionStream = new CompressionStream('gzip');
    
    response.body?.pipeTo(compressionStream.writable);
    compressionStream.readable.pipeTo(writable);

    return new Response(readable, {
      ...response,
      headers: {
        ...Object.fromEntries(response.headers),
        'Content-Encoding': 'gzip',
        'Vary': 'Accept-Encoding',
      },
    });
  }
}

// TCP/HTTP connection optimization
const connectionConfig = {
  http2: {
    pushEnabled: true,
    maxConcurrentStreams: 100,
    initialWindowSize: 65535 * 4,
  },
  keepAlive: {
    enabled: true,
    timeout: 90, // seconds
    interval: 30,
  },
  preconnect: {
    origins: [
      'https://api.company.com',
      'https://cdn.company.com',
      'https://analytics.company.com',
    ],
  },
};
```

---

## Deployment สำหรับ Edge Worker

```yaml
# cloudflare-worker-ci.yaml
name: Deploy Edge Worker

on:
  push:
    branches: [main]
    paths:
      - 'workers/**'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
        working-directory: workers
      
      - name: Run tests
        run: npm test
        working-directory: workers
      
      - name: Build
        run: npm run build
        working-directory: workers
      
      - name: Deploy to Cloudflare Workers
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          workingDirectory: workers
          command: deploy --env production
      
      - name: Smoke Test
        run: |
          curl -f https://api.company.com/health
          echo "Edge worker deployed and healthy"
```

---

## สรุปตาราง Edge Computing

| Technology | Use Case | Latency | Limitations |
|-----------|----------|---------|-------------|
| Cloudflare Workers | Authentication, caching, routing | 1-5ms | 10ms CPU limit, no filesystem |
| AWS Lambda@Edge | Request/response manipulation | 5-15ms | 5s timeout, limited memory |
| Fastly Compute@Edge | Complex edge logic | 1-5ms | Wasm-based |
| Vercel Edge Functions | Next.js edge rendering | 1-5ms | Tied to Vercel |
| Deno Deploy | Modern JS at edge | 1-5ms | Deno runtime only |

| CDN Pattern | TTL | When to Purge | Use Case |
|------------|-----|--------------|----------|
| Static Assets | 1 year | On deploy | Images, JS, CSS |
| API Products | 5 min | On update | Product catalog |
| Search Results | 30s | Rarely | Search queries |
| User Data | Private | On change | Profile, settings |
| Checkout | 0 | N/A | Cart, payment |

| Optimization | Impact | Complexity |
|-------------|--------|-----------|
| HTTP/3 (QUIC) | 20-30% latency improvement | Low |
| Brotli compression | 20-30% smaller payloads | Low |
| Edge authentication | Eliminate origin auth roundtrip | Medium |
| Edge caching | 80-95% cache hit rate | Medium |
| Geographic routing | 30-50% latency reduction | Medium |
| A/B testing at edge | Zero origin overhead | Medium |

---

## สรุป

Edge Computing สำหรับ Microservices:

1. **Cloudflare Workers** - authentication, rate limiting, routing ที่ edge ด้วย TypeScript
2. **Lambda@Edge** - integration กับ AWS ecosystem, complex request manipulation
3. **CDN Strategy** - caching policies ที่เหมาะสมสำหรับแต่ละ content type
4. **Geographic Routing** - ลด latency ด้วยการ route ไปยัง region ที่ใกล้ที่สุด
5. **A/B Testing** - ทดสอบ features โดยไม่ impact origin performance
6. **Edge Auth** - ตรวจสอบ JWT ที่ edge ลด load บน origin
7. **Personalization** - ปรับ content สำหรับ user แต่ละคนที่ edge
8. **Compression** - HTTP/3, Brotli เพื่อลด payload size และ latency

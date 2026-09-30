# Part 53: Service Discovery และ Load Balancing

## บทนำ

ใน Microservices Architecture จำนวน Service Instance เปลี่ยนแปลงตลอดเวลา — Auto-scaling เพิ่ม Instance ใหม่, Deployment ทำให้ Instance เก่าหายไป, Node ล้มเหลว Service Discovery คือกลไกที่ช่วยให้ Service ค้นหา Endpoint ของ Service อื่นได้โดยอัตโนมัติ

---

## 1. ประเภทของ Service Discovery

### 1.1 Client-side Discovery
Client ถาม Service Registry โดยตรง แล้วเลือก Instance เอง

```
Client → Query ServiceRegistry → [instance-1:3001, instance-2:3002, instance-3:3003]
       → Load Balance locally → instance-2:3002
```

### 1.2 Server-side Discovery
Client ถาม Load Balancer แล้ว LB ถาม Registry เอง

```
Client → Load Balancer (Nginx/Envoy) → ServiceRegistry → instance-1:3001
```

### 1.3 DNS-based Discovery
ใช้ DNS SRV records เพื่อ resolve service address

```
client.getServiceAddress('inventory-service')
  → DNS: _inventory._tcp.service.consul → [10.0.0.1:3002, 10.0.0.2:3002]
```

---

## 2. Consul Service Discovery

### 2.1 ติดตั้งและ Config Consul

```yaml
# consul/config/server.hcl
datacenter = "dc1"
data_dir = "/consul/data"
log_level = "INFO"
node_name = "consul-server-1"

server = true
bootstrap_expect = 1

ui_config {
  enabled = true
}

addresses {
  http = "0.0.0.0"
  dns = "0.0.0.0"
}

ports {
  http = 8500
  dns = 8600
}

connect {
  enabled = true
}

performance {
  raft_multiplier = 1
}
```

### 2.2 Docker Compose สำหรับ Consul

```yaml
# docker-compose.consul.yml
version: '3.8'

services:
  consul:
    image: hashicorp/consul:1.18
    command: agent -server -bootstrap-expect=1 -ui -client=0.0.0.0
    environment:
      CONSUL_BIND_INTERFACE: eth0
    ports:
      - "8500:8500"   # HTTP API & UI
      - "8600:8600/udp" # DNS
      - "8300:8300"   # Server RPC
      - "8301:8301"   # LAN Serf
    volumes:
      - consul_data:/consul/data
      - ./consul/config:/consul/config
    healthcheck:
      test: ["CMD", "consul", "members"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Registrator — auto-register Docker containers
  registrator:
    image: gliderlabs/registrator:latest
    command: -internal consul://consul:8500
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock
    depends_on:
      consul:
        condition: service_healthy

volumes:
  consul_data:
```

---

## 3. TypeScript Consul Client Library

```typescript
// packages/shared/src/service-discovery/consul.client.ts

import Consul from 'consul';
import { EventEmitter } from 'events';

export interface ServiceInstance {
  id: string;
  name: string;
  address: string;
  port: number;
  tags: string[];
  meta: Record<string, string>;
  healthy: boolean;
}

export interface HealthCheckConfig {
  http?: string;
  tcp?: string;
  grpc?: string;
  interval: string;
  timeout: string;
  deregisterCriticalServiceAfter?: string;
}

export class ConsulServiceDiscovery extends EventEmitter {
  private readonly consul: Consul.Consul;
  private readonly watchMap: Map<string, Consul.Watch> = new Map();
  private readonly serviceCache: Map<string, ServiceInstance[]> = new Map();

  constructor(
    private readonly config: {
      host: string;
      port: number;
      token?: string;
    }
  ) {
    super();
    this.consul = new Consul({
      host: config.host,
      port: config.port,
      promisify: true,
      defaults: config.token
        ? { token: config.token }
        : undefined,
    });
  }

  // ลงทะเบียน Service
  async register(params: {
    id: string;
    name: string;
    address: string;
    port: number;
    tags?: string[];
    meta?: Record<string, string>;
    healthCheck?: HealthCheckConfig;
  }): Promise<void> {
    const registration: Consul.Agent.Service.RegisterOptions = {
      id: params.id,
      name: params.name,
      address: params.address,
      port: params.port,
      tags: params.tags ?? [],
      meta: params.meta ?? {},
    };

    if (params.healthCheck) {
      registration.check = {
        http: params.healthCheck.http,
        tcp: params.healthCheck.tcp,
        grpc: params.healthCheck.grpc,
        interval: params.healthCheck.interval,
        timeout: params.healthCheck.timeout,
        deregistercriticalserviceafter:
          params.healthCheck.deregisterCriticalServiceAfter ?? '30s',
      };
    }

    await this.consul.agent.service.register(registration);
    console.log(`[Consul] Registered service: ${params.name} (${params.id})`);
  }

  // ยกเลิกการลงทะเบียน
  async deregister(serviceId: string): Promise<void> {
    await this.consul.agent.service.deregister(serviceId);
    console.log(`[Consul] Deregistered service: ${serviceId}`);
  }

  // ค้นหา Service ที่ Healthy
  async getHealthyInstances(serviceName: string): Promise<ServiceInstance[]> {
    const result = await this.consul.health.service({
      service: serviceName,
      passing: true, // Only healthy instances
    }) as Array<{
      Service: {
        ID: string;
        Service: string;
        Address: string;
        Port: number;
        Tags: string[];
        Meta: Record<string, string>;
      };
      Checks: Array<{ Status: string }>;
    }>;

    return result.map((entry) => ({
      id: entry.Service.ID,
      name: entry.Service.Service,
      address: entry.Service.Address,
      port: entry.Service.Port,
      tags: entry.Service.Tags,
      meta: entry.Service.Meta,
      healthy: entry.Checks.every((c) => c.Status === 'passing'),
    }));
  }

  // Watch Service Changes (real-time)
  watchService(serviceName: string): void {
    if (this.watchMap.has(serviceName)) {
      return; // Already watching
    }

    const watch = this.consul.watch({
      method: this.consul.health.service,
      options: {
        service: serviceName,
        passing: true,
      } as Consul.Health.ServiceOptions,
    });

    watch.on('change', (data: unknown) => {
      const instances = (data as Array<{
        Service: {
          ID: string;
          Service: string;
          Address: string;
          Port: number;
          Tags: string[];
          Meta: Record<string, string>;
        };
        Checks: Array<{ Status: string }>;
      }>).map((entry) => ({
        id: entry.Service.ID,
        name: entry.Service.Service,
        address: entry.Service.Address,
        port: entry.Service.Port,
        tags: entry.Service.Tags,
        meta: entry.Service.Meta,
        healthy: entry.Checks.every((c) => c.Status === 'passing'),
      }));

      this.serviceCache.set(serviceName, instances);
      this.emit('service-changed', { serviceName, instances });

      console.log(
        `[Consul] Service ${serviceName} updated: ${instances.length} healthy instances`
      );
    });

    watch.on('error', (err: Error) => {
      console.error(`[Consul] Watch error for ${serviceName}:`, err);
      this.emit('error', { serviceName, error: err });
    });

    this.watchMap.set(serviceName, watch);
  }

  // Get cached instances (fast, no network call)
  getCachedInstances(serviceName: string): ServiceInstance[] {
    return this.serviceCache.get(serviceName) ?? [];
  }

  // Key-Value Store
  async kvGet(key: string): Promise<string | null> {
    const result = await this.consul.kv.get(key) as { Value: string } | null;
    if (!result) return null;
    return Buffer.from(result.Value, 'base64').toString('utf-8');
  }

  async kvSet(key: string, value: string): Promise<void> {
    await this.consul.kv.set(key, value);
  }

  async kvDelete(key: string): Promise<void> {
    await this.consul.kv.del(key);
  }

  stopWatch(serviceName: string): void {
    const watch = this.watchMap.get(serviceName);
    if (watch) {
      watch.end();
      this.watchMap.delete(serviceName);
    }
  }

  stopAllWatches(): void {
    for (const [name, watch] of this.watchMap) {
      watch.end();
      this.watchMap.delete(name);
    }
  }
}
```

---

## 4. Load Balancing Algorithms

```typescript
// packages/shared/src/load-balancer/load-balancer.ts

import { ServiceInstance } from '../service-discovery/consul.client';

export interface LoadBalancer {
  select(instances: ServiceInstance[]): ServiceInstance | null;
  onSuccess(instanceId: string, responseTimeMs: number): void;
  onFailure(instanceId: string, error: Error): void;
}

// ─── Round Robin ────────────────────────────────────────────────────────────

export class RoundRobinLoadBalancer implements LoadBalancer {
  private counter = 0;

  select(instances: ServiceInstance[]): ServiceInstance | null {
    if (instances.length === 0) return null;
    const index = this.counter % instances.length;
    this.counter = (this.counter + 1) % Number.MAX_SAFE_INTEGER;
    return instances[index];
  }

  onSuccess(_instanceId: string, _responseTimeMs: number): void {}
  onFailure(_instanceId: string, _error: Error): void {}
}

// ─── Least Connections ───────────────────────────────────────────────────────

export class LeastConnectionsLoadBalancer implements LoadBalancer {
  private readonly connections: Map<string, number> = new Map();

  select(instances: ServiceInstance[]): ServiceInstance | null {
    if (instances.length === 0) return null;

    let minConnections = Infinity;
    let selected: ServiceInstance | null = null;

    for (const instance of instances) {
      const connections = this.connections.get(instance.id) ?? 0;
      if (connections < minConnections) {
        minConnections = connections;
        selected = instance;
      }
    }

    if (selected) {
      this.connections.set(
        selected.id,
        (this.connections.get(selected.id) ?? 0) + 1
      );
    }

    return selected;
  }

  onSuccess(instanceId: string, _responseTimeMs: number): void {
    const current = this.connections.get(instanceId) ?? 0;
    this.connections.set(instanceId, Math.max(0, current - 1));
  }

  onFailure(instanceId: string, _error: Error): void {
    const current = this.connections.get(instanceId) ?? 0;
    this.connections.set(instanceId, Math.max(0, current - 1));
  }
}

// ─── Weighted Round Robin ────────────────────────────────────────────────────

export class WeightedRoundRobinLoadBalancer implements LoadBalancer {
  private currentIndex = -1;
  private currentWeight = 0;
  private maxWeight = 0;
  private gcdWeight = 0;

  private getWeight(instance: ServiceInstance): number {
    const weight = parseInt(instance.meta['weight'] ?? '1', 10);
    return isNaN(weight) || weight <= 0 ? 1 : weight;
  }

  select(instances: ServiceInstance[]): ServiceInstance | null {
    if (instances.length === 0) return null;
    if (instances.length === 1) return instances[0];

    const weights = instances.map((i) => this.getWeight(i));
    this.maxWeight = Math.max(...weights);
    this.gcdWeight = weights.reduce(gcd);

    // Nginx-style weighted round-robin (smooth)
    while (true) {
      this.currentIndex = (this.currentIndex + 1) % instances.length;

      if (this.currentIndex === 0) {
        this.currentWeight = this.currentWeight - this.gcdWeight;
        if (this.currentWeight <= 0) {
          this.currentWeight = this.maxWeight;
          if (this.currentWeight === 0) return null;
        }
      }

      const instance = instances[this.currentIndex];
      if (this.getWeight(instance) >= this.currentWeight) {
        return instance;
      }
    }
  }

  onSuccess(_instanceId: string, _responseTimeMs: number): void {}
  onFailure(_instanceId: string, _error: Error): void {}
}

// ─── Random ──────────────────────────────────────────────────────────────────

export class RandomLoadBalancer implements LoadBalancer {
  select(instances: ServiceInstance[]): ServiceInstance | null {
    if (instances.length === 0) return null;
    return instances[Math.floor(Math.random() * instances.length)];
  }

  onSuccess(_instanceId: string, _responseTimeMs: number): void {}
  onFailure(_instanceId: string, _error: Error): void {}
}

// ─── Exponential Moving Average (Response Time-aware) ───────────────────────

export class EWMALoadBalancer implements LoadBalancer {
  private readonly latency: Map<string, number> = new Map();
  private readonly ALPHA = 0.3; // smoothing factor

  select(instances: ServiceInstance[]): ServiceInstance | null {
    if (instances.length === 0) return null;

    // เลือก instance ที่มี latency ต่ำที่สุด
    let minLatency = Infinity;
    let selected: ServiceInstance | null = null;

    for (const instance of instances) {
      const avgLatency = this.latency.get(instance.id) ?? 100; // default 100ms
      if (avgLatency < minLatency) {
        minLatency = avgLatency;
        selected = instance;
      }
    }

    return selected;
  }

  onSuccess(instanceId: string, responseTimeMs: number): void {
    const current = this.latency.get(instanceId) ?? responseTimeMs;
    this.latency.set(
      instanceId,
      this.ALPHA * responseTimeMs + (1 - this.ALPHA) * current
    );
  }

  onFailure(instanceId: string, _error: Error): void {
    // เพิ่ม latency สมมุติเมื่อ error เพื่อลด traffic ไปยัง unhealthy instance
    const current = this.latency.get(instanceId) ?? 1000;
    this.latency.set(instanceId, current * 2);
  }
}

// Helper
function gcd(a: number, b: number): number {
  return b === 0 ? a : gcd(b, a % b);
}
```

---

## 5. Service Client ที่ใช้ Consul + Load Balancer

```typescript
// packages/shared/src/service-discovery/service.client.ts

import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import { ConsulServiceDiscovery, ServiceInstance } from './consul.client';
import {
  LeastConnectionsLoadBalancer,
  LoadBalancer,
} from '../load-balancer/load-balancer';

export interface CircuitBreakerState {
  state: 'CLOSED' | 'OPEN' | 'HALF_OPEN';
  failures: number;
  lastFailureTime: number;
  nextAttemptTime: number;
}

export class ServiceClient {
  private readonly loadBalancer: LoadBalancer;
  private readonly circuitBreakers: Map<string, CircuitBreakerState> = new Map();
  private readonly http: AxiosInstance;

  private readonly FAILURE_THRESHOLD = 5;
  private readonly OPEN_TIMEOUT_MS = 30000;

  constructor(
    private readonly discovery: ConsulServiceDiscovery,
    loadBalancer?: LoadBalancer
  ) {
    this.loadBalancer = loadBalancer ?? new LeastConnectionsLoadBalancer();
    this.http = axios.create({
      timeout: 10000,
    });

    // Request interceptor — inject tracing headers
    this.http.interceptors.request.use((config) => {
      if (!config.headers['x-request-id']) {
        config.headers['x-request-id'] = generateRequestId();
      }
      return config;
    });
  }

  async request<T = unknown>(
    serviceName: string,
    config: AxiosRequestConfig
  ): Promise<AxiosResponse<T>> {
    const circuitBreaker = this.getCircuitBreaker(serviceName);

    // Circuit Breaker: OPEN → ไม่ส่ง Request
    if (circuitBreaker.state === 'OPEN') {
      const now = Date.now();
      if (now < circuitBreaker.nextAttemptTime) {
        throw new Error(
          `Circuit breaker OPEN for ${serviceName}. Retry after ${Math.ceil(
            (circuitBreaker.nextAttemptTime - now) / 1000
          )}s`
        );
      }
      // ถึงเวลาลอง Half-Open
      circuitBreaker.state = 'HALF_OPEN';
    }

    const instance = await this.selectInstance(serviceName);
    if (!instance) {
      throw new Error(`No healthy instances available for service: ${serviceName}`);
    }

    const url = `http://${instance.address}:${instance.port}${config.url}`;
    const startTime = Date.now();

    try {
      const response = await this.http.request<T>({
        ...config,
        url,
      });

      const duration = Date.now() - startTime;
      this.loadBalancer.onSuccess(instance.id, duration);
      this.onSuccess(serviceName);

      return response;
    } catch (error) {
      const duration = Date.now() - startTime;
      this.loadBalancer.onFailure(instance.id, error as Error);
      this.onFailure(serviceName, error as Error);

      throw error;
    }
  }

  async get<T = unknown>(
    serviceName: string,
    path: string,
    config?: AxiosRequestConfig
  ): Promise<T> {
    const response = await this.request<T>(serviceName, {
      ...config,
      method: 'GET',
      url: path,
    });
    return response.data;
  }

  async post<T = unknown>(
    serviceName: string,
    path: string,
    data?: unknown,
    config?: AxiosRequestConfig
  ): Promise<T> {
    const response = await this.request<T>(serviceName, {
      ...config,
      method: 'POST',
      url: path,
      data,
    });
    return response.data;
  }

  private async selectInstance(serviceName: string): Promise<ServiceInstance | null> {
    // ลอง cache ก่อน
    let instances = this.discovery.getCachedInstances(serviceName);

    // ถ้าไม่มี cache ให้ query ใหม่
    if (instances.length === 0) {
      instances = await this.discovery.getHealthyInstances(serviceName);
      // Start watching for future updates
      this.discovery.watchService(serviceName);
    }

    return this.loadBalancer.select(instances);
  }

  private getCircuitBreaker(serviceName: string): CircuitBreakerState {
    if (!this.circuitBreakers.has(serviceName)) {
      this.circuitBreakers.set(serviceName, {
        state: 'CLOSED',
        failures: 0,
        lastFailureTime: 0,
        nextAttemptTime: 0,
      });
    }
    return this.circuitBreakers.get(serviceName)!;
  }

  private onSuccess(serviceName: string): void {
    const cb = this.getCircuitBreaker(serviceName);
    if (cb.state === 'HALF_OPEN') {
      cb.state = 'CLOSED';
      cb.failures = 0;
      console.log(`[CircuitBreaker] ${serviceName} → CLOSED (recovered)`);
    }
    cb.failures = 0;
  }

  private onFailure(serviceName: string, _error: Error): void {
    const cb = this.getCircuitBreaker(serviceName);
    cb.failures += 1;
    cb.lastFailureTime = Date.now();

    if (cb.failures >= this.FAILURE_THRESHOLD && cb.state === 'CLOSED') {
      cb.state = 'OPEN';
      cb.nextAttemptTime = Date.now() + this.OPEN_TIMEOUT_MS;
      console.warn(
        `[CircuitBreaker] ${serviceName} → OPEN (${cb.failures} failures)`
      );
    }
  }
}

function generateRequestId(): string {
  return `req-${Date.now()}-${Math.random().toString(36).slice(2, 9)}`;
}
```

---

## 6. Consul Health Check Registration ใน Service

```typescript
// packages/shared/src/service-discovery/service.registry.ts

import os from 'os';
import { ConsulServiceDiscovery } from './consul.client';

export interface ServiceRegistrationConfig {
  name: string;
  port: number;
  tags?: string[];
  meta?: Record<string, string>;
  healthCheckPath?: string;
}

export async function registerService(
  discovery: ConsulServiceDiscovery,
  config: ServiceRegistrationConfig
): Promise<() => Promise<void>> {
  const hostname = process.env.HOSTNAME ?? os.hostname();
  const address = process.env.SERVICE_ADDRESS ?? getLocalIp();
  const serviceId = `${config.name}-${hostname}-${config.port}`;

  const healthCheckUrl = config.healthCheckPath
    ? `http://${address}:${config.port}${config.healthCheckPath}`
    : `http://${address}:${config.port}/health`;

  await discovery.register({
    id: serviceId,
    name: config.name,
    address,
    port: config.port,
    tags: [
      ...(config.tags ?? []),
      `version=${process.env.APP_VERSION ?? '1.0.0'}`,
    ],
    meta: {
      ...config.meta,
      environment: process.env.NODE_ENV ?? 'development',
    },
    healthCheck: {
      http: healthCheckUrl,
      interval: '10s',
      timeout: '5s',
      deregisterCriticalServiceAfter: '30s',
    },
  });

  // Graceful deregistration on shutdown
  const deregister = async () => {
    await discovery.deregister(serviceId);
    console.log(`[ServiceRegistry] Deregistered: ${serviceId}`);
  };

  process.on('SIGTERM', async () => {
    await deregister();
    process.exit(0);
  });

  process.on('SIGINT', async () => {
    await deregister();
    process.exit(0);
  });

  console.log(
    `[ServiceRegistry] Registered: ${serviceId} at ${address}:${config.port}`
  );

  return deregister;
}

function getLocalIp(): string {
  const interfaces = os.networkInterfaces();
  for (const name of Object.keys(interfaces)) {
    for (const iface of interfaces[name] ?? []) {
      if (iface.family === 'IPv4' && !iface.internal) {
        return iface.address;
      }
    }
  }
  return '127.0.0.1';
}
```

---

## 7. DNS-based Discovery กับ Consul DNS

```typescript
// packages/shared/src/service-discovery/dns.discovery.ts
// ใช้ DNS SRV records ที่ Consul expose ผ่าน port 8600

import dns from 'dns';
import { promisify } from 'util';

const resolveSrv = promisify(dns.resolveSrv);
const resolve4 = promisify(dns.resolve4);

// Configure DNS to use Consul
// ใน Node.js ต้องตั้งค่า /etc/resolv.conf หรือใช้ --dns-server flag
// หรือ set custom resolver

export class DnsServiceDiscovery {
  private readonly domain: string;

  constructor(consulDomain = 'service.consul') {
    this.domain = consulDomain;
  }

  async resolveService(
    serviceName: string,
    tag?: string
  ): Promise<Array<{ address: string; port: number; priority: number; weight: number }>> {
    // Consul DNS format: [tag.]<service>.service.[datacenter.]consul
    const hostname = tag
      ? `${tag}.${serviceName}.${this.domain}`
      : `${serviceName}.${this.domain}`;

    try {
      const records = await resolveSrv(`_${serviceName}._tcp.${this.domain}`);

      const resolved = await Promise.all(
        records.map(async (record) => {
          const addresses = await resolve4(record.name);
          return {
            address: addresses[0],
            port: record.port,
            priority: record.priority,
            weight: record.weight,
          };
        })
      );

      return resolved;
    } catch (error) {
      console.error(`DNS resolution failed for ${hostname}:`, error);
      return [];
    }
  }

  async resolveA(serviceName: string): Promise<string[]> {
    try {
      return await resolve4(`${serviceName}.${this.domain}`);
    } catch {
      return [];
    }
  }
}
```

---

## 8. Kubernetes Service Discovery

```yaml
# k8s/inventory-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service
  namespace: microservices
  labels:
    app: inventory-service
    version: v1.2.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inventory-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: inventory-service
        version: v1.2.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3002"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: inventory-service-sa
      containers:
        - name: inventory-service
          image: myrepo/inventory-service:1.2.0
          ports:
            - name: http
              containerPort: 3002
          env:
            - name: PORT
              value: "3002"
            - name: NODE_ENV
              value: production
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: inventory-service
  namespace: microservices
  labels:
    app: inventory-service
spec:
  selector:
    app: inventory-service
  ports:
    - name: http
      port: 80
      targetPort: http
  type: ClusterIP
---
# Headless Service สำหรับ DNS-based discovery โดยตรง
apiVersion: v1
kind: Service
metadata:
  name: inventory-service-headless
  namespace: microservices
spec:
  clusterIP: None  # Headless
  selector:
    app: inventory-service
  ports:
    - port: 3002
      targetPort: http
```

```typescript
// packages/shared/src/service-discovery/kubernetes.discovery.ts
// ใน Kubernetes, DNS ทำงานอัตโนมัติ
// inventory-service.microservices.svc.cluster.local resolves ไปยัง Service IP

export class KubernetesServiceDiscovery {
  private readonly namespace: string;
  private readonly clusterDomain: string;

  constructor(
    namespace = process.env.K8S_NAMESPACE ?? 'default',
    clusterDomain = 'cluster.local'
  ) {
    this.namespace = namespace;
    this.clusterDomain = clusterDomain;
  }

  // ได้ DNS ของ Service
  getServiceDns(serviceName: string): string {
    return `${serviceName}.${this.namespace}.svc.${this.clusterDomain}`;
  }

  // ได้ URL ของ Service
  getServiceUrl(serviceName: string, port = 80, protocol = 'http'): string {
    return `${protocol}://${this.getServiceDns(serviceName)}:${port}`;
  }

  // ใช้ Headless Service เพื่อ get individual pod IPs
  async getPodIps(headlessServiceName: string): Promise<string[]> {
    const { default: dns } = await import('dns');
    const resolve4 = (hostname: string) =>
      new Promise<string[]>((resolve, reject) =>
        dns.resolve4(hostname, (err, addresses) =>
          err ? reject(err) : resolve(addresses)
        )
      );

    try {
      return await resolve4(
        `${headlessServiceName}.${this.namespace}.svc.${this.clusterDomain}`
      );
    } catch {
      return [];
    }
  }
}
```

---

## 9. Envoy xDS API Integration

```typescript
// packages/shared/src/envoy/xds.client.ts
// Envoy Dynamic Discovery Service - TypeScript client

import { credentials, loadPackageDefinition } from '@grpc/grpc-js';
import { loadSync } from '@grpc/proto-loader';
import path from 'path';

// Envoy EDS (Endpoint Discovery Service) types
interface Locality {
  region?: string;
  zone?: string;
  sub_zone?: string;
}

interface LbEndpoint {
  endpoint: {
    address: {
      socket_address: {
        address: string;
        port_value: number;
      };
    };
  };
  health_status?: number; // 0=UNKNOWN, 1=HEALTHY, 2=UNHEALTHY
  load_balancing_weight?: { value: number };
}

interface LocalityLbEndpoints {
  locality: Locality;
  lb_endpoints: LbEndpoint[];
  load_balancing_weight?: { value: number };
}

interface ClusterLoadAssignment {
  cluster_name: string;
  endpoints: LocalityLbEndpoints[];
}

// Envoy Control Plane (xDS Server) for Node.js
// ใช้ @envoy-node/control-plane library

export class EnvoyControlPlane {
  private readonly clusters: Map<string, ClusterLoadAssignment> = new Map();
  private readonly version: number = 0;

  updateCluster(clusterName: string, endpoints: Array<{
    address: string;
    port: number;
    weight?: number;
    healthy?: boolean;
    zone?: string;
  }>): void {
    const clusterLoadAssignment: ClusterLoadAssignment = {
      cluster_name: clusterName,
      endpoints: [{
        locality: { zone: 'local' },
        lb_endpoints: endpoints.map(ep => ({
          endpoint: {
            address: {
              socket_address: {
                address: ep.address,
                port_value: ep.port,
              },
            },
          },
          health_status: ep.healthy === false ? 2 : 1,
          load_balancing_weight: ep.weight
            ? { value: ep.weight }
            : undefined,
        })),
      }],
    };

    this.clusters.set(clusterName, clusterLoadAssignment);
    console.log(
      `[EnvoyControlPlane] Updated cluster ${clusterName} with ${endpoints.length} endpoints`
    );
  }

  getCluster(clusterName: string): ClusterLoadAssignment | undefined {
    return this.clusters.get(clusterName);
  }

  getAllClusters(): ClusterLoadAssignment[] {
    return Array.from(this.clusters.values());
  }
}
```

---

## 10. Spring Cloud LoadBalancer Equivalent ใน Node.js

```typescript
// packages/shared/src/load-balancer/smart-load-balancer.ts
// เทียบเท่า Spring Cloud LoadBalancer พร้อม health-aware routing

import { ConsulServiceDiscovery, ServiceInstance } from '../service-discovery/consul.client';
import { LoadBalancer, RoundRobinLoadBalancer } from './load-balancer';
import axios from 'axios';

interface InstanceStats {
  successCount: number;
  failureCount: number;
  avgResponseTime: number;
  lastHealthCheck: Date;
  isHealthy: boolean;
  circuitState: 'CLOSED' | 'OPEN' | 'HALF_OPEN';
  circuitOpenedAt?: Date;
}

export class SmartLoadBalancer {
  private readonly stats: Map<string, InstanceStats> = new Map();
  private readonly algorithm: LoadBalancer;
  private readonly healthCheckInterval: ReturnType<typeof setInterval>;

  constructor(
    private readonly discovery: ConsulServiceDiscovery,
    algorithm?: LoadBalancer
  ) {
    this.algorithm = algorithm ?? new RoundRobinLoadBalancer();

    // Background health checks
    this.healthCheckInterval = setInterval(
      () => this.performHealthChecks(),
      30000
    );
  }

  async getUrl(serviceName: string, path: string): Promise<string> {
    const instances = await this.discovery.getHealthyInstances(serviceName);
    const healthyInstances = instances.filter(i =>
      this.isInstanceHealthy(i.id)
    );

    if (healthyInstances.length === 0) {
      throw new Error(`No healthy instances for: ${serviceName}`);
    }

    const selected = this.algorithm.select(healthyInstances);
    if (!selected) {
      throw new Error(`Load balancer returned null for: ${serviceName}`);
    }

    return `http://${selected.address}:${selected.port}${path}`;
  }

  private isInstanceHealthy(instanceId: string): boolean {
    const stats = this.stats.get(instanceId);
    if (!stats) return true; // Assume healthy if no stats

    if (stats.circuitState === 'OPEN') {
      // Check if enough time has passed to try again
      const openedAt = stats.circuitOpenedAt?.getTime() ?? 0;
      if (Date.now() - openedAt > 30000) {
        stats.circuitState = 'HALF_OPEN';
        return true;
      }
      return false;
    }

    return stats.isHealthy;
  }

  private getOrCreateStats(instanceId: string): InstanceStats {
    if (!this.stats.has(instanceId)) {
      this.stats.set(instanceId, {
        successCount: 0,
        failureCount: 0,
        avgResponseTime: 0,
        lastHealthCheck: new Date(),
        isHealthy: true,
        circuitState: 'CLOSED',
      });
    }
    return this.stats.get(instanceId)!;
  }

  recordSuccess(instanceId: string, responseTimeMs: number): void {
    const stats = this.getOrCreateStats(instanceId);
    stats.successCount++;
    stats.avgResponseTime =
      stats.avgResponseTime === 0
        ? responseTimeMs
        : stats.avgResponseTime * 0.7 + responseTimeMs * 0.3;

    if (stats.circuitState === 'HALF_OPEN') {
      stats.circuitState = 'CLOSED';
      stats.failureCount = 0;
      console.log(`[SmartLB] Instance ${instanceId} circuit CLOSED`);
    }

    this.algorithm.onSuccess(instanceId, responseTimeMs);
  }

  recordFailure(instanceId: string, error: Error): void {
    const stats = this.getOrCreateStats(instanceId);
    stats.failureCount++;

    if (stats.failureCount >= 5 && stats.circuitState === 'CLOSED') {
      stats.circuitState = 'OPEN';
      stats.circuitOpenedAt = new Date();
      console.warn(
        `[SmartLB] Instance ${instanceId} circuit OPEN (${stats.failureCount} failures)`
      );
    }

    this.algorithm.onFailure(instanceId, error);
  }

  private async performHealthChecks(): Promise<void> {
    for (const [instanceId, stats] of this.stats) {
      // ดึง instance info จาก ID
      // instanceId format: "service-hostname-port"
      const parts = instanceId.split('-');
      if (parts.length < 3) continue;

      const port = parseInt(parts[parts.length - 1]);
      const hostname = parts[parts.length - 2];

      try {
        const start = Date.now();
        await axios.get(`http://${hostname}:${port}/health`, { timeout: 5000 });
        const duration = Date.now() - start;

        stats.isHealthy = true;
        stats.lastHealthCheck = new Date();

        if (stats.circuitState === 'OPEN') {
          stats.circuitState = 'HALF_OPEN';
        }

        console.log(`[HealthCheck] ${instanceId}: healthy (${duration}ms)`);
      } catch {
        stats.isHealthy = false;
        stats.lastHealthCheck = new Date();
        console.warn(`[HealthCheck] ${instanceId}: unhealthy`);
      }
    }
  }

  getStats(): Map<string, InstanceStats> {
    return new Map(this.stats);
  }

  destroy(): void {
    clearInterval(this.healthCheckInterval);
    this.discovery.stopAllWatches();
  }
}
```

---

## 11. Express Middleware สำหรับ Service Discovery

```typescript
// packages/shared/src/middleware/discovery.middleware.ts

import { Request, Response, NextFunction } from 'express';
import { ConsulServiceDiscovery } from '../service-discovery/consul.client';
import { ServiceClient } from '../service-discovery/service.client';

// Attach service client to request
export function createDiscoveryMiddleware(
  discovery: ConsulServiceDiscovery
) {
  const client = new ServiceClient(discovery);

  return (req: Request, _res: Response, next: NextFunction) => {
    // @ts-expect-error — adding custom property
    req.services = client;
    next();
  };
}

// Health check endpoint
export function healthCheckHandler(
  _req: Request,
  res: Response
): void {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
  });
}

// Ready check — ตรวจสอบ dependencies
export function createReadyHandler(
  checks: Array<{
    name: string;
    check: () => Promise<boolean>;
  }>
) {
  return async (_req: Request, res: Response): Promise<void> => {
    const results = await Promise.allSettled(
      checks.map(async (c) => ({
        name: c.name,
        healthy: await c.check(),
      }))
    );

    const checkResults = results.map((r) =>
      r.status === 'fulfilled'
        ? r.value
        : { name: 'unknown', healthy: false }
    );

    const allHealthy = checkResults.every((r) => r.healthy);

    res.status(allHealthy ? 200 : 503).json({
      ready: allHealthy,
      checks: checkResults,
    });
  };
}
```

---

## 12. Full Service Bootstrap Example

```typescript
// packages/inventory-service/src/main.ts

import express from 'express';
import { ConsulServiceDiscovery } from '@shared/service-discovery/consul.client';
import { registerService } from '@shared/service-discovery/service.registry';
import {
  createDiscoveryMiddleware,
  createReadyHandler,
  healthCheckHandler,
} from '@shared/middleware/discovery.middleware';
import { Pool } from 'pg';

const PORT = parseInt(process.env.PORT ?? '3002');
const CONSUL_HOST = process.env.CONSUL_HOST ?? 'consul';
const CONSUL_PORT = parseInt(process.env.CONSUL_PORT ?? '8500');

async function bootstrap() {
  const app = express();
  app.use(express.json());

  // Database
  const pool = new Pool({
    host: process.env.DB_HOST ?? 'postgres',
    database: 'inventory_db',
    user: 'postgres',
    password: process.env.DB_PASSWORD,
  });

  // Service Discovery
  const discovery = new ConsulServiceDiscovery({
    host: CONSUL_HOST,
    port: CONSUL_PORT,
  });

  // Middleware
  app.use(createDiscoveryMiddleware(discovery));

  // Health endpoints
  app.get('/health', healthCheckHandler);
  app.get(
    '/health/ready',
    createReadyHandler([
      {
        name: 'database',
        check: async () => {
          const result = await pool.query('SELECT 1');
          return result.rowCount === 1;
        },
      },
      {
        name: 'consul',
        check: async () => {
          const instances = await discovery.getHealthyInstances('inventory-service');
          return true; // consul is reachable if we got here
        },
      },
    ])
  );

  // Routes
  app.post('/reservations', async (req, res) => {
    // ... business logic
    res.status(201).json({ reservationId: 'res-123' });
  });

  // Start server
  const server = app.listen(PORT, () => {
    console.log(`Inventory Service listening on port ${PORT}`);
  });

  // Register with Consul
  await registerService(discovery, {
    name: 'inventory-service',
    port: PORT,
    tags: ['v1', 'inventory'],
    meta: {
      weight: '100',
      region: process.env.REGION ?? 'ap-southeast-1',
    },
    healthCheckPath: '/health',
  });

  // Watch for other services we depend on
  discovery.watchService('order-service');
  discovery.on('service-changed', ({ serviceName, instances }) => {
    console.log(`[Discovery] ${serviceName}: ${instances.length} healthy instances`);
  });

  return server;
}

bootstrap().catch((err) => {
  console.error('Failed to start:', err);
  process.exit(1);
});
```

---

## 13. Docker Compose ครบ

```yaml
# docker-compose.full.yml
version: '3.8'

services:
  consul:
    image: hashicorp/consul:1.18
    command: >
      agent -server -bootstrap-expect=1 -ui -client=0.0.0.0
      -advertise=consul
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    volumes:
      - consul_data:/consul/data
    healthcheck:
      test: ["CMD", "consul", "members"]
      interval: 10s
      timeout: 5s
      retries: 5

  order-service:
    build: ./packages/order-service
    environment:
      PORT: 3001
      CONSUL_HOST: consul
      CONSUL_PORT: 8500
      DB_HOST: postgres
    labels:
      SERVICE_NAME: order-service
      SERVICE_TAGS: "v1,orders"
    depends_on:
      consul:
        condition: service_healthy
    deploy:
      replicas: 2

  inventory-service:
    build: ./packages/inventory-service
    environment:
      PORT: 3002
      CONSUL_HOST: consul
      CONSUL_PORT: 8500
      DB_HOST: postgres
    deploy:
      replicas: 3

  payment-service:
    build: ./packages/payment-service
    environment:
      PORT: 3003
      CONSUL_HOST: consul
      CONSUL_PORT: 8500
    deploy:
      replicas: 2

  # Envoy Proxy
  envoy:
    image: envoyproxy/envoy:v1.29-latest
    volumes:
      - ./envoy/envoy.yaml:/etc/envoy/envoy.yaml
    ports:
      - "9000:9000"  # Admin
      - "8080:8080"  # Proxy
    command: -c /etc/envoy/envoy.yaml --log-level info

volumes:
  consul_data:
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Consul Service Discovery** — วิธีลงทะเบียน Service, ทำ Health Check, และ Watch การเปลี่ยนแปลง

2. **Load Balancing Algorithms** — Round Robin, Least Connections, Weighted, Random, และ EWMA (latency-aware)

3. **Circuit Breaker Integration** — รวม Circuit Breaker เข้ากับ Load Balancer เพื่อป้องกัน cascading failures

4. **DNS-based Discovery** — ใช้ Consul DNS หรือ Kubernetes DNS สำหรับ service resolution

5. **Kubernetes-native Discovery** — ClusterIP Service, Headless Service, และ DNS patterns

6. **Smart Load Balancer** — Load Balancer ที่ตรวจสอบ health เองและปรับ routing อัตโนมัติ

**Best Practice:** ใน Production ควรใช้ **Server-side discovery** ผ่าน Kubernetes Service หรือ Service Mesh (Istio/Linkerd) แทนการ implement Client-side discovery เองเพื่อลดความซับซ้อน

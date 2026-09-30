# Part 74: AI/ML Integration ใน Microservices

## บทนำ

การนำ AI/ML มาใช้ใน Microservices Architecture เป็นเทรนด์ที่กำลังเติบโตอย่างรวดเร็ว บทนี้จะครอบคลุมตั้งแต่การ serve ML models, Feature Store Pattern, A/B Testing สำหรับ ML Models, Vector Databases ไปจนถึงการ integrate กับ LLM ผ่าน LangChain

---

## 1. ML Model Serving ด้วย TensorFlow Serving

### 1.1 โครงสร้าง TensorFlow Serving

```yaml
# docker-compose.yml สำหรับ TF Serving
version: '3.8'
services:
  tensorflow-serving:
    image: tensorflow/serving:2.13.0
    ports:
      - "8500:8500"  # gRPC
      - "8501:8501"  # REST API
    volumes:
      - ./models:/models
    environment:
      MODEL_NAME: recommendation_model
      MODEL_BASE_PATH: /models
    command: >
      --model_config_file=/models/model_config.config
      --monitoring_config_file=/models/monitoring_config.config
      --enable_batching=true
      --batching_parameters_file=/models/batching_parameters.txt

  model-proxy:
    build: ./model-proxy
    ports:
      - "3000:3000"
    environment:
      TF_SERVING_HOST: tensorflow-serving
      TF_SERVING_REST_PORT: 8501
      TF_SERVING_GRPC_PORT: 8500
```

```protobuf
// model_config.config
model_config_list {
  config {
    name: "recommendation_model"
    base_path: "/models/recommendation"
    model_platform: "tensorflow"
    model_version_policy {
      specific {
        versions: 1
        versions: 2
      }
    }
  }
  config {
    name: "sentiment_model"
    base_path: "/models/sentiment"
    model_platform: "tensorflow"
    model_version_policy {
      latest {
        num_versions: 2
      }
    }
  }
}
```

### 1.2 TypeScript Client สำหรับ TF Serving

```typescript
// src/ml/tf-serving-client.ts
import axios, { AxiosInstance } from 'axios';
import * as grpc from '@grpc/grpc-js';
import * as protoLoader from '@grpc/proto-loader';
import { Logger } from '../utils/logger';

interface TFServingConfig {
  host: string;
  restPort: number;
  grpcPort: number;
  timeout: number;
  maxRetries: number;
}

interface PredictionRequest {
  modelName: string;
  version?: number;
  instances: unknown[];
  signatureName?: string;
}

interface PredictionResponse {
  predictions: unknown[];
  modelSpec: {
    name: string;
    version: string;
    signatureName: string;
  };
}

export class TFServingClient {
  private httpClient: AxiosInstance;
  private grpcClient: any;
  private readonly config: TFServingConfig;
  private readonly logger: Logger;

  constructor(config: TFServingConfig) {
    this.config = config;
    this.logger = new Logger('TFServingClient');

    this.httpClient = axios.create({
      baseURL: `http://${config.host}:${config.restPort}`,
      timeout: config.timeout,
      headers: { 'Content-Type': 'application/json' },
    });

    this.initGrpcClient();
  }

  private initGrpcClient(): void {
    const PROTO_PATH = __dirname + '/proto/prediction_service.proto';
    const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
      keepCase: true,
      longs: String,
      enums: String,
      defaults: true,
      oneofs: true,
    });

    const protoDescriptor = grpc.loadPackageDefinition(packageDefinition) as any;
    const { PredictionService } = protoDescriptor.tensorflow.serving;

    this.grpcClient = new PredictionService(
      `${this.config.host}:${this.config.grpcPort}`,
      grpc.credentials.createInsecure()
    );
  }

  async predict(request: PredictionRequest): Promise<PredictionResponse> {
    const versionPath = request.version ? `/versions/${request.version}` : '';
    const url = `/v1/models/${request.modelName}${versionPath}:predict`;

    let lastError: Error | null = null;

    for (let attempt = 0; attempt <= this.config.maxRetries; attempt++) {
      try {
        const response = await this.httpClient.post<PredictionResponse>(url, {
          signature_name: request.signatureName || 'serving_default',
          instances: request.instances,
        });

        this.logger.info('Prediction successful', {
          model: request.modelName,
          version: request.version,
          instanceCount: request.instances.length,
        });

        return response.data;
      } catch (error: any) {
        lastError = error;
        this.logger.warn(`Prediction attempt ${attempt + 1} failed`, {
          error: error.message,
          model: request.modelName,
        });

        if (attempt < this.config.maxRetries) {
          await this.sleep(Math.pow(2, attempt) * 100);
        }
      }
    }

    throw new Error(`Prediction failed after ${this.config.maxRetries} retries: ${lastError?.message}`);
  }

  async getModelStatus(modelName: string): Promise<any> {
    const response = await this.httpClient.get(`/v1/models/${modelName}`);
    return response.data;
  }

  async getModelMetadata(modelName: string, version?: number): Promise<any> {
    const versionPath = version ? `/versions/${version}` : '';
    const response = await this.httpClient.get(
      `/v1/models/${modelName}${versionPath}/metadata`
    );
    return response.data;
  }

  private sleep(ms: number): Promise<void> {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }
}
```

---

## 2. NVIDIA Triton Inference Server

### 2.1 Triton Configuration

```yaml
# kubernetes/triton-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton-inference-server
  namespace: ml-serving
spec:
  replicas: 2
  selector:
    matchLabels:
      app: triton-server
  template:
    metadata:
      labels:
        app: triton-server
    spec:
      containers:
        - name: triton
          image: nvcr.io/nvidia/tritonserver:23.08-py3
          args:
            - tritonserver
            - --model-repository=s3://ml-models-bucket/triton-models
            - --strict-model-config=false
            - --allow-grpc=true
            - --grpc-port=8001
            - --allow-http=true
            - --http-port=8000
            - --allow-metrics=true
            - --metrics-port=8002
            - --log-verbose=1
          ports:
            - containerPort: 8000
              name: http
            - containerPort: 8001
              name: grpc
            - containerPort: 8002
              name: metrics
          resources:
            requests:
              cpu: "4"
              memory: "8Gi"
              nvidia.com/gpu: "1"
            limits:
              cpu: "8"
              memory: "16Gi"
              nvidia.com/gpu: "1"
          env:
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access-key-id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret-access-key
          livenessProbe:
            httpGet:
              path: /v2/health/live
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /v2/health/ready
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 5
```

```python
# src/ml/triton_client.py
import tritonclient.http as httpclient
import tritonclient.grpc as grpcclient
import numpy as np
from typing import List, Dict, Any, Optional
import logging
from dataclasses import dataclass
from concurrent.futures import ThreadPoolExecutor
import time

logger = logging.getLogger(__name__)

@dataclass
class ModelInput:
    name: str
    data: np.ndarray
    datatype: str

@dataclass
class TritonConfig:
    host: str
    http_port: int = 8000
    grpc_port: int = 8001
    timeout: int = 30
    max_connections: int = 10
    use_ssl: bool = False

class TritonInferenceClient:
    def __init__(self, config: TritonConfig):
        self.config = config
        self._http_client = None
        self._grpc_client = None
        self._executor = ThreadPoolExecutor(max_workers=config.max_connections)

    def _get_http_client(self) -> httpclient.InferenceServerClient:
        if self._http_client is None:
            self._http_client = httpclient.InferenceServerClient(
                url=f"{self.config.host}:{self.config.http_port}",
                verbose=False,
                ssl=self.config.use_ssl,
                connection_timeout=self.config.timeout,
                network_timeout=self.config.timeout,
            )
        return self._http_client

    def _get_grpc_client(self) -> grpcclient.InferenceServerClient:
        if self._grpc_client is None:
            self._grpc_client = grpcclient.InferenceServerClient(
                url=f"{self.config.host}:{self.config.grpc_port}",
                verbose=False,
                ssl=self.config.use_ssl,
            )
        return self._grpc_client

    def infer(
        self,
        model_name: str,
        inputs: List[ModelInput],
        output_names: List[str],
        model_version: str = "",
    ) -> Dict[str, np.ndarray]:
        client = self._get_grpc_client()

        triton_inputs = []
        for inp in inputs:
            triton_input = grpcclient.InferInput(inp.name, inp.data.shape, inp.datatype)
            triton_input.set_data_from_numpy(inp.data)
            triton_inputs.append(triton_input)

        triton_outputs = [
            grpcclient.InferRequestedOutput(name) for name in output_names
        ]

        start_time = time.time()
        response = client.infer(
            model_name=model_name,
            inputs=triton_inputs,
            outputs=triton_outputs,
            model_version=model_version,
        )
        latency_ms = (time.time() - start_time) * 1000

        logger.info(f"Triton inference completed: model={model_name}, latency={latency_ms:.2f}ms")

        return {
            name: response.as_numpy(name) for name in output_names
        }

    def batch_infer(
        self,
        model_name: str,
        batch_inputs: List[List[ModelInput]],
        output_names: List[str],
    ) -> List[Dict[str, np.ndarray]]:
        """Parallel batch inference"""
        futures = [
            self._executor.submit(self.infer, model_name, inputs, output_names)
            for inputs in batch_inputs
        ]
        return [f.result() for f in futures]

    def get_model_config(self, model_name: str) -> Dict[str, Any]:
        client = self._get_http_client()
        return client.get_model_config(model_name)

    def is_server_ready(self) -> bool:
        try:
            client = self._get_http_client()
            return client.is_server_ready()
        except Exception:
            return False
```

---

## 3. Feature Store Pattern

### 3.1 Feature Store ด้วย Feast

```python
# feature_store/feature_definitions.py
from feast import Entity, FeatureView, Field, FileSource, PushSource
from feast.types import Float32, Int64, String, Array
from datetime import timedelta

# Define entities
user_entity = Entity(
    name="user_id",
    join_keys=["user_id"],
    description="User identifier",
)

product_entity = Entity(
    name="product_id",
    join_keys=["product_id"],
    description="Product identifier",
)

# User behavioral features
user_behavior_source = FileSource(
    path="s3://feature-store/user_behavior/*.parquet",
    timestamp_field="event_timestamp",
    created_timestamp_column="created",
)

user_behavior_fv = FeatureView(
    name="user_behavior",
    entities=[user_entity],
    ttl=timedelta(days=7),
    schema=[
        Field(name="total_purchases_7d", dtype=Int64),
        Field(name="avg_order_value_7d", dtype=Float32),
        Field(name="category_affinity", dtype=Array(Float32)),
        Field(name="last_active_hours_ago", dtype=Float32),
        Field(name="churn_probability", dtype=Float32),
        Field(name="lifetime_value", dtype=Float32),
    ],
    online=True,
    source=user_behavior_source,
    tags={"team": "ml-platform", "version": "v2"},
)

# Product features
product_source = PushSource(
    name="product_push_source",
    batch_source=FileSource(
        path="s3://feature-store/products/*.parquet",
        timestamp_field="event_timestamp",
    ),
)

product_fv = FeatureView(
    name="product_features",
    entities=[product_entity],
    ttl=timedelta(hours=1),
    schema=[
        Field(name="price", dtype=Float32),
        Field(name="stock_quantity", dtype=Int64),
        Field(name="avg_rating", dtype=Float32),
        Field(name="purchase_count_24h", dtype=Int64),
        Field(name="embedding", dtype=Array(Float32)),
    ],
    online=True,
    source=product_source,
)
```

```typescript
// src/ml/feature-store-client.ts
import Redis from 'ioredis';
import { Pool } from 'pg';

interface FeatureRequest {
  entityName: string;
  entityIds: string[];
  featureNames: string[];
}

interface FeatureResult {
  entityId: string;
  features: Record<string, number | number[] | string | null>;
  timestamp: Date;
}

export class FeatureStoreClient {
  private redis: Redis;
  private db: Pool;
  private readonly TTL = 3600; // 1 hour default

  constructor(redisUrl: string, dbConfig: any) {
    this.redis = new Redis(redisUrl, {
      maxRetriesPerRequest: 3,
      enableReadyCheck: true,
      lazyConnect: true,
    });

    this.db = new Pool(dbConfig);
  }

  async getOnlineFeatures(request: FeatureRequest): Promise<FeatureResult[]> {
    const results: FeatureResult[] = [];
    const cacheMisses: string[] = [];

    // Try Redis cache first (L1)
    for (const entityId of request.entityIds) {
      const cacheKey = this.buildCacheKey(request.entityName, entityId, request.featureNames);
      const cached = await this.redis.get(cacheKey);

      if (cached) {
        results.push(JSON.parse(cached));
      } else {
        cacheMisses.push(entityId);
      }
    }

    // Fetch cache misses from database (L2)
    if (cacheMisses.length > 0) {
      const dbResults = await this.fetchFromDatabase({
        ...request,
        entityIds: cacheMisses,
      });

      for (const result of dbResults) {
        const cacheKey = this.buildCacheKey(request.entityName, result.entityId, request.featureNames);
        await this.redis.setex(cacheKey, this.TTL, JSON.stringify(result));
        results.push(result);
      }
    }

    return results;
  }

  async pushFeatures(
    entityName: string,
    entityId: string,
    features: Record<string, any>,
    timestamp: Date = new Date()
  ): Promise<void> {
    // Write to database
    await this.db.query(
      `INSERT INTO feature_store.${entityName}_features 
       (entity_id, features, event_timestamp, created_at)
       VALUES ($1, $2, $3, NOW())
       ON CONFLICT (entity_id) DO UPDATE 
       SET features = $2, event_timestamp = $3, updated_at = NOW()`,
      [entityId, JSON.stringify(features), timestamp]
    );

    // Invalidate cache
    const keys = await this.redis.keys(`feature:${entityName}:${entityId}:*`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }

    // Write to Redis for real-time serving
    const cacheKey = `feature:${entityName}:${entityId}:latest`;
    await this.redis.setex(
      cacheKey,
      this.TTL,
      JSON.stringify({ entityId, features, timestamp })
    );
  }

  private async fetchFromDatabase(request: FeatureRequest): Promise<FeatureResult[]> {
    const placeholders = request.entityIds.map((_, i) => `$${i + 1}`).join(',');
    const featureSelector = request.featureNames
      .map((f) => `features->>'${f}' as "${f}"`)
      .join(', ');

    const query = `
      SELECT entity_id, ${featureSelector}, event_timestamp
      FROM feature_store.${request.entityName}_features
      WHERE entity_id IN (${placeholders})
      AND event_timestamp > NOW() - INTERVAL '7 days'
    `;

    const { rows } = await this.db.query(query, request.entityIds);

    return rows.map((row) => ({
      entityId: row.entity_id,
      features: request.featureNames.reduce((acc, name) => {
        acc[name] = row[name] ? JSON.parse(row[name]) : null;
        return acc;
      }, {} as Record<string, any>),
      timestamp: row.event_timestamp,
    }));
  }

  private buildCacheKey(entityName: string, entityId: string, featureNames: string[]): string {
    const sortedFeatures = [...featureNames].sort().join(',');
    return `feature:${entityName}:${entityId}:${sortedFeatures}`;
  }
}
```

---

## 4. A/B Testing สำหรับ ML Models

### 4.1 ML Model A/B Testing Framework

```typescript
// src/ml/ab-testing/model-ab-test.ts
import { createHash } from 'crypto';
import { Redis } from 'ioredis';
import { MetricsCollector } from '../metrics/collector';

interface ModelVariant {
  name: string;
  modelName: string;
  modelVersion: number;
  weight: number; // percentage 0-100
  isControl: boolean;
}

interface ABTestConfig {
  testId: string;
  name: string;
  variants: ModelVariant[];
  startDate: Date;
  endDate: Date;
  metrics: string[];
  successCriteria: {
    metric: string;
    minimumDetectableEffect: number;
    confidence: number;
  };
}

interface ABTestResult {
  variant: ModelVariant;
  prediction: unknown;
  latencyMs: number;
  testId: string;
  assignmentReason: string;
}

export class MLModelABTest {
  private redis: Redis;
  private metrics: MetricsCollector;

  constructor(redis: Redis, metrics: MetricsCollector) {
    this.redis = redis;
    this.metrics = metrics;
  }

  async assignVariant(
    testConfig: ABTestConfig,
    userId: string,
    context?: Record<string, any>
  ): Promise<ModelVariant> {
    // Check for forced assignment (for QA/testing)
    const forcedVariant = await this.getForcedAssignment(testConfig.testId, userId);
    if (forcedVariant) {
      return forcedVariant;
    }

    // Check if user already assigned
    const cachedAssignment = await this.getCachedAssignment(testConfig.testId, userId);
    if (cachedAssignment) {
      return cachedAssignment;
    }

    // Deterministic assignment based on user hash
    const variant = this.deterministicAssign(testConfig.variants, testConfig.testId, userId);

    // Cache assignment for consistency
    await this.cacheAssignment(testConfig.testId, userId, variant);

    // Track assignment event
    await this.metrics.increment('ab_test.assignment', {
      test_id: testConfig.testId,
      variant: variant.name,
      user_id: userId,
    });

    return variant;
  }

  private deterministicAssign(
    variants: ModelVariant[],
    testId: string,
    userId: string
  ): ModelVariant {
    const hash = createHash('sha256')
      .update(`${testId}:${userId}`)
      .digest('hex');

    const bucket = parseInt(hash.substring(0, 8), 16) % 100;

    let cumulative = 0;
    for (const variant of variants) {
      cumulative += variant.weight;
      if (bucket < cumulative) {
        return variant;
      }
    }

    // Fallback to control
    return variants.find((v) => v.isControl) || variants[0];
  }

  async trackPrediction(
    testId: string,
    userId: string,
    variantName: string,
    prediction: unknown,
    latencyMs: number,
    outcome?: Record<string, number>
  ): Promise<void> {
    const key = `ab_test:${testId}:${variantName}`;
    const timestamp = Date.now();

    await this.redis.zadd(
      `${key}:predictions`,
      timestamp,
      JSON.stringify({ userId, prediction, latencyMs, timestamp })
    );

    // Update metrics
    await this.redis.hincrby(`${key}:stats`, 'prediction_count', 1);
    await this.redis.hincrbyfloat(`${key}:stats`, 'total_latency', latencyMs);

    if (outcome) {
      for (const [metric, value] of Object.entries(outcome)) {
        await this.redis.hincrbyfloat(`${key}:outcomes`, metric, value);
        await this.redis.hincrby(`${key}:outcomes`, `${metric}_count`, 1);
      }
    }
  }

  async getTestResults(testId: string, variants: string[]): Promise<any> {
    const results: Record<string, any> = {};

    for (const variant of variants) {
      const key = `ab_test:${testId}:${variant}`;
      const stats = await this.redis.hgetall(`${key}:stats`);
      const outcomes = await this.redis.hgetall(`${key}:outcomes`);

      const predCount = parseInt(stats.prediction_count || '0');
      const totalLatency = parseFloat(stats.total_latency || '0');

      results[variant] = {
        predictionCount: predCount,
        avgLatencyMs: predCount > 0 ? totalLatency / predCount : 0,
        outcomes: this.computeOutcomeStats(outcomes),
      };
    }

    return {
      testId,
      results,
      winner: this.computeWinner(results),
      statisticalSignificance: this.computeSignificance(results),
    };
  }

  private computeOutcomeStats(outcomes: Record<string, string>): Record<string, number> {
    const stats: Record<string, number> = {};
    const metrics = new Set<string>();

    for (const key of Object.keys(outcomes)) {
      if (!key.endsWith('_count')) {
        metrics.add(key);
      }
    }

    for (const metric of metrics) {
      const total = parseFloat(outcomes[metric] || '0');
      const count = parseInt(outcomes[`${metric}_count`] || '0');
      stats[metric] = count > 0 ? total / count : 0;
    }

    return stats;
  }

  private computeWinner(results: Record<string, any>): string | null {
    // Simplified: pick variant with best conversion rate
    let bestVariant: string | null = null;
    let bestValue = -Infinity;

    for (const [variant, data] of Object.entries(results)) {
      const convRate = data.outcomes?.conversion_rate || 0;
      if (convRate > bestValue) {
        bestValue = convRate;
        bestVariant = variant;
      }
    }

    return bestVariant;
  }

  private computeSignificance(results: Record<string, any>): number {
    // Simplified z-test significance computation
    const variants = Object.values(results);
    if (variants.length < 2) return 0;

    const control = variants.find((v: any) => v.isControl) || variants[0];
    const treatment = variants.find((v: any) => !v.isControl) || variants[1];

    const p1 = control.outcomes?.conversion_rate || 0;
    const p2 = treatment.outcomes?.conversion_rate || 0;
    const n1 = control.predictionCount || 1;
    const n2 = treatment.predictionCount || 1;

    const pPool = (p1 * n1 + p2 * n2) / (n1 + n2);
    const se = Math.sqrt(pPool * (1 - pPool) * (1 / n1 + 1 / n2));
    const z = Math.abs((p2 - p1) / (se || 1));

    // Convert z-score to p-value (approximate)
    return 1 - Math.exp(-0.717 * z - 0.416 * z * z);
  }

  private async getForcedAssignment(testId: string, userId: string): Promise<ModelVariant | null> {
    const forced = await this.redis.get(`ab_test:forced:${testId}:${userId}`);
    return forced ? JSON.parse(forced) : null;
  }

  private async getCachedAssignment(testId: string, userId: string): Promise<ModelVariant | null> {
    const cached = await this.redis.get(`ab_test:assignment:${testId}:${userId}`);
    return cached ? JSON.parse(cached) : null;
  }

  private async cacheAssignment(
    testId: string,
    userId: string,
    variant: ModelVariant
  ): Promise<void> {
    await this.redis.setex(
      `ab_test:assignment:${testId}:${userId}`,
      86400 * 30, // 30 days
      JSON.stringify(variant)
    );
  }
}
```

---

## 5. Model Versioning และ Registry

### 5.1 MLflow Integration

```python
# src/ml/model_registry.py
import mlflow
import mlflow.sklearn
import mlflow.pytorch
from mlflow.tracking import MlflowClient
from mlflow.entities import ViewType
from typing import Optional, Dict, Any, List
import logging
import json
import hashlib

logger = logging.getLogger(__name__)

class ModelRegistry:
    def __init__(self, tracking_uri: str, registry_uri: str):
        mlflow.set_tracking_uri(tracking_uri)
        self.client = MlflowClient(tracking_uri=tracking_uri, registry_uri=registry_uri)
        
    def register_model(
        self,
        model_name: str,
        model_uri: str,
        description: str = "",
        tags: Optional[Dict[str, str]] = None,
    ) -> str:
        """Register a model and return the version"""
        result = mlflow.register_model(model_uri, model_name)
        
        if description:
            self.client.update_registered_model(
                name=model_name,
                description=description,
            )
        
        if tags:
            for key, value in tags.items():
                self.client.set_registered_model_tag(model_name, key, value)
        
        logger.info(f"Registered model {model_name} version {result.version}")
        return result.version

    def promote_to_staging(self, model_name: str, version: str) -> None:
        self.client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage="Staging",
            archive_existing_versions=False,
        )
        logger.info(f"Promoted {model_name} v{version} to Staging")

    def promote_to_production(
        self,
        model_name: str,
        version: str,
        archive_previous: bool = True,
    ) -> None:
        self.client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage="Production",
            archive_existing_versions=archive_previous,
        )
        logger.info(f"Promoted {model_name} v{version} to Production")

    def get_production_model(self, model_name: str) -> Any:
        """Load the current production model"""
        model_uri = f"models:/{model_name}/Production"
        return mlflow.pyfunc.load_model(model_uri)

    def get_model_versions(
        self,
        model_name: str,
        stages: Optional[List[str]] = None,
    ) -> List[Dict[str, Any]]:
        filter_str = f"name='{model_name}'"
        versions = self.client.search_model_versions(filter_str)
        
        if stages:
            versions = [v for v in versions if v.current_stage in stages]
        
        return [
            {
                "version": v.version,
                "stage": v.current_stage,
                "status": v.status,
                "created_at": v.creation_timestamp,
                "run_id": v.run_id,
                "source": v.source,
                "description": v.description,
            }
            for v in versions
        ]

    def compare_models(
        self,
        model_name: str,
        version_a: str,
        version_b: str,
    ) -> Dict[str, Any]:
        """Compare metrics between two model versions"""
        def get_run_metrics(version: str) -> Dict[str, float]:
            mv = self.client.get_model_version(model_name, version)
            run = self.client.get_run(mv.run_id)
            return run.data.metrics

        metrics_a = get_run_metrics(version_a)
        metrics_b = get_run_metrics(version_b)
        
        comparison = {}
        all_metrics = set(metrics_a.keys()) | set(metrics_b.keys())
        
        for metric in all_metrics:
            val_a = metrics_a.get(metric)
            val_b = metrics_b.get(metric)
            improvement = None
            if val_a and val_b:
                improvement = ((val_b - val_a) / val_a) * 100
            
            comparison[metric] = {
                "version_a": val_a,
                "version_b": val_b,
                "improvement_pct": improvement,
            }
        
        return {
            "model_name": model_name,
            "version_a": version_a,
            "version_b": version_b,
            "metrics": comparison,
        }
```

---

## 6. Online vs Offline Inference

### 6.1 Online Inference Service

```typescript
// src/ml/inference/online-inference.ts
import { TFServingClient } from '../tf-serving-client';
import { FeatureStoreClient } from '../feature-store-client';
import { MetricsCollector } from '../metrics/collector';
import { Cache } from '../cache';

interface OnlineInferenceConfig {
  modelName: string;
  modelVersion?: number;
  maxBatchSize: number;
  batchTimeoutMs: number;
  cacheEnabled: boolean;
  cacheTtlSeconds: number;
}

interface InferenceRequest {
  requestId: string;
  userId: string;
  contextFeatures: Record<string, any>;
}

export class OnlineInferenceService {
  private pendingBatch: Array<{
    request: InferenceRequest;
    resolve: (value: any) => void;
    reject: (error: Error) => void;
  }> = [];
  private batchTimer: NodeJS.Timeout | null = null;

  constructor(
    private tfClient: TFServingClient,
    private featureStore: FeatureStoreClient,
    private cache: Cache,
    private metrics: MetricsCollector,
    private config: OnlineInferenceConfig
  ) {}

  async infer(request: InferenceRequest): Promise<any> {
    // Check cache
    if (this.config.cacheEnabled) {
      const cacheKey = this.buildCacheKey(request);
      const cached = await this.cache.get(cacheKey);
      if (cached) {
        this.metrics.increment('inference.cache_hit');
        return cached;
      }
    }

    // Add to batch queue
    return new Promise((resolve, reject) => {
      this.pendingBatch.push({ request, resolve, reject });

      if (this.pendingBatch.length >= this.config.maxBatchSize) {
        this.flushBatch();
      } else if (!this.batchTimer) {
        this.batchTimer = setTimeout(() => this.flushBatch(), this.config.batchTimeoutMs);
      }
    });
  }

  private async flushBatch(): Promise<void> {
    if (this.batchTimer) {
      clearTimeout(this.batchTimer);
      this.batchTimer = null;
    }

    const batch = this.pendingBatch.splice(0, this.config.maxBatchSize);
    if (batch.length === 0) return;

    const startTime = Date.now();

    try {
      // Fetch features for all users in batch
      const userIds = batch.map((item) => item.request.userId);
      const features = await this.featureStore.getOnlineFeatures({
        entityName: 'user',
        entityIds: userIds,
        featureNames: ['embedding', 'behavior_vector', 'segment'],
      });

      // Build instances for batch prediction
      const instances = batch.map((item, idx) => ({
        ...features[idx]?.features,
        ...item.request.contextFeatures,
      }));

      // Batch predict
      const predictions = await this.tfClient.predict({
        modelName: this.config.modelName,
        version: this.config.modelVersion,
        instances,
      });

      // Resolve individual requests
      for (let i = 0; i < batch.length; i++) {
        const prediction = predictions.predictions[i];
        
        if (this.config.cacheEnabled) {
          const cacheKey = this.buildCacheKey(batch[i].request);
          await this.cache.set(cacheKey, prediction, this.config.cacheTtlSeconds);
        }

        batch[i].resolve(prediction);
      }

      const latency = Date.now() - startTime;
      this.metrics.histogram('inference.batch_latency_ms', latency, {
        model: this.config.modelName,
        batch_size: batch.length.toString(),
      });

    } catch (error: any) {
      for (const item of batch) {
        item.reject(error);
      }
      this.metrics.increment('inference.batch_error');
    }
  }

  private buildCacheKey(request: InferenceRequest): string {
    const contextHash = JSON.stringify(request.contextFeatures);
    return `inference:${this.config.modelName}:${request.userId}:${contextHash}`;
  }
}
```

### 6.2 Offline Batch Inference

```python
# src/ml/batch_inference.py
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import ArrayType, FloatType
import mlflow.pyfunc
import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import boto3
import logging

logger = logging.getLogger(__name__)

class BatchInferencePipeline:
    def __init__(self, spark: SparkSession, model_name: str, model_stage: str = "Production"):
        self.spark = spark
        self.model_name = model_name
        self.model_stage = model_stage
        self.s3_client = boto3.client('s3')

    def run(
        self,
        input_path: str,
        output_path: str,
        date: datetime,
        partition_size: int = 1000,
    ) -> None:
        logger.info(f"Starting batch inference for {self.model_name} on {date.date()}")

        # Load model
        model_uri = f"models:/{self.model_name}/{self.model_stage}"
        model = mlflow.pyfunc.load_model(model_uri)

        # Load features
        features_df = self.spark.read.parquet(
            f"{input_path}/date={date.strftime('%Y-%m-%d')}"
        )

        feature_columns = [c for c in features_df.columns if c != 'user_id']

        # UDF for batch prediction
        model_broadcast = self.spark.sparkContext.broadcast(model)

        @F.pandas_udf(ArrayType(FloatType()))
        def predict_batch(features_iter):
            m = model_broadcast.value
            for features_pd in features_iter:
                predictions = m.predict(features_pd)
                yield pd.Series(predictions.tolist())

        # Run predictions
        predictions_df = features_df.withColumn(
            'predictions',
            predict_batch(*[F.col(c) for c in feature_columns])
        ).withColumn(
            'inference_timestamp',
            F.lit(datetime.utcnow().isoformat())
        ).withColumn(
            'model_version',
            F.lit(self.model_stage)
        )

        # Write results
        output_date = date.strftime('%Y-%m-%d')
        predictions_df.repartition(partition_size).write.mode('overwrite').parquet(
            f"{output_path}/date={output_date}"
        )

        count = predictions_df.count()
        logger.info(f"Batch inference complete: {count} predictions written to {output_path}")
        
        # Write metrics
        self._write_metrics({
            'date': output_date,
            'model_name': self.model_name,
            'model_stage': self.model_stage,
            'record_count': count,
            'status': 'success',
        })

    def _write_metrics(self, metrics: dict) -> None:
        self.s3_client.put_object(
            Bucket='ml-metrics',
            Key=f"batch-inference/{self.model_name}/{metrics['date']}/metrics.json",
            Body=str(metrics).encode(),
        )
```

---

## 7. Vector Databases สำหรับ Semantic Search

### 7.1 pgvector Integration

```sql
-- migrations/001_create_vector_tables.sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE product_embeddings (
    id BIGSERIAL PRIMARY KEY,
    product_id VARCHAR(100) NOT NULL UNIQUE,
    title TEXT NOT NULL,
    description TEXT,
    embedding vector(1536) NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- HNSW index for approximate nearest neighbor search
CREATE INDEX product_embeddings_embedding_idx 
ON product_embeddings 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Exact search index (IVFFlat)
CREATE INDEX product_embeddings_ivfflat_idx
ON product_embeddings
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

CREATE TABLE document_chunks (
    id BIGSERIAL PRIMARY KEY,
    document_id VARCHAR(100) NOT NULL,
    chunk_index INTEGER NOT NULL,
    content TEXT NOT NULL,
    embedding vector(1536) NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(document_id, chunk_index)
);

CREATE INDEX doc_chunks_embedding_idx
ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

```typescript
// src/ml/vector-db/pgvector-client.ts
import { Pool, PoolClient } from 'pg';
import OpenAI from 'openai';

interface VectorSearchOptions {
  limit?: number;
  threshold?: number;
  filter?: Record<string, any>;
  useExactSearch?: boolean;
}

interface SearchResult {
  id: string;
  content: string;
  similarity: number;
  metadata: Record<string, any>;
}

export class PgVectorClient {
  private pool: Pool;
  private openai: OpenAI;

  constructor(dbConfig: any, openaiApiKey: string) {
    this.pool = new Pool(dbConfig);
    this.openai = new OpenAI({ apiKey: openaiApiKey });
  }

  async generateEmbedding(text: string): Promise<number[]> {
    const response = await this.openai.embeddings.create({
      model: 'text-embedding-ada-002',
      input: text.replace(/\n/g, ' '),
    });
    return response.data[0].embedding;
  }

  async upsertProductEmbedding(
    productId: string,
    title: string,
    description: string,
    metadata: Record<string, any> = {}
  ): Promise<void> {
    const text = `${title} ${description}`;
    const embedding = await this.generateEmbedding(text);

    await this.pool.query(
      `INSERT INTO product_embeddings (product_id, title, description, embedding, metadata)
       VALUES ($1, $2, $3, $4::vector, $5)
       ON CONFLICT (product_id) DO UPDATE
       SET title = $2, description = $3, embedding = $4::vector, metadata = $5, updated_at = NOW()`,
      [productId, title, description, JSON.stringify(embedding), JSON.stringify(metadata)]
    );
  }

  async semanticSearch(
    query: string,
    options: VectorSearchOptions = {}
  ): Promise<SearchResult[]> {
    const { limit = 10, threshold = 0.7, filter, useExactSearch = false } = options;

    const queryEmbedding = await this.generateEmbedding(query);

    const operator = useExactSearch ? '<->' : '<=>';  // L2 vs cosine
    const orderBy = useExactSearch
      ? `embedding <-> $1::vector`
      : `1 - (embedding <=> $1::vector) DESC`;

    let whereClause = `1 - (embedding <=> $1::vector) >= $3`;
    const params: any[] = [JSON.stringify(queryEmbedding), limit, threshold];

    if (filter) {
      const filterConditions = Object.entries(filter)
        .map(([key, value], idx) => `metadata->>'${key}' = $${idx + 4}`)
        .join(' AND ');
      whereClause += ` AND ${filterConditions}`;
      params.push(...Object.values(filter));
    }

    const query_sql = `
      SELECT 
        product_id as id,
        title,
        description as content,
        metadata,
        1 - (embedding <=> $1::vector) as similarity
      FROM product_embeddings
      WHERE ${whereClause}
      ORDER BY ${orderBy}
      LIMIT $2
    `;

    const { rows } = await this.pool.query(query_sql, params);

    return rows.map((row) => ({
      id: row.id,
      content: row.content,
      similarity: parseFloat(row.similarity),
      metadata: row.metadata,
    }));
  }

  async hybridSearch(
    query: string,
    keywords: string[],
    options: VectorSearchOptions = {}
  ): Promise<SearchResult[]> {
    const queryEmbedding = await this.generateEmbedding(query);
    const { limit = 10, threshold = 0.6 } = options;

    const tsQuery = keywords.join(' & ');

    const { rows } = await this.pool.query(
      `WITH vector_results AS (
        SELECT 
          product_id,
          title,
          description,
          metadata,
          1 - (embedding <=> $1::vector) AS vector_score,
          0 AS text_score
        FROM product_embeddings
        WHERE 1 - (embedding <=> $1::vector) >= $3
      ),
      text_results AS (
        SELECT
          product_id,
          title,
          description,
          metadata,
          0 AS vector_score,
          ts_rank(to_tsvector('english', title || ' ' || description), to_tsquery($4)) AS text_score
        FROM product_embeddings
        WHERE to_tsvector('english', title || ' ' || description) @@ to_tsquery($4)
      ),
      combined AS (
        SELECT product_id, title, description, metadata,
          MAX(vector_score) AS vector_score,
          MAX(text_score) AS text_score
        FROM (SELECT * FROM vector_results UNION ALL SELECT * FROM text_results) t
        GROUP BY product_id, title, description, metadata
      )
      SELECT 
        product_id as id,
        description as content,
        metadata,
        (0.7 * vector_score + 0.3 * text_score) AS similarity
      FROM combined
      ORDER BY similarity DESC
      LIMIT $2`,
      [JSON.stringify(queryEmbedding), limit, threshold, tsQuery]
    );

    return rows.map((row) => ({
      id: row.id,
      content: row.content,
      similarity: parseFloat(row.similarity),
      metadata: row.metadata,
    }));
  }
}
```

---

## 8. LLM Integration ด้วย LangChain

### 8.1 LangChain Service

```typescript
// src/ml/llm/langchain-service.ts
import { ChatOpenAI } from '@langchain/openai';
import { HumanMessage, SystemMessage, AIMessage } from '@langchain/core/messages';
import { PromptTemplate, ChatPromptTemplate } from '@langchain/core/prompts';
import { RunnableSequence, RunnablePassthrough } from '@langchain/core/runnables';
import { StringOutputParser } from '@langchain/core/output_parsers';
import { StructuredOutputParser } from 'langchain/output_parsers';
import { z } from 'zod';
import { PgVectorClient } from '../vector-db/pgvector-client';
import { Redis } from 'ioredis';

interface RAGConfig {
  modelName: string;
  temperature: number;
  maxTokens: number;
  vectorSearchLimit: number;
  cacheEnabled: boolean;
}

export class LangChainRAGService {
  private llm: ChatOpenAI;
  private vectorDb: PgVectorClient;
  private redis: Redis;

  constructor(
    openaiApiKey: string,
    private config: RAGConfig,
    vectorDb: PgVectorClient,
    redis: Redis
  ) {
    this.llm = new ChatOpenAI({
      openAIApiKey: openaiApiKey,
      modelName: config.modelName,
      temperature: config.temperature,
      maxTokens: config.maxTokens,
      streaming: true,
    });

    this.vectorDb = vectorDb;
    this.redis = redis;
  }

  async answerQuestion(
    question: string,
    sessionId: string,
    context?: string
  ): Promise<string> {
    const cacheKey = `llm:rag:${sessionId}:${question}`;
    
    if (this.config.cacheEnabled) {
      const cached = await this.redis.get(cacheKey);
      if (cached) return cached;
    }

    // Retrieve relevant context
    const relevantDocs = await this.vectorDb.semanticSearch(question, {
      limit: this.config.vectorSearchLimit,
      threshold: 0.7,
    });

    const contextText = relevantDocs
      .map((doc, idx) => `[${idx + 1}] ${doc.content}`)
      .join('\n\n');

    const systemPrompt = `You are a helpful assistant. Answer questions based on the provided context.
If the context doesn't contain enough information, say so clearly.
Always cite the source numbers when referencing context.

Context:
${contextText}

Additional context: ${context || 'None'}`;

    const chain = RunnableSequence.from([
      ChatPromptTemplate.fromMessages([
        ['system', systemPrompt],
        ['human', '{question}'],
      ]),
      this.llm,
      new StringOutputParser(),
    ]);

    const answer = await chain.invoke({ question });

    if (this.config.cacheEnabled) {
      await this.redis.setex(cacheKey, 300, answer);
    }

    return answer;
  }

  async extractStructuredData<T extends z.ZodSchema>(
    text: string,
    schema: T,
    instructions?: string
  ): Promise<z.infer<T>> {
    const parser = StructuredOutputParser.fromZodSchema(schema);
    const formatInstructions = parser.getFormatInstructions();

    const prompt = ChatPromptTemplate.fromMessages([
      ['system', `Extract structured information from the text.
${instructions || ''}

${formatInstructions}`],
      ['human', '{text}'],
    ]);

    const chain = RunnableSequence.from([prompt, this.llm, parser]);
    return await chain.invoke({ text });
  }

  async streamAnswer(
    question: string,
    contextDocs: string[],
    onChunk: (chunk: string) => void
  ): Promise<void> {
    const contextText = contextDocs.join('\n\n');

    const messages = [
      new SystemMessage(`Answer based on context:\n\n${contextText}`),
      new HumanMessage(question),
    ];

    const stream = await this.llm.stream(messages);

    for await (const chunk of stream) {
      if (chunk.content) {
        onChunk(chunk.content as string);
      }
    }
  }

  async createSummaryChain(documents: string[]): Promise<string> {
    const summaryPrompt = PromptTemplate.fromTemplate(`
Summarize the following documents concisely:

{documents}

Provide a structured summary with key points.`);

    const chain = RunnableSequence.from([
      summaryPrompt,
      this.llm,
      new StringOutputParser(),
    ]);

    return await chain.invoke({ documents: documents.join('\n\n---\n\n') });
  }
}
```

### 8.2 Recommendation Service ด้วย ML

```typescript
// src/ml/recommendation/recommendation-service.ts
import { TFServingClient } from '../tf-serving-client';
import { FeatureStoreClient } from '../feature-store-client';
import { PgVectorClient } from '../vector-db/pgvector-client';
import { MLModelABTest } from '../ab-testing/model-ab-test';

interface RecommendationRequest {
  userId: string;
  contextItemId?: string;
  limit: number;
  strategy: 'collaborative' | 'content' | 'hybrid' | 'semantic';
}

interface RecommendationResult {
  productId: string;
  score: number;
  strategy: string;
  explanation?: string;
}

export class RecommendationService {
  constructor(
    private tfClient: TFServingClient,
    private featureStore: FeatureStoreClient,
    private vectorDb: PgVectorClient,
    private abTest: MLModelABTest,
    private testConfig: any
  ) {}

  async getRecommendations(request: RecommendationRequest): Promise<RecommendationResult[]> {
    // Assign A/B test variant
    const variant = await this.abTest.assignVariant(this.testConfig, request.userId);
    const strategy = request.strategy || 'hybrid';

    let results: RecommendationResult[];

    switch (strategy) {
      case 'collaborative':
        results = await this.collaborativeFilter(request, variant);
        break;
      case 'content':
        results = await this.contentBased(request);
        break;
      case 'semantic':
        results = await this.semanticRecommendation(request);
        break;
      case 'hybrid':
      default:
        results = await this.hybridRecommendation(request, variant);
    }

    // Track prediction for A/B test
    await this.abTest.trackPrediction(
      this.testConfig.testId,
      request.userId,
      variant.name,
      results,
      0 // latency tracked separately
    );

    return results.slice(0, request.limit);
  }

  private async collaborativeFilter(
    request: RecommendationRequest,
    variant: any
  ): Promise<RecommendationResult[]> {
    const userFeatures = await this.featureStore.getOnlineFeatures({
      entityName: 'user',
      entityIds: [request.userId],
      featureNames: ['embedding', 'interaction_history'],
    });

    const predictions = await this.tfClient.predict({
      modelName: variant.modelName,
      version: variant.modelVersion,
      instances: [userFeatures[0]?.features],
    });

    const scores = predictions.predictions[0] as number[];

    return scores
      .map((score, idx) => ({
        productId: `product_${idx}`,
        score,
        strategy: 'collaborative',
      }))
      .sort((a, b) => b.score - a.score);
  }

  private async contentBased(
    request: RecommendationRequest
  ): Promise<RecommendationResult[]> {
    if (!request.contextItemId) return [];

    // Use vector similarity for content-based
    const results = await this.vectorDb.semanticSearch(
      `product similar to ${request.contextItemId}`,
      { limit: request.limit * 2 }
    );

    return results.map((r) => ({
      productId: r.id,
      score: r.similarity,
      strategy: 'content',
    }));
  }

  private async semanticRecommendation(
    request: RecommendationRequest
  ): Promise<RecommendationResult[]> {
    const userBehavior = await this.featureStore.getOnlineFeatures({
      entityName: 'user',
      entityIds: [request.userId],
      featureNames: ['recent_searches', 'interest_tags'],
    });

    const interestQuery = (userBehavior[0]?.features?.interest_tags as string[])?.join(' ') || '';

    const results = await this.vectorDb.semanticSearch(interestQuery, {
      limit: request.limit,
      threshold: 0.6,
    });

    return results.map((r) => ({
      productId: r.id,
      score: r.similarity,
      strategy: 'semantic',
    }));
  }

  private async hybridRecommendation(
    request: RecommendationRequest,
    variant: any
  ): Promise<RecommendationResult[]> {
    const [collaborative, semantic] = await Promise.all([
      this.collaborativeFilter(request, variant),
      this.semanticRecommendation(request),
    ]);

    // Merge with weighted scores
    const scoreMap = new Map<string, number>();

    for (const r of collaborative) {
      scoreMap.set(r.productId, (scoreMap.get(r.productId) || 0) + r.score * 0.6);
    }
    for (const r of semantic) {
      scoreMap.set(r.productId, (scoreMap.get(r.productId) || 0) + r.score * 0.4);
    }

    return Array.from(scoreMap.entries())
      .map(([productId, score]) => ({ productId, score, strategy: 'hybrid' }))
      .sort((a, b) => b.score - a.score);
  }
}
```

---

## 9. Model Monitoring และ Drift Detection

```python
# src/ml/monitoring/drift_detector.py
import numpy as np
import pandas as pd
from scipy import stats
from typing import Dict, List, Optional, Tuple
import logging
from datetime import datetime, timedelta
import boto3
import json

logger = logging.getLogger(__name__)

class DataDriftDetector:
    """Detect statistical drift in input features"""
    
    def __init__(
        self,
        reference_data: pd.DataFrame,
        significance_level: float = 0.05,
        drift_threshold: float = 0.1,
    ):
        self.reference_data = reference_data
        self.significance_level = significance_level
        self.drift_threshold = drift_threshold
        self.reference_stats = self._compute_stats(reference_data)

    def _compute_stats(self, data: pd.DataFrame) -> Dict[str, Dict]:
        stats_dict = {}
        for col in data.columns:
            if data[col].dtype in [np.float64, np.int64]:
                stats_dict[col] = {
                    'mean': float(data[col].mean()),
                    'std': float(data[col].std()),
                    'min': float(data[col].min()),
                    'max': float(data[col].max()),
                    'p25': float(data[col].quantile(0.25)),
                    'p50': float(data[col].quantile(0.50)),
                    'p75': float(data[col].quantile(0.75)),
                    'histogram': np.histogram(data[col].dropna(), bins=20)[0].tolist(),
                }
        return stats_dict

    def detect_drift(self, current_data: pd.DataFrame) -> Dict[str, any]:
        drift_results = {
            'timestamp': datetime.utcnow().isoformat(),
            'overall_drift': False,
            'feature_drift': {},
        }

        for col in self.reference_data.columns:
            if col not in current_data.columns:
                continue

            ref = self.reference_data[col].dropna()
            cur = current_data[col].dropna()

            # Kolmogorov-Smirnov test
            ks_stat, p_value = stats.ks_2samp(ref, cur)
            
            # Population Stability Index
            psi = self._compute_psi(ref, cur)

            # Jensen-Shannon Divergence
            js_div = self._compute_js_divergence(ref, cur)

            is_drifted = (
                p_value < self.significance_level or
                psi > self.drift_threshold or
                js_div > 0.1
            )

            drift_results['feature_drift'][col] = {
                'ks_statistic': float(ks_stat),
                'p_value': float(p_value),
                'psi': float(psi),
                'js_divergence': float(js_div),
                'is_drifted': is_drifted,
                'current_mean': float(cur.mean()),
                'reference_mean': float(ref.mean()),
                'mean_shift_pct': float(abs(cur.mean() - ref.mean()) / (ref.mean() + 1e-10) * 100),
            }

            if is_drifted:
                drift_results['overall_drift'] = True
                logger.warning(f"Data drift detected in feature '{col}': PSI={psi:.3f}, p-value={p_value:.4f}")

        drift_results['drift_score'] = sum(
            1 for v in drift_results['feature_drift'].values() if v['is_drifted']
        ) / max(len(drift_results['feature_drift']), 1)

        return drift_results

    def _compute_psi(self, reference: pd.Series, current: pd.Series, bins: int = 10) -> float:
        """Population Stability Index"""
        ref_hist, bin_edges = np.histogram(reference, bins=bins)
        cur_hist, _ = np.histogram(current, bins=bin_edges)

        ref_pct = ref_hist / (ref_hist.sum() + 1e-10)
        cur_pct = cur_hist / (cur_hist.sum() + 1e-10)

        ref_pct = np.clip(ref_pct, 1e-10, None)
        cur_pct = np.clip(cur_pct, 1e-10, None)

        psi = np.sum((cur_pct - ref_pct) * np.log(cur_pct / ref_pct))
        return float(psi)

    def _compute_js_divergence(self, reference: pd.Series, current: pd.Series) -> float:
        """Jensen-Shannon Divergence"""
        ref_hist, bin_edges = np.histogram(reference, bins=20, density=True)
        cur_hist, _ = np.histogram(current, bins=bin_edges, density=True)

        ref_pct = np.clip(ref_hist / ref_hist.sum(), 1e-10, None)
        cur_pct = np.clip(cur_hist / cur_hist.sum(), 1e-10, None)

        m = 0.5 * (ref_pct + cur_pct)
        js = 0.5 * stats.entropy(ref_pct, m) + 0.5 * stats.entropy(cur_pct, m)
        return float(js)
```

---

## 10. Production Deployment Pattern

### 10.1 Kubernetes ML Deployment

```yaml
# kubernetes/ml-serving-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-inference-service
  namespace: ml-serving
  labels:
    app: ml-inference
    version: v2
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ml-inference
  template:
    metadata:
      labels:
        app: ml-inference
        version: v2
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - ml-inference
                topologyKey: kubernetes.io/hostname
      containers:
        - name: inference-service
          image: registry.company.com/ml-inference:v2.3.1
          ports:
            - containerPort: 3000
              name: http
            - containerPort: 9090
              name: metrics
          resources:
            requests:
              cpu: "2"
              memory: "4Gi"
            limits:
              cpu: "4"
              memory: "8Gi"
          env:
            - name: TF_SERVING_HOST
              value: tensorflow-serving-service
            - name: FEATURE_STORE_REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-credentials
                  key: api-key
            - name: NODE_ENV
              value: production
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 2
          volumeMounts:
            - name: model-config
              mountPath: /app/config
      volumes:
        - name: model-config
          configMap:
            name: ml-model-config
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ml-inference-hpa
  namespace: ml-serving
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-inference-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Pods
      pods:
        metric:
          name: inference_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

---

## สรุป

บทนี้ครอบคลุม AI/ML Integration ใน Microservices อย่างครบถ้วน:

1. **ML Model Serving** - TensorFlow Serving และ NVIDIA Triton สำหรับ high-performance inference
2. **Feature Store Pattern** - การจัดการ features ด้วย Feast, Redis, และ PostgreSQL
3. **A/B Testing** - การทดสอบ ML models อย่างเป็นระบบด้วย statistical significance
4. **Model Versioning** - MLflow สำหรับ model registry และ lifecycle management
5. **Online vs Offline Inference** - Batching สำหรับ real-time และ Spark สำหรับ batch processing
6. **Vector Databases** - pgvector สำหรับ semantic search และ hybrid search
7. **LLM Integration** - LangChain RAG pattern สำหรับ question answering
8. **Drift Detection** - การตรวจจับ data drift ด้วย KS-test และ PSI
9. **Production Deployment** - Kubernetes HPA และ monitoring patterns

Key Takeaways:
- ใช้ Feature Store เป็น single source of truth สำหรับ ML features
- A/B testing ควรมี statistical significance ก่อน promote model
- Vector databases ช่วยให้ semantic search มีประสิทธิภาพสูง
- Monitor data drift อย่างสม่ำเสมอเพื่อตรวจจับ model degradation
- Batch inference ประหยัดค่าใช้จ่ายสำหรับ non-real-time predictions

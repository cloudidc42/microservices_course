# Part 73: Microservices ใน Cloud Platforms (AWS, GCP, Azure)

## บทนำ

Cloud Platforms ให้ managed services ที่ช่วยลด operational overhead อย่างมาก Part นี้จะครอบคลุมการ deploy Microservices บน AWS, GCP และ Azure รวมถึงการใช้ cloud-native services อย่างมีประสิทธิภาพ

## หัวข้อที่ครอบคลุม

1. AWS EKS + RDS + ElastiCache
2. AWS Service Mesh กับ App Mesh
3. AWS API Gateway + Lambda
4. GCP GKE + Cloud SQL
5. Azure AKS + CosmosDB
6. Multi-Cloud Strategy
7. Cloud Cost Optimization
8. Infrastructure as Code ด้วย Terraform

---

## 1. AWS Infrastructure

### 1.1 EKS Cluster Setup ด้วย Terraform

```hcl
# terraform/aws/eks-cluster.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
  
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "eks/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "${var.cluster_name}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway   = true
  single_nat_gateway   = false  # HA: NAT gateway per AZ
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    Environment = var.environment
    ManagedBy   = "terraform"
  }
  
  public_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/elb"                    = 1
  }
  
  private_subnet_tags = {
    "kubernetes.io/cluster/${var.cluster_name}" = "shared"
    "kubernetes.io/role/internal-elb"           = 1
  }
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
  
  cluster_name    = var.cluster_name
  cluster_version = "1.28"
  
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  # EKS Addons
  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
    aws-ebs-csi-driver = {
      most_recent = true
    }
  }
  
  # Node Groups
  eks_managed_node_groups = {
    general = {
      name           = "general"
      instance_types = ["m6i.xlarge"]
      
      min_size     = 3
      max_size     = 10
      desired_size = 3
      
      labels = {
        role = "general"
      }
      
      taints = []
      
      tags = {
        ExtraTag = "general-nodes"
      }
    }
    
    spot = {
      name           = "spot"
      instance_types = ["m6i.xlarge", "m5.xlarge", "m5a.xlarge"]
      capacity_type  = "SPOT"
      
      min_size     = 0
      max_size     = 20
      desired_size = 2
      
      labels = {
        role = "spot"
      }
      
      taints = [
        {
          key    = "spot"
          value  = "true"
          effect = "NO_SCHEDULE"
        }
      ]
    }
    
    gpu = {
      name           = "gpu"
      instance_types = ["g4dn.xlarge"]
      
      min_size     = 0
      max_size     = 5
      desired_size = 0
      
      labels = {
        role         = "gpu"
        "nvidia.com/gpu" = "true"
      }
      
      taints = [
        {
          key    = "nvidia.com/gpu"
          value  = "true"
          effect = "NO_SCHEDULE"
        }
      ]
    }
  }
  
  tags = {
    Environment = var.environment
    Terraform   = "true"
  }
}
```

### 1.2 RDS PostgreSQL ด้วย Terraform

```hcl
# terraform/aws/rds.tf

module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"
  
  identifier = "${var.cluster_name}-postgres"
  
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = "db.r7g.xlarge"
  allocated_storage = 100
  max_allocated_storage = 1000  # Autoscaling
  
  db_name  = "microservices"
  username = "dbadmin"
  
  # Password ใน AWS Secrets Manager
  manage_master_user_password = true
  
  # Multi-AZ สำหรับ HA
  multi_az = true
  
  # Backup
  backup_retention_period = 7
  backup_window           = "03:00-06:00"
  maintenance_window      = "Mon:00:00-Mon:03:00"
  
  # Storage
  storage_type      = "gp3"
  storage_encrypted = true
  
  # VPC
  vpc_security_group_ids = [module.rds_security_group.security_group_id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  # Performance
  performance_insights_enabled          = true
  performance_insights_retention_period = 7
  
  # Monitoring
  monitoring_interval    = 60
  monitoring_role_name   = "rds-monitoring-role"
  create_monitoring_role = true
  
  # Parameter group
  family = "postgres15"
  
  parameters = [
    {
      name  = "max_connections"
      value = "500"
    },
    {
      name  = "shared_buffers"
      value = "4GB"
    },
    {
      name  = "effective_cache_size"
      value = "12GB"
    },
    {
      name  = "work_mem"
      value = "16MB"
    },
    {
      name  = "maintenance_work_mem"
      value = "1GB"
    },
    {
      name  = "log_min_duration_statement"
      value = "1000"  # Log queries > 1s
    },
  ]
  
  # Read replicas
  create_db_instance = true
  
  tags = {
    Environment = var.environment
    Service     = "database"
  }
}

# Read Replica
resource "aws_db_instance" "replica" {
  identifier             = "${var.cluster_name}-postgres-replica"
  replicate_source_db    = module.rds.db_instance_identifier
  instance_class         = "db.r7g.large"
  
  storage_encrypted      = true
  publicly_accessible    = false
  vpc_security_group_ids = [module.rds_security_group.security_group_id]
  
  performance_insights_enabled = true
  
  tags = {
    Environment = var.environment
    Role        = "read-replica"
  }
}
```

### 1.3 ElastiCache Redis

```hcl
# terraform/aws/elasticache.tf

resource "aws_elasticache_replication_group" "main" {
  replication_group_id = "${var.cluster_name}-redis"
  description          = "Redis cluster for microservices"
  
  node_type               = "cache.r7g.large"
  num_cache_clusters      = 3
  port                    = 6379
  parameter_group_name    = "default.redis7.cluster.on"
  
  # Cluster mode enabled
  automatic_failover_enabled = true
  multi_az_enabled           = true
  
  subnet_group_name = aws_elasticache_subnet_group.main.name
  security_group_ids = [module.redis_security_group.security_group_id]
  
  # Encryption
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token                 = var.redis_auth_token
  
  # Backup
  snapshot_retention_limit = 7
  snapshot_window          = "05:00-06:00"
  maintenance_window       = "sun:23:00-mon:01:30"
  
  # Auto minor version upgrade
  auto_minor_version_upgrade = true
  
  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis_slow_log.name
    destination_type = "cloudwatch-logs"
    log_format       = "text"
    log_type         = "slow-log"
  }
  
  tags = {
    Environment = var.environment
    Service     = "cache"
  }
}
```

---

## 2. AWS Service Discovery กับ Cloud Map

```typescript
// aws/service-discovery.ts
import {
  ServiceDiscoveryClient,
  RegisterInstanceCommand,
  DeregisterInstanceCommand,
  DiscoverInstancesCommand,
} from '@aws-sdk/client-servicediscovery';
import os from 'os';

export class AWSServiceDiscovery {
  private client: ServiceDiscoveryClient;
  private instanceId: string;
  
  constructor(private config: {
    namespace: string;
    serviceName: string;
    port: number;
    region: string;
  }) {
    this.client = new ServiceDiscoveryClient({
      region: config.region,
    });
    
    this.instanceId = `${os.hostname()}-${process.pid}`;
  }
  
  async register(): Promise<void> {
    const command = new RegisterInstanceCommand({
      ServiceId: await this.getServiceId(),
      InstanceId: this.instanceId,
      Attributes: {
        AWS_INSTANCE_IPV4: await this.getLocalIP(),
        AWS_INSTANCE_PORT: String(this.config.port),
        ENVIRONMENT: process.env.NODE_ENV || 'production',
        VERSION: process.env.APP_VERSION || '1.0.0',
      },
    });
    
    await this.client.send(command);
    console.log(`Registered with AWS Cloud Map: ${this.instanceId}`);
    
    // Deregister on shutdown
    process.on('SIGTERM', () => this.deregister());
    process.on('SIGINT', () => this.deregister());
  }
  
  async deregister(): Promise<void> {
    const command = new DeregisterInstanceCommand({
      ServiceId: await this.getServiceId(),
      InstanceId: this.instanceId,
    });
    
    await this.client.send(command);
    console.log(`Deregistered from AWS Cloud Map: ${this.instanceId}`);
  }
  
  async discover(serviceName: string): Promise<Array<{ host: string; port: number }>> {
    const command = new DiscoverInstancesCommand({
      NamespaceName: this.config.namespace,
      ServiceName: serviceName,
      MaxResults: 10,
      HealthStatus: 'HEALTHY',
    });
    
    const response = await this.client.send(command);
    
    return (response.Instances || []).map(instance => ({
      host: instance.Attributes!['AWS_INSTANCE_IPV4'],
      port: parseInt(instance.Attributes!['AWS_INSTANCE_PORT']),
    }));
  }
  
  private async getServiceId(): Promise<string> {
    // สามารถ cache ค่านี้ได้
    return process.env.AWS_CLOUD_MAP_SERVICE_ID || '';
  }
  
  private async getLocalIP(): Promise<string> {
    const interfaces = os.networkInterfaces();
    for (const iface of Object.values(interfaces)) {
      for (const addr of iface || []) {
        if (addr.family === 'IPv4' && !addr.internal) {
          return addr.address;
        }
      }
    }
    return '127.0.0.1';
  }
}
```

---

## 3. AWS Lambda Integration

### 3.1 Lambda Function สำหรับ Event Processing

```typescript
// lambda/event-processor/handler.ts
import { SQSEvent, SQSHandler, SQSRecord } from 'aws-lambda';
import { Pool } from 'pg';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

let db: Pool | null = null;

// Reuse connections across Lambda invocations
function getDB(): Pool {
  if (!db) {
    db = new Pool({
      host: process.env.DB_HOST,
      port: parseInt(process.env.DB_PORT || '5432'),
      database: process.env.DB_NAME,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      max: 5, // Lambda parallel = small pool
      idleTimeoutMillis: 60000,
      ssl: { rejectUnauthorized: false },
    });
  }
  return db;
}

const s3 = new S3Client({ region: process.env.AWS_REGION });

export const handler: SQSHandler = async (event: SQSEvent) => {
  const failedRecords: string[] = [];
  
  for (const record of event.Records) {
    try {
      await processRecord(record);
    } catch (error) {
      console.error(`Failed to process record ${record.messageId}:`, error);
      failedRecords.push(record.messageId);
    }
  }
  
  // SQS batch item failures - ส่ง error records กลับไปยัง DLQ
  if (failedRecords.length > 0) {
    return {
      batchItemFailures: failedRecords.map(id => ({
        itemIdentifier: id,
      })),
    };
  }
};

async function processRecord(record: SQSRecord): Promise<void> {
  const event = JSON.parse(record.body);
  const db = getDB();
  
  switch (event.type) {
    case 'order.created':
      await handleOrderCreated(db, event);
      break;
    case 'payment.processed':
      await handlePaymentProcessed(db, event);
      break;
    case 'report.generate':
      await generateReport(event);
      break;
    default:
      console.warn(`Unknown event type: ${event.type}`);
  }
}

async function handleOrderCreated(db: Pool, event: any): Promise<void> {
  await db.query(
    `INSERT INTO order_analytics (order_id, customer_id, amount, created_at)
     VALUES ($1, $2, $3, $4)
     ON CONFLICT (order_id) DO NOTHING`,
    [event.orderId, event.customerId, event.amount, event.timestamp]
  );
}

async function handlePaymentProcessed(db: Pool, event: any): Promise<void> {
  await db.query(
    `UPDATE order_analytics 
     SET payment_status = 'completed', payment_id = $2
     WHERE order_id = $1`,
    [event.orderId, event.paymentId]
  );
}

async function generateReport(event: any): Promise<void> {
  const db = getDB();
  const result = await db.query(
    `SELECT date_trunc('day', created_at) as date,
            COUNT(*) as orders,
            SUM(amount) as revenue
     FROM order_analytics
     WHERE created_at >= $1 AND created_at < $2
     GROUP BY 1
     ORDER BY 1`,
    [event.startDate, event.endDate]
  );
  
  const reportKey = `reports/${event.reportId}.json`;
  await s3.send(new PutObjectCommand({
    Bucket: process.env.REPORTS_BUCKET,
    Key: reportKey,
    Body: JSON.stringify(result.rows),
    ContentType: 'application/json',
  }));
  
  console.log(`Report saved to s3://${process.env.REPORTS_BUCKET}/${reportKey}`);
}
```

### 3.2 API Gateway + Lambda สำหรับ Serverless Microservices

```yaml
# serverless.yml (Serverless Framework)
service: order-analytics-service

frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs18.x
  region: ap-southeast-1
  stage: ${opt:stage, 'production'}
  
  environment:
    DB_HOST: ${ssm:/microservices/db/host}
    DB_NAME: analytics
    DB_USER: analytics_user
    DB_PASSWORD: ${ssm:/microservices/db/password~true}
    REPORTS_BUCKET: !Ref ReportsBucket
  
  iam:
    role:
      statements:
      - Effect: Allow
        Action:
        - s3:PutObject
        - s3:GetObject
        Resource: !Sub 'arn:aws:s3:::${ReportsBucket}/*'
      - Effect: Allow
        Action:
        - ssm:GetParameter
        Resource: 'arn:aws:ssm:*:*:parameter/microservices/*'
      - Effect: Allow
        Action:
        - xray:PutTraceSegments
        - xray:PutTelemetryRecords
        Resource: '*'
  
  tracing:
    apiGateway: true
    lambda: true
  
  logs:
    restApi:
      accessLogging: true
      format: >-
        {"requestId":"$context.requestId",
        "ip":"$context.identity.sourceIp",
        "caller":"$context.identity.caller",
        "user":"$context.identity.user",
        "requestTime":"$context.requestTime",
        "httpMethod":"$context.httpMethod",
        "resourcePath":"$context.resourcePath",
        "status":"$context.status",
        "protocol":"$context.protocol",
        "responseLength":"$context.responseLength"}

functions:
  processEvents:
    handler: dist/handler.handler
    timeout: 30
    memorySize: 256
    reservedConcurrency: 10
    events:
    - sqs:
        arn: !GetAtt EventQueue.Arn
        batchSize: 10
        maximumBatchingWindow: 5
        functionResponseType: ReportBatchItemFailures
  
  getAnalytics:
    handler: dist/analytics.handler
    timeout: 10
    events:
    - http:
        path: /analytics
        method: get
        authorizer:
          name: jwtAuthorizer
          type: TOKEN
          identitySource: method.request.header.Authorization

resources:
  Resources:
    EventQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: !Sub '${self:service}-${self:provider.stage}'
        VisibilityTimeout: 60
        MessageRetentionPeriod: 86400
        RedrivePolicy:
          deadLetterTargetArn: !GetAtt DeadLetterQueue.Arn
          maxReceiveCount: 3
    
    DeadLetterQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: !Sub '${self:service}-${self:provider.stage}-dlq'
        MessageRetentionPeriod: 1209600  # 14 days
    
    ReportsBucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: !Sub '${self:service}-reports-${self:provider.stage}'
        VersioningConfiguration:
          Status: Enabled
        LifecycleConfiguration:
          Rules:
          - Id: TransitionToGlacier
            Status: Enabled
            Transitions:
            - TransitionInDays: 90
              StorageClass: GLACIER
```

---

## 4. GCP GKE Setup

### 4.1 GKE Cluster ด้วย Terraform

```hcl
# terraform/gcp/gke.tf

provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_container_cluster" "primary" {
  name     = "${var.cluster_name}"
  location = var.region
  
  # We can't create a cluster with no node pool defined, but we want to only use
  # separately managed node pools. So we create the smallest possible default
  # node pool and immediately delete it.
  remove_default_node_pool = true
  initial_node_count       = 1
  
  network    = google_compute_network.main.name
  subnetwork = google_compute_subnetwork.main.name
  
  # Workload Identity
  workload_identity_config {
    workload_pool = "${var.project_id}.svc.id.goog"
  }
  
  # Logging
  logging_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
  
  # Monitoring
  monitoring_config {
    enable_components = ["SYSTEM_COMPONENTS", "WORKLOADS"]
  }
  
  # Addons
  addons_config {
    horizontal_pod_autoscaling {
      disabled = false
    }
    http_load_balancing {
      disabled = false
    }
    network_policy_config {
      disabled = false
    }
    gce_persistent_disk_csi_driver_config {
      enabled = true
    }
  }
  
  # Network Policy
  network_policy {
    enabled = true
    provider = "CALICO"
  }
  
  # Private cluster
  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }
}

resource "google_container_node_pool" "general" {
  name       = "general"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = 3
  
  autoscaling {
    min_node_count = 3
    max_node_count = 20
  }
  
  node_config {
    preemptible  = false
    machine_type = "n2-standard-4"
    disk_size_gb = 100
    disk_type    = "pd-ssd"
    
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
    
    workload_metadata_config {
      mode = "GKE_METADATA"
    }
    
    labels = {
      env  = var.environment
      role = "general"
    }
  }
  
  management {
    auto_repair  = true
    auto_upgrade = true
  }
  
  upgrade_settings {
    max_surge       = 1
    max_unavailable = 0
  }
}

# Cloud SQL PostgreSQL
resource "google_sql_database_instance" "main" {
  name             = "${var.cluster_name}-postgres"
  database_version = "POSTGRES_15"
  region           = var.region
  
  settings {
    tier = "db-custom-4-16384"  # 4 vCPU, 16GB RAM
    
    availability_type = "REGIONAL"  # HA
    
    disk_autoresize       = true
    disk_autoresize_limit = 1000
    disk_size             = 100
    disk_type             = "PD_SSD"
    
    backup_configuration {
      enabled                        = true
      start_time                     = "03:00"
      point_in_time_recovery_enabled = true
      backup_retention_settings {
        retained_backups = 7
      }
    }
    
    database_flags {
      name  = "max_connections"
      value = "500"
    }
    
    database_flags {
      name  = "cloudsql.iam_authentication"
      value = "on"
    }
    
    insights_config {
      query_insights_enabled  = true
      query_string_length     = 1024
      record_application_tags = true
      record_client_address   = true
    }
  }
  
  deletion_protection = true
}
```

---

## 5. Azure AKS Setup

### 5.1 AKS ด้วย Bicep

```bicep
// azure/main.bicep

@description('Cluster name')
param clusterName string = 'microservices-cluster'

@description('Location')
param location string = resourceGroup().location

@description('Kubernetes version')
param kubernetesVersion string = '1.28.0'

// AKS Cluster
resource aksCluster 'Microsoft.ContainerService/managedClusters@2023-07-01' = {
  name: clusterName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: kubernetesVersion
    dnsPrefix: clusterName
    
    agentPoolProfiles: [
      {
        name: 'systempool'
        count: 3
        vmSize: 'Standard_D4s_v3'
        osType: 'Linux'
        mode: 'System'
        enableAutoScaling: true
        minCount: 3
        maxCount: 10
        vnetSubnetID: aksSubnet.id
        nodeTaints: [
          'CriticalAddonsOnly=true:NoSchedule'
        ]
      }
      {
        name: 'userpool'
        count: 3
        vmSize: 'Standard_D8s_v3'
        osType: 'Linux'
        mode: 'User'
        enableAutoScaling: true
        minCount: 1
        maxCount: 20
        vnetSubnetID: aksSubnet.id
        spotMaxPrice: -1
      }
    ]
    
    networkProfile: {
      networkPlugin: 'azure'
      networkPolicy: 'azure'
      serviceCidr: '10.0.0.0/16'
      dnsServiceIP: '10.0.0.10'
      loadBalancerSku: 'standard'
    }
    
    addonProfiles: {
      omsagent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalyticsWorkspace.id
        }
      }
      azurepolicy: {
        enabled: true
      }
      azureKeyVaultSecretsProvider: {
        enabled: true
        config: {
          enableSecretRotation: 'true'
          rotationPollInterval: '2m'
        }
      }
    }
    
    aadProfile: {
      managed: true
      enableAzureRBAC: true
    }
    
    autoUpgradeProfile: {
      upgradeChannel: 'patch'
    }
    
    oidcIssuerProfile: {
      enabled: true
    }
    
    securityProfile: {
      workloadIdentity: {
        enabled: true
      }
    }
  }
}
```

---

## 6. Multi-Cloud Kubernetes Federation

```typescript
// multi-cloud/cluster-manager.ts
import { KubeConfig, AppsV1Api, CoreV1Api } from '@kubernetes/client-node';
import path from 'path';

interface ClusterConfig {
  name: string;
  cloud: 'aws' | 'gcp' | 'azure';
  region: string;
  kubeconfigPath: string;
}

export class MultiClusterManager {
  private clusters: Map<string, {
    apps: AppsV1Api;
    core: CoreV1Api;
    config: ClusterConfig;
  }> = new Map();
  
  addCluster(config: ClusterConfig): void {
    const kubeConfig = new KubeConfig();
    kubeConfig.loadFromFile(config.kubeconfigPath);
    
    this.clusters.set(config.name, {
      apps: kubeConfig.makeApiClient(AppsV1Api),
      core: kubeConfig.makeApiClient(CoreV1Api),
      config,
    });
    
    console.log(`Added cluster: ${config.name} (${config.cloud}/${config.region})`);
  }
  
  async deployToAll(
    namespace: string,
    deployment: object
  ): Promise<void> {
    const promises = [...this.clusters.values()].map(async ({ apps, config }) => {
      try {
        await apps.createNamespacedDeployment(namespace, deployment as any);
        console.log(`Deployed to ${config.name}`);
      } catch (error: any) {
        if (error.statusCode === 409) {
          // Update if exists
          const dep = deployment as any;
          await apps.replaceNamespacedDeployment(
            dep.metadata.name,
            namespace,
            deployment as any
          );
          console.log(`Updated in ${config.name}`);
        } else {
          throw error;
        }
      }
    });
    
    await Promise.allSettled(promises);
  }
  
  async getDeploymentStatus(name: string, namespace: string): Promise<Map<string, string>> {
    const status = new Map<string, string>();
    
    for (const [clusterName, { apps }] of this.clusters) {
      try {
        const response = await apps.readNamespacedDeployment(name, namespace);
        const dep = response.body;
        const ready = dep.status?.readyReplicas || 0;
        const desired = dep.spec?.replicas || 0;
        status.set(clusterName, `${ready}/${desired} ready`);
      } catch {
        status.set(clusterName, 'not found');
      }
    }
    
    return status;
  }
  
  async getClusterHealth(): Promise<Map<string, boolean>> {
    const health = new Map<string, boolean>();
    
    for (const [clusterName, { core }] of this.clusters) {
      try {
        await core.listNode();
        health.set(clusterName, true);
      } catch {
        health.set(clusterName, false);
      }
    }
    
    return health;
  }
}
```

---

## 7. Infrastructure Monitoring

### 7.1 CloudWatch Alerts

```typescript
// monitoring/cloudwatch-alerts.ts
import {
  CloudWatchClient,
  PutMetricAlarmCommand,
  MetricDatum,
  PutMetricDataCommand,
} from '@aws-sdk/client-cloudwatch';

export class CloudWatchMonitor {
  private client: CloudWatchClient;
  
  constructor(region: string) {
    this.client = new CloudWatchClient({ region });
  }
  
  async createHighLatencyAlert(
    serviceName: string,
    thresholdMs: number = 500
  ): Promise<void> {
    await this.client.send(new PutMetricAlarmCommand({
      AlarmName: `${serviceName}-high-latency`,
      MetricName: 'Latency',
      Namespace: `Microservices/${serviceName}`,
      Statistic: 'p99',
      Period: 60,
      EvaluationPeriods: 5,
      Threshold: thresholdMs,
      ComparisonOperator: 'GreaterThanThreshold',
      AlarmDescription: `P99 latency > ${thresholdMs}ms for ${serviceName}`,
      AlarmActions: [process.env.SNS_ALARM_ARN!],
      TreatMissingData: 'notBreaching',
      ExtendedStatistic: 'p99',
    }));
  }
  
  async createErrorRateAlert(
    serviceName: string,
    errorRatePercent: number = 5
  ): Promise<void> {
    await this.client.send(new PutMetricAlarmCommand({
      AlarmName: `${serviceName}-high-error-rate`,
      Metrics: [
        {
          Id: 'errors',
          MetricStat: {
            Metric: {
              Namespace: `Microservices/${serviceName}`,
              MetricName: 'Errors',
            },
            Period: 60,
            Stat: 'Sum',
          },
        },
        {
          Id: 'requests',
          MetricStat: {
            Metric: {
              Namespace: `Microservices/${serviceName}`,
              MetricName: 'Requests',
            },
            Period: 60,
            Stat: 'Sum',
          },
        },
        {
          Id: 'errorRate',
          Expression: 'errors / requests * 100',
          Label: 'Error Rate %',
        },
      ],
      ComparisonOperator: 'GreaterThanThreshold',
      EvaluationPeriods: 3,
      Threshold: errorRatePercent,
      AlarmDescription: `Error rate > ${errorRatePercent}% for ${serviceName}`,
      AlarmActions: [process.env.SNS_ALARM_ARN!],
    }));
  }
  
  async publishMetric(
    namespace: string,
    metrics: Array<{ name: string; value: number; unit?: string; dimensions?: Record<string, string> }>
  ): Promise<void> {
    const metricData: MetricDatum[] = metrics.map(m => ({
      MetricName: m.name,
      Value: m.value,
      Unit: (m.unit as any) || 'None',
      Timestamp: new Date(),
      Dimensions: m.dimensions
        ? Object.entries(m.dimensions).map(([Name, Value]) => ({ Name, Value }))
        : [],
    }));
    
    await this.client.send(new PutMetricDataCommand({
      Namespace: namespace,
      MetricData: metricData,
    }));
  }
}
```

---

## สรุป

| Cloud Platform | Kubernetes | Database | Cache | Event Streaming |
|---------------|-----------|---------|-------|----------------|
| AWS | EKS | RDS Aurora | ElastiCache | MSK (Kafka) |
| GCP | GKE | Cloud SQL | Memorystore | Cloud Pub/Sub |
| Azure | AKS | Azure Database | Azure Cache | Event Hubs |

แนวทางเลือก Cloud Platform:
- **AWS**: Ecosystem ใหญ่ที่สุด, มาก managed services
- **GCP**: ดีสำหรับ ML/AI workloads, Kubernetes ต้นกำเนิด
- **Azure**: เหมาะกับ enterprise ที่ใช้ Microsoft stack

Multi-Cloud Strategy:
- Deploy ใน multiple clouds เพื่อ availability
- ใช้ Kubernetes เป็น abstraction layer
- Terraform สำหรับ infrastructure as code ที่ portable
- ระวัง vendor lock-in กับ managed services

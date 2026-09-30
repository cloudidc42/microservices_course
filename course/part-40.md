# Part 40: Platform Engineering & Internal Developer Platform (IDP)

## บทนำ

Platform Engineering คือ discipline ที่สร้าง Internal Developer Platform (IDP) เพื่อให้นักพัฒนาสามารถ deploy, monitor, และจัดการ services ได้เอง โดยไม่ต้องรอ Ops team ในบทนี้เราจะสร้าง Production-Grade IDP ด้วย Backstage, Port, และ automation tools

## 1. Internal Developer Platform Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                Internal Developer Platform                   │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Service  │  │ Scaffolding│ │ CI/CD    │  │Observ.  │   │
│  │ Catalog  │  │Templates │  │ Pipeline │  │ Portal  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Secret  │  │ Database │  │ Infra as │  │  Cost   │   │
│  │ Manager  │  │ Provision│  │  Code    │  │Dashboard│   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           Self-Service Portal (Backstage)           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 2. Backstage Software Catalog

### 2.1 Service catalog.yaml

```yaml
# catalog-info.yaml (ใน root ของแต่ละ service repo)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: product-service
  title: Product Service
  description: Manages product catalog, inventory, and pricing
  annotations:
    # Link to CI/CD
    github.com/project-slug: org/product-service
    # Link to documentation
    backstage.io/techdocs-ref: dir:.
    # Link to monitoring
    grafana/dashboard-selector: product-service
    # Link to runbook
    opsgenie.com/component-selector: product-service
    # SLO tracking
    slo-targets/availability: "99.9"
    slo-targets/latency-p99: "500ms"
    # Cost center
    finops/cost-center: engineering
    finops/team: catalog-team
  labels:
    tier: "production"
    language: "nodejs"
    framework: "express"
  tags:
    - nodejs
    - postgres
    - redis
    - kafka
spec:
  type: service
  lifecycle: production
  owner: group:catalog-team
  system: ecommerce-platform
  dependsOn:
    - component:inventory-service
    - component:pricing-service
    - resource:products-postgres-db
    - resource:products-redis-cache
  providesApis:
    - product-api
  consumesApis:
    - inventory-api
    - pricing-api

---
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: product-api
  title: Product API
  description: RESTful API for product management
spec:
  type: openapi
  lifecycle: production
  owner: group:catalog-team
  system: ecommerce-platform
  definition:
    $text: https://api.example.com/openapi.yaml

---
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: products-postgres-db
  title: Products PostgreSQL Database
  description: Primary database for product service
spec:
  type: database
  owner: group:platform-team
  system: ecommerce-platform
```

### 2.2 Backstage Configuration

```yaml
# backstage/app-config.yaml
app:
  title: Engineering Portal
  baseUrl: https://backstage.internal.example.com

organization:
  name: Example Engineering

backend:
  baseUrl: https://backstage.internal.example.com
  listen:
    port: 7007
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}
      database: backstage_plugin_catalog

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

catalog:
  locations:
    # Discover all service catalogs automatically
    - type: github-discovery
      target: https://github.com/org/*/blob/main/catalog-info.yaml
    # Platform-wide catalog
    - type: url
      target: https://github.com/org/platform-catalog/blob/main/all.yaml
  
  rules:
    - allow: [Component, API, Resource, System, Group, User]

auth:
  providers:
    github:
      development:
        clientId: ${GITHUB_CLIENT_ID}
        clientSecret: ${GITHUB_CLIENT_SECRET}

techdocs:
  builder: external
  generator:
    runIn: docker
  publisher:
    type: awsS3
    awsS3:
      bucketName: techdocs-bucket
      region: ap-southeast-1

kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - url: ${KUBERNETES_API_SERVER_URL}
          name: production
          authProvider: serviceAccount
          serviceAccountToken: ${KUBERNETES_SA_TOKEN}
          caData: ${KUBERNETES_CA_DATA}

grafana:
  domain: https://grafana.internal.example.com
  unifiedAlerting: true
```

## 3. Service Scaffolding Templates

### 3.1 Backstage Template for New Microservice

```yaml
# backstage/templates/microservice-template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: microservice-nodejs
  title: Node.js Microservice
  description: Create a new Node.js microservice with all best practices pre-configured
  tags:
    - nodejs
    - microservice
    - recommended
spec:
  owner: group:platform-team
  type: service
  
  parameters:
    - title: Service Information
      required: [name, description, owner, system]
      properties:
        name:
          title: Service Name
          type: string
          description: Name of the service (lowercase, hyphenated)
          pattern: '^[a-z][a-z0-9-]*[a-z0-9]$'
          ui:autofocus: true
        description:
          title: Description
          type: string
          maxLength: 200
        owner:
          title: Team Owner
          type: string
          ui:field: OwnerPicker
          ui:options:
            allowedKinds: [Group]
        system:
          title: System
          type: string
          ui:field: EntityPicker
          ui:options:
            catalogFilter:
              kind: System
    
    - title: Technical Configuration
      required: [databaseType, messagingEnabled, httpPort]
      properties:
        databaseType:
          title: Database
          type: string
          enum: [postgres, mongodb, none]
          default: postgres
        messagingEnabled:
          title: Enable Kafka Messaging
          type: boolean
          default: true
        httpPort:
          title: HTTP Port
          type: integer
          default: 3000
          minimum: 1024
          maximum: 65535
        enableCaching:
          title: Enable Redis Caching
          type: boolean
          default: true
        enableTracing:
          title: Enable OpenTelemetry Tracing
          type: boolean
          default: true
    
    - title: Repository Configuration
      required: [repoUrl]
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts: [github.com]
            allowedOrganizations: [org]
  
  steps:
    - id: fetch-template
      name: Fetch Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          system: ${{ parameters.system }}
          databaseType: ${{ parameters.databaseType }}
          messagingEnabled: ${{ parameters.messagingEnabled }}
          httpPort: ${{ parameters.httpPort }}
          enableCaching: ${{ parameters.enableCaching }}
          enableTracing: ${{ parameters.enableTracing }}
    
    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: [github.com]
        description: ${{ parameters.description }}
        repoUrl: ${{ parameters.repoUrl }}
        defaultBranch: main
        repoVisibility: private
        topics:
          - microservice
          - nodejs
          - ${{ parameters.system }}
        requireCodeOwnerReviews: true
        requiredStatusChecks:
          - ci/build
          - ci/test
          - ci/lint
    
    - id: register-catalog
      name: Register in Software Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
    
    - id: create-argocd-app
      name: Create ArgoCD Application
      action: http:backstage:request
      input:
        method: POST
        path: /api/argocd/applications
        body:
          name: ${{ parameters.name }}
          project: default
          source:
            repoURL: ${{ steps.publish.output.remoteUrl }}
            targetRevision: main
            path: k8s
          destination:
            server: https://kubernetes.default.svc
            namespace: production
    
    - id: create-pagerduty-service
      name: Create PagerDuty Service
      action: pagerduty:service:create
      input:
        name: ${{ parameters.name }}
        description: ${{ parameters.description }}
        escalationPolicyId: P1234567
        alertGrouping: time
    
    - id: create-slack-channel
      name: Create Slack Channel
      action: http:backstage:request
      input:
        method: POST
        path: /api/slack/channels
        body:
          name: "service-${{ parameters.name }}"
          description: "Alerts and notifications for ${{ parameters.name }}"
  
  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
      - title: View in Catalog
        icon: catalog
        entityRef: ${{ steps.register-catalog.output.entityRef }}
```

## 4. Self-Service Infrastructure Provisioning

### 4.1 Database Provisioning API

```javascript
// platform-api/routes/databases.js
const express = require('express');
const router = express.Router();
const { TerraformRunner } = require('../services/terraformRunner');
const { validateRequest } = require('../middleware/validate');
const Joi = require('joi');

const dbSchema = Joi.object({
  name: Joi.string().pattern(/^[a-z][a-z0-9_]*$/).required(),
  team: Joi.string().required(),
  environment: Joi.string().valid('staging', 'production').required(),
  dbType: Joi.string().valid('postgres', 'mysql', 'mongodb').required(),
  size: Joi.string().valid('small', 'medium', 'large', 'xlarge').required(),
  highAvailability: Joi.boolean().default(false),
  backupRetentionDays: Joi.number().min(1).max(35).default(7),
});

const DB_SIZES = {
  small:  { class: 'db.t3.small',   storage: 20,   monthlyCost: 50 },
  medium: { class: 'db.t3.medium',  storage: 100,  monthlyCost: 150 },
  large:  { class: 'db.m5.large',   storage: 500,  monthlyCost: 400 },
  xlarge: { class: 'db.m5.xlarge',  storage: 2000, monthlyCost: 900 },
};

router.post('/', validateRequest(dbSchema), async (req, res) => {
  const { name, team, environment, dbType, size, highAvailability, backupRetentionDays } = req.body;
  
  const sizeConfig = DB_SIZES[size];
  const monthlyCost = sizeConfig.monthlyCost * (highAvailability ? 2 : 1);
  
  // Require manager approval for large/expensive databases
  if (monthlyCost > 500) {
    return res.status(202).json({
      status: 'pending_approval',
      message: 'Database provisioning requires manager approval (>$500/month)',
      estimatedCost: { monthly: monthlyCost, annually: monthlyCost * 12 },
      approvalUrl: `https://platform.example.com/approvals/db-${name}`,
    });
  }
  
  const provisioningId = `db-${name}-${Date.now()}`;
  
  // Async provisioning via Terraform
  res.status(202).json({
    status: 'provisioning',
    provisioningId,
    estimatedTime: '5-10 minutes',
    statusUrl: `https://platform.example.com/api/provisions/${provisioningId}`,
    estimatedCost: { monthly: monthlyCost },
  });
  
  // Run Terraform in background
  setImmediate(async () => {
    try {
      const tf = new TerraformRunner({
        workingDir: `/tmp/provisions/${provisioningId}`,
        stateBackend: `s3://terraform-state/databases/${name}`,
      });
      
      await tf.init();
      await tf.apply({
        variables: {
          db_identifier: name,
          db_engine: dbType,
          instance_class: sizeConfig.class,
          allocated_storage: sizeConfig.storage,
          multi_az: highAvailability,
          backup_retention: backupRetentionDays,
          environment,
          team,
        }
      });
      
      const outputs = await tf.output();
      
      // Store credentials in Vault
      await storeInVault(`secret/databases/${name}`, {
        host: outputs.db_host,
        port: outputs.db_port,
        username: outputs.db_username,
        password: outputs.db_password,
        database: name,
      });
      
      // Update provision status
      await updateProvisioningStatus(provisioningId, 'completed', {
        host: outputs.db_host,
        credentialsPath: `secret/databases/${name}`,
      });
      
      // Notify team
      await notifySlack(`#service-${team}`, 
        `✅ Database \`${name}\` is ready!\nHost: ${outputs.db_host}\nCredentials: Vault at \`secret/databases/${name}\``
      );
      
    } catch (error) {
      await updateProvisioningStatus(provisioningId, 'failed', { error: error.message });
      await notifySlack(`#service-${team}`, `❌ Database provisioning failed: ${error.message}`);
    }
  });
});

module.exports = router;
```

### 4.2 Terraform Runner Service

```javascript
// platform-api/services/terraformRunner.js
const { execSync, spawn } = require('child_process');
const fs = require('fs');
const path = require('path');

class TerraformRunner {
  constructor(config) {
    this.workingDir = config.workingDir;
    this.stateBackend = config.stateBackend;
    this.env = {
      ...process.env,
      TF_DATA_DIR: path.join(this.workingDir, '.terraform'),
      AWS_REGION: process.env.AWS_REGION || 'ap-southeast-1',
    };
  }
  
  async init() {
    fs.mkdirSync(this.workingDir, { recursive: true });
    
    // Copy Terraform modules
    execSync(`cp -r /opt/terraform-modules/rds/* ${this.workingDir}/`);
    
    await this.run(['init', '-backend-config=bucket=terraform-state', `-backend-config=key=${this.stateBackend}`]);
  }
  
  async plan(variables = {}) {
    const varArgs = Object.entries(variables)
      .flatMap(([k, v]) => ['-var', `${k}=${v}`]);
    
    const planFile = path.join(this.workingDir, 'tfplan');
    await this.run(['plan', '-out', planFile, ...varArgs]);
    
    // Parse plan output for cost estimation
    const output = await this.run(['show', '-json', planFile]);
    return JSON.parse(output);
  }
  
  async apply(options = {}) {
    const { variables = {}, autoApprove = true } = options;
    
    const varArgs = Object.entries(variables)
      .flatMap(([k, v]) => ['-var', `${k}=${v}`]);
    
    const args = ['apply', ...varArgs];
    if (autoApprove) args.push('-auto-approve');
    
    await this.run(args);
  }
  
  async output() {
    const result = await this.run(['output', '-json']);
    const outputs = JSON.parse(result);
    
    return Object.fromEntries(
      Object.entries(outputs).map(([k, v]) => [k, v.value])
    );
  }
  
  async destroy(variables = {}) {
    const varArgs = Object.entries(variables)
      .flatMap(([k, v]) => ['-var', `${k}=${v}`]);
    
    await this.run(['destroy', '-auto-approve', ...varArgs]);
  }
  
  run(args) {
    return new Promise((resolve, reject) => {
      const logs = [];
      const child = spawn('terraform', args, {
        cwd: this.workingDir,
        env: this.env,
      });
      
      child.stdout.on('data', d => {
        const line = d.toString();
        logs.push(line);
        console.log(`[TF] ${line.trim()}`);
      });
      
      child.stderr.on('data', d => {
        console.error(`[TF ERR] ${d.toString().trim()}`);
      });
      
      child.on('close', code => {
        if (code !== 0) {
          reject(new Error(`Terraform ${args[0]} failed with code ${code}`));
        } else {
          resolve(logs.join(''));
        }
      });
    });
  }
}

module.exports = { TerraformRunner };
```

## 5. Developer CLI Tool

```javascript
#!/usr/bin/env node
// platform-cli/index.js

const { program } = require('commander');
const axios = require('axios');
const inquirer = require('inquirer');
const chalk = require('chalk');
const ora = require('ora');

const API_URL = process.env.PLATFORM_API_URL || 'https://platform.example.com/api';

program
  .name('platform')
  .description('Internal Developer Platform CLI')
  .version('1.0.0');

// Service creation
program
  .command('create-service <name>')
  .description('Create a new microservice from template')
  .option('-t, --template <template>', 'Service template', 'nodejs-microservice')
  .option('-o, --owner <team>', 'Owning team')
  .action(async (name, options) => {
    const answers = await inquirer.prompt([
      {
        type: 'input',
        name: 'description',
        message: 'Service description:',
        validate: v => v.length > 0,
      },
      {
        type: 'list',
        name: 'database',
        message: 'Database:',
        choices: ['postgres', 'mongodb', 'none'],
        default: 'postgres',
      },
      {
        type: 'confirm',
        name: 'kafka',
        message: 'Enable Kafka messaging?',
        default: true,
      },
      {
        type: 'confirm',
        name: 'redis',
        message: 'Enable Redis caching?',
        default: true,
      },
    ]);
    
    const spinner = ora('Creating service...').start();
    
    try {
      const response = await axios.post(`${API_URL}/services`, {
        name,
        template: options.template,
        owner: options.owner || await getTeamFromGitConfig(),
        description: answers.description,
        database: answers.database,
        kafkaEnabled: answers.kafka,
        redisEnabled: answers.redis,
      }, {
        headers: { Authorization: `Bearer ${getAuthToken()}` },
      });
      
      spinner.succeed('Service created!');
      
      console.log('\n' + chalk.green('✨ Your new service is ready!'));
      console.log(chalk.cyan(`Repository: ${response.data.repoUrl}`));
      console.log(chalk.cyan(`Catalog: ${response.data.catalogUrl}`));
      console.log(chalk.cyan(`CI/CD: ${response.data.cicdUrl}`));
      
    } catch (error) {
      spinner.fail('Failed to create service');
      console.error(chalk.red(error.response?.data?.message || error.message));
      process.exit(1);
    }
  });

// Database provisioning
program
  .command('provision-db')
  .description('Provision a new database')
  .action(async () => {
    const answers = await inquirer.prompt([
      { type: 'input', name: 'name', message: 'Database name:' },
      { type: 'list', name: 'dbType', message: 'Database type:', choices: ['postgres', 'mysql', 'mongodb'] },
      { type: 'list', name: 'size', message: 'Size:', choices: [
        { name: 'Small ($50/mo) - t3.small, 20GB', value: 'small' },
        { name: 'Medium ($150/mo) - t3.medium, 100GB', value: 'medium' },
        { name: 'Large ($400/mo) - m5.large, 500GB', value: 'large' },
      ]},
      { type: 'list', name: 'environment', message: 'Environment:', choices: ['staging', 'production'] },
      { type: 'confirm', name: 'ha', message: 'Enable High Availability (Multi-AZ)?', default: false },
    ]);
    
    const spinner = ora('Provisioning database...').start();
    
    try {
      const response = await axios.post(`${API_URL}/databases`, {
        ...answers,
        highAvailability: answers.ha,
      }, {
        headers: { Authorization: `Bearer ${getAuthToken()}` },
      });
      
      spinner.succeed('Provisioning started!');
      console.log(chalk.yellow(`Status: ${response.data.statusUrl}`));
      console.log('Check status with: ' + chalk.cyan(`platform db-status ${response.data.provisioningId}`));
      
    } catch (error) {
      spinner.fail('Provisioning failed');
      console.error(chalk.red(error.response?.data?.message || error.message));
    }
  });

// Service status dashboard
program
  .command('status <service>')
  .description('Show service status')
  .action(async (service) => {
    const spinner = ora('Fetching status...').start();
    
    try {
      const [health, metrics, alerts] = await Promise.all([
        axios.get(`${API_URL}/services/${service}/health`),
        axios.get(`${API_URL}/services/${service}/metrics`),
        axios.get(`${API_URL}/services/${service}/alerts`),
      ]);
      
      spinner.stop();
      
      const h = health.data;
      const m = metrics.data;
      
      console.log(chalk.bold(`\n📊 ${service} Status\n`));
      
      // Health
      const healthIcon = h.healthy ? chalk.green('✅') : chalk.red('❌');
      console.log(`${healthIcon} Health: ${h.healthy ? 'Healthy' : 'Degraded'}`);
      console.log(`   Replicas: ${h.readyReplicas}/${h.desiredReplicas}`);
      
      // Metrics
      console.log(`\n${chalk.bold('Metrics (last 5min):')}`);
      console.log(`   Request Rate: ${m.requestsPerSecond.toFixed(1)}/s`);
      console.log(`   Error Rate: ${(m.errorRate * 100).toFixed(2)}%`);
      console.log(`   p50 Latency: ${m.p50Latency}ms`);
      console.log(`   p99 Latency: ${m.p99Latency}ms`);
      
      // Active alerts
      if (alerts.data.length > 0) {
        console.log(`\n${chalk.bold(chalk.red('🚨 Active Alerts:'))}`);
        alerts.data.forEach(alert => {
          console.log(`   ${chalk.red('•')} ${alert.name}: ${alert.message}`);
        });
      }
      
    } catch (error) {
      spinner.fail(`Cannot get status for ${service}`);
      console.error(chalk.red(error.response?.data?.message || error.message));
    }
  });

function getAuthToken() {
  try {
    return require('fs').readFileSync(`${process.env.HOME}/.platform-token`, 'utf8').trim();
  } catch {
    console.error(chalk.red('Not logged in. Run: platform login'));
    process.exit(1);
  }
}

async function getTeamFromGitConfig() {
  try {
    const { execSync } = require('child_process');
    return execSync('git config user.team', { stdio: 'pipe' }).toString().trim();
  } catch {
    return null;
  }
}

program.parse(process.argv);
```

## 6. Service Health Aggregator

```javascript
// platform-api/services/healthAggregator.js
const axios = require('axios');
const { KubeConfig, CoreV1Api, AppsV1Api } = require('@kubernetes/client-node');

class ServiceHealthAggregator {
  constructor() {
    const kc = new KubeConfig();
    kc.loadFromDefault();
    this.k8sCoreApi = kc.makeApiClient(CoreV1Api);
    this.k8sAppsApi = kc.makeApiClient(AppsV1Api);
  }
  
  async getServiceHealth(serviceName, namespace = 'production') {
    const [deployment, pods, events] = await Promise.all([
      this.k8sAppsApi.readNamespacedDeployment(serviceName, namespace)
        .then(r => r.body).catch(() => null),
      this.k8sCoreApi.listNamespacedPod(namespace, undefined, undefined, undefined, undefined,
        `app=${serviceName}`).then(r => r.body.items).catch(() => []),
      this.k8sCoreApi.listNamespacedEvent(namespace, undefined, undefined, undefined,
        undefined, `involvedObject.name=${serviceName}`).then(r => r.body.items).catch(() => []),
    ]);
    
    const health = {
      service: serviceName,
      namespace,
      healthy: false,
      desiredReplicas: 0,
      readyReplicas: 0,
      availableReplicas: 0,
      pods: [],
      recentEvents: [],
      conditions: [],
    };
    
    if (deployment) {
      health.desiredReplicas = deployment.spec.replicas || 0;
      health.readyReplicas = deployment.status.readyReplicas || 0;
      health.availableReplicas = deployment.status.availableReplicas || 0;
      health.healthy = health.readyReplicas >= Math.ceil(health.desiredReplicas * 0.5);
      health.conditions = deployment.status.conditions || [];
    }
    
    health.pods = pods.map(pod => ({
      name: pod.metadata.name,
      phase: pod.status.phase,
      ready: pod.status.conditions?.find(c => c.type === 'Ready')?.status === 'True',
      restartCount: pod.status.containerStatuses?.[0]?.restartCount || 0,
      age: pod.metadata.creationTimestamp,
      node: pod.spec.nodeName,
      ip: pod.status.podIP,
    }));
    
    health.recentEvents = events
      .sort((a, b) => new Date(b.lastTimestamp) - new Date(a.lastTimestamp))
      .slice(0, 10)
      .map(e => ({
        type: e.type,
        reason: e.reason,
        message: e.message,
        count: e.count,
        timestamp: e.lastTimestamp,
      }));
    
    return health;
  }
  
  async getClusterHealth(namespace = 'production') {
    const { body: deployments } = await this.k8sAppsApi.listNamespacedDeployment(namespace);
    
    const services = await Promise.all(
      deployments.items.map(d => 
        this.getServiceHealth(d.metadata.name, namespace)
      )
    );
    
    return {
      namespace,
      totalServices: services.length,
      healthyServices: services.filter(s => s.healthy).length,
      degradedServices: services.filter(s => !s.healthy),
      services: services.sort((a, b) => {
        // Unhealthy first
        if (a.healthy !== b.healthy) return a.healthy ? 1 : -1;
        return a.service.localeCompare(b.service);
      }),
    };
  }
}

module.exports = ServiceHealthAggregator;
```

## 7. Golden Path Templates

```bash
#!/bin/bash
# scripts/create-service.sh
# Golden path for creating a new microservice

set -euo pipefail

SERVICE_NAME=$1
TEAM=$2

echo "Creating service: $SERVICE_NAME for team: $TEAM"

# 1. Create from template
git clone https://github.com/org/microservice-template.git $SERVICE_NAME
cd $SERVICE_NAME
rm -rf .git

# 2. Configure service
sed -i "s/TEMPLATE_NAME/$SERVICE_NAME/g" package.json
sed -i "s/TEMPLATE_NAME/$SERVICE_NAME/g" catalog-info.yaml
sed -i "s/TEMPLATE_TEAM/$TEAM/g" catalog-info.yaml

# 3. Initialize git
git init
git add .
git commit -m "Initial service scaffold from golden path template"

# 4. Create GitHub repo
gh repo create org/$SERVICE_NAME \
  --private \
  --description "Microservice: $SERVICE_NAME" \
  --team $TEAM

# 5. Push
git remote add origin https://github.com/org/$SERVICE_NAME.git
git push -u origin main

# 6. Setup branch protection
gh api repos/org/$SERVICE_NAME/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci/build","ci/test"]}' \
  --field enforce_admins=false \
  --field required_pull_request_reviews='{"required_approving_review_count":1}' \
  --field restrictions=null

# 7. Register in Backstage
curl -X POST https://backstage.internal.example.com/api/catalog/locations \
  -H "Authorization: Bearer $BACKSTAGE_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"type\": \"url\", \"target\": \"https://github.com/org/$SERVICE_NAME/blob/main/catalog-info.yaml\"}"

echo "✅ Service $SERVICE_NAME created successfully!"
echo "   GitHub: https://github.com/org/$SERVICE_NAME"
echo "   Backstage: https://backstage.internal.example.com/catalog/default/component/$SERVICE_NAME"
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Backstage IDP** - Software catalog, API documentation, tech docs ในที่เดียว
2. **Service Templates** - Scaffolding templates สำหรับ golden path development
3. **Self-Service Provisioning** - Developers provision databases/infrastructure เองโดยไม่รอ Ops
4. **Terraform Automation** - Infrastructure as Code ที่ triggered จาก developer portal
5. **Platform CLI** - Developer-friendly CLI สำหรับ create, deploy, monitor services
6. **Health Aggregator** - Kubernetes deployment health ด้วย k8s API
7. **Golden Path** - Opinionated template ที่ enforce best practices ตั้งแต่วันแรก

## แบบฝึกหัด

1. Setup Backstage instance ด้วย GitHub integration และ software catalog
2. สร้าง scaffolding template สำหรับ Python FastAPI microservice
3. Implement self-service database provisioning API ด้วย Terraform
4. สร้าง platform CLI ที่ support create-service, provision-db, และ status commands
5. Build health dashboard ที่แสดง status ทุก service ใน namespace พร้อม auto-refresh

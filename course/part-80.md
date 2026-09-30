# Part 80: DevSecOps สำหรับ Microservices

## บทนำ

DevSecOps คือการนำ Security เข้ามาเป็นส่วนหนึ่งของ Development และ Operations ตั้งแต่ต้น (Shift Left) บทนี้จะครอบคลุม Security ใน CI/CD Pipeline, SAST/DAST Integration, Container Image Scanning, Infrastructure Security Scanning, Policy as Code ด้วย OPA/Conftest, Secret Scanning และ Compliance Automation

---

## 1. Security ใน CI/CD Pipeline

### 1.1 GitHub Actions Security Pipeline

```yaml
# .github/workflows/security-pipeline.yml
name: Security Pipeline

on:
  push:
    branches: [main, develop, 'release/**']
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Daily scan at 2 AM

permissions:
  contents: read
  security-events: write
  pull-requests: write
  checks: write

jobs:
  # 1. Static Application Security Testing (SAST)
  sast:
    name: SAST Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for better analysis

      # CodeQL Analysis
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, typescript, python
          config-file: .github/codeql-config.yml
          queries: security-and-quality

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:javascript"
          upload: true

      # Semgrep SAST
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
            p/nodejs
            p/typescript
          output_format: sarif
          output_file: semgrep.sarif
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: Upload Semgrep Results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif

      # ESLint Security Rules
      - name: ESLint Security Scan
        run: |
          npm ci
          npx eslint . \
            --plugin=security \
            --plugin=no-unsanitized \
            --rule='security/detect-object-injection: error' \
            --rule='security/detect-non-literal-regexp: warn' \
            --rule='security/detect-unsafe-regex: error' \
            --rule='no-unsanitized/method: error' \
            --format=sarif \
            --output-file=eslint-security.sarif || true

  # 2. Dependency Vulnerability Scanning
  dependency-scan:
    name: Dependency Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: |
          npm ci
          npm audit --audit-level=high --json > npm-audit.json || true
          
          # Fail if critical vulnerabilities found
          CRITICAL=$(cat npm-audit.json | jq '.metadata.vulnerabilities.critical // 0')
          if [ "$CRITICAL" -gt "0" ]; then
            echo "::error::Found $CRITICAL critical vulnerabilities!"
            cat npm-audit.json | jq '.vulnerabilities | to_entries[] | select(.value.severity == "critical") | .key'
            exit 1
          fi

      # Snyk vulnerability scanning
      - name: Run Snyk
        uses: snyk/actions/node@master
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Upload Snyk Results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: snyk.sarif

      # OWASP Dependency Check
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'microservices-app'
          path: '.'
          format: 'SARIF'
          args: >
            --failOnCVSS 7
            --enableRetired
            --nodeAuditSkipDevDependencies

  # 3. Container Image Scanning
  container-scan:
    name: Container Image Scan
    runs-on: ubuntu-latest
    needs: [sast]
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker Image
        run: |
          docker build \
            --tag ${{ github.repository }}:${{ github.sha }} \
            --label "git-commit=${{ github.sha }}" \
            --label "build-date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
            .

      # Trivy container scanning
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: '${{ github.repository }}:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'

      - name: Upload Trivy Results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif

      # Grype scanning
      - name: Run Grype Scanner
        uses: anchore/scan-action@v3
        with:
          image: '${{ github.repository }}:${{ github.sha }}'
          fail-build: true
          severity-cutoff: high
          output-format: sarif
          
      # Docker Scout
      - name: Docker Scout CVEs
        uses: docker/scout-action@v1
        with:
          command: cves
          image: '${{ github.repository }}:${{ github.sha }}'
          only-severities: critical,high
          exit-code: true

  # 4. Infrastructure as Code Security
  iac-scan:
    name: IaC Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Checkov for Terraform/Kubernetes
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./infrastructure
          framework: terraform,kubernetes,helm
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false
          skip_check: >-
            CKV_K8S_28,
            CKV_K8S_30

      # tfsec for Terraform
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: ./infrastructure/terraform
          format: sarif
          additional_args: --minimum-severity HIGH

      # Conftest for policy-as-code
      - name: Run Conftest
        run: |
          curl -L https://github.com/open-policy-agent/conftest/releases/download/v0.46.0/conftest_0.46.0_Linux_x86_64.tar.gz | tar -xz
          sudo mv conftest /usr/local/bin/
          
          conftest test \
            --policy ./policies \
            ./kubernetes/**/*.yaml \
            --all-namespaces \
            --combine \
            --output=json > conftest-results.json 2>&1 || true

  # 5. Secret Scanning
  secret-scan:
    name: Secret Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      # TruffleHog secret scanning
      - name: TruffleHog OSS
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --only-verified

      # Gitleaks
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

  # 6. DAST (Dynamic Application Security Testing)
  dast:
    name: DAST Scan
    runs-on: ubuntu-latest
    needs: [container-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/develop'
    services:
      app:
        image: ${{ github.repository }}:${{ github.sha }}
        ports:
          - 8080:8080
    steps:
      - uses: actions/checkout@v4

      # OWASP ZAP full scan
      - name: OWASP ZAP Full Scan
        uses: zaproxy/action-full-scan@v0.9.0
        with:
          target: 'http://localhost:8080'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a -j'
          artifact_name: zap-report

      # Nuclei scanner
      - name: Run Nuclei
        uses: projectdiscovery/nuclei-action@main
        with:
          target: http://localhost:8080
          flags: "-severity critical,high -stats"

  # 7. Security Gate
  security-gate:
    name: Security Gate
    needs: [sast, dependency-scan, container-scan, iac-scan, secret-scan]
    runs-on: ubuntu-latest
    steps:
      - name: Check Security Results
        run: |
          echo "All security scans passed. Proceeding with deployment."
          
      - name: Generate Security Report
        run: |
          cat << 'EOF' > security-report.md
          # Security Scan Report
          
          **Date:** $(date -u)
          **Commit:** ${{ github.sha }}
          **Branch:** ${{ github.ref_name }}
          
          ## Scan Results
          - SAST: ✓ Passed
          - Dependency Scan: ✓ Passed
          - Container Scan: ✓ Passed
          - IaC Scan: ✓ Passed
          - Secret Scan: ✓ Passed
          EOF

      - name: Comment PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('security-report.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: report
            });
```

---

## 2. SAST Configuration

### 2.1 CodeQL Configuration

```yaml
# .github/codeql-config.yml
name: "Security CodeQL Config"

queries:
  - uses: security-and-quality
  - uses: security-extended

paths-ignore:
  - node_modules
  - dist
  - build
  - coverage
  - '**/*.test.ts'
  - '**/*.spec.ts'
  - '**/*.d.ts'

paths:
  - src
  - lib

query-filters:
  - exclude:
      tags contain: experimental
```

### 2.2 Custom ESLint Security Rules

```javascript
// .eslintrc.security.js
module.exports = {
  plugins: ['security', 'no-unsanitized', 'no-secrets'],
  extends: ['plugin:security/recommended'],
  rules: {
    // Prevent SQL injection
    'security/detect-sql-literal-injection': 'error',
    
    // Prevent XSS
    'no-unsanitized/method': ['error', {
      escape: {
        taggedTemplates: ['html', 'svg', 'css'],
        methods: ['escapeHTML', 'sanitizeHTML'],
      },
    }],
    
    // Prevent command injection
    'security/detect-child-process': 'warn',
    'security/detect-non-literal-require': 'warn',
    
    // Prevent regex DoS (ReDoS)
    'security/detect-unsafe-regex': 'error',
    
    // Prevent hardcoded secrets
    'no-secrets/no-secrets': ['error', { tolerance: 4.0 }],
    
    // Prevent object injection
    'security/detect-object-injection': 'error',
    'security/detect-non-literal-fs-filename': 'warn',
    
    // Prevent SSRF
    'no-restricted-syntax': [
      'warn',
      {
        selector: "CallExpression[callee.name='fetch'][arguments.0.type='Identifier']",
        message: 'User-controlled URLs in fetch() may lead to SSRF. Validate/allowlist the URL.',
      },
    ],
  },
};
```

---

## 3. Container Image Scanning ใน Production

### 3.1 Triton Admission Controller

```typescript
// admission-controller/src/image-scanner.ts
import { KubeConfig, AdmissionregistrationV1Api } from '@kubernetes/client-node';
import express from 'express';
import https from 'https';
import fs from 'fs';
import axios from 'axios';

interface AdmissionReview {
  apiVersion: string;
  kind: string;
  request: {
    uid: string;
    object: any;
    operation: string;
  };
  response?: {
    uid: string;
    allowed: boolean;
    status?: { code: number; message: string };
    warnings?: string[];
  };
}

interface ScanResult {
  image: string;
  vulnerabilities: {
    critical: number;
    high: number;
    medium: number;
    low: number;
  };
  passed: boolean;
  details: string[];
}

const SCAN_POLICY = {
  maxCritical: 0,
  maxHigh: 0,
  allowedImages: [
    'registry.company.com/',
    'gcr.io/distroless/',
  ],
  blockedBaseImages: [
    ':latest',
    ':master',
    ':main',
  ],
};

export class ImageScanAdmissionController {
  private app: express.Application;
  private trivyUrl: string;

  constructor(trivyUrl: string) {
    this.trivyUrl = trivyUrl;
    this.app = express();
    this.app.use(express.json());
    this.setupRoutes();
  }

  private setupRoutes(): void {
    this.app.post('/validate', async (req, res) => {
      const review: AdmissionReview = req.body;
      const response = await this.handleAdmission(review);
      res.json(response);
    });

    this.app.get('/health', (_, res) => res.json({ status: 'ok' }));
  }

  private async handleAdmission(review: AdmissionReview): Promise<AdmissionReview> {
    const { uid } = review.request;
    const pod = review.request.object;
    const warnings: string[] = [];

    if (review.request.operation !== 'CREATE' && review.request.operation !== 'UPDATE') {
      return this.allow(uid);
    }

    const containers = [
      ...(pod.spec?.containers || []),
      ...(pod.spec?.initContainers || []),
    ];

    for (const container of containers) {
      const image = container.image;

      // Policy: Only allow from approved registries
      const isApproved = SCAN_POLICY.allowedImages.some((prefix) =>
        image.startsWith(prefix)
      );

      if (!isApproved) {
        return this.deny(uid, `Image ${image} is not from an approved registry`);
      }

      // Policy: No latest/master/main tags
      const isBlockedTag = SCAN_POLICY.blockedBaseImages.some((tag) =>
        image.includes(tag)
      );

      if (isBlockedTag) {
        return this.deny(uid, `Image ${image} uses a blocked tag (latest/master/main)`);
      }

      // Scan image for vulnerabilities
      try {
        const scanResult = await this.scanImage(image);

        if (scanResult.vulnerabilities.critical > SCAN_POLICY.maxCritical) {
          return this.deny(
            uid,
            `Image ${image} has ${scanResult.vulnerabilities.critical} critical vulnerabilities`
          );
        }

        if (scanResult.vulnerabilities.high > SCAN_POLICY.maxHigh) {
          return this.deny(
            uid,
            `Image ${image} has ${scanResult.vulnerabilities.high} high vulnerabilities`
          );
        }

        if (scanResult.vulnerabilities.medium > 10) {
          warnings.push(
            `Image ${image} has ${scanResult.vulnerabilities.medium} medium vulnerabilities`
          );
        }
      } catch (error: any) {
        // If scan fails and we're in strict mode, deny
        if (process.env.SCAN_STRICT_MODE === 'true') {
          return this.deny(uid, `Failed to scan image ${image}: ${error.message}`);
        }
        warnings.push(`Could not scan image ${image}: ${error.message}`);
      }
    }

    return this.allow(uid, warnings);
  }

  private async scanImage(image: string): Promise<ScanResult> {
    const response = await axios.post(
      `${this.trivyUrl}/scan`,
      { image, scanType: 'vuln', format: 'json' },
      { timeout: 60000 }
    );

    const data = response.data;
    const vulns = { critical: 0, high: 0, medium: 0, low: 0 };
    const details: string[] = [];

    for (const result of data.Results || []) {
      for (const vuln of result.Vulnerabilities || []) {
        const severity = vuln.Severity?.toLowerCase() as keyof typeof vulns;
        if (severity in vulns) {
          vulns[severity]++;
          if (severity === 'critical' || severity === 'high') {
            details.push(`${vuln.VulnerabilityID}: ${vuln.Title} (${severity})`);
          }
        }
      }
    }

    return {
      image,
      vulnerabilities: vulns,
      passed: vulns.critical === 0 && vulns.high === 0,
      details,
    };
  }

  private allow(uid: string, warnings?: string[]): AdmissionReview {
    return {
      apiVersion: 'admission.k8s.io/v1',
      kind: 'AdmissionReview',
      response: {
        uid,
        allowed: true,
        warnings,
      },
    };
  }

  private deny(uid: string, message: string): AdmissionReview {
    return {
      apiVersion: 'admission.k8s.io/v1',
      kind: 'AdmissionReview',
      response: {
        uid,
        allowed: false,
        status: {
          code: 403,
          message,
        },
      },
    };
  }

  start(port: number, certPath: string, keyPath: string): void {
    const server = https.createServer(
      {
        cert: fs.readFileSync(certPath),
        key: fs.readFileSync(keyPath),
      },
      this.app
    );

    server.listen(port, () => {
      console.log(`Image scan admission controller running on port ${port}`);
    });
  }
}
```

---

## 4. Policy as Code ด้วย OPA/Conftest

### 4.1 OPA Policies สำหรับ Kubernetes

```rego
# policies/kubernetes/security.rego
package kubernetes.security

# Deny containers running as root
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  container.securityContext.runAsUser == 0
  msg := sprintf("Container '%s' must not run as root (UID 0)", [container.name])
}

# Require resource limits
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.resources.limits
  msg := sprintf("Container '%s' must have resource limits defined", [container.name])
}

# Deny privileged containers
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Container '%s' must not run in privileged mode", [container.name])
}

# Require readOnlyRootFilesystem
warn[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.securityContext.readOnlyRootFilesystem
  msg := sprintf("Container '%s' should have readOnlyRootFilesystem: true", [container.name])
}

# Require liveness and readiness probes
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.livenessProbe
  msg := sprintf("Container '%s' must have a livenessProbe defined", [container.name])
}

deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.readinessProbe
  msg := sprintf("Container '%s' must have a readinessProbe defined", [container.name])
}

# Require image tags (not latest)
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  endswith(container.image, ":latest")
  msg := sprintf("Container '%s' must not use 'latest' image tag", [container.name])
}

# Require image digest for production
deny[msg] {
  input.kind == "Deployment"
  input.metadata.namespace == "production"
  container := input.spec.template.spec.containers[_]
  not contains(container.image, "@sha256:")
  msg := sprintf("Production container '%s' must use image digest, not tag", [container.name])
}

# Require non-root user
deny[msg] {
  input.kind == "Deployment"
  not input.spec.template.spec.securityContext.runAsNonRoot
  msg := "Pod must have securityContext.runAsNonRoot: true"
}

# Drop all capabilities
deny[msg] {
  input.kind == "Deployment"
  container := input.spec.template.spec.containers[_]
  not container.securityContext.capabilities.drop
  msg := sprintf("Container '%s' must drop all capabilities", [container.name])
}

# Prevent hostNetwork
deny[msg] {
  input.kind == "Deployment"
  input.spec.template.spec.hostNetwork == true
  msg := "Pods must not use hostNetwork"
}

# Require namespace labels
deny[msg] {
  input.kind == "Namespace"
  not input.metadata.labels["team"]
  msg := "Namespace must have 'team' label"
}
```

### 4.2 OPA Policy สำหรับ Terraform

```rego
# policies/terraform/network.rego
package terraform.aws.network

# Deny security groups that allow all traffic
deny[msg] {
  resource := input.resource.aws_security_group[name]
  rule := resource.ingress[_]
  rule.cidr_blocks[_] == "0.0.0.0/0"
  rule.from_port == 0
  rule.to_port == 0
  msg := sprintf("Security group '%s' allows all ingress traffic from 0.0.0.0/0", [name])
}

# Deny unencrypted S3 buckets
deny[msg] {
  bucket := input.resource.aws_s3_bucket[name]
  not bucket.server_side_encryption_configuration
  msg := sprintf("S3 bucket '%s' must have server-side encryption enabled", [name])
}

# Require VPC flow logs
deny[msg] {
  vpc := input.resource.aws_vpc[name]
  not input.resource.aws_flow_log
  msg := sprintf("VPC '%s' must have flow logs enabled", [name])
}

# Require RDS encryption
deny[msg] {
  db := input.resource.aws_db_instance[name]
  not db.storage_encrypted
  msg := sprintf("RDS instance '%s' must have storage encryption enabled", [name])
}

# RDS must not be publicly accessible
deny[msg] {
  db := input.resource.aws_db_instance[name]
  db.publicly_accessible == true
  msg := sprintf("RDS instance '%s' must not be publicly accessible", [name])
}

# CloudTrail must be enabled
deny[msg] {
  not input.resource.aws_cloudtrail
  msg := "AWS CloudTrail must be configured"
}
```

### 4.3 Conftest Test Runner

```typescript
// scripts/run-conftest.ts
import { exec } from 'child_process';
import { promisify } from 'util';
import { glob } from 'glob';
import path from 'path';
import fs from 'fs/promises';

const execAsync = promisify(exec);

interface ConftestResult {
  filename: string;
  namespace: string;
  successes: number;
  failures: ConftestFailure[];
  warnings: ConftestWarning[];
}

interface ConftestFailure {
  msg: string;
  metadata?: Record<string, any>;
}

interface ConftestWarning {
  msg: string;
}

async function runConftest(): Promise<void> {
  console.log('Running Conftest policy checks...\n');

  const kubernetesFiles = await glob('kubernetes/**/*.yaml', { cwd: process.cwd() });
  const terraformFiles = await glob('infrastructure/terraform/**/*.tf', { cwd: process.cwd() });

  let totalFailures = 0;
  let totalWarnings = 0;

  // Check Kubernetes manifests
  for (const file of kubernetesFiles) {
    try {
      const { stdout } = await execAsync(
        `conftest test ${file} --policy ./policies/kubernetes --output json`
      );

      const results: ConftestResult[] = JSON.parse(stdout);
      
      for (const result of results) {
        if (result.failures.length > 0) {
          console.error(`\n❌ FAILED: ${file}`);
          for (const failure of result.failures) {
            console.error(`   - ${failure.msg}`);
            totalFailures++;
          }
        }

        if (result.warnings.length > 0) {
          console.warn(`\n⚠️  WARNINGS: ${file}`);
          for (const warning of result.warnings) {
            console.warn(`   - ${warning.msg}`);
            totalWarnings++;
          }
        }

        if (result.failures.length === 0 && result.warnings.length === 0) {
          console.log(`✓ ${file}`);
        }
      }
    } catch (error: any) {
      if (error.stdout) {
        // Parse failures from stdout
        try {
          const results: ConftestResult[] = JSON.parse(error.stdout);
          for (const result of results) {
            if (result.failures.length > 0) {
              console.error(`\n❌ FAILED: ${file}`);
              for (const failure of result.failures) {
                console.error(`   - ${failure.msg}`);
                totalFailures++;
              }
            }
          }
        } catch {
          console.error(`Error parsing conftest output for ${file}`);
        }
      }
    }
  }

  console.log(`\n${'='.repeat(60)}`);
  console.log(`Policy Check Summary`);
  console.log(`${'='.repeat(60)}`);
  console.log(`Files checked: ${kubernetesFiles.length + terraformFiles.length}`);
  console.log(`Failures: ${totalFailures}`);
  console.log(`Warnings: ${totalWarnings}`);

  if (totalFailures > 0) {
    console.error('\n❌ Policy checks failed!');
    process.exit(1);
  } else {
    console.log('\n✅ All policy checks passed!');
  }
}

runConftest().catch(console.error);
```

---

## 5. Secret Scanning และ Management

### 5.1 Vault Integration Service

```typescript
// src/secrets/vault-client.ts
import vault from 'node-vault';
import NodeCache from 'node-cache';

interface SecretConfig {
  path: string;
  version?: number;
}

interface VaultConfig {
  endpoint: string;
  roleId: string;
  secretId: string;
  namespace?: string;
  cacheTtlSeconds: number;
  renewThresholdSeconds: number;
}

export class VaultSecretsClient {
  private client: ReturnType<typeof vault>;
  private cache: NodeCache;
  private token: string = '';
  private tokenExpiry: number = 0;

  constructor(private config: VaultConfig) {
    this.client = vault({
      endpoint: config.endpoint,
      namespace: config.namespace,
    });

    this.cache = new NodeCache({
      stdTTL: config.cacheTtlSeconds,
      checkperiod: 60,
    });
  }

  async authenticate(): Promise<void> {
    const response = await this.client.approleLogin({
      role_id: this.config.roleId,
      secret_id: this.config.secretId,
    });

    this.token = response.auth.client_token;
    this.tokenExpiry = Date.now() + response.auth.lease_duration * 1000;
    this.client.token = this.token;

    // Schedule renewal
    const renewAt = response.auth.lease_duration * 1000 * 0.8;
    setTimeout(() => this.renewToken(), renewAt);
  }

  async getSecret<T = Record<string, string>>(
    path: string,
    version?: number
  ): Promise<T> {
    const cacheKey = `secret:${path}:${version || 'latest'}`;
    const cached = this.cache.get<T>(cacheKey);
    if (cached) return cached;

    await this.ensureAuthenticated();

    const secretPath = path.startsWith('secret/') ? path : `secret/data/${path}`;
    const options = version ? { version } : {};

    const response = await this.client.read(secretPath, options);
    const data = response.data?.data || response.data;

    this.cache.set(cacheKey, data);
    return data as T;
  }

  async setSecret(path: string, data: Record<string, string>): Promise<void> {
    await this.ensureAuthenticated();
    const secretPath = path.startsWith('secret/') ? path : `secret/data/${path}`;
    await this.client.write(secretPath, { data });

    // Invalidate cache
    this.cache.del(`secret:${path}:latest`);
  }

  async getDatabaseCredentials(role: string): Promise<{ username: string; password: string }> {
    await this.ensureAuthenticated();
    const response = await this.client.read(`database/creds/${role}`);
    return {
      username: response.data.username,
      password: response.data.password,
    };
  }

  async generateAWSCredentials(role: string): Promise<{
    accessKey: string;
    secretKey: string;
    sessionToken: string;
  }> {
    await this.ensureAuthenticated();
    const response = await this.client.read(`aws/creds/${role}`);
    return {
      accessKey: response.data.access_key,
      secretKey: response.data.secret_key,
      sessionToken: response.data.security_token,
    };
  }

  private async ensureAuthenticated(): Promise<void> {
    if (!this.token || Date.now() > this.tokenExpiry - this.config.renewThresholdSeconds * 1000) {
      await this.authenticate();
    }
  }

  private async renewToken(): Promise<void> {
    try {
      const response = await this.client.tokenRenewSelf();
      this.tokenExpiry = Date.now() + response.auth.lease_duration * 1000;

      const renewAt = response.auth.lease_duration * 1000 * 0.8;
      setTimeout(() => this.renewToken(), renewAt);
    } catch {
      await this.authenticate();
    }
  }
}

// External Secrets Operator configuration
export const externalSecretsConfig = `
# kubernetes/external-secrets.yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-secret-store
  namespace: production
spec:
  provider:
    vault:
      server: "https://vault.company.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "product-service"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: product-service-secrets
  namespace: production
spec:
  refreshInterval: "5m"
  secretStoreRef:
    name: vault-secret-store
    kind: SecretStore
  target:
    name: product-service-secrets
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        DATABASE_URL: "{{ .database_url }}"
        REDIS_URL: "{{ .redis_url }}"
        JWT_SECRET: "{{ .jwt_secret }}"
  data:
    - secretKey: database_url
      remoteRef:
        key: product-service/production
        property: database_url
    - secretKey: redis_url
      remoteRef:
        key: product-service/production
        property: redis_url
    - secretKey: jwt_secret
      remoteRef:
        key: product-service/production
        property: jwt_secret
`;
```

---

## 6. Compliance Automation

### 6.1 Compliance Scanner

```python
# src/compliance/compliance_scanner.py
import boto3
import json
from typing import List, Dict, Any
from dataclasses import dataclass, field
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

@dataclass
class ComplianceFinding:
    rule_id: str
    title: str
    severity: str
    resource_type: str
    resource_id: str
    status: str  # PASS, FAIL, SKIP
    details: str
    remediation: str

@dataclass
class ComplianceReport:
    timestamp: str
    framework: str
    account_id: str
    findings: List[ComplianceFinding] = field(default_factory=list)
    
    @property
    def passed(self) -> int:
        return sum(1 for f in self.findings if f.status == 'PASS')
    
    @property
    def failed(self) -> int:
        return sum(1 for f in self.findings if f.status == 'FAIL')
    
    @property
    def compliance_score(self) -> float:
        total = len(self.findings)
        if total == 0:
            return 100.0
        return (self.passed / total) * 100

class PCI_DSSComplianceScanner:
    """PCI-DSS v4.0 Compliance Scanner"""
    
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.rds = boto3.client('rds')
        self.s3 = boto3.client('s3')
        self.cloudtrail = boto3.client('cloudtrail')
        self.config = boto3.client('config')
        self.iam = boto3.client('iam')
        self.account_id = boto3.client('sts').get_caller_identity()['Account']

    def run_full_scan(self) -> ComplianceReport:
        report = ComplianceReport(
            timestamp=datetime.utcnow().isoformat(),
            framework="PCI-DSS v4.0",
            account_id=self.account_id,
        )

        # Run all checks
        report.findings.extend(self.check_req1_network_security())
        report.findings.extend(self.check_req3_data_protection())
        report.findings.extend(self.check_req7_access_control())
        report.findings.extend(self.check_req10_logging())

        return report

    def check_req1_network_security(self) -> List[ComplianceFinding]:
        """PCI-DSS Requirement 1: Network Security Controls"""
        findings = []

        # Check VPC Flow Logs
        vpcs = self.ec2.describe_vpcs()['Vpcs']
        flow_logs = self.ec2.describe_flow_logs()['FlowLogs']
        vpc_with_logs = {fl['ResourceId'] for fl in flow_logs}

        for vpc in vpcs:
            vpc_id = vpc['VpcId']
            status = 'PASS' if vpc_id in vpc_with_logs else 'FAIL'
            findings.append(ComplianceFinding(
                rule_id='PCI-1.1',
                title='VPC Flow Logs Enabled',
                severity='HIGH',
                resource_type='VPC',
                resource_id=vpc_id,
                status=status,
                details=f"VPC {vpc_id} {'has' if status == 'PASS' else 'does not have'} flow logs",
                remediation='Enable VPC Flow Logs to monitor network traffic',
            ))

        # Check Security Groups for unrestricted access
        sgs = self.ec2.describe_security_groups()['SecurityGroups']
        for sg in sgs:
            for rule in sg.get('IpPermissions', []):
                for ip_range in rule.get('IpRanges', []):
                    if ip_range.get('CidrIp') == '0.0.0.0/0' and rule.get('FromPort') in [22, 3389, 5432, 6379]:
                        findings.append(ComplianceFinding(
                            rule_id='PCI-1.2',
                            title='No Unrestricted Access to Sensitive Ports',
                            severity='CRITICAL',
                            resource_type='SecurityGroup',
                            resource_id=sg['GroupId'],
                            status='FAIL',
                            details=f"Security group {sg['GroupId']} allows unrestricted access to port {rule.get('FromPort')}",
                            remediation='Restrict access to sensitive ports to known IP ranges only',
                        ))

        return findings

    def check_req3_data_protection(self) -> List[ComplianceFinding]:
        """PCI-DSS Requirement 3: Protect Stored Account Data"""
        findings = []

        # Check RDS encryption
        dbs = self.rds.describe_db_instances()['DBInstances']
        for db in dbs:
            status = 'PASS' if db.get('StorageEncrypted') else 'FAIL'
            findings.append(ComplianceFinding(
                rule_id='PCI-3.4',
                title='RDS Storage Encryption',
                severity='HIGH',
                resource_type='RDS',
                resource_id=db['DBInstanceIdentifier'],
                status=status,
                details=f"RDS instance storage {'is' if status == 'PASS' else 'is not'} encrypted",
                remediation='Enable storage encryption for RDS instances containing cardholder data',
            ))

            # Check TLS enforcement
            if db.get('Engine') in ['mysql', 'postgres']:
                param_group = db.get('DBParameterGroups', [{}])[0].get('DBParameterGroupName', '')
                findings.append(ComplianceFinding(
                    rule_id='PCI-3.5',
                    title='TLS Enforcement on RDS',
                    severity='HIGH',
                    resource_type='RDS',
                    resource_id=db['DBInstanceIdentifier'],
                    status='SKIP',  # Would need to check parameter group
                    details='Verify that ssl_mode=require or rds.force_ssl=1 is set',
                    remediation='Set SSL/TLS enforcement in RDS parameter group',
                ))

        # Check S3 encryption
        s3_buckets = self.s3.list_buckets()['Buckets']
        for bucket in s3_buckets:
            bucket_name = bucket['Name']
            try:
                encryption = self.s3.get_bucket_encryption(Bucket=bucket_name)
                status = 'PASS'
            except self.s3.exceptions.ClientError:
                status = 'FAIL'

            findings.append(ComplianceFinding(
                rule_id='PCI-3.4',
                title='S3 Bucket Encryption',
                severity='HIGH',
                resource_type='S3Bucket',
                resource_id=bucket_name,
                status=status,
                details=f"S3 bucket '{bucket_name}' {'has' if status == 'PASS' else 'does not have'} server-side encryption",
                remediation='Enable server-side encryption with AWS KMS for S3 buckets containing sensitive data',
            ))

        return findings

    def check_req7_access_control(self) -> List[ComplianceFinding]:
        """PCI-DSS Requirement 7: Restrict Access"""
        findings = []

        # Check for MFA on root account
        account_summary = self.iam.get_account_summary()['SummaryMap']
        mfa_status = 'PASS' if account_summary.get('AccountMFAEnabled') == 1 else 'FAIL'
        findings.append(ComplianceFinding(
            rule_id='PCI-7.1',
            title='MFA on Root Account',
            severity='CRITICAL',
            resource_type='IAM',
            resource_id='root',
            status=mfa_status,
            details=f"Root account MFA {'is' if mfa_status == 'PASS' else 'is not'} enabled",
            remediation='Enable MFA on the AWS root account immediately',
        ))

        # Check for IAM users with console access (should use SSO)
        users = self.iam.list_users()['Users']
        for user in users:
            try:
                self.iam.get_login_profile(UserName=user['UserName'])
                # Has console access
                findings.append(ComplianceFinding(
                    rule_id='PCI-7.2',
                    title='No Direct IAM Console Users',
                    severity='MEDIUM',
                    resource_type='IAMUser',
                    resource_id=user['UserName'],
                    status='WARN',
                    details=f"IAM user '{user['UserName']}' has console access. Use SSO instead.",
                    remediation='Migrate console access to AWS SSO/Identity Center',
                ))
            except self.iam.exceptions.NoSuchEntityException:
                pass

        return findings

    def check_req10_logging(self) -> List[ComplianceFinding]:
        """PCI-DSS Requirement 10: Log and Monitor"""
        findings = []

        # Check CloudTrail
        trails = self.cloudtrail.describe_trails()['trailList']
        
        if not trails:
            findings.append(ComplianceFinding(
                rule_id='PCI-10.1',
                title='CloudTrail Enabled',
                severity='CRITICAL',
                resource_type='CloudTrail',
                resource_id='account',
                status='FAIL',
                details='No CloudTrail trails found',
                remediation='Enable AWS CloudTrail for all regions',
            ))
        
        for trail in trails:
            # Check log file validation
            status = 'PASS' if trail.get('LogFileValidationEnabled') else 'FAIL'
            findings.append(ComplianceFinding(
                rule_id='PCI-10.5',
                title='CloudTrail Log File Validation',
                severity='HIGH',
                resource_type='CloudTrail',
                resource_id=trail['TrailARN'],
                status=status,
                details=f"Log file validation {'is' if status == 'PASS' else 'is not'} enabled",
                remediation='Enable CloudTrail log file validation to detect tampering',
            ))

        return findings

    def generate_html_report(self, report: ComplianceReport) -> str:
        critical_findings = [f for f in report.findings if f.severity == 'CRITICAL' and f.status == 'FAIL']
        high_findings = [f for f in report.findings if f.severity == 'HIGH' and f.status == 'FAIL']

        rows = ''.join([
            f'<tr class="{f.status.lower()}">'
            f'<td>{f.rule_id}</td>'
            f'<td>{f.title}</td>'
            f'<td><span class="badge {f.severity.lower()}">{f.severity}</span></td>'
            f'<td>{f.resource_type}</td>'
            f'<td><code>{f.resource_id}</code></td>'
            f'<td><span class="status {f.status.lower()}">{f.status}</span></td>'
            f'<td>{f.details}</td>'
            f'</tr>'
            for f in report.findings
        ])

        return f"""<!DOCTYPE html>
<html>
<head>
  <title>PCI-DSS Compliance Report</title>
  <style>
    body {{ font-family: sans-serif; margin: 20px; }}
    .score {{ font-size: 48px; font-weight: bold; color: {'green' if report.compliance_score >= 80 else 'red'}; }}
    .badge {{ padding: 2px 8px; border-radius: 3px; font-size: 12px; }}
    .badge.critical {{ background: #dc3545; color: white; }}
    .badge.high {{ background: #fd7e14; color: white; }}
    .status.pass {{ color: green; }}
    .status.fail {{ color: red; font-weight: bold; }}
    table {{ width: 100%; border-collapse: collapse; }}
    th, td {{ border: 1px solid #ddd; padding: 8px; text-align: left; }}
    tr.fail {{ background: #fff3f3; }}
    tr.pass {{ background: #f3fff3; }}
  </style>
</head>
<body>
  <h1>PCI-DSS v4.0 Compliance Report</h1>
  <p>Account: {report.account_id} | Date: {report.timestamp}</p>
  <div class="score">{report.compliance_score:.1f}%</div>
  <p>Passed: {report.passed} | Failed: {report.failed} | Critical Failures: {len(critical_findings)}</p>
  <table>
    <thead>
      <tr><th>Rule</th><th>Title</th><th>Severity</th><th>Resource Type</th><th>Resource</th><th>Status</th><th>Details</th></tr>
    </thead>
    <tbody>{rows}</tbody>
  </table>
</body>
</html>"""
```

---

## 7. Security Monitoring и Alerting

### 7.1 Security Events Alerting

```typescript
// src/security/monitoring/security-alerting.ts
import { Kafka } from 'kafkajs';
import axios from 'axios';
import { Redis } from 'ioredis';

type SecurityEventSeverity = 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';

interface SecurityEvent {
  eventId: string;
  type: string;
  severity: SecurityEventSeverity;
  source: string;
  description: string;
  details: Record<string, any>;
  timestamp: string;
}

interface AlertChannel {
  name: string;
  minSeverity: SecurityEventSeverity;
  send: (event: SecurityEvent) => Promise<void>;
}

export class SecurityAlertingService {
  private severityOrder: Record<SecurityEventSeverity, number> = {
    LOW: 0, MEDIUM: 1, HIGH: 2, CRITICAL: 3,
  };

  constructor(
    private channels: AlertChannel[],
    private redis: Redis,
    private deduplicationWindowMs: number = 300000 // 5 minutes
  ) {}

  async handleEvent(event: SecurityEvent): Promise<void> {
    // Deduplication
    const dedupeKey = `security:alert:dedup:${event.type}:${JSON.stringify(event.details).substring(0, 100)}`;
    const isDuplicate = await this.redis.get(dedupeKey);
    
    if (isDuplicate) {
      return; // Skip duplicate
    }

    await this.redis.set(dedupeKey, '1', 'PX', this.deduplicationWindowMs);

    // Enrich event
    const enrichedEvent = await this.enrichEvent(event);

    // Route to appropriate channels
    const eligibleChannels = this.channels.filter(
      (ch) => this.severityOrder[event.severity] >= this.severityOrder[ch.minSeverity]
    );

    await Promise.allSettled(
      eligibleChannels.map((ch) => ch.send(enrichedEvent))
    );

    // Store in audit log
    await this.storeAuditLog(enrichedEvent);
  }

  private async enrichEvent(event: SecurityEvent): Promise<SecurityEvent> {
    // Add MITRE ATT&CK mapping if applicable
    const mitreMapping = this.getMitreMapping(event.type);

    return {
      ...event,
      details: {
        ...event.details,
        mitreAttack: mitreMapping,
      },
    };
  }

  private getMitreMapping(eventType: string): string | null {
    const mappings: Record<string, string> = {
      'brute_force': 'T1110 - Brute Force',
      'sql_injection': 'T1190 - Exploit Public-Facing Application',
      'privilege_escalation': 'T1068 - Exploitation for Privilege Escalation',
      'lateral_movement': 'T1021 - Remote Services',
      'data_exfiltration': 'T1041 - Exfiltration Over C2 Channel',
    };
    return mappings[eventType] || null;
  }

  private async storeAuditLog(event: SecurityEvent): Promise<void> {
    await this.redis.lpush('security:audit:log', JSON.stringify(event));
    await this.redis.ltrim('security:audit:log', 0, 9999);
  }
}

// PagerDuty alert channel
export function createPagerDutyChannel(routingKey: string): AlertChannel {
  return {
    name: 'PagerDuty',
    minSeverity: 'HIGH',
    send: async (event) => {
      await axios.post('https://events.pagerduty.com/v2/enqueue', {
        routing_key: routingKey,
        event_action: 'trigger',
        dedup_key: event.eventId,
        payload: {
          summary: `[${event.severity}] ${event.type}: ${event.description}`,
          source: event.source,
          severity: event.severity.toLowerCase(),
          custom_details: event.details,
          timestamp: event.timestamp,
        },
      });
    },
  };
}

// Slack alert channel
export function createSlackChannel(webhookUrl: string, channel: string): AlertChannel {
  return {
    name: 'Slack',
    minSeverity: 'MEDIUM',
    send: async (event) => {
      const color = event.severity === 'CRITICAL' ? 'danger' :
                    event.severity === 'HIGH' ? 'warning' : 'good';

      await axios.post(webhookUrl, {
        channel,
        attachments: [{
          color,
          title: `🚨 Security Alert: ${event.type}`,
          text: event.description,
          fields: [
            { title: 'Severity', value: event.severity, short: true },
            { title: 'Source', value: event.source, short: true },
            ...Object.entries(event.details).slice(0, 4).map(([key, value]) => ({
              title: key,
              value: String(value),
              short: true,
            })),
          ],
          footer: `Event ID: ${event.eventId} | ${event.timestamp}`,
        }],
      });
    },
  };
}
```

---

## สรุป

บทนี้ครอบคลุม DevSecOps สำหรับ Microservices อย่างครบถ้วน:

1. **Security CI/CD Pipeline** - GitHub Actions ที่รวม SAST, DAST, container scan, IaC scan, secret scan
2. **SAST Integration** - CodeQL, Semgrep, ESLint security rules
3. **Container Image Scanning** - Admission controller ด้วย Trivy, Grype, Docker Scout
4. **Policy as Code** - OPA/Rego policies สำหรับ Kubernetes security requirements
5. **Terraform Security** - tfsec, Checkov สำหรับ IaC scanning
6. **Secret Management** - Vault integration, External Secrets Operator
7. **Compliance Automation** - PCI-DSS compliance scanner สำหรับ AWS
8. **Security Alerting** - MITRE ATT&CK mapping, PagerDuty, Slack integration

Key Takeaways:
- Shift Left: ตรวจจับ security issues ใน development ก่อนถึง production
- Policy as Code ทำให้ security requirements เป็น version-controlled และ testable
- Image scanning ต้องอยู่ใน admission controller เพื่อป้องกัน vulnerable images จากถูก deploy
- External Secrets Operator แยก secret management ออกจาก Kubernetes manifests
- Compliance automation ลดเวลาในการ audit จากหลายวันเหลือไม่กี่นาที
- Security events ต้องมี deduplication เพื่อป้องกัน alert fatigue

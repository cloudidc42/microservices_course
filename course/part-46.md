# Part 46: CI/CD Pipeline ขั้นสูง

## บทนำ

CI/CD Pipeline ที่ดีคือหัวใจของการพัฒนา Microservices สมัยใหม่ ในบทนี้เราจะเรียนรู้วิธีสร้าง Pipeline ที่มีประสิทธิภาพสูง ครอบคลุมตั้งแต่ GitHub Actions matrix builds, Docker layer caching, การ deploy หลาย environment, Semantic Versioning ด้วย Conventional Commits, การสร้าง Changelog อัตโนมัติ, Deployment Gates, และการ Rollback อัตโนมัติ

---

## 1. โครงสร้างโปรเจกต์

```
microservices-cicd/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── cd-staging.yml
│   │   ├── cd-production.yml
│   │   ├── release.yml
│   │   └── rollback.yml
│   ├── CODEOWNERS
│   └── pull_request_template.md
├── services/
│   ├── user-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── .dockerignore
│   ├── order-service/
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── package.json
│   └── payment-service/
│       ├── src/
│       ├── Dockerfile
│       └── package.json
├── k8s/
│   ├── base/
│   ├── overlays/
│   │   ├── staging/
│   │   └── production/
├── scripts/
│   ├── smoke-test.sh
│   ├── rollback.sh
│   └── generate-changelog.sh
├── docker-compose.yml
└── release.config.js
```

---

## 2. GitHub Actions Matrix Builds

### 2.1 CI Workflow หลัก

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop, 'feature/**', 'fix/**']
  pull_request:
    branches: [main, develop]

env:
  REGISTRY: ghcr.io
  IMAGE_PREFIX: ${{ github.repository_owner }}

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ตรวจสอบว่า service ไหนมีการเปลี่ยนแปลง
  changes:
    runs-on: ubuntu-latest
    outputs:
      user-service: ${{ steps.filter.outputs.user-service }}
      order-service: ${{ steps.filter.outputs.order-service }}
      payment-service: ${{ steps.filter.outputs.payment-service }}
      shared: ${{ steps.filter.outputs.shared }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            user-service:
              - 'services/user-service/**'
            order-service:
              - 'services/order-service/**'
            payment-service:
              - 'services/payment-service/**'
            shared:
              - 'shared/**'
              - 'package.json'
              - 'tsconfig.json'

  # Matrix build สำหรับทุก service
  build-and-test:
    needs: changes
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        service:
          - name: user-service
            changed: ${{ needs.changes.outputs.user-service }}
            port: 3001
          - name: order-service
            changed: ${{ needs.changes.outputs.order-service }}
            port: 3002
          - name: payment-service
            changed: ${{ needs.changes.outputs.payment-service }}
            port: 3003
        node-version: [18, 20]
        exclude:
          # Skip ถ้า service ไม่มีการเปลี่ยนแปลง (และไม่ใช่ main branch)
          - service:
              changed: 'false'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
          cache-dependency-path: services/${{ matrix.service.name }}/package-lock.json

      - name: Install dependencies
        working-directory: services/${{ matrix.service.name }}
        run: npm ci --prefer-offline

      - name: Run linting
        working-directory: services/${{ matrix.service.name }}
        run: npm run lint

      - name: Run type checking
        working-directory: services/${{ matrix.service.name }}
        run: npm run type-check

      - name: Run unit tests
        working-directory: services/${{ matrix.service.name }}
        run: npm run test:unit -- --coverage
        env:
          NODE_ENV: test

      - name: Run integration tests
        working-directory: services/${{ matrix.service.name }}
        run: npm run test:integration
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db

      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        if: matrix.node-version == 20
        with:
          file: services/${{ matrix.service.name }}/coverage/lcov.info
          flags: ${{ matrix.service.name }}
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: SonarCloud scan
        if: matrix.node-version == 20 && github.event_name == 'pull_request'
        uses: SonarSource/sonarcloud-github-action@v2
        with:
          projectBaseDir: services/${{ matrix.service.name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

  # Security scanning
  security-scan:
    runs-on: ubuntu-latest
    needs: changes
    strategy:
      matrix:
        service: [user-service, order-service, payment-service]
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        with:
          args: --project-name=${{ matrix.service }} --severity-threshold=high
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: 'services/${{ matrix.service }}'
          severity: 'CRITICAL,HIGH'
          format: 'sarif'
          output: 'trivy-results-${{ matrix.service }}.sarif'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results-${{ matrix.service }}.sarif'

  # Docker build พร้อม layer caching
  docker-build:
    needs: [build-and-test, security-scan]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, order-service, payment-service]
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU (multi-arch support)
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: |
            image=moby/buildkit:latest
            network=host

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: services/${{ matrix.service }}
          file: services/${{ matrix.service }}/Dockerfile
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: |
            type=gha,scope=${{ matrix.service }}
            type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}:buildcache
          cache-to: |
            type=gha,mode=max,scope=${{ matrix.service }}
            type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}:buildcache,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}
            GIT_BRANCH=${{ github.ref_name }}
          provenance: true
          sbom: true

      - name: Sign Docker image
        if: github.event_name != 'pull_request'
        run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/${{ matrix.service }}@${{ steps.build.outputs.digest }}
        env:
          COSIGN_EXPERIMENTAL: true

  # Notify on failure
  notify-failure:
    needs: [build-and-test, docker-build]
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ CI Pipeline Failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*CI Pipeline Failed*\nRepository: ${{ github.repository }}\nBranch: ${{ github.ref_name }}\nCommit: ${{ github.sha }}\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 3. Docker Layer Caching แบบ Advanced

### 3.1 Multi-stage Dockerfile ที่ Optimized

```dockerfile
# services/user-service/Dockerfile
# syntax=docker/dockerfile:1.6

# ===== Stage 1: Dependencies =====
FROM node:20-alpine AS deps
WORKDIR /app

# Copy package files เท่านั้น (cache layer ที่ไม่ค่อยเปลี่ยน)
COPY package*.json ./
COPY tsconfig*.json ./

# ใช้ BuildKit cache mount สำหรับ npm cache
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production && \
    cp -R node_modules prod_node_modules && \
    npm ci

# ===== Stage 2: Builder =====
FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/tsconfig*.json ./
COPY src/ ./src/

# ใช้ BuildKit cache mount สำหรับ TypeScript build cache
RUN --mount=type=cache,target=/app/.tsbuildinfo \
    npm run build

# ===== Stage 3: Production =====
FROM node:20-alpine AS production

# Security: run as non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs

WORKDIR /app

# Copy production dependencies เท่านั้น
COPY --from=deps --chown=nodeuser:nodejs /app/prod_node_modules ./node_modules
COPY --from=builder --chown=nodeuser:nodejs /app/dist ./dist
COPY --chown=nodeuser:nodejs package.json ./

# Security: ไม่ใช้ root
USER nodeuser

# Metadata labels
ARG BUILD_DATE
ARG GIT_COMMIT
ARG GIT_BRANCH
LABEL org.opencontainers.image.created="${BUILD_DATE}" \
      org.opencontainers.image.revision="${GIT_COMMIT}" \
      org.opencontainers.image.source="https://github.com/org/repo" \
      org.opencontainers.image.ref.name="${GIT_BRANCH}"

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3001/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

CMD ["node", "dist/server.js"]
```

### 3.2 Docker Compose สำหรับ Local Development

```yaml
# docker-compose.yml
version: '3.9'

services:
  user-service:
    build:
      context: ./services/user-service
      dockerfile: Dockerfile
      target: production
      cache_from:
        - type=gha,scope=user-service
      args:
        BUILD_DATE: ${BUILD_DATE:-unknown}
        GIT_COMMIT: ${GIT_COMMIT:-unknown}
    image: microservices/user-service:local
    ports:
      - "3001:3001"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://admin:password@postgres:5432/users_db
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET:-dev-secret-change-in-prod}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - microservices-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  order-service:
    build:
      context: ./services/order-service
      dockerfile: Dockerfile
      target: production
      cache_from:
        - type=gha,scope=order-service
    image: microservices/order-service:local
    ports:
      - "3002:3002"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://admin:password@postgres:5432/orders_db
      USER_SERVICE_URL: http://user-service:3001
    depends_on:
      - user-service
      - postgres
    networks:
      - microservices-network
    restart: unless-stopped

  payment-service:
    build:
      context: ./services/payment-service
      dockerfile: Dockerfile
      target: production
      cache_from:
        - type=gha,scope=payment-service
    image: microservices/payment-service:local
    ports:
      - "3003:3003"
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://admin:password@postgres:5432/payments_db
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
    depends_on:
      - postgres
    networks:
      - microservices-network
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
      POSTGRES_MULTIPLE_DATABASES: users_db,orders_db,payments_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-multiple-dbs.sh:/docker-entrypoint-initdb.d/init-multiple-dbs.sh
    ports:
      - "5432:5432"
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-devpassword}
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  microservices-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  postgres_data:
  redis_data:
```

---

## 4. Semantic Versioning กับ Conventional Commits

### 4.1 Commitlint Configuration

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      [
        'feat',     // Feature ใหม่
        'fix',      // Bug fix
        'docs',     // Documentation
        'style',    // Formatting
        'refactor', // Code refactoring
        'perf',     // Performance improvement
        'test',     // Tests
        'build',    // Build system
        'ci',       // CI configuration
        'chore',    // Maintenance
        'revert',   // Revert commit
      ],
    ],
    'scope-enum': [
      2,
      'always',
      [
        'user-service',
        'order-service',
        'payment-service',
        'shared',
        'ci',
        'docker',
        'k8s',
        'deps',
      ],
    ],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [2, 'always', 200],
  },
};
```

### 4.2 Release Please Configuration

```json
// release-please-config.json
{
  "release-type": "node",
  "separate-pull-requests": true,
  "packages": {
    "services/user-service": {
      "release-type": "node",
      "changelog-path": "CHANGELOG.md",
      "bump-minor-pre-major": true,
      "bump-patch-for-minor-pre-major": true,
      "draft": false,
      "prerelease": false
    },
    "services/order-service": {
      "release-type": "node",
      "changelog-path": "CHANGELOG.md"
    },
    "services/payment-service": {
      "release-type": "node",
      "changelog-path": "CHANGELOG.md"
    }
  },
  "plugins": [
    {
      "type": "sentence-case"
    },
    {
      "type": "linked-versions",
      "groupName": "microservices-core",
      "components": ["user-service", "order-service", "payment-service"]
    }
  ]
}
```

### 4.3 Release Workflow

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write
  packages: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    outputs:
      releases_created: ${{ steps.release.outputs.releases_created }}
      user-service--release_created: ${{ steps.release.outputs['services/user-service--release_created'] }}
      user-service--version: ${{ steps.release.outputs['services/user-service--version'] }}
      order-service--release_created: ${{ steps.release.outputs['services/order-service--release_created'] }}
      order-service--version: ${{ steps.release.outputs['services/order-service--version'] }}
      payment-service--release_created: ${{ steps.release.outputs['services/payment-service--release_created'] }}
      payment-service--version: ${{ steps.release.outputs['services/payment-service--version'] }}
    steps:
      - uses: google-github-actions/release-please-action@v4
        id: release
        with:
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json

  # Build และ push Docker images สำหรับ release
  publish-release-images:
    needs: release-please
    if: needs.release-please.outputs.releases_created
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - service: user-service
            release_created: ${{ needs.release-please.outputs['user-service--release_created'] }}
            version: ${{ needs.release-please.outputs['user-service--version'] }}
          - service: order-service
            release_created: ${{ needs.release-please.outputs['order-service--release_created'] }}
            version: ${{ needs.release-please.outputs['order-service--version'] }}
          - service: payment-service
            release_created: ${{ needs.release-please.outputs['payment-service--release_created'] }}
            version: ${{ needs.release-please.outputs['payment-service--version'] }}
    steps:
      - uses: actions/checkout@v4
        if: matrix.release_created

      - name: Set up Docker Buildx
        if: matrix.release_created
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        if: matrix.release_created
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push versioned image
        if: matrix.release_created
        uses: docker/build-push-action@v5
        with:
          context: services/${{ matrix.service }}
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/${{ matrix.service }}:${{ matrix.version }}
            ghcr.io/${{ github.repository_owner }}/${{ matrix.service }}:latest
          cache-from: type=gha,scope=${{ matrix.service }}
          cache-to: type=gha,mode=max,scope=${{ matrix.service }}

      - name: Create deployment manifest
        if: matrix.release_created
        run: |
          echo "${{ matrix.service }}=${{ matrix.version }}" >> deployment-manifest.txt

      - name: Trigger deployment to staging
        if: matrix.release_created
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.DISPATCH_TOKEN }}
          event-type: deploy-staging
          client-payload: |
            {
              "service": "${{ matrix.service }}",
              "version": "${{ matrix.version }}",
              "image": "ghcr.io/${{ github.repository_owner }}/${{ matrix.service }}:${{ matrix.version }}"
            }
```

---

## 5. Multi-Environment Deployments

### 5.1 Staging Deployment

```yaml
# .github/workflows/cd-staging.yml
name: Deploy to Staging

on:
  repository_dispatch:
    types: [deploy-staging]
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to deploy'
        required: true
        type: choice
        options:
          - user-service
          - order-service
          - payment-service
          - all
      version:
        description: 'Version to deploy (leave empty for latest)'
        required: false
        type: string

env:
  ENVIRONMENT: staging
  K8S_NAMESPACE: microservices-staging
  CLUSTER_NAME: staging-cluster

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/checkout@v4

      - name: Set deployment variables
        id: vars
        run: |
          SERVICE="${{ github.event.client_payload.service || inputs.service }}"
          VERSION="${{ github.event.client_payload.version || inputs.version || 'latest' }}"
          echo "service=${SERVICE}" >> $GITHUB_OUTPUT
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "deploy_time=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_OUTPUT

      - name: Configure kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      - name: Set up kubeconfig
        run: |
          echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 -d > $HOME/.kube/config
          kubectl config use-context ${{ env.CLUSTER_NAME }}

      - name: Deploy service
        id: deploy
        run: |
          SERVICE="${{ steps.vars.outputs.service }}"
          VERSION="${{ steps.vars.outputs.version }}"
          
          if [ "$SERVICE" = "all" ]; then
            SERVICES=("user-service" "order-service" "payment-service")
          else
            SERVICES=("$SERVICE")
          fi
          
          for svc in "${SERVICES[@]}"; do
            echo "Deploying $svc version $VERSION..."
            
            # Update image in Kubernetes
            kubectl set image deployment/${svc} \
              ${svc}=ghcr.io/${{ github.repository_owner }}/${svc}:${VERSION} \
              -n ${{ env.K8S_NAMESPACE }}
            
            # Annotate deployment for tracking
            kubectl annotate deployment/${svc} \
              deployment.kubernetes.io/version="${VERSION}" \
              deployment.kubernetes.io/deploy-time="${{ steps.vars.outputs.deploy_time }}" \
              deployment.kubernetes.io/deployed-by="${{ github.actor }}" \
              -n ${{ env.K8S_NAMESPACE }} \
              --overwrite
            
            # Wait for rollout
            kubectl rollout status deployment/${svc} \
              -n ${{ env.K8S_NAMESPACE }} \
              --timeout=300s || {
                echo "Deployment failed for $svc"
                kubectl rollout undo deployment/${svc} -n ${{ env.K8S_NAMESPACE }}
                exit 1
              }
          done

      - name: Run smoke tests
        run: |
          chmod +x ./scripts/smoke-test.sh
          ./scripts/smoke-test.sh staging ${{ steps.vars.outputs.service }}
        env:
          BASE_URL: https://staging.myapp.com
          API_KEY: ${{ secrets.STAGING_API_KEY }}

      - name: Run synthetic monitoring check
        run: |
          # ตรวจสอบ synthetic monitoring ผ่าน Datadog
          curl -sf -X POST "https://api.datadoghq.com/api/v1/synthetics/tests/trigger/ci" \
            -H "DD-API-KEY: ${{ secrets.DD_API_KEY }}" \
            -H "DD-APPLICATION-KEY: ${{ secrets.DD_APP_KEY }}" \
            -H "Content-Type: application/json" \
            -d '{
              "tests": [
                {"public_id": "abc-123-def", "startUrl": "https://staging.myapp.com"},
                {"public_id": "xyz-456-uvw", "startUrl": "https://staging.myapp.com/api/health"}
              ]
            }' | jq -r '.results[] | select(.result.passed == false) | .result' | \
            (! grep -q .)

      - name: Notify deployment success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Staging Deployment Successful",
              "attachments": [{
                "color": "good",
                "fields": [
                  {"title": "Service", "value": "${{ steps.vars.outputs.service }}", "short": true},
                  {"title": "Version", "value": "${{ steps.vars.outputs.version }}", "short": true},
                  {"title": "Environment", "value": "Staging", "short": true},
                  {"title": "Deployed by", "value": "${{ github.actor }}", "short": true}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify deployment failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Staging Deployment Failed",
              "attachments": [{
                "color": "danger",
                "fields": [
                  {"title": "Service", "value": "${{ steps.vars.outputs.service }}", "short": true},
                  {"title": "Version", "value": "${{ steps.vars.outputs.version }}", "short": true},
                  {"title": "View Logs", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 5.2 Production Deployment พร้อม Gates

```yaml
# .github/workflows/cd-production.yml
name: Deploy to Production

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to deploy'
        required: true
        type: choice
        options:
          - user-service
          - order-service
          - payment-service
      version:
        description: 'Version to deploy'
        required: true
        type: string
      deploy_strategy:
        description: 'Deployment strategy'
        required: true
        type: choice
        options:
          - rolling-update
          - blue-green
          - canary
        default: rolling-update

env:
  ENVIRONMENT: production
  K8S_NAMESPACE: microservices-production

jobs:
  # Deployment Gate 1: Pre-flight checks
  pre-flight-checks:
    runs-on: ubuntu-latest
    outputs:
      staging_verified: ${{ steps.verify-staging.outputs.verified }}
      tests_passed: ${{ steps.verify-tests.outputs.passed }}
    steps:
      - name: Verify staging deployment exists
        id: verify-staging
        run: |
          # ตรวจสอบว่า version นี้ถูก deploy ไปยัง staging แล้ว
          STAGING_VERSION=$(kubectl get deployment ${{ inputs.service }} \
            -n microservices-staging \
            -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/version}')
          
          if [ "$STAGING_VERSION" = "${{ inputs.version }}" ]; then
            echo "verified=true" >> $GITHUB_OUTPUT
            echo "✅ Version ${{ inputs.version }} found in staging"
          else
            echo "verified=false" >> $GITHUB_OUTPUT
            echo "❌ Version ${{ inputs.version }} not found in staging (found: $STAGING_VERSION)"
            exit 1
          fi
        env:
          KUBECONFIG_DATA: ${{ secrets.STAGING_KUBECONFIG }}

      - name: Verify all tests passed for this version
        id: verify-tests
        run: |
          # ตรวจสอบผ่าน GitHub API ว่า CI ผ่าน
          STATUS=$(gh api repos/${{ github.repository }}/commits/$(git rev-list -n 1 ${{ inputs.service }}-v${{ inputs.version }})/check-runs \
            --jq '[.check_runs[] | select(.name | startswith("build-and-test")) | .conclusion] | all(. == "success")')
          
          if [ "$STATUS" = "true" ]; then
            echo "passed=true" >> $GITHUB_OUTPUT
          else
            echo "passed=false" >> $GITHUB_OUTPUT
            exit 1
          fi
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # Deployment Gate 2: Manual approval สำหรับ production
  production-approval:
    needs: pre-flight-checks
    runs-on: ubuntu-latest
    environment:
      name: production-approval
    steps:
      - name: Approval checkpoint
        run: echo "Production deployment approved"

  # Deploy to production
  deploy-production:
    needs: [pre-flight-checks, production-approval]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/setup-kubectl@v3

      - name: Set up kubeconfig
        run: |
          echo "${{ secrets.PROD_KUBECONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Record pre-deployment state
        id: pre-deploy
        run: |
          CURRENT_VERSION=$(kubectl get deployment ${{ inputs.service }} \
            -n ${{ env.K8S_NAMESPACE }} \
            -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/version}')
          echo "previous_version=${CURRENT_VERSION}" >> $GITHUB_OUTPUT
          
          CURRENT_IMAGE=$(kubectl get deployment ${{ inputs.service }} \
            -n ${{ env.K8S_NAMESPACE }} \
            -o jsonpath='{.spec.template.spec.containers[0].image}')
          echo "previous_image=${CURRENT_IMAGE}" >> $GITHUB_OUTPUT

      - name: Deploy with strategy
        id: deploy
        run: |
          STRATEGY="${{ inputs.deploy_strategy }}"
          SERVICE="${{ inputs.service }}"
          VERSION="${{ inputs.version }}"
          IMAGE="ghcr.io/${{ github.repository_owner }}/${SERVICE}:${VERSION}"
          
          case $STRATEGY in
            rolling-update)
              kubectl set image deployment/${SERVICE} \
                ${SERVICE}=${IMAGE} \
                -n ${{ env.K8S_NAMESPACE }}
              
              kubectl rollout status deployment/${SERVICE} \
                -n ${{ env.K8S_NAMESPACE }} \
                --timeout=600s
              ;;
            
            blue-green)
              # สร้าง green deployment
              kubectl apply -f - <<EOF
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: ${SERVICE}-green
            namespace: ${{ env.K8S_NAMESPACE }}
            labels:
              app: ${SERVICE}
              slot: green
              version: ${VERSION}
          spec:
            replicas: $(kubectl get deployment ${SERVICE} -n ${{ env.K8S_NAMESPACE }} -o jsonpath='{.spec.replicas}')
            selector:
              matchLabels:
                app: ${SERVICE}
                slot: green
            template:
              metadata:
                labels:
                  app: ${SERVICE}
                  slot: green
              spec:
                containers:
                - name: ${SERVICE}
                  image: ${IMAGE}
                  ports:
                  - containerPort: 3000
          EOF
              
              # รอให้ green deployment พร้อม
              kubectl rollout status deployment/${SERVICE}-green \
                -n ${{ env.K8S_NAMESPACE }} --timeout=300s
              
              # Switch traffic ไปยัง green
              kubectl patch service ${SERVICE} \
                -n ${{ env.K8S_NAMESPACE }} \
                -p '{"spec":{"selector":{"slot":"green"}}}'
              
              # ลบ blue deployment เก่า
              kubectl delete deployment ${SERVICE}-blue \
                -n ${{ env.K8S_NAMESPACE }} --ignore-not-found
              
              # Rename green เป็น blue สำหรับครั้งหน้า
              kubectl get deployment ${SERVICE}-green \
                -n ${{ env.K8S_NAMESPACE }} -o yaml | \
                sed 's/name: '${SERVICE}'-green/name: '${SERVICE}'-blue/' | \
                kubectl apply -f -
              ;;
            
            canary)
              # Deploy canary ที่ 10% ของ traffic
              TOTAL_REPLICAS=$(kubectl get deployment ${SERVICE} \
                -n ${{ env.K8S_NAMESPACE }} -o jsonpath='{.spec.replicas}')
              CANARY_REPLICAS=$(( TOTAL_REPLICAS / 10 + 1 ))
              
              kubectl apply -f - <<EOF
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: ${SERVICE}-canary
            namespace: ${{ env.K8S_NAMESPACE }}
            annotations:
              deployment.kubernetes.io/canary: "true"
              deployment.kubernetes.io/version: "${VERSION}"
          spec:
            replicas: ${CANARY_REPLICAS}
            selector:
              matchLabels:
                app: ${SERVICE}
                track: canary
            template:
              metadata:
                labels:
                  app: ${SERVICE}
                  track: canary
              spec:
                containers:
                - name: ${SERVICE}
                  image: ${IMAGE}
          EOF
              
              echo "canary_deployed=true" >> $GITHUB_OUTPUT
              echo "Canary deployed with ${CANARY_REPLICAS}/${TOTAL_REPLICAS} replicas"
              ;;
          esac

      - name: Run production smoke tests
        run: |
          sleep 30  # รอให้ service พร้อม
          ./scripts/smoke-test.sh production ${{ inputs.service }}
        env:
          BASE_URL: https://myapp.com
          API_KEY: ${{ secrets.PROD_API_KEY }}

      - name: Monitor error rates (5 minutes)
        run: |
          echo "Monitoring error rates for 5 minutes..."
          for i in {1..10}; do
            ERROR_RATE=$(curl -sf \
              "https://api.datadoghq.com/api/v1/query?from=$(date -d '1 minute ago' +%s)&to=$(date +%s)&query=sum:${{ inputs.service }}.errors{env:production}.as_rate()" \
              -H "DD-API-KEY: ${{ secrets.DD_API_KEY }}" \
              -H "DD-APPLICATION-KEY: ${{ secrets.DD_APP_KEY }}" | \
              jq '.series[0].pointlist[-1][1] // 0')
            
            echo "Error rate: $ERROR_RATE errors/sec"
            
            if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
              echo "❌ Error rate too high: $ERROR_RATE. Triggering rollback..."
              exit 1
            fi
            
            sleep 30
          done
          echo "✅ Error rates within acceptable limits"

      - name: Finalize canary deployment
        if: inputs.deploy_strategy == 'canary' && success()
        run: |
          # Promote canary to stable
          kubectl set image deployment/${{ inputs.service }} \
            ${{ inputs.service }}=ghcr.io/${{ github.repository_owner }}/${{ inputs.service }}:${{ inputs.version }} \
            -n ${{ env.K8S_NAMESPACE }}
          
          kubectl rollout status deployment/${{ inputs.service }} \
            -n ${{ env.K8S_NAMESPACE }} --timeout=300s
          
          # ลบ canary deployment
          kubectl delete deployment ${{ inputs.service }}-canary \
            -n ${{ env.K8S_NAMESPACE }} --ignore-not-found

      - name: Rollback on failure
        if: failure() && steps.deploy.outcome == 'success'
        run: |
          echo "🔄 Rolling back to version ${{ steps.pre-deploy.outputs.previous_version }}..."
          kubectl rollout undo deployment/${{ inputs.service }} \
            -n ${{ env.K8S_NAMESPACE }}
          
          kubectl rollout status deployment/${{ inputs.service }} \
            -n ${{ env.K8S_NAMESPACE }} --timeout=300s
          
          echo "✅ Rollback completed"

      - name: Create deployment record
        if: success()
        run: |
          gh api repos/${{ github.repository }}/deployments \
            -f ref="${{ inputs.version }}" \
            -f environment="production" \
            -f description="Deployed ${{ inputs.service }} v${{ inputs.version }}" \
            -F auto_merge=false \
            -F required_contexts='[]'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 6. Automated Rollback

### 6.1 Rollback Workflow

```yaml
# .github/workflows/rollback.yml
name: Emergency Rollback

on:
  workflow_dispatch:
    inputs:
      service:
        description: 'Service to rollback'
        required: true
        type: choice
        options:
          - user-service
          - order-service
          - payment-service
          - all-services
      environment:
        description: 'Environment to rollback'
        required: true
        type: choice
        options:
          - staging
          - production
      revision:
        description: 'Revision number to rollback to (leave empty for previous)'
        required: false
        type: string
      reason:
        description: 'Reason for rollback (for audit log)'
        required: true
        type: string

jobs:
  emergency-rollback:
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}-emergency
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: |
          if [ "${{ inputs.environment }}" = "production" ]; then
            echo "${{ secrets.PROD_KUBECONFIG }}" | base64 -d > $HOME/.kube/config
          else
            echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 -d > $HOME/.kube/config
          fi

      - name: Get rollback information
        id: rollback-info
        run: |
          NAMESPACE="microservices-${{ inputs.environment }}"
          SERVICE="${{ inputs.service }}"
          
          if [ "$SERVICE" = "all-services" ]; then
            SERVICES=("user-service" "order-service" "payment-service")
          else
            SERVICES=("$SERVICE")
          fi
          
          for svc in "${SERVICES[@]}"; do
            echo "=== Rollback history for $svc ==="
            kubectl rollout history deployment/$svc -n $NAMESPACE
            
            CURRENT=$(kubectl get deployment $svc -n $NAMESPACE \
              -o jsonpath='{.spec.template.spec.containers[0].image}')
            echo "Current image: $CURRENT"
          done

      - name: Execute rollback
        id: rollback
        run: |
          NAMESPACE="microservices-${{ inputs.environment }}"
          SERVICE="${{ inputs.service }}"
          REVISION="${{ inputs.revision }}"
          
          if [ "$SERVICE" = "all-services" ]; then
            SERVICES=("user-service" "order-service" "payment-service")
          else
            SERVICES=("$SERVICE")
          fi
          
          for svc in "${SERVICES[@]}"; do
            echo "Rolling back $svc..."
            
            if [ -n "$REVISION" ]; then
              kubectl rollout undo deployment/$svc \
                --to-revision=$REVISION \
                -n $NAMESPACE
            else
              kubectl rollout undo deployment/$svc -n $NAMESPACE
            fi
            
            kubectl rollout status deployment/$svc \
              -n $NAMESPACE --timeout=300s || {
                echo "❌ Rollback failed for $svc"
                exit 1
              }
            
            NEW_IMAGE=$(kubectl get deployment $svc -n $NAMESPACE \
              -o jsonpath='{.spec.template.spec.containers[0].image}')
            echo "✅ $svc rolled back to: $NEW_IMAGE"
          done

      - name: Verify rollback success
        run: |
          NAMESPACE="microservices-${{ inputs.environment }}"
          sleep 30
          ./scripts/smoke-test.sh ${{ inputs.environment }} ${{ inputs.service }}
        env:
          BASE_URL: ${{ inputs.environment == 'production' && 'https://myapp.com' || 'https://staging.myapp.com' }}
          API_KEY: ${{ inputs.environment == 'production' && secrets.PROD_API_KEY || secrets.STAGING_API_KEY }}

      - name: Create incident record
        run: |
          gh issue create \
            --title "🚨 Emergency Rollback: ${{ inputs.service }} in ${{ inputs.environment }}" \
            --body "## Rollback Details
          
          **Service:** ${{ inputs.service }}
          **Environment:** ${{ inputs.environment }}
          **Triggered by:** ${{ github.actor }}
          **Timestamp:** $(date -u)
          **Reason:** ${{ inputs.reason }}
          
          ## Actions Taken
          - Emergency rollback executed
          - Smoke tests passed after rollback
          
          ## Next Steps
          - [ ] Investigate root cause
          - [ ] Fix the issue
          - [ ] Re-deploy when ready
          " \
            --label "incident,rollback" \
            --assignee ${{ github.actor }}
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Send emergency notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚨 EMERGENCY ROLLBACK EXECUTED",
              "attachments": [{
                "color": "danger",
                "fields": [
                  {"title": "Service", "value": "${{ inputs.service }}", "short": true},
                  {"title": "Environment", "value": "${{ inputs.environment }}", "short": true},
                  {"title": "Triggered by", "value": "${{ github.actor }}", "short": true},
                  {"title": "Reason", "value": "${{ inputs.reason }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 7. Smoke Tests Script

```bash
#!/bin/bash
# scripts/smoke-test.sh
# Production-ready smoke test script

set -euo pipefail

ENVIRONMENT="${1:-staging}"
SERVICE="${2:-all}"
BASE_URL="${BASE_URL:-https://staging.myapp.com}"
API_KEY="${API_KEY:-}"
MAX_RETRIES=5
RETRY_DELAY=10
TIMEOUT=30

# สีสำหรับ output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Function สำหรับทำ HTTP request พร้อม retry
make_request() {
    local url="$1"
    local expected_status="${2:-200}"
    local method="${3:-GET}"
    local data="${4:-}"
    local retries=0
    
    while [ $retries -lt $MAX_RETRIES ]; do
        HTTP_STATUS=$(curl -sf \
            -o /tmp/response.json \
            -w "%{http_code}" \
            -X "$method" \
            -H "Authorization: Bearer $API_KEY" \
            -H "Content-Type: application/json" \
            --max-time $TIMEOUT \
            ${data:+-d "$data"} \
            "$url" 2>/dev/null || echo "000")
        
        if [ "$HTTP_STATUS" = "$expected_status" ]; then
            return 0
        fi
        
        retries=$((retries + 1))
        if [ $retries -lt $MAX_RETRIES ]; then
            log_warn "Request failed (status: $HTTP_STATUS), retrying in ${RETRY_DELAY}s... ($retries/$MAX_RETRIES)"
            sleep $RETRY_DELAY
        fi
    done
    
    log_error "Request to $url failed after $MAX_RETRIES attempts (last status: $HTTP_STATUS)"
    return 1
}

# Smoke tests สำหรับแต่ละ service
test_user_service() {
    log_info "Testing User Service..."
    
    # Health check
    make_request "$BASE_URL/api/users/health" 200 || return 1
    
    # ตรวจสอบ API version
    VERSION=$(cat /tmp/response.json | jq -r '.version // empty')
    [ -n "$VERSION" ] && log_info "User Service version: $VERSION"
    
    # ทดสอบ endpoint หลัก
    make_request "$BASE_URL/api/users" 200 || return 1
    
    # ทดสอบ authentication
    make_request "$BASE_URL/api/auth/token" 200 "POST" \
        '{"username": "smoketest@example.com", "password": "SmokeTest123!"}' || return 1
    
    TEST_TOKEN=$(cat /tmp/response.json | jq -r '.access_token')
    
    # ทดสอบ authenticated endpoint
    HTTP_STATUS=$(curl -sf \
        -o /dev/null \
        -w "%{http_code}" \
        -H "Authorization: Bearer $TEST_TOKEN" \
        "$BASE_URL/api/users/profile")
    
    [ "$HTTP_STATUS" = "200" ] || {
        log_error "Authenticated profile endpoint failed: $HTTP_STATUS"
        return 1
    }
    
    log_info "✅ User Service smoke tests passed"
    return 0
}

test_order_service() {
    log_info "Testing Order Service..."
    
    make_request "$BASE_URL/api/orders/health" 200 || return 1
    make_request "$BASE_URL/api/orders" 200 || return 1
    
    log_info "✅ Order Service smoke tests passed"
    return 0
}

test_payment_service() {
    log_info "Testing Payment Service..."
    
    make_request "$BASE_URL/api/payments/health" 200 || return 1
    
    # ทดสอบ payment methods (ไม่ต้องสร้าง real payment)
    make_request "$BASE_URL/api/payments/methods" 200 || return 1
    
    log_info "✅ Payment Service smoke tests passed"
    return 0
}

test_connectivity() {
    log_info "Testing service connectivity..."
    
    # ทดสอบว่า services คุยกันได้
    make_request "$BASE_URL/api/orders/with-user-details" 200 || {
        log_error "Order service cannot communicate with User service"
        return 1
    }
    
    log_info "✅ Service connectivity tests passed"
    return 0
}

# Main execution
FAILED_TESTS=()

if [ "$SERVICE" = "all" ] || [ "$SERVICE" = "user-service" ]; then
    test_user_service || FAILED_TESTS+=("user-service")
fi

if [ "$SERVICE" = "all" ] || [ "$SERVICE" = "order-service" ]; then
    test_order_service || FAILED_TESTS+=("order-service")
fi

if [ "$SERVICE" = "all" ] || [ "$SERVICE" = "payment-service" ]; then
    test_payment_service || FAILED_TESTS+=("payment-service")
fi

if [ "$SERVICE" = "all" ]; then
    test_connectivity || FAILED_TESTS+=("connectivity")
fi

# Report results
if [ ${#FAILED_TESTS[@]} -eq 0 ]; then
    log_info "🎉 All smoke tests passed for $ENVIRONMENT!"
    exit 0
else
    log_error "❌ Smoke tests failed for: ${FAILED_TESTS[*]}"
    exit 1
fi
```

---

## 8. Kubernetes Manifests

### 8.1 Base Configuration

```yaml
# k8s/base/user-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  labels:
    app: user-service
    part-of: microservices
  annotations:
    deployment.kubernetes.io/version: "0.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: user-service
        version: latest
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3001"
        prometheus.io/path: "/metrics"
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
                        - user-service
                topologyKey: kubernetes.io/hostname
      
      terminationGracePeriodSeconds: 60
      
      containers:
        - name: user-service
          image: ghcr.io/org/user-service:latest
          imagePullPolicy: Always
          ports:
            - name: http
              containerPort: 3001
              protocol: TCP
          
          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3001"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: user-service-secrets
                  key: jwt-secret
          
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            failureThreshold: 30
            periodSeconds: 10
          
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
          
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1001
            capabilities:
              drop:
                - ALL
      
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: user-service

---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: user-service

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
```

---

## 9. Health Check Endpoints

```typescript
// services/user-service/src/health/health.controller.ts
import { Controller, Get } from '@nestjs/common';
import { InjectConnection } from '@nestjs/typeorm';
import { Connection } from 'typeorm';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  version: string;
  timestamp: string;
  uptime: number;
  checks: Record<string, CheckResult>;
}

interface CheckResult {
  status: 'pass' | 'fail' | 'warn';
  latencyMs?: number;
  message?: string;
}

@Controller('health')
export class HealthController {
  private readonly startTime = Date.now();

  constructor(
    @InjectConnection() private readonly db: Connection,
    @InjectRedis() private readonly redis: Redis,
  ) {}

  @Get('live')
  async liveness(): Promise<{ status: string }> {
    // Liveness probe: ถ้า return ได้ = process ยังทำงานอยู่
    return { status: 'ok' };
  }

  @Get('ready')
  async readiness(): Promise<HealthStatus> {
    const checks: Record<string, CheckResult> = {};

    // Check database
    const dbStart = Date.now();
    try {
      await this.db.query('SELECT 1');
      checks.database = {
        status: 'pass',
        latencyMs: Date.now() - dbStart,
      };
    } catch (error) {
      checks.database = {
        status: 'fail',
        message: 'Database connection failed',
      };
    }

    // Check Redis
    const redisStart = Date.now();
    try {
      await this.redis.ping();
      checks.redis = {
        status: 'pass',
        latencyMs: Date.now() - redisStart,
      };
    } catch (error) {
      checks.redis = {
        status: 'warn',
        message: 'Redis connection failed (degraded mode)',
      };
    }

    const allPassed = Object.values(checks).every((c) => c.status === 'pass');
    const hasFail = Object.values(checks).some((c) => c.status === 'fail');

    const status: HealthStatus = {
      status: hasFail ? 'unhealthy' : allPassed ? 'healthy' : 'degraded',
      version: process.env.npm_package_version || '0.0.0',
      timestamp: new Date().toISOString(),
      uptime: Math.floor((Date.now() - this.startTime) / 1000),
      checks,
    };

    // Return 503 ถ้า unhealthy
    if (hasFail) {
      throw Object.assign(new Error('Service unhealthy'), {
        getStatus: () => 503,
        getResponse: () => status,
      });
    }

    return status;
  }

  @Get('startup')
  async startup(): Promise<{ status: string }> {
    // Startup probe: ตรวจสอบว่า initialization เสร็จแล้ว
    const isReady = await this.checkInitialization();
    if (!isReady) {
      throw Object.assign(new Error('Not started'), {
        getStatus: () => 503,
      });
    }
    return { status: 'started' };
  }

  private async checkInitialization(): Promise<boolean> {
    try {
      // ตรวจสอบ database migrations ทำเสร็จแล้ว
      const migrations = await this.db.query(
        'SELECT COUNT(*) FROM migrations WHERE status = $1',
        ['pending'],
      );
      if (parseInt(migrations[0].count) > 0) return false;

      return true;
    } catch {
      return false;
    }
  }
}
```

---

## 10. Changelog Generation Script

```bash
#!/bin/bash
# scripts/generate-changelog.sh
# สร้าง CHANGELOG.md โดยอัตโนมัติจาก conventional commits

set -euo pipefail

SERVICE="${1:-all}"
OUTPUT_FILE="${2:-CHANGELOG.md}"
SINCE_TAG="${3:-}"

# ดึง commits ตาม conventional commits format
generate_section() {
    local type="$1"
    local title="$2"
    local since="$3"
    
    local commits
    if [ -n "$since" ]; then
        commits=$(git log "${since}..HEAD" --pretty=format:"%s|%H|%an|%ad" \
            --date=short --no-merges 2>/dev/null || echo "")
    else
        commits=$(git log --pretty=format:"%s|%H|%an|%ad" \
            --date=short --no-merges 2>/dev/null || echo "")
    fi
    
    local section=""
    while IFS='|' read -r subject hash author date; do
        if echo "$subject" | grep -qE "^${type}(\([^)]+\))?:"; then
            local scope=$(echo "$subject" | sed -n 's/^'${type}'(\([^)]*\)):.*/\1/p')
            local message=$(echo "$subject" | sed 's/^'${type}'([^)]*): //' | sed 's/^'${type}': //')
            local short_hash="${hash:0:7}"
            
            if [ -n "$scope" ]; then
                section+="- **${scope}**: ${message} ([${short_hash}](../../commit/${hash}))\n"
            else
                section+="- ${message} ([${short_hash}](../../commit/${hash}))\n"
            fi
        fi
    done <<< "$commits"
    
    if [ -n "$section" ]; then
        echo -e "### ${title}\n\n${section}"
    fi
}

# หา version ล่าสุดจาก git tags
LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
SINCE="${SINCE_TAG:-$LATEST_TAG}"

# สร้าง changelog
cat > "$OUTPUT_FILE" << EOF
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased] - $(date +%Y-%m-%d)

EOF

# เพิ่มแต่ละ section
features=$(generate_section "feat" "Features" "$SINCE")
[ -n "$features" ] && echo "$features" >> "$OUTPUT_FILE"

fixes=$(generate_section "fix" "Bug Fixes" "$SINCE")
[ -n "$fixes" ] && echo "$fixes" >> "$OUTPUT_FILE"

perf=$(generate_section "perf" "Performance Improvements" "$SINCE")
[ -n "$perf" ] && echo "$perf" >> "$OUTPUT_FILE"

docs=$(generate_section "docs" "Documentation" "$SINCE")
[ -n "$docs" ] && echo "$docs" >> "$OUTPUT_FILE"

# เพิ่ม Breaking changes
BREAKING=$(git log "${SINCE}..HEAD" --pretty=format:"%B" 2>/dev/null | \
    grep -A1 "BREAKING CHANGE:" | grep -v "BREAKING CHANGE:" | \
    grep -v "^--$" || echo "")

if [ -n "$BREAKING" ]; then
    echo -e "### ⚠ BREAKING CHANGES\n" >> "$OUTPUT_FILE"
    echo "$BREAKING" | while read -r line; do
        [ -n "$line" ] && echo "- $line" >> "$OUTPUT_FILE"
    done
    echo "" >> "$OUTPUT_FILE"
fi

echo "✅ Changelog generated: $OUTPUT_FILE"
```

---

## สรุป

| หัวข้อ | เทคโนโลยี | วัตถุประสงค์ |
|--------|-----------|-------------|
| Matrix Builds | GitHub Actions matrix strategy | Build และ test หลาย service พร้อมกัน |
| Docker Caching | BuildKit cache mounts + GHA cache | ลด build time ลงสูงสุด 80% |
| Multi-env Deploy | Staging → Production gates | ป้องกัน broken code เข้า production |
| Semantic Versioning | Conventional Commits + Release Please | Version อัตโนมัติตาม commit type |
| Deployment Gates | Smoke tests + Synthetic monitoring | ตรวจสอบ deploy ก่อน promote |
| Rollback | kubectl rollout undo + automation | กู้คืนอัตโนมัติเมื่อ error rate สูง |
| Security Scan | Snyk + Trivy | ตรวจหา vulnerability ใน code และ image |
| Changelog | Conventional commits parser | สร้าง CHANGELOG.md อัตโนมัติ |

> **Best Practice**: ทุก deploy ไปยัง production ต้องผ่าน staging ก่อน และต้องมี smoke test ผ่าน ถ้า error rate สูงกว่า 5% ภายใน 5 นาที จะ rollback อัตโนมัติ

# Part 49: CI/CD Pipeline

## CI/CD คืออะไร?

**CI (Continuous Integration)**: การ integrate code จากหลาย developer เข้าด้วยกันบ่อยๆ (ทุก commit) พร้อม automated tests

**CD (Continuous Delivery)**: การ build และ test code อัตโนมัติ พร้อม deploy ไป staging environment โดยอัตโนมัติ แต่ production ต้อง manual approve

**CD (Continuous Deployment)**: เหมือน Delivery แต่ deploy ไป production อัตโนมัติด้วย (ไม่ต้อง manual approve)

```
Code Push → CI (Build + Test) → CD (Deploy to Staging) → Manual Approve → Deploy to Production
            Continuous Integration   Continuous Delivery
                                                         ↑ Continuous Deployment (ไม่มี step นี้)
```

---

## GitHub Actions Overview

### โครงสร้าง Workflow

```yaml
# .github/workflows/ci.yml
name: Workflow Name

on:                              # Triggers
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'         # Daily at 2am
  workflow_dispatch:             # Manual trigger

jobs:
  job-name:
    runs-on: ubuntu-latest       # Runner
    steps:
      - name: Step Name
        uses: actions/checkout@v4  # Action
      - name: Another Step
        run: echo "Hello"          # Shell command
```

### Triggers ที่ใช้บ่อย

```yaml
on:
  push:
    branches: [main, 'release/*']
    tags: ['v*']
    paths:
      - 'src/**'
      - 'package.json'
    paths-ignore:
      - '**.md'
      - 'docs/**'

  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy target'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
      debug:
        description: 'Enable debug mode'
        type: boolean
        default: false
```

---

## CI Pipeline แบบสมบูรณ์

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  POSTGRES_DB: test_db
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: password

jobs:
  # ============================================================
  # Job 1: Lint and Type Check
  # ============================================================
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run Prettier check
        run: npm run format:check

      - name: Run TypeScript type check
        run: npm run type-check

  # ============================================================
  # Job 2: Unit Tests
  # ============================================================
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:unit -- --coverage --ci

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          flags: unit
          fail_ci_if_error: false

  # ============================================================
  # Job 3: Integration Tests
  # ============================================================
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: ${{ env.POSTGRES_DB }}
          POSTGRES_USER: ${{ env.POSTGRES_USER }}
          POSTGRES_PASSWORD: ${{ env.POSTGRES_PASSWORD }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db
      TEST_DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db
      REDIS_URL: redis://localhost:6379
      TEST_REDIS_URL: redis://localhost:6379/1
      JWT_SECRET: ${{ secrets.JWT_SECRET_TEST }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run database migrations
        run: npx prisma migrate deploy

      - name: Run integration tests
        run: npm run test:integration -- --coverage --ci

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          flags: integration

  # ============================================================
  # Job 4: Security Audit
  # ============================================================
  security:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: npm audit
        run: npm audit --audit-level=high

      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  # ============================================================
  # Job 5: Build Docker Image (only on push to main)
  # ============================================================
  build:
    name: Build Docker Image
    needs: [lint, unit-tests, integration-tests]
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
```

---

## CD Pipeline แบบสมบูรณ์

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options: [staging, production]

jobs:
  # ============================================================
  # Deploy to Staging (auto)
  # ============================================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push staging image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:staging
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Deploy to staging server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            # Pull latest image
            docker pull ghcr.io/${{ github.repository }}:staging
            
            # Run migrations
            docker run --rm \
              --env-file /etc/myapp/staging.env \
              ghcr.io/${{ github.repository }}:staging \
              npx prisma migrate deploy
            
            # Stop old container
            docker stop myapp-staging || true
            docker rm myapp-staging || true
            
            # Start new container
            docker run -d \
              --name myapp-staging \
              --restart unless-stopped \
              -p 3001:3000 \
              --env-file /etc/myapp/staging.env \
              ghcr.io/${{ github.repository }}:staging
            
            # Health check
            sleep 10
            curl -f http://localhost:3001/health || exit 1

      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "✅ Staging deploy successful: ${{ github.sha }}",
              "channel": "#deployments"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "❌ Staging deploy failed! Branch: ${{ github.ref }}, SHA: ${{ github.sha }}",
              "channel": "#deployments"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # ============================================================
  # Deploy to Production (manual approve required)
  # ============================================================
  deploy-production:
    name: Deploy to Production
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    # GitHub will require manual approval before running this job
    # Set this in: Settings → Environments → production → Required reviewers

    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Tag staging image as production
        run: |
          docker pull ghcr.io/${{ github.repository }}:staging
          docker tag ghcr.io/${{ github.repository }}:staging ghcr.io/${{ github.repository }}:production
          docker tag ghcr.io/${{ github.repository }}:staging ghcr.io/${{ github.repository }}:${{ github.sha }}
          docker push ghcr.io/${{ github.repository }}:production
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Deploy to production (Blue-Green)
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            # ============================================
            # Blue-Green Deployment
            # ============================================
            
            # Determine current color
            CURRENT=$(cat /etc/myapp/current_color 2>/dev/null || echo "blue")
            if [ "$CURRENT" = "blue" ]; then
              NEXT="green"
              NEXT_PORT=3001
            else
              NEXT="blue"
              NEXT_PORT=3000
            fi
            
            echo "Current: $CURRENT, Deploying to: $NEXT ($NEXT_PORT)"
            
            # Pull new image
            docker pull ghcr.io/${{ github.repository }}:${{ github.sha }}
            
            # Run migrations (before starting new version)
            docker run --rm \
              --env-file /etc/myapp/production.env \
              ghcr.io/${{ github.repository }}:${{ github.sha }} \
              npx prisma migrate deploy
            
            # Start new version
            docker stop myapp-$NEXT || true
            docker rm myapp-$NEXT || true
            
            docker run -d \
              --name myapp-$NEXT \
              --restart unless-stopped \
              -p $NEXT_PORT:3000 \
              --env-file /etc/myapp/production.env \
              ghcr.io/${{ github.repository }}:${{ github.sha }}
            
            # Health check new version
            sleep 15
            for i in {1..5}; do
              if curl -sf http://localhost:$NEXT_PORT/health; then
                echo "Health check passed"
                break
              fi
              echo "Health check attempt $i failed, waiting..."
              sleep 5
              if [ $i -eq 5 ]; then
                echo "Health check failed! Rolling back..."
                docker stop myapp-$NEXT
                exit 1
              fi
            done
            
            # Switch nginx to new version
            sed -i "s/proxy_pass http:\/\/localhost:[0-9]*/proxy_pass http:\/\/localhost:$NEXT_PORT/" /etc/nginx/sites-available/myapp
            nginx -t && nginx -s reload
            
            # Stop old version (after nginx switch)
            sleep 5
            docker stop myapp-$CURRENT || true
            
            # Update current color
            echo $NEXT > /etc/myapp/current_color
            
            echo "✅ Deployment complete. Now serving from $NEXT ($NEXT_PORT)"

      - name: Notify deployment complete
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "text": "🚀 Production deployment complete! SHA: ${{ github.sha }}",
              "channel": "#deployments"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Docker Multi-Stage Build

```dockerfile
# Dockerfile

# ============================================================
# Stage 1: Base dependencies
# ============================================================
FROM node:20-alpine AS base

WORKDIR /app

# Install only production dependencies first (cache optimization)
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# ============================================================
# Stage 2: Builder (compile TypeScript)
# ============================================================
FROM node:20-alpine AS builder

WORKDIR /app

# Copy base dependencies
COPY --from=base /app/node_modules ./node_modules

# Install all dependencies (including devDependencies for build)
COPY package*.json ./
RUN npm ci && npm cache clean --force

# Copy source code
COPY tsconfig.json ./
COPY src ./src
COPY prisma ./prisma

# Generate Prisma client
RUN npx prisma generate

# Compile TypeScript
RUN npm run build

# ============================================================
# Stage 3: Runner (production image)
# ============================================================
FROM node:20-alpine AS runner

# Security: create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy production node_modules from base stage
COPY --from=base /app/node_modules ./node_modules

# Copy compiled JS from builder stage
COPY --from=builder /app/dist ./dist

# Copy Prisma files (needed for migrations at runtime)
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY prisma ./prisma

# Copy package.json for version info
COPY package.json ./

# Change ownership to non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

# Start application
CMD ["node", "dist/server.js"]
```

```yaml
# docker-compose.yml (สำหรับ development)
version: '3.8'

services:
  app:
    build:
      context: .
      target: runner
    ports:
      - '3000:3000'
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/myapp
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    volumes:
      - ./prisma:/app/prisma  # mount prisma for migrations

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'postgres']
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

---

## Environment-Specific Configuration

```yaml
# .github/workflows/deploy-env.yml
name: Deploy to Environment

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        type: choice
        options: [staging, production]

jobs:
  deploy:
    name: Deploy to ${{ github.event.inputs.environment }}
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
      - uses: actions/checkout@v4

      - name: Set environment config
        id: config
        run: |
          if [ "${{ github.event.inputs.environment }}" = "production" ]; then
            echo "host=${{ secrets.PROD_HOST }}" >> $GITHUB_OUTPUT
            echo "port=3000" >> $GITHUB_OUTPUT
            echo "replicas=3" >> $GITHUB_OUTPUT
          else
            echo "host=${{ secrets.STAGING_HOST }}" >> $GITHUB_OUTPUT
            echo "port=3001" >> $GITHUB_OUTPUT
            echo "replicas=1" >> $GITHUB_OUTPUT
          fi

      - name: Deploy
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ steps.config.outputs.host }}
          username: deploy
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            echo "Deploying to ${{ github.event.inputs.environment }}"
            echo "Port: ${{ steps.config.outputs.port }}"
            echo "Replicas: ${{ steps.config.outputs.replicas }}"
```

---

## Database Migration ใน CD

```bash
# scripts/migrate.sh
#!/bin/bash
set -e

echo "Starting database migration..."

# Check if database is ready
max_attempts=30
attempt=1
while ! pg_isready -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" 2>/dev/null; do
  if [ $attempt -eq $max_attempts ]; then
    echo "Database not ready after $max_attempts attempts"
    exit 1
  fi
  echo "Waiting for database... (attempt $attempt/$max_attempts)"
  sleep 2
  attempt=$((attempt + 1))
done

echo "Database is ready. Running migrations..."

# Backup current schema (production only)
if [ "$NODE_ENV" = "production" ]; then
  echo "Creating schema backup..."
  pg_dump "$DATABASE_URL" --schema-only > /tmp/schema_backup_$(date +%Y%m%d_%H%M%S).sql
fi

# Run Prisma migrations
npx prisma migrate deploy

echo "Migrations complete!"
```

```yaml
# In CI/CD pipeline:
- name: Run database migrations
  run: |
    # Wait for DB
    until pg_isready -h localhost -p 5432 -U postgres; do
      echo "Waiting for database..."
      sleep 2
    done
    
    # Run migrations
    npx prisma migrate deploy
  env:
    DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp
```

---

## Rollback Strategy

```bash
# scripts/rollback.sh
#!/bin/bash

# Rollback สำหรับ Blue-Green deployment
CURRENT=$(cat /etc/myapp/current_color)
if [ "$CURRENT" = "blue" ]; then
  PREVIOUS="green"
  PREVIOUS_PORT=3001
else
  PREVIOUS="blue"
  PREVIOUS_PORT=3000
fi

echo "Rolling back from $CURRENT to $PREVIOUS"

# Check if previous version is still running
if docker ps --filter "name=myapp-$PREVIOUS" --format "{{.Names}}" | grep -q "myapp-$PREVIOUS"; then
  # Switch nginx back
  sed -i "s/proxy_pass http:\/\/localhost:[0-9]*/proxy_pass http:\/\/localhost:$PREVIOUS_PORT/" /etc/nginx/sites-available/myapp
  nginx -t && nginx -s reload
  
  # Update color file
  echo $PREVIOUS > /etc/myapp/current_color
  
  echo "✅ Rolled back to $PREVIOUS version"
else
  echo "❌ Previous version is not running! Cannot rollback automatically."
  echo "Manual intervention required."
  exit 1
fi
```

### Rollback Database Migration

```typescript
// prisma/migrations/rollback.ts
// Prisma ไม่มี built-in rollback แต่ทำได้โดย:
// 1. สร้าง new migration ที่ reverse การเปลี่ยนแปลง
// 2. ใช้ prisma migrate diff เพื่อสร้าง SQL สำหรับ rollback

// ตัวอย่าง rollback migration:
// npx prisma migrate resolve --rolled-back "20240101_add_column"
// แล้วสร้าง migration ใหม่ที่ reverse การ change
```

---

## Blue-Green Deployment แบบละเอียด

```
                    ┌─────────────────┐
                    │   Load Balancer  │
                    │    (Nginx)       │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                              │
    ┌─────────▼────────┐          ┌─────────▼────────┐
    │   Blue (Active)  │          │  Green (Standby) │
    │   Port 3000      │          │   Port 3001      │
    │   Version 1.0    │          │   Version 1.1    │
    └──────────────────┘          └──────────────────┘
              │                              │
    ┌─────────▼────────────────────────────▼────────┐
    │              PostgreSQL Database                │
    └───────────────────────────────────────────────┘
```

**ขั้นตอน Blue-Green Deployment:**

1. Green กำลัง run อยู่ (production traffic)
2. Deploy version ใหม่ไปที่ Blue
3. Run migrations
4. Health check Blue
5. Switch load balancer จาก Green → Blue
6. Monitor Blue
7. ถ้าปัญหา: switch กลับ Green ทันที (< 1 minute)
8. ถ้าดี: shutdown Green

```nginx
# /etc/nginx/sites-available/myapp
upstream myapp {
    server localhost:3000;  # Blue
    # server localhost:3001;  # Green (switch by commenting/uncommenting)
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://myapp;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 30s;
        proxy_read_timeout 30s;
    }
    
    location /health {
        proxy_pass http://myapp/health;
        access_log off;
    }
}
```

---

## Health Check After Deployment

```typescript
// scripts/health-check.ts
import axios from 'axios';

interface HealthCheckOptions {
  url: string;
  maxRetries?: number;
  retryDelay?: number;
  timeout?: number;
}

async function healthCheck(options: HealthCheckOptions): Promise<boolean> {
  const { url, maxRetries = 10, retryDelay = 5000, timeout = 10000 } = options;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      console.log(`Health check attempt ${attempt}/${maxRetries}: ${url}`);

      const response = await axios.get(url, {
        timeout,
        validateStatus: null,
      });

      if (response.status === 200 && response.data.status === 'ok') {
        console.log(`✅ Health check passed on attempt ${attempt}`);
        return true;
      }

      console.log(`Health check returned status ${response.status}`);
    } catch (error) {
      console.log(`Health check failed: ${(error as Error).message}`);
    }

    if (attempt < maxRetries) {
      console.log(`Waiting ${retryDelay}ms before retry...`);
      await new Promise(resolve => setTimeout(resolve, retryDelay));
    }
  }

  console.error(`❌ Health check failed after ${maxRetries} attempts`);
  return false;
}

// Health check endpoint ใน app
// GET /health
// {
//   "status": "ok",
//   "timestamp": "2024-01-01T00:00:00.000Z",
//   "version": "1.2.3",
//   "uptime": 3600,
//   "database": "connected",
//   "redis": "connected"
// }

async function healthEndpoint(req: any, res: any) {
  const checks = {
    database: 'unknown',
    redis: 'unknown',
  };

  try {
    await prisma.$queryRaw`SELECT 1`;
    checks.database = 'connected';
  } catch {
    checks.database = 'error';
  }

  try {
    await redis.ping();
    checks.redis = 'connected';
  } catch {
    checks.redis = 'error';
  }

  const allHealthy = Object.values(checks).every(v => v === 'connected');

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'ok' : 'error',
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version,
    uptime: process.uptime(),
    ...checks,
  });
}
```

---

## Notification: Slack Integration

```yaml
# .github/workflows/notify.yml

# Reusable workflow for Slack notifications
on:
  workflow_call:
    inputs:
      status:
        required: true
        type: string
      environment:
        required: true
        type: string
      version:
        required: true
        type: string
    secrets:
      SLACK_WEBHOOK_URL:
        required: true

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1.26.0
        with:
          payload: |
            {
              "blocks": [
                {
                  "type": "header",
                  "text": {
                    "type": "plain_text",
                    "text": "${{ inputs.status == 'success' && '✅' || '❌' }} Deploy ${{ inputs.status }}"
                  }
                },
                {
                  "type": "section",
                  "fields": [
                    {
                      "type": "mrkdwn",
                      "text": "*Environment:*\n${{ inputs.environment }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Version:*\n${{ inputs.version }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Branch:*\n${{ github.ref_name }}"
                    },
                    {
                      "type": "mrkdwn",
                      "text": "*Triggered by:*\n${{ github.actor }}"
                    }
                  ]
                },
                {
                  "type": "actions",
                  "elements": [
                    {
                      "type": "button",
                      "text": {"type": "plain_text", "text": "View Run"},
                      "url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                    }
                  ]
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
```

---

## Complete Pipeline Examples

```yaml
# .github/workflows/full-pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # ============================================================
  # Stage 1: Static Analysis
  # ============================================================
  static-analysis:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check
      - run: npm run type-check
      - run: npm audit --audit-level=moderate

  # ============================================================
  # Stage 2: Tests (parallel)
  # ============================================================
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: static-analysis
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:unit -- --coverage --ci
      - uses: codecov/codecov-action@v3
        with: { flags: unit }

  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: static-analysis
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: --health-cmd "redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      DATABASE_URL: postgresql://postgres:password@localhost:5432/test_db
      REDIS_URL: redis://localhost:6379
      JWT_SECRET: test-secret
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx prisma migrate deploy
      - run: npm run test:integration -- --coverage --ci
      - uses: codecov/codecov-action@v3
        with: { flags: integration }

  # ============================================================
  # Stage 3: Build Image
  # ============================================================
  build-image:
    name: Build Docker Image
    needs: [unit-tests, integration-tests]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    outputs:
      image: ${{ steps.push.outputs.imageid }}
      tags: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha
            type=ref,event=branch
      - id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ============================================================
  # Stage 4: Deploy Staging
  # ============================================================
  deploy-staging:
    name: Deploy Staging
    needs: build-image
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: deploy
          key: ${{ secrets.STAGING_SSH_KEY }}
          script: |
            IMAGE="ghcr.io/${{ github.repository }}:sha-${{ github.sha }}"
            docker pull $IMAGE
            docker run --rm --env-file /etc/myapp/staging.env $IMAGE npx prisma migrate deploy
            docker stop myapp-staging || true && docker rm myapp-staging || true
            docker run -d --name myapp-staging --restart unless-stopped \
              -p 3001:3000 --env-file /etc/myapp/staging.env $IMAGE
            sleep 10 && curl -f http://localhost:3001/health

  # ============================================================
  # Stage 5: Deploy Production (manual approve)
  # ============================================================
  deploy-production:
    name: Deploy Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: deploy
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            IMAGE="ghcr.io/${{ github.repository }}:sha-${{ github.sha }}"
            docker pull $IMAGE
            docker run --rm --env-file /etc/myapp/production.env $IMAGE npx prisma migrate deploy
            docker stop myapp-prod || true && docker rm myapp-prod || true
            docker run -d --name myapp-prod --restart unless-stopped \
              -p 3000:3000 --env-file /etc/myapp/production.env $IMAGE
            sleep 15 && curl -f http://localhost:3000/health
```

---

## Package.json Scripts

```json
{
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc --project tsconfig.build.json",
    "start": "node dist/server.js",
    "lint": "eslint src --ext .ts,.tsx",
    "lint:fix": "eslint src --ext .ts,.tsx --fix",
    "format": "prettier --write 'src/**/*.{ts,tsx,json}'",
    "format:check": "prettier --check 'src/**/*.{ts,tsx,json}'",
    "type-check": "tsc --noEmit",
    "test": "jest",
    "test:unit": "jest --testPathPattern='__tests__/unit'",
    "test:integration": "jest --testPathPattern='__tests__/integration'",
    "test:e2e": "playwright test",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --forceExit --maxWorkers=2",
    "db:migrate": "prisma migrate deploy",
    "db:migrate:dev": "prisma migrate dev",
    "db:seed": "tsx prisma/seed.ts",
    "db:generate": "prisma generate",
    "docker:build": "docker build -t myapp .",
    "docker:run": "docker-compose up -d"
  }
}
```

---

## สรุป CI/CD Best Practices

```
CI Pipeline:
1. Fast feedback: lint/type-check ก่อน test (fail fast)
2. Parallel jobs: unit และ integration tests run พร้อมกัน
3. Cache: npm ci + node_modules cache ประหยัดเวลา
4. Services: ใช้ GitHub Actions services สำหรับ DB/Redis
5. Fail loud: ตั้งค่าให้ fail ถ้า test ไม่ผ่าน

CD Pipeline:
1. Build once, deploy many: build image ครั้งเดียว ใช้ทุก environment
2. Database migrations: run ก่อน deploy app เสมอ
3. Health checks: ตรวจสอบหลัง deploy ก่อน route traffic
4. Blue-Green: zero-downtime deployment
5. Rollback plan: ต้อง rollback ได้ภายใน 5 นาที

Security:
1. ไม่ใส่ secrets ใน workflow files (ใช้ GitHub Secrets)
2. Pin action versions: actions/checkout@v4 ไม่ใช่ @latest
3. Least privilege: GITHUB_TOKEN permissions
4. Scan dependencies: npm audit ใน CI
5. Signed images: cosign สำหรับ production images
```

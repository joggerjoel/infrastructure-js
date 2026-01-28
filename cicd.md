# CI/CD Pipeline Guide

---
**Status**: 💡 **GUIDANCE** - Best practices for CI/CD implementation  
**Priority**: 🔴 **P0.8** - Essential for reliable, repeatable deployments  
**Last Updated**: 2026-01-28  
**Owned By**: Backend Team, DevOps  

---

## Overview

Continuous Integration and Continuous Deployment (CI/CD) automates the process of testing, building, and deploying code. This guide covers GitHub Actions workflows, testing automation, deployment strategies, and best practices for reliable releases.

---

## CI/CD Fundamentals

### What is CI/CD?

**Continuous Integration (CI)**:
- Automatically test code on every commit
- Catch bugs early
- Maintain code quality
- Prevent broken builds

**Continuous Deployment (CD)**:
- Automatically deploy passing builds
- Reduce manual errors
- Fast, frequent releases
- Rollback capability

### Benefits

✅ **Catch bugs before production**: Automated testing on every commit  
✅ **Consistent deployments**: Same process every time  
✅ **Fast releases**: Deploy in minutes, not hours  
✅ **Easy rollbacks**: Revert to previous version instantly  
✅ **Documentation**: Deployment history in git tags  
✅ **Confidence**: Tests must pass before deploy  

---

## GitHub Actions Basics

### Workflow Structure

```yaml
# .github/workflows/ci.yml
name: CI                          # Workflow name

on:                               # Triggers
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:                             # Jobs to run
  test:                           # Job name
    runs-on: ubuntu-latest        # Runner OS
    
    steps:                        # Steps in job
      - uses: actions/checkout@v3 # Action: checkout code
      
      - name: Setup Node.js       # Step name
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci               # Command to run
      
      - name: Run tests
        run: npm test
```

### Key Concepts

| Concept | Description | Example |
|---------|-------------|---------|
| **Workflow** | Automated process | CI, Deploy, Release |
| **Job** | Set of steps | test, build, deploy |
| **Step** | Individual task | Install deps, run tests |
| **Action** | Reusable step | `actions/checkout@v3` |
| **Runner** | Machine running workflow | `ubuntu-latest` |
| **Trigger** | Event starting workflow | push, pull_request |
| **Artifact** | File shared between jobs | Build output, test results |

---

## Complete CI Workflow

### Comprehensive CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'

jobs:
  # Job 1: Lint and format check
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Check formatting
        run: npm run format:check
      
      - name: TypeScript check
        run: npm run type-check

  # Job 2: Run tests
  test:
    name: Test
    runs-on: ubuntu-latest
    
    services:
      # PostgreSQL service
      postgres:
        image: postgres:14
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
      
      # Redis service
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run database migrations
        run: npm run migrate:test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
      
      - name: Run unit tests
        run: npm run test:unit
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
      
      - name: Generate coverage report
        run: npm run test:coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          fail_ci_if_error: true

  # Job 3: Security checks
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Run npm audit
        run: npm audit --audit-level=high
        continue-on-error: true
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
        continue-on-error: true

  # Job 4: Build
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]  # Wait for lint and test to pass
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Generate API documentation
        run: npm run generate:docs
      
      - name: Check API docs are in sync
        run: |
          if [[ -n $(git status --porcelain docs/) ]]; then
            echo "❌ API documentation is out of sync!"
            echo "Run 'npm run generate:docs' and commit the changes"
            exit 1
          fi
      
      - name: Build application
        run: npm run build
        env:
          NODE_ENV: production
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-artifacts
          path: |
            dist/
            package.json
            package-lock.json
          retention-days: 30

  # Job 5: E2E tests (optional, runs in parallel)
  e2e:
    name: E2E Tests
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'  # Only on PRs
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
```

---

## Deployment Workflows

### Deploy to Staging (Automatic)

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches: [develop]

env:
  NODE_VERSION: '18'
  AWS_REGION: us-east-1

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging-api.myapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
        env:
          NODE_ENV: production
          GIT_COMMIT: ${{ github.sha }}
          BUILD_DATE: ${{ github.event.head_commit.timestamp }}
      
      - name: Run database migrations
        run: npm run migrate:latest
        env:
          DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Build Docker image
        run: |
          docker build \
            --build-arg VERSION=${{ github.sha }} \
            --build-arg GIT_COMMIT=${{ github.sha }} \
            --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
            -t myapp:staging \
            .
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Tag and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: staging-${{ github.sha }}
        run: |
          docker tag myapp:staging $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker tag myapp:staging $ECR_REGISTRY/$ECR_REPOSITORY:staging-latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:staging-latest
      
      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster staging-cluster \
            --service myapp-service \
            --force-new-deployment
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster staging-cluster \
            --services myapp-service
      
      - name: Run smoke tests
        run: npm run test:smoke
        env:
          API_URL: https://staging-api.myapp.com
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚀 Deployed to Staging",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Deployment to Staging Successful*\n\nCommit: `${{ github.sha }}`\nAuthor: ${{ github.actor }}\n<https://staging-api.myapp.com|View Staging>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Deploy to Production (Manual Approval)

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    tags:
      - 'v*.*.*'  # Trigger on version tags (v1.2.3)

env:
  NODE_VERSION: '18'
  AWS_REGION: us-east-1

jobs:
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.myapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Get version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
        env:
          NODE_ENV: production
          GIT_COMMIT: ${{ github.sha }}
          BUILD_DATE: ${{ github.event.head_commit.timestamp }}
      
      - name: Create database backup
        run: |
          aws rds create-db-snapshot \
            --db-instance-identifier prod-db \
            --db-snapshot-identifier prod-db-pre-deploy-${{ github.sha }}
      
      - name: Wait for backup
        run: |
          aws rds wait db-snapshot-available \
            --db-snapshot-identifier prod-db-pre-deploy-${{ github.sha }}
      
      - name: Run database migrations
        run: npm run migrate:latest
        env:
          DATABASE_URL: ${{ secrets.PRODUCTION_DATABASE_URL }}
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Build Docker image
        run: |
          docker build \
            --build-arg VERSION=${{ steps.version.outputs.VERSION }} \
            --build-arg GIT_COMMIT=${{ github.sha }} \
            --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
            -t myapp:${{ steps.version.outputs.VERSION }} \
            .
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Tag and push image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: ${{ steps.version.outputs.VERSION }}
        run: |
          docker tag myapp:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker tag myapp:$IMAGE_TAG $ECR_REGISTRY/$ECR_REPOSITORY:latest
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
      
      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster production-cluster \
            --service myapp-service \
            --force-new-deployment
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster production-cluster \
            --services myapp-service
      
      - name: Run smoke tests
        run: npm run test:smoke
        env:
          API_URL: https://api.myapp.com
      
      - name: Record deployment
        run: |
          curl -X POST ${{ secrets.DEPLOYMENT_TRACKER_URL }} \
            -H "Content-Type: application/json" \
            -d '{
              "version": "${{ steps.version.outputs.VERSION }}",
              "environment": "production",
              "commit": "${{ github.sha }}",
              "deployed_by": "${{ github.actor }}",
              "deployed_at": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'"
            }'
      
      - name: Create GitHub Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ steps.version.outputs.VERSION }}
          release_name: Release v${{ steps.version.outputs.VERSION }}
          body: |
            ## Changes in v${{ steps.version.outputs.VERSION }}
            
            See [CHANGELOG.md](CHANGELOG.md) for details.
            
            **Deployed to Production**: $(date -u +"%Y-%m-%d %H:%M:%S UTC")
            **Commit**: ${{ github.sha }}
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚀 Deployed to Production",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment Successful*\n\nVersion: `v${{ steps.version.outputs.VERSION }}`\nCommit: `${{ github.sha }}`\nAuthor: ${{ github.actor }}\n<https://api.myapp.com|View Production>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Notify team
        run: |
          echo "✅ Production deployment complete!"
          echo "Version: v${{ steps.version.outputs.VERSION }}"
          echo "Monitor: https://datadog.com/dashboard/production"
```

---

## Environment Management

### GitHub Environments

```yaml
# Configure environments in GitHub repo settings:
# Settings → Environments → New environment

# staging environment:
# - No protection rules
# - Auto-deploys on push to develop

# production environment:
# - Require manual approval (1 reviewer)
# - Restrict to main branch
# - Wait timer: 5 minutes

# In workflow:
jobs:
  deploy:
    environment:
      name: production        # Environment name
      url: https://api.myapp.com  # Deployment URL
```

### Environment Secrets

```bash
# GitHub Settings → Secrets and variables → Actions

# Repository secrets (all environments):
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
CODECOV_TOKEN
SLACK_WEBHOOK

# Environment-specific secrets:
# staging:
STAGING_DATABASE_URL
STAGING_REDIS_URL

# production:
PRODUCTION_DATABASE_URL
PRODUCTION_REDIS_URL
STRIPE_LIVE_KEY
```

---

## Testing Strategies

### Test Matrix (Multiple Versions)

```yaml
# .github/workflows/test-matrix.yml
name: Test Matrix

on: [push, pull_request]

jobs:
  test:
    name: Test Node ${{ matrix.node-version }}
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        node-version: [16, 18, 20]
        os: [ubuntu-latest, windows-latest, macos-latest]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
```

### Performance Testing

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM
  workflow_dispatch:      # Manual trigger

jobs:
  performance:
    name: Performance Test
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install k6
        run: |
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6
      
      - name: Run load tests
        run: k6 run tests/load/api-load-test.js
      
      - name: Upload results
        uses: actions/upload-artifact@v3
        with:
          name: performance-results
          path: results/
      
      - name: Check thresholds
        run: |
          # Fail if P95 response time > 200ms
          # Fail if error rate > 1%
          npm run check:performance-thresholds
```

---

## Deployment Strategies

### Blue-Green Deployment

```yaml
# .github/workflows/blue-green-deploy.yml
name: Blue-Green Deployment

jobs:
  deploy:
    steps:
      # ... build steps ...
      
      - name: Deploy to green environment
        run: |
          aws ecs update-service \
            --cluster production-cluster \
            --service myapp-green \
            --task-definition myapp:${{ github.sha }}
      
      - name: Wait for green deployment
        run: |
          aws ecs wait services-stable \
            --cluster production-cluster \
            --services myapp-green
      
      - name: Run smoke tests on green
        run: npm run test:smoke
        env:
          API_URL: https://green.api.myapp.com
      
      - name: Switch traffic to green
        run: |
          aws route53 change-resource-record-sets \
            --hosted-zone-id ${{ secrets.ROUTE53_ZONE_ID }} \
            --change-batch file://route53-green.json
      
      - name: Monitor for errors
        run: |
          sleep 300  # Wait 5 minutes
          # Check error rate
          if [[ $(check_error_rate) > 1 ]]; then
            echo "High error rate detected, rolling back"
            exit 1
          fi
      
      - name: Decommission blue environment
        if: success()
        run: |
          aws ecs update-service \
            --cluster production-cluster \
            --service myapp-blue \
            --desired-count 0
```

### Canary Deployment

```yaml
# Deploy to 10% of traffic first, then 50%, then 100%
- name: Canary deployment - 10%
  run: |
    kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
    kubectl patch deployment myapp -p '{"spec":{"replicas":1}}'  # 1 of 10 pods

- name: Monitor canary
  run: sleep 300 && check_metrics

- name: Canary deployment - 50%
  if: success()
  run: kubectl patch deployment myapp -p '{"spec":{"replicas":5}}'

- name: Monitor canary
  run: sleep 300 && check_metrics

- name: Full deployment - 100%
  if: success()
  run: kubectl patch deployment myapp -p '{"spec":{"replicas":10}}'
```

---

## Rollback Procedures

### Automatic Rollback on Failure

```yaml
# .github/workflows/deploy-with-rollback.yml
jobs:
  deploy:
    steps:
      # ... deployment steps ...
      
      - name: Run smoke tests
        id: smoke-test
        run: npm run test:smoke
        continue-on-error: true
      
      - name: Rollback on failure
        if: steps.smoke-test.outcome == 'failure'
        run: |
          echo "🚨 Smoke tests failed, rolling back!"
          
          # Rollback ECS to previous task definition
          PREVIOUS_TASK_DEF=$(aws ecs describe-services \
            --cluster production-cluster \
            --services myapp-service \
            --query 'services[0].deployments[1].taskDefinition' \
            --output text)
          
          aws ecs update-service \
            --cluster production-cluster \
            --service myapp-service \
            --task-definition $PREVIOUS_TASK_DEF \
            --force-new-deployment
          
          exit 1
      
      - name: Notify on rollback
        if: steps.smoke-test.outcome == 'failure'
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text":"🚨 Deployment failed, rolled back to previous version"}'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Manual Rollback Workflow

```yaml
# .github/workflows/rollback.yml
name: Rollback Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to rollback to (e.g., 1.2.3)'
        required: true
      confirm:
        description: 'Type "ROLLBACK" to confirm'
        required: true

jobs:
  rollback:
    name: Rollback to v${{ github.event.inputs.version }}
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Validate confirmation
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "ROLLBACK" ]; then
            echo "❌ Confirmation failed. Type 'ROLLBACK' to confirm."
            exit 1
          fi
      
      - name: Rollback deployment
        run: |
          aws ecs update-service \
            --cluster production-cluster \
            --service myapp-service \
            --task-definition myapp:${{ github.event.inputs.version }} \
            --force-new-deployment
      
      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "⏪ Production Rollback",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Rolled back to v${{ github.event.inputs.version }}*\n\nTriggered by: ${{ github.actor }}"
                }
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Caching Strategies

### Dependency Caching

```yaml
jobs:
  build:
    steps:
      - uses: actions/checkout@v3
      
      # NPM cache
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'  # Automatically caches node_modules
      
      # Or manual caching
      - name: Cache dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.npm
            node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      # Docker layer caching
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build with cache
        uses: docker/build-push-action@v4
        with:
          context: .
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Monitoring & Notifications

### Slack Notifications

```yaml
jobs:
  notify:
    steps:
      - name: Notify on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Build Successful",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Build ${{ github.run_number }} passed*\n\nBranch: `${{ github.ref }}`\nCommit: `${{ github.sha }}`\nAuthor: ${{ github.actor }}"
                }
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "❌ Build Failed",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "*Build ${{ github.run_number }} failed*\n\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Logs>"
                }
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Email Notifications

```yaml
- name: Send email on failure
  if: failure()
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.gmail.com
    server_port: 465
    username: ${{ secrets.EMAIL_USERNAME }}
    password: ${{ secrets.EMAIL_PASSWORD }}
    subject: "❌ Build ${{ github.run_number }} Failed"
    to: team@myapp.com
    from: ci@myapp.com
    body: |
      Build failed for commit ${{ github.sha }}
      
      View logs: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

---

## Security Best Practices

### Secrets Management

```yaml
# ❌ DON'T: Hardcode secrets
- name: Deploy
  run: |
    API_KEY="sk_live_1234567890"  # NEVER DO THIS
    deploy.sh

# ✅ DO: Use GitHub secrets
- name: Deploy
  run: deploy.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}

# ✅ DO: Use environment protection
environment:
  name: production  # Requires approval
```

### Least Privilege Access

```yaml
# Limit permissions
permissions:
  contents: read
  packages: write
  deployments: write

# Or specific permissions
permissions:
  contents: read      # Read repo
  pull-requests: write  # Comment on PRs
  statuses: write     # Update commit status
```

### Code Scanning

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  codeql:
    name: CodeQL Analysis
    runs-on: ubuntu-latest
    
    permissions:
      security-events: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript, typescript
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2
```

---

## CI/CD Checklist

### Pre-Deployment Checklist

- [ ] All tests pass (unit, integration, e2e)
- [ ] Code coverage meets threshold (>80%)
- [ ] Linting passes
- [ ] Type checking passes
- [ ] Security scan passes (no high vulnerabilities)
- [ ] API documentation is in sync
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] Secrets properly stored
- [ ] Build artifacts generated
- [ ] Smoke tests written

### Deployment Checklist

- [ ] Version bumped appropriately
- [ ] CHANGELOG updated
- [ ] Database backup created (production)
- [ ] Migrations run successfully
- [ ] Docker image built and tagged
- [ ] Image pushed to registry
- [ ] Service updated with new image
- [ ] Health checks passing
- [ ] Smoke tests passing
- [ ] Monitoring dashboards checked
- [ ] Error rates normal
- [ ] Team notified

### Post-Deployment Checklist

- [ ] Deployment recorded in tracking system
- [ ] GitHub release created
- [ ] Slack notification sent
- [ ] Documentation updated
- [ ] Monitoring for errors (15 minutes)
- [ ] Rollback plan ready
- [ ] Previous version tagged for rollback

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **Tests timeout** | Database not ready | Add health checks to services |
| **Build fails randomly** | Flaky tests | Identify and fix non-deterministic tests |
| **Secrets not available** | Wrong environment | Check environment configuration |
| **Docker build fails** | Cache issue | Clear cache, rebuild |
| **Deploy succeeds but app crashes** | Environment mismatch | Check environment variables |

### Debug Workflow

```yaml
# Enable debug logging
- name: Debug
  run: |
    echo "GitHub context:"
    echo "${{ toJSON(github) }}"
    
    echo "Environment variables:"
    env | sort
    
    echo "Current directory:"
    pwd
    ls -la

# Step debugging
- name: Setup tmate session
  uses: mxschmitt/action-tmate@v3
  if: failure()  # Only on failure
```

---

## Best Practices Summary

### DO ✅

1. **Test before deploy**: Always run tests in CI
2. **Use environments**: Separate staging and production
3. **Require reviews**: Manual approval for production
4. **Cache dependencies**: Speed up builds
5. **Run security scans**: Catch vulnerabilities early
6. **Monitor deployments**: Watch for errors
7. **Have rollback plan**: Quick recovery from failures
8. **Notify team**: Slack/email on deploy
9. **Version everything**: Tag releases
10. **Document workflows**: Comment complex steps

### DON'T ❌

1. **Don't skip tests**: Never deploy without testing
2. **Don't hardcode secrets**: Always use GitHub secrets
3. **Don't deploy on Friday**: Risky timing
4. **Don't ignore failures**: Investigate and fix
5. **Don't have manual steps**: Automate everything
6. **Don't deploy without backup**: Always backup production DB
7. **Don't ignore security scans**: Fix vulnerabilities
8. **Don't deploy without monitoring**: Watch for errors
9. **Don't forget rollback**: Have quick recovery plan
10. **Don't make workflows too complex**: Keep it simple

---

## Quick Start Templates

### Minimal CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run build
```

### Minimal Deploy Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run build
      - run: npm run deploy
        env:
          API_KEY: ${{ secrets.API_KEY }}
```

---

## Summary

### Key Principles

1. **Automate Everything**: No manual deployment steps
2. **Test First**: Never deploy untested code
3. **Fast Feedback**: CI runs in minutes, not hours
4. **Safe Deploys**: Rollback capability always available
5. **Visibility**: Team knows what's deployed when

### Time Investment

- **Initial setup**: 4-8 hours
- **Per new workflow**: 1-2 hours
- **Maintenance**: Minimal (automated)
- **ROI**: Massive (prevents production bugs, speeds up releases)

### Related Documentation
- [Deployment Runbook](./DEPLOYMENT.md)
- [Environment Configuration](./ENVIRONMENT.md)
- [Versioning Guide](./VERSIONING.md)
- [Monitoring](./MONITORING.md)

---

**Last Updated**: 2026-01-27  
**Owner**: DevOps Team  
**Review**: Quarterly or when deployment process changes
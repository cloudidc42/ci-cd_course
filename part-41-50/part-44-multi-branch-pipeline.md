# Part 44: Multi-Branch Pipeline Strategy

## สารบัญ
1. [ความสำคัญของ Branch Strategy](#ความสำคัญของ-branch-strategy)
2. [Branch Naming Conventions](#branch-naming-conventions)
3. [Pipeline Per Branch Type](#pipeline-per-branch-type)
4. [Feature Branch Pipeline](#feature-branch-pipeline)
5. [Develop Branch Pipeline](#develop-branch-pipeline)
6. [Main Branch Pipeline](#main-branch-pipeline)
7. [Environment Promotion](#environment-promotion)
8. [Branch Policies](#branch-policies)
9. [Exercises](#exercises)

---

## ความสำคัญของ Branch Strategy

Branch strategy กำหนดว่าทีม development ทำงานอย่างไรกับ code repository และ pipeline จะต้องสอดคล้องกับ strategy นั้น

### Branching Models ยอดนิยม

#### 1. GitHub Flow (Simple)
```
main ──────────────────────────────────────────────────────>
       \                         /
        feature/xxx ────────────
```
- ง่าย เหมาะกับ small teams
- Deploy ตรงจาก main

#### 2. Git Flow (Traditional)
```
main ─────────────────────────────────────────────────────>
  \                     /
   release/1.0 ────────
                 \
develop ─────────────────────────────────────────────────>
  \              /  \              /
   feature/a ──      feature/b ──
```
- ซับซ้อนกว่า เหมาะกับ scheduled releases
- มี develop, feature, release, hotfix branches

#### 3. Trunk-Based Development
```
main (trunk) ──────────────────────────────────────────>
  \     /  \   /  \     /
   A ──     B─     C ──    (short-lived branches < 1 day)
```
- Modern approach
- Feature flags สำหรับ incomplete features
- เหมาะกับ continuous deployment

### แนวทางในคอร์สนี้: GitHub Flow + Environment Branches

```
main ──────────────────────────────────────────────────>
  │                                                      
  │  (auto promote)                                      
  ▼                                                      
staging ──────────────────────────────────────────────>
  │                                                      
  │  (manual promote)                                    
  ▼                                                      
production ───────────────────────────────────────────>

Feature branches:
feature/* ──> PR to main
hotfix/*  ──> PR to main (fast-track)
```

---

## Branch Naming Conventions

### Standard Conventions

```bash
# Feature branches
feature/JIRA-123-add-payment
feature/user-authentication
feature/improve-search-performance

# Bug fix
fix/JIRA-456-login-error
bugfix/null-pointer-exception
hotfix/critical-security-patch

# Release
release/1.2.0
release/2024-Q1

# Chore/Maintenance
chore/update-dependencies
chore/refactor-auth-module

# Documentation
docs/api-documentation
docs/deployment-guide

# Environment branches
develop      # development environment
staging      # staging environment
main         # production environment
```

### Branch Naming Enforcement

```yaml
# .github/workflows/branch-naming.yaml
name: Branch Name Check

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  check-branch-name:
    runs-on: ubuntu-latest
    steps:
    - name: Check branch name
      run: |
        BRANCH_NAME="${{ github.head_ref }}"
        PATTERN="^(feature|fix|hotfix|release|chore|docs|test|refactor)\/[a-z0-9-]+$"
        
        if [[ ! "$BRANCH_NAME" =~ $PATTERN ]]; then
          echo "❌ Branch name '$BRANCH_NAME' doesn't follow naming convention!"
          echo "Expected pattern: $PATTERN"
          echo "Examples:"
          echo "  - feature/add-payment"
          echo "  - fix/login-error"
          echo "  - hotfix/security-patch"
          exit 1
        fi
        
        echo "✅ Branch name '$BRANCH_NAME' is valid"
```

---

## Pipeline Per Branch Type

### Matrix ของ Pipeline behaviors

```
Branch Type    | Test | Build | Deploy Dev | Deploy Staging | Deploy Prod
---------------|------|-------|------------|----------------|------------
feature/*      |  ✅  |  ✅   |    🔄 PR   |      ❌        |     ❌
develop        |  ✅  |  ✅   |    ✅      |      ❌        |     ❌
main           |  ✅  |  ✅   |    ✅      |    ✅ auto     |  ✅ manual
release/*      |  ✅  |  ✅   |    ❌      |    ✅ auto     |  ✅ manual
hotfix/*       |  ✅  |  ✅   |    ✅      |    ✅ auto     |  ✅ fast-track
```

---

## Feature Branch Pipeline

```yaml
# .github/workflows/feature-branch.yaml
name: Feature Branch CI

on:
  push:
    branches:
    - 'feature/**'
    - 'fix/**'
    - 'chore/**'
  pull_request:
    branches:
    - main
    - develop

env:
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================================
  # JOB 1: Code Quality
  # ============================================================
  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Lint
      run: npm run lint
    
    - name: Type check
      run: npm run type-check
    
    - name: Format check
      run: npm run format:check

  # ============================================================
  # JOB 2: Unit Tests
  # ============================================================
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: code-quality
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm run test:unit -- --coverage
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        flags: unit-tests
        fail_ci_if_error: false

  # ============================================================
  # JOB 3: Integration Tests
  # ============================================================
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: code-quality
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
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
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run database migrations
      env:
        DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
      run: npm run db:migrate
    
    - name: Run integration tests
      env:
        DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
        REDIS_URL: redis://localhost:6379
        NODE_ENV: test
      run: npm run test:integration

  # ============================================================
  # JOB 4: Build Docker Image (only on PR to main)
  # ============================================================
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    if: github.event_name == 'pull_request'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.DOCKER_REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=pr
          type=sha,prefix=pr-${{ github.event.number }}-
    
    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # ============================================================
  # JOB 5: Deploy Preview (ใน PR)
  # ============================================================
  deploy-preview:
    name: Deploy Preview Environment
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'pull_request'
    environment:
      name: preview-pr-${{ github.event.number }}
      url: ${{ steps.deploy.outputs.preview-url }}
    
    steps:
    - name: Deploy to preview
      id: deploy
      run: |
        # สมมติใช้ Kubernetes namespace ต่อ PR
        NAMESPACE="preview-pr-${{ github.event.number }}"
        
        kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
        
        helm upgrade --install webapp ./helm/webapp \
          --namespace $NAMESPACE \
          --set image.tag=pr-${{ github.event.number }}-${{ github.sha }} \
          --set ingress.host=pr-${{ github.event.number }}.preview.mycompany.com
        
        echo "preview-url=https://pr-${{ github.event.number }}.preview.mycompany.com" >> $GITHUB_OUTPUT
    
    - name: Comment PR with preview URL
      uses: actions/github-script@v7
      with:
        script: |
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `🚀 Preview deployed at: ${{ steps.deploy.outputs.preview-url }}`
          })
```

---

## Develop Branch Pipeline

```yaml
# .github/workflows/develop.yaml
name: Develop Branch Pipeline

on:
  push:
    branches:
    - develop

env:
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================================
  # JOB 1: Full Test Suite
  # ============================================================
  full-tests:
    name: Full Test Suite
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Lint
      run: npm run lint
    
    - name: Unit tests
      run: npm run test:unit
    
    - name: Integration tests
      run: npm run test:integration

  # ============================================================
  # JOB 2: Build and Push
  # ============================================================
  build-and-push:
    name: Build and Push Image
    runs-on: ubuntu-latest
    needs: full-tests
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.DOCKER_REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=sha,format=short,prefix=develop-
          type=raw,value=develop-latest
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # ============================================================
  # JOB 3: Deploy to Development
  # ============================================================
  deploy-development:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: build-and-push
    environment:
      name: development
      url: https://dev.mycompany.com
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Kustomize
      uses: imranismail/setup-kustomize@v2
    
    - name: Update image tag
      run: |
        cd k8s/overlays/development
        kustomize edit set image \
          webapp=${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:develop-${{ github.sha }}
    
    - name: Apply to cluster
      env:
        KUBECONFIG: ${{ secrets.DEV_KUBECONFIG }}
      run: |
        kubectl apply -k k8s/overlays/development
        kubectl rollout status deployment/webapp -n development --timeout=5m

  # ============================================================
  # JOB 4: Smoke Tests
  # ============================================================
  smoke-tests:
    name: Smoke Tests
    runs-on: ubuntu-latest
    needs: deploy-development
    steps:
    - uses: actions/checkout@v4
    
    - name: Wait for deployment
      run: sleep 30
    
    - name: Run smoke tests
      run: |
        # Basic health check
        curl -f https://dev.mycompany.com/health || exit 1
        
        # API endpoint check
        curl -f https://dev.mycompany.com/api/v1/status || exit 1
        
        echo "✅ Smoke tests passed!"
    
    - name: Notify success
      if: success()
      uses: 8398a7/action-slack@v3
      with:
        status: success
        text: "✅ Development deployment successful: develop@${{ github.sha }}"
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
    
    - name: Notify failure
      if: failure()
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: "❌ Development deployment failed: develop@${{ github.sha }}"
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Main Branch Pipeline

```yaml
# .github/workflows/main.yaml
name: Main Branch Pipeline

on:
  push:
    branches:
    - main
  workflow_dispatch:
    inputs:
      deploy_to_production:
        description: 'Deploy to production?'
        required: true
        default: 'false'
        type: boolean

env:
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================================
  # JOB 1: Full Test Suite + Security Scan
  # ============================================================
  test-and-scan:
    name: Test and Security Scan
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # ต้องการ full history สำหรับ SonarQube
    
    - name: Run all tests
      run: |
        npm ci
        npm run test:all -- --coverage
    
    - name: SonarQube analysis
      uses: SonarSource/sonarqube-scan-action@master
      env:
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
    
    - name: Trivy vulnerability scan
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'

  # ============================================================
  # JOB 2: Build Production Image
  # ============================================================
  build-production:
    name: Build Production Image
    runs-on: ubuntu-latest
    needs: test-and-scan
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      full-image: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.DOCKER_REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,format=long,prefix=
          type=raw,value=latest
          # Auto-tag ถ้า commit มี version tag
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
    
    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        build-args: |
          BUILD_DATE=${{ github.event.head_commit.timestamp }}
          VCS_REF=${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        provenance: true   # SLSA provenance
        sbom: true         # Software Bill of Materials
    
    - name: Scan production image
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
        severity: 'CRITICAL,HIGH'
        exit-code: '1'

  # ============================================================
  # JOB 3: Deploy to Staging (Auto)
  # ============================================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-production
    environment:
      name: staging
      url: https://staging.mycompany.com
    
    steps:
    - uses: actions/checkout@v4
      with:
        repository: myorg/config-repo
        token: ${{ secrets.CONFIG_REPO_PAT }}
    
    - name: Update staging image
      run: |
        cd overlays/staging
        kustomize edit set image \
          webapp=${{ needs.build-production.outputs.full-image }}
    
    - name: Create PR or Auto-commit
      run: |
        git config --global user.email "ci@mycompany.com"
        git config --global user.name "CI Bot"
        git add .
        git commit -m "auto: deploy ${{ github.sha }} to staging"
        git push

  # ============================================================
  # JOB 4: E2E Tests on Staging
  # ============================================================
  e2e-staging:
    name: E2E Tests on Staging
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
    - uses: actions/checkout@v4
    
    - name: Wait for deployment
      run: |
        for i in {1..30}; do
          if curl -f https://staging.mycompany.com/health; then
            echo "Deployment ready!"
            break
          fi
          echo "Waiting... attempt $i"
          sleep 10
        done
    
    - name: Run E2E tests
      run: |
        npm ci
        npm run test:e2e -- --config=staging
      env:
        BASE_URL: https://staging.mycompany.com
        E2E_USER: ${{ secrets.E2E_USER }}
        E2E_PASS: ${{ secrets.E2E_PASS }}

  # ============================================================
  # JOB 5: Deploy to Production (Manual Approval)
  # ============================================================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: e2e-staging
    environment:
      name: production
      url: https://mycompany.com
    # GitHub Environments รองรับ required_reviewers
    # ต้องตั้งค่าใน GitHub Settings > Environments
    
    steps:
    - uses: actions/checkout@v4
      with:
        repository: myorg/config-repo
        token: ${{ secrets.CONFIG_REPO_PAT }}
    
    - name: Update production image
      run: |
        cd overlays/production
        kustomize edit set image \
          webapp=${{ needs.build-production.outputs.full-image }}
    
    - name: Commit and push
      run: |
        git config --global user.email "ci@mycompany.com"
        git config --global user.name "CI Bot"
        git add .
        git commit -m "deploy: ${{ github.sha }} to production [skip ci]"
        git push
    
    - name: Notify deployment
      uses: 8398a7/action-slack@v3
      with:
        status: success
        text: |
          🚀 Production deployment completed!
          Image: ${{ needs.build-production.outputs.full-image }}
          Deployed by: ${{ github.actor }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Environment Promotion

### Promotion Strategy

```yaml
# .github/workflows/promote.yaml
name: Environment Promotion

on:
  workflow_dispatch:
    inputs:
      source_environment:
        description: 'Source environment'
        required: true
        type: choice
        options:
        - staging
        - production
      target_environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
        - staging
        - production
      image_tag:
        description: 'Image tag to promote (leave empty for current)'
        required: false

jobs:
  promote:
    name: Promote ${{ github.event.inputs.source_environment }} to ${{ github.event.inputs.target_environment }}
    runs-on: ubuntu-latest
    environment:
      name: ${{ github.event.inputs.target_environment }}
    
    steps:
    - name: Checkout config repo
      uses: actions/checkout@v4
      with:
        repository: myorg/config-repo
        token: ${{ secrets.CONFIG_REPO_PAT }}
    
    - name: Get current image from source
      id: source-image
      run: |
        if [ -n "${{ github.event.inputs.image_tag }}" ]; then
          echo "IMAGE_TAG=${{ github.event.inputs.image_tag }}" >> $GITHUB_OUTPUT
        else
          # ดึง current image จาก source environment
          IMAGE=$(cat overlays/${{ github.event.inputs.source_environment }}/kustomization.yaml \
            | grep 'newTag:' | awk '{print $2}')
          echo "IMAGE_TAG=$IMAGE" >> $GITHUB_OUTPUT
        fi
    
    - name: Update target environment
      run: |
        cd overlays/${{ github.event.inputs.target_environment }}
        kustomize edit set image \
          webapp=ghcr.io/myorg/webapp:${{ steps.source-image.outputs.IMAGE_TAG }}
    
    - name: Commit and push
      run: |
        git config --global user.email "ci@mycompany.com"
        git config --global user.name "CI Bot"
        git add .
        git commit -m "promote: ${{ steps.source-image.outputs.IMAGE_TAG }} from ${{ github.event.inputs.source_environment }} to ${{ github.event.inputs.target_environment }}"
        git push
    
    - name: Create promotion record
      run: |
        echo "Promotion record:"
        echo "  Source: ${{ github.event.inputs.source_environment }}"
        echo "  Target: ${{ github.event.inputs.target_environment }}"
        echo "  Image: ghcr.io/myorg/webapp:${{ steps.source-image.outputs.IMAGE_TAG }}"
        echo "  Promoted by: ${{ github.actor }}"
        echo "  Time: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### Automated Promotion Pipeline

```yaml
# Environment promotion chain
name: Promotion Chain

on:
  workflow_run:
    workflows: ["E2E Tests - Staging"]
    types: [completed]
    branches: [main]

jobs:
  check-staging-tests:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
    - name: Trigger production promotion
      uses: actions/github-script@v7
      with:
        script: |
          await github.rest.actions.createWorkflowDispatch({
            owner: context.repo.owner,
            repo: context.repo.repo,
            workflow_id: 'promote.yaml',
            ref: 'main',
            inputs: {
              source_environment: 'staging',
              target_environment: 'production'
            }
          })
```

---

## Branch Policies

### GitHub Branch Protection Rules

```yaml
# .github/branch-protection.yaml
# ใช้กับ GitHub API หรือ Terraform

branch_protection:
  main:
    required_status_checks:
      strict: true  # ต้อง up-to-date กับ base branch
      contexts:
      - "code-quality"
      - "unit-tests"
      - "integration-tests"
      - "build"
      - "security-scan"
    
    required_pull_request_reviews:
      required_approving_review_count: 2
      dismiss_stale_reviews: true
      require_code_owner_reviews: true
      require_last_push_approval: true
    
    restrictions:
      users: []
      teams: ["platform-team"]  # เฉพาะ platform-team merge ได้
    
    enforce_admins: true
    required_linear_history: true  # ไม่อนุญาต merge commits
    allow_force_pushes: false
    allow_deletions: false
    
  develop:
    required_status_checks:
      strict: false
      contexts:
      - "code-quality"
      - "unit-tests"
    
    required_pull_request_reviews:
      required_approving_review_count: 1
      dismiss_stale_reviews: true
    
    allow_force_pushes: false
```

### Terraform Branch Protection

```hcl
# terraform/github-branch-protection.tf
resource "github_branch_protection" "main" {
  repository_id = github_repository.app.node_id
  pattern       = "main"
  
  required_status_checks {
    strict = true
    contexts = [
      "code-quality",
      "unit-tests",
      "integration-tests",
      "security-scan"
    ]
  }
  
  required_pull_request_reviews {
    required_approving_review_count = 2
    dismiss_stale_reviews           = true
    require_code_owner_reviews      = true
  }
  
  restrict_pushes {
    push_allowances = [
      "myorg/platform-team"
    ]
  }
  
  enforce_admins         = true
  require_linear_history = true
  allows_force_pushes    = false
  allows_deletions       = false
}

resource "github_branch_protection" "develop" {
  repository_id = github_repository.app.node_id
  pattern       = "develop"
  
  required_status_checks {
    strict   = false
    contexts = ["code-quality", "unit-tests"]
  }
  
  required_pull_request_reviews {
    required_approving_review_count = 1
    dismiss_stale_reviews           = true
  }
  
  allows_force_pushes = false
  allows_deletions    = false
}
```

### CODEOWNERS

```
# .github/CODEOWNERS

# Default owners สำหรับทุกไฟล์
*                           @myorg/platform-team

# Frontend code
/src/frontend/**            @myorg/frontend-team @myorg/platform-team

# Backend code
/src/backend/**             @myorg/backend-team @myorg/platform-team

# Infrastructure
/k8s/**                     @myorg/platform-team
/terraform/**               @myorg/platform-team

# CI/CD workflows
/.github/workflows/**       @myorg/platform-team

# Security-sensitive files
/src/auth/**                @myorg/security-team @myorg/platform-team
/src/payments/**            @myorg/security-team @myorg/payment-team
```

### PR Template

```markdown
<!-- .github/pull_request_template.md -->
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix (non-breaking change)
- [ ] New feature (non-breaking change)
- [ ] Breaking change
- [ ] Documentation update
- [ ] Performance improvement
- [ ] Refactoring

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings
- [ ] Tests pass locally

## Screenshots (if applicable)

## Related Issues
Closes #ISSUE_NUMBER

## Deployment Notes
Any special deployment considerations
```

---

## Exercises

### Exercise 1: Setup Branch Strategy

```bash
#!/bin/bash
# exercise-1-setup-branches.sh

# สร้าง branches ตาม strategy
git checkout -b develop
git push origin develop

git checkout -b staging
git push origin staging

# กลับไป main
git checkout main

# สร้าง feature branch
git checkout -b feature/add-user-auth
echo "New auth code" > src/auth.js
git add .
git commit -m "feat: add user authentication"
git push origin feature/add-user-auth

# สร้าง PR
gh pr create \
  --title "Add user authentication" \
  --body "Implements JWT-based authentication" \
  --base main \
  --head feature/add-user-auth
```

### Exercise 2: Configure Pipeline per Branch

```yaml
# .github/workflows/ci-cd.yaml
name: CI/CD Pipeline

on:
  push:
    branches: ['**']
  pull_request:
    branches: [main, develop]

jobs:
  determine-pipeline:
    runs-on: ubuntu-latest
    outputs:
      run-tests: ${{ steps.check.outputs.run-tests }}
      run-build: ${{ steps.check.outputs.run-build }}
      deploy-env: ${{ steps.check.outputs.deploy-env }}
    steps:
    - id: check
      run: |
        BRANCH="${{ github.ref_name }}"
        EVENT="${{ github.event_name }}"
        
        # Default values
        echo "run-tests=true" >> $GITHUB_OUTPUT
        echo "run-build=false" >> $GITHUB_OUTPUT
        echo "deploy-env=none" >> $GITHUB_OUTPUT
        
        # Feature/fix branches: test only
        if [[ "$BRANCH" == feature/* ]] || [[ "$BRANCH" == fix/* ]]; then
          echo "run-build=false" >> $GITHUB_OUTPUT
          echo "deploy-env=none" >> $GITHUB_OUTPUT
        fi
        
        # Develop branch: build + deploy dev
        if [[ "$BRANCH" == "develop" ]]; then
          echo "run-build=true" >> $GITHUB_OUTPUT
          echo "deploy-env=development" >> $GITHUB_OUTPUT
        fi
        
        # Main branch: build + deploy staging + prod
        if [[ "$BRANCH" == "main" ]]; then
          echo "run-build=true" >> $GITHUB_OUTPUT
          echo "deploy-env=staging" >> $GITHUB_OUTPUT
        fi

  tests:
    needs: determine-pipeline
    if: needs.determine-pipeline.outputs.run-tests == 'true'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: echo "Running tests..."

  build:
    needs: [determine-pipeline, tests]
    if: needs.determine-pipeline.outputs.run-build == 'true'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: echo "Building image..."

  deploy:
    needs: [determine-pipeline, build]
    if: needs.determine-pipeline.outputs.deploy-env != 'none'
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-pipeline.outputs.deploy-env }}
    steps:
    - run: echo "Deploying to ${{ needs.determine-pipeline.outputs.deploy-env }}"
```

### Exercise 3: Cleanup PR Preview Environments

```yaml
# .github/workflows/cleanup-preview.yaml
name: Cleanup Preview Environment

on:
  pull_request:
    types: [closed]

jobs:
  cleanup:
    runs-on: ubuntu-latest
    steps:
    - name: Delete preview namespace
      run: |
        NAMESPACE="preview-pr-${{ github.event.number }}"
        kubectl delete namespace $NAMESPACE --ignore-not-found
        echo "Cleaned up namespace: $NAMESPACE"
    
    - name: Remove GitHub Environment
      uses: actions/github-script@v7
      with:
        script: |
          try {
            await github.rest.repos.deleteAnEnvironment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              environment_name: `preview-pr-${context.issue.number}`
            })
            console.log('Environment deleted')
          } catch (error) {
            console.log('Environment not found or already deleted')
          }
```

---

## สรุป

Multi-Branch Pipeline Strategy ที่ดีต้องมี:

1. **ชัดเจนใน branch naming** - ทุกคนเข้าใจตรงกัน
2. **Pipeline behavior ต่างกันต่อ branch type** - feature vs main ต่างกัน
3. **Environment isolation** - dev, staging, production แยกกัน
4. **Automated promotion** - ลด manual work
5. **Branch protection** - ป้องกันการ push ตรงไปยัง main
6. **CODEOWNERS** - กำหนดผู้รับผิดชอบ code review

---

*ส่วนต่อไป: Part 45 - Monorepo CI/CD*

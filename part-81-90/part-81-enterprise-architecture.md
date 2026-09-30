# Part 81: Enterprise CI/CD Architecture

## บทนำ

ในองค์กรขนาดใหญ่ การออกแบบ CI/CD Architecture ไม่ใช่เรื่องที่ทำครั้งเดียวแล้วจบ แต่เป็นกระบวนการต่อเนื่องที่ต้องพัฒนาไปพร้อมกับการเติบโตขององค์กร บทนี้จะครอบคลุมแนวคิด รูปแบบ และวิธีปฏิบัติที่ใช้จริงในองค์กรระดับ Enterprise

## สารบัญ

1. [ความเข้าใจ Enterprise CI/CD](#enterprise-cicd)
2. [Enterprise Patterns สำหรับ CI/CD](#enterprise-patterns)
3. [Multi-Team Coordination](#multi-team-coordination)
4. [Golden Path Architecture](#golden-path)
5. [Shared Infrastructure](#shared-infrastructure)
6. [Governance Model](#governance-model)
7. [Scaling Challenges และ Solutions](#scaling)
8. [Case Studies](#case-studies)
9. [แบบฝึกหัด](#exercises)

---

## 1. ความเข้าใจ Enterprise CI/CD {#enterprise-cicd}

### Enterprise CI/CD คืออะไร?

Enterprise CI/CD แตกต่างจาก CI/CD ทั่วไปในหลายด้าน:

| ด้าน | Small Team | Enterprise |
|------|-----------|------------|
| จำนวน Developers | < 50 คน | 500+ คน |
| จำนวน Repositories | < 50 | 500-5000+ |
| Pipeline runs/day | < 1,000 | 10,000-1,000,000+ |
| Teams | 1-5 | 50-500+ |
| Compliance Requirements | ต่ำ | สูงมาก |
| Cost Sensitivity | ปานกลาง | สูงมาก |

### ความท้าทายหลักของ Enterprise

**1. Scale ของ Infrastructure**
```
ตัวอย่าง: บริษัทที่มี Developer 2,000 คน
- Git commits/day: ~5,000-10,000
- Pipeline triggers/day: ~20,000-50,000
- Build minutes/day: ~200,000+
- Artifact storage: หลาย TB/เดือน
- Cost: $50,000-500,000+/เดือน
```

**2. Organizational Complexity**
- หลาย Business Units
- หลาย Technology Stacks
- หลาย Deployment Environments
- หลาย Teams with different maturity levels

**3. Security และ Compliance**
- SOX, PCI-DSS, HIPAA, GDPR
- Code signing requirements
- Audit trail requirements
- Separation of Duties

### ทำไม Enterprise ถึงล้มเหลว

สาเหตุที่พบบ่อยที่สุดของการล้มเหลว CI/CD ใน Enterprise:

```
1. Tooling Fragmentation (45%)
   - แต่ละ team ใช้ tools ต่างกัน
   - ไม่มี standard
   - Integration ยาก

2. Lack of Platform Team (30%)
   - ทุก team ต้อง manage CI/CD เอง
   - ไม่มี shared knowledge
   - Duplicated effort

3. Poor Governance (15%)
   - ไม่มี policy enforcement
   - Security gaps
   - Compliance issues

4. Technical Debt (10%)
   - Legacy systems
   - Slow migration
```

---

## 2. Enterprise Patterns สำหรับ CI/CD {#enterprise-patterns}

### Pattern 1: Hub and Spoke Model

```
                    ┌─────────────────┐
                    │  Central CI/CD  │
                    │    Platform     │
                    │   (Hub)         │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
     ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
     │  Team A     │  │  Team B     │  │  Team C     │
     │  (Spoke)    │  │  (Spoke)    │  │  (Spoke)    │
     └─────────────┘  └─────────────┘  └─────────────┘
```

**ข้อดี:**
- Centralized control
- Standardization ง่าย
- Cost efficient

**ข้อเสีย:**
- Single point of failure
- Bottleneck
- Autonomy ต่ำ

### Pattern 2: Federated Model

```
┌─────────────────────────────────────────────────────┐
│                   Platform Layer                     │
│    (Common tools, standards, governance)             │
└──────────────┬───────────────────────────────────────┘
               │ Provides standards & tools
    ┌──────────▼──────────┐  ┌──────────────────────┐
    │   Domain A          │  │   Domain B           │
    │   Own CI/CD         │  │   Own CI/CD          │
    │   Team              │  │   Team               │
    └──────────┬──────────┘  └──────────┬───────────┘
               │                        │
    ┌──────────▼──────────┐  ┌──────────▼───────────┐
    │  App Teams          │  │  App Teams           │
    └─────────────────────┘  └──────────────────────┘
```

**ข้อดี:**
- Autonomy สูง
- Scale ได้ดี
- Domain ownership ชัดเจน

**ข้อเสีย:**
- ต้องการ Platform Team ที่แข็งแกร่ง
- Complex governance

### Pattern 3: Microservice CI/CD Architecture

สำหรับองค์กรที่ใช้ Microservices:

```yaml
# ตัวอย่าง: Architecture สำหรับ 100+ Microservices

Shared Infrastructure:
  - Container Registry: shared, per-domain namespaces
  - Secret Management: Vault, per-team policies
  - Artifact Storage: Nexus/Artifactory, per-team paths
  - Pipeline Execution: Kubernetes-based runners

Per-Service Pipelines:
  - Independent build/test/deploy
  - Shared pipeline templates
  - Service mesh integration

Cross-Service Coordination:
  - Contract testing (Pact)
  - Integration test environments
  - Release coordination tools
```

### Pattern 4: Monorepo Enterprise Architecture

```
/monorepo
├── .github/
│   ├── workflows/
│   │   ├── _templates/           # Reusable workflows
│   │   ├── ci-backend.yml
│   │   ├── ci-frontend.yml
│   │   └── cd-production.yml
├── packages/
│   ├── shared/                   # Shared libraries
│   ├── team-a/
│   │   ├── service-1/
│   │   └── service-2/
│   └── team-b/
│       ├── app-1/
│       └── app-2/
├── tools/
│   ├── nx.json                   # Build orchestration
│   └── turbo.json
└── CODEOWNERS
```

**Affected-based Builds (Nx/Turborepo):**
```bash
# Build เฉพาะ services ที่มีการเปลี่ยนแปลง
nx affected:build --base=main --head=HEAD

# Test เฉพาะ services ที่ได้รับผลกระทบ
nx affected:test --base=main --head=HEAD

# ประหยัดเวลา: จาก 45 นาที เหลือ 8 นาที
```

---

## 3. Multi-Team Coordination {#multi-team-coordination}

### ปัญหาของ Multi-Team CI/CD

เมื่อมีหลาย teams ทำงานร่วมกัน ปัญหาที่เกิดขึ้นบ่อย:

**1. Environment Contention**
```
ปัญหา:
- Team A กำลัง deploy ใน Staging
- Team B ต้องการ test ใน Staging
- Team C break Staging environment

แนวทางแก้ไข:
- Dynamic environments per branch
- Environment reservation system
- Parallel test environments
```

**2. Dependency Hell**
```
Service A (Team A) depends on:
  └── Service B (Team B) v2.3
      └── Service C (Team C) v1.8
          └── Service D (Team D) v3.1

เมื่อ Team D เปลี่ยน Service D interface
→ กระทบ C, B, A ทั้งหมด
```

**การแก้ปัญหา Dependency:**
```yaml
# Consumer Driven Contract Testing
# pact.yml

provider:
  name: service-d

consumers:
  - service-c
  - service-b-via-c

verification:
  publishResults: true
  providerVersion: ${VERSION}
  
# ถ้า contract fail → pipeline ของ Team D fail
# ก่อนที่จะ affect downstream services
```

### Release Train Model

```
Week 1        Week 2        Week 3        Week 4
│             │             │             │
▼             ▼             ▼             ▼
Feature       Feature       Feature       Release
Freeze        Testing       Hardening     Day
(Mon)         (Mon-Fri)     (Mon-Thu)     (Fri)

Teams board train ที่ Feature Freeze
ถ้าไม่พร้อม → ต้องรอ train หน้า
```

**Release Train Coordination Pipeline:**
```yaml
name: Release Train Coordination

on:
  schedule:
    - cron: '0 9 * * 1'  # ทุกวันจันทร์เช้า

jobs:
  check-release-readiness:
    runs-on: ubuntu-latest
    steps:
      - name: Check all services health
        run: |
          services=(
            "service-a" "service-b" "service-c"
            "service-d" "service-e"
          )
          
          for service in "${services[@]}"; do
            status=$(curl -s "https://api.internal/${service}/health")
            if [ "$status" != "healthy" ]; then
              echo "❌ ${service} is not healthy"
              exit 1
            fi
            echo "✅ ${service} is healthy"
          done
      
      - name: Check contract tests
        run: |
          pact-broker can-i-deploy \
            --pacticipant all-services \
            --to production
      
      - name: Notify teams
        run: |
          ./scripts/notify-release-train.sh
```

### API Versioning Strategy สำหรับ Multi-Team

```
Strategy 1: Semver-based
  v1.0.0 → v1.1.0 (backward compatible)
  v1.0.0 → v2.0.0 (breaking change)

Strategy 2: Date-based  
  /api/2024-01-01/users
  /api/2024-06-01/users (new version)

Strategy 3: Header-based
  X-API-Version: 2
  Accept: application/vnd.company.v2+json
```

---

## 4. Golden Path Architecture {#golden-path}

### Golden Path คืออะไร?

Golden Path คือ "เส้นทางที่แนะนำ" สำหรับการสร้างและ deploy applications ในองค์กร:

```
Developer เริ่มต้นสร้าง service ใหม่:

Without Golden Path:
1. ค้นหา template ใน wiki (ไม่รู้จะหาที่ไหน)
2. Copy pipeline จาก service อื่น (อาจ outdated)
3. ตั้งค่า infrastructure เอง (ไม่รู้จะทำยังไง)
4. Debug issues (ใช้เวลา 2-3 วัน)
ใช้เวลา: ~1 สัปดาห์

With Golden Path:
1. เรียก CLI: company-cli new service my-service
2. ตอบคำถาม 5 ข้อ (language, type, team)
3. Service พร้อม deploy ใน 10 นาที
ใช้เวลา: ~30 นาที
```

### การสร้าง Golden Path

**Step 1: Discovery - เข้าใจ Developer Journey**
```bash
# Survey ทีม developers
1. Pain points ใน CI/CD workflow ที่ใหญ่ที่สุด?
2. ใช้เวลาเฉลี่ยเท่าไหร่ในการตั้งค่า pipeline ใหม่?
3. อะไรที่ทำซ้ำบ่อยที่สุด?
4. อะไรที่ error บ่อยที่สุด?
```

**Step 2: Design Golden Path Components**
```
Golden Path Components:
┌──────────────────────────────────────────────┐
│            Service Template                   │
│  - Language templates (Go, Java, Python, TS)  │
│  - Standard directory structure               │
│  - Pre-configured CI/CD                       │
│  - Built-in security scanning                 │
└──────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│         Infrastructure Template               │
│  - Kubernetes manifests                       │
│  - Terraform modules                          │
│  - Service mesh config                        │
│  - Observability setup                        │
└──────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────┐
│          Pipeline Template                    │
│  - Build, test, security scan                 │
│  - Multi-environment deployment               │
│  - Automated quality gates                    │
│  - Notification & reporting                   │
└──────────────────────────────────────────────┘
```

**Step 3: Implement Service Scaffolding CLI**
```bash
#!/bin/bash
# company-cli new service

SERVICE_NAME=$1
LANGUAGE=$2
TEAM=$3

echo "🚀 Creating new service: ${SERVICE_NAME}"

# สร้าง repository
gh repo create "company/${SERVICE_NAME}" \
  --template "company/service-template-${LANGUAGE}" \
  --private

# Clone
git clone "https://github.com/company/${SERVICE_NAME}"
cd "${SERVICE_NAME}"

# ตั้งค่า team-specific config
sed -i "s/{{TEAM}}/${TEAM}/g" .github/workflows/*.yml
sed -i "s/{{SERVICE_NAME}}/${SERVICE_NAME}/g" deploy/*.yaml

# ตั้งค่า secrets
gh secret set DEPLOY_TOKEN \
  --repo "company/${SERVICE_NAME}" \
  --body "$(vault read -field=token secret/deploy/${TEAM})"

# ตั้งค่า branch protection
gh api repos/company/${SERVICE_NAME}/branches/main/protection \
  --method PUT \
  --field required_status_checks='{"strict":true,"contexts":["ci/build","ci/test","ci/security"]}'

echo "✅ Service ${SERVICE_NAME} created successfully!"
echo "📋 Next steps:"
echo "   1. cd ${SERVICE_NAME}"
echo "   2. git checkout -b feature/initial-implementation"
echo "   3. Start coding!"
```

### Pipeline Template Design

```yaml
# .github/workflows/golden-path-ci.yml
# Template ที่ทุก service ใช้เป็น base

name: Golden Path CI/CD

on:
  push:
    branches: [main, 'release/**']
  pull_request:
    branches: [main]

env:
  SERVICE_NAME: ${{ vars.SERVICE_NAME }}
  TEAM: ${{ vars.TEAM }}
  
jobs:
  # Stage 1: Quality Gates
  quality-check:
    name: Quality Gates
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Lint
        uses: ./.github/actions/lint
        
      - name: Unit Tests
        uses: ./.github/actions/test
        with:
          test-type: unit
          coverage-threshold: '80'
          
      - name: SAST Security Scan
        uses: ./.github/actions/security-scan
        with:
          scan-type: sast
          
      - name: SCA Dependency Scan
        uses: ./.github/actions/security-scan
        with:
          scan-type: sca
  
  # Stage 2: Build & Package
  build:
    name: Build & Package
    needs: quality-check
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.build.outputs.image-tag }}
    steps:
      - name: Build Docker Image
        id: build
        uses: ./.github/actions/build
        with:
          registry: ${{ vars.REGISTRY }}
          sign-image: true
          
      - name: Container Security Scan
        uses: ./.github/actions/security-scan
        with:
          scan-type: container
          image: ${{ steps.build.outputs.image-tag }}
  
  # Stage 3: Deploy Dev
  deploy-dev:
    name: Deploy to Dev
    needs: build
    environment: dev
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        uses: ./.github/actions/deploy
        with:
          environment: dev
          image-tag: ${{ needs.build.outputs.image-tag }}
          
      - name: Smoke Tests
        uses: ./.github/actions/smoke-test
        with:
          environment: dev
  
  # Stage 4: Deploy Staging (main branch only)
  deploy-staging:
    name: Deploy to Staging
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        uses: ./.github/actions/deploy
        with:
          environment: staging
          image-tag: ${{ needs.build.outputs.image-tag }}
          
      - name: Integration Tests
        uses: ./.github/actions/integration-test
        
      - name: Performance Tests
        uses: ./.github/actions/perf-test
        with:
          threshold-p95: '200ms'
  
  # Stage 5: Deploy Production (manual approval)
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: 
      name: production
      url: https://${{ vars.SERVICE_NAME }}.company.com
    runs-on: ubuntu-latest
    steps:
      - name: Deploy with Canary
        uses: ./.github/actions/deploy
        with:
          environment: production
          image-tag: ${{ needs.build.outputs.image-tag }}
          strategy: canary
          canary-percentage: '10'
```

---

## 5. Shared Infrastructure {#shared-infrastructure}

### Infrastructure as Code สำหรับ Shared CI/CD Components

```hcl
# terraform/modules/cicd-infrastructure/main.tf

# Kubernetes Cluster สำหรับ CI/CD Runners
module "cicd_cluster" {
  source = "./modules/gke-cluster"
  
  name         = "cicd-${var.environment}"
  region       = var.region
  machine_type = "n2-standard-8"
  
  node_pools = {
    "small-runners" = {
      machine_type = "n2-standard-4"
      min_count    = 5
      max_count    = 100
      spot         = true  # ประหยัดค่าใช้จ่าย 60-70%
    }
    "large-runners" = {
      machine_type = "n2-standard-16"
      min_count    = 2
      max_count    = 20
      spot         = false
    }
    "gpu-runners" = {
      machine_type  = "n1-standard-8"
      accelerator   = "nvidia-tesla-t4"
      accelerator_count = 1
      min_count    = 0
      max_count    = 10
    }
  }
}

# Artifact Registry
resource "google_artifact_registry_repository" "containers" {
  location      = var.region
  repository_id = "containers"
  format        = "DOCKER"
  
  cleanup_policies {
    id     = "delete-old-images"
    action = "DELETE"
    
    condition {
      older_than   = "2592000s"  # 30 วัน
      tag_state    = "UNTAGGED"
    }
  }
}

# Secret Manager
resource "google_secret_manager_secret" "cicd_secrets" {
  for_each  = var.secret_names
  secret_id = each.value
  
  replication {
    automatic = true
  }
}
```

### Container Registry ระดับ Enterprise

```yaml
# registry-config.yaml
# Nexus Repository Manager Configuration

repositories:
  # Docker repositories
  docker-internal:
    type: hosted
    policy: allow_redeploy_latest
    cleanup: 30-days
    
  docker-proxy-dockerhub:
    type: proxy
    remote-url: https://registry-1.docker.io
    cache-enabled: true
    negative-cache: true
    
  docker-proxy-gcr:
    type: proxy
    remote-url: https://gcr.io
    
  docker-group:
    type: group
    members:
      - docker-internal
      - docker-proxy-dockerhub
      - docker-proxy-gcr

# Access Control
security:
  roles:
    developer:
      privileges:
        - nx-repository-view-docker-docker-internal-browse
        - nx-repository-view-docker-docker-group-browse
    ci-system:
      privileges:
        - nx-repository-view-docker-*-*
        - nx-component-upload
```

### Shared Secrets Management

```
Enterprise Secret Management Architecture:

┌─────────────────────────────────────────┐
│         HashiCorp Vault Cluster         │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Primary │  │ Replica │  │ Replica │ │
│  └─────────┘  └─────────┘  └─────────┘ │
└─────────────────────────────────────────┘
         │
         │ Dynamic Secrets
         │
    ┌────▼────────────────────────────┐
    │  Secret Paths by Team           │
    │                                 │
    │  secret/team-a/prod/*           │
    │  secret/team-b/prod/*           │
    │  secret/shared/databases/*      │
    │  secret/shared/certificates/*   │
    └─────────────────────────────────┘
```

```hcl
# vault-policies.hcl

# Policy สำหรับ Team A CI/CD
path "secret/team-a/*" {
  capabilities = ["read", "list"]
}

path "secret/shared/databases/team-a-*" {
  capabilities = ["read"]
}

# Dynamic Database Credentials
path "database/creds/team-a-role" {
  capabilities = ["read"]
}

# ไม่อนุญาตให้แก้ไข production secrets ผ่าน CI
path "secret/team-a/prod/*" {
  capabilities = ["read"]
  
  # ต้องมี MFA
  mfa_methods = ["totp"]
}
```

---

## 6. Governance Model {#governance-model}

### CI/CD Governance Framework

```
Enterprise CI/CD Governance Framework:

┌─────────────────────────────────────────────┐
│           STRATEGIC LAYER                    │
│  - CI/CD Standards & Policies                │
│  - Technology Radar                          │
│  - Architecture Review Board                 │
└─────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────┐
│           TACTICAL LAYER                     │
│  - Platform Team (Core)                      │
│  - Security Team Integration                 │
│  - Compliance Requirements                   │
└─────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────────────────┐
│         OPERATIONAL LAYER                    │
│  - Team-level Pipelines                      │
│  - Developer Workflows                       │
│  - Day-to-day Operations                     │
└─────────────────────────────────────────────┘
```

### Pipeline Policy as Code

```yaml
# opa/pipeline-policy.rego
# Open Policy Agent สำหรับ enforce pipeline standards

package pipeline.policy

# ทุก pipeline ต้องมี security scanning
deny[msg] {
  not has_security_scan
  msg = "Pipeline must include security scanning"
}

has_security_scan {
  input.jobs[_].steps[_].uses == "company/sast-action@v1"
}

# ทุก production deployment ต้องมี approval
deny[msg] {
  job := input.jobs[_]
  is_production_deploy(job)
  not has_required_approval(job)
  msg = sprintf("Production deployment job '%v' must require approval", [job.name])
}

is_production_deploy(job) {
  job.environment.name == "production"
}

has_required_approval(job) {
  job.environment.url != null
  count(job.environment.reviewers) > 0
}

# Image ต้อง sign ก่อน production
deny[msg] {
  job := input.jobs[_]
  is_production_deploy(job)
  not uses_signed_image(job)
  msg = "Production deployments must use signed container images"
}
```

### Compliance Gates

```yaml
# compliance-gate.yml
# Automated Compliance Checking

name: Compliance Gate

jobs:
  compliance-check:
    steps:
      # 1. License Compliance
      - name: Check Licenses
        run: |
          license-checker \
            --onlyAllow "MIT;Apache-2.0;BSD-2-Clause;BSD-3-Clause;ISC" \
            --production \
            --failOnUnlicensed
      
      # 2. Vulnerability Compliance
      - name: Check CVE Policy
        run: |
          # ไม่อนุญาต Critical/High CVE ใน production
          trivy image \
            --exit-code 1 \
            --severity CRITICAL,HIGH \
            --ignore-unfixed \
            ${{ env.IMAGE_TAG }}
      
      # 3. Code Coverage Compliance
      - name: Check Coverage Threshold
        run: |
          coverage=$(cat coverage.json | jq '.total.lines.pct')
          threshold=80
          
          if (( $(echo "$coverage < $threshold" | bc -l) )); then
            echo "❌ Coverage ${coverage}% is below threshold ${threshold}%"
            exit 1
          fi
          echo "✅ Coverage ${coverage}% meets threshold"
      
      # 4. Dependency Freshness
      - name: Check Dependency Age
        run: |
          # แจ้งเตือนถ้า dependency เก่ากว่า 6 เดือน
          npm audit --json | \
            jq '.vulnerabilities | 
                to_entries[] | 
                select(.value.severity == "high" or .value.severity == "critical")' | \
            wc -l | \
            xargs -I{} test {} -eq 0
```

### Audit Trail

```yaml
# audit-logger.yml
# บันทึก audit trail สำหรับทุก deployment

name: Deployment Audit

jobs:
  log-deployment:
    steps:
      - name: Log to Audit System
        run: |
          curl -X POST https://audit.internal/events \
            -H "Authorization: Bearer ${{ secrets.AUDIT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "event_type": "deployment",
              "service": "${{ env.SERVICE_NAME }}",
              "version": "${{ env.VERSION }}",
              "environment": "${{ env.ENVIRONMENT }}",
              "deployer": "${{ github.actor }}",
              "timestamp": "${{ env.TIMESTAMP }}",
              "commit_sha": "${{ github.sha }}",
              "pr_number": "${{ github.event.pull_request.number }}",
              "approved_by": ${{ toJson(env.APPROVERS) }},
              "pipeline_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
            }'
```

---

## 7. Scaling Challenges และ Solutions {#scaling}

### Challenge 1: Build Performance at Scale

**ปัญหา:** เมื่อมี 1000+ builds/day, pipeline ช้าลงเรื่อยๆ

**วิธีแก้: Distributed Build Cache**
```yaml
# cache-architecture.yml

# Level 1: Local Cache (per runner)
# Level 2: Remote Cache (shared)
# Level 3: Distributed Build (Bazel/Buck2)

# การตั้งค่า Remote Cache ด้วย Bazel
build --remote_cache=grpc://cache.internal:9092
build --remote_instance_name=default

# Cache Hit Rate เป้าหมาย: >85%
# ผล: Build time ลดจาก 25 นาที → 8 นาที
```

**Benchmark การปรับปรุง:**
```
Before optimization:
- Average build time: 25 min
- P95 build time: 45 min
- Cache hit rate: 12%
- Monthly build cost: $45,000

After optimization:
- Average build time: 8 min (-68%)
- P95 build time: 15 min (-67%)
- Cache hit rate: 87%
- Monthly build cost: $18,000 (-60%)
```

### Challenge 2: Runner Management at Scale

```yaml
# runner-autoscaling.yml
# Kubernetes-based runner autoscaling

apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: enterprise-runners
spec:
  template:
    spec:
      repository: company/enterprise
      labels:
        - ubuntu-latest
        - self-hosted
      
---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: enterprise-runners-autoscaler
spec:
  scaleTargetRef:
    name: enterprise-runners
    
  minReplicas: 10
  maxReplicas: 500
  
  metrics:
    - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
      repositoryNames:
        - company/service-a
        - company/service-b
        
  scaleDownDelaySecondsAfterScaleOut: 300
```

### Challenge 3: Cost Optimization

```python
# scripts/cost-optimizer.py
# วิเคราะห์และ optimize CI/CD costs

import json
from datetime import datetime, timedelta

def analyze_pipeline_costs():
    """วิเคราะห์ cost ของแต่ละ pipeline"""
    
    pipelines = fetch_pipeline_metrics()
    
    report = []
    for pipeline in pipelines:
        cost_per_run = calculate_cost(
            duration_minutes=pipeline['avg_duration'],
            runner_type=pipeline['runner_type'],
            run_count=pipeline['runs_per_day']
        )
        
        # หา optimization opportunities
        opportunities = []
        
        if pipeline['cache_hit_rate'] < 0.7:
            potential_saving = cost_per_run * 0.4
            opportunities.append({
                'type': 'improve_cache',
                'potential_saving': potential_saving,
                'recommendation': 'เพิ่ม caching layer'
            })
        
        if pipeline['avg_duration'] > 20 and pipeline['parallelizable']:
            potential_saving = cost_per_run * 0.3
            opportunities.append({
                'type': 'parallelize',
                'potential_saving': potential_saving,
                'recommendation': 'แบ่ง jobs เป็น parallel'
            })
        
        if pipeline['uses_large_runner'] and pipeline['avg_cpu'] < 0.4:
            potential_saving = cost_per_run * 0.5
            opportunities.append({
                'type': 'right_size_runner',
                'potential_saving': potential_saving,
                'recommendation': 'ลด runner size ลง'
            })
        
        report.append({
            'pipeline': pipeline['name'],
            'monthly_cost': cost_per_run * pipeline['runs_per_day'] * 30,
            'opportunities': opportunities
        })
    
    return sorted(report, key=lambda x: x['monthly_cost'], reverse=True)
```

---

## 8. Case Studies {#case-studies}

### Case Study 1: ธนาคารขนาดใหญ่ - Migration from Jenkins to Modern CI/CD

**บริบท:**
- 800 developers, 400 repositories
- 15 ปีของ Jenkins configuration
- 2,000+ pipeline jobs
- Compliance requirements: SOX, PCI-DSS

**ปัญหาเดิม:**
```
- Build pipeline ใช้เวลาเฉลี่ย 45 นาที
- Jenkins อัพเกรดยาก (มี custom plugins 50+)
- Security vulnerabilities ใน pipeline infrastructure
- Audit trail ไม่สมบูรณ์
- 3 Ops ต้องดูแล Jenkins ตลอดเวลา
```

**Solution ที่นำมาใช้:**
```
Phase 1 (3 เดือน): Assessment & Planning
- Audit ทุก pipeline
- Define golden path
- Train Platform Team

Phase 2 (6 เดือน): Foundation
- ตั้งค่า new platform (GitHub Actions + Argo CD)
- สร้าง shared pipeline templates
- Migrate 20% ของ repositories

Phase 3 (9 เดือน): Migration
- Migrate อีก 60% ของ repositories
- Decommission Jenkins ทีละส่วน
- Training developers

Phase 4 (3 เดือน): Optimization
- Fine-tune performance
- Full decommission ของ Jenkins
- Handover to teams
```

**ผลลัพธ์:**
```
Build time: 45 min → 12 min (73% improvement)
Deployment frequency: 2x/week → 3x/day
MTTR: 4 hours → 45 minutes
Compliance audit time: 3 days → 2 hours (automated)
Infrastructure cost: -40%
Developer satisfaction: 6.2/10 → 8.7/10
```

### Case Study 2: E-commerce Platform - Scaling to Black Friday

**บริบท:**
- 200 developers, 600 microservices
- Traffic spike 50x ในวัน Black Friday
- Zero tolerance for downtime

**CI/CD Strategy:**
```yaml
# black-friday-strategy.yml

# 4 สัปดาห์ก่อน Black Friday:
pre-black-friday:
  deployment-freeze: false
  enhanced-testing:
    - load-testing: 2x-normal
    - chaos-engineering: enabled
    - canary-percentage: 5%  # เพิ่มความระมัดระวัง
  
# 2 สัปดาห์ก่อน Black Friday:
near-black-friday:
  deployment-freeze: partial  # เฉพาะ critical paths
  enhanced-testing:
    - load-testing: 5x-normal
    - canary-percentage: 1%  # ระมัดระวังมาก
  
# วัน Black Friday:
black-friday:
  deployment-freeze: true  # ห้าม deploy
  exception-process: emergency-change-only
  rollback-ready: true
  
# หลัง Black Friday (1 สัปดาห์):
post-black-friday:
  deployment-freeze: false
  catch-up-sprints: true
```

---

## 9. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Enterprise Architecture Assessment

**เป้าหมาย:** ประเมิน maturity level ของ CI/CD ในองค์กร

**ขั้นตอน:**
1. ดาวน์โหลด CI/CD Maturity Assessment template
2. ประเมิน 5 มิติ: People, Process, Technology, Security, Measurement
3. กำหนด current state และ desired state
4. สร้าง roadmap 12 เดือน

**Maturity Model:**
```
Level 1 - Initial:
  - Manual deployments
  - No standard process
  - Ad-hoc tooling

Level 2 - Managed:
  - Basic CI implemented
  - Some automation
  - Team-level standards

Level 3 - Defined:
  - CI/CD for all services
  - Org-wide standards
  - Shared templates

Level 4 - Measured:
  - DORA metrics tracked
  - SLO for pipelines
  - Cost visibility

Level 5 - Optimizing:
  - Continuous improvement
  - Platform as Product
  - AI-assisted optimization
```

### แบบฝึกหัดที่ 2: Golden Path Design

**เป้าหมาย:** ออกแบบ Golden Path สำหรับ organization สมมติ

**สถานการณ์:**
- บริษัท SaaS, 300 developers
- Languages: Python, Go, TypeScript
- Infrastructure: AWS, Kubernetes
- Compliance: SOC2

**งาน:**
1. สร้าง service template สำหรับ 1 language
2. ออกแบบ pipeline template ที่ครอบคลุม security requirements
3. สร้าง CLI command สำหรับ scaffold service ใหม่
4. กำหนด Golden Path documentation

```bash
# Expected output:
$ company-cli new service my-service --lang python --team backend

Creating service: my-service
├── Repository: github.com/company/my-service ✅
├── CI/CD Pipeline: configured ✅
├── Infrastructure: provisioned ✅
├── Security Scanning: enabled ✅
├── Observability: configured ✅
└── Documentation: created ✅

Service ready in 8 minutes!
Next steps: cd my-service && make dev
```

### แบบฝึกหัดที่ 3: Multi-Team Coordination

**สถานการณ์:**
- 5 teams, 10 microservices
- 3 services มี circular dependency
- Release deadline ใน 2 สัปดาห์

**งาน:**
1. วาด dependency graph
2. ระบุ critical path
3. ออกแบบ contract testing strategy
4. สร้าง release coordination plan

### แบบฝึกหัดที่ 4: Cost Optimization

**สถานการณ์:**
- Current CI/CD cost: $80,000/month
- 60% ของ builds ใช้ large runners แต่ average CPU < 30%
- Cache hit rate: 45%
- 30% ของ pipeline runs ไม่จำเป็น (triggered by non-code changes)

**งาน:**
1. วิเคราะห์ savings potential
2. สร้าง optimization plan
3. คำนวณ expected cost reduction
4. Define success metrics

```python
# คำนวณ potential savings
current_cost = 80_000  # USD/month

# Optimization 1: Right-size runners
runner_saving = current_cost * 0.60 * 0.40  # 60% builds, 40% potential saving
# = $19,200/month

# Optimization 2: Improve cache
cache_improvement = current_cost * 0.30  # ถ้า cache hit rate 45%→85%
# = $24,000/month

# Optimization 3: Reduce unnecessary runs
trigger_optimization = current_cost * 0.30 * 0.60
# = $14,400/month

total_potential_saving = runner_saving + cache_improvement + trigger_optimization
# = $57,600/month (72% reduction!)
print(f"Potential savings: ${total_potential_saving:,.0f}/month")
```

---

## สรุป

Enterprise CI/CD Architecture ต้องการการวางแผนที่รอบคอบและมองภาพรวม key takeaways:

1. **Start with People and Process** ก่อน Technology
2. **Golden Path ลด friction** และเพิ่ม developer productivity ได้มหาศาล
3. **Governance ไม่ใช่ Bottleneck** ถ้าออกแบบด้วย automation
4. **Cost visibility สำคัญมาก** ใน enterprise scale
5. **Incremental migration** ดีกว่า Big Bang approach

## อ่านเพิ่มเติม

- "Accelerate" by Nicole Forsgren, Jez Humble, Gene Kim
- "Team Topologies" by Matthew Skelton, Manuel Pais
- "The Phoenix Project" by Gene Kim, Kevin Behr, George Spafford
- Google SRE Book: https://sre.google/books/
- DORA Research: https://dora.dev/research/

---

*Part 81 จาก 100 | CI/CD Mastery Course*

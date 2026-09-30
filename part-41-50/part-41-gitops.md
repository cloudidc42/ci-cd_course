# Part 41: GitOps — แนวคิดและการนำไปปฏิบัติ

## สารบัญ
1. [GitOps คืออะไร?](#gitops-คืออะไร)
2. [Git as Single Source of Truth](#git-as-single-source-of-truth)
3. [Pull vs Push Deployment](#pull-vs-push-deployment)
4. [GitOps Operators](#gitops-operators)
5. [GitOps Workflow](#gitops-workflow)
6. [ข้อดีและความท้าทาย](#ขอดีและความทาทาย)
7. [Best Practices](#best-practices)
8. [Workshop Exercises](#workshop-exercises)

---

## GitOps คืออะไร?

GitOps เป็นแนวทางการทำงาน (operational framework) ที่นำ Git ไปใช้เป็น "single source of truth" สำหรับการจัดการ infrastructure และ application deployment โดยใช้หลักการของ DevOps ร่วมกับ version control

### ประวัติและต้นกำเนิด

GitOps ถูกแนะนำครั้งแรกโดย Weaveworks ในปี 2017 โดย Alexis Richardson โดยมีแนวคิดหลักว่า:

```
"Operations by pull request"
```

ทุกการเปลี่ยนแปลงใน production environment ต้องผ่าน Git pull request เท่านั้น

### หลักการพื้นฐาน 4 ข้อของ GitOps

```
┌─────────────────────────────────────────────────────────────┐
│                    GitOps Principles                         │
├─────────────────────────────────────────────────────────────┤
│ 1. Declarative  - ระบบทุกอย่างต้องถูก describe แบบ          │
│                   declarative                                │
│                                                             │
│ 2. Versioned    - desired state ต้อง versioned อยู่ใน Git   │
│                   และ immutable                              │
│                                                             │
│ 3. Pulled       - software agents ต้อง pull การ approve     │
│                   configuration จาก source                  │
│                                                             │
│ 4. Continuously - software agents ต้อง continuously        │
│    Reconciled     observe actual state และ attempt to       │
│                   apply desired state                       │
└─────────────────────────────────────────────────────────────┘
```

### เปรียบเทียบ Traditional DevOps vs GitOps

| ด้าน | Traditional DevOps | GitOps |
|------|-------------------|--------|
| การ deploy | Push จาก CI pipeline | Pull จาก operator |
| State management | Manual/Scripts | Declarative in Git |
| Rollback | Manual process | git revert |
| Audit trail | CI logs | Git history |
| Access control | CI/CD system | Git permissions |
| Drift detection | Manual | Automated |

---

## Git as Single Source of Truth

ใน GitOps ทุก configuration และ desired state จะถูกเก็บไว้ใน Git repository ทำให้ Git กลายเป็น "single source of truth"

### โครงสร้าง Repository

#### Pattern 1: Monorepo
```
gitops-repo/
├── applications/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   └── database/
│       ├── statefulset.yaml
│       └── pvc.yaml
├── infrastructure/
│   ├── namespaces/
│   │   ├── production.yaml
│   │   ├── staging.yaml
│   │   └── development.yaml
│   ├── rbac/
│   │   ├── roles.yaml
│   │   └── rolebindings.yaml
│   └── network-policies/
│       └── default-deny.yaml
└── environments/
    ├── production/
    │   └── kustomization.yaml
    ├── staging/
    │   └── kustomization.yaml
    └── development/
        └── kustomization.yaml
```

#### Pattern 2: Multi-Repo (แยก App และ Config)
```
# Repository 1: Application Code
app-repo/
├── src/
├── tests/
├── Dockerfile
└── .github/workflows/
    └── ci.yaml      # Build & push image, then update config repo

# Repository 2: GitOps Configuration
config-repo/
├── base/
│   ├── deployment.yaml    # Template
│   └── kustomization.yaml
└── overlays/
    ├── production/
    │   ├── kustomization.yaml
    │   └── patch.yaml
    └── staging/
        ├── kustomization.yaml
        └── patch.yaml
```

### Declarative Configuration Examples

#### Kubernetes Deployment
```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: "1.0.0"
    spec:
      containers:
      - name: webapp
        image: myregistry/webapp:1.0.0   # GitOps operator จะ update นี้
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

#### Kustomize Overlay สำหรับ Production
```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
- ../../base

patches:
- path: replica-patch.yaml
- path: resource-patch.yaml

images:
- name: myregistry/webapp
  newTag: "2.1.0"   # GitOps automation จะ update tag นี้

commonLabels:
  environment: production

configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=INFO
  - MAX_CONNECTIONS=100
```

```yaml
# overlays/production/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 10   # Production ใช้ 10 replicas
```

### Secrets Management ใน GitOps

ปัญหาสำคัญของ GitOps คือการจัดการ secrets เนื่องจาก secrets ไม่ควรถูกเก็บใน Git โดยตรง

#### วิธีที่ 1: Sealed Secrets
```yaml
# ก่อน seal - secret ปกติ
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
data:
  url: cG9zdGdyZXNxbDovL3VzZXI6cGFzc3dvcmRAZGI6NTQzMi9hcHA=
```

```bash
# ติดตั้ง kubeseal
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Seal the secret
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

```yaml
# หลัง seal - SealedSecret (safe to commit to Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-secret
  namespace: production
spec:
  encryptedData:
    url: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...  # encrypted value
  template:
    metadata:
      name: db-secret
      namespace: production
    type: Opaque
```

#### วิธีที่ 2: External Secrets Operator
```yaml
# ExternalSecret ดึง secrets จาก AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  data:
  - secretKey: url
    remoteRef:
      key: production/database
      property: url
```

```yaml
# ClusterSecretStore
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
```

---

## Pull vs Push Deployment

นี่คือความแตกต่างที่สำคัญที่สุดระหว่าง GitOps และ Traditional CI/CD

### Push-Based Deployment (Traditional)

```
┌──────────┐    Push     ┌───────────┐    Deploy    ┌────────────┐
│Developer │──commit────>│ CI/CD     │─────────────>│ Kubernetes │
│          │             │ Pipeline  │              │ Cluster    │
└──────────┘             └───────────┘              └────────────┘
                              │
                    (ต้องมี credentials ของ cluster)
```

```yaml
# GitHub Actions - Push-based deployment
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build and push Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
        docker push myapp:${{ github.sha }}
    
    # ต้องมี kubeconfig หรือ credentials ใน CI/CD
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
      
    - name: Deploy to Kubernetes
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG }}   # ⚠️ credentials ใน CI/CD
      run: |
        kubectl set image deployment/webapp \
          webapp=myapp:${{ github.sha }}
        kubectl rollout status deployment/webapp
```

**ปัญหาของ Push-based:**
- CI/CD system ต้องมี credentials เข้าถึง cluster โดยตรง
- Firewall/network ต้องเปิดให้ CI/CD เข้าถึง cluster
- ถ้า CI/CD ถูก compromise ทั้ง cluster ก็มีความเสี่ยง
- ยากต่อการ detect configuration drift

### Pull-Based Deployment (GitOps)

```
┌──────────┐    Push     ┌───────────┐
│Developer │──commit────>│    Git    │
│          │             │ Repository│
└──────────┘             └─────┬─────┘
                               │ watches
                         ┌─────▼─────┐    reconcile   ┌────────────┐
                         │  GitOps   │───────────────>│ Kubernetes │
                         │ Operator  │                │ Cluster    │
                         │(ArgoCD/   │<─────────────── │            │
                         │ Flux)     │  observes state │            │
                         └───────────┘                └────────────┘
```

```yaml
# ArgoCD Application - Pull-based deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/config-repo.git
    targetRevision: HEAD
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true     # ลบ resources ที่ไม่อยู่ใน Git
      selfHeal: true  # แก้ไข drift โดยอัตโนมัติ
    syncOptions:
    - CreateNamespace=true
```

**ข้อดีของ Pull-based:**
- ไม่ต้อง expose cluster credentials ออกนอก
- Cluster เริ่ม operation เอง (more secure)
- Automatic drift detection และ remediation
- Network: cluster ต้องการเพียง outbound access ไปยัง Git

### Comparison Table

| ประเด็น | Push | Pull |
|---------|------|------|
| Security | CI/CD ต้องเข้า cluster | Operator อยู่ใน cluster |
| Credentials | ใน CI/CD | ใน cluster เท่านั้น |
| Drift Detection | ไม่มี | มี (automatic) |
| Network | Inbound to cluster | Outbound from cluster |
| Complexity | ง่ายกว่า | ต้องการ operator |
| Real-time sync | ทุก deploy | Continuous (15s-5m) |

---

## GitOps Operators

### ArgoCD

ArgoCD เป็น GitOps operator ที่ได้รับความนิยมสูงสุด

```
┌─────────────────────────────────────────────────┐
│                  ArgoCD                          │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌──────────┐   ┌──────────┐   ┌────────────┐  │
│  │   API    │   │  Repo    │   │App         │  │
│  │  Server  │   │  Server  │   │Controller  │  │
│  └────┬─────┘   └────┬─────┘   └─────┬──────┘  │
│       │              │               │          │
│  ┌────▼─────────────▼───────────────▼──────┐  │
│  │              Redis Cache                 │  │
│  └────────────────────────────────────────-─┘  │
│                                                 │
└─────────────────────────────────────────────────┘
```

**ติดตั้ง ArgoCD:**
```bash
# สร้าง namespace
kubectl create namespace argocd

# ติดตั้ง ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ตรวจสอบสถานะ
kubectl get pods -n argocd

# เข้าถึง UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# ดู initial password
argocd admin initial-password -n argocd
```

### Flux CD

Flux เป็นอีก GitOps operator ที่ใช้ CNCF incubating project

```bash
# ติดตั้ง Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux กับ GitHub
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

### Jenkins X

Jenkins X เหมาะสำหรับ team ที่ใช้ Jenkins อยู่แล้ว

```yaml
# jenkins-x.yml - Pipeline configuration
buildPack: go
pipelineConfig:
  pipelines:
    pullRequest:
      build:
        preSteps:
        - command: make test
    release:
      build:
        steps:
        - command: make build
        - command: skaffold build
      promote:
        steps:
        - command: jx promote --version $VERSION --env staging
```

---

## GitOps Workflow

### Complete GitOps Workflow

```
Developer                Git Repo              CI System         GitOps Operator     Cluster
    │                       │                      │                    │               │
    │─── git push ─────────>│                      │                    │               │
    │                       │─── trigger CI ──────>│                    │               │
    │                       │                      │─── build image ───>│               │
    │                       │                      │─── push to registry│               │
    │                       │                      │─── create PR ─────>│               │
    │                       │<── PR created ────────│                   │               │
    │<── PR notification ───│                       │                    │               │
    │─── review & merge ───>│                       │                    │               │
    │                       │                       │                   │               │
    │                       │ (config updated) ──────────────────── pulls changes       │
    │                       │                       │                    │─── apply ───>│
    │                       │                       │                    │<── status ───│
    │                       │                       │           sync status updated     │
```

### Step 1: Developer Workflow

```bash
# Developer สร้าง feature branch
git checkout -b feature/add-payment

# ทำงานและ commit
git add .
git commit -m "feat: add payment processing"
git push origin feature/add-payment

# สร้าง PR
gh pr create --title "Add payment processing" \
  --body "Implements payment flow with Stripe integration"
```

### Step 2: CI Pipeline (Build & Test)

```yaml
# .github/workflows/ci.yaml
name: CI Pipeline

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run tests
      run: |
        go test ./...
        
    - name: Build Docker image
      if: github.event_name == 'push' && github.ref == 'refs/heads/main'
      run: |
        docker build -t ghcr.io/myorg/webapp:${{ github.sha }} .
        echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
        docker push ghcr.io/myorg/webapp:${{ github.sha }}
        
  update-config:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
    - name: Checkout config repo
      uses: actions/checkout@v4
      with:
        repository: myorg/config-repo
        token: ${{ secrets.CONFIG_REPO_TOKEN }}
        
    - name: Update image tag
      run: |
        cd overlays/production
        kustomize edit set image \
          ghcr.io/myorg/webapp=ghcr.io/myorg/webapp:${{ github.sha }}
          
    - name: Commit and push
      run: |
        git config user.email "ci@myorg.com"
        git config user.name "CI Bot"
        git add .
        git commit -m "chore: update webapp to ${{ github.sha }}"
        git push
```

### Step 3: GitOps Operator Reconciliation

ArgoCD จะ detect การเปลี่ยนแปลงใน config repo และ sync cluster

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/config-repo.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - Validate=true
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Step 4: Notification & Monitoring

```yaml
# ArgoCD Notification - Slack notification
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
  template.app-deployed: |
    message: |
      {{if eq .app.status.operationState.phase "Succeeded"}}:white_check_mark:{{else}}:x:{{end}}
      Application *{{.app.metadata.name}}* sync {{.app.status.operationState.phase}}
      Revision: `{{.app.status.sync.revision}}`
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
```

---

## ข้อดีและความท้าทาย

### ข้อดีของ GitOps

#### 1. Security ที่ดีขึ้น
```
Traditional: CI/CD → Cluster (ต้องการ credentials)
GitOps:      Cluster ← Git (cluster pull เอง)

ผลลัพธ์:
- ลด attack surface
- ไม่ต้อง store cluster credentials ใน CI/CD
- Audit trail ชัดเจนผ่าน Git history
```

#### 2. Developer Experience ที่ดีกว่า
```bash
# Rollback ง่ายมาก
git revert HEAD~1
git push

# ดู history ชัดเจน
git log --oneline overlays/production/

# Compare versions
git diff v1.0.0 v1.1.0 -- overlays/production/
```

#### 3. Consistency และ Reliability
```yaml
# ทุก environment ใช้ Git เป็น source of truth
# ไม่มีการ manual change ที่ไม่ได้ track
# Drift detection โดยอัตโนมัติ

# ArgoCD จะ show OutOfSync เมื่อ cluster state ≠ Git state
```

#### 4. Faster Recovery
```
MTTR (Mean Time To Recovery) ลดลงเพราะ:
1. Rollback = git revert (ไม่กี่นาที)
2. Disaster recovery = apply Git configs
3. New cluster = bootstrap จาก Git
```

### ความท้าทายของ GitOps

#### 1. Secret Management ซับซ้อน
```
ปัญหา: Secrets ไม่ควรอยู่ใน Git
วิธีแก้:
- Sealed Secrets (encrypt secrets ก่อน store ใน Git)
- External Secrets Operator (ดึงจาก Vault/AWS/GCP)
- SOPS (Mozilla Secret Operations)
```

```bash
# SOPS example
# Encrypt secret
sops --encrypt --age age1xxx secret.yaml > secret.enc.yaml

# Decrypt (locally)
sops --decrypt secret.enc.yaml

# สร้าง .sops.yaml config
cat > .sops.yaml << EOF
creation_rules:
  - path_regex: .*.yaml
    age: age1xxx
EOF
```

#### 2. Git Repository Management
```
ความท้าทาย:
- Monorepo vs Multi-repo
- Repository access control
- Merge conflicts
- PR review process สำหรับ config changes
```

#### 3. Learning Curve
```
Team ต้องเรียนรู้:
- Kubernetes manifests
- Kustomize/Helm
- GitOps operator (ArgoCD/Flux)
- Secret management tools
```

#### 4. Progressive Delivery
```yaml
# GitOps ทำ canary/blue-green ได้ แต่ต้องการ tooling เพิ่ม
# เช่น Argo Rollouts

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: webapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 20   # 20% traffic ไปยัง new version
      - pause: {}       # รอ manual approval
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
```

---

## Best Practices

### 1. Repository Structure

```
# Best Practice: แยก App code และ Config
app-repo/
├── src/                    # Application code
├── Dockerfile
└── .github/workflows/
    └── ci.yaml             # Build, test, push image
                            # แล้ว update config-repo

config-repo/
├── base/                   # Base configurations
├── overlays/               # Environment-specific
│   ├── development/
│   ├── staging/
│   └── production/
└── clusters/               # Cluster-specific configs
    ├── us-east/
    └── ap-southeast/
```

### 2. Naming Conventions

```yaml
# Image tag conventions
myapp:latest          # ❌ ไม่ดี - ไม่รู้ว่า version อะไร
myapp:1.2.3           # ✅ SemVer
myapp:abc1234         # ✅ Git SHA
myapp:2024-01-15      # ✅ Date-based
myapp:1.2.3-abc1234   # ✅ SemVer + Git SHA (best)
```

### 3. Sync Policies

```yaml
# Development: Auto-sync เพื่อ fast iteration
syncPolicy:
  automated:
    prune: true
    selfHeal: true

# Production: Semi-auto หรือ Manual sync
syncPolicy:
  automated:
    prune: false      # ไม่ prune โดยอัตโนมัติ
    selfHeal: true    # แต่แก้ drift ได้
  syncOptions:
  - ApplyOutOfSyncOnly=true
```

### 4. GitOps Pipeline Pattern

```yaml
# .github/workflows/gitops-update.yaml
name: Update GitOps Config

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Docker image tag to deploy'
        required: true
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'

jobs:
  update-config:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout config repo
      uses: actions/checkout@v4
      with:
        repository: myorg/config-repo
        token: ${{ secrets.CONFIG_REPO_TOKEN }}
        
    - name: Setup Kustomize
      uses: imranismail/setup-kustomize@v2
      
    - name: Update image tag
      run: |
        cd overlays/${{ github.event.inputs.environment }}
        kustomize edit set image myapp=myregistry/myapp:${{ github.event.inputs.image_tag }}
        
    - name: Create Pull Request
      uses: peter-evans/create-pull-request@v5
      with:
        token: ${{ secrets.CONFIG_REPO_TOKEN }}
        commit-message: "chore: update myapp to ${{ github.event.inputs.image_tag }} in ${{ github.event.inputs.environment }}"
        title: "Deploy myapp:${{ github.event.inputs.image_tag }} to ${{ github.event.inputs.environment }}"
        body: |
          ## Deployment Update
          
          - **Environment**: ${{ github.event.inputs.environment }}
          - **Image Tag**: ${{ github.event.inputs.image_tag }}
          - **Triggered by**: ${{ github.actor }}
          
          Auto-generated by CI/CD pipeline
        branch: deploy/${{ github.event.inputs.environment }}-${{ github.event.inputs.image_tag }}
        base: main
```

### 5. Environment Promotion Strategy

```
┌──────────────────────────────────────────────────────────────┐
│                 Environment Promotion Flow                    │
│                                                              │
│  Dev ──auto──> Staging ──approval──> Production              │
│                                                              │
│  1. CI builds image, pushes to registry                      │
│  2. Auto-update Dev environment                              │
│  3. Integration tests run in Dev                             │
│  4. PR created to promote to Staging                         │
│  5. QA approval required                                     │
│  6. PR created to promote to Production                      │
│  7. Engineering manager approval required                    │
└──────────────────────────────────────────────────────────────┘
```

```yaml
# Promotion script
#!/bin/bash
# promote.sh

SOURCE_ENV=$1
TARGET_ENV=$2
IMAGE_TAG=$3

echo "Promoting $IMAGE_TAG from $SOURCE_ENV to $TARGET_ENV"

# ดึง current tag จาก source environment
CURRENT_TAG=$(kustomize edit get image myapp --path overlays/$SOURCE_ENV | cut -d: -f2)

# Update target environment
cd overlays/$TARGET_ENV
kustomize edit set image myapp=myregistry/myapp:$CURRENT_TAG

# Commit และ push
git add .
git commit -m "chore: promote myapp to $TARGET_ENV (tag: $CURRENT_TAG)"
git push
```

### 6. Disaster Recovery

```bash
# ขั้นตอน Disaster Recovery ด้วย GitOps

# 1. สร้าง Kubernetes cluster ใหม่
eksctl create cluster -f cluster-config.yaml

# 2. ติดตั้ง GitOps operator
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Register Git repository
argocd repo add https://github.com/myorg/config-repo.git \
  --username myuser \
  --password $GITHUB_TOKEN

# 4. Apply all ArgoCD applications
kubectl apply -f argocd-applications/

# 5. ArgoCD จะ sync ทุกอย่างจาก Git โดยอัตโนมัติ
# ใช้เวลาประมาณ 5-10 นาทีแทนที่จะเป็น ชั่วโมง
```

### 7. Multi-Cluster GitOps

```yaml
# config-repo structure สำหรับ multi-cluster
config-repo/
├── base/
│   └── ...
├── overlays/
│   ├── cluster-us-east-1/
│   │   ├── production/
│   │   └── staging/
│   ├── cluster-ap-southeast-1/
│   │   ├── production/
│   │   └── staging/
│   └── cluster-eu-west-1/
│       ├── production/
│       └── staging/
└── argocd/
    ├── appsets/
    │   └── all-clusters.yaml   # ApplicationSet
    └── clusters/
        ├── us-east.yaml
        ├── ap-southeast.yaml
        └── eu-west.yaml
```

```yaml
# ArgoCD ApplicationSet - deploy ไปหลาย clusters พร้อมกัน
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: webapp-all-clusters
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: us-east-1
        url: https://k8s-us-east.example.com
        region: us-east-1
      - cluster: ap-southeast-1
        url: https://k8s-ap.example.com
        region: ap-southeast-1
  template:
    metadata:
      name: 'webapp-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/config-repo.git
        targetRevision: HEAD
        path: 'overlays/cluster-{{cluster}}/production'
      destination:
        server: '{{url}}'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## Workshop Exercises

### Exercise 1: Setup GitOps Repository

```bash
# สร้าง config repository
mkdir gitops-workshop && cd gitops-workshop
git init

# สร้างโครงสร้าง
mkdir -p base overlays/{development,staging,production}

# สร้าง base deployment
cat > base/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.24
        ports:
        - containerPort: 80
EOF

cat > base/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF

cat > base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
EOF

# สร้าง development overlay
cat > overlays/development/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: development
bases:
- ../../base
patches:
- path: replica-patch.yaml
EOF

cat > overlays/development/replica-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 1
EOF

# Build และตรวจสอบ
kustomize build overlays/development

# Commit
git add .
git commit -m "Initial GitOps configuration"
```

### Exercise 2: Setup ArgoCD

```bash
# ติดตั้ง ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# รอให้ pods ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=300s

# ดู initial password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD Password: $ARGOCD_PASSWORD"

# Login
argocd login localhost:8080 \
  --username admin \
  --password $ARGOCD_PASSWORD \
  --insecure

# สร้าง Application
argocd app create webapp \
  --repo https://github.com/YOURORG/gitops-workshop.git \
  --path overlays/development \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace development \
  --sync-policy automated

# ตรวจสอบ sync status
argocd app get webapp
argocd app sync webapp
```

### Exercise 3: Test GitOps Flow

```bash
# 1. เปลี่ยน replica count ใน Git
cat > overlays/development/replica-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
EOF

git add . && git commit -m "scale: increase replicas to 3"
git push

# 2. ดู ArgoCD sync
argocd app get webapp --watch

# 3. ตรวจสอบ pods
kubectl get pods -n development

# 4. ทดสอบ drift detection
# Manual change (ไม่ผ่าน Git)
kubectl scale deployment webapp -n development --replicas=1

# ArgoCD จะ detect drift และ restore
# ดู ArgoCD UI หรือ
argocd app get webapp

# 5. Rollback
git revert HEAD
git push
```

### Exercise 4: Image Update Automation

```yaml
# .github/workflows/image-update.yaml
name: Update Image in GitOps

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]

jobs:
  update-gitops:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    steps:
    - name: Get image tag
      id: tag
      run: |
        echo "IMAGE_TAG=${{ github.event.workflow_run.head_sha }}" >> $GITHUB_OUTPUT
        
    - name: Checkout config repo
      uses: actions/checkout@v4
      with:
        repository: myorg/config-repo
        token: ${{ secrets.CONFIG_PAT }}
        
    - name: Update image tag
      run: |
        cd overlays/development
        sed -i "s|image: nginx:.*|image: myapp:${{ steps.tag.outputs.IMAGE_TAG }}|" \
          ../../base/deployment.yaml
          
    - name: Commit and push
      run: |
        git config --global user.email "bot@myorg.com"
        git config --global user.name "GitOps Bot"
        git add .
        git commit -m "auto: update image to ${{ steps.tag.outputs.IMAGE_TAG }}"
        git push
```

---

## สรุป

GitOps เป็นแนวทางที่ทรงพลังสำหรับการจัดการ Kubernetes deployments โดย:

1. **Git เป็น single source of truth** - ทุกการเปลี่ยนแปลงต้องผ่าน Git
2. **Pull-based deployment** - ปลอดภัยกว่า push-based
3. **Declarative configuration** - ง่ายต่อการ review และ audit
4. **Automatic reconciliation** - ตรวจและแก้ drift โดยอัตโนมัติ
5. **Easy rollback** - แค่ `git revert`

### Next Steps
- ส่วน 42 จะเจาะลึก ArgoCD
- ส่วน 43 จะพูดถึง Flux CD
- ส่วน 44 จะพูดถึง Multi-Branch Pipeline Strategy

---

*หมายเหตุ: สำหรับ production environments ควรใช้ proper secret management เช่น Sealed Secrets หรือ External Secrets Operator เสมอ*

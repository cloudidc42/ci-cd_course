# Part 43: Flux CD — GitOps Toolkit สำหรับ Kubernetes

## สารบัญ
1. [Flux CD คืออะไร?](#flux-cd-คืออะไร)
2. [Flux v2 Architecture](#flux-v2-architecture)
3. [Bootstrap Flux](#bootstrap-flux)
4. [Core Flux CRDs](#core-flux-crds)
5. [Image Automation](#image-automation)
6. [Multi-Tenancy](#multi-tenancy)
7. [Notifications](#notifications)
8. [Exercises](#exercises)

---

## Flux CD คืออะไร?

Flux CD เป็น GitOps tool สำหรับ Kubernetes ที่พัฒนาโดย Weaveworks และเป็น CNCF Graduated project

### Flux vs ArgoCD

| ด้าน | Flux | ArgoCD |
|------|------|--------|
| UI | ไม่มี built-in (ใช้ Weave Gitops) | มี built-in UI ที่สวยงาม |
| Architecture | Microservices controllers | Monolithic |
| Multi-tenancy | ออกแบบมาเพื่อ multi-tenancy | AppProject |
| Helm | HelmRelease CRD | Application CRD |
| Kustomize | Kustomization CRD | Application CRD |
| Notifications | Alert/Provider CRDs | Notifications controller |
| CLI | flux CLI | argocd CLI |

---

## Flux v2 Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Flux v2 Components                        │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐  │
│  │  Source        │  │  Kustomize     │  │  Helm              │  │
│  │  Controller    │  │  Controller    │  │  Controller        │  │
│  │                │  │                │  │                    │  │
│  │ - GitRepository│  │ - Kustomization│  │ - HelmRelease      │  │
│  │ - HelmRepository│  │ - reconcile   │  │ - HelmChart        │  │
│  │ - Bucket       │  │   manifests    │  │ - reconcile        │  │
│  │ - OCIRepository│  │                │  │   releases         │  │
│  └────────────────┘  └────────────────┘  └────────────────────┘  │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐  │
│  │  Image         │  │  Notification  │  │  Image             │  │
│  │  Reflector     │  │  Controller    │  │  Automation        │  │
│  │  Controller    │  │                │  │  Controller        │  │
│  │                │  │ - Alert        │  │                    │  │
│  │ - ImageRepository  │ - Provider    │  │ - ImageUpdateAuto  │  │
│  │ - ImagePolicy  │  │                │  │ - policy-based     │  │
│  └────────────────┘  └────────────────┘  └────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### CRDs ของ Flux

```
Source Controller:
├── GitRepository    - ดึง manifests จาก Git
├── HelmRepository  - ดึง Helm charts จาก repo
├── HelmChart       - สร้าง chart artifact
├── Bucket          - ดึงจาก S3/GCS/Azure Blob
└── OCIRepository   - ดึงจาก OCI registry

Kustomize Controller:
└── Kustomization   - deploy ด้วย kustomize

Helm Controller:
└── HelmRelease     - deploy Helm charts

Image Controller:
├── ImageRepository  - scan container registry
├── ImagePolicy      - policy สำหรับ image selection
└── ImageUpdateAutomation - auto-update image tags ใน Git

Notification Controller:
├── Provider         - webhook/notification endpoint
├── Alert            - กำหนด events ที่จะ notify
└── Receiver         - inbound webhook receiver
```

---

## Bootstrap Flux

### ติดตั้ง Flux CLI

```bash
# macOS/Linux
curl -s https://fluxcd.io/install.sh | sudo bash

# หรือใช้ brew
brew install fluxcd/tap/flux

# ตรวจสอบ prerequisites
flux check --pre

# Output:
# ► checking prerequisites
# ✔ Kubernetes 1.28.0 >=1.20.6-0
# ✔ prerequisites checks passed
```

### Bootstrap กับ GitHub

```bash
# Export GitHub token
export GITHUB_TOKEN=ghp_xxx...

# Bootstrap Flux
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal   # ถ้าเป็น personal account

# Bootstrap สำหรับ organization
flux bootstrap github \
  --owner=myorg \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --team=platform-team \
  --read-write-key   # สร้าง deploy key แบบ read-write
```

### Bootstrap กับ GitLab

```bash
export GITLAB_TOKEN=glpat-xxx...

flux bootstrap gitlab \
  --owner=mygroup \
  --repository=fleet-infra \
  --branch=main \
  --path=./clusters/production \
  --token-auth
```

### Bootstrap กับ Generic Git

```bash
# สำหรับ self-hosted Git (Gitea, Bitbucket, etc.)
flux bootstrap git \
  --url=ssh://git@git.mycompany.com/myorg/fleet-infra \
  --branch=main \
  --path=clusters/production \
  --private-key-file=/path/to/id_rsa
```

### โครงสร้างหลังจาก Bootstrap

```
fleet-infra/
└── clusters/
    └── my-cluster/
        └── flux-system/
            ├── gotk-components.yaml    # Flux controllers
            ├── gotk-sync.yaml          # GitRepository + Kustomization
            └── kustomization.yaml      # Kustomize config
```

```yaml
# gotk-sync.yaml ที่ถูกสร้างโดย bootstrap
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: ssh://git@github.com/myorg/fleet-infra.git
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/my-cluster
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

---

## Core Flux CRDs

### GitRepository

```yaml
# ดึง manifests จาก Git repository
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: webapp-config
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/config-repo.git
  ref:
    branch: main
  secretRef:
    name: github-credentials
---
# Secret สำหรับ authentication
apiVersion: v1
kind: Secret
metadata:
  name: github-credentials
  namespace: flux-system
type: Opaque
stringData:
  username: myuser
  password: ghp_xxx...   # GitHub PAT
```

```yaml
# GitRepository พร้อม include/exclude paths
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: webapp-config
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/myorg/config-repo.git
  ref:
    branch: main
  ignore: |
    # ไม่สนใจ development environment
    /overlays/development/
    /docs/
    *.md
```

### Kustomization

```yaml
# Deploy โดยใช้ Kustomize
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: webapp-production
  namespace: flux-system
spec:
  interval: 10m
  path: ./overlays/production
  prune: true          # ลบ resources ที่ไม่อยู่ใน Git
  sourceRef:
    kind: GitRepository
    name: webapp-config
  
  # Health checks
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: webapp
    namespace: production
  
  # ลำดับการ apply (dependency)
  dependsOn:
  - name: infrastructure-production
  
  # Timeout
  timeout: 5m
  
  # Force recreate (สำหรับ debugging)
  force: false
  
  # Post-build substitution
  postBuild:
    substitute:
      cluster_region: ap-southeast-1
      cluster_env: production
    substituteFrom:
    - kind: ConfigMap
      name: cluster-vars
    - kind: Secret
      name: cluster-secrets
```

```yaml
# ตัวอย่าง manifest ที่ใช้ variable substitution
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  template:
    spec:
      containers:
      - name: webapp
        env:
        - name: REGION
          value: "${cluster_region}"
        - name: ENVIRONMENT
          value: "${cluster_env}"
```

### HelmRepository

```yaml
# Helm Repository (chart repository)
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: prometheus-community
  namespace: flux-system
spec:
  interval: 30m
  url: https://prometheus-community.github.io/helm-charts
---
# OCI Helm Repository
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  type: oci
  interval: 5m
  url: oci://ghcr.io/stefanprodan/charts
```

### HelmRelease

```yaml
# Deploy Helm chart ผ่าน Flux
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: prometheus-stack
  namespace: monitoring
spec:
  interval: 30m
  
  # Chart source
  chart:
    spec:
      chart: kube-prometheus-stack
      version: ">=56.0.0 <57.0.0"   # Semver constraint
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  
  # Values
  values:
    grafana:
      enabled: true
      adminPassword: "admin123"
      ingress:
        enabled: true
        hosts:
        - grafana.mycompany.com
    
    prometheus:
      prometheusSpec:
        retention: 30d
  
  # Values files จาก external sources
  valuesFrom:
  - kind: ConfigMap
    name: prometheus-values
    valuesKey: values.yaml
  - kind: Secret
    name: prometheus-secrets
    valuesKey: secret-values.yaml
    optional: true
  
  # Upgrade options
  upgrade:
    remediation:
      retries: 3
      strategy: rollback
  
  # Install options
  install:
    remediation:
      retries: 3
  
  # Test after deploy
  test:
    enable: true
    ignoreFailures: false
  
  # Dependency
  dependsOn:
  - name: cert-manager
    namespace: cert-manager
```

```yaml
# HelmRelease ที่ override values สำหรับ production
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: webapp
  namespace: production
spec:
  interval: 5m
  chart:
    spec:
      chart: webapp
      version: "1.x"
      sourceRef:
        kind: HelmRepository
        name: mycompany-charts
        namespace: flux-system
  
  values:
    replicaCount: 5
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 2000m
        memory: 1Gi
    
    autoscaling:
      enabled: true
      minReplicas: 5
      maxReplicas: 20
      targetCPUUtilizationPercentage: 70
    
    ingress:
      enabled: true
      className: nginx
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
      hosts:
      - host: app.mycompany.com
        paths:
        - path: /
          pathType: Prefix
      tls:
      - secretName: webapp-tls
        hosts:
        - app.mycompany.com
```

### Bucket (S3/GCS)

```yaml
# ดึง manifests จาก S3 bucket
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: Bucket
metadata:
  name: config-bucket
  namespace: flux-system
spec:
  interval: 5m
  provider: aws
  bucketName: my-gitops-config
  endpoint: s3.ap-southeast-1.amazonaws.com
  region: ap-southeast-1
  secretRef:
    name: aws-credentials
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: flux-system
type: Opaque
stringData:
  accesskey: AKIAIOSFODNN7EXAMPLE
  secretkey: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

---

## Image Automation

### Image Reflector

```yaml
# Scan container registry สำหรับ image tags
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: webapp
  namespace: flux-system
spec:
  image: ghcr.io/myorg/webapp
  interval: 1m
  secretRef:
    name: ghcr-credentials
---
apiVersion: v1
kind: Secret
metadata:
  name: ghcr-credentials
  namespace: flux-system
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <BASE64_DOCKER_CONFIG>
```

### Image Policy

```yaml
# Policy สำหรับ semver
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: webapp-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: webapp
  policy:
    semver:
      range: ">=1.0.0 <2.0.0"   # ใช้ 1.x.x
---
# Policy สำหรับ latest ที่ match regex
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: webapp-main-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: webapp
  filterTags:
    pattern: '^main-[a-fA-F0-9]+-(?P<ts>.*)'
    extract: '$ts'
  policy:
    numerical:
      order: asc   # เอา tag ที่ timestamp สูงสุด (ล่าสุด)
```

### Image Update Automation

```yaml
# Auto-update image tags ใน Git
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: webapp-auto-update
  namespace: flux-system
spec:
  interval: 30m
  
  # Source repo ที่จะ update
  sourceRef:
    kind: GitRepository
    name: webapp-config
  
  # Git commit/push config
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxcdbot@mycompany.com
        name: Flux Bot
      messageTemplate: |
        Auto-update: {{ range .Updated.Images }}{{ println .}}{{end}}
    push:
      branch: main
  
  # Update policy
  update:
    path: ./overlays/production
    strategy: Setters   # ใช้ marker comments ใน manifests
```

```yaml
# Deployment ที่มี marker comment สำหรับ Image Automation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  template:
    spec:
      containers:
      - name: webapp
        image: ghcr.io/myorg/webapp:1.0.0 # {"$imagepolicy": "flux-system:webapp-policy"}
```

---

## Multi-Tenancy

### Tenant Structure

```
fleet-infra/
├── clusters/
│   └── production/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   └── gotk-sync.yaml
│       └── tenants/
│           ├── team-frontend.yaml
│           ├── team-backend.yaml
│           └── team-data.yaml
├── tenants/
│   ├── base/
│   │   ├── namespace.yaml
│   │   ├── service-account.yaml
│   │   └── rbac.yaml
│   └── overlays/
│       ├── team-frontend/
│       │   ├── kustomization.yaml
│       │   └── patch.yaml
│       └── team-backend/
│           ├── kustomization.yaml
│           └── patch.yaml
```

### Tenant Onboarding

```yaml
# tenants/base/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: placeholder
---
# tenants/base/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flux
  namespace: placeholder
---
# tenants/base/rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: flux-reconciler
  namespace: placeholder
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: flux
  namespace: placeholder
```

```yaml
# clusters/production/tenants/team-frontend.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-frontend
  namespace: flux-system
spec:
  interval: 5m
  path: ./tenants/overlays/team-frontend
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  serviceAccountName: tenant-manager  # ใช้ restricted service account
```

```yaml
# tenants/overlays/team-frontend/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: frontend

bases:
- ../../base

patches:
- patch: |
    - op: replace
      path: /metadata/name
      value: frontend
  target:
    kind: Namespace
- patch: |
    - op: replace
      path: /metadata/namespace
      value: frontend
  target:
    kind: ServiceAccount
- patch: |
    - op: replace
      path: /metadata/namespace
      value: frontend
  target:
    kind: RoleBinding

resources:
- git-repository.yaml  # Team-specific GitRepository
- kustomization.yaml   # Team-specific app deployments
```

```yaml
# tenants/overlays/team-frontend/git-repository.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: team-frontend
  namespace: frontend
spec:
  interval: 1m
  url: https://github.com/myorg/frontend-config.git
  ref:
    branch: main
  secretRef:
    name: frontend-git-credentials
```

### Cross-Namespace SourceRef

```yaml
# อนุญาตให้ Kustomization ใน namespace อื่นใช้ GitRepository
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: frontend-app
  namespace: frontend
spec:
  interval: 5m
  path: ./production
  prune: true
  sourceRef:
    kind: GitRepository
    name: team-frontend
    namespace: frontend  # ต้องระบุ namespace
  serviceAccountName: flux   # ต้องมี permissions
```

---

## Notifications

### Provider Configuration

```yaml
# Slack Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: slack
  namespace: flux-system
spec:
  type: slack
  channel: "#deployments"
  secretRef:
    name: slack-url
---
apiVersion: v1
kind: Secret
metadata:
  name: slack-url
  namespace: flux-system
type: Opaque
stringData:
  address: https://hooks.slack.com/services/xxx/yyy/zzz
```

```yaml
# GitHub Commit Status Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: github-status
  namespace: flux-system
spec:
  type: github
  address: https://github.com/myorg/config-repo
  secretRef:
    name: github-token
---
apiVersion: v1
kind: Secret
metadata:
  name: github-token
  namespace: flux-system
type: Opaque
stringData:
  token: ghp_xxx...
```

```yaml
# PagerDuty Provider
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Provider
metadata:
  name: pagerduty
  namespace: flux-system
spec:
  type: pagerduty
  address: https://events.pagerduty.com
  secretRef:
    name: pagerduty-key
```

### Alert Configuration

```yaml
# Alert สำหรับ Kustomization failures
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: kustomization-alerts
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: error
  eventSources:
  - kind: Kustomization
    name: '*'    # ทุก Kustomization
  
  # Exclude events ที่ไม่สำคัญ
  exclusionList:
  - "no changes since last reconciliation"
  - "auth secret not found"
```

```yaml
# Alert สำหรับ HelmRelease
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: helm-alerts
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: error
  eventSources:
  - kind: HelmRelease
    name: '*'
    namespace: '*'
  summary: "Helm release issue in cluster"
```

```yaml
# Alert สำหรับทุก resources (info + error)
apiVersion: notification.toolkit.fluxcd.io/v1beta3
kind: Alert
metadata:
  name: all-deployments
  namespace: flux-system
spec:
  providerRef:
    name: slack
  eventSeverity: info   # info รวม error ด้วย
  eventSources:
  - kind: GitRepository
    name: '*'
  - kind: Kustomization
    name: '*'
  - kind: HelmRelease
    name: '*'
```

### Webhook Receiver

```yaml
# Receiver สำหรับรับ GitHub webhooks
apiVersion: notification.toolkit.fluxcd.io/v1
kind: Receiver
metadata:
  name: github-webhook
  namespace: flux-system
spec:
  type: github
  events:
  - "ping"
  - "push"
  secretRef:
    name: github-webhook-token
  resources:
  - apiVersion: source.toolkit.fluxcd.io/v1
    kind: GitRepository
    name: webapp-config
---
apiVersion: v1
kind: Secret
metadata:
  name: github-webhook-token
  namespace: flux-system
type: Opaque
stringData:
  token: my-webhook-secret
```

```bash
# ดู Receiver URL
kubectl -n flux-system get receiver github-webhook
# output: URL จะอยู่ใน status.webhookPath

# ตั้งค่า GitHub webhook
# Settings > Webhooks > Add webhook
# Payload URL: https://flux-webhook.mycompany.com/hook/xxx
# Content type: application/json
# Secret: my-webhook-secret
# Events: push, ping
```

---

## Flux CLI Commands

### ดู Status

```bash
# ดู flux components
flux check

# ดู sources ทั้งหมด
flux get sources all

# ดู kustomizations
flux get kustomizations

# ดู helmreleases
flux get helmreleases --all-namespaces

# ดู image automation
flux get image all
```

### Reconcile

```bash
# Force reconcile ทันที
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Reconcile พร้อม watching
flux reconcile kustomization webapp --with-source

# Suspend reconciliation
flux suspend kustomization webapp

# Resume reconciliation
flux resume kustomization webapp
```

### Logs

```bash
# ดู logs ของ source controller
flux logs --kind=GitRepository --name=flux-system

# ดู logs ทุก controllers
flux logs --all-namespaces --level=error

# Stream logs
flux logs -f --level=info
```

### Export

```bash
# Export GitRepository เป็น YAML
flux export source git webapp-config

# Export HelmRelease
flux export helmrelease webapp

# Export ทุกอย่าง
flux export source all > sources.yaml
flux export kustomization all > kustomizations.yaml
```

### Create Resources

```bash
# สร้าง GitRepository
flux create source git webapp-config \
  --url=https://github.com/myorg/config-repo \
  --branch=main \
  --interval=1m \
  --secret-ref=github-credentials

# สร้าง Kustomization
flux create kustomization webapp \
  --target-namespace=production \
  --source=webapp-config \
  --path=./overlays/production \
  --prune=true \
  --interval=10m

# สร้าง HelmRelease
flux create helmrelease webapp \
  --target-namespace=production \
  --source=HelmRepository/mycompany-charts \
  --chart=webapp \
  --chart-version=">=1.0.0 <2.0.0" \
  --interval=30m
```

---

## Exercises

### Exercise 1: Bootstrap Flux

```bash
#!/bin/bash
# exercise-1-bootstrap.sh

set -e

# ตั้งค่าตัวแปร
GITHUB_USER="myusername"
GITHUB_REPO="fleet-infra-exercise"
CLUSTER_PATH="clusters/development"

# ตรวจสอบ prerequisites
flux check --pre

# Bootstrap
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./$CLUSTER_PATH \
  --personal \
  --components-extra=image-reflector-controller,image-automation-controller

# ตรวจสอบ
flux check
kubectl get pods -n flux-system

echo "=== Bootstrap สำเร็จ! ==="
echo "Repository: https://github.com/$GITHUB_USER/$GITHUB_REPO"
```

### Exercise 2: Deploy แรกด้วย Flux

```bash
#!/bin/bash
# exercise-2-first-deploy.sh

REPO_DIR="/tmp/fleet-infra-exercise"

# Clone fleet-infra
git clone https://github.com/$GITHUB_USER/$GITHUB_REPO $REPO_DIR
cd $REPO_DIR

# สร้าง directory สำหรับ application
mkdir -p clusters/development/apps

# สร้าง GitRepository
cat > clusters/development/apps/source.yaml << 'EOF'
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 30s
  url: https://github.com/stefanprodan/podinfo
  ref:
    branch: master
EOF

# สร้าง Kustomization
cat > clusters/development/apps/kustomization.yaml << 'EOF'
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: podinfo
  namespace: flux-system
spec:
  interval: 5m0s
  path: ./kustomize
  prune: true
  sourceRef:
    kind: GitRepository
    name: podinfo
  targetNamespace: default
EOF

# Commit และ push
git add .
git commit -m "feat: add podinfo deployment"
git push

# รอ Flux sync
echo "รอ Flux reconcile..."
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# ตรวจสอบ
flux get all
kubectl get pods -n default
```

### Exercise 3: HelmRelease

```yaml
# exercise-3-helmrelease.yaml
# เพิ่มไฟล์นี้ใน clusters/development/monitoring/

---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: grafana
  namespace: flux-system
spec:
  interval: 30m
  url: https://grafana.github.io/helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: grafana
  namespace: monitoring
spec:
  interval: 30m
  chart:
    spec:
      chart: grafana
      version: ">=7.0.0 <8.0.0"
      sourceRef:
        kind: HelmRepository
        name: grafana
        namespace: flux-system
  values:
    adminPassword: "admin"
    service:
      type: ClusterIP
    ingress:
      enabled: false
  install:
    createNamespace: true
  upgrade:
    remediation:
      retries: 3
```

### Exercise 4: Image Automation

```bash
#!/bin/bash
# exercise-4-image-automation.sh

cd /tmp/fleet-infra-exercise

# สร้าง ImageRepository
cat > clusters/development/apps/image-repository.yaml << 'EOF'
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: podinfo
  namespace: flux-system
spec:
  image: ghcr.io/stefanprodan/podinfo
  interval: 1m0s
EOF

# สร้าง ImagePolicy (ใช้ semver)
cat > clusters/development/apps/image-policy.yaml << 'EOF'
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: podinfo
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo
  policy:
    semver:
      range: ">=6.0.0"
EOF

# สร้าง ImageUpdateAutomation
cat > clusters/development/apps/image-update.yaml << 'EOF'
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: podinfo-auto-update
  namespace: flux-system
spec:
  interval: 1m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: fluxbot@mycompany.com
        name: Flux Bot
      messageTemplate: "auto: update image tags"
    push:
      branch: main
  update:
    path: ./clusters/development/apps
    strategy: Setters
EOF

git add .
git commit -m "feat: add image automation"
git push

# Force reconcile
flux reconcile source git flux-system
flux reconcile image repository podinfo
flux reconcile image policy podinfo

# ดูผลลัพธ์
flux get image all
```

### Exercise 5: Multi-Tenant Setup

```bash
#!/bin/bash
# exercise-5-multi-tenant.sh

cd /tmp/fleet-infra-exercise

# สร้าง tenant structure
mkdir -p {tenants/base,tenants/overlays/{team-alpha,team-beta}}

# Base tenant resources
cat > tenants/base/namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: placeholder
EOF

cat > tenants/base/service-account.yaml << 'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flux
  namespace: placeholder
EOF

cat > tenants/base/role-binding.yaml << 'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: flux-reconciler
  namespace: placeholder
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: ServiceAccount
  name: flux
  namespace: placeholder
EOF

cat > tenants/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- namespace.yaml
- service-account.yaml
- role-binding.yaml
EOF

# Team Alpha overlay
cat > tenants/overlays/team-alpha/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: team-alpha
bases:
- ../../base
namePrefix: team-alpha-
patches:
- patch: |
    - op: replace
      path: /metadata/name
      value: team-alpha
  target:
    kind: Namespace
EOF

# Commit
git add .
git commit -m "feat: add multi-tenant structure"
git push
```

---

## Flux vs ArgoCD Decision Guide

```
เลือก Flux เมื่อ:
✅ ต้องการ multi-tenancy ที่ดีกว่า
✅ Team ชอบ CLI-first approach
✅ ต้องการ image automation ที่ built-in
✅ ต้องการ integration กับ Helm ที่ดี
✅ ต้องการ lightweight controller design

เลือก ArgoCD เมื่อ:
✅ ต้องการ Web UI ที่สวยงาม
✅ Team ใหม่ต้องการ visualization
✅ ต้องการ SSO/RBAC ที่ครบครัน
✅ Multi-cluster management
✅ Advanced sync policies (waves, hooks)
```

---

## สรุป

Flux CD v2 เป็น GitOps toolkit ที่ทรงพลังด้วย:

1. **Modular Architecture** - แยก controllers ตาม responsibility
2. **GitRepository/Kustomization** - deploy manifests จาก Git
3. **HelmRelease** - จัดการ Helm releases
4. **Image Automation** - auto-update image tags
5. **Multi-Tenancy** - รองรับหลาย teams ใน cluster เดียว
6. **Notifications** - แจ้งเตือนผ่าน Slack, PagerDuty, etc.

---

*ส่วนต่อไป: Part 44 - Multi-Branch Pipeline Strategy*

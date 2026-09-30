# Part 42: ArgoCD — ติดตั้งและใช้งาน GitOps Operator

## สารบัญ
1. [ArgoCD คืออะไร?](#argocd-คืออะไร)
2. [Architecture](#architecture)
3. [การติดตั้ง ArgoCD](#การติดตั้ง-argocd)
4. [Application CRD](#application-crd)
5. [Sync Policies](#sync-policies)
6. [Health Checks](#health-checks)
7. [Rollback](#rollback)
8. [Multi-Cluster](#multi-cluster)
9. [SSO Integration](#sso-integration)
10. [ArgoCD CLI](#argocd-cli)
11. [Notifications](#notifications)
12. [Image Updater](#image-updater)
13. [Exercises](#exercises)

---

## ArgoCD คืออะไร?

ArgoCD (Argo Continuous Delivery) เป็น declarative, GitOps continuous delivery tool สำหรับ Kubernetes ที่ได้รับความนิยมสูงสุดในปัจจุบัน

### จุดเด่นของ ArgoCD
- **Real-time sync**: monitor Git repository และ sync ไปยัง cluster อัตโนมัติ
- **Web UI ที่สวยงาม**: visualize application และ resource tree
- **Multi-cluster support**: จัดการหลาย Kubernetes clusters จาก UI เดียว
- **Multiple config formats**: รองรับ Kustomize, Helm, Jsonnet, plain YAML
- **RBAC**: Role-Based Access Control ที่ละเอียด
- **SSO**: รองรับ OIDC, SAML, GitHub, GitLab, Google

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         ArgoCD                                    │
│                                                                   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌──────────────┐  │
│  │   API Server    │    │  Repo Server    │    │    App       │  │
│  │                 │    │                 │    │ Controller   │  │
│  │ - REST/gRPC API │    │ - Clone repos   │    │              │  │
│  │ - WebSocket     │    │ - Generate      │    │ - Reconcile  │  │
│  │ - Auth          │    │   manifests     │    │ - Detect     │  │
│  │ - RBAC          │    │ - Cache         │    │   drift      │  │
│  └────────┬────────┘    └────────┬────────┘    └──────┬───────┘  │
│           │                      │                    │           │
│           └──────────────────────┴────────────────────┘          │
│                                  │                               │
│                         ┌────────▼────────┐                      │
│                         │  Redis Cache    │                      │
│                         └─────────────────┘                      │
│                                                                   │
│  ┌─────────────────┐    ┌─────────────────┐                      │
│  │  Dex (OIDC)     │    │  Notifications  │                      │
│  │                 │    │  Controller     │                      │
│  └─────────────────┘    └─────────────────┘                      │
└──────────────────────────────────────────────────────────────────┘
        │                                          │
        ▼                                          ▼
   Git Repository                         Kubernetes Cluster(s)
```

### Components
- **API Server**: กลาง hub สำหรับ CLI, UI และ CI/CD systems
- **Repository Server**: cache และ generate Kubernetes manifests จาก Git
- **Application Controller**: reconcile actual vs desired state
- **Dex**: built-in OIDC provider สำหรับ SSO
- **Redis**: cache สำหรับ repository และ application state

---

## การติดตั้ง ArgoCD

### วิธีที่ 1: Standard Installation

```bash
# สร้าง namespace
kubectl create namespace argocd

# ติดตั้ง ArgoCD (stable version)
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ตรวจสอบ pods
kubectl get pods -n argocd -w

# Output ที่ต้องการ:
# argocd-application-controller-0       1/1     Running
# argocd-dex-server-xxx                 1/1     Running
# argocd-notifications-controller-xxx   1/1     Running
# argocd-redis-xxx                      1/1     Running
# argocd-repo-server-xxx                1/1     Running
# argocd-server-xxx                     1/1     Running
```

### วิธีที่ 2: Helm Installation

```bash
# เพิ่ม Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# สร้าง values.yaml
cat > argocd-values.yaml << 'EOF'
global:
  domain: argocd.mycompany.com

server:
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
    hosts:
    - argocd.mycompany.com
    tls:
    - secretName: argocd-tls
      hosts:
      - argocd.mycompany.com

configs:
  params:
    server.insecure: false
  
  cm:
    # เปิด status badge
    statusbadge.enabled: "true"
    
    # Resource tracking method
    application.resourceTrackingMethod: annotation

redis-ha:
  enabled: true   # HA สำหรับ production

controller:
  replicas: 2

server:
  replicas: 2

repoServer:
  replicas: 2
EOF

# ติดตั้ง
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values argocd-values.yaml \
  --version 6.7.0
```

### วิธีที่ 3: HA Installation (Production)

```yaml
# argocd-ha-install.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/ha/install.yaml

patches:
- path: replica-patch.yaml
- path: resources-patch.yaml

configMapGenerator:
- name: argocd-cm
  behavior: merge
  literals:
  - application.resourceTrackingMethod=annotation
  - resource.exclusions=|
      - apiGroups:
        - cilium.io
        kinds:
        - CiliumIdentity
        clusters:
        - "*"
```

```yaml
# replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
  namespace: argocd
spec:
  replicas: 3
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  replicas: 3
```

### Access ArgoCD UI

```bash
# วิธีที่ 1: Port-forward (development)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# วิธีที่ 2: LoadBalancer
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

# วิธีที่ 3: NodePort
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'

# ดู initial password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""

# Login ด้วย CLI
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure

# เปลี่ยน password
argocd account update-password \
  --current-password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --new-password MyNewSecurePassword123!
```

---

## Application CRD

Application เป็น Custom Resource หลักของ ArgoCD

### Basic Application

```yaml
# webapp-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
  labels:
    team: platform
    environment: production
  finalizers:
  - resources-finalizer.argocd.argoproj.io   # ลบ resources เมื่อ app ถูกลบ
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/config-repo.git
    targetRevision: HEAD          # branch หรือ tag
    path: overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - Validate=true
    - ApplyOutOfSyncOnly=true
```

### Application with Helm

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-stack
  namespace: argocd
spec:
  project: monitoring
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    targetRevision: 56.6.0
    chart: kube-prometheus-stack
    helm:
      releaseName: prometheus
      values: |
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
            storageSpec:
              volumeClaimTemplate:
                spec:
                  storageClassName: fast-ssd
                  resources:
                    requests:
                      storage: 100Gi
        
        alertmanager:
          enabled: true
          config:
            global:
              slack_api_url: 'https://hooks.slack.com/services/xxx'
            route:
              receiver: 'slack'
            receivers:
            - name: 'slack'
              slack_configs:
              - channel: '#alerts'
  
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    - ServerSideApply=true   # ใช้ Server-Side Apply สำหรับ large resources
```

### Application with Multiple Sources

```yaml
# ArgoCD 2.6+ รองรับ multiple sources
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp-with-secrets
  namespace: argocd
spec:
  project: default
  
  sources:
  # Source 1: Helm chart
  - repoURL: https://charts.mycompany.com
    targetRevision: 1.2.3
    chart: webapp
    helm:
      valueFiles:
      - $values/environments/production/values.yaml
  
  # Source 2: Values ใน Git
  - repoURL: https://github.com/myorg/config-repo.git
    targetRevision: HEAD
    ref: values   # อ้างอิงด้วย ref name
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

### AppProject (RBAC)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: production
  namespace: argocd
spec:
  description: Production environment project
  
  # Repositories ที่ project นี้ใช้ได้
  sourceRepos:
  - https://github.com/myorg/config-repo.git
  - https://charts.mycompany.com
  
  # Clusters/Namespaces ที่ deploy ได้
  destinations:
  - server: https://k8s-prod.mycompany.com
    namespace: production
  - server: https://k8s-prod.mycompany.com
    namespace: production-*
  
  # Kubernetes resource ที่อนุญาต
  namespaceResourceWhitelist:
  - group: apps
    kind: Deployment
  - group: ""
    kind: Service
  - group: ""
    kind: ConfigMap
  - group: networking.k8s.io
    kind: Ingress
  
  # Cluster-scoped resources ที่อนุญาต
  clusterResourceBlacklist:
  - group: ""
    kind: Namespace   # ไม่อนุญาตสร้าง Namespace
  - group: rbac.authorization.k8s.io
    kind: ClusterRole
  
  # RBAC rules
  roles:
  - name: developer
    description: Developer read-only access
    policies:
    - p, proj:production:developer, applications, get, production/*, allow
    - p, proj:production:developer, applications, sync, production/*, allow
    groups:
    - myorg:developers
  
  - name: admin
    description: Production admin
    policies:
    - p, proj:production:admin, applications, *, production/*, allow
    groups:
    - myorg:platform-team
  
  # Orphaned resource monitoring
  orphanedResources:
    warn: true
    ignore:
    - group: ""
      kind: ConfigMap
      name: kube-root-ca.crt
```

### ApplicationSet

```yaml
# สร้าง applications สำหรับหลาย environments โดยอัตโนมัติ
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: webapp-environments
  namespace: argocd
spec:
  generators:
  # Generator 1: List
  - list:
      elements:
      - environment: development
        namespace: development
        replicas: "1"
        imageTag: "latest"
      - environment: staging
        namespace: staging
        replicas: "2"
        imageTag: "1.2.3"
      - environment: production
        namespace: production
        replicas: "5"
        imageTag: "1.2.2"
  
  # Generator 2: Git Directory (สร้าง app ต่อ directory)
  # - git:
  #     repoURL: https://github.com/myorg/config-repo.git
  #     revision: HEAD
  #     directories:
  #     - path: overlays/*
  
  template:
    metadata:
      name: 'webapp-{{environment}}'
      labels:
        environment: '{{environment}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/config-repo.git
        targetRevision: HEAD
        path: 'overlays/{{environment}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

---

## Sync Policies

### Manual Sync

```yaml
spec:
  syncPolicy: {}  # ไม่มี automated sync
```

```bash
# Manual sync ผ่าน CLI
argocd app sync webapp

# Sync specific resources
argocd app sync webapp --resource apps:Deployment:webapp

# Sync พร้อม prune
argocd app sync webapp --prune

# Dry run
argocd app sync webapp --dry-run
```

### Automated Sync

```yaml
spec:
  syncPolicy:
    automated:
      prune: true       # ลบ resources ที่ไม่อยู่ใน Git
      selfHeal: true    # แก้ manual changes อัตโนมัติ
      allowEmpty: false # ป้องกัน sync เมื่อ source ว่าง
```

### Sync Waves และ Hooks

```yaml
# Sync waves ควบคุมลำดับการ sync
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  annotations:
    argocd.argoproj.io/sync-wave: "-1"  # Deploy ก่อน (wave -1)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # Deploy ตามปกติ (wave 0)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # Deploy หลังสุด (wave 1)
```

```yaml
# Pre-sync hook: ทำงานก่อน sync
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:1.2.3
        command: ["./migrate.sh"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
      restartPolicy: Never
```

```yaml
# Post-sync hook: ทำงานหลัง sync สำเร็จ
apiVersion: batch/v1
kind: Job
metadata:
  name: smoke-test
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: smoke-test
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          curl -f http://webapp/health || exit 1
          echo "Smoke test passed!"
      restartPolicy: Never
```

```yaml
# Sync-fail hook: ทำงานเมื่อ sync ล้มเหลว
apiVersion: batch/v1
kind: Job
metadata:
  name: notify-failure
  annotations:
    argocd.argoproj.io/hook: SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: notify
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          curl -X POST $SLACK_WEBHOOK \
            -H 'Content-type: application/json' \
            -d '{"text":"Deployment failed! Please check ArgoCD"}'
        env:
        - name: SLACK_WEBHOOK
          valueFrom:
            secretKeyRef:
              name: slack-secret
              key: webhook-url
      restartPolicy: Never
```

---

## Health Checks

ArgoCD ตรวจสอบ health ของ resources ตาม built-in checks หรือ custom checks

### Built-in Health Checks

```yaml
# Deployment: Healthy เมื่อ desired replicas == available replicas
# Service: Healthy เสมอ (ถ้า type LoadBalancer รอ IP)
# Ingress: Healthy เมื่อมี address
# PVC: Healthy เมื่อ Bound
# StatefulSet: Healthy เมื่อ ready replicas == desired
```

### Custom Health Check

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  # Custom health check สำหรับ custom resource
  resource.customizations.health.myapp.io_MyCustomResource: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.phase == "Running" then
        hs.status = "Healthy"
        hs.message = "Application is running"
      elseif obj.status.phase == "Failed" then
        hs.status = "Degraded"
        hs.message = obj.status.message
      else
        hs.status = "Progressing"
        hs.message = "Waiting for application to start"
      end
    else
      hs.status = "Progressing"
      hs.message = "Waiting for status"
    end
    return hs
  
  # Resource exclusions (resources ที่ไม่ต้องการ track)
  resource.exclusions: |
    - apiGroups:
      - cilium.io
      kinds:
      - CiliumIdentity
      clusters:
      - "*"
```

### Ignore Differences

```yaml
spec:
  ignoreDifferences:
  # Ignore Deployment replicas (ถ้า HPA จัดการ)
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas
  
  # Ignore ConfigMap data changes (จาก external source)
  - group: ""
    kind: ConfigMap
    name: app-config
    jsonPointers:
    - /data/RUNTIME_CONFIG
  
  # Ignore annotation ที่ถูกเพิ่มโดย admission webhook
  - group: ""
    kind: Service
    jsonPointers:
    - /metadata/annotations/kubectl.kubernetes.io~1last-applied-configuration
```

---

## Rollback

### Manual Rollback

```bash
# ดู history
argocd app history webapp

# Output:
# ID  DATE                           REVISION
# 0   2024-01-15 10:00:00 +0000 UTC  abc123
# 1   2024-01-15 11:00:00 +0000 UTC  def456
# 2   2024-01-15 12:00:00 +0000 UTC  ghi789

# Rollback ไปยัง revision ก่อนหน้า
argocd app rollback webapp 1

# ตรวจสอบ
argocd app get webapp
```

### Rollback ผ่าน Git

```bash
# วิธีที่ดีกว่า: rollback ผ่าน Git
git log --oneline overlays/production/

# Revert commit ล่าสุด
git revert HEAD
git push

# หรือ revert ไปยัง specific commit
git revert abc123..HEAD
git push

# ArgoCD จะ detect การเปลี่ยนแปลงและ sync อัตโนมัติ
```

### Automated Rollback ด้วย Analysis

```yaml
# ใช้ Argo Rollouts สำหรับ automated rollback
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: webapp
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 2m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 100
  
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
        image: myapp:1.2.3
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.95   # 95% success rate
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{
            app="webapp",
            status!~"5.."
          }[5m])) /
          sum(rate(http_requests_total{
            app="webapp"
          }[5m]))
```

---

## Multi-Cluster

### Register External Cluster

```bash
# ดู clusters ปัจจุบัน
argocd cluster list

# เพิ่ม cluster ใหม่
# ต้องมี kubeconfig ของ cluster นั้น
argocd cluster add production-cluster \
  --name production \
  --kubeconfig /path/to/kubeconfig

# ตรวจสอบ
argocd cluster list

# Output:
# SERVER                                  NAME        VERSION  STATUS
# https://k8s-prod.mycompany.com          production  1.28     Successful
# https://k8s-staging.mycompany.com       staging     1.28     Successful
# https://kubernetes.default.svc          in-cluster  1.28     Successful
```

### Multi-Cluster ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: webapp-multi-cluster
  namespace: argocd
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          environment: production
  
  template:
    metadata:
      name: 'webapp-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/config-repo.git
        targetRevision: HEAD
        path: overlays/production
      destination:
        server: '{{server}}'
        namespace: production
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Cluster Secret สำหรับ Multi-Cluster

```yaml
# Cluster registration secret
apiVersion: v1
kind: Secret
metadata:
  name: production-cluster
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
    environment: production
    region: ap-southeast-1
type: Opaque
stringData:
  name: production
  server: https://k8s-prod.mycompany.com
  config: |
    {
      "bearerToken": "<TOKEN>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<BASE64_CA_DATA>"
      }
    }
```

---

## SSO Integration

### GitHub OAuth

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.mycompany.com
  
  dex.config: |
    connectors:
    - type: github
      id: github
      name: GitHub
      config:
        clientID: $dex-github-client-id
        clientSecret: $dex-github-client-secret
        redirectURI: https://argocd.mycompany.com/api/dex/callback
        orgs:
        - name: myorg
          teams:
          - platform-team
          - developers
```

### RBAC สำหรับ SSO

```yaml
# argocd-rbac-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  
  policy.csv: |
    # Platform team: full admin
    g, myorg:platform-team, role:admin
    
    # Developers: read all, sync non-production
    g, myorg:developers, role:developer
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, development/*, allow
    p, role:developer, applications, sync, staging/*, allow
    p, role:developer, logs, get, */*, allow
    
    # QA team: sync staging only
    g, myorg:qa-team, role:qa
    p, role:qa, applications, get, */*, allow
    p, role:qa, applications, sync, staging/*, allow
```

### Okta SAML Integration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.mycompany.com
  
  dex.config: |
    connectors:
    - type: saml
      id: okta
      name: Okta
      config:
        ssoURL: https://mycompany.okta.com/app/argo/sso/saml
        caData: |
          <BASE64_ENCODED_CERTIFICATE>
        redirectURI: https://argocd.mycompany.com/api/dex/callback
        usernameAttr: email
        emailAttr: email
        groupsAttr: groups
```

---

## ArgoCD CLI

### Application Management

```bash
# ดู applications ทั้งหมด
argocd app list

# ดู details ของ application
argocd app get webapp

# ดู application tree
argocd app get webapp --tree

# Sync application
argocd app sync webapp

# Sync พร้อม options
argocd app sync webapp \
  --prune \
  --force \
  --timeout 300

# Sync แบบ dry-run
argocd app sync webapp --dry-run

# ดู diff
argocd app diff webapp

# ดู logs ของ pod
argocd app logs webapp --pod webapp-xxx

# Wait จนกว่า sync จะเสร็จ
argocd app wait webapp --sync --health --timeout 300
```

### Repository Management

```bash
# เพิ่ม Git repository
argocd repo add https://github.com/myorg/config-repo.git \
  --username myuser \
  --password $GITHUB_TOKEN

# เพิ่ม SSH repository
argocd repo add git@github.com:myorg/config-repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa

# ดู repositories
argocd repo list

# ลบ repository
argocd repo rm https://github.com/myorg/config-repo.git
```

### Account Management

```bash
# ดู accounts
argocd account list

# สร้าง user token (API access)
argocd account generate-token --account ci-bot

# อัปเดต password
argocd account update-password

# ดู current user
argocd account get-user-info
```

### Cluster Management

```bash
# ดู clusters
argocd cluster list

# เพิ่ม cluster
argocd cluster add my-cluster \
  --name production

# ดู cluster info
argocd cluster get https://k8s-prod.mycompany.com

# ลบ cluster
argocd cluster rm https://k8s-prod.mycompany.com
```

---

## Notifications

### ติดตั้ง Notifications Controller

```bash
# Notifications มาพร้อมกับ ArgoCD ตั้งแต่ v2.3
# แต่ต้อง configure เพิ่มเติม
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.2/manifests/install.yaml
```

### Slack Notifications

```yaml
# argocd-notifications-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Slack service
  service.slack: |
    token: $slack-token
  
  # Templates
  template.app-deployed: |
    email:
      subject: Application {{.app.metadata.name}} deployed
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [{
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          }, {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          }, {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }]
        }]
  
  template.app-sync-failed: |
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#E96D76",
          "fields": [{
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          }, {
            "title": "Error",
            "value": "{{.app.status.operationState.message}}",
            "short": false
          }]
        }]
  
  # Triggers
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
  
  trigger.on-sync-failed: |
    - description: Application sync failed
      send:
      - app-sync-failed
      when: app.status.operationState.phase in ['Error', 'Failed']
  
  # Default subscriptions
  subscriptions: |
    - recipients:
      - slack:deployments
      triggers:
      - on-deployed
      - on-sync-failed
```

```yaml
# Notifications secret
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  slack-token: xoxb-xxx-xxx-xxx   # Slack Bot Token
```

### PagerDuty Integration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.pagerduty: |
    token: $pagerduty-token
  
  template.app-degraded: |
    pagerduty:
      summary: "Application {{.app.metadata.name}} is degraded"
      severity: critical
      source: ArgoCD
      component: "{{.app.metadata.name}}"
      details: |
        Application: {{.app.metadata.name}}
        Health: {{.app.status.health.status}}
        Message: {{.app.status.health.message}}
  
  trigger.on-health-degraded: |
    - description: Application is degraded
      send:
      - app-degraded
      when: app.status.health.status == 'Degraded'
```

### Application-level Notifications

```yaml
# Application metadata สำหรับ notifications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: deployments
    notifications.argoproj.io/subscribe.on-sync-failed.slack: alerts
    notifications.argoproj.io/subscribe.on-health-degraded.pagerduty: ""
```

---

## Image Updater

ArgoCD Image Updater อัพเดต image tags ใน Git repositories โดยอัตโนมัติ

### ติดตั้ง Image Updater

```bash
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

### Configuration

```yaml
# สำหรับ Application ที่ใช้ Kustomize
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
  annotations:
    # บอก Image Updater ว่า container ไหนต้องดูแล
    argocd-image-updater.argoproj.io/image-list: |
      webapp=myregistry/webapp

    # Update strategy
    argocd-image-updater.argoproj.io/webapp.update-strategy: semver
    
    # Tag constraint
    argocd-image-updater.argoproj.io/webapp.tag-constraint: ">=1.0.0 <2.0.0"
    
    # Write back method
    argocd-image-updater.argoproj.io/write-back-method: git
    
    # Git branch สำหรับ write back
    argocd-image-updater.argoproj.io/write-back-target: "kustomization"
    
    # Git repository credential
    argocd-image-updater.argoproj.io/git-branch: main
```

```yaml
# Image Updater ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  registries.conf: |
    registries:
    - name: Docker Hub
      api_url: https://registry-1.docker.io
      prefix: docker.io
      ping: yes
      credentials: secret:argocd/dockerhub-creds#creds
      
    - name: GitHub Container Registry
      api_url: https://ghcr.io
      prefix: ghcr.io
      ping: no
      credentials: secret:argocd/ghcr-creds#creds
```

### Latest Tag Strategy

```yaml
annotations:
  # อัพเดตเป็น latest เสมอ
  argocd-image-updater.argoproj.io/webapp.update-strategy: latest
  
  # Filter tags ด้วย regex
  argocd-image-updater.argoproj.io/webapp.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
  
  # Ignore tags
  argocd-image-updater.argoproj.io/webapp.ignore-tags: latest, dev, test
```

---

## Exercises

### Exercise 1: ติดตั้งและ Configure ArgoCD

```bash
#!/bin/bash
# exercise-1-setup.sh

set -e

echo "=== ติดตั้ง ArgoCD ==="

# สร้าง namespace
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -

# ติดตั้ง ArgoCD
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

echo "รอให้ pods ready..."
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd \
  --timeout=300s

# ดึง initial password
ARGOCD_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

echo "=== ArgoCD ติดตั้งสำเร็จ ==="
echo "URL: https://localhost:8080"
echo "Username: admin"
echo "Password: $ARGOCD_PASS"
echo ""
echo "เปิด port-forward ด้วย:"
echo "kubectl port-forward svc/argocd-server -n argocd 8080:443"
```

### Exercise 2: สร้าง Application ใน ArgoCD

```bash
#!/bin/bash
# exercise-2-create-app.sh

# Login
argocd login localhost:8080 \
  --username admin \
  --password $ARGOCD_PASS \
  --insecure

# สร้าง Application
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy auto \
  --auto-prune \
  --self-heal

# ตรวจสอบ
argocd app get guestbook

# รอ sync
argocd app wait guestbook --sync --health --timeout 120

echo "=== Guestbook deployed! ==="
kubectl get pods -n default
```

### Exercise 3: ทดสอบ GitOps Flow

```bash
#!/bin/bash
# exercise-3-gitops-flow.sh

# สมมติว่าเรามี config repo อยู่แล้ว
REPO_URL="https://github.com/myorg/argocd-config"
REPO_PATH="/tmp/argocd-config"

# Clone config repo
git clone $REPO_URL $REPO_PATH
cd $REPO_PATH

# แก้ไข replica count
cat > overlays/development/replica-patch.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
EOF

# Commit และ push
git add . 
git commit -m "scale: increase replicas to 3 in development"
git push

# ดู ArgoCD sync
echo "รอ ArgoCD sync..."
sleep 30

argocd app get webapp-dev
kubectl get pods -n development

echo "=== Scaling สำเร็จ! ==="
```

### Exercise 4: Sync Waves

```yaml
# exercise-4-sync-waves.yaml
# ทดสอบ sync waves ด้วย database และ application

apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:16
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          value: myapp
        - name: POSTGRES_PASSWORD
          value: secret
---
# Migration Job (wave -1: หลัง DB แต่ก่อน app)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: Sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: flyway:10
        command: ["flyway", "migrate"]
        env:
        - name: FLYWAY_URL
          value: "jdbc:postgresql://postgresql:5432/myapp"
      restartPolicy: Never
---
# Application (wave 0: default)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  replicas: 3
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
        image: myapp:1.0.0
```

### Exercise 5: Setup Notifications

```bash
#!/bin/bash
# exercise-5-notifications.sh

# สร้าง Slack secret
kubectl create secret generic argocd-notifications-secret \
  -n argocd \
  --from-literal=slack-token=xoxb-YOUR-SLACK-TOKEN \
  --dry-run=client -o yaml | kubectl apply -f -

# Apply notifications configmap
cat > notifications-config.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  
  template.my-custom-template: |
    slack:
      attachments: |
        [{
          "title": "{{.app.metadata.name}} - {{.app.status.operationState.phase}}",
          "color": "{{if eq .app.status.operationState.phase "Succeeded"}}#18be52{{else}}#E96D76{{end}}"
        }]
  
  trigger.on-deployed: |
    - send: [my-custom-template]
      when: app.status.operationState.phase in ['Succeeded']
  
  subscriptions: |
    - recipients:
      - slack:#deployments
      triggers:
      - on-deployed
EOF

kubectl apply -f notifications-config.yaml

# ทดสอบด้วยการ trigger sync
argocd app sync guestbook
```

---

## สรุป

ArgoCD เป็น GitOps operator ที่มีความสามารถครบครัน:

1. **Application CRD** - declarative ทุกอย่างใน Kubernetes
2. **Sync Policies** - auto/manual พร้อม waves และ hooks
3. **Health Checks** - built-in และ custom
4. **Rollback** - ง่ายทั้งผ่าน CLI และ Git
5. **Multi-Cluster** - จัดการหลาย clusters จาก UI เดียว
6. **SSO** - รองรับ GitHub, GitLab, Okta, etc.
7. **Notifications** - Slack, PagerDuty, Email
8. **Image Updater** - อัพเดต image tags อัตโนมัติ

ArgoCD เป็นหนึ่งใน CNCF Graduated projects และเป็นมาตรฐานสำหรับ GitOps บน Kubernetes

---

*ส่วนต่อไป: Part 43 - Flux CD*

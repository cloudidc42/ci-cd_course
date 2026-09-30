# Part 86: Advanced GitOps Patterns

## บทนำ

GitOps ไม่ใช่แค่ "ใช้ Git เป็น single source of truth" แต่เป็น operational framework ที่สมบูรณ์ บทนี้จะพาไปสู่ advanced patterns ที่ช่วยให้จัดการ multi-cluster environments, databases, และการทำ progressive delivery แบบ GitOps-native ได้อย่างมืออาชีพ

## สารบัญ

1. [GitOps Fundamentals Revisited](#fundamentals)
2. [Multi-Cluster GitOps](#multi-cluster)
3. [Progressive Delivery กับ GitOps](#progressive-delivery)
4. [GitOps กับ Databases](#databases)
5. [Drift Detection และ Reconciliation](#drift)
6. [Secrets Management ใน GitOps](#secrets)
7. [GitOps Security](#security)
8. [Case Studies](#case-studies)
9. [แบบฝึกหัด](#exercises)

---

## 1. GitOps Fundamentals Revisited {#fundamentals}

### GitOps Principles

```
4 GitOps Principles (OpenGitOps):

1. DECLARATIVE
   - ทุก desired state เขียนเป็น declarative format
   - ไม่ใช่ imperative commands

2. VERSIONED AND IMMUTABLE
   - Git เป็น source of truth
   - Immutable artifacts (container images, Helm charts)

3. PULLED AUTOMATICALLY
   - Agent ใน cluster pull changes จาก Git
   - ไม่ใช่ push จากนอก cluster

4. CONTINUOUSLY RECONCILED
   - Agent ตรวจสอบและ reconcile ตลอดเวลา
   - Automatic drift correction
```

### GitOps Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Git Repository                       │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │         Desired State (Kubernetes manifests)    │    │
│  │  apps/                                          │    │
│  │  ├── payment-service/                           │    │
│  │  │   ├── deployment.yaml                        │    │
│  │  │   ├── service.yaml                           │    │
│  │  │   └── ingress.yaml                           │    │
│  │  └── order-service/                             │    │
│  │      └── ...                                    │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
          │ git pull (periodic)
          ▼
┌─────────────────────────────────────────────────────────┐
│                 Kubernetes Cluster                        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              ArgoCD / Flux                       │    │
│  │  - Watches Git repository                       │    │
│  │  - Compares desired vs actual state             │    │
│  │  - Reconciles differences                       │    │
│  └──────────────────────┬──────────────────────────┘    │
│                         │ applies                        │
│  ┌──────────────────────▼──────────────────────────┐    │
│  │              Actual State                        │    │
│  │  (Running deployments, services, etc.)           │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 2. Multi-Cluster GitOps {#multi-cluster}

### Multi-Cluster Challenges

```
ปัญหาของ Multi-Cluster GitOps:

1. Configuration sprawl
   - 10 clusters × 50 apps = 500 configurations ที่ต้อง manage

2. Consistency
   - ทำอย่างไรให้ทุก cluster มี config ที่ถูกต้อง?

3. Environment promotion
   - dev → staging → production ข้าม clusters

4. Secret management
   - แต่ละ cluster ต้องการ secrets ของตัวเอง

5. Audit trail
   - รู้ว่า cluster ไหน deploy อะไรเมื่อไหร่
```

### Hub-Spoke GitOps Architecture

```
ใช้ ArgoCD ใน Hub cluster จัดการ Spoke clusters:

┌────────────────────────────────────────────────────┐
│            Git Repository (Config)                  │
│                                                    │
│  clusters/                                         │
│  ├── production-us-east/                           │
│  │   └── apps.yaml                                 │
│  ├── production-us-west/                           │
│  │   └── apps.yaml                                 │
│  └── staging/                                      │
│      └── apps.yaml                                 │
└────────────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────────┐
│          Hub Cluster (Management)                   │
│                                                    │
│  ArgoCD (manages all clusters)                     │
│  ├── ApplicationSet for prod-us-east               │
│  ├── ApplicationSet for prod-us-west               │
│  └── ApplicationSet for staging                    │
└───────────┬─────────────────┬──────────────────────┘
            │ manages          │ manages
            ▼                  ▼
┌──────────────────┐  ┌──────────────────┐
│ Spoke Cluster 1  │  │ Spoke Cluster 2  │
│ (prod-us-east)   │  │ (prod-us-west)   │
└──────────────────┘  └──────────────────┘
```

### ApplicationSet Pattern

```yaml
# argocd/applicationset/production-apps.yaml
# Deploy เดียวกันไปทุก production clusters

apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: production-apps
  namespace: argocd
spec:
  generators:
    # List ของ clusters
    - clusters:
        selector:
          matchLabels:
            environment: production
  
  template:
    metadata:
      name: '{{name}}-production-apps'
    spec:
      project: production
      
      source:
        repoURL: https://github.com/company/gitops-config
        targetRevision: main
        path: 'apps/{{metadata.annotations.region}}'
      
      destination:
        server: '{{server}}'
        namespace: production
      
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
```

### Environment Promotion Pipeline

```
GitOps Environment Promotion:

1. Developer แก้ไข app (code/config)
2. Build pipeline สร้าง immutable artifact
3. Pipeline update image tag ใน dev environment config
4. ArgoCD sync ไป dev cluster
5. Automated tests run ใน dev
6. ถ้า pass → promote ไป staging (PR หรือ auto)
7. ถ้า pass → promote ไป production (manual approval)
```

```yaml
# .github/workflows/promote.yml
# Automated environment promotion

name: Environment Promotion

on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main]

jobs:
  promote-dev:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    steps:
      - name: Checkout GitOps Config
        uses: actions/checkout@v4
        with:
          repository: company/gitops-config
          token: ${{ secrets.GITOPS_TOKEN }}
      
      - name: Update Image Tag (Dev)
        run: |
          SERVICE=${{ env.SERVICE_NAME }}
          NEW_TAG=${{ env.IMAGE_TAG }}
          
          # Update image tag ใน dev config
          yq eval -i \
            ".spec.template.spec.containers[0].image = \"registry.company.com/${SERVICE}:${NEW_TAG}\"" \
            "apps/dev/${SERVICE}/deployment.yaml"
          
          git config user.name "GitOps Bot"
          git config user.email "gitops-bot@company.com"
          git add .
          git commit -m "chore: deploy ${SERVICE}:${NEW_TAG} to dev

          Source: ${{ github.event.workflow_run.html_url }}"
          git push
      
      - name: Wait for Dev Sync
        run: |
          # Wait for ArgoCD ให้ sync ไป dev
          argocd app wait "${SERVICE}-dev" \
            --sync \
            --timeout 300
      
      - name: Run Smoke Tests
        run: |
          npm run test:smoke -- --env dev
  
  promote-staging:
    needs: promote-dev
    runs-on: ubuntu-latest
    steps:
      - name: Create PR to Staging
        uses: peter-evans/create-pull-request@v5
        with:
          token: ${{ secrets.GITOPS_TOKEN }}
          title: "Deploy ${{ env.SERVICE_NAME }}:${{ env.IMAGE_TAG }} to Staging"
          body: |
            ## Deployment Summary
            
            - **Service**: ${{ env.SERVICE_NAME }}
            - **Version**: ${{ env.IMAGE_TAG }}
            - **Dev deployment**: ✅ Successful
            - **Smoke tests**: ✅ Passed
            
            ### Checklist
            - [ ] Review changes
            - [ ] Approve to deploy to staging
          base: main
          branch: "deploy/${{ env.SERVICE_NAME }}-${{ env.IMAGE_TAG }}-staging"
```

---

## 3. Progressive Delivery กับ GitOps {#progressive-delivery}

### Argo Rollouts Integration

```yaml
# apps/production/payment-service/rollout.yaml

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 10
  
  strategy:
    canary:
      # Analysis Template ที่จะใช้ตรวจสอบ
      analysis:
        templates:
          - templateName: payment-success-rate
        args:
          - name: service-name
            value: payment-service
      
      # Progressive traffic shifting
      steps:
        - setWeight: 5
        - pause: {duration: 5m}
        - analysis:
            templates:
              - templateName: payment-success-rate
        - setWeight: 25
        - pause: {duration: 5m}
        - analysis:
            templates:
              - templateName: payment-success-rate
        - setWeight: 50
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: payment-success-rate
        - setWeight: 100
      
      # Traffic routing via Ingress
      trafficRouting:
        nginx:
          stableIngress: payment-service-stable
  
  selector:
    matchLabels:
      app: payment-service
  
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
        - name: payment-service
          image: registry.company.com/payment-service:latest
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi

---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-success-rate
spec:
  args:
    - name: service-name
  
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.99
      failureLimit: 3
      
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              service="{{args.service-name}}",
              status_code!~"5.."
            }[5m])) /
            sum(rate(http_requests_total{
              service="{{args.service-name}}"
            }[5m]))
    
    - name: latency-p99
      interval: 1m
      count: 5
      successCondition: result[0] <= 0.2  # 200ms
      
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99, 
              rate(http_request_duration_seconds_bucket{
                service="{{args.service-name}}"
              }[5m])
            )
```

### Flagger GitOps Integration

```yaml
# apps/production/payment-service/canary.yaml
# Flagger Canary object (managed via GitOps)

apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payment-service
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  
  progressDeadlineSeconds: 120
  
  service:
    port: 80
    targetPort: 8080
    gateways:
      - public-gateway.istio-system.svc.cluster.local
    hosts:
      - payment.company.com
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
  
  analysis:
    interval: 1m
    threshold: 10
    maxWeight: 50
    stepWeight: 5
    
    metrics:
      - name: request-success-rate
        thresholdRange:
          min: 99
        interval: 1m
      
      - name: request-duration
        thresholdRange:
          max: 500
        interval: 30s
    
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://tester/
        timeout: 30s
        metadata:
          type: bash
          cmd: "curl -sd 'test' http://payment-service-canary/api/check | grep 'ok'"
      
      - name: load-test
        url: http://tester/
        timeout: 5s
        metadata:
          type: cmd
          cmd: "hey -z 1m -q 10 -c 2 http://payment-service-canary/"
```

---

## 4. GitOps กับ Databases {#databases}

### Database Schema GitOps

```
Database migrations ใน GitOps workflow:

ปัญหาหลัก:
- Database เป็น stateful → ต่างจาก stateless services
- Migration ต้องทำตามลำดับ (ordering)
- Rollback ยากกว่า services

Solutions:
1. Flyway/Liquibase ใน init containers
2. SchemaHero (Kubernetes-native)
3. Atlas (declarative schema management)
```

### SchemaHero - Declarative Database Schema

```yaml
# database/payment-schema.yaml
# Database schema เป็น Kubernetes resources

apiVersion: databases.schemahero.io/v1alpha4
kind: Table
metadata:
  name: payments
  namespace: production
spec:
  database: payment-db
  
  schema:
    postgres:
      columns:
        - name: id
          type: uuid
          constraints:
            notNull: true
          default: gen_random_uuid()
        
        - name: amount
          type: decimal
          constraints:
            notNull: true
        
        - name: currency
          type: varchar(3)
          constraints:
            notNull: true
        
        - name: status
          type: varchar(20)
          constraints:
            notNull: true
          default: "'pending'"
        
        - name: created_at
          type: timestamp
          constraints:
            notNull: true
          default: now()
        
        - name: updated_at
          type: timestamp
          constraints:
            notNull: true
          default: now()
      
      primaryKey:
        - id
      
      indexes:
        - name: payments_status_idx
          columns: [status]
        
        - name: payments_created_at_idx
          columns: [created_at]

---
# เพิ่ม column ใหม่: แค่ add ลงใน schema
# SchemaHero จะ generate migration อัตโนมัติ
apiVersion: databases.schemahero.io/v1alpha4
kind: Table
metadata:
  name: payments
spec:
  schema:
    postgres:
      columns:
        # ... existing columns ...
        - name: metadata  # NEW COLUMN
          type: jsonb
```

### Database Migration Pipeline

```yaml
# .github/workflows/database-migration.yml

name: Database Migration

on:
  push:
    branches: [main]
    paths:
      - 'database/**'
      - 'migrations/**'

jobs:
  validate-migration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate Migration Syntax
        run: |
          # ตรวจสอบ migration files
          flyway \
            -url="jdbc:postgresql://localhost:5432/testdb" \
            -user=test \
            -password=test \
            validate
      
      - name: Dry Run Migration
        run: |
          # ทดสอบ migration บน test database
          flyway \
            -url="jdbc:postgresql://${{ secrets.TEST_DB_URL }}" \
            -user=${{ secrets.TEST_DB_USER }} \
            -password=${{ secrets.TEST_DB_PASSWORD }} \
            migrate \
            --dryRun=true
      
      - name: Check Backwards Compatibility
        run: |
          # ตรวจสอบว่า migration ไม่ break existing queries
          python3 scripts/check-migration-compatibility.py \
            --migration-path migrations/V${{ env.VERSION }}__*.sql \
            --existing-queries queries/
  
  apply-dev-migration:
    needs: validate-migration
    environment: dev
    steps:
      - name: Apply Migration to Dev
        run: |
          flyway \
            -url="jdbc:postgresql://${{ secrets.DEV_DB_URL }}" \
            migrate
      
      - name: Verify Migration
        run: |
          python3 scripts/verify-migration.py \
            --environment dev
  
  apply-production-migration:
    needs: apply-dev-migration
    environment: production
    steps:
      - name: Pre-migration Backup
        run: |
          pg_dump \
            ${{ secrets.PROD_DB_URL }} \
            --format=custom \
            --file=backup_$(date +%Y%m%d_%H%M%S).dump
          
          # Upload ไปยัง secure storage
          aws s3 cp backup_*.dump s3://company-db-backups/
      
      - name: Apply Migration
        run: |
          flyway \
            -url="jdbc:postgresql://${{ secrets.PROD_DB_URL }}" \
            -outOfOrder=false \
            migrate
```

---

## 5. Drift Detection และ Reconciliation {#drift}

### Configuration Drift คืออะไร?

```
Drift คือความแตกต่างระหว่าง desired state (Git) และ actual state (cluster):

Desired State (Git):
  payment-service:
    replicas: 3
    image: payment-service:v1.2.3
    
Actual State (Cluster):
  payment-service:
    replicas: 5  ← ถูกแก้ไขด้วยมือ!
    image: payment-service:v1.2.3

นี่คือ drift → ArgoCD จะ reconcile กลับไปเป็น 3 replicas
```

### ArgoCD Drift Detection

```yaml
# argocd/applications/payment-service.yaml

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
  namespace: argocd
spec:
  project: production
  
  source:
    repoURL: https://github.com/company/gitops-config
    targetRevision: main
    path: apps/production/payment-service
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    # Auto-sync เมื่อ detect drift
    automated:
      prune: true    # ลบ resources ที่ไม่อยู่ใน Git
      selfHeal: true # แก้ไข manual changes อัตโนมัติ
    
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      
    # Retry ถ้า sync fail
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  # Ignore specific fields
  ignoreDifferences:
    # ไม่ check replicas ถ้ามี HPA
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    
    # ไม่ check certificate ที่ rotate อัตโนมัติ
    - group: ""
      kind: Secret
      name: tls-secret
      jsonPointers:
        - /data/tls.crt
        - /data/tls.key
```

### Custom Drift Detection

```python
# scripts/drift-detector.py
# Custom drift detection สำหรับ business logic

import kubernetes
import yaml
import json

class DriftDetector:
    def __init__(self, cluster_config: str, git_repo: str):
        kubernetes.config.load_kube_config(cluster_config)
        self.k8s = kubernetes.client.AppsV1Api()
        self.git_repo = git_repo
    
    def detect_drift(self, namespace: str) -> list:
        """ตรวจสอบ drift ทุก deployments ใน namespace"""
        
        drifts = []
        
        # ดึง desired state จาก Git
        desired_deployments = self._get_desired_state(namespace)
        
        # ดึง actual state จาก cluster
        actual_deployments = self.k8s.list_namespaced_deployment(namespace)
        
        for deployment in actual_deployments.items:
            name = deployment.metadata.name
            
            if name not in desired_deployments:
                drifts.append({
                    "type": "unauthorized_deployment",
                    "name": name,
                    "severity": "high",
                    "description": f"Deployment {name} not in Git config"
                })
                continue
            
            desired = desired_deployments[name]
            
            # ตรวจสอบ image version
            actual_image = deployment.spec.template.spec.containers[0].image
            desired_image = desired['spec']['template']['spec']['containers'][0]['image']
            
            if actual_image != desired_image:
                drifts.append({
                    "type": "image_drift",
                    "name": name,
                    "severity": "high",
                    "actual": actual_image,
                    "desired": desired_image,
                    "description": f"Image mismatch for {name}"
                })
            
            # ตรวจสอบ replicas (if no HPA)
            if not self._has_hpa(name, namespace):
                actual_replicas = deployment.spec.replicas
                desired_replicas = desired['spec']['replicas']
                
                if actual_replicas != desired_replicas:
                    drifts.append({
                        "type": "replica_drift",
                        "name": name,
                        "severity": "medium",
                        "actual": actual_replicas,
                        "desired": desired_replicas
                    })
        
        return drifts
    
    def auto_remediate(self, drift: dict):
        """แก้ไข drift อัตโนมัติ"""
        
        if drift['type'] == 'image_drift':
            # Trigger ArgoCD sync
            self._trigger_argocd_sync(drift['name'])
        
        elif drift['type'] == 'replica_drift':
            # Update replicas back to desired
            deployment = self.k8s.read_namespaced_deployment(
                drift['name'], 
                "production"
            )
            deployment.spec.replicas = drift['desired']
            self.k8s.patch_namespaced_deployment(
                drift['name'],
                "production",
                deployment
            )
        
        elif drift['type'] == 'unauthorized_deployment':
            # Alert security team (don't auto-delete)
            self._alert_security_team(drift)
```

---

## 6. Secrets Management ใน GitOps {#secrets}

### Sealed Secrets

```yaml
# ปัญหา: ไม่สามารถ commit secrets ใน Git

# วิธีแก้: Sealed Secrets (Bitnami)
# SealedSecret → ถอดรหัสได้เฉพาะ controller ใน cluster นั้น

# สร้าง SealedSecret:
echo -n "my-secret-value" | \
  kubeseal \
    --controller-name=sealed-secrets \
    --format yaml \
    > sealed-secret.yaml

# ผลลัพธ์ที่ safe to commit:
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: payment-db-secret
  namespace: production
spec:
  encryptedData:
    DATABASE_URL: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    # encrypted ด้วย public key ของ cluster
    # ถอดรหัสได้แค่ใน cluster นั้น
```

### External Secrets Operator

```yaml
# external-secrets/payment-service-secret.yaml
# Pull secrets จาก Vault/AWS Secrets Manager เข้า Kubernetes

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-service-secrets
  namespace: production
spec:
  refreshInterval: 1h
  
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  
  target:
    name: payment-service-secrets
    creationPolicy: Owner
  
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: secret/production/payment-service
        property: database_url
    
    - secretKey: API_KEY
      remoteRef:
        key: secret/production/payment-service
        property: api_key
    
    - secretKey: JWT_SECRET
      remoteRef:
        key: secret/production/payment-service
        property: jwt_secret

---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.company.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "payment-service"
```

---

## 7. GitOps Security {#security}

### Supply Chain Security ใน GitOps

```
GitOps Supply Chain Threats:

1. Compromised Git repository
   Mitigation: Signed commits, branch protection

2. Compromised container image
   Mitigation: Image signing (Cosign), admission control

3. Compromised GitOps controller
   Mitigation: RBAC, network policy, auditing

4. Privilege escalation via GitOps
   Mitigation: Minimal permissions, namespace isolation

5. Secret exposure
   Mitigation: Sealed Secrets, External Secrets
```

### SLSA (Supply-chain Levels for Software Artifacts)

```yaml
# .github/workflows/slsa-compliant-build.yml

name: SLSA-Compliant Build

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and Push Image
        id: build
        run: |
          IMAGE="registry.company.com/payment-service:${{ github.sha }}"
          
          docker buildx build \
            --platform linux/amd64,linux/arm64 \
            --tag $IMAGE \
            --push \
            --file Dockerfile \
            .
          
          # Get digest
          DIGEST=$(docker inspect \
            --format='{{index .RepoDigests 0}}' \
            $IMAGE | cut -d@ -f2)
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT
      
      - name: Sign Image with Cosign
        env:
          COSIGN_EXPERIMENTAL: 1
        run: |
          cosign sign \
            --yes \
            registry.company.com/payment-service@${{ steps.build.outputs.image-digest }}
      
      - name: Generate SBOM
        run: |
          syft \
            registry.company.com/payment-service@${{ steps.build.outputs.image-digest }} \
            --output spdx-json=sbom.spdx.json
      
      - name: Attest SBOM
        env:
          COSIGN_EXPERIMENTAL: 1
        run: |
          cosign attest \
            --yes \
            --predicate sbom.spdx.json \
            --type spdxjson \
            registry.company.com/payment-service@${{ steps.build.outputs.image-digest }}
  
  verify-before-deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Verify Image Signature
        run: |
          cosign verify \
            --certificate-identity-regexp="https://github.com/company/payment-service" \
            --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
            registry.company.com/payment-service@${{ needs.build.outputs.image-digest }}
      
      - name: Verify SBOM Attestation
        run: |
          cosign verify-attestation \
            --type spdxjson \
            --certificate-identity-regexp="https://github.com/company/payment-service" \
            --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
            registry.company.com/payment-service@${{ needs.build.outputs.image-digest }}
```

---

## 8. Case Studies {#case-studies}

### Case Study 1: Financial Services Multi-Region GitOps

**บริบท:**
- Global payment processor
- 15 regions, 3 cloud providers
- 200+ microservices
- Zero-downtime requirement

**GitOps Architecture:**
```
Repository Structure:
├── base/              # Shared configs
│   ├── common/
│   └── security/
├── overlays/          # Environment-specific
│   ├── us-east-prod/
│   ├── eu-west-prod/
│   ├── ap-south-prod/
│   └── ...
└── policies/          # OPA policies

Promotion Flow:
1. Code merge → staging
2. Automated tests pass → create promotion PR
3. Manual approval → production rollout (1 region)
4. Monitoring period (1 hour)
5. Auto-promote to remaining regions
```

**ผลลัพธ์:**
```
Configuration drift incidents: 0 (vs 50+/year before)
Deployment success rate: 99.8%
Mean time to deploy: 45 min (vs 3 hours)
Rollback time: 3 minutes (vs 2 hours)
Audit compliance: Automated
```

### Case Study 2: Database GitOps Migration

**ปัญหาเดิม:**
```
- DBA ต้องทำ migrations ด้วยมือ
- 3 ครั้ง/เดือน มี migration fail ใน production
- ไม่มี automated rollback
- DBA เป็น bottleneck
```

**โซลูชัน: SchemaHero + GitOps**
```
1. เขียน schema เป็น Kubernetes CRDs
2. Git PR review ก่อน apply
3. SchemaHero generate migration อัตโนมัติ
4. Automated testing ใน non-production
5. Progressive rollout ใน production
```

**ผลลัพธ์:**
```
Migration failures: 3/month → 0
DBA involvement: 40 hours/month → 5 hours/month
Migration lead time: 1 week → 1 day
Rollback capability: Manual 4 hours → Automated 5 minutes
```

---

## 9. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Multi-Cluster Setup

**งาน:** ออกแบบ multi-cluster GitOps สำหรับ:
- 3 environments: dev, staging, production
- Production: 2 regions (us-east, eu-west)
- 10 services

**Deliverables:**
1. Repository structure
2. ArgoCD ApplicationSet configuration
3. Environment promotion strategy
4. Rollback procedure

### แบบฝึกหัดที่ 2: Drift Detection

**งาน:** สร้าง drift detection script ที่:
1. Compare Git config กับ cluster state
2. ตรวจสอบ 5 สิ่ง: image tags, replicas, resource limits, secrets presence, network policies
3. สร้าง report
4. Alert ถ้าพบ critical drift

### แบบฝึกหัดที่ 3: Secrets Strategy

**งาน:** เลือก secrets strategy สำหรับ:
- 3 environments
- 50 services
- Team: 20 developers
- Compliance: SOC2

เปรียบเทียบ:
1. Sealed Secrets
2. External Secrets + Vault
3. External Secrets + AWS Secrets Manager

**Criteria:** Security, Ease of use, Cost, Compliance

### แบบฝึกหัดที่ 4: GitOps Security Audit

**งาน:** Audit GitOps setup สำหรับ security:
1. ตรวจสอบ Git repository permissions
2. ตรวจสอบ ArgoCD RBAC
3. ตรวจสอบ image signing
4. ตรวจสอบ secrets management
5. สร้าง security improvement plan

---

## สรุป

Advanced GitOps เป็นมากกว่าแค่ "ใช้ Git":

1. **Multi-cluster management** ต้องการ Hub-Spoke หรือ Federation patterns
2. **Progressive delivery กับ GitOps** = Argo Rollouts/Flagger ที่ config-driven
3. **Databases ใน GitOps** ต้องการ careful approach (SchemaHero, Flyway)
4. **Drift detection** ต้องทั้ง automated remediation และ alerting
5. **Security** ต้องครอบคลุม supply chain ทั้งหมด

## อ่านเพิ่มเติม

- OpenGitOps: https://opengitops.dev
- ArgoCD Documentation: https://argo-cd.readthedocs.io
- Flux Documentation: https://fluxcd.io/docs
- Flagger: https://flagger.app
- SchemaHero: https://schemahero.io
- SLSA Framework: https://slsa.dev

---

*Part 86 จาก 100 | CI/CD Mastery Course*

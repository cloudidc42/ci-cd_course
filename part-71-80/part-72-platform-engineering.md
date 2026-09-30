# Part 72: Platform Engineering

## บทนำ

Platform Engineering คือแนวทางการสร้างและดูแล Internal Developer Platform (IDP) โดยมุ่งเน้นการปรับปรุง Developer Experience และเพิ่มประสิทธิภาพการส่งมอบ software

ในบทนี้เราจะเรียนรู้:
- บทบาทและความรับผิดชอบของ Platform Team
- Paved Roads และ Golden Paths
- Abstraction layer เหนือ Kubernetes และ Cloud
- Developer Self-service Portal
- Platform as a Product
- แบบฝึกหัดปฏิบัติ

---

## 72.1 Platform Engineering คืออะไร?

### ความแตกต่างระหว่าง DevOps และ Platform Engineering

```
DevOps (แบบเก่า):
Developer ─────────────────────────────► Production
         ↑ Dev รับผิดชอบทุกอย่าง

Platform Engineering:
Developer ──► Platform (Self-service) ──► Production
              ↑ Platform team ดูแล
              ↑ นักพัฒนา consume services
```

### ทำไม Platform Engineering ถึงสำคัญ?

**Gartner Prediction:**
> "ภายในปี 2026, 80% ขององค์กรซอฟต์แวร์ขนาดใหญ่จะมี Platform Engineering team"

**ตัวเลขที่น่าสนใจ:**
- Developer ใช้เวลา ~35% กับงาน infrastructure/tooling
- เวลาเฉลี่ยในการ onboard developer ใหม่: 3-6 เดือน
- Platform team ช่วยลด onboarding เป็น 2-4 สัปดาห์

---

## 72.2 บทบาทของ Platform Team

### Team Topology

```
┌─────────────────────────────────────────────────────────────┐
│                     Platform Team                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Developer   │  │ Infrastructure│  │    Security &    │  │
│  │  Experience  │  │  Engineering  │  │   Compliance     │  │
│  └──────────────┘  └──────────────┘  └──────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │    CI/CD     │  │  Observability│                        │
│  │  Engineering │  │  Engineering  │                        │
│  └──────────────┘  └──────────────┘                         │
└─────────────────────────────────────────────────────────────┘
                           ↓ provides
┌─────────────────────────────────────────────────────────────┐
│                  Stream-aligned Teams                        │
│  Team A    │    Team B    │    Team C    │    Team D         │
│  payments  │  user-mgmt   │  inventory   │  notifications    │
└─────────────────────────────────────────────────────────────┘
```

### Core Responsibilities

**1. Developer Experience (DevEx)**
- ออกแบบและดูแล Developer Portal (Backstage)
- สร้างและบำรุงรักษา templates
- เขียนเอกสารและ tutorials
- เก็บ feedback จาก developer

**2. Infrastructure Engineering**
- จัดการ Kubernetes clusters
- ดูแล Cloud infrastructure (Terraform)
- Network topology
- Cost optimization

**3. CI/CD Engineering**
- ออกแบบ pipeline templates
- ดูแล CI/CD platform (Jenkins, GitHub Actions, ArgoCD)
- Build artifact management
- Release automation

**4. Observability Engineering**
- ดูแล monitoring stack (Prometheus, Grafana)
- Log aggregation (ELK/Loki)
- Tracing (Jaeger/Tempo)
- Alerting

---

## 72.3 Paved Roads

Paved Road คือเส้นทางที่ Platform team ลงทุนสร้างและบำรุงรักษา เพื่อให้ developer ทำตามได้ง่ายและปลอดภัย

### หลักการ Paved Road

```
Hard path (ไม่มี paved road):
Developer → ค้นหาเอง → ลองผิดลองถูก → อาจเกิด security issues
           ↑ ใช้เวลานาน, ไม่ consistent

Paved Road:
Developer → ใช้ template/guideline → Follow best practices
           ↑ เร็ว, consistent, secure
```

### ตัวอย่าง Paved Road: CI/CD Pipeline Template

```yaml
# .github/workflows/paved-road-cicd.yaml
# Template นี้ Platform team สร้างและดูแล
# Developer แค่ copy และปรับ parameters

name: Platform CI/CD Pipeline

on:
  push:
    branches: [main, 'release/**']
  pull_request:
    branches: [main]

# ตัวแปรที่ Platform กำหนด (ไม่ควรแก้ไข)
env:
  REGISTRY: ghcr.io
  SONAR_HOST: https://sonar.mycompany.com
  ARGOCD_SERVER: argocd.mycompany.com

jobs:
  # ─── Stage 1: Code Quality ───────────────────────────────
  lint-and-test:
    name: Lint & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup language runtime
        uses: mycompany/setup-runtime@v2  # Custom action ของ Platform team
        with:
          language: ${{ vars.LANGUAGE }}    # Developer กำหนด
          version: ${{ vars.RUNTIME_VERSION }}

      - name: Run linter
        run: make lint

      - name: Run unit tests
        run: make test-unit

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  # ─── Stage 2: Security Scanning ─────────────────────────
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: lint-and-test
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - name: Run SAST (Semgrep)
        uses: returntocorp/semgrep-action@v1
        with:
          config: p/default

      - name: Dependency vulnerability scan
        uses: mycompany/dep-scan@v1  # Platform custom action
        with:
          fail-on: HIGH,CRITICAL

      - name: SonarQube scan
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ env.SONAR_HOST }}

  # ─── Stage 3: Build & Push ──────────────────────────────
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: security-scan
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/mycompany/${{ github.event.repository.name }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=sha-
            type=semver,pattern={{version}}

      - name: Build and push
        id: push
        uses: docker/build-push-action@v5
        with:
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Sign image
        if: github.event_name != 'pull_request'
        uses: mycompany/sign-image@v1  # Cosign wrapper
        with:
          image: ${{ env.REGISTRY }}/mycompany/${{ github.event.repository.name }}@${{ steps.push.outputs.digest }}

  # ─── Stage 4: Deploy to Staging ─────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - name: Update staging manifest
        uses: mycompany/update-deployment@v1
        with:
          argocd-server: ${{ env.ARGOCD_SERVER }}
          app-name: ${{ github.event.repository.name }}-staging
          image-tag: ${{ needs.build.outputs.image-tag }}
          argocd-token: ${{ secrets.ARGOCD_TOKEN }}

      - name: Wait for rollout
        uses: mycompany/wait-rollout@v1
        with:
          app-name: ${{ github.event.repository.name }}-staging
          timeout: 300

      - name: Run smoke tests
        run: |
          curl -f https://${{ github.event.repository.name }}.staging.mycompany.com/health

  # ─── Stage 5: Deploy to Production ──────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production  # ต้องการ approval
    steps:
      - name: Deploy to production
        uses: mycompany/update-deployment@v1
        with:
          argocd-server: ${{ env.ARGOCD_SERVER }}
          app-name: ${{ github.event.repository.name }}-prod
          image-tag: ${{ needs.build.outputs.image-tag }}
          argocd-token: ${{ secrets.ARGOCD_TOKEN }}
          strategy: canary  # Canary deployment
          canary-weight: 20
```

---

## 72.4 Abstractions เหนือ Kubernetes

### Platform Team สร้าง Abstractions

Platform team สร้างชั้น abstraction ที่ซ่อน Kubernetes complexity จาก developer

**ระดับ Abstraction:**

```
Level 3 (Developer sees):
  "Deploy my app with 2 replicas, needs PostgreSQL"

Level 2 (Platform translates):
  Helm values.yaml, Kubernetes Deployment, Service, HPA

Level 1 (Infrastructure):
  AWS EKS, RDS, ElasticCache, ALB
```

### Custom Resource Definitions (CRDs) สำหรับ Developer

```yaml
# Platform CRD: Application
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.platform.mycompany.com
spec:
  group: platform.mycompany.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - image
                - replicas
              properties:
                image:
                  type: string
                  description: Container image
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 50
                port:
                  type: integer
                  default: 8080
                resources:
                  type: object
                  properties:
                    size:
                      type: string
                      enum: [small, medium, large, xlarge]
                      default: small
                database:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                    type:
                      type: string
                      enum: [postgresql, mysql, redis]
                    size:
                      type: string
                      enum: [small, medium, large]
                autoscaling:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                      default: true
                    minReplicas:
                      type: integer
                      default: 2
                    maxReplicas:
                      type: integer
                      default: 10
                    targetCPU:
                      type: integer
                      default: 70
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
```

```yaml
# Developer สร้าง Application (ง่ายมาก)
apiVersion: platform.mycompany.com/v1
kind: Application
metadata:
  name: payment-service
  namespace: team-payments
spec:
  image: ghcr.io/mycompany/payment-service:v1.2.3
  replicas: 3
  port: 8080
  resources:
    size: medium  # Platform แปลเป็น requests/limits
  database:
    enabled: true
    type: postgresql
    size: small
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 20
    targetCPU: 70
```

### Operator สำหรับ Application CRD

```go
// platform-operator/controllers/application_controller.go
package controllers

import (
    "context"
    "fmt"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    platformv1 "github.com/mycompany/platform-operator/api/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

type ApplicationReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *ApplicationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // ดึง Application resource
    var app platformv1.Application
    if err := r.Get(ctx, req.NamespacedName, &app); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // แปลง resources.size เป็น actual resource requests
    resources := r.sizeToResources(app.Spec.Resources.Size)

    // สร้าง Deployment
    deployment := r.buildDeployment(&app, resources)
    if err := r.createOrUpdate(ctx, deployment); err != nil {
        return ctrl.Result{}, err
    }

    // สร้าง Service
    service := r.buildService(&app)
    if err := r.createOrUpdate(ctx, service); err != nil {
        return ctrl.Result{}, err
    }

    // สร้าง HPA ถ้า autoscaling enabled
    if app.Spec.Autoscaling.Enabled {
        hpa := r.buildHPA(&app)
        if err := r.createOrUpdate(ctx, hpa); err != nil {
            return ctrl.Result{}, err
        }
    }

    // สร้าง Database ถ้า database enabled
    if app.Spec.Database.Enabled {
        dbClaim := r.buildDatabaseClaim(&app)
        if err := r.createOrUpdate(ctx, dbClaim); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Update status
    app.Status.Phase = "Running"
    app.Status.DeployedImage = app.Spec.Image
    if err := r.Status().Update(ctx, &app); err != nil {
        return ctrl.Result{}, err
    }

    log.Info("Reconciled Application", "name", app.Name)
    return ctrl.Result{}, nil
}

func (r *ApplicationReconciler) sizeToResources(size string) corev1.ResourceRequirements {
    sizes := map[string]corev1.ResourceRequirements{
        "small": {
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("100m"),
                corev1.ResourceMemory: resource.MustParse("128Mi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("200m"),
                corev1.ResourceMemory: resource.MustParse("256Mi"),
            },
        },
        "medium": {
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("500m"),
                corev1.ResourceMemory: resource.MustParse("512Mi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("1"),
                corev1.ResourceMemory: resource.MustParse("1Gi"),
            },
        },
        "large": {
            Requests: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("2"),
                corev1.ResourceMemory: resource.MustParse("2Gi"),
            },
            Limits: corev1.ResourceList{
                corev1.ResourceCPU:    resource.MustParse("4"),
                corev1.ResourceMemory: resource.MustParse("4Gi"),
            },
        },
    }
    return sizes[size]
}
```

---

## 72.5 Developer Self-service Portal

### Service Catalog UI

```typescript
// backstage-plugin/src/components/ServiceCatalog.tsx
import React, { useState } from 'react';
import {
  Button,
  Card,
  Grid,
  Typography,
  Dialog,
  TextField,
  Select,
} from '@material-ui/core';

interface CreateServiceForm {
  name: string;
  team: string;
  language: string;
  database: string;
  cacheEnabled: boolean;
}

export const CreateServiceWizard = () => {
  const [step, setStep] = useState(1);
  const [form, setForm] = useState<CreateServiceForm>({
    name: '',
    team: '',
    language: 'go',
    database: 'postgresql',
    cacheEnabled: false,
  });

  const handleCreate = async () => {
    // เรียก Backstage Scaffolder API
    const response = await fetch('/api/scaffolder/v2/tasks', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        templateRef: 'template:default/go-microservice',
        values: form,
      }),
    });

    const { id } = await response.json();
    // Redirect to task status page
    window.location.href = `/create/tasks/${id}`;
  };

  return (
    <Card>
      <Typography variant="h5">สร้าง Service ใหม่</Typography>

      {/* Step 1: Basic Info */}
      {step === 1 && (
        <Grid container spacing={2}>
          <Grid item xs={12}>
            <TextField
              label="ชื่อ Service"
              value={form.name}
              onChange={e => setForm({ ...form, name: e.target.value })}
              helperText="lowercase, kebab-case เท่านั้น"
              fullWidth
            />
          </Grid>
          <Grid item xs={12}>
            <Select
              label="ทีม"
              value={form.team}
              onChange={e => setForm({ ...form, team: e.target.value })}
              fullWidth
            >
              <option value="team-payments">Payments</option>
              <option value="team-users">Users</option>
              <option value="team-inventory">Inventory</option>
            </Select>
          </Grid>
          <Grid item xs={12}>
            <Button
              variant="contained"
              color="primary"
              onClick={() => setStep(2)}
            >
              ถัดไป
            </Button>
          </Grid>
        </Grid>
      )}

      {/* Step 2: Technology */}
      {step === 2 && (
        <Grid container spacing={2}>
          <Grid item xs={12}>
            <Select
              label="ภาษา"
              value={form.language}
              onChange={e => setForm({ ...form, language: e.target.value })}
              fullWidth
            >
              <option value="go">Go (แนะนำ)</option>
              <option value="typescript">TypeScript/Node.js</option>
              <option value="python">Python</option>
              <option value="java">Java/Spring Boot</option>
            </Select>
          </Grid>
          <Grid item xs={12}>
            <Select
              label="Database"
              value={form.database}
              onChange={e => setForm({ ...form, database: e.target.value })}
              fullWidth
            >
              <option value="postgresql">PostgreSQL</option>
              <option value="mysql">MySQL</option>
              <option value="none">ไม่ต้องการ</option>
            </Select>
          </Grid>
          <Grid item xs={12}>
            <Button onClick={() => setStep(1)}>ย้อนกลับ</Button>
            <Button
              variant="contained"
              color="primary"
              onClick={() => setStep(3)}
            >
              ถัดไป
            </Button>
          </Grid>
        </Grid>
      )}

      {/* Step 3: Review & Create */}
      {step === 3 && (
        <Grid container spacing={2}>
          <Grid item xs={12}>
            <Typography variant="h6">สรุปการตั้งค่า</Typography>
            <Typography>ชื่อ: {form.name}</Typography>
            <Typography>ทีม: {form.team}</Typography>
            <Typography>ภาษา: {form.language}</Typography>
            <Typography>Database: {form.database}</Typography>
          </Grid>
          <Grid item xs={12}>
            <Button onClick={() => setStep(2)}>ย้อนกลับ</Button>
            <Button
              variant="contained"
              color="primary"
              onClick={handleCreate}
            >
              สร้าง Service
            </Button>
          </Grid>
        </Grid>
      )}
    </Card>
  );
};
```

---

## 72.6 Platform as a Product

Platform team ต้องคิดถึง developer เหมือนกับ customer ไม่ใช่แค่ผู้ใช้ภายใน

### Product Management สำหรับ Platform

```
Platform Roadmap Q1 2025:
┌────────────────────────────────────────────────────────────────┐
│  Priority  │  Feature                    │  Status  │  ETA     │
├────────────────────────────────────────────────────────────────┤
│  P0        │  One-click database setup    │  In Prog │  Jan 15  │
│  P0        │  Simplified deployment       │  Done    │  Jan 1   │
│  P1        │  Cost visibility per team    │  Planned │  Feb 1   │
│  P1        │  Security scan in pipeline  │  Done    │  Dec 15  │
│  P2        │  Auto-scaling by default    │  Planned │  Mar 1   │
│  P3        │  Multi-cloud support        │  Backlog │  Q2      │
└────────────────────────────────────────────────────────────────┘
```

### NPS Survey สำหรับ Developer

```typescript
// platform-nps-survey.ts
interface NPSSurvey {
  score: number;  // 0-10
  category: 'deploying' | 'monitoring' | 'debugging' | 'onboarding';
  comments: string;
  team: string;
  timestamp: Date;
}

// ส่ง survey หลังจาก developer ทำ action
async function sendNPSSurvey(action: string, userId: string) {
  await fetch('/api/platform/nps', {
    method: 'POST',
    body: JSON.stringify({
      trigger: action,
      userId,
      timestamp: new Date(),
    }),
  });
}

// คำนวณ NPS score
function calculateNPS(surveys: NPSSurvey[]): number {
  const promoters = surveys.filter(s => s.score >= 9).length;
  const detractors = surveys.filter(s => s.score <= 6).length;
  const total = surveys.length;

  return ((promoters - detractors) / total) * 100;
}
```

### Platform SLA

```yaml
# platform-sla.yaml
platform_sla:
  availability:
    backstage_portal: 99.9%
    ci_cd_pipeline: 99.5%
    artifact_registry: 99.9%

  performance:
    pipeline_start_time: < 30 seconds
    template_creation_time: < 5 minutes
    deployment_time: < 10 minutes

  support:
    p0_response_time: 15 minutes
    p1_response_time: 2 hours
    p2_response_time: 1 business day
    p3_response_time: 1 week

  on_call:
    schedule: PagerDuty rotation (platform-team)
    escalation: Head of Platform Engineering
```

---

## 72.7 Infrastructure Abstraction Layer

### Terraform Module Library

```hcl
# modules/microservice-infra/main.tf
# Platform team สร้าง module นี้
# Developer ใช้โดยไม่ต้องรู้ AWS details

variable "service_name" {
  type        = string
  description = "ชื่อ service"
}

variable "environment" {
  type        = string
  description = "development, staging, production"
}

variable "database" {
  type = object({
    enabled       = bool
    instance_class = string  # small, medium, large
    storage_gb    = number
  })
  default = {
    enabled       = false
    instance_class = "small"
    storage_gb    = 20
  }
}

variable "cache" {
  type = object({
    enabled    = bool
    node_type  = string  # small, medium, large
  })
  default = {
    enabled   = false
    node_type = "small"
  }
}

# ─── Database ─────────────────────────────────────────────────
module "database" {
  count  = var.database.enabled ? 1 : 0
  source = "../rds"

  identifier     = "${var.service_name}-${var.environment}"
  engine         = "postgres"
  engine_version = "14"

  # แปลง size เป็น actual AWS instance class
  instance_class = lookup({
    small  = "db.t3.micro"
    medium = "db.t3.medium"
    large  = "db.r5.large"
  }, var.database.instance_class, "db.t3.micro")

  allocated_storage = var.database.storage_gb
  subnet_ids        = data.aws_subnets.private.ids
  vpc_security_group_ids = [aws_security_group.database.id]

  backup_retention_period = var.environment == "production" ? 30 : 7
  deletion_protection     = var.environment == "production"

  tags = local.common_tags
}

# ─── Cache ────────────────────────────────────────────────────
module "cache" {
  count  = var.cache.enabled ? 1 : 0
  source = "../elasticache"

  cluster_id = "${var.service_name}-${var.environment}"

  node_type = lookup({
    small  = "cache.t3.micro"
    medium = "cache.t3.medium"
    large  = "cache.r6g.large"
  }, var.cache.node_type, "cache.t3.micro")

  subnet_group_name  = aws_elasticache_subnet_group.this.name
  security_group_ids = [aws_security_group.cache.id]

  tags = local.common_tags
}

# ─── Kubernetes Namespace ─────────────────────────────────────
resource "kubernetes_namespace" "this" {
  metadata {
    name = "${var.service_name}-${var.environment}"
    labels = {
      "app.kubernetes.io/managed-by" = "platform"
      "team"                         = var.service_name
      "environment"                  = var.environment
    }
  }
}

# ─── Resource Quotas ──────────────────────────────────────────
resource "kubernetes_resource_quota" "this" {
  metadata {
    name      = "quota"
    namespace = kubernetes_namespace.this.metadata[0].name
  }

  spec {
    hard = {
      "requests.cpu"    = var.environment == "production" ? "8" : "4"
      "requests.memory" = var.environment == "production" ? "16Gi" : "8Gi"
      "limits.cpu"      = var.environment == "production" ? "16" : "8"
      "limits.memory"   = var.environment == "production" ? "32Gi" : "16Gi"
      "pods"            = "50"
    }
  }
}

output "database_endpoint" {
  value = var.database.enabled ? module.database[0].endpoint : null
}

output "cache_endpoint" {
  value = var.cache.enabled ? module.cache[0].endpoint : null
}

output "namespace" {
  value = kubernetes_namespace.this.metadata[0].name
}
```

**Developer ใช้ module:**

```hcl
# Developer's terraform/main.tf
module "payment_service_infra" {
  source = "git::https://github.com/mycompany/terraform-modules.git//microservice-infra?ref=v2.3.0"

  service_name = "payment-service"
  environment  = "production"

  database = {
    enabled       = true
    instance_class = "medium"
    storage_gb    = 50
  }

  cache = {
    enabled   = true
    node_type = "small"
  }
}

output "db_endpoint" {
  value = module.payment_service_infra.database_endpoint
}
```

---

## 72.8 Platform Metrics

### ตัวชี้วัดสำคัญของ Platform Team

```python
# platform-metrics-collector.py
# Script สำหรับเก็บ Platform metrics

import requests
import json
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import List

@dataclass
class PlatformMetrics:
    # Developer productivity metrics
    avg_onboarding_time_days: float
    template_usage_count: int
    self_service_ratio: float  # % ที่ทำเองโดยไม่ต้องขอ Platform team

    # Platform reliability metrics
    backstage_uptime: float
    pipeline_success_rate: float
    avg_build_time_minutes: float

    # Developer satisfaction
    nps_score: float
    support_ticket_count: int
    avg_resolution_time_hours: float


def collect_github_metrics(org: str, token: str) -> dict:
    """เก็บ metrics จาก GitHub"""
    headers = {"Authorization": f"token {token}"}

    # นับ repositories ใหม่
    repos = requests.get(
        f"https://api.github.com/orgs/{org}/repos",
        headers=headers,
        params={"sort": "created", "per_page": 100},
    ).json()

    new_repos_30d = [
        r for r in repos
        if datetime.strptime(r["created_at"], "%Y-%m-%dT%H:%M:%SZ")
        > datetime.now() - timedelta(days=30)
    ]

    # เวลาเฉลี่ยในการ merge PR
    prs = requests.get(
        f"https://api.github.com/search/issues",
        headers=headers,
        params={
            "q": f"org:{org} is:pr is:merged",
            "sort": "created",
            "per_page": 100,
        },
    ).json()

    return {
        "new_repos_30d": len(new_repos_30d),
        "total_prs_merged": prs.get("total_count", 0),
    }


def calculate_developer_toil(support_tickets: List[dict]) -> dict:
    """คำนวณ toil ที่ developer ต้องเจอ"""
    categories = {}
    for ticket in support_tickets:
        cat = ticket.get("category", "other")
        categories[cat] = categories.get(cat, 0) + 1

    total = len(support_tickets)
    return {
        "total_tickets": total,
        "by_category": {
            k: {"count": v, "percentage": v/total*100 if total > 0 else 0}
            for k, v in categories.items()
        },
    }
```

### Grafana Dashboard สำหรับ Platform

```json
{
  "dashboard": {
    "title": "Platform Engineering Metrics",
    "panels": [
      {
        "title": "Developer Onboarding Time",
        "type": "stat",
        "targets": [
          {
            "expr": "avg(platform_onboarding_duration_days)"
          }
        ]
      },
      {
        "title": "Self-service Rate",
        "type": "gauge",
        "targets": [
          {
            "expr": "sum(platform_self_service_requests) / sum(platform_total_requests) * 100"
          }
        ],
        "options": {
          "thresholds": [
            { "value": 0, "color": "red" },
            { "value": 50, "color": "yellow" },
            { "value": 80, "color": "green" }
          ]
        }
      },
      {
        "title": "Template Usage (30 days)",
        "type": "bar",
        "targets": [
          {
            "expr": "sum by (template_name) (platform_template_usage_total)"
          }
        ]
      },
      {
        "title": "Platform NPS Score",
        "type": "stat",
        "targets": [
          {
            "expr": "platform_nps_score"
          }
        ]
      }
    ]
  }
}
```

---

## 72.9 การ Migrate จาก Traditional Ops ไปสู่ Platform Engineering

### Migration Roadmap

```
Phase 1 (เดือน 1-3): Foundation
├── สร้าง Platform Team
├── ติดตั้ง Backstage
├── Catalog existing services
└── สร้าง basic CI/CD template

Phase 2 (เดือน 3-6): Self-service
├── Terraform module library
├── Database self-service (Crossplane)
├── Monitoring self-service (Grafana dashboards)
└── Developer onboarding wizard

Phase 3 (เดือน 6-12): Optimization
├── Cost visibility per team
├── Security automation
├── Advanced deployment strategies
└── Platform analytics

Phase 4 (ปีที่ 2): Platform as Product
├── NPS tracking
├── Platform roadmap based on feedback
├── Advanced self-service (everything)
└── Internal marketplace
```

### Change Management

```markdown
# Platform Adoption Playbook

## Week 1-2: Pilot Team
- เลือก 1 team ที่ enthusiastic
- ทำงานร่วมกับ team ตลอด
- เก็บ feedback ทุกวัน
- ปรับปรุงทันที

## Week 3-4: Early Adopters
- ขยายไปอีก 2-3 teams
- สร้าง documentation จาก lessons learned
- Office hours: ทุกวันพุธ 14:00-15:00

## Month 2-3: Broad Rollout
- Training sessions สำหรับทุก team
- อัพเดท CONTRIBUTING.md
- Deprecate old ways gracefully

## Ongoing: Continuous Improvement
- Monthly surveys
- Quarterly roadmap review
- Annual platform audit
```

---

## 72.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Platform Team Charter

```markdown
# Platform Team Charter Template

## Mission Statement
[อธิบาย mission ของ Platform team ในไม่เกิน 2 ประโยค]

## Team Structure
- Platform Engineering Lead: _____
- Developer Experience Engineer: _____
- Infrastructure Engineer: _____
- CI/CD Engineer: _____

## Services ที่เราให้
1. _____
2. _____
3. _____

## Success Metrics
- Developer NPS: เป้าหมาย _____
- Self-service rate: เป้าหมาย _____%
- Onboarding time: เป้าหมาย _____ วัน
- Pipeline success rate: เป้าหมาย _____%

## SLA ของเรา
- P0 (Critical): ตอบใน _____ นาที
- P1 (High): ตอบใน _____ ชั่วโมง
- P2 (Medium): ตอบใน _____ วัน
```

### แบบฝึกหัดที่ 2: สร้าง Terraform Module

```hcl
# exercises/terraform-module/main.tf
# สร้าง module สำหรับ S3 bucket ที่ปลอดภัย

variable "bucket_name" {
  type        = string
  description = "ชื่อ bucket"
}

variable "environment" {
  type    = string
  default = "development"
}

variable "versioning" {
  type        = bool
  default     = false
  description = "เปิด versioning หรือไม่"
}

# TODO: Implement the following:
# 1. S3 bucket with enforced encryption
# 2. Block all public access
# 3. Enable versioning if requested
# 4. Add lifecycle policy based on environment
# 5. Add bucket policy for specific IAM roles only
# 6. Output bucket name and ARN

resource "aws_s3_bucket" "this" {
  bucket = "${var.bucket_name}-${var.environment}"

  tags = {
    Name        = var.bucket_name
    Environment = var.environment
    ManagedBy   = "platform-team"
  }
}

# TODO: เพิ่ม resources ที่จำเป็น
```

### แบบฝึกหัดที่ 3: สร้าง Platform Operator

```go
// exercises/platform-operator/main.go
// สร้าง Kubernetes Operator สำหรับ simple "WebApp" CRD

// WebApp spec:
// - image: string
// - replicas: int (default: 2)
// - port: int (default: 8080)
// - domain: string (optional) - สร้าง Ingress ถ้ามี

// Operator ต้องสร้าง:
// 1. Deployment
// 2. Service
// 3. Ingress (ถ้ามี domain)
// 4. HPA (ถ้า replicas > 1)

// TODO: Implement the operator
```

### แบบฝึกหัดที่ 4: Platform Metrics Dashboard

```python
# exercises/platform-metrics.py
# สร้าง script เก็บและแสดง Platform metrics

# เก็บข้อมูลจาก:
# 1. GitHub API: จำนวน repos ใหม่, PR merge time
# 2. ArgoCD API: deployment success rate, deployment frequency
# 3. Custom survey API: NPS scores

# แสดงผลเป็น:
# - Text report
# - Export to Prometheus metrics

# TODO: Implement
import requests
import json

def get_github_metrics(org: str, token: str) -> dict:
    # TODO: implement
    pass

def get_argocd_metrics(argocd_url: str, token: str) -> dict:
    # TODO: implement
    pass

def calculate_platform_score(metrics: dict) -> float:
    # TODO: implement scoring algorithm
    pass

if __name__ == "__main__":
    metrics = {
        "github": get_github_metrics("mycompany", "TOKEN"),
        "argocd": get_argocd_metrics("argocd.mycompany.com", "TOKEN"),
    }
    score = calculate_platform_score(metrics)
    print(f"Platform Score: {score}/100")
```

---

## สรุป

Platform Engineering เปลี่ยน paradigm จาก "Dev vs Ops" เป็น "Platform enables everyone":

1. **Platform เป็น Product** ไม่ใช่แค่ tools collection
2. **Developer เป็น Customer** ต้องได้รับ good experience
3. **Paved Roads** ช่วยให้ทำสิ่งที่ถูกต้องได้ง่าย
4. **Self-service** ลด bottleneck และเพิ่มความเร็ว
5. **Abstractions** ซ่อน complexity จาก developer

Platform Engineering ที่ดีทำให้ developer ไม่ต้องคิดถึง infrastructure และสามารถ focus ที่ business logic ได้อย่างเต็มที่

### ขั้นตอนถัดไป

- ศึกษา [Part 73: CI/CD Metrics & DORA](./part-73-cicd-metrics.md)
- Platform Engineering community: https://platformengineering.org
- Team Topologies book: https://teamtopologies.com

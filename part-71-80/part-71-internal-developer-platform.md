# Part 71: Internal Developer Platform (IDP)

## บทนำ

Internal Developer Platform (IDP) คือแพลตฟอร์มที่ทีม Platform Engineering สร้างขึ้นเพื่อให้นักพัฒนาสามารถ self-service ความต้องการด้านโครงสร้างพื้นฐาน การ deploy และการจัดการระบบได้โดยไม่ต้องรอทีม Ops หรือ Infrastructure

ในบทนี้เราจะเรียนรู้:
- แนวคิดและประโยชน์ของ IDP
- การติดตั้งและใช้งาน Backstage.io
- การสร้าง Software Catalog
- Tech Radar
- Golden Paths
- Self-service Infrastructure
- แบบฝึกหัดปฏิบัติ

---

## 71.1 แนวคิด Internal Developer Platform

### IDP คืออะไร?

IDP ไม่ใช่แค่เครื่องมือหรือผลิตภัณฑ์เดียว แต่คือชั้นนามธรรม (abstraction layer) ที่ซ่อนความซับซ้อนของโครงสร้างพื้นฐานจากนักพัฒนา

```
┌─────────────────────────────────────────────────────────────┐
│                    Developer Experience                      │
├─────────────────────────────────────────────────────────────┤
│                  Internal Developer Platform                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐  │
│  │ Software │ │  CI/CD   │ │ Infra    │ │  Monitoring  │  │
│  │ Catalog  │ │ Pipeline │ │ Provisio │ │  & Logging   │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘  │
├─────────────────────────────────────────────────────────────┤
│              Underlying Infrastructure & Tools               │
│  Kubernetes  │  Terraform  │  AWS/GCP/Azure  │  Datadog    │
└─────────────────────────────────────────────────────────────┘
```

### ปัญหาที่ IDP แก้ไข

**1. Cognitive Overload**
นักพัฒนาต้องรู้จักเครื่องมือหลายสิบอย่าง:
- Kubernetes, Helm, Terraform
- AWS Console, IAM roles
- Prometheus, Grafana, ELK Stack
- Jenkins, GitHub Actions, ArgoCD

**2. Waiting Time**
- รอ Ops team จัดการ infrastructure: 2-3 วัน
- รอสร้าง repository ใหม่: 1 วัน
- รอ database provisioning: 3-5 วัน

**3. Inconsistency**
- แต่ละทีม deploy แตกต่างกัน
- Configuration drift ระหว่าง environments
- Security practices ไม่เหมือนกัน

### คุณค่าของ IDP

```
ก่อน IDP:
Developer → Request Ticket → Ops Team → (2-5 days) → Infrastructure Ready

หลัง IDP:
Developer → Self-service Portal → (5-30 minutes) → Infrastructure Ready
```

---

## 71.2 Backstage.io: Open Source IDP Framework

### Backstage คืออะไร?

Backstage เป็น open-source platform ที่ Spotify พัฒนาขึ้น ปัจจุบันเป็น CNCF project ช่วยให้องค์กรสร้าง Developer Portal ของตัวเองได้

### การติดตั้ง Backstage

**ความต้องการระบบ:**
- Node.js 18+
- Yarn
- Docker
- PostgreSQL (production)

**ขั้นตอนที่ 1: สร้าง Backstage App**

```bash
# ติดตั้ง Backstage CLI
npx @backstage/create-app@latest

# ตอบคำถาม:
? Enter a name for the app [required] my-company-backstage
? Select database for the backend: PostgreSQL

# รอให้ติดตั้งเสร็จ...
cd my-company-backstage
```

**ขั้นตอนที่ 2: โครงสร้างไฟล์**

```
my-company-backstage/
├── app-config.yaml          # Main configuration
├── app-config.production.yaml
├── packages/
│   ├── app/                 # Frontend
│   │   ├── src/
│   │   │   ├── App.tsx
│   │   │   └── components/
│   │   └── package.json
│   └── backend/             # Backend
│       ├── src/
│       │   └── index.ts
│       └── package.json
├── plugins/                 # Custom plugins
└── package.json
```

**ขั้นตอนที่ 3: Configuration หลัก**

```yaml
# app-config.yaml
app:
  title: My Company Developer Portal
  baseUrl: http://localhost:3000

organization:
  name: My Company

backend:
  baseUrl: http://localhost:7007
  listen:
    port: 7007
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: ${POSTGRES_PORT}
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

auth:
  environment: development
  providers:
    github:
      development:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}

catalog:
  import:
    entityFilename: catalog-info.yaml
    pullRequestBranchName: backstage-integration
  rules:
    - allow: [Component, System, API, Resource, Location, User, Group]
  locations:
    - type: file
      target: ../../examples/entities.yaml
    - type: url
      target: https://github.com/mycompany/services/blob/main/catalog-info.yaml
```

**ขั้นตอนที่ 4: รัน Development Server**

```bash
yarn dev
# เปิด http://localhost:3000
```

---

## 71.3 Software Catalog

Software Catalog คือ "แผนที่" ของ software ecosystem ทั้งหมดในองค์กร

### Entity Types

```yaml
# catalog-info.yaml - Component
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: Handles payment processing for the platform
  tags:
    - payment
    - critical
    - java
  annotations:
    github.com/project-slug: mycompany/payment-service
    backstage.io/techdocs-ref: dir:.
    prometheus.io/rule: payment_service_errors_total
    pagerduty.com/service-id: PABC123
  links:
    - url: https://payment.mycompany.com
      title: Production URL
      icon: web
    - url: https://grafana.mycompany.com/d/payment
      title: Grafana Dashboard
      icon: dashboard
spec:
  type: service
  lifecycle: production
  owner: team-payments
  system: payment-platform
  dependsOn:
    - component:postgres-payment
    - component:redis-cache
  providesApis:
    - payment-api
  consumesApis:
    - notification-api
    - user-api
```

```yaml
# catalog-info.yaml - API
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payment-api
  description: Payment processing API
  tags:
    - rest
    - payment
spec:
  type: openapi
  lifecycle: production
  owner: team-payments
  system: payment-platform
  definition:
    $text: ./openapi.yaml
```

```yaml
# catalog-info.yaml - System
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: payment-platform
  description: Complete payment processing platform
spec:
  owner: team-payments
  domain: finance
```

```yaml
# catalog-info.yaml - Group (Team)
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: team-payments
  description: Payment team
spec:
  type: team
  profile:
    displayName: Payments Team
    email: payments@mycompany.com
  parent: engineering
  children: []
  members:
    - john.doe
    - jane.smith
```

### Auto-discovery ด้วย GitHub Integration

```yaml
# app-config.yaml
catalog:
  providers:
    github:
      mycompany:
        organization: mycompany
        catalogPath: /catalog-info.yaml
        filters:
          branch: main
          repository: '.*'
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
```

### Catalog Ingestion Pipeline

```typescript
// packages/backend/src/plugins/catalog.ts
import { CatalogBuilder } from '@backstage/plugin-catalog-backend';
import { GithubOrgEntityProvider } from '@backstage/plugin-catalog-backend-module-github';
import { Router } from 'express';
import { PluginEnvironment } from '../types';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  const builder = await CatalogBuilder.create(env);

  // เพิ่ม GitHub Org Provider
  builder.addEntityProvider(
    GithubOrgEntityProvider.fromConfig(env.config, {
      id: 'production',
      orgUrl: 'https://github.com/mycompany',
      logger: env.logger,
      schedule: env.scheduler.createScheduledTaskRunner({
        frequency: { minutes: 60 },
        timeout: { minutes: 15 },
      }),
    }),
  );

  const { processingEngine, router } = await builder.build();
  await processingEngine.start();
  return router;
}
```

---

## 71.4 Tech Radar

Tech Radar คือเครื่องมือช่วยให้องค์กรตัดสินใจเกี่ยวกับเทคโนโลยีที่ใช้งาน

### โครงสร้าง Tech Radar

```
┌─────────────────────────────────────────────────────────┐
│                      TECH RADAR                         │
│                                                         │
│    HOLD    │   ASSESS   │   TRIAL    │   ADOPT         │
│ ───────────┼────────────┼────────────┼──────────       │
│  เทคโนโลยี │ ศึกษาและ  │ ทดลองใช้  │  ใช้งานได้     │
│  ที่ควร   │  ประเมิน  │  ในบางโปร │  ในทุกโปร     │
│  หยุดใช้  │            │  เจค       │  เจค           │
└─────────────────────────────────────────────────────────┘
```

### ติดตั้ง Tech Radar Plugin

```bash
# Frontend plugin
yarn --cwd packages/app add @backstage-community/plugin-tech-radar

# Backend plugin (optional)
yarn --cwd packages/backend add @backstage-community/plugin-tech-radar-backend
```

```typescript
// packages/app/src/App.tsx
import { TechRadarPage } from '@backstage-community/plugin-tech-radar';

const routes = (
  <FlatRoutes>
    {/* ... existing routes ... */}
    <Route path="/tech-radar" element={<TechRadarPage />} />
  </FlatRoutes>
);
```

### สร้าง Custom Tech Radar Data

```typescript
// packages/app/src/components/TechRadar/TechRadarData.ts
import {
  TechRadarLoaderResponse,
  RadarQuadrant,
  RadarRing,
  RadarEntry,
} from '@backstage-community/plugin-tech-radar';

export const techRadarData: TechRadarLoaderResponse = {
  quadrants: [
    { id: 'infrastructure', name: 'Infrastructure' },
    { id: 'languages', name: 'Languages & Frameworks' },
    { id: 'data', name: 'Data Management' },
    { id: 'platforms', name: 'Platforms & Tools' },
  ],
  rings: [
    { id: 'adopt', name: 'ADOPT', color: '#5ba300' },
    { id: 'trial', name: 'TRIAL', color: '#009eb0' },
    { id: 'assess', name: 'ASSESS', color: '#c7ba00' },
    { id: 'hold', name: 'HOLD', color: '#e09b96' },
  ],
  entries: [
    // Infrastructure
    {
      id: 'kubernetes',
      title: 'Kubernetes',
      quadrant: 'infrastructure',
      ring: 'adopt',
      description: 'Container orchestration platform ของเรา',
      moved: 0,
      url: 'https://kubernetes.io',
    },
    {
      id: 'terraform',
      title: 'Terraform',
      quadrant: 'infrastructure',
      ring: 'adopt',
      description: 'Infrastructure as Code tool หลัก',
      moved: 0,
    },
    {
      id: 'pulumi',
      title: 'Pulumi',
      quadrant: 'infrastructure',
      ring: 'trial',
      description: 'IaC ด้วย general-purpose languages',
      moved: 1, // moving up
    },
    {
      id: 'nomad',
      title: 'HashiCorp Nomad',
      quadrant: 'infrastructure',
      ring: 'hold',
      description: 'เราใช้ Kubernetes แทน',
      moved: -1, // moving down
    },
    // Languages
    {
      id: 'golang',
      title: 'Go',
      quadrant: 'languages',
      ring: 'adopt',
      description: 'ภาษาหลักสำหรับ microservices',
      moved: 0,
    },
    {
      id: 'typescript',
      title: 'TypeScript',
      quadrant: 'languages',
      ring: 'adopt',
      description: 'สำหรับ frontend และ Node.js',
      moved: 0,
    },
    {
      id: 'rust',
      title: 'Rust',
      quadrant: 'languages',
      ring: 'assess',
      description: 'ศึกษาสำหรับ performance-critical services',
      moved: 1,
    },
    // Platforms
    {
      id: 'argocd',
      title: 'ArgoCD',
      quadrant: 'platforms',
      ring: 'adopt',
      description: 'GitOps deployment tool',
      moved: 0,
    },
    {
      id: 'crossplane',
      title: 'Crossplane',
      quadrant: 'platforms',
      ring: 'trial',
      description: 'Kubernetes-native infrastructure provisioning',
      moved: 1,
    },
  ],
};
```

---

## 71.5 Golden Paths

Golden Path คือ "เส้นทางที่ดีที่สุด" ที่ platform team กำหนดไว้ให้นักพัฒนาทำตาม ซึ่งรับประกันว่าเป็น best practice

### Software Templates (Scaffolder)

```bash
# ติดตั้ง plugin
yarn --cwd packages/backend add @backstage/plugin-scaffolder-backend
```

**ตัวอย่าง Template: Go Microservice**

```yaml
# templates/go-microservice/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: go-microservice
  title: Go Microservice
  description: สร้าง Go microservice พร้อม CI/CD และ monitoring
  tags:
    - go
    - microservice
    - recommended
spec:
  owner: platform-team
  type: service

  parameters:
    - title: Service Information
      required:
        - name
        - description
        - owner
      properties:
        name:
          title: Service Name
          type: string
          description: ชื่อ service (lowercase, kebab-case)
          pattern: '^[a-z][a-z0-9-]*$'
          ui:autofocus: true
        description:
          title: Description
          type: string
          description: อธิบาย service นี้ทำอะไร
        owner:
          title: Owner
          type: string
          description: ทีมเจ้าของ
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group

    - title: Infrastructure
      required:
        - environment
        - database
      properties:
        environment:
          title: Initial Environment
          type: string
          enum:
            - development
            - staging
            - production
          default: development
        database:
          title: Database Type
          type: string
          enum:
            - postgresql
            - mysql
            - none
          default: postgresql
        cache:
          title: Enable Redis Cache
          type: boolean
          default: false

    - title: Repository
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com
            allowedOwners:
              - mycompany

  steps:
    - id: fetch-base
      name: Fetch Base Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          database: ${{ parameters.database }}
          cache: ${{ parameters.cache }}

    - id: fetch-docs
      name: Fetch Documentation Template
      action: fetch:plain
      input:
        targetPath: ./docs
        url: ./docs

    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: ['github.com']
        description: ${{ parameters.description }}
        repoUrl: ${{ parameters.repoUrl }}
        defaultBranch: main
        repoVisibility: private
        topics:
          - go
          - microservice

    - id: create-argocd-app
      name: Create ArgoCD Application
      action: argocd:create-resources
      input:
        appName: ${{ parameters.name }}
        argoInstance: production
        projectName: ${{ parameters.owner }}
        namespace: ${{ parameters.owner }}-${{ parameters.environment }}
        repoUrl: ${{ steps['publish'].output.remoteUrl }}
        path: deploy/

    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml

  output:
    links:
      - title: Repository
        url: ${{ steps['publish'].output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}
      - title: ArgoCD Application
        url: https://argocd.mycompany.com/applications/${{ parameters.name }}
```

**Skeleton Template สำหรับ Go Microservice:**

```
templates/go-microservice/skeleton/
├── catalog-info.yaml
├── README.md
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handler/
│   ├── service/
│   └── repository/
├── deploy/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
├── .github/
│   └── workflows/
│       ├── ci.yaml
│       └── cd.yaml
├── Dockerfile
├── go.mod
└── Makefile
```

```go
// skeleton/cmd/server/main.go
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"

    "github.com/mycompany/${{ values.name }}/internal/handler"
    "go.opentelemetry.io/otel"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }

    // Initialize tracing
    tp, err := initTracer("${{ values.name }}")
    if err != nil {
        log.Fatal(err)
    }
    defer tp.Shutdown(context.Background())
    otel.SetTracerProvider(tp)

    // Setup routes
    mux := http.NewServeMux()
    handler.RegisterRoutes(mux)

    fmt.Printf("${{ values.name }} listening on :%s\n", port)
    log.Fatal(http.ListenAndServe(":"+port, mux))
}
```

---

## 71.6 Self-service Infrastructure

### Crossplane สำหรับ Infrastructure Provisioning

```yaml
# XRD (Composite Resource Definition) สำหรับ Database
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xpostgresqlinstances.platform.mycompany.com
spec:
  group: platform.mycompany.com
  names:
    kind: XPostgreSQLInstance
    plural: xpostgresqlinstances
  claimNames:
    kind: PostgreSQLInstance
    plural: postgresqlinstances
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                parameters:
                  type: object
                  properties:
                    storageGB:
                      type: integer
                      minimum: 5
                      maximum: 1000
                    instanceClass:
                      type: string
                      enum:
                        - small
                        - medium
                        - large
                    environment:
                      type: string
                      enum:
                        - development
                        - staging
                        - production
                  required:
                    - storageGB
                    - instanceClass
                    - environment
```

```yaml
# Composition - แปลง claim เป็น AWS RDS
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: postgresql-aws
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: platform.mycompany.com/v1alpha1
    kind: XPostgreSQLInstance
  resources:
    - name: rdsinstance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            region: ap-southeast-1
            engine: postgres
            engineVersion: "14"
            skipFinalSnapshot: true
            publiclyAccessible: false
            autoMinorVersionUpgrade: true
      patches:
        - fromFieldPath: spec.parameters.storageGB
          toFieldPath: spec.forProvider.allocatedStorage
        - fromFieldPath: spec.parameters.instanceClass
          toFieldPath: spec.forProvider.instanceClass
          transforms:
            - type: map
              map:
                small: db.t3.micro
                medium: db.t3.medium
                large: db.r5.large
```

```yaml
# นักพัฒนาขอ Database (claim)
apiVersion: platform.mycompany.com/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: my-app-db
  namespace: team-payments
spec:
  parameters:
    storageGB: 20
    instanceClass: medium
    environment: production
  writeConnectionSecretToRef:
    name: my-app-db-connection
```

### Backstage เชื่อมกับ Crossplane

```typescript
// plugins/backend/src/plugins/scaffolder.ts
import { createTemplateAction } from '@backstage/plugin-scaffolder-backend';
import * as k8s from '@kubernetes/client-node';

export const createDatabaseAction = () => {
  return createTemplateAction({
    id: 'platform:create-database',
    description: 'สร้าง PostgreSQL database ผ่าน Crossplane',
    schema: {
      input: {
        required: ['name', 'namespace', 'storageGB', 'instanceClass'],
        type: 'object',
        properties: {
          name: { type: 'string', title: 'Database Name' },
          namespace: { type: 'string', title: 'Namespace' },
          storageGB: { type: 'number', title: 'Storage (GB)' },
          instanceClass: {
            type: 'string',
            enum: ['small', 'medium', 'large'],
          },
        },
      },
    },
    async handler(ctx) {
      const kc = new k8s.KubeConfig();
      kc.loadFromDefault();
      const customObjectsApi = kc.makeApiClient(k8s.CustomObjectsApi);

      await customObjectsApi.createNamespacedCustomObject(
        'platform.mycompany.com',
        'v1alpha1',
        ctx.input.namespace,
        'postgresqlinstances',
        {
          apiVersion: 'platform.mycompany.com/v1alpha1',
          kind: 'PostgreSQLInstance',
          metadata: {
            name: ctx.input.name,
            namespace: ctx.input.namespace,
          },
          spec: {
            parameters: {
              storageGB: ctx.input.storageGB,
              instanceClass: ctx.input.instanceClass,
              environment: 'production',
            },
            writeConnectionSecretToRef: {
              name: `${ctx.input.name}-connection`,
            },
          },
        },
      );

      ctx.logger.info(`Database ${ctx.input.name} กำลังสร้าง...`);
    },
  });
};
```

---

## 71.7 TechDocs — Documentation as Code

### ตั้งค่า TechDocs

```yaml
# app-config.yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'awsS3'
    awsS3:
      bucketName: ${TECHDOCS_S3_BUCKET}
      region: ap-southeast-1
      credentials:
        roleArn: arn:aws:iam::123456789:role/TechDocsS3Role
```

### mkdocs.yml สำหรับ Service Documentation

```yaml
# mkdocs.yml
site_name: Payment Service
site_description: เอกสารประกอบ Payment Service
repo_url: https://github.com/mycompany/payment-service
docs_dir: docs

nav:
  - หน้าหลัก: index.md
  - สถาปัตยกรรม:
    - ภาพรวม: architecture/overview.md
    - Data Flow: architecture/data-flow.md
  - API Reference:
    - REST API: api/rest.md
    - Events: api/events.md
  - Operations:
    - Deployment: ops/deployment.md
    - Runbook: ops/runbook.md
    - Alerts: ops/alerts.md
  - Development:
    - Getting Started: dev/getting-started.md
    - Testing: dev/testing.md

plugins:
  - techdocs-core
```

---

## 71.8 Backstage Plugins ที่น่าสนใจ

### Kubernetes Plugin

```bash
yarn --cwd packages/app add @backstage/plugin-kubernetes
yarn --cwd packages/backend add @backstage/plugin-kubernetes-backend
```

```yaml
# app-config.yaml
kubernetes:
  serviceLocatorMethod:
    type: 'multiTenant'
  clusterLocatorMethods:
    - type: 'config'
      clusters:
        - url: https://k8s-prod.mycompany.com
          name: production
          authProvider: 'serviceAccount'
          serviceAccountToken: ${K8S_PROD_TOKEN}
          caData: ${K8S_PROD_CA}
        - url: https://k8s-staging.mycompany.com
          name: staging
          authProvider: 'serviceAccount'
          serviceAccountToken: ${K8S_STAGING_TOKEN}
```

### Cost Insights Plugin

```typescript
// packages/app/src/plugins/CostInsights.ts
import { createPlugin } from '@backstage/core-plugin-api';
import { CostInsightsPage } from '@backstage/plugin-cost-insights';

// เพิ่มใน navigation
export const CostInsightsNav = () => (
  <SidebarItem icon={MoneyIcon} to="cost-insights" text="Cost Insights" />
);
```

### GitHub Actions Plugin

```bash
yarn --cwd packages/app add @backstage/plugin-github-actions
```

---

## 71.9 การ Deploy Backstage ใน Production

### Docker Deployment

```dockerfile
# Dockerfile
FROM node:18-bookworm-slim AS build

WORKDIR /app

COPY package.json yarn.lock ./
COPY packages/app/package.json packages/app/
COPY packages/backend/package.json packages/backend/

RUN yarn install --frozen-lockfile --network-timeout 600000

COPY . .

RUN yarn tsc
RUN yarn --cwd packages/backend build

# Production image
FROM node:18-bookworm-slim

WORKDIR /app

ENV NODE_ENV production

COPY --from=build /app/yarn.lock ./
COPY --from=build /app/package.json ./
COPY --from=build /app/packages/backend/dist ./packages/backend/dist
COPY --from=build /app/packages/backend/package.json ./packages/backend/

RUN yarn install --frozen-lockfile --production --network-timeout 600000

COPY app-config.yaml ./
COPY app-config.production.yaml ./

EXPOSE 7007

CMD ["node", "packages/backend", "--config", "app-config.yaml", "--config", "app-config.production.yaml"]
```

### Kubernetes Deployment

```yaml
# k8s/backstage-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      serviceAccountName: backstage
      containers:
        - name: backstage
          image: ghcr.io/mycompany/backstage:latest
          ports:
            - containerPort: 7007
          env:
            - name: POSTGRES_HOST
              valueFrom:
                secretKeyRef:
                  name: backstage-secrets
                  key: POSTGRES_HOST
            - name: POSTGRES_PORT
              value: "5432"
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: backstage-secrets
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: backstage-secrets
                  key: POSTGRES_PASSWORD
            - name: GITHUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: backstage-secrets
                  key: GITHUB_TOKEN
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /healthcheck
              port: 7007
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthcheck
              port: 7007
            initialDelaySeconds: 30
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: backstage
  namespace: platform
spec:
  selector:
    app: backstage
  ports:
    - port: 80
      targetPort: 7007
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backstage
  namespace: platform
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - backstage.mycompany.com
      secretName: backstage-tls
  rules:
    - host: backstage.mycompany.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: backstage
                port:
                  number: 80
```

---

## 71.10 Metrics & Analytics

### Track IDP Usage

```typescript
// Custom analytics plugin
import { analyticsApiRef, useAnalytics } from '@backstage/core-plugin-api';

// ใน component
const analytics = useAnalytics();

// Track template usage
analytics.captureEvent('click', 'template-used', {
  attributes: {
    templateName: template.metadata.name,
    owner: user.email,
  },
});
```

### Dashboard สำหรับ Platform Team

```typescript
// Plugin สำหรับ Platform Metrics
export const PlatformMetricsPage = () => {
  return (
    <Grid container spacing={3}>
      <Grid item xs={12} md={3}>
        <InfoCard title="Services in Catalog">
          <Typography variant="h2">{serviceCount}</Typography>
          <Typography variant="body2">+{newThisMonth} เดือนนี้</Typography>
        </InfoCard>
      </Grid>
      <Grid item xs={12} md={3}>
        <InfoCard title="Templates Used">
          <Typography variant="h2">{templatesUsed}</Typography>
          <Typography variant="body2">ใน 30 วันที่ผ่านมา</Typography>
        </InfoCard>
      </Grid>
      <Grid item xs={12} md={3}>
        <InfoCard title="Teams Onboarded">
          <Typography variant="h2">{teamsOnboarded}</Typography>
        </InfoCard>
      </Grid>
      <Grid item xs={12} md={3}>
        <InfoCard title="Avg Onboarding Time">
          <Typography variant="h2">{avgOnboardingTime}h</Typography>
          <Typography variant="body2">ลดลง {reduction}% จากปีที่แล้ว</Typography>
        </InfoCard>
      </Grid>
    </Grid>
  );
};
```

---

## 71.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ติดตั้ง Backstage พื้นฐาน

**เป้าหมาย:** ติดตั้ง Backstage และสร้าง Software Catalog

```bash
# ขั้นตอน:
# 1. สร้าง Backstage app
npx @backstage/create-app@latest --name mycompany-portal

# 2. เพิ่ม component ในไฟล์ catalog-info.yaml
cat > my-service/catalog-info.yaml << 'EOF'
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: my-first-service
  description: บริการแรกของฉันใน Backstage
  tags:
    - nodejs
    - api
spec:
  type: service
  lifecycle: development
  owner: my-team
EOF

# 3. รัน Backstage
cd mycompany-portal
yarn dev

# 4. Import catalog ผ่าน UI
# ไปที่ http://localhost:3000/catalog-import
# ใส่ path ของ catalog-info.yaml
```

### แบบฝึกหัดที่ 2: สร้าง Software Template

**เป้าหมาย:** สร้าง template สำหรับ Node.js API service

```yaml
# templates/nodejs-api/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: nodejs-api-service
  title: Node.js API Service
  description: Template สำหรับ Node.js Express API
  tags:
    - nodejs
    - express
    - api
spec:
  owner: platform-team
  type: service

  parameters:
    - title: Service Details
      required:
        - name
        - description
      properties:
        name:
          title: Service Name
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
        description:
          title: Description
          type: string
        nodeVersion:
          title: Node.js Version
          type: string
          enum: ['18', '20']
          default: '20'

  steps:
    - id: fetch
      name: Fetch Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          nodeVersion: ${{ parameters.nodeVersion }}

    - id: publish
      name: Publish
      action: publish:github
      input:
        repoUrl: github.com?owner=mycompany&repo=${{ parameters.name }}
        description: ${{ parameters.description }}

    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
```

### แบบฝึกหัดที่ 3: สร้าง Tech Radar

**เป้าหมาย:** กำหนด Tech Radar สำหรับองค์กรของคุณ

```typescript
// exercises/tech-radar-data.ts
// สร้างข้อมูล Tech Radar สำหรับองค์กรของคุณ
// โดยประเมิน:
// - Languages & Frameworks ที่ทีมใช้
// - Infrastructure tools
// - Testing tools
// - Monitoring & Observability

export const myOrgTechRadar = {
  quadrants: [
    { id: 'languages', name: 'Languages & Frameworks' },
    { id: 'infrastructure', name: 'Infrastructure' },
    { id: 'testing', name: 'Testing' },
    { id: 'observability', name: 'Observability' },
  ],
  rings: [
    { id: 'adopt', name: 'ADOPT', color: '#5ba300' },
    { id: 'trial', name: 'TRIAL', color: '#009eb0' },
    { id: 'assess', name: 'ASSESS', color: '#c7ba00' },
    { id: 'hold', name: 'HOLD', color: '#e09b96' },
  ],
  entries: [
    // TODO: เพิ่มเทคโนโลยีของทีมคุณที่นี่
    // แต่ละ entry ต้องมี:
    // - id: unique identifier
    // - title: ชื่อเทคโนโลยี
    // - quadrant: หนึ่งใน quadrant ที่กำหนด
    // - ring: adopt/trial/assess/hold
    // - description: เหตุผลที่เลือก ring นี้
  ],
};
```

### แบบฝึกหัดที่ 4: Self-service Database

**เป้าหมาย:** ใช้ Crossplane สร้าง database ผ่าน Backstage template

```bash
# 1. ติดตั้ง Crossplane
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane \
  --namespace crossplane-system \
  --create-namespace \
  crossplane-stable/crossplane

# 2. Apply XRD และ Composition จากตัวอย่าง

# 3. สร้าง Database Claim
kubectl apply -f - << 'EOF'
apiVersion: platform.mycompany.com/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: test-db
  namespace: default
spec:
  parameters:
    storageGB: 10
    instanceClass: small
    environment: development
  writeConnectionSecretToRef:
    name: test-db-connection
EOF

# 4. ดูสถานะ
kubectl get postgresqlinstance test-db
kubectl get secret test-db-connection -o yaml
```

---

## สรุป

Internal Developer Platform (IDP) เป็นการลงทุนที่สำคัญสำหรับองค์กรที่ต้องการ:

1. **เพิ่มความเร็ว** ในการ onboard developer ใหม่และสร้าง service ใหม่
2. **ลด cognitive load** ของนักพัฒนา
3. **บังคับใช้ best practices** ผ่าน golden paths
4. **เพิ่ม visibility** ของ software ecosystem
5. **Self-service** infrastructure และ tooling

Backstage.io เป็นเครื่องมือที่ยืดหยุ่นและมีระบบ plugin ที่แข็งแกร่ง สามารถปรับแต่งได้ตามความต้องการขององค์กร

### ขั้นตอนถัดไป

- ศึกษา [Part 72: Platform Engineering](./part-72-platform-engineering.md)
- ดู Backstage plugin marketplace: https://backstage.io/plugins
- เข้าร่วม Backstage community: https://discord.com/invite/backstage-687207715902193673

---

*หมายเหตุ: เนื้อหาในบทนี้ใช้ Backstage version 1.x โปรดตรวจสอบเอกสารล่าสุดที่ https://backstage.io/docs*

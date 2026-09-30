# Part 83: Building Internal CI/CD Platform

## บทนำ

Internal Developer Platform (IDP) คือ product ที่ Platform team สร้างขึ้นเพื่อให้ Developer teams ใช้ deploy applications ได้โดยไม่ต้องรู้รายละเอียดของ infrastructure ลึกๆ บทนี้จะครอบคลุมการออกแบบ สร้าง และ operate Internal CI/CD Platform

## สารบัญ

1. [Platform Design Principles](#design-principles)
2. [Self-Service Portal](#self-service-portal)
3. [Pipeline Templates และ Abstractions](#pipeline-templates)
4. [Infrastructure Abstraction Layer](#infrastructure-abstraction)
5. [Developer Documentation Strategy](#documentation)
6. [Platform Adoption Strategy](#adoption)
7. [Platform Operations และ SLOs](#operations)
8. [Case Studies](#case-studies)
9. [แบบฝึกหัด](#exercises)

---

## 1. Platform Design Principles {#design-principles}

### Platform Engineering Philosophy

```
Platform Engineering คือการ apply Product Thinking
ไปยัง Internal Developer Tools

ไม่ใช่:
"เราจะ deploy Kubernetes เพราะมัน popular"

แต่คือ:
"เราจะทำให้ Developer deploy application ได้ใน 10 นาที
 โดยไม่ต้องรู้เรื่อง Kubernetes"
```

### Core Design Principles

**1. Developer-First Design**
```
ทุกการตัดสินใจ platform ต้องถามว่า:
"มันทำให้ Developer ทำงานได้ง่ายขึ้นไหม?"

ไม่ใช่:
"มันเป็น best practice ไหม?"
หรือ
"Infrastructure team ชอบไหม?"
```

**2. Paved Road (ไม่ใช่ Walled Garden)**
```
Paved Road:
- มี default path ที่ดี (fast, secure, reliable)
- Developer เลือกออกจาก path ได้ถ้าต้องการ
- Platform ให้ tools ไม่ใช่ locks

Walled Garden (สิ่งที่ต้องหลีกเลี่ยง):
- บังคับ Developer ใช้ tools ที่ platform กำหนด
- ไม่มีทางออก
- ทำให้ Developer frustrated
```

**3. Self-Service by Default**
```
Level 0: ต้องขอ Ops team ทุกอย่าง
Level 1: มี form ให้กรอก แต่ต้องรอ Ops approve
Level 2: Automated แต่ต้องติดต่อ Platform team
Level 3: 100% self-service, no human in loop

เป้าหมาย: Level 3 สำหรับ common tasks
```

**4. Cognitive Load Reduction**
```
Platform ต้องลด cognitive load ของ Developer

สิ่ง Developer ไม่ควรต้องรู้:
- Kubernetes YAML syntax (ทั้งหมด)
- Terraform modules ซับซ้อน
- Networking details
- Security certificate management
- Load balancer configuration

สิ่ง Developer ควรรู้:
- Application specification (runtime, resources, envs)
- Deployment strategy
- Business logic
```

### Platform Architecture

```
┌────────────────────────────────────────────────────────────┐
│                   Developer Experience Layer                │
│                                                            │
│  CLI Tool    │  Web Portal  │  API  │  IDE Plugins        │
└──────────────┴──────────────┴───────┴─────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                   Platform Core                             │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │   Service     │  │   Pipeline   │  │   Environment    │ │
│  │   Registry   │  │   Orchestr.  │  │   Manager        │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │   Secret     │  │   Artifact   │  │   Observability  │ │
│  │   Manager    │  │   Store      │  │   Setup          │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                   Infrastructure Layer                      │
│                                                            │
│  Kubernetes │ Terraform │ Vault │ Prometheus │ ArgoCD      │
└────────────────────────────────────────────────────────────┘
```

---

## 2. Self-Service Portal {#self-service-portal}

### Backstage.io - Foundation สำหรับ Internal Platform

Backstage คือ open-source platform จาก Spotify ที่ใช้สร้าง Developer Portal:

```
Backstage Components:
1. Software Catalog - รายการทุก services
2. TechDocs - Documentation
3. Software Templates - Service scaffolding
4. Plugins - ขยาย functionality
```

### Service Catalog Design

```yaml
# catalog-info.yaml
# ทุก service ต้องมีไฟล์นี้

apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: Handles all payment processing
  
  annotations:
    # ลิงค์ไปยัง resources ต่างๆ
    github.com/project-slug: company/payment-service
    backstage.io/techdocs-ref: dir:.
    prometheus.io/alert: payment-service-alerts
    pagerduty.com/service-id: "12345"
    
  tags:
    - payment
    - critical
    - pci-scope
    
  links:
    - url: https://payment.company.com
      title: Production
      icon: web
    - url: https://runbook.internal/payment-service
      title: Runbook
      icon: book
    - url: https://dashboard.company.com/d/payment
      title: Dashboard
      icon: dashboard

spec:
  type: service
  lifecycle: production
  owner: team-payments
  
  dependsOn:
    - component:database-service
    - resource:payment-database
    - component:fraud-detection-service
  
  providesApis:
    - payment-api
    
  consumesApis:
    - fraud-detection-api
    - notification-api
```

### Software Templates

```yaml
# templates/new-service/template.yaml
# Template สำหรับสร้าง service ใหม่

apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: create-backend-service
  title: Create Backend Service
  description: สร้าง backend service ใหม่พร้อม CI/CD pipeline

spec:
  owner: platform-team
  type: service
  
  parameters:
    - title: Service Information
      required: [name, description, owner]
      properties:
        name:
          title: Service Name
          type: string
          pattern: '^[a-z][a-z0-9-]{2,50}$'
          description: lowercase, alphanumeric, hyphens only
          
        description:
          title: Description
          type: string
          
        owner:
          title: Owner Team
          type: string
          ui:field: OwnerPicker
          
        language:
          title: Programming Language
          type: string
          enum: [go, python, typescript, java]
          default: go
          
        service_type:
          title: Service Type
          type: string
          enum: [api, worker, scheduler, event-processor]
          default: api
    
    - title: Infrastructure
      properties:
        resources:
          title: Resource Allocation
          type: string
          enum: [small, medium, large, xlarge]
          default: small
          description: >
            small: 0.1 CPU, 128Mi RAM
            medium: 0.5 CPU, 512Mi RAM
            large: 1 CPU, 1Gi RAM
            xlarge: 2 CPU, 2Gi RAM
        
        databases:
          title: Required Databases
          type: array
          items:
            type: string
            enum: [postgres, redis, mongodb]
          
        pci_scope:
          title: PCI Scope
          type: boolean
          default: false
          description: Is this service in PCI cardholder data scope?
  
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
          language: ${{ parameters.language }}
          service_type: ${{ parameters.service_type }}
          resources: ${{ parameters.resources }}
    
    - id: publish
      name: Create Repository
      action: publish:github
      input:
        allowedHosts: ['github.com']
        description: ${{ parameters.description }}
        repoUrl: github.com/company/${{ parameters.name }}
        defaultBranch: main
        repoVisibility: private
        
    - id: configure-pipeline
      name: Configure CI/CD Pipeline
      action: github:actions:dispatch
      input:
        workflowId: setup-new-service.yml
        repoUrl: github.com/company/platform-tools
        branchOrTagName: main
        workflowInputs:
          service_name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          language: ${{ parameters.language }}
          pci_scope: ${{ parameters.pci_scope }}
    
    - id: provision-infrastructure
      name: Provision Infrastructure
      action: http:backstage:request
      input:
        method: POST
        path: /api/platform/provision
        body:
          service_name: ${{ parameters.name }}
          resources: ${{ parameters.resources }}
          databases: ${{ parameters.databases }}
    
    - id: register-catalog
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
        url: ${{ steps['register-catalog'].output.entityRef }}
      - title: CI/CD Pipeline
        url: https://github.com/company/${{ parameters.name }}/actions
```

---

## 3. Pipeline Templates และ Abstractions {#pipeline-templates}

### Pipeline Abstraction Design

แทนที่ Developer จะต้องเขียน YAML ซับซ้อน เราสร้าง higher-level abstractions:

```yaml
# pipeline.yaml (สิ่งที่ Developer เขียน)
# Simple, declarative spec

apiVersion: platform.company.com/v1
kind: Pipeline
metadata:
  name: payment-service

spec:
  language: go
  type: api-service
  
  deploy:
    environments:
      - dev
      - staging
      - production
    
    strategy:
      type: canary
      analysis:
        interval: 5m
        threshold: 5  # error rate threshold
  
  quality:
    test_coverage: 80
    security_scan: true
    
  integrations:
    datadog: true
    pagerduty: payment-service-oncall
```

```yaml
# Platform แปลงเป็น GitHub Actions workflow จริงๆ
# Developer ไม่ต้องเห็นหรือเข้าใจส่วนนี้

name: payment-service CI/CD

on:
  push:
    branches: [main, 'feature/**']
  pull_request:

env:
  SERVICE_NAME: payment-service
  LANGUAGE: go

jobs:
  # ... Generated from template ...
  build:
    runs-on: ubuntu-latest
    container: golang:1.21
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: go build ./...
  
  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Test with Coverage
        run: |
          go test -coverprofile=coverage.out ./...
          coverage=$(go tool cover -func coverage.out | tail -1 | awk '{print $3}' | tr -d '%')
          if (( $(echo "$coverage < 80" | bc -l) )); then
            echo "Coverage ${coverage}% < 80%"
            exit 1
          fi
  # ... more generated jobs ...
```

### Template Engine

```python
# platform/template_engine.py

import yaml
from jinja2 import Environment, FileSystemLoader

class PipelineTemplateEngine:
    """แปลง high-level pipeline spec เป็น CI/CD workflow"""
    
    def __init__(self, templates_dir: str):
        self.env = Environment(
            loader=FileSystemLoader(templates_dir),
            trim_blocks=True,
            lstrip_blocks=True
        )
    
    def render(self, pipeline_spec: dict) -> str:
        """แปลง pipeline spec เป็น GitHub Actions YAML"""
        
        # Validate spec
        self._validate_spec(pipeline_spec)
        
        # เลือก template ตาม language
        template_name = f"{pipeline_spec['spec']['language']}/workflow.yml.j2"
        template = self.env.get_template(template_name)
        
        # Render template
        context = self._build_context(pipeline_spec)
        rendered = template.render(context)
        
        # Validate rendered YAML
        yaml.safe_load(rendered)
        
        return rendered
    
    def _build_context(self, spec: dict) -> dict:
        """สร้าง context สำหรับ template rendering"""
        
        deployment_spec = spec['spec'].get('deploy', {})
        quality_spec = spec['spec'].get('quality', {})
        
        return {
            "service_name": spec['metadata']['name'],
            "language": spec['spec']['language'],
            "service_type": spec['spec'].get('type', 'api-service'),
            
            # Deployment
            "environments": deployment_spec.get('environments', ['dev', 'staging', 'production']),
            "strategy": deployment_spec.get('strategy', {'type': 'rolling'}),
            
            # Quality
            "test_coverage": quality_spec.get('test_coverage', 80),
            "security_scan": quality_spec.get('security_scan', True),
            
            # Computed values
            "has_production": 'production' in deployment_spec.get('environments', []),
            "is_canary": deployment_spec.get('strategy', {}).get('type') == 'canary',
            "needs_approval": 'production' in deployment_spec.get('environments', []),
        }
```

### Reusable Action Library

```yaml
# .github/actions/platform-deploy/action.yml
# Reusable deployment action ที่ teams ใช้

name: Platform Deploy
description: Deploy service using platform standards

inputs:
  service-name:
    required: true
  environment:
    required: true
  image-tag:
    required: true
  strategy:
    required: false
    default: 'rolling'
  canary-percentage:
    required: false
    default: '10'

outputs:
  deployment-url:
    description: URL ของ deployed service
  deployment-id:
    description: ID สำหรับ tracking

runs:
  using: 'composite'
  steps:
    - name: Validate Inputs
      shell: bash
      run: |
        # ตรวจสอบ environment name ถูกต้อง
        if [[ ! "${{ inputs.environment }}" =~ ^(dev|staging|production)$ ]]; then
          echo "Invalid environment: ${{ inputs.environment }}"
          exit 1
        fi
    
    - name: Get Deployment Config
      id: config
      shell: bash
      run: |
        # Load deployment config จาก platform
        CONFIG=$(curl -s \
          "https://platform.internal/api/services/${{ inputs.service-name }}/deploy-config?env=${{ inputs.environment }}" \
          -H "Authorization: Bearer ${{ env.PLATFORM_TOKEN }}")
        
        echo "namespace=$(echo $CONFIG | jq -r .namespace)" >> $GITHUB_OUTPUT
        echo "cluster=$(echo $CONFIG | jq -r .cluster)" >> $GITHUB_OUTPUT
    
    - name: Deploy
      shell: bash
      run: |
        if [ "${{ inputs.strategy }}" = "canary" ]; then
          # Canary deployment
          kubectl argo rollouts set image \
            rollout/${{ inputs.service-name }} \
            ${{ inputs.service-name }}=${{ inputs.image-tag }} \
            --namespace ${{ steps.config.outputs.namespace }}
        else
          # Rolling deployment
          kubectl set image \
            deployment/${{ inputs.service-name }} \
            ${{ inputs.service-name }}=${{ inputs.image-tag }} \
            --namespace ${{ steps.config.outputs.namespace }}
        fi
    
    - name: Wait for Rollout
      shell: bash
      run: |
        kubectl rollout status \
          deployment/${{ inputs.service-name }} \
          --namespace ${{ steps.config.outputs.namespace }} \
          --timeout=10m
    
    - name: Get Deployment URL
      id: get-url
      shell: bash
      run: |
        URL=$(kubectl get ingress \
          ${{ inputs.service-name }} \
          -n ${{ steps.config.outputs.namespace }} \
          -o jsonpath='{.spec.rules[0].host}')
        echo "deployment-url=https://${URL}" >> $GITHUB_OUTPUT
```

---

## 4. Infrastructure Abstraction Layer {#infrastructure-abstraction}

### Service Specification API

```yaml
# service-spec.yaml
# Developer กำหนด "what" ไม่ใช่ "how"

apiVersion: platform.company.com/v1
kind: Service
metadata:
  name: user-service
  team: team-identity
  
spec:
  runtime:
    language: python
    version: "3.11"
    
  resources:
    preset: medium  # Platform จัดการ CPU/Memory
    
  scaling:
    min: 2
    max: 10
    target_cpu: 70
    
  networking:
    port: 8080
    public: false  # internal service only
    
  storage:
    databases:
      - type: postgresql
        size: medium
        ha: true
    caches:
      - type: redis
        size: small
    
  secrets:
    - name: DATABASE_URL
      from: vault://team-identity/db/user-service
    - name: JWT_SECRET
      from: vault://team-identity/auth/jwt
    
  observability:
    metrics: true
    tracing: true
    log_level: info
    
  dependencies:
    - service: email-service
      critical: false
    - service: auth-service
      critical: true
```

```python
# platform/service_provisioner.py

class ServiceProvisioner:
    """แปลง Service Spec เป็น infrastructure resources"""
    
    def provision(self, spec: dict) -> dict:
        """Provision ทุก resources ที่ service ต้องการ"""
        
        results = {
            "service_name": spec['metadata']['name'],
            "resources_created": []
        }
        
        # 1. ตั้งค่า Kubernetes resources
        k8s_resources = self._provision_kubernetes(spec)
        results["resources_created"].extend(k8s_resources)
        
        # 2. ตั้งค่า databases
        if 'databases' in spec['spec'].get('storage', {}):
            db_resources = self._provision_databases(spec)
            results["resources_created"].extend(db_resources)
        
        # 3. ตั้งค่า caches
        if 'caches' in spec['spec'].get('storage', {}):
            cache_resources = self._provision_caches(spec)
            results["resources_created"].extend(cache_resources)
        
        # 4. ตั้งค่า secrets
        if 'secrets' in spec['spec']:
            secret_resources = self._configure_secrets(spec)
            results["resources_created"].extend(secret_resources)
        
        # 5. ตั้งค่า observability
        obs_resources = self._configure_observability(spec)
        results["resources_created"].extend(obs_resources)
        
        # 6. Register ใน service catalog
        self._register_in_catalog(spec)
        
        return results
    
    def _provision_kubernetes(self, spec: dict) -> list:
        """สร้าง Kubernetes resources"""
        
        resources = self._get_resource_preset(
            spec['spec']['resources']['preset']
        )
        
        k8s_manifests = {
            "Deployment": self._create_deployment_manifest(spec, resources),
            "Service": self._create_service_manifest(spec),
            "HPA": self._create_hpa_manifest(spec),
        }
        
        if not spec['spec']['networking'].get('public', True):
            # Internal service: no Ingress needed
            pass
        else:
            k8s_manifests["Ingress"] = self._create_ingress_manifest(spec)
        
        # Apply manifests
        for resource_type, manifest in k8s_manifests.items():
            self.kubectl.apply(manifest)
        
        return [f"kubernetes/{resource_type}/{spec['metadata']['name']}" 
                for resource_type in k8s_manifests.keys()]
    
    def _get_resource_preset(self, preset: str) -> dict:
        """แปลง preset เป็น actual resource requests/limits"""
        
        presets = {
            "small": {
                "requests": {"cpu": "100m", "memory": "128Mi"},
                "limits": {"cpu": "500m", "memory": "256Mi"}
            },
            "medium": {
                "requests": {"cpu": "500m", "memory": "512Mi"},
                "limits": {"cpu": "1000m", "memory": "1Gi"}
            },
            "large": {
                "requests": {"cpu": "1000m", "memory": "1Gi"},
                "limits": {"cpu": "2000m", "memory": "2Gi"}
            },
            "xlarge": {
                "requests": {"cpu": "2000m", "memory": "2Gi"},
                "limits": {"cpu": "4000m", "memory": "4Gi"}
            }
        }
        
        return presets.get(preset, presets["small"])
```

---

## 5. Developer Documentation Strategy {#documentation}

### Documentation Architecture

```
Platform Documentation ที่ดี:

1. Conceptual (WHY)
   - Architecture overview
   - Design decisions
   - Philosophy

2. Guides (HOW)
   - Getting started
   - Common tasks
   - Tutorials

3. Reference (WHAT)
   - API reference
   - Configuration reference
   - CLI reference

4. Troubleshooting
   - Common issues
   - Debugging guides
   - FAQ
```

### TechDocs Integration

```yaml
# docs/index.md
# Documentation ที่ถูก publish ไปยัง Backstage

# Payment Service

## Overview
Payment Service จัดการการชำระเงินทั้งหมดของระบบ

## Architecture
[Diagram here]

## Getting Started

### Prerequisites
- Docker Desktop installed
- Platform CLI (`platform-cli`) installed
- Access to dev environment

### Development Setup
```bash
# Clone repository
git clone https://github.com/company/payment-service

# Install dependencies
make install

# Start local development
make dev

# Run tests
make test
```

### Deployment

```bash
# Deploy to dev (automatic on push)
git push origin feature/my-feature

# Deploy to staging (automatic on merge to main)
git checkout main && git merge feature/my-feature

# Deploy to production (requires approval)
platform-cli deploy --env production --ticket CHG-2024-0001
```

### API Reference
See [API Documentation](./api.md)

### Troubleshooting
See [Troubleshooting Guide](./troubleshooting.md)
```

### Interactive Documentation

```typescript
// platform-portal/src/components/DeploymentGuide.tsx
// Interactive guide ที่ปรับตาม service ของ user

import React, { useState, useEffect } from 'react';

interface DeploymentGuideProps {
  serviceName: string;
}

export const DeploymentGuide: React.FC<DeploymentGuideProps> = ({ serviceName }) => {
  const [service, setService] = useState(null);
  const [selectedEnv, setSelectedEnv] = useState('dev');
  
  useEffect(() => {
    // Load service config
    fetch(`/api/catalog/services/${serviceName}`)
      .then(r => r.json())
      .then(setService);
  }, [serviceName]);
  
  return (
    <div className="deployment-guide">
      <h2>How to Deploy {serviceName}</h2>
      
      <div className="env-selector">
        {['dev', 'staging', 'production'].map(env => (
          <button 
            key={env}
            onClick={() => setSelectedEnv(env)}
            className={selectedEnv === env ? 'active' : ''}
          >
            {env}
          </button>
        ))}
      </div>
      
      {selectedEnv === 'dev' && (
        <div className="guide-steps">
          <h3>Deploy to Development</h3>
          <p>Development deployments happen automatically when you push to your branch.</p>
          <CodeBlock language="bash">
            {`# Push your changes
git push origin feature/${serviceName}-my-feature

# Pipeline will run automatically
# Status: ${service?.environments?.dev?.lastDeployment?.status || 'unknown'}`}
          </CodeBlock>
        </div>
      )}
      
      {selectedEnv === 'production' && (
        <div className="guide-steps">
          <h3>Deploy to Production</h3>
          
          {service?.pci_scope && (
            <Alert type="warning">
              This service is in PCI scope. Additional approvals required.
            </Alert>
          )}
          
          <Steps>
            <Step number={1}>
              <p>Create a change ticket</p>
              <Button onClick={() => openServiceNow(serviceName)}>
                Create Change Ticket
              </Button>
            </Step>
            
            <Step number={2}>
              <p>Get approval from Tech Lead and Release Manager</p>
            </Step>
            
            <Step number={3}>
              <p>Trigger deployment using CLI</p>
              <CodeBlock language="bash">
                {`platform-cli deploy \\
  --service ${serviceName} \\
  --env production \\
  --ticket CHG-XXXX`}
              </CodeBlock>
            </Step>
          </Steps>
        </div>
      )}
    </div>
  );
};
```

---

## 6. Platform Adoption Strategy {#adoption}

### Developer Adoption Funnel

```
Awareness → Trial → Adoption → Advocacy

1. AWARENESS
   - Developer blog posts
   - Internal tech talks
   - Slack announcements
   
2. TRIAL
   - Easy onboarding (< 30 min)
   - Sandbox environment
   - Guided tutorials
   
3. ADOPTION
   - Migrate existing services
   - Support during migration
   - Success stories
   
4. ADVOCACY
   - Developer champions program
   - Recognition for early adopters
   - Community building
```

### Onboarding Timeline

```
Week 1: Discovery
  Day 1-2: Kickoff meeting, platform demo
  Day 3-5: Team assessment, identify pilot service

Week 2-3: Pilot Migration
  Day 8-10: Platform setup, account provisioning
  Day 11-15: Migrate pilot service to platform
  Day 16-21: Stabilization, team training

Week 4: Scale-up
  Day 22-28: Migrate remaining services
  Day 28: Review and retrospective
```

### Developer Champions Program

```python
# platform/champions.py

CHAMPION_BENEFITS = {
    "recognition": [
        "Featured on Platform Portal homepage",
        "Company blog post",
        "Internal award"
    ],
    "access": [
        "Early access to new features",
        "Direct Slack channel with Platform team",
        "Monthly 1:1 with Platform PM"
    ],
    "responsibilities": [
        "Help teammates adopt platform",
        "Report bugs and feedback",
        "Participate in user research"
    ]
}

CHAMPION_CRITERIA = {
    "services_migrated": 3,
    "team_members_trained": 5,
    "bugs_reported": 2,
    "satisfaction_score": 8.0  # out of 10
}
```

### Adoption Metrics

```python
# metrics/adoption.py

def calculate_adoption_metrics(period: str) -> dict:
    """วัด adoption ของ platform"""
    
    return {
        # Usage metrics
        "total_services": count_total_services(),
        "services_on_platform": count_platform_services(),
        "adoption_rate": calculate_adoption_rate(),
        
        # Developer experience metrics
        "avg_onboarding_time": avg_onboarding_minutes(),
        "developer_satisfaction": avg_satisfaction_score(),
        "support_tickets": count_support_tickets(period),
        
        # Business metrics
        "time_to_first_deploy": avg_time_to_first_deploy(),
        "deployment_frequency": avg_deployments_per_day(),
        "pipeline_success_rate": calculate_success_rate(),
        
        # Growth metrics
        "new_services_this_week": new_services_count(period),
        "active_developers": count_active_developers(period)
    }
```

---

## 7. Platform Operations และ SLOs {#operations}

### Platform SLOs

```yaml
# slos.yaml
# Platform SLOs ที่ต้องรักษาไว้

slos:
  availability:
    target: 99.9%  # 8.7 hours downtime/year
    window: 30-days
    
  pipeline_success_rate:
    target: 95%
    window: 7-days
    
  p50_pipeline_duration:
    target: 10m
    window: 7-days
    
  p95_pipeline_duration:
    target: 30m
    window: 7-days
    
  onboarding_time:
    target: 60m  # < 1 hour to deploy first service
    window: 30-days
    
  support_response_time:
    target: 4h  # business hours
    window: 30-days
```

### Platform Monitoring

```yaml
# monitoring/platform-alerts.yaml

groups:
  - name: platform.alerts
    rules:
      - alert: PlatformAvailabilityLow
        expr: |
          (1 - (sum(rate(pipeline_runs_failed_total[5m])) / 
                sum(rate(pipeline_runs_total[5m])))) < 0.999
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Platform availability dropped below 99.9%"
          runbook: https://runbook.internal/platform-availability
      
      - alert: PipelineDurationHigh
        expr: |
          histogram_quantile(0.95, rate(pipeline_duration_seconds_bucket[5m])) > 1800
        for: 10m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "P95 pipeline duration > 30 minutes"
          
      - alert: OnboardingTimeHigh
        expr: avg(onboarding_duration_minutes) > 90
        for: 1h
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Average onboarding time > 90 minutes"
```

---

## 8. Case Studies {#case-studies}

### Case Study 1: Spotify - Backstage Origin Story

**บริบท:**
- 2,000+ engineers
- 1,000+ microservices
- No central way to find information about services

**ปัญหา:**
```
"New engineer ใช้เวลา 2 สัปดาห์แรกหาว่า:
- Service ที่ต้องการอยู่ที่ไหน
- ใครเป็น owner
- Documentation อยู่ที่ไหน
- ทำไถึง production อย่างไร"
```

**โซลูชัน:** Backstage
```
- Unified Software Catalog: รู้จัก services ทั้งหมด
- TechDocs: Documentation ใน catalog
- Software Templates: สร้าง service ใหม่ในนาที
- Plugins ecosystem: ขยายได้
```

**ผลลัพธ์:**
```
Onboarding time: 2 weeks → 2 days
Service discovery: hours → seconds
Developer satisfaction: ปรับปรุงอย่างมีนัยสำคัญ
Open-sourced ในปี 2020 → used by 1000+ companies
```

### Case Study 2: Fintech Startup Scaling from 20 to 200 Developers

**Timeline:**
```
Year 1 (20 developers):
- GitHub Actions สำหรับ CI
- Manual deployment scripts
- Works fine

Year 2 (80 developers):
- Multiple teams, different tools
- Onboarding ยาก
- "We need a platform!"

Year 3 (200 developers):
- Internal platform launch
- Backstage + custom CLI
- Golden Path implemented
```

**ผลลัพธ์หลัง Platform Launch:**
```
Time to first production deploy: 3 days → 4 hours
Developer onboarding: 2 weeks → 3 days
Platform adoption rate: 85% ใน 6 เดือน
Incident related to "wrong config": -70%
Infrastructure cost (per service): -30%
```

---

## 9. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Service Catalog Design

**งาน:** ออกแบบ Service Catalog สำหรับองค์กรสมมติ
- 50 services, 10 teams
- 3 environments (dev, staging, prod)
- Mixed technology stack

**Deliverables:**
1. catalog-info.yaml template
2. Documentation structure
3. Service ownership model
4. Dependency mapping

### แบบฝึกหัดที่ 2: Pipeline Template Engine

**งาน:** สร้าง simple pipeline template engine

```python
# template_engine.py
# TODO: Implement

def render_pipeline(spec: dict) -> str:
    """
    Input:
    {
        "name": "my-service",
        "language": "python",
        "test_coverage": 80,
        "deploy_environments": ["dev", "staging", "production"]
    }
    
    Output: GitHub Actions YAML string
    """
    pass

# Test
spec = {
    "name": "user-service",
    "language": "python",
    "test_coverage": 80,
    "deploy_environments": ["dev", "staging", "production"]
}

result = render_pipeline(spec)
print(result)
```

### แบบฝึกหัดที่ 3: Platform Adoption Plan

**งาน:** สร้าง 90-day adoption plan สำหรับ:
- 5 teams, 50 developers
- 30 existing services ที่ต้อง migrate
- 3 different technology stacks

**Template:**
```
Week 1-2: Discovery
  - [ ] Team assessments complete
  - [ ] Pilot teams identified
  - [ ] Current state documented

Week 3-6: Foundation
  - [ ] Platform environment ready
  - [ ] Pilot teams onboarded
  - [ ] Documentation written

Week 7-12: Scale
  - [ ] All teams onboarded
  - [ ] 80% services migrated
  - [ ] Champions identified
```

### แบบฝึกหัดที่ 4: SLO Definition

**งาน:** กำหนด SLOs สำหรับ Internal CI/CD Platform

1. ระบุ user journeys หลัก 5 อย่าง
2. กำหนด SLI สำหรับแต่ละ journey
3. กำหนด SLO target
4. สร้าง alerting rules

---

## สรุป

การสร้าง Internal CI/CD Platform ที่ดีต้องการ:

1. **Product Mindset** - ปฏิบัติต่อ Developer เหมือน Customer
2. **Self-Service** - ลด human-in-the-loop
3. **Abstraction Layers** - Developer ไม่ควรต้องรู้ infrastructure detail
4. **Great Documentation** - Platform ที่ดี + Documentation แย่ = Platform ที่ไม่ถูกใช้
5. **Adoption Strategy** - Technical solution ไม่พอ ต้องมี change management
6. **Clear SLOs** - Platform team รับผิดชอบต่อ developer experience

## อ่านเพิ่มเติม

- Backstage.io Documentation: https://backstage.io/docs
- "Platform Engineering" book by Camille Fournier
- Internal Developer Platform: https://internaldeveloperplatform.org
- CNCF Platforms White Paper: https://tag-app-delivery.cncf.io/whitepapers/platforms/

---

*Part 83 จาก 100 | CI/CD Mastery Course*

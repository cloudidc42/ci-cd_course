# Part 59: Multi-Cloud CI/CD

## บทนำ

Multi-Cloud Strategy คือการใช้บริการ Cloud จากหลาย providers พร้อมกัน แทนที่จะพึ่งพา provider เดียว ในโลก CI/CD สมัยใหม่ หลายองค์กรเลือกใช้ AWS สำหรับ primary workloads, GCP สำหรับ ML/AI และ Azure สำหรับ enterprise integrations

บทนี้จะครอบคลุม:
- Multi-Cloud Challenges
- Cloud-Agnostic Tools
- Terraform Multi-Cloud
- Cross-Cloud Deployments
- Cost Considerations
- Workshop และ Exercises

---

## 1. Multi-Cloud Challenges

### 1.1 ความท้าทายหลัก

```
Challenge 1: Identity & Access Management
- AWS ใช้ IAM
- GCP ใช้ IAM & Service Accounts
- Azure ใช้ Azure AD & Managed Identities
→ ต้องมี unified identity management

Challenge 2: Networking
- แต่ละ cloud มี VPC/VNet architecture ต่างกัน
- Cross-cloud connectivity ซับซ้อน
- Latency ระหว่าง clouds

Challenge 3: Storage
- S3 vs GCS vs Azure Blob Storage
- Different APIs
- Cross-cloud data transfer costs

Challenge 4: Container Registry
- ECR vs GCR/Artifact Registry vs ACR
- Authentication ต่างกัน
- Image pull ข้ามระหว่าง clouds

Challenge 5: Secrets Management
- AWS Secrets Manager vs GCP Secret Manager vs Azure Key Vault
- ต้องมี abstraction layer

Challenge 6: Monitoring
- CloudWatch vs Cloud Monitoring vs Azure Monitor
- ต้องรวม metrics จากหลาย sources
```

### 1.2 Multi-Cloud Architecture Pattern

```
┌──────────────────────────────────────────────────────────────┐
│  Multi-Cloud Architecture                                    │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │     AWS      │  │     GCP      │  │     Azure        │  │
│  │              │  │              │  │                  │  │
│  │  Primary     │  │  ML Workload │  │  Enterprise      │  │
│  │  Compute     │  │  BigQuery    │  │  Active Dir      │  │
│  │  RDS         │  │  Vertex AI   │  │  Office 365      │  │
│  └──────┬───────┘  └──────┬───────┘  └───────┬──────────┘  │
│         │                 │                   │             │
│  ┌──────┴─────────────────┴───────────────────┴──────────┐  │
│  │  Cloud-Agnostic Layer                                  │  │
│  │  ├── Kubernetes (EKS/GKE/AKS)                         │  │
│  │  ├── Terraform                                        │  │
│  │  ├── Vault (Secrets)                                  │  │
│  │  ├── Prometheus + Grafana (Monitoring)                │  │
│  │  └── GitHub Actions (CI/CD)                          │  │
│  └─────────────────────────────────────────────────────-─┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Cloud-Agnostic Tools

### 2.1 Terraform Multi-Cloud

```hcl
# terraform/providers.tf
# กำหนด providers สำหรับหลาย clouds

terraform {
  required_version = ">= 1.7.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
  
  # Remote state ใน S3
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "multi-cloud/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# AWS Provider
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# GCP Provider
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}

# Azure Provider
provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
  }
  subscription_id = var.azure_subscription_id
}
```

### 2.2 Multi-Cloud Variables

```hcl
# terraform/variables.tf

variable "project_name" {
  description = "Project name สำหรับ resource naming"
  type        = string
  default     = "myapp"
}

variable "environment" {
  description = "Environment (production/staging/development)"
  type        = string
  validation {
    condition     = contains(["production", "staging", "development"], var.environment)
    error_message = "Environment ต้องเป็น: production, staging, หรือ development"
  }
}

# AWS Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-southeast-1"
}

variable "aws_account_id" {
  description = "AWS Account ID"
  type        = string
  sensitive   = true
}

# GCP Variables
variable "gcp_project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "gcp_region" {
  description = "GCP region"
  type        = string
  default     = "asia-southeast1"
}

# Azure Variables
variable "azure_subscription_id" {
  description = "Azure Subscription ID"
  type        = string
  sensitive   = true
}

variable "azure_location" {
  description = "Azure location"
  type        = string
  default     = "Southeast Asia"
}

# Multi-cloud Configuration
variable "cloud_providers" {
  description = "Cloud providers ที่จะใช้"
  type        = list(string)
  default     = ["aws"]  # เริ่มต้นด้วย AWS เท่านั้น
  
  validation {
    condition = alltrue([
      for provider in var.cloud_providers :
      contains(["aws", "gcp", "azure"], provider)
    ])
    error_message = "Cloud providers ต้องเป็น: aws, gcp, หรือ azure"
  }
}
```

### 2.3 Multi-Cloud Kubernetes

```hcl
# terraform/kubernetes/main.tf
# สร้าง Kubernetes cluster ใน multiple clouds

locals {
  cluster_name = "${var.project_name}-${var.environment}"
}

# AWS EKS
module "eks" {
  count  = contains(var.cloud_providers, "aws") ? 1 : 0
  source = "./modules/aws-eks"
  
  cluster_name    = "${local.cluster_name}-aws"
  aws_region      = var.aws_region
  node_count      = var.environment == "production" ? 3 : 1
  node_type       = var.environment == "production" ? "m5.xlarge" : "t3.medium"
  
  tags = {
    CloudProvider = "aws"
    Project       = var.project_name
    Environment   = var.environment
  }
}

# GCP GKE
module "gke" {
  count  = contains(var.cloud_providers, "gcp") ? 1 : 0
  source = "./modules/gcp-gke"
  
  cluster_name  = "${local.cluster_name}-gcp"
  gcp_project   = var.gcp_project_id
  gcp_region    = var.gcp_region
  node_count    = var.environment == "production" ? 3 : 1
  machine_type  = var.environment == "production" ? "n1-standard-4" : "n1-standard-2"
}

# Azure AKS
module "aks" {
  count  = contains(var.cloud_providers, "azure") ? 1 : 0
  source = "./modules/azure-aks"
  
  cluster_name    = "${local.cluster_name}-azure"
  location        = var.azure_location
  node_count      = var.environment == "production" ? 3 : 1
  vm_size         = var.environment == "production" ? "Standard_D4s_v3" : "Standard_D2s_v3"
}
```

---

## 3. Multi-Cloud Container Registry

### 3.1 Mirror Images across Clouds

```yaml
# .github/workflows/multi-cloud-publish.yml
name: Publish to Multiple Registries

on:
  release:
    types: [created]

jobs:
  build-and-publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Login to all registries
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ECRPushRole
          aws-region: ap-southeast-1

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
          service_account: ${{ secrets.GCP_SA_EMAIL }}

      - name: Configure Docker for GCR
        run: gcloud auth configure-docker asia.gcr.io

      # Build image เดียว แล้ว push ไปหลาย registries
      - name: Build and Push to All Registries
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.ref_name }}
            ghcr.io/${{ github.repository }}:latest
            ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.ref_name }}
            ${{ steps.login-ecr.outputs.registry }}/myapp:latest
            asia.gcr.io/${{ secrets.GCP_PROJECT_ID }}/myapp:${{ github.ref_name }}
            asia.gcr.io/${{ secrets.GCP_PROJECT_ID }}/myapp:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # Sign images ด้วย Cosign
          provenance: true
          sbom: true

      # Sign ทุก images
      - name: Sign Images
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          for image in \
            "ghcr.io/${{ github.repository }}:${{ github.ref_name }}" \
            "${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.ref_name }}" \
            "asia.gcr.io/${{ secrets.GCP_PROJECT_ID }}/myapp:${{ github.ref_name }}"; do
            echo "Signing ${image}..."
            cosign sign --yes "${image}"
          done
```

---

## 4. Multi-Cloud Deployment Pipeline

### 4.1 Deploy ไปหลาย Clouds

```yaml
# .github/workflows/multi-cloud-deploy.yml
name: Multi-Cloud Deployment

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      clouds:
        description: 'Clouds to deploy to (comma-separated)'
        required: true
        default: 'aws'
        type: string

jobs:
  # Build once
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - name: Build and Push
        id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  # Deploy to AWS EKS
  deploy-aws:
    runs-on: ubuntu-latest
    needs: build
    if: contains(github.event.inputs.clouds || 'aws', 'aws')
    environment: production-aws
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/EKSDeployRole
          aws-region: ap-southeast-1

      - name: Update EKS Kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name production-aws \
            --region ap-southeast-1

      - name: Deploy to EKS
        run: |
          helm upgrade --install myapp ./helm \
            --namespace production \
            --set image.repository=ghcr.io/${{ github.repository }} \
            --set image.tag=${{ github.sha }} \
            --set cloud=aws \
            --wait

  # Deploy to GCP GKE
  deploy-gcp:
    runs-on: ubuntu-latest
    needs: build
    if: contains(github.event.inputs.clouds || '', 'gcp')
    environment: production-gcp
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WIF_PROVIDER }}
          service_account: ${{ secrets.GCP_SA_EMAIL }}

      - name: Get GKE Credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: production-gcp
          location: asia-southeast1

      - name: Deploy to GKE
        run: |
          helm upgrade --install myapp ./helm \
            --namespace production \
            --set image.repository=ghcr.io/${{ github.repository }} \
            --set image.tag=${{ github.sha }} \
            --set cloud=gcp \
            --wait

  # Deploy to Azure AKS
  deploy-azure:
    runs-on: ubuntu-latest
    needs: build
    if: contains(github.event.inputs.clouds || '', 'azure')
    environment: production-azure
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Get AKS Credentials
        uses: azure/aks-set-context@v3
        with:
          cluster-name: production-azure
          resource-group: myapp-production

      - name: Deploy to AKS
        run: |
          helm upgrade --install myapp ./helm \
            --namespace production \
            --set image.repository=ghcr.io/${{ github.repository }} \
            --set image.tag=${{ github.sha }} \
            --set cloud=azure \
            --wait

  # Verify all deployments
  verify:
    runs-on: ubuntu-latest
    needs: [deploy-aws, deploy-gcp, deploy-azure]
    if: always()
    steps:
      - name: Check Deployment Results
        run: |
          AWS_STATUS="${{ needs.deploy-aws.result }}"
          GCP_STATUS="${{ needs.deploy-gcp.result }}"
          AZURE_STATUS="${{ needs.deploy-azure.result }}"
          
          echo "AWS: ${AWS_STATUS}"
          echo "GCP: ${GCP_STATUS}"
          echo "Azure: ${AZURE_STATUS}"
          
          # Fail ถ้า required deployments fail
          if [ "${AWS_STATUS}" = "failure" ]; then
            echo "❌ AWS deployment failed"
            exit 1
          fi
```

---

## 5. Terraform Multi-Cloud Modules

### 5.1 Abstract Cloud Resources

```hcl
# terraform/modules/cloud-storage/main.tf
# Module ที่ abstract cloud storage ข้ามหลาย providers

variable "cloud_provider" {
  description = "Cloud provider"
  type        = string
  validation {
    condition     = contains(["aws", "gcp", "azure"], var.cloud_provider)
    error_message = "Cloud provider ต้องเป็น aws, gcp, หรือ azure"
  }
}

variable "bucket_name" {
  description = "Storage bucket name"
  type        = string
}

variable "region" {
  description = "Cloud region"
  type        = string
}

variable "versioning_enabled" {
  description = "Enable versioning"
  type        = bool
  default     = true
}

# AWS S3
resource "aws_s3_bucket" "storage" {
  count  = var.cloud_provider == "aws" ? 1 : 0
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "storage" {
  count  = var.cloud_provider == "aws" && var.versioning_enabled ? 1 : 0
  bucket = aws_s3_bucket.storage[0].id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "storage" {
  count  = var.cloud_provider == "aws" ? 1 : 0
  bucket = aws_s3_bucket.storage[0].id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# GCP Cloud Storage
resource "google_storage_bucket" "storage" {
  count    = var.cloud_provider == "gcp" ? 1 : 0
  name     = var.bucket_name
  location = upper(var.region)
  
  versioning {
    enabled = var.versioning_enabled
  }
  
  uniform_bucket_level_access = true
  
  encryption {
    default_kms_key_name = null  # ใช้ Google-managed keys
  }
}

# Azure Blob Storage
resource "azurerm_storage_account" "storage" {
  count                    = var.cloud_provider == "azure" ? 1 : 0
  name                     = replace(var.bucket_name, "-", "")
  resource_group_name      = var.azure_resource_group
  location                 = var.region
  account_tier             = "Standard"
  account_replication_type = "LRS"
  
  blob_properties {
    versioning_enabled = var.versioning_enabled
  }
  
  min_tls_version = "TLS1_2"
}

# Outputs (uniform interface)
output "bucket_name" {
  value = var.cloud_provider == "aws" ? aws_s3_bucket.storage[0].id :
          var.cloud_provider == "gcp" ? google_storage_bucket.storage[0].name :
          azurerm_storage_account.storage[0].name
}

output "bucket_url" {
  value = var.cloud_provider == "aws" ? "s3://${aws_s3_bucket.storage[0].id}" :
          var.cloud_provider == "gcp" ? "gs://${google_storage_bucket.storage[0].name}" :
          "az://${azurerm_storage_account.storage[0].name}"
}
```

---

## 6. Multi-Cloud Secrets Management

### 6.1 HashiCorp Vault เป็น Universal Secret Store

```yaml
# k8s/vault/multi-cloud-vault.yaml
# Vault ทำหน้าที่เป็น central secrets store สำหรับทุก clouds

apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault
  namespace: vault-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vault
  template:
    metadata:
      labels:
        app: vault
    spec:
      containers:
        - name: vault
          image: hashicorp/vault:1.15
          ports:
            - containerPort: 8200
              name: http
            - containerPort: 8201
              name: cluster
          
          env:
            - name: VAULT_CLUSTER_ADDR
              value: "https://$(POD_IP):8201"
            - name: VAULT_API_ADDR
              value: "https://$(POD_IP):8200"
          
          readinessProbe:
            httpGet:
              path: /v1/sys/health
              port: 8200
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 5
          
          volumeMounts:
            - name: vault-config
              mountPath: /vault/config
            - name: vault-data
              mountPath: /vault/data
      
      volumes:
        - name: vault-config
          configMap:
            name: vault-config
        - name: vault-data
          persistentVolumeClaim:
            claimName: vault-data
```

### 6.2 Vault Configuration สำหรับ Multi-Cloud

```hcl
# vault/config/multi-cloud-auth.tf

# AWS Auth Method
resource "vault_auth_backend" "aws" {
  type = "aws"
  path = "aws"
}

resource "vault_aws_auth_backend_role" "eks_role" {
  backend                         = vault_auth_backend.aws.path
  role                            = "eks-production"
  auth_type                       = "iam"
  bound_iam_principal_arns        = ["arn:aws:iam::123456789012:role/EKSNodeRole"]
  resolve_aws_unique_ids          = false
  token_ttl                       = 3600
  token_policies                  = ["eks-production-policy"]
}

# GCP Auth Method
resource "vault_auth_backend" "gcp" {
  type = "gcp"
  path = "gcp"
}

resource "vault_gcp_auth_backend_role" "gke_role" {
  backend                = vault_auth_backend.gcp.path
  role                   = "gke-production"
  type                   = "iam"
  bound_service_accounts = ["gke-workload@my-project.iam.gserviceaccount.com"]
  token_ttl              = 3600
  token_policies         = ["gke-production-policy"]
}

# Azure Auth Method
resource "vault_auth_backend" "azure" {
  type = "azure"
  path = "azure"
}

# Common Policy สำหรับ production workloads
resource "vault_policy" "production" {
  name = "production-policy"
  
  policy = <<EOT
# Application secrets
path "secret/data/production/*" {
  capabilities = ["read"]
}

# Database credentials
path "database/creds/production-role" {
  capabilities = ["read"]
}

# PKI certificates
path "pki/issue/production" {
  capabilities = ["create", "update"]
}
EOT
}

# Secrets Engines
resource "vault_mount" "secret" {
  path = "secret"
  type = "kv-v2"
}

resource "vault_mount" "database" {
  path = "database"
  type = "database"
}

# Database secret engine configuration
resource "vault_database_secret_backend_connection" "postgres" {
  backend       = vault_mount.database.path
  name          = "myapp-postgres"
  allowed_roles = ["production-role"]
  
  postgresql {
    connection_url = "postgresql://vault:${var.postgres_vault_password}@postgres.production.svc.cluster.local:5432/myapp"
  }
}

resource "vault_database_secret_backend_role" "production" {
  backend             = vault_mount.database.path
  name                = "production-role"
  db_name             = vault_database_secret_backend_connection.postgres.name
  creation_statements = ["CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";"]
  default_ttl         = "1h"
  max_ttl             = "24h"
}
```

---

## 7. Multi-Cloud Cost Optimization

### 7.1 Cost Monitoring

```python
# scripts/cloud_cost_monitor.py
"""
Monitor และ compare costs ระหว่าง clouds
"""
import json
import os
from datetime import datetime, timedelta
from typing import Dict, List
import urllib.request
import urllib.parse


class MultiCloudCostMonitor:
    """Monitor cloud costs across multiple providers"""
    
    def __init__(self):
        self.aws_region = os.getenv('AWS_DEFAULT_REGION', 'ap-southeast-1')
        self.gcp_project = os.getenv('GCP_PROJECT_ID', '')
        self.azure_subscription = os.getenv('AZURE_SUBSCRIPTION_ID', '')
    
    def get_aws_costs(self, days: int = 30) -> Dict:
        """ดึง AWS costs"""
        import subprocess
        
        end_date = datetime.now().strftime('%Y-%m-%d')
        start_date = (datetime.now() - timedelta(days=days)).strftime('%Y-%m-%d')
        
        result = subprocess.run(
            ['aws', 'ce', 'get-cost-and-usage',
             '--time-period', f'Start={start_date},End={end_date}',
             '--granularity', 'MONTHLY',
             '--metrics', 'UnblendedCost',
             '--group-by', 'Type=DIMENSION,Key=SERVICE'],
            capture_output=True, text=True
        )
        
        if result.returncode != 0:
            return {"error": result.stderr, "total": 0}
        
        data = json.loads(result.stdout)
        total = sum(
            float(group['Metrics']['UnblendedCost']['Amount'])
            for result_period in data.get('ResultsByTime', [])
            for group in result_period.get('Groups', [])
        )
        
        return {
            "provider": "aws",
            "total_usd": total,
            "period_days": days
        }
    
    def get_gcp_costs(self, days: int = 30) -> Dict:
        """ดึง GCP costs (placeholder)"""
        # ในชีวิตจริงจะใช้ BigQuery billing export หรือ Cloud Billing API
        return {
            "provider": "gcp",
            "total_usd": 0,  # placeholder
            "period_days": days,
            "note": "Implement with BigQuery billing export"
        }
    
    def compare_costs(self) -> None:
        """Compare costs ระหว่าง clouds"""
        print("Multi-Cloud Cost Report")
        print("=" * 50)
        
        aws_costs = self.get_aws_costs()
        gcp_costs = self.get_gcp_costs()
        
        clouds = [aws_costs, gcp_costs]
        
        for cloud in clouds:
            if "error" not in cloud:
                print(f"\n{cloud['provider'].upper()}:")
                print(f"  Total (30 days): ${cloud['total_usd']:.2f}")
        
        total = sum(c.get('total_usd', 0) for c in clouds if 'error' not in c)
        print(f"\nTotal Multi-Cloud: ${total:.2f}")


# Cloud cost optimization recommendations
def generate_cost_recommendations(costs: List[Dict]) -> List[str]:
    """สร้าง cost optimization recommendations"""
    recommendations = []
    
    for cloud_cost in costs:
        if cloud_cost.get('total_usd', 0) > 10000:
            recommendations.append(
                f"⚠️  {cloud_cost['provider'].upper()} costs (${cloud_cost['total_usd']:.0f}) สูงกว่า threshold $10,000 — ควรตรวจสอบ reserved instances"
            )
    
    recommendations.append("💡 ใช้ Spot/Preemptible instances สำหรับ non-critical workloads")
    recommendations.append("💡 Implement auto-scaling เพื่อลด over-provisioning")
    recommendations.append("💡 Review data transfer costs ระหว่าง clouds")
    
    return recommendations


if __name__ == '__main__':
    monitor = MultiCloudCostMonitor()
    monitor.compare_costs()
```

---

## 8. Multi-Cloud Networking

### 8.1 Cross-Cloud VPN

```hcl
# terraform/networking/cross-cloud-vpn.tf
# สร้าง VPN ระหว่าง AWS และ GCP

# AWS VPN Gateway
resource "aws_vpn_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "aws-to-gcp-vpn"
  }
}

resource "aws_customer_gateway" "gcp" {
  bgp_asn    = 65000
  ip_address = google_compute_address.vpn_ip.address
  type       = "ipsec.1"
  
  tags = {
    Name = "gcp-customer-gateway"
  }
}

resource "aws_vpn_connection" "aws_to_gcp" {
  vpn_gateway_id      = aws_vpn_gateway.main.id
  customer_gateway_id = aws_customer_gateway.gcp.id
  type                = "ipsec.1"
  static_routes_only  = false
  
  tags = {
    Name = "aws-gcp-vpn-connection"
  }
}

# GCP VPN
resource "google_compute_vpn_gateway" "main" {
  name    = "gcp-to-aws-vpn"
  network = google_compute_network.main.id
  region  = var.gcp_region
}

resource "google_compute_address" "vpn_ip" {
  name   = "gcp-vpn-ip"
  region = var.gcp_region
}

# GCP VPN Tunnels
resource "google_compute_vpn_tunnel" "tunnel1" {
  name          = "aws-vpn-tunnel-1"
  peer_ip       = aws_vpn_connection.aws_to_gcp.tunnel1_address
  shared_secret = aws_vpn_connection.aws_to_gcp.tunnel1_preshared_key
  
  target_vpn_gateway = google_compute_vpn_gateway.main.id
  
  local_traffic_selector  = ["10.0.0.0/16"]  # GCP CIDR
  remote_traffic_selector = ["172.16.0.0/16"] # AWS CIDR
}
```

---

## 9. Workshop: Multi-Cloud Deployment

### Lab 1: Deploy ไป AWS และ GCP

```bash
#!/bin/bash
# workshop/lab1-multi-cloud-deploy.sh

echo "=== Multi-Cloud Deployment Lab ==="

# ตรวจสอบ credentials
check_aws() {
    aws sts get-caller-identity >/dev/null 2>&1
    return $?
}

check_gcp() {
    gcloud auth list --filter="status:ACTIVE" --format="value(account)" 2>/dev/null | grep -q "."
    return $?
}

if check_aws; then
    echo "✅ AWS authenticated"
    AWS_AVAILABLE=true
else
    echo "⚠️  AWS not authenticated, skipping"
    AWS_AVAILABLE=false
fi

if check_gcp; then
    echo "✅ GCP authenticated"
    GCP_AVAILABLE=true
else
    echo "⚠️  GCP not authenticated, skipping"
    GCP_AVAILABLE=false
fi

if [ "${AWS_AVAILABLE}" = "false" ] && [ "${GCP_AVAILABLE}" = "false" ]; then
    echo "❌ No cloud provider authenticated"
    exit 1
fi

# Build image
echo "Building Docker image..."
docker build -t myapp:lab1 .

# Deploy to available clouds
if [ "${AWS_AVAILABLE}" = "true" ]; then
    echo "Deploying to AWS..."
    
    # Login to ECR
    AWS_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
    AWS_REGION=$(aws configure get region)
    ECR_URL="${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    
    aws ecr get-login-password | docker login --username AWS --password-stdin "${ECR_URL}"
    
    docker tag myapp:lab1 "${ECR_URL}/myapp:lab1"
    docker push "${ECR_URL}/myapp:lab1"
    
    echo "✅ Pushed to ECR: ${ECR_URL}/myapp:lab1"
fi

if [ "${GCP_AVAILABLE}" = "true" ]; then
    echo "Deploying to GCP..."
    
    GCP_PROJECT=$(gcloud config get-value project)
    GCR_URL="asia.gcr.io/${GCP_PROJECT}"
    
    gcloud auth configure-docker asia.gcr.io --quiet
    
    docker tag myapp:lab1 "${GCR_URL}/myapp:lab1"
    docker push "${GCR_URL}/myapp:lab1"
    
    echo "✅ Pushed to GCR: ${GCR_URL}/myapp:lab1"
fi

echo ""
echo "=== Deployment Summary ==="
echo "AWS: ${AWS_AVAILABLE}"
echo "GCP: ${GCP_AVAILABLE}"
```

---

## 10. สรุปและ Best Practices

### Multi-Cloud Strategy Checklist

```markdown
## Multi-Cloud CI/CD Checklist

### Strategy
- [ ] กำหนด primary cloud provider
- [ ] Document reasons สำหรับ multi-cloud
- [ ] Define cloud-specific vs shared services
- [ ] Plan for vendor lock-in mitigation

### Infrastructure
- [ ] ใช้ Terraform สำหรับ infrastructure-as-code
- [ ] Abstract cloud-specific APIs
- [ ] Implement unified networking
- [ ] Centralized secrets management (Vault)

### CI/CD Pipeline
- [ ] Build once, deploy everywhere
- [ ] Cloud-specific deployment stages
- [ ] Unified monitoring
- [ ] Cross-cloud testing

### Cost Management
- [ ] Implement cost monitoring
- [ ] Set budgets และ alerts
- [ ] Optimize data transfer
- [ ] Use reserved instances

### Security
- [ ] Unified IAM strategy
- [ ] Consistent network policies
- [ ] Cross-cloud audit logging
- [ ] Regular security reviews
```

---

## อ้างอิง

- [Terraform Multi-Cloud](https://developer.hashicorp.com/terraform/docs)
- [AWS Multi-Cloud Strategy](https://aws.amazon.com/solutions/implementations/multi-cloud-networking/)
- [GCP Multi-Cloud](https://cloud.google.com/architecture/multicloud)
- [Azure Arc](https://azure.microsoft.com/en-us/products/azure-arc/)
- [HashiCorp Vault](https://developer.hashicorp.com/vault/docs)
- [Cloud Native Computing Foundation](https://www.cncf.io/)

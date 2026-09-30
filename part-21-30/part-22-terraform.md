# Part 22: Infrastructure as Code ด้วย Terraform

## บทนำ

Infrastructure as Code (IaC) คือการจัดการ infrastructure โดยใช้ code แทนการ configure ด้วยมือ Terraform เป็นเครื่องมือ IaC ที่ได้รับความนิยมมากที่สุด สร้างโดย HashiCorp โดยรองรับ cloud providers หลายเจ้าเช่น AWS, GCP, Azure, และอีกกว่า 1000+ providers

**ทำไมต้องใช้ IaC?**
- **Reproducibility**: สร้าง environment เดิมซ้ำได้เสมอ
- **Version Control**: ติดตามการเปลี่ยนแปลง infrastructure ด้วย Git
- **Automation**: Deploy infrastructure เป็นส่วนหนึ่งของ CI/CD pipeline
- **Documentation**: Code คือ documentation ของ infrastructure
- **Collaboration**: Team ทำงานร่วมกันได้บน infrastructure

---

## สารบัญ

1. Terraform คืออะไร
2. ติดตั้ง Terraform
3. HCL Syntax
4. Providers (AWS/GCP/Azure)
5. Resources, Variables, Outputs
6. State Management
7. Terraform Commands
8. Modules
9. Workspaces
10. Remote State
11. Terraform Cloud
12. CI/CD Integration
13. Security Best Practices
14. Terragrunt Introduction
15. แบบฝึกหัด

---

## 1. Terraform คืออะไร?

Terraform ใช้ declarative approach - เราบอกว่าอยากได้ infrastructure แบบไหน ไม่ใช่วิธีสร้าง

```
Declarative:  "ฉันต้องการ EC2 instance ขนาด t3.medium จำนวน 3 ตัว"
Imperative:   "สร้าง VM 1 -> configure network -> attach storage -> ..."
```

### Terraform Workflow

```
Write (HCL) → Plan (preview) → Apply (create) → Destroy (cleanup)
```

### Terraform vs Alternative Tools

| Tool | Approach | Language | Cloud |
|------|----------|----------|-------|
| Terraform | Declarative | HCL | Multi-cloud |
| CloudFormation | Declarative | JSON/YAML | AWS only |
| Pulumi | Imperative | Python/JS/Go | Multi-cloud |
| Ansible | Imperative | YAML | Multi-cloud + OS |
| CDK | Imperative | Python/JS/TS | AWS/Azure/GCP |

---

## 2. ติดตั้ง Terraform

### ติดตั้งบน Linux/Ubuntu

```bash
# วิธีที่ 1: ผ่าน HashiCorp apt repository
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform

# ตรวจสอบ
terraform version

# วิธีที่ 2: Download binary โดยตรง
curl -fsSL https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip \
  -o terraform.zip
unzip terraform.zip
sudo mv terraform /usr/local/bin/
terraform version
```

### ติดตั้งบน macOS

```bash
# Homebrew
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# หรือผ่าน tfenv (Terraform version manager)
brew install tfenv
tfenv install 1.6.0
tfenv use 1.6.0
```

### ติดตั้งบน Windows

```powershell
# Chocolatey
choco install terraform

# หรือ Scoop
scoop install terraform

# ตรวจสอบ
terraform version
```

### Setup Auto-completion

```bash
# Bash
terraform -install-autocomplete
source ~/.bashrc

# Zsh
terraform -install-autocomplete
source ~/.zshrc
```

---

## 3. HCL Syntax

HCL (HashiCorp Configuration Language) เป็น configuration language ที่ออกแบบให้อ่านง่ายและเขียนง่าย

### Blocks

```hcl
# รูปแบบ block
<BLOCK_TYPE> "<BLOCK_LABEL_1>" "<BLOCK_LABEL_2>" {
  # arguments
  argument_name = argument_value
}

# ตัวอย่าง
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  
  tags = {
    Name        = "WebServer"
    Environment = "Production"
  }
}
```

### Types

```hcl
# String
name = "my-application"
description = "A ${var.environment} deployment"  # String interpolation

# Number
port        = 8080
max_size    = 10
min_size    = 2

# Boolean
enable_dns  = true
multi_az    = false

# List
availability_zones = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]

# Map
tags = {
  Project     = "my-project"
  Environment = "production"
  ManagedBy   = "terraform"
}

# Object
server_config = {
  type     = "t3.medium"
  count    = 3
  enabled  = true
}

# Set
security_groups = toset(["sg-001", "sg-002", "sg-003"])

# Tuple
mixed_list = ["string", 42, true]
```

### Expressions

```hcl
# Conditional expression
instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

# For expression (list)
public_ips = [for instance in aws_instance.web : instance.public_ip]

# For expression (map)
instance_ids = {for instance in aws_instance.web : instance.tags.Name => instance.id}

# Splat expression
all_private_ips = aws_instance.web[*].private_ip

# Dynamic block
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port   = ingress.value.from_port
    to_port     = ingress.value.to_port
    protocol    = ingress.value.protocol
    cidr_blocks = ingress.value.cidr_blocks
  }
}
```

### Functions

```hcl
# String functions
upper_name   = upper(var.name)           # MY-APP
lower_name   = lower(var.name)           # my-app
trimmed      = trimspace("  hello  ")    # "hello"
formatted    = format("-%03d", 42)       # "-042"
joined       = join(",", ["a","b","c"])  # "a,b,c"
split_list   = split(",", "a,b,c")       # ["a","b","c"]

# Collection functions
length_val   = length(var.list)          # จำนวน elements
flatten_list = flatten([[1,2],[3,4]])    # [1,2,3,4]
merge_map    = merge(map1, map2)         # รวม maps
keys_list    = keys(var.map)             # list ของ keys
values_list  = values(var.map)           # list ของ values
contains_val = contains(["a","b"], "a") # true

# Numeric functions
min_val = min(1, 2, 3)                  # 1
max_val = max(1, 2, 3)                  # 3
abs_val = abs(-5)                       # 5
ceil_val = ceil(1.2)                    # 2
floor_val = floor(1.9)                  # 1

# Encoding functions
base64_encoded = base64encode("hello")
json_encoded   = jsonencode({key = "value"})
json_decoded   = jsondecode("{\"key\":\"value\"}")

# File functions
file_content = file("${path.module}/scripts/init.sh")
template     = templatefile("${path.module}/templates/config.tpl", {
  server_name = var.server_name
  port        = var.port
})

# Type functions
to_string = tostring(42)              # "42"
to_number = tonumber("42")           # 42
to_list   = tolist(toset(["a","b"])) # ["a","b"]
```

---

## 4. Providers

Providers คือ plugin ที่ให้ Terraform ทำงานกับ API ต่างๆ

### AWS Provider

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

# provider.tf
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

# Multi-region provider
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "ap_southeast"
  region = "ap-southeast-1"
}

# ใช้ provider ที่ระบุ
resource "aws_s3_bucket" "us_bucket" {
  provider = aws.us_east
  bucket   = "my-us-bucket"
}
```

### GCP Provider

```hcl
# versions.tf
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
    google-beta = {
      source  = "hashicorp/google-beta"
      version = "~> 5.0"
    }
  }
}

# provider.tf
provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

# Authentication
# 1. Service Account Key (ไม่แนะนำสำหรับ production)
provider "google" {
  credentials = file("service-account.json")
  project     = var.project_id
  region      = var.region
}

# 2. Application Default Credentials (แนะนำ)
# gcloud auth application-default login
provider "google" {
  project = var.project_id
  region  = var.region
}

# 3. Workload Identity Federation (best practice)
provider "google" {
  project = var.project_id
  region  = var.region
  # ใช้ WIF credentials อัตโนมัติจาก environment
}
```

### Azure Provider

```hcl
# versions.tf
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.0"
    }
  }
}

# provider.tf
provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    virtual_machine {
      delete_os_disk_on_deletion     = true
      graceful_shutdown              = false
    }
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
  }
  
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
}
```

---

## 5. Resources, Variables, Outputs

### Variables

```hcl
# variables.tf

# String variable
variable "environment" {
  type        = string
  description = "Deployment environment (dev/staging/production)"
  
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

# Number variable with default
variable "instance_count" {
  type        = number
  description = "Number of EC2 instances"
  default     = 2
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}

# Map variable
variable "instance_types" {
  type = map(string)
  description = "EC2 instance type per environment"
  default = {
    dev        = "t3.micro"
    staging    = "t3.small"
    production = "t3.medium"
  }
}

# List variable
variable "allowed_cidr_blocks" {
  type        = list(string)
  description = "CIDR blocks allowed to access the application"
  default     = []
}

# Object variable
variable "database_config" {
  type = object({
    engine         = string
    version        = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  description = "RDS database configuration"
  default = {
    engine         = "postgres"
    version        = "15.3"
    instance_class = "db.t3.medium"
    storage_gb     = 100
    multi_az       = false
  }
}

# Sensitive variable
variable "db_password" {
  type        = string
  description = "Database password"
  sensitive   = true
}

# terraform.tfvars (ไม่ commit ไปใน git!)
environment         = "production"
instance_count      = 3
db_password         = "super-secret-password"
allowed_cidr_blocks = ["10.0.0.0/8", "172.16.0.0/12"]
database_config = {
  engine         = "postgres"
  version        = "15.3"
  instance_class = "db.t3.large"
  storage_gb     = 500
  multi_az       = true
}
```

### Locals

```hcl
# locals.tf
locals {
  # Computed values
  name_prefix = "${var.project_name}-${var.environment}"
  
  # Common tags
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    CreatedAt   = timestamp()
  }
  
  # Conditional logic
  is_production = var.environment == "production"
  
  # Computed instance type
  instance_type = local.is_production ? "t3.large" : "t3.micro"
  
  # AZ list
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
  
  # Subnet CIDR blocks
  public_subnet_cidrs  = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i + 10)]
}
```

### Complete AWS Infrastructure Example

```hcl
# main.tf - Complete AWS VPC + ECS setup

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

data "aws_region" "current" {}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Type = "Public"
    "kubernetes.io/role/elb" = "1"
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-private-${count.index + 1}"
    Type = "Private"
    "kubernetes.io/role/internal-elb" = "1"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-igw"
  })
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = local.is_production ? length(local.azs) : 1
  domain = "vpc"
  
  depends_on = [aws_internet_gateway.main]
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-eip-nat-${count.index + 1}"
  })
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count         = local.is_production ? length(local.azs) : 1
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  depends_on = [aws_internet_gateway.main]
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-nat-${count.index + 1}"
  })
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-rt-public"
  })
}

resource "aws_route_table" "private" {
  count  = local.is_production ? length(local.azs) : 1
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[local.is_production ? count.index : 0].id
  }
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-rt-private-${count.index + 1}"
  })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[local.is_production ? count.index : 0].id
}

# Security Groups
resource "aws_security_group" "alb" {
  name_prefix = "${local.name_prefix}-alb-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for Application Load Balancer"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from internet"
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from internet"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound"
  }
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-sg-alb"
  })
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "app" {
  name_prefix = "${local.name_prefix}-app-"
  vpc_id      = aws_vpc.main.id
  description = "Security group for application servers"
  
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "App port from ALB"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound"
  }
  
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-sg-app"
  })
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### Outputs

```hcl
# outputs.tf

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "alb_dns_name" {
  description = "DNS name of the Application Load Balancer"
  value       = aws_lb.main.dns_name
}

output "rds_endpoint" {
  description = "RDS instance endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

output "ecr_repository_urls" {
  description = "Map of ECR repository URLs"
  value = {
    for name, repo in aws_ecr_repository.repos : 
    name => repo.repository_url
  }
}

output "infrastructure_summary" {
  description = "Summary of created infrastructure"
  value = {
    environment    = var.environment
    region         = data.aws_region.current.name
    vpc_id         = aws_vpc.main.id
    public_subnets = length(aws_subnet.public)
    private_subnets = length(aws_subnet.private)
    nat_gateways   = length(aws_nat_gateway.main)
  }
}
```

---

## 6. State Management

### Terraform State คืออะไร?

Terraform State เก็บข้อมูลเกี่ยวกับ infrastructure ที่ Terraform จัดการ ทำให้ Terraform รู้ว่า resource ไหนถูกสร้างไปแล้ว

```bash
# ดู state
terraform state list

# ดู state ของ resource เฉพาะ
terraform state show aws_instance.web_server

# Import existing resource เข้า state
terraform import aws_instance.existing_server i-1234567890abcdef0

# Remove resource จาก state (ไม่ destroy!)
terraform state rm aws_instance.old_server

# Move resource ใน state
terraform state mv aws_instance.server aws_instance.app_server

# ดู state file โดยตรง
cat terraform.tfstate | python3 -m json.tool | head -50
```

### State Locking

State locking ป้องกันการ modify state พร้อมกัน

```hcl
# ใช้ DynamoDB สำหรับ state locking บน AWS
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name    = "Terraform State Lock Table"
    Purpose = "terraform-state-locking"
  }
}
```

---

## 7. Terraform Commands

### Core Commands

```bash
# Initialize project (download providers and modules)
terraform init

# Format code
terraform fmt

# Validate configuration
terraform validate

# Plan changes
terraform plan

# Plan with specific variables
terraform plan \
  -var="environment=production" \
  -var-file="production.tfvars"

# Plan and save to file
terraform plan -out=tfplan

# Apply changes
terraform apply

# Apply from saved plan
terraform apply tfplan

# Apply without confirmation
terraform apply -auto-approve

# Destroy all resources
terraform destroy

# Destroy specific resource
terraform destroy -target=aws_instance.web_server

# Show outputs
terraform output
terraform output vpc_id
terraform output -json

# Refresh state (sync with actual infrastructure)
terraform refresh

# Graph dependencies
terraform graph | dot -Tsvg > graph.svg
```

### Advanced Commands

```bash
# Force unlock state
terraform force-unlock <LOCK_ID>

# Taint resource (mark for replacement)
terraform taint aws_instance.web_server

# Untaint resource
terraform untaint aws_instance.web_server

# Console (interactive expression evaluator)
terraform console
> var.environment
"production"
> length(var.availability_zones)
3
> aws_vpc.main.id
"vpc-12345678"

# Provider schemas
terraform providers schema -json

# Debug logging
TF_LOG=DEBUG terraform apply
TF_LOG_PATH=./debug.log terraform apply
```

---

## 8. Modules

Modules คือ reusable Terraform code packages

### สร้าง Module

```
modules/
├── vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
├── ecs-cluster/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── rds/
    ├── main.tf
    ├── variables.tf
    └── outputs.tf
```

```hcl
# modules/vpc/variables.tf
variable "name" {
  type        = string
  description = "Name prefix for all resources"
}

variable "vpc_cidr" {
  type        = string
  description = "CIDR block for VPC"
  default     = "10.0.0.0/16"
}

variable "az_count" {
  type        = number
  description = "Number of availability zones"
  default     = 3
}

variable "enable_nat_gateway" {
  type        = bool
  description = "Enable NAT Gateway for private subnets"
  default     = true
}

variable "single_nat_gateway" {
  type        = bool
  description = "Use single NAT Gateway (cost optimization for non-prod)"
  default     = false
}

variable "tags" {
  type        = map(string)
  description = "Tags to apply to all resources"
  default     = {}
}
```

```hcl
# modules/vpc/main.tf
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, var.az_count)
  
  public_subnet_cidrs  = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i, az in local.azs : cidrsubnet(var.vpc_cidr, 8, i + 10)]
  
  nat_gateway_count = var.single_nat_gateway ? 1 : (var.enable_nat_gateway ? length(local.azs) : 0)
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Type = "Public"
  })
}

resource "aws_subnet" "private" {
  count             = length(local.azs)
  vpc_id            = aws_vpc.this.id
  cidr_block        = local.private_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
  
  tags = merge(var.tags, {
    Name = "${var.name}-private-${count.index + 1}"
    Type = "Private"
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id
  
  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}

resource "aws_eip" "nat" {
  count  = local.nat_gateway_count
  domain = "vpc"
  
  depends_on = [aws_internet_gateway.this]
}

resource "aws_nat_gateway" "this" {
  count         = local.nat_gateway_count
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(var.tags, {
    Name = "${var.name}-nat-${count.index + 1}"
  })
  
  depends_on = [aws_internet_gateway.this]
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }
  
  tags = merge(var.tags, {
    Name = "${var.name}-rt-public"
  })
}

resource "aws_route_table" "private" {
  count  = length(local.azs)
  vpc_id = aws_vpc.this.id
  
  dynamic "route" {
    for_each = local.nat_gateway_count > 0 ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.this[var.single_nat_gateway ? 0 : count.index].id
    }
  }
  
  tags = merge(var.tags, {
    Name = "${var.name}-rt-private-${count.index + 1}"
  })
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  value = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ids" {
  description = "List of NAT Gateway IDs"
  value       = aws_nat_gateway.this[*].id
}
```

### ใช้ Module

```hcl
# environments/production/main.tf

module "vpc" {
  source = "../../modules/vpc"  # local module
  # หรือ
  # source = "terraform-aws-modules/vpc/aws"  # Terraform Registry
  # version = "5.0.0"
  
  name       = "myapp-prod"
  vpc_cidr   = "10.0.0.0/16"
  az_count   = 3
  
  enable_nat_gateway = true
  single_nat_gateway = false  # HA NAT for production
  
  tags = {
    Environment = "production"
    Project     = "myapp"
  }
}

module "ecs_cluster" {
  source = "../../modules/ecs-cluster"
  
  name               = "myapp-prod"
  vpc_id             = module.vpc.vpc_id
  private_subnet_ids = module.vpc.private_subnet_ids
  
  tags = {
    Environment = "production"
  }
}

module "rds" {
  source = "../../modules/rds"
  
  identifier         = "myapp-prod"
  engine             = "postgres"
  engine_version     = "15.3"
  instance_class     = "db.t3.large"
  
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  allowed_sg_ids     = [module.ecs_cluster.security_group_id]
  
  multi_az           = true
  backup_retention   = 7
  
  tags = {
    Environment = "production"
  }
}
```

### Terraform Registry Modules

```hcl
# ใช้ community modules จาก Terraform Registry
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  
  cluster_name    = "myapp-cluster"
  cluster_version = "1.28"
  
  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnet_ids
  cluster_endpoint_public_access = true
  
  eks_managed_node_group_defaults = {
    ami_type       = "AL2_x86_64"
    instance_types = ["m5.large"]
  }
  
  eks_managed_node_groups = {
    green = {
      min_size     = 2
      max_size     = 10
      desired_size = 3
      
      instance_types = ["t3.large"]
      capacity_type  = "ON_DEMAND"
    }
    
    spot = {
      min_size       = 0
      max_size       = 5
      desired_size   = 0
      
      instance_types = ["t3.large", "t3a.large"]
      capacity_type  = "SPOT"
    }
  }
  
  tags = local.common_tags
}
```

---

## 9. Workspaces

Workspaces ช่วยให้เราจัดการ multiple environments ด้วย codebase เดียว

```bash
# List workspaces
terraform workspace list

# Create workspace
terraform workspace new production
terraform workspace new staging
terraform workspace new development

# Switch workspace
terraform workspace select production

# Show current workspace
terraform workspace show

# Delete workspace
terraform workspace delete development
```

```hcl
# ใช้ workspace ใน configuration
locals {
  workspace_config = {
    development = {
      instance_type   = "t3.micro"
      instance_count  = 1
      db_instance_class = "db.t3.micro"
      multi_az        = false
    }
    staging = {
      instance_type   = "t3.small"
      instance_count  = 2
      db_instance_class = "db.t3.small"
      multi_az        = false
    }
    production = {
      instance_type   = "t3.large"
      instance_count  = 5
      db_instance_class = "db.t3.large"
      multi_az        = true
    }
  }
  
  config = local.workspace_config[terraform.workspace]
}

resource "aws_instance" "app" {
  count         = local.config.instance_count
  ami           = data.aws_ami.app.id
  instance_type = local.config.instance_type
  
  tags = {
    Name        = "${terraform.workspace}-app-${count.index + 1}"
    Environment = terraform.workspace
  }
}
```

---

## 10. Remote State

### AWS S3 Backend

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "production/terraform.tfstate"
    region         = "ap-southeast-1"
    encrypt        = true
    
    # State locking with DynamoDB
    dynamodb_table = "terraform-state-locks"
    
    # KMS encryption
    kms_key_id     = "alias/terraform-state"
  }
}
```

```bash
# สร้าง S3 bucket และ DynamoDB table ก่อน
aws s3api create-bucket \
  --bucket my-terraform-state-bucket \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket my-terraform-state-bucket \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/terraform-state"
      }
    }]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket my-terraform-state-bucket \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# สร้าง DynamoDB table
aws dynamodb create-table \
  --table-name terraform-state-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-southeast-1
```

### GCS Backend (GCP)

```hcl
terraform {
  backend "gcs" {
    bucket  = "my-terraform-state-bucket"
    prefix  = "production/terraform/state"
  }
}
```

### Azure Blob Storage Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
  }
}
```

### Remote State Data Source

```hcl
# อ่าน state จาก remote location อื่น
data "terraform_remote_state" "vpc" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state-bucket"
    key    = "production/vpc/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

# ใช้ output จาก remote state
resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.vpc.outputs.private_subnet_ids[0]
  vpc_security_group_ids = [
    data.terraform_remote_state.vpc.outputs.app_security_group_id
  ]
}
```

---

## 11. Terraform Cloud

```hcl
# backend.tf - ใช้ Terraform Cloud
terraform {
  cloud {
    organization = "my-organization"
    
    workspaces {
      name = "production"
      # หรือใช้ tags
      # tags = ["production", "aws"]
    }
  }
}
```

```bash
# Login to Terraform Cloud
terraform login

# Initialize
terraform init
```

---

## 12. CI/CD Integration กับ GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'
      - '.github/workflows/terraform.yml'
  pull_request:
    branches: [main]
    paths:
      - 'infrastructure/**'

permissions:
  id-token: write   # สำหรับ OIDC authentication
  contents: read
  pull-requests: write

env:
  TF_VERSION: "1.6.0"
  AWS_REGION: "ap-southeast-1"
  TF_DIR: "infrastructure/environments/production"

jobs:
  terraform-check:
    name: Terraform Format & Validate
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}
    
    - name: Terraform Format Check
      run: terraform fmt -check -recursive
      working-directory: infrastructure
    
    - name: Terraform Validate
      run: |
        terraform init -backend=false
        terraform validate
      working-directory: ${{ env.TF_DIR }}

  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: terraform-check
    outputs:
      plan-exit-code: ${{ steps.plan.outputs.exitcode }}
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}
    
    - name: Terraform Init
      run: terraform init
      working-directory: ${{ env.TF_DIR }}
    
    - name: Terraform Plan
      id: plan
      run: |
        terraform plan \
          -var-file="terraform.tfvars" \
          -out=tfplan \
          -detailed-exitcode \
          -no-color 2>&1 | tee plan_output.txt
        echo "exitcode=$?" >> $GITHUB_OUTPUT
      working-directory: ${{ env.TF_DIR }}
      continue-on-error: true
    
    - name: Upload Plan Artifact
      uses: actions/upload-artifact@v3
      with:
        name: terraform-plan
        path: |
          ${{ env.TF_DIR }}/tfplan
          ${{ env.TF_DIR }}/plan_output.txt
        retention-days: 7
    
    - name: Comment Plan on PR
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v7
      with:
        script: |
          const fs = require('fs');
          const planOutput = fs.readFileSync('${{ env.TF_DIR }}/plan_output.txt', 'utf8');
          
          const truncated = planOutput.length > 65000 
            ? planOutput.substring(0, 65000) + '\n... (truncated)' 
            : planOutput;
          
          const output = `## Terraform Plan Output
          
          <details>
          <summary>Show Plan</summary>
          
          \`\`\`hcl
          ${truncated}
          \`\`\`
          </details>
          
          *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
          
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: output
          });
    
    - name: Check Plan Exit Code
      if: steps.plan.outputs.exitcode == '1'
      run: exit 1

  terraform-apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: |
      github.ref == 'refs/heads/main' && 
      github.event_name == 'push' &&
      needs.terraform-plan.outputs.plan-exit-code == '2'
    environment: production
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}
    
    - name: Terraform Init
      run: terraform init
      working-directory: ${{ env.TF_DIR }}
    
    - name: Download Plan
      uses: actions/download-artifact@v3
      with:
        name: terraform-plan
        path: ${{ env.TF_DIR }}
    
    - name: Terraform Apply
      run: terraform apply -auto-approve tfplan
      working-directory: ${{ env.TF_DIR }}
    
    - name: Get Outputs
      id: outputs
      run: terraform output -json > outputs.json
      working-directory: ${{ env.TF_DIR }}
    
    - name: Notify Success
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "✅ Terraform Apply completed successfully!\nCommit: ${{ github.sha }}\nEnvironment: production"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### AWS IAM Role สำหรับ GitHub Actions OIDC

```hcl
# github-actions-iam.tf

# OIDC Provider
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = ["sts.amazonaws.com"]
  
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1"
  ]
}

# IAM Role
resource "aws_iam_role" "github_actions_terraform" {
  name = "github-actions-terraform"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRoleWithWebIdentity"
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Condition = {
        StringLike = {
          "token.actions.githubusercontent.com:sub" = 
            "repo:my-org/my-repo:*"
        }
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
      }
    }]
  })
}

# IAM Policy
resource "aws_iam_role_policy" "github_actions_terraform" {
  role = aws_iam_role.github_actions_terraform.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::my-terraform-state-bucket",
          "arn:aws:s3:::my-terraform-state-bucket/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = "arn:aws:dynamodb:*:*:table/terraform-state-locks"
      },
      # เพิ่ม permissions สำหรับ resources ที่ Terraform จัดการ
      {
        Effect = "Allow"
        Action = [
          "ec2:*",
          "ecs:*",
          "rds:*",
          "elasticloadbalancing:*",
          "autoscaling:*",
          "iam:*",
          "logs:*",
          "ecr:*"
        ]
        Resource = "*"
      }
    ]
  })
}
```

---

## 13. Security Best Practices

### ไม่เก็บ Secrets ใน Code

```hcl
# ❌ ห้ามทำแบบนี้
resource "aws_db_instance" "main" {
  password = "my-plaintext-password"  # อันตราย!
}

# ✅ ใช้ AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/myapp/db-password"
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}

# ✅ หรือใช้ SSM Parameter Store
data "aws_ssm_parameter" "db_password" {
  name            = "/production/myapp/db-password"
  with_decryption = true
}

resource "aws_db_instance" "main" {
  password = data.aws_ssm_parameter.db_password.value
}
```

### .gitignore สำหรับ Terraform

```gitignore
# .gitignore

# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log
crash.*.log

# Exclude all .tfvars files (may contain secrets)
*.tfvars
*.tfvars.json

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI configuration files
.terraformrc
terraform.rc

# Plan output
tfplan
*.tfplan

# Lock file (keep this in git for reproducible builds)
# .terraform.lock.hcl  # <-- DO commit this file
```

### Sentinel Policies (Terraform Enterprise/Cloud)

```python
# sentinel/restrict-instance-types.sentinel
import "tfplan/v2" as tfplan

# อนุญาต instance types ที่กำหนด
allowed_instance_types = [
  "t3.micro",
  "t3.small",
  "t3.medium",
  "t3.large",
  "m5.large",
  "m5.xlarge"
]

# ตรวจสอบ EC2 instances ทั้งหมด
ec2_instances = filter tfplan.resource_changes as _, resource_change {
  resource_change.type is "aws_instance" and
  resource_change.change.actions contains "create"
}

# Main rule
main = rule {
  all ec2_instances as _, instance {
    instance.change.after.instance_type in allowed_instance_types
  }
}
```

---

## 14. Terragrunt Introduction

Terragrunt เป็น thin wrapper สำหรับ Terraform ที่ช่วยจัดการ:
- DRY (Don't Repeat Yourself) configuration
- Remote state management
- Multiple accounts/environments

```
infrastructure/
├── terragrunt.hcl          # root config
├── modules/
│   ├── vpc/
│   ├── ecs/
│   └── rds/
└── live/
    ├── production/
    │   ├── account.hcl
    │   ├── vpc/
    │   │   └── terragrunt.hcl
    │   ├── ecs/
    │   │   └── terragrunt.hcl
    │   └── rds/
    │       └── terragrunt.hcl
    └── staging/
        ├── account.hcl
        ├── vpc/
        │   └── terragrunt.hcl
        └── rds/
            └── terragrunt.hcl
```

```hcl
# terragrunt.hcl (root)
locals {
  account_vars     = read_terragrunt_config(find_in_parent_folders("account.hcl"))
  environment_vars = read_terragrunt_config(find_in_parent_folders("env.hcl"))
  
  account_id  = local.account_vars.locals.account_id
  environment = local.environment_vars.locals.environment
  aws_region  = "ap-southeast-1"
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
provider "aws" {
  region = "${local.aws_region}"
  
  assume_role {
    role_arn = "arn:aws:iam::${local.account_id}:role/terraform-executor"
  }
}
EOF
}

remote_state {
  backend = "s3"
  config = {
    encrypt        = true
    bucket         = "my-terraform-state-${local.account_id}"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = local.aws_region
    dynamodb_table = "terraform-state-locks"
  }
  
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
}

inputs = merge(
  local.account_vars.locals,
  local.environment_vars.locals,
)
```

```hcl
# live/production/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/vpc"
}

inputs = {
  name               = "myapp-prod"
  vpc_cidr           = "10.0.0.0/16"
  az_count           = 3
  enable_nat_gateway = true
  single_nat_gateway = false
  
  tags = {
    Environment = "production"
    Project     = "myapp"
  }
}
```

```bash
# Terragrunt commands
# Run all modules
terragrunt run-all apply

# Run specific module
cd live/production/vpc
terragrunt apply

# Plan all
terragrunt run-all plan

# Destroy all
terragrunt run-all destroy
```

---

## 15. แบบฝึกหัด

### แบบฝึกหัดที่ 1: First Terraform Project

```hcl
# exercise-1/main.tf
# สร้าง web server บน AWS ที่มี:
# - EC2 instance
# - Security group ที่เปิด port 80
# - Elastic IP

# TODO: กรอก code ให้สมบูรณ์

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-southeast-1"
}

# TODO: สร้าง Security Group
resource "aws_security_group" "web" {
  # ...
}

# TODO: ดึง Latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  # ...
}

# TODO: สร้าง EC2 Instance
resource "aws_instance" "web" {
  # ...
  
  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
  EOF
}

# TODO: สร้าง Elastic IP และ Associate กับ EC2

# TODO: Output public IP
```

### แบบฝึกหัดที่ 2: VPC Module

```bash
# สร้าง VPC module structure
mkdir -p exercise-2/{modules/vpc,environments/{dev,production}}

# TODO: สร้างไฟล์ต่อไปนี้:
# exercise-2/modules/vpc/main.tf
# exercise-2/modules/vpc/variables.tf
# exercise-2/modules/vpc/outputs.tf
# exercise-2/environments/dev/main.tf
# exercise-2/environments/production/main.tf

# Dev environment: single NAT gateway, 2 AZs
# Production: multiple NAT gateways, 3 AZs, multi-az RDS
```

### แบบฝึกหัดที่ 3: Remote State

```hcl
# exercise-3/bootstrap/main.tf
# สร้าง infrastructure สำหรับเก็บ Terraform state:
# 1. S3 bucket with versioning และ encryption
# 2. DynamoDB table สำหรับ state locking
# 3. IAM role สำหรับ GitHub Actions

# Hint: ต้อง apply bootstrap ก่อน แล้วค่อย configure backend

# TODO: สร้าง S3 bucket
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-${data.aws_caller_identity.current.account_id}"
  
  lifecycle {
    prevent_destroy = true  # ป้องกันการลบโดยบังเอิญ
  }
}

# TODO: Enable versioning, encryption, block public access

# TODO: สร้าง DynamoDB table

# TODO: สร้าง IAM role สำหรับ GitHub Actions
```

### แบบฝึกหัดที่ 4: CI/CD Pipeline

```yaml
# exercise-4/.github/workflows/terraform.yml
# สร้าง complete GitHub Actions workflow สำหรับ Terraform ที่:
# 1. Format check และ validate บน PR
# 2. Plan บน PR และ comment ผลลัพธ์
# 3. Apply อัตโนมัติเมื่อ merge ไป main
# 4. Notify ผ่าน Slack

# TODO: สร้าง workflow ที่สมบูรณ์
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# TODO: เพิ่ม jobs ต่อไปนี้:
# - terraform-fmt: ตรวจสอบ format
# - terraform-plan: สร้าง plan บน PR
# - terraform-apply: apply บน main (ต้องผ่าน approval)
```

### แบบฝึกหัดที่ 5: EKS Cluster

```hcl
# exercise-5/main.tf
# Deploy EKS cluster ด้วย Terraform

# ใช้ community module:
# source = "terraform-aws-modules/eks/aws"

# Requirements:
# - VPC with 3 AZs
# - EKS 1.28
# - Node group with t3.medium instances (min 2, max 5)
# - Enable OIDC provider
# - Add admin user to aws-auth configmap

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"
  
  cluster_name    = "exercise-cluster"
  cluster_version = "1.28"
  
  # TODO: กรอก VPC configuration
  # TODO: กรอก node group configuration
  # TODO: กรอก additional configurations
}

# TODO: สร้าง kubeconfig output
output "kubeconfig" {
  description = "Kubeconfig for kubectl"
  sensitive   = true
  value = # ...
}
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **Terraform Basics**: HCL syntax, providers, resources
2. **State Management**: Local state, remote state, state locking
3. **Modules**: การสร้างและใช้ reusable modules
4. **Workspaces**: จัดการหลาย environment
5. **CI/CD Integration**: GitHub Actions + Terraform + OIDC
6. **Security**: ไม่เก็บ secrets ใน code, OIDC authentication
7. **Terragrunt**: DRY configuration management

**Key Takeaways:**
- ใช้ remote state เสมอใน team environment
- ใช้ modules สำหรับ reusable components
- ใช้ OIDC แทน long-lived AWS credentials
- อย่าเก็บ secrets ใน code หรือ state
- ใช้ `terraform plan` ก่อน `apply` เสมอ

**ใน Part 23** เราจะเรียนรู้เรื่อง Ansible สำหรับ Configuration Management

---

## แหล่งอ้างอิง

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Registry](https://registry.terraform.io/)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terragrunt Documentation](https://terragrunt.gruntwork.io/)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)

# Part 66: Infrastructure Testing ด้วย Terratest

## บทนำ: ทำไมต้อง Test Infrastructure?

เมื่อ Infrastructure as Code (IaC) กลายเป็น standard practice, การทดสอบ infrastructure code จึงมีความสำคัญไม่ต่างจาก application code

### ปัญหาหากไม่ Test Infrastructure

1. **Drift**: Infrastructure จริงไม่ตรงกับ code
2. **Silent Failures**: Module ทำงานผิดแต่ไม่มีใครรู้
3. **Security Misconfiguration**: Port เปิดผิด, permission กว้างเกิน
4. **Expensive Mistakes**: Deploy infrastructure ผิดพลาด → cost สูง
5. **Cross-Module Issues**: Module A และ B ทำงานร่วมกันได้ไหม?

### Levels of Infrastructure Testing

```
Level 1: Static Analysis
  ├── terraform validate
  ├── terraform fmt
  ├── tflint
  └── checkov/tfsec (security)

Level 2: Unit Tests (Terratest)
  ├── ทดสอบ module แต่ละตัว
  ├── Mock external dependencies
  └── เร็ว, ไม่สร้าง real resources

Level 3: Integration Tests (Terratest)
  ├── Deploy จริงบน cloud
  ├── ทดสอบ connectivity
  └── ตรวจสอบ outputs

Level 4: End-to-End Tests
  ├── Deploy complete environment
  ├── ทดสอบ application behavior
  └── Load testing
```

---

## 1. Terratest คืออะไร?

Terratest เป็น Go library สำหรับ testing infrastructure code โดยรองรับ Terraform, Kubernetes, Docker, AWS, GCP, Azure และอื่น ๆ

### 1.1 Setup Go Environment

```bash
# ติดตั้ง Go (>= 1.21)
wget https://go.dev/dl/go1.21.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.21.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# สร้าง Go module สำหรับ tests
mkdir -p test && cd test
go mod init github.com/company/infrastructure-tests

# Install Terratest
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/gruntwork-io/terratest/modules/aws
go get github.com/gruntwork-io/terratest/modules/http-helper
go get github.com/gruntwork-io/terratest/modules/retry
go get github.com/stretchr/testify/assert
go get github.com/stretchr/testify/require
```

### 1.2 โครงสร้าง Infrastructure Test Project

```
infrastructure/
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── rds/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── test/
    ├── go.mod
    ├── go.sum
    ├── unit/
    │   ├── vpc_test.go
    │   └── rds_test.go
    ├── integration/
    │   ├── vpc_integration_test.go
    │   └── eks_integration_test.go
    └── e2e/
        └── full_stack_test.go
```

---

## 2. Terraform Modules

### 2.1 VPC Module ที่จะทดสอบ

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-${var.azs[count.index]}"
    Type = "Public"
  })
}

resource "aws_subnet" "private" {
  count = length(var.private_subnets)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]
  
  tags = merge(var.tags, {
    Name = "${var.name}-private-${var.azs[count.index]}"
    Type = "Private"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}

resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? length(var.public_subnets) : 0
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(var.tags, {
    Name = "${var.name}-nat-${var.azs[count.index]}"
  })
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? length(var.public_subnets) : 0
  
  domain = "vpc"
  
  tags = merge(var.tags, {
    Name = "${var.name}-eip-nat-${var.azs[count.index]}"
  })
}

# Route tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(var.tags, {
    Name = "${var.name}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

```hcl
# modules/vpc/variables.tf
variable "name" {
  description = "Name prefix for all resources"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr must be a valid CIDR block."
  }
}

variable "public_subnets" {
  description = "List of CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnets" {
  description = "List of CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "azs" {
  description = "List of availability zones"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Create NAT gateways for private subnets"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ids" {
  description = "IDs of NAT gateways"
  value       = aws_nat_gateway.main[*].id
}
```

---

## 3. เขียน Terratest Tests

### 3.1 Unit Tests สำหรับ VPC Module

```go
// test/unit/vpc_test.go
package unit

import (
    "testing"
    "fmt"
    "strings"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

const awsRegion = "ap-southeast-1"

// TestVPCCreation ทดสอบการสร้าง VPC
func TestVPCCreation(t *testing.T) {
    t.Parallel()
    
    // Unique name เพื่อหลีกเลี่ยง conflicts ระหว่าง parallel tests
    uniqueID := terraform.UniqueID()
    name := fmt.Sprintf("test-vpc-%s", strings.ToLower(uniqueID))
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        // Path ไปยัง Terraform module
        TerraformDir: "../../modules/vpc",
        
        // Input variables
        Vars: map[string]interface{}{
            "name":    name,
            "vpc_cidr": "10.0.0.0/16",
            "public_subnets": []string{
                "10.0.1.0/24",
                "10.0.2.0/24",
            },
            "private_subnets": []string{
                "10.0.10.0/24",
                "10.0.11.0/24",
            },
            "azs": []string{
                "ap-southeast-1a",
                "ap-southeast-1b",
            },
            "enable_nat_gateway": false,  // ปิด NAT เพื่อประหยัด cost ใน test
            "tags": map[string]string{
                "Environment": "test",
                "ManagedBy":   "terratest",
            },
        },
        
        // Environment variables สำหรับ AWS
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": awsRegion,
        },
    })
    
    // Cleanup เมื่อ test เสร็จ
    defer terraform.Destroy(t, terraformOptions)
    
    // Deploy
    terraform.InitAndApply(t, terraformOptions)
    
    // ==================== Assertions ====================
    
    // ดึง outputs
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    vpcCIDR := terraform.Output(t, terraformOptions, "vpc_cidr")
    publicSubnetIDs := terraform.OutputList(t, terraformOptions, "public_subnet_ids")
    privateSubnetIDs := terraform.OutputList(t, terraformOptions, "private_subnet_ids")
    
    // ตรวจสอบ VPC
    assert.NotEmpty(t, vpcID, "VPC ID should not be empty")
    assert.Equal(t, "10.0.0.0/16", vpcCIDR, "VPC CIDR should match")
    
    // ตรวจสอบ subnets count
    assert.Equal(t, 2, len(publicSubnetIDs), "Should have 2 public subnets")
    assert.Equal(t, 2, len(privateSubnetIDs), "Should have 2 private subnets")
    
    // ตรวจสอบ VPC attributes ผ่าน AWS SDK
    vpc := aws.GetVpcById(t, vpcID, awsRegion)
    require.NotNil(t, vpc, "VPC should exist")
    
    assert.True(t, *vpc.EnableDnsHostnames, "DNS hostnames should be enabled")
    assert.True(t, *vpc.EnableDnsSupport, "DNS support should be enabled")
    
    // ตรวจสอบ tags
    tags := aws.GetTagsForVpc(t, vpcID, awsRegion)
    assert.Equal(t, name+"-vpc", tags["Name"])
    assert.Equal(t, "test", tags["Environment"])
    
    // ตรวจสอบ subnets
    for _, subnetID := range publicSubnetIDs {
        subnet := aws.GetSubnetById(t, subnetID, awsRegion)
        assert.True(t, *subnet.MapPublicIpOnLaunch, 
            "Public subnet should map public IPs")
    }
    
    for _, subnetID := range privateSubnetIDs {
        subnet := aws.GetSubnetById(t, subnetID, awsRegion)
        assert.False(t, *subnet.MapPublicIpOnLaunch, 
            "Private subnet should not map public IPs")
    }
}

// TestVPCNATGateway ทดสอบ NAT Gateway
func TestVPCNATGateway(t *testing.T) {
    t.Parallel()
    
    uniqueID := terraform.UniqueID()
    name := fmt.Sprintf("test-vpc-nat-%s", strings.ToLower(uniqueID))
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../../modules/vpc",
        Vars: map[string]interface{}{
            "name": name,
            "vpc_cidr": "10.1.0.0/16",
            "public_subnets": []string{"10.1.1.0/24"},
            "private_subnets": []string{"10.1.10.0/24"},
            "azs": []string{"ap-southeast-1a"},
            "enable_nat_gateway": true,
        },
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": awsRegion,
        },
    })
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    natGatewayIDs := terraform.OutputList(t, terraformOptions, "nat_gateway_ids")
    assert.Equal(t, 1, len(natGatewayIDs), "Should have 1 NAT gateway")
    assert.NotEmpty(t, natGatewayIDs[0], "NAT gateway ID should not be empty")
}
```

### 3.2 Integration Tests สำหรับ Network Connectivity

```go
// test/integration/vpc_integration_test.go
package integration

import (
    "testing"
    "fmt"
    "time"
    "strings"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/http-helper"
    "github.com/gruntwork-io/terratest/modules/retry"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// TestVPCEndToEndConnectivity ทดสอบ connectivity ของ VPC
func TestVPCEndToEndConnectivity(t *testing.T) {
    t.Parallel()
    
    uniqueID := terraform.UniqueID()
    name := fmt.Sprintf("test-network-%s", strings.ToLower(uniqueID))
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../../environments/test",
        Vars: map[string]interface{}{
            "name":    name,
            "region":  "ap-southeast-1",
        },
    })
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    // ดึง outputs
    publicInstanceIP := terraform.Output(t, terraformOptions, "public_instance_ip")
    privateInstanceIP := terraform.Output(t, terraformOptions, "private_instance_ip")
    
    // ตรวจสอบว่า public instance เข้าถึงได้จากภายนอก
    t.Log("Testing public instance connectivity...")
    
    httpHelper.HttpGetWithRetry(t, 
        fmt.Sprintf("http://%s:8080/health", publicInstanceIP),
        nil,
        3,  // max retries
        5 * time.Second,  // sleep between retries
        200,  // expected status code
        "healthy",  // expected body substring
    )
    
    // ตรวจสอบว่า private instance ไม่เข้าถึงได้จากภายนอก
    t.Log("Testing private instance is not publicly accessible...")
    
    _, err := httpHelper.HttpGetE(t, 
        fmt.Sprintf("http://%s:8080/health", privateInstanceIP),
        nil,
    )
    assert.Error(t, err, "Private instance should not be publicly accessible")
    
    // ทดสอบว่า private instance เข้าถึง internet ผ่าน NAT ได้
    t.Log("Testing private instance outbound connectivity via NAT...")
    
    output := aws.GetSsmOutput(t, name+"-private-instance", 
        "curl -s https://api.ipify.org", "ap-southeast-1")
    
    require.NotEmpty(t, output, "Private instance should have outbound internet access")
}
```

### 3.3 RDS Module Tests

```go
// test/integration/rds_test.go
package integration

import (
    "database/sql"
    "fmt"
    "testing"
    "time"
    "strings"
    
    _ "github.com/lib/pq"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/retry"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestRDSModule(t *testing.T) {
    t.Parallel()
    
    uniqueID := terraform.UniqueID()
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../../modules/rds",
        Vars: map[string]interface{}{
            "identifier":     fmt.Sprintf("test-db-%s", strings.ToLower(uniqueID)),
            "engine":         "postgres",
            "engine_version": "15.3",
            "instance_class": "db.t3.micro",
            "username":       "testuser",
            "password":       generateSecurePassword(),
            "allocated_storage": 20,
            "skip_final_snapshot": true,
            "deletion_protection": false,
        },
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "ap-southeast-1",
        },
    })
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    // ==================== RDS Assertions ====================
    
    endpoint := terraform.Output(t, terraformOptions, "endpoint")
    port := terraform.Output(t, terraformOptions, "port")
    dbName := terraform.Output(t, terraformOptions, "db_name")
    
    assert.NotEmpty(t, endpoint)
    assert.Equal(t, "5432", port)
    
    // ทดสอบว่า DB เข้าถึงได้และ query ได้
    t.Log("Testing database connectivity...")
    
    dbURL := fmt.Sprintf(
        "host=%s port=%s dbname=%s user=%s password=%s sslmode=require",
        endpoint, port, dbName,
        terraform.Output(t, terraformOptions, "username"),
        terraformOptions.Vars["password"],
    )
    
    // รอให้ RDS พร้อม
    var db *sql.DB
    
    retry.DoWithRetry(t, "Connect to RDS", 10, 30*time.Second, func() (string, error) {
        var err error
        db, err = sql.Open("postgres", dbURL)
        if err != nil {
            return "", err
        }
        return "", db.Ping()
    })
    
    defer db.Close()
    
    // Run test queries
    var version string
    err := db.QueryRow("SELECT version()").Scan(&version)
    require.NoError(t, err, "Should be able to query database")
    assert.Contains(t, version, "PostgreSQL 15")
    
    // ทดสอบ write/read
    _, err = db.Exec("CREATE TABLE IF NOT EXISTS test_table (id SERIAL PRIMARY KEY, value TEXT)")
    require.NoError(t, err, "Should be able to create table")
    
    _, err = db.Exec("INSERT INTO test_table (value) VALUES ($1)", "test-value")
    require.NoError(t, err, "Should be able to insert data")
    
    var value string
    err = db.QueryRow("SELECT value FROM test_table LIMIT 1").Scan(&value)
    require.NoError(t, err, "Should be able to query data")
    assert.Equal(t, "test-value", value)
    
    // ทดสอบ encryption (ตรวจสอบว่า storage encrypted)
    dbInstance := aws.GetRdsInstance(t, 
        terraform.Output(t, terraformOptions, "identifier"),
        "ap-southeast-1",
    )
    
    assert.True(t, *dbInstance.StorageEncrypted, "RDS storage should be encrypted")
    assert.NotEmpty(t, *dbInstance.KmsKeyId, "Should have KMS key")
    
    // ตรวจสอบ backup
    assert.GreaterOrEqual(t, *dbInstance.BackupRetentionPeriod, int64(7), 
        "Backup retention should be at least 7 days")
    
    // ตรวจสอบ Multi-AZ สำหรับ production
    if strings.Contains(uniqueID, "prod") {
        assert.True(t, *dbInstance.MultiAZ, "Production RDS should be Multi-AZ")
    }
}

func generateSecurePassword() string {
    // Generate random secure password for testing
    return "T3st@" + terraform.UniqueID()[:8]
}
```

### 3.4 EKS Module Tests

```go
// test/integration/eks_test.go
package integration

import (
    "fmt"
    "testing"
    "time"
    "strings"
    
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/k8s"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func TestEKSCluster(t *testing.T) {
    t.Parallel()
    
    uniqueID := terraform.UniqueID()
    clusterName := fmt.Sprintf("test-eks-%s", strings.ToLower(uniqueID))
    
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../../modules/eks",
        Vars: map[string]interface{}{
            "cluster_name":    clusterName,
            "cluster_version": "1.28",
            "node_groups": map[string]interface{}{
                "general": map[string]interface{}{
                    "instance_types": []string{"t3.medium"},
                    "min_size":       1,
                    "max_size":       3,
                    "desired_size":   2,
                },
            },
        },
        EnvVars: map[string]string{
            "AWS_DEFAULT_REGION": "ap-southeast-1",
        },
    })
    
    defer terraform.Destroy(t, terraformOptions)
    
    // EKS ใช้เวลานาน
    terraform.InitAndApplyE(t, terraformOptions)
    
    // ==================== EKS Cluster Assertions ====================
    
    // ตรวจสอบ cluster status
    cluster := aws.GetEksCluster(t, clusterName, "ap-southeast-1")
    assert.Equal(t, "ACTIVE", *cluster.Status)
    assert.Equal(t, "1.28", *cluster.Version)
    
    // ดึง kubeconfig
    kubeConfigPath := fmt.Sprintf("/tmp/kubeconfig-%s", uniqueID)
    aws.GetEksClusterKubeconfig(t, clusterName, "ap-southeast-1", kubeConfigPath)
    
    // Setup k8s client
    kubectlOptions := k8s.NewKubectlOptions("", kubeConfigPath, "")
    
    // ตรวจสอบ nodes
    t.Log("Waiting for nodes to be ready...")
    k8s.WaitUntilAllNodesReady(t, kubectlOptions, 20, 30*time.Second)
    
    nodes := k8s.GetNodes(t, kubectlOptions)
    assert.Equal(t, 2, len(nodes), "Should have 2 worker nodes")
    
    for _, node := range nodes {
        assert.Equal(t, corev1.ConditionTrue, 
            k8s.GetNodeStatus(node, corev1.NodeReady),
            "Node should be ready",
        )
    }
    
    // ทดสอบ deploy sample workload
    t.Log("Testing workload deployment...")
    
    k8s.CreateNamespace(t, kubectlOptions, "test")
    defer k8s.DeleteNamespace(t, kubectlOptions, "test")
    
    testOptions := k8s.NewKubectlOptions("", kubeConfigPath, "test")
    
    // Deploy nginx
    k8s.KubectlApply(t, testOptions, "../../test/fixtures/nginx-deployment.yaml")
    defer k8s.KubectlDelete(t, testOptions, "../../test/fixtures/nginx-deployment.yaml")
    
    // Wait for pods
    k8s.WaitUntilNumPodsCreated(t, testOptions, metav1.ListOptions{
        LabelSelector: "app=nginx",
    }, 2, 10, 30*time.Second)
    
    pods := k8s.ListPods(t, testOptions, metav1.ListOptions{
        LabelSelector: "app=nginx",
    })
    
    for _, pod := range pods {
        k8s.WaitUntilPodAvailable(t, testOptions, pod.Name, 10, 30*time.Second)
    }
    
    // ทดสอบ service
    service := k8s.GetService(t, testOptions, "nginx")
    require.NotNil(t, service)
    
    // ตรวจสอบ security features
    assert.True(t, *cluster.ResourcesVpcConfig.EndpointPublicAccess,
        "Public endpoint should be accessible")
    
    // Verify RBAC is enabled
    k8s.RunKubectl(t, testOptions, "auth", "can-i", "get", "pods", "--all-namespaces")
}
```

---

## 4. Test Isolation และ Cleanup

### 4.1 Test Fixtures

```hcl
# test/fixtures/complete-vpc/main.tf
# ใช้สำหรับ integration tests เท่านั้น

module "vpc" {
  source = "../../../modules/vpc"
  
  name    = var.name
  vpc_cidr = "10.99.0.0/16"
  
  azs = ["ap-southeast-1a", "ap-southeast-1b"]
  
  public_subnets  = ["10.99.1.0/24", "10.99.2.0/24"]
  private_subnets = ["10.99.10.0/24", "10.99.11.0/24"]
  
  enable_nat_gateway = var.enable_nat_gateway
  
  tags = {
    Environment = "test"
    ManagedBy   = "terratest"
    TestID      = var.test_id
  }
}

variable "name" {}
variable "test_id" {}
variable "enable_nat_gateway" {
  default = false
}
```

### 4.2 Parallel Test Setup

```go
// test/helpers/helpers.go
package helpers

import (
    "fmt"
    "os"
    "testing"
    "time"
    
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/require"
)

// TestConfig รวม configuration สำหรับ tests
type TestConfig struct {
    Region    string
    UniqueID  string
    Name      string
    Tags      map[string]string
}

// NewTestConfig สร้าง test configuration ใหม่
func NewTestConfig(prefix string) TestConfig {
    uniqueID := terraform.UniqueID()
    
    return TestConfig{
        Region:   getEnvOrDefault("AWS_REGION", "ap-southeast-1"),
        UniqueID: uniqueID,
        Name:     fmt.Sprintf("%s-%s", prefix, uniqueID),
        Tags: map[string]string{
            "Environment": "test",
            "ManagedBy":   "terratest",
            "CreatedAt":   time.Now().Format(time.RFC3339),
        },
    }
}

func getEnvOrDefault(key, defaultVal string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return defaultVal
}

// SkipIfNotIntegration ข้าม test ถ้าไม่ใช่ integration test
func SkipIfNotIntegration(t *testing.T) {
    if os.Getenv("RUN_INTEGRATION_TESTS") != "true" {
        t.Skip("Skipping integration test. Set RUN_INTEGRATION_TESTS=true to run.")
    }
}

// RequireAWSCredentials ตรวจสอบว่ามี AWS credentials
func RequireAWSCredentials(t *testing.T) {
    require.NotEmpty(t, os.Getenv("AWS_ACCESS_KEY_ID"), 
        "AWS_ACCESS_KEY_ID must be set")
    require.NotEmpty(t, os.Getenv("AWS_SECRET_ACCESS_KEY"), 
        "AWS_SECRET_ACCESS_KEY must be set")
}
```

---

## 5. CI Integration สำหรับ Infrastructure Tests

### 5.1 GitHub Actions Pipeline

```yaml
# .github/workflows/infrastructure-test.yml
name: Infrastructure Tests

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/**'
      - '.github/workflows/infrastructure-test.yml'
  pull_request:
    branches: [main]
    paths: ['infrastructure/**']
  schedule:
    # Run nightly integration tests
    - cron: '0 20 * * *'  # 03:00 Bangkok time

jobs:
  # ==================== Static Analysis ====================
  static-analysis:
    name: Static Analysis
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.6.0'
      
      - name: Terraform Format Check
        run: |
          cd infrastructure
          terraform fmt -check -recursive
      
      - name: Terraform Validate
        run: |
          for dir in $(find infrastructure/modules -name "*.tf" -exec dirname {} \; | sort -u); do
            echo "Validating $dir"
            cd "$dir"
            terraform init -backend=false
            terraform validate
            cd - > /dev/null
          done
      
      - name: tflint
        uses: terraform-linters/setup-tflint@v4
        
      - name: Run tflint
        run: |
          cd infrastructure
          tflint --recursive
      
      - name: Checkov security scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: infrastructure/modules
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: checkov.sarif

  # ==================== Unit Tests ====================
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: static-analysis
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
          cache: true
          cache-dependency-path: infrastructure/test/go.sum
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.6.0'
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_TEST_ROLE_ARN }}
          aws-region: ap-southeast-1
      
      - name: Run unit tests
        run: |
          cd infrastructure/test
          go test ./unit/... \
            -v \
            -timeout 30m \
            -run TestVPC \
            -parallel 4
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: unit-test-results
          path: infrastructure/test/unit/

  # ==================== Integration Tests ====================
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    if: github.event_name == 'schedule' || github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
          cache: true
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.6.0'
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_TEST_ROLE_ARN }}
          aws-region: ap-southeast-1
      
      - name: Run integration tests
        env:
          RUN_INTEGRATION_TESTS: "true"
        run: |
          cd infrastructure/test
          go test ./integration/... \
            -v \
            -timeout 60m \
            -run TestVPCEndToEnd \
            -parallel 2
      
      - name: Ensure cleanup on failure
        if: failure()
        run: |
          echo "Tests failed, ensuring cleanup..."
          # Force destroy any leftover resources
          cd infrastructure/test
          go run ./scripts/cleanup.go --tag Environment=test --tag ManagedBy=terratest
```

---

## 6. Test Costs Management

### 6.1 Terratest Cost Controls

```go
// test/helpers/cost_guard.go
package helpers

import (
    "testing"
    "os"
)

// EnforceTestBudget ตรวจสอบว่า test จะไม่ใช้ budget เกิน
func EnforceTestBudget(t *testing.T) {
    // ตรวจสอบว่า test รันในเวลาที่กำหนด (off-peak hours)
    if os.Getenv("ALLOW_EXPENSIVE_TESTS") != "true" {
        t.Skip("Expensive infrastructure tests disabled. Set ALLOW_EXPENSIVE_TESTS=true")
    }
}

// UseSpotInstances ปรับ config ให้ใช้ spot instances ใน test
func UseSpotInstances(vars map[string]interface{}) map[string]interface{} {
    vars["use_spot_instances"] = true
    vars["spot_price"] = "0.05"
    return vars
}

// MinimalResources ลด resource size ให้เล็กที่สุดสำหรับ test
func MinimalResources(vars map[string]interface{}) map[string]interface{} {
    vars["instance_type"] = "t3.micro"
    vars["db_instance_class"] = "db.t3.micro"
    vars["node_instance_type"] = "t3.small"
    vars["min_nodes"] = 1
    vars["max_nodes"] = 2
    return vars
}
```

---

## 7. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Test VPC Module

```go
// TODO: เขียน tests สำหรับ scenarios ต่อไปนี้:

// 1. TestVPCWithCustomCIDR
//    - ทดสอบ VPC ด้วย custom CIDR
//    - ตรวจสอบ subnets ไม่ overlap

// 2. TestVPCWithMultipleNATGateways  
//    - NAT gateway ต่อ AZ (high availability)
//    - ตรวจสอบ route tables ถูก configure

// 3. TestVPCPeeringConnection
//    - สร้าง 2 VPCs
//    - สร้าง peering connection
//    - ทดสอบ connectivity ระหว่าง VPCs

// 4. TestVPCFlowLogs
//    - ตรวจสอบว่า Flow Logs ถูกเปิดใช้งาน
//    - Logs ถูกส่งไป CloudWatch

func TestVPCWithCustomCIDR(t *testing.T) {
    // TODO: implement
}
```

### แบบฝึกหัดที่ 2: Security Testing

```go
// สร้าง security tests ที่ตรวจสอบ:

// 1. Security Groups ไม่มี inbound 0.0.0.0/0 บน sensitive ports
// 2. S3 buckets ไม่เป็น public
// 3. RDS encryption enabled
// 4. EBS volumes encrypted
// 5. CloudTrail enabled
// 6. GuardDuty enabled

func TestNoPublicInboundOnSensitivePorts(t *testing.T) {
    // Deploy infrastructure
    // Get all security groups
    // Check each security group's inbound rules
    // Assert no rules with 0.0.0.0/0 on ports 22, 3306, 5432, 6379, 27017
}
```

### แบบฝึกหัดที่ 3: Cost Estimation Test

```go
// สร้าง test ที่:
// 1. Deploy module
// 2. ดึง resource list
// 3. ประมาณค่าใช้จ่าย (ใช้ infracost API)
// 4. Assert ว่า monthly cost < $100

func TestModuleCostEstimate(t *testing.T) {
    // hint: ใช้ infracost CLI หรือ Infracost Go client
    // https://www.infracost.io/docs/
}
```

### สรุปบทที่ 66

ในบทนี้เราได้เรียนรู้:
- **Terratest**: Go library สำหรับ testing Terraform code
- **Infrastructure Testing Levels**: Static analysis → Unit → Integration → E2E
- **VPC Module Tests**: ทดสอบ networking configuration
- **RDS Module Tests**: ทดสอบ database resources
- **EKS Module Tests**: ทดสอบ Kubernetes cluster
- **Test Isolation**: ทำให้ tests ไม่รบกวนกัน
- **CI Integration**: GitHub Actions สำหรับ running infrastructure tests
- **Cost Management**: ควบคุมค่าใช้จ่ายใน test environment

บทถัดไปเราจะเรียนรู้ Policy as Code ด้วย OPA (Open Policy Agent)

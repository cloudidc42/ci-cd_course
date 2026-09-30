# Part 67: Policy as Code ด้วย OPA (Open Policy Agent)

## บทนำ: Policy as Code คืออะไร?

Policy as Code คือ practice ของการนำ organizational policies, compliance rules, และ security requirements มาเขียนในรูปแบบ code ที่สามารถ test, version control, และ automate ได้

### ทำไมต้อง Policy as Code?

```
ปัญหาแบบเดิม:
- Policy เขียนเป็น Word document
- ทีมรักษาความปลอดภัยต้อง review ทุก deployment manually
- Human error สูง
- ไม่มีการ enforce อย่างสม่ำเสมอ
- Scale ไม่ได้

Policy as Code แก้ได้:
- Policy เป็น code → สามารถ test ได้
- Automated enforcement ใน CI/CD pipeline
- Version controlled → tracking changes
- Consistent enforcement
- Fast feedback สำหรับ developers
```

---

## 1. OPA (Open Policy Agent)

### 1.1 OPA คืออะไร?

OPA เป็น general-purpose policy engine ที่ใช้ภาษา Rego สำหรับเขียน policies สามารถ integrate กับ Kubernetes, Terraform, HTTP APIs, Microservices และอื่น ๆ

### 1.2 Rego Language Basics

```rego
# policies/basics.rego
package basics

# Simple rule
allow := true

# Rule with condition
is_admin := true {
    input.user.role == "admin"
}

# Rule with multiple conditions (AND)
can_deploy := true {
    input.user.role == "developer"
    input.environment != "production"
}

# Rule with OR (multiple definitions)
is_authorized := true {
    input.user.role == "admin"
}

is_authorized := true {
    input.user.role == "developer"
    input.environment == "dev"
}

# Comprehension
approved_environments := {env | 
    env := input.environments[_]
    env != "production"
}

# Functions
in_subnet(ip, subnet) := true {
    net.cidr_contains(subnet, ip)
}

# Array/Set comprehension
blocked_users := {user.id |
    user := input.users[_]
    user.status == "suspended"
}
```

### 1.3 Install OPA

```bash
# ติดตั้ง OPA
curl -L -o opa https://openpolicyagent.org/downloads/v0.58.0/opa_linux_amd64_static
chmod 755 ./opa
sudo mv opa /usr/local/bin/

# ทดสอบ
opa version

# Install Conftest (สำหรับใช้กับ Kubernetes/Terraform)
curl -L https://github.com/open-policy-agent/conftest/releases/download/v0.45.0/conftest_0.45.0_Linux_x86_64.tar.gz | tar xz
sudo mv conftest /usr/local/bin/
```

---

## 2. Validating Kubernetes Manifests

### 2.1 K8s Security Policies

```rego
# policies/kubernetes/security.rego
package kubernetes.security

import future.keywords.in
import future.keywords.if
import future.keywords.contains

# ==================== Deny Rules ====================

# ห้าม containers รันเป็น root
deny[msg] {
    input.kind == "Pod"
    container := input.spec.containers[_]
    not container.securityContext.runAsNonRoot
    msg := sprintf(
        "Container '%v' must set securityContext.runAsNonRoot=true",
        [container.name]
    )
}

# ห้ามใช้ latest tag
deny[msg] {
    input.kind in {"Deployment", "StatefulSet", "DaemonSet"}
    container := input.spec.template.spec.containers[_]
    endswith(container.image, ":latest")
    msg := sprintf(
        "Container '%v' must not use 'latest' image tag",
        [container.name]
    )
}

deny[msg] {
    input.kind in {"Deployment", "StatefulSet", "DaemonSet"}
    container := input.spec.template.spec.containers[_]
    not contains(container.image, ":")
    msg := sprintf(
        "Container '%v' must specify an explicit image tag",
        [container.name]
    )
}

# ต้องมี resource limits
deny[msg] {
    input.kind in {"Deployment", "StatefulSet", "DaemonSet"}
    container := input.spec.template.spec.containers[_]
    not container.resources.limits.memory
    msg := sprintf(
        "Container '%v' must set memory limits",
        [container.name]
    )
}

deny[msg] {
    input.kind in {"Deployment", "StatefulSet", "DaemonSet"}
    container := input.spec.template.spec.containers[_]
    not container.resources.limits.cpu
    msg := sprintf(
        "Container '%v' must set CPU limits",
        [container.name]
    )
}

# ห้าม privileged containers
deny[msg] {
    input.kind == "Pod"
    container := input.spec.containers[_]
    container.securityContext.privileged == true
    msg := sprintf(
        "Container '%v' must not be privileged",
        [container.name]
    )
}

# ต้อง readOnlyRootFilesystem
deny[msg] {
    input.kind in {"Deployment", "StatefulSet"}
    container := input.spec.template.spec.containers[_]
    not container.securityContext.readOnlyRootFilesystem
    msg := sprintf(
        "Container '%v' should set readOnlyRootFilesystem=true",
        [container.name]
    )
}

# ==================== Warning Rules ====================

warn[msg] {
    input.kind in {"Deployment", "StatefulSet"}
    not input.spec.template.spec.securityContext.runAsGroup
    msg := "Pod should set securityContext.runAsGroup"
}

warn[msg] {
    input.kind == "Service"
    input.spec.type == "LoadBalancer"
    msg := "LoadBalancer service type exposes service to internet"
}

# ==================== Namespace Policies ====================

deny[msg] {
    input.kind in {"Deployment", "StatefulSet", "DaemonSet"}
    input.metadata.namespace == "default"
    msg := "Resources should not be deployed in 'default' namespace"
}

deny[msg] {
    input.kind == "Pod"
    not input.metadata.namespace
    msg := "Pod must specify a namespace"
}
```

### 2.2 Network Policy Validation

```rego
# policies/kubernetes/network.rego
package kubernetes.network

import future.keywords.in

# ต้องมี NetworkPolicy ในทุก namespace
deny[msg] {
    input.kind == "Namespace"
    namespace := input.metadata.name
    not has_network_policy(namespace)
    msg := sprintf(
        "Namespace '%v' must have a NetworkPolicy",
        [namespace]
    )
}

has_network_policy(namespace) {
    input.kind == "NetworkPolicy"
    input.metadata.namespace == namespace
}

# NetworkPolicy ต้องไม่ allow all ingress
deny[msg] {
    input.kind == "NetworkPolicy"
    allows_all_ingress
    msg := sprintf(
        "NetworkPolicy '%v' in '%v' allows all ingress traffic",
        [input.metadata.name, input.metadata.namespace]
    )
}

allows_all_ingress {
    rule := input.spec.ingress[_]
    count(rule) == 0  # Empty ingress rule allows all
}

allows_all_ingress {
    rule := input.spec.ingress[_]
    from := rule.from[_]
    from == {}  # Empty from allows all sources
}
```

### 2.3 Resource Quota Policy

```rego
# policies/kubernetes/resources.rego
package kubernetes.resources

import future.keywords.in

# Resource limits
max_limits := {
    "production": {
        "cpu": "8000m",
        "memory": "16Gi"
    },
    "staging": {
        "cpu": "4000m", 
        "memory": "8Gi"
    },
    "development": {
        "cpu": "2000m",
        "memory": "4Gi"
    }
}

# ตรวจสอบ resource limits ไม่เกิน max
deny[msg] {
    input.kind in {"Deployment", "StatefulSet"}
    container := input.spec.template.spec.containers[_]
    env := input.metadata.labels.environment
    
    limit := max_limits[env]
    
    cpu_millicores(container.resources.limits.cpu) > cpu_millicores(limit.cpu)
    
    msg := sprintf(
        "Container '%v' CPU limit %v exceeds maximum %v for %v environment",
        [container.name, container.resources.limits.cpu, limit.cpu, env]
    )
}

cpu_millicores(cpu) = millicores {
    endswith(cpu, "m")
    millicores := to_number(trim_suffix(cpu, "m"))
}

cpu_millicores(cpu) = millicores {
    not endswith(cpu, "m")
    millicores := to_number(cpu) * 1000
}

# ต้องมี labels สำคัญ
required_labels := {
    "app",
    "version", 
    "environment",
    "team"
}

deny[msg] {
    input.kind in {"Deployment", "StatefulSet"}
    required_label := required_labels[_]
    not input.metadata.labels[required_label]
    msg := sprintf(
        "%v '%v' missing required label: %v",
        [input.kind, input.metadata.name, required_label]
    )
}
```

---

## 3. Validating Terraform Plans

### 3.1 Setup Conftest สำหรับ Terraform

```bash
# สร้าง policy directory
mkdir -p policies/terraform

# Run conftest กับ Terraform plan
terraform plan -out=tfplan.binary
terraform show -json tfplan.binary > tfplan.json
conftest test tfplan.json --policy policies/terraform/
```

### 3.2 Terraform Security Policies

```rego
# policies/terraform/security.rego
package terraform.security

import future.keywords.in
import future.keywords.if

# ==================== AWS S3 Policies ====================

# S3 buckets ต้อง enable versioning
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket"
    resource.change.after.versioning[_].enabled != true
    msg := sprintf(
        "S3 bucket '%v' must have versioning enabled",
        [resource.address]
    )
}

# S3 buckets ต้องไม่เป็น public
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_s3_bucket_acl"
    resource.change.after.acl in {"public-read", "public-read-write", "authenticated-read"}
    msg := sprintf(
        "S3 bucket '%v' must not be public",
        [resource.address]
    )
}

# ==================== AWS Security Group ====================

# Security groups ต้องไม่เปิด port 22 จาก 0.0.0.0/0
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group"
    rule := resource.change.after.ingress[_]
    rule.from_port <= 22
    rule.to_port >= 22
    "0.0.0.0/0" in rule.cidr_blocks
    msg := sprintf(
        "Security group '%v' must not allow SSH (port 22) from 0.0.0.0/0",
        [resource.address]
    )
}

# ห้ามเปิด port ไปยัง internet บน sensitive services
sensitive_ports := {
    3306,  # MySQL
    5432,  # PostgreSQL
    6379,  # Redis
    27017, # MongoDB
    9200,  # Elasticsearch
}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_security_group"
    rule := resource.change.after.ingress[_]
    port := sensitive_ports[_]
    rule.from_port <= port
    rule.to_port >= port
    "0.0.0.0/0" in rule.cidr_blocks
    msg := sprintf(
        "Security group '%v' must not expose port %v to 0.0.0.0/0",
        [resource.address, port]
    )
}

# ==================== AWS RDS ====================

# RDS ต้อง encrypt storage
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.after.storage_encrypted != true
    msg := sprintf(
        "RDS instance '%v' must have storage encryption enabled",
        [resource.address]
    )
}

# RDS ต้องมี backup retention >= 7 days
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.after.backup_retention_period < 7
    msg := sprintf(
        "RDS instance '%v' backup retention must be at least 7 days",
        [resource.address]
    )
}

# RDS ต้องไม่ public
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_db_instance"
    resource.change.after.publicly_accessible == true
    msg := sprintf(
        "RDS instance '%v' must not be publicly accessible",
        [resource.address]
    )
}

# ==================== AWS EC2 ====================

# EC2 ต้องไม่เป็น public
deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_instance"
    resource.change.after.associate_public_ip_address == true
    not is_bastion(resource)
    msg := sprintf(
        "EC2 instance '%v' must not have public IP (use NAT or bastion)",
        [resource.address]
    )
}

is_bastion(resource) {
    resource.change.after.tags.Role == "bastion"
}

# ==================== Cost Controls ====================

# ห้ามใช้ instance types ที่ราคาแพงเกิน
expensive_instances := {
    "p4d.24xlarge",
    "p3.16xlarge",
    "x1e.32xlarge",
    "u-24tb1.metal"
}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type == "aws_instance"
    resource.change.after.instance_type in expensive_instances
    msg := sprintf(
        "EC2 instance '%v' uses expensive instance type '%v'. Please justify.",
        [resource.address, resource.change.after.instance_type]
    )
}
```

### 3.3 Tagging Policies

```rego
# policies/terraform/tagging.rego
package terraform.tagging

import future.keywords.in

required_tags := {
    "Environment",
    "Team",
    "Project",
    "ManagedBy",
    "CostCenter"
}

# Resources ที่ต้อง tag
taggable_resources := {
    "aws_instance",
    "aws_db_instance",
    "aws_s3_bucket",
    "aws_vpc",
    "aws_subnet",
    "aws_security_group",
    "aws_eks_cluster",
    "aws_elasticache_cluster"
}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type in taggable_resources
    required_tag := required_tags[_]
    not resource.change.after.tags[required_tag]
    msg := sprintf(
        "Resource '%v' (%v) missing required tag: '%v'",
        [resource.address, resource.type, required_tag]
    )
}

# Environment tag ต้องมีค่าที่ถูกต้อง
valid_environments := {"development", "staging", "production", "shared"}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type in taggable_resources
    resource.change.after.tags.Environment
    not resource.change.after.tags.Environment in valid_environments
    msg := sprintf(
        "Resource '%v' has invalid Environment tag '%v'. Must be one of: %v",
        [resource.address, resource.change.after.tags.Environment, valid_environments]
    )
}
```

---

## 4. Pipeline Gate Policies

### 4.1 Deployment Gate Policies

```rego
# policies/deployment/gates.rego
package deployment.gates

import future.keywords.in
import future.keywords.if

# ==================== Time-based Gates ====================

# ห้าม deploy บน production ในช่วง business hours ของ weekend
deny[msg] {
    input.environment == "production"
    input.deployment_time
    time := time.parse_rfc3339_ns(input.deployment_time)
    day := time.weekday(time)
    day in {0, 6}  # 0=Sunday, 6=Saturday
    msg := "Production deployments are not allowed on weekends"
}

# ห้าม deploy บน production ในช่วง freezing period
deny[msg] {
    input.environment == "production"
    is_in_freeze_period(input.deployment_time)
    msg := "Deployment is blocked during freeze period"
}

is_in_freeze_period(deployment_time) {
    time := time.parse_rfc3339_ns(deployment_time)
    
    # Year-end freeze: Dec 24 - Jan 2
    month := time.date(time)[1]
    day := time.date(time)[2]
    
    month == 12
    day >= 24
}

is_in_freeze_period(deployment_time) {
    time := time.parse_rfc3339_ns(deployment_time)
    
    month := time.date(time)[1]
    day := time.date(time)[2]
    
    month == 1
    day <= 2
}

# ==================== Approval Gates ====================

# Production deployment ต้องมี approvals
deny[msg] {
    input.environment == "production"
    count(input.approvals) < 2
    msg := sprintf(
        "Production deployment requires 2 approvals, got %v",
        [count(input.approvals)]
    )
}

# Approvals ต้องมาจาก different people
deny[msg] {
    input.environment == "production"
    count(input.approvals) >= 2
    approval_usernames := {a.username | a := input.approvals[_]}
    count(approval_usernames) < count(input.approvals)
    msg := "Approvals must come from different people"
}

# Approver ต้องไม่ใช่คนที่ create deployment
deny[msg] {
    input.environment == "production"
    approval := input.approvals[_]
    approval.username == input.requester
    msg := sprintf(
        "User '%v' cannot approve their own deployment",
        [approval.username]
    )
}

# ==================== Test Coverage Gates ====================

deny[msg] {
    input.environment in {"staging", "production"}
    input.test_coverage < 80
    msg := sprintf(
        "Test coverage %v%% is below minimum 80%% for %v deployment",
        [input.test_coverage, input.environment]
    )
}

# ==================== Security Scan Gates ====================

deny[msg] {
    scan := input.security_scans[_]
    scan.type == "container_scan"
    scan.critical_vulnerabilities > 0
    msg := sprintf(
        "Container has %v critical vulnerabilities. Fix before deploying.",
        [scan.critical_vulnerabilities]
    )
}

deny[msg] {
    scan := input.security_scans[_]
    scan.type == "sast"
    scan.high_findings > 0
    input.environment in {"staging", "production"}
    msg := sprintf(
        "SAST scan found %v high severity issues",
        [scan.high_findings]
    )
}
```

### 4.2 CI Pipeline Policies

```rego
# policies/pipeline/ci_requirements.rego
package pipeline.requirements

import future.keywords.in

# ==================== Required Checks ====================

required_checks := {
    "unit_tests",
    "integration_tests",
    "security_scan",
    "lint",
    "type_check"
}

required_checks_for_production := required_checks | {
    "e2e_tests",
    "performance_tests",
    "compliance_scan"
}

deny[msg] {
    input.target_environment != "production"
    required_check := required_checks[_]
    not check_passed(input.checks, required_check)
    msg := sprintf(
        "Required check '%v' has not passed",
        [required_check]
    )
}

deny[msg] {
    input.target_environment == "production"
    required_check := required_checks_for_production[_]
    not check_passed(input.checks, required_check)
    msg := sprintf(
        "Required check '%v' has not passed for production deployment",
        [required_check]
    )
}

check_passed(checks, check_name) {
    check := checks[_]
    check.name == check_name
    check.status == "passed"
}

# ==================== Branch Policies ====================

deny[msg] {
    input.target_environment == "production"
    input.source_branch != "main"
    msg := sprintf(
        "Production deployments must come from 'main' branch, got '%v'",
        [input.source_branch]
    )
}

deny[msg] {
    input.target_environment == "staging"
    not startswith(input.source_branch, "main")
    not startswith(input.source_branch, "release/")
    msg := sprintf(
        "Staging deployments must come from 'main' or 'release/*' branches",
        []
    )
}
```

---

## 5. OPA บน Kubernetes (OPA Gatekeeper)

### 5.1 Gatekeeper ConstraintTemplate

```yaml
# k8s/gatekeeper/required-labels.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
    namespaces:
      - production
      - staging
  parameters:
    labels:
      - app
      - version
      - team
      - environment
```

### 5.2 Container Security Constraint

```yaml
# k8s/gatekeeper/container-security.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8scontainersecurity
spec:
  crd:
    spec:
      names:
        kind: K8sContainerSecurity
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8scontainersecurity
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Container '%v' must not be privileged", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          endswith(container.image, ":latest")
          msg := sprintf("Container '%v' must not use latest tag", [container.name])
        }
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container '%v' must set memory limits", [container.name])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sContainerSecurity
metadata:
  name: container-security-policy
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    excludedNamespaces:
      - kube-system
      - kube-public
```

---

## 6. Testing OPA Policies

### 6.1 OPA Unit Tests

```rego
# policies/kubernetes/security_test.rego
package kubernetes.security

import future.keywords.in

# ==================== Test deny privileged ====================

test_deny_privileged_container if {
    deny[msg] with input as {
        "kind": "Pod",
        "metadata": {"name": "test-pod"},
        "spec": {
            "containers": [{
                "name": "app",
                "image": "nginx:1.21",
                "securityContext": {
                    "privileged": true
                }
            }]
        }
    }
    
    msg == "Container 'app' must not be privileged"
}

test_allow_non_privileged_container if {
    count(deny) == 0 with input as {
        "kind": "Pod",
        "metadata": {"name": "test-pod"},
        "spec": {
            "containers": [{
                "name": "app",
                "image": "nginx:1.21",
                "securityContext": {
                    "privileged": false,
                    "runAsNonRoot": true
                }
            }]
        }
    }
}

# ==================== Test latest tag ====================

test_deny_latest_tag if {
    deny[msg] with input as {
        "kind": "Deployment",
        "metadata": {"name": "test-deploy", "namespace": "production"},
        "spec": {
            "template": {
                "spec": {
                    "containers": [{
                        "name": "app",
                        "image": "nginx:latest",
                        "resources": {
                            "limits": {
                                "cpu": "500m",
                                "memory": "512Mi"
                            }
                        }
                    }]
                }
            }
        }
    }
    
    msg == "Container 'app' must not use 'latest' image tag"
}

test_allow_specific_tag if {
    # ต้องไม่มี deny สำหรับ image tag
    deny_msgs := [msg | deny[msg] with input as {
        "kind": "Deployment",
        "metadata": {"name": "test-deploy", "namespace": "app"},
        "spec": {
            "template": {
                "spec": {
                    "containers": [{
                        "name": "app",
                        "image": "nginx:1.21.6",
                        "resources": {
                            "limits": {
                                "cpu": "500m",
                                "memory": "512Mi"
                            }
                        },
                        "securityContext": {
                            "runAsNonRoot": true,
                            "readOnlyRootFilesystem": true
                        }
                    }]
                }
            }
        }
    }]
    
    # ไม่ควรมี deny เกี่ยวกับ image tag
    not "Container 'app' must not use 'latest' image tag" in deny_msgs
}
```

```bash
# รัน OPA tests
opa test policies/kubernetes/ -v

# Output:
# data.kubernetes.security.test_deny_privileged_container: PASS (1.2ms)
# data.kubernetes.security.test_allow_non_privileged_container: PASS (0.8ms)
# data.kubernetes.security.test_deny_latest_tag: PASS (1.1ms)
# data.kubernetes.security.test_allow_specific_tag: PASS (0.9ms)
# PASS: 4/4
```

### 6.2 Conftest สำหรับ Kubernetes

```bash
# Test Kubernetes manifests
conftest test k8s/deployments/ \
  --policy policies/kubernetes/ \
  --output table

# Test specific file
conftest test k8s/deployments/api-deployment.yaml \
  --policy policies/kubernetes/security.rego \
  --policy policies/kubernetes/network.rego
```

---

## 7. GitHub Actions Pipeline

```yaml
# .github/workflows/policy-check.yml
name: Policy as Code Checks

on:
  push:
    branches: [main, develop]
    paths:
      - 'k8s/**'
      - 'infrastructure/**'
      - 'policies/**'
  pull_request:
    branches: [main]

jobs:
  # ==================== K8s Policy Checks ====================
  kubernetes-policies:
    name: Kubernetes Policy Checks
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Conftest
        run: |
          curl -L https://github.com/open-policy-agent/conftest/releases/download/v0.45.0/conftest_0.45.0_Linux_x86_64.tar.gz | tar xz
          sudo mv conftest /usr/local/bin/
      
      - name: Run OPA policy tests
        run: |
          curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
          chmod 755 ./opa
          ./opa test policies/kubernetes/ -v --format pretty
      
      - name: Validate K8s manifests
        run: |
          conftest test k8s/ \
            --policy policies/kubernetes/ \
            --output github \
            --all-namespaces
      
      - name: Annotate PR with violations
        if: failure() && github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            // Parse conftest output and create PR review comments
            const violations = JSON.parse(process.env.CONFTEST_OUTPUT || '[]');
            // ... create inline comments

  # ==================== Terraform Policy Checks ====================
  terraform-policies:
    name: Terraform Policy Checks
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.6.0'
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_POLICY_CHECK_ROLE }}
          aws-region: ap-southeast-1
      
      - name: Terraform Plan
        run: |
          cd infrastructure/environments/staging
          terraform init
          terraform plan -out=tfplan.binary
          terraform show -json tfplan.binary > tfplan.json
      
      - name: Install Conftest
        run: |
          curl -L https://github.com/open-policy-agent/conftest/releases/download/v0.45.0/conftest_0.45.0_Linux_x86_64.tar.gz | tar xz
          sudo mv conftest /usr/local/bin/
      
      - name: Run Terraform policy checks
        run: |
          conftest test infrastructure/environments/staging/tfplan.json \
            --policy policies/terraform/ \
            --output table
      
      - name: Run OPA policy tests
        run: |
          ./opa test policies/terraform/ -v

  # ==================== Deployment Gate Check ====================
  deployment-gate:
    name: Check Deployment Gate Policies
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Collect deployment metadata
        id: metadata
        run: |
          cat > deployment-input.json << EOF
          {
            "environment": "staging",
            "source_branch": "${{ github.ref_name }}",
            "requester": "${{ github.actor }}",
            "deployment_time": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "test_coverage": 85,
            "checks": [
              {"name": "unit_tests", "status": "passed"},
              {"name": "integration_tests", "status": "passed"},
              {"name": "security_scan", "status": "passed"},
              {"name": "lint", "status": "passed"},
              {"name": "type_check", "status": "passed"}
            ],
            "security_scans": [
              {"type": "container_scan", "critical_vulnerabilities": 0, "high_vulnerabilities": 2},
              {"type": "sast", "high_findings": 0}
            ],
            "approvals": []
          }
          EOF
      
      - name: Check deployment gate
        run: |
          ./opa eval \
            --input deployment-input.json \
            --data policies/deployment/ \
            --format pretty \
            "data.deployment.gates.deny"
```

---

## 8. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: K8s Security Policies

```
สร้าง policy suite สำหรับ Kubernetes ที่:

1. ห้าม containers:
   - รันเป็น UID 0 (root)
   - ใช้ host network/PID/IPC
   - Mount host paths
   - ใช้ privileged mode

2. บังคับ:
   - Resource limits บน containers ทุกตัว
   - ReadOnlyRootFilesystem
   - Labels ที่จำเป็น
   - Image ต้องมาจาก registry ที่อนุมัติแล้ว

3. Warnings:
   - Ingress ที่ไม่มี TLS
   - Services ที่เป็น LoadBalancer
   - PersistentVolumes ขนาดใหญ่

พร้อม OPA unit tests สำหรับทุก rule
```

### แบบฝึกหัดที่ 2: Terraform Cost Controls

```rego
# สร้าง policy ที่:
# 1. Block expensive EC2 instances (> $0.5/hr)
# 2. Require justification tags สำหรับ large resources
# 3. Limit number of EIP ต่อ account
# 4. ห้าม NAT Gateway มากกว่า 3 ตัว

package terraform.cost

# TODO: Implement cost control policies
```

### แบบฝึกหัดที่ 3: Custom Deployment Gate

```json
// สร้าง deployment gate policy ที่:
// 1. Production deployment ต้องผ่าน:
//    - 2 approvals จาก senior engineers
//    - ไม่อยู่ในช่วง peak hours (17:00-09:00 UTC+7)
//    - Security scan ไม่มี critical issues
//    - Performance tests ผ่าน

// Test ด้วย input:
{
  "environment": "production",
  "requester": "junior_dev",
  "deployment_time": "2024-01-15T10:00:00+07:00",
  "approvals": [
    {"username": "senior1", "role": "senior_engineer"},
    {"username": "junior_dev", "role": "developer"}
  ]
}

// Expected: DENY เพราะ requester approve ตัวเอง
```

### สรุปบทที่ 67

ในบทนี้เราได้เรียนรู้:
- **OPA/Rego**: General-purpose policy engine และภาษา Rego
- **K8s Policies**: Validate manifests สำหรับ security, resources, networking
- **Terraform Policies**: Validate infrastructure changes ก่อน apply
- **Pipeline Gates**: บังคับ approval workflows และ quality checks
- **OPA Gatekeeper**: Admission controller สำหรับ Kubernetes
- **Testing Policies**: Unit tests สำหรับ Rego policies
- **CI Integration**: ใช้ Conftest ใน GitHub Actions pipeline

บทถัดไปเราจะเรียนรู้ Compliance Automation สำหรับ standards เช่น SOC2, PCI-DSS

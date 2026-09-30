# Part 79: Multi-Tenant CI/CD

## บทนำ

Multi-tenant CI/CD คือการออกแบบ pipeline และ infrastructure ที่รองรับหลาย tenants (ลูกค้า, ทีม, หรือ business units) ในระบบเดียวกัน โดยมี isolation, security และ resource management ที่เหมาะสม

ในบทนี้เราจะเรียนรู้:
- Tenant Isolation ใน Pipelines
- Namespace per Tenant
- Resource Quotas
- Tenant-specific Configurations
- Billing per Tenant
- แบบฝึกหัดปฏิบัติ

---

## 79.1 แนวคิด Multi-Tenant CI/CD

### ระดับของ Isolation

```
┌─────────────────────────────────────────────────────────────┐
│                   Isolation Levels                           │
│                                                             │
│  Shared Everything (Least Isolation)                        │
│  ├── Same cluster                                           │
│  ├── Same namespace                                         │
│  └── Shared resources                                       │
│                                                             │
│  Namespace Isolation                                        │
│  ├── Same cluster                                           │
│  ├── Separate namespaces                                    │
│  └── Network policies                                       │
│                                                             │
│  Cluster Isolation                                          │
│  ├── Separate clusters                                      │
│  ├── Shared control plane (Virtual Cluster)                 │
│  └── Dedicated data plane                                   │
│                                                             │
│  Full Isolation (Most Isolation)                            │
│  ├── Separate AWS accounts                                  │
│  ├── Separate VPCs                                          │
│  └── Dedicated everything                                   │
└─────────────────────────────────────────────────────────────┘
```

### เมื่อไหร่ควรใช้แบบไหน?

```yaml
isolation_decision_matrix:
  shared_everything:
    when:
      - All tenants are internal teams
      - Low security requirements
      - Cost optimization priority
    examples:
      - Internal developer teams
      - Different squads same company

  namespace_isolation:
    when:
      - Different teams, same organization
      - Moderate security requirements
      - Need resource limits per team
    examples:
      - Enterprise software with departments
      - SaaS product with team-based plans

  cluster_isolation:
    when:
      - External customers
      - Compliance requirements (HIPAA, PCI-DSS)
      - High security requirements
    examples:
      - Financial services SaaS
      - Healthcare applications

  full_isolation:
    when:
      - Highly regulated industries
      - Enterprise contracts requiring dedicated infrastructure
      - Highest security requirements
    examples:
      - Government contracts
      - Large enterprise customers
```

---

## 79.2 Tenant Isolation ใน Pipelines

### GitHub Actions: Tenant-scoped Pipelines

```yaml
# .github/workflows/multi-tenant-deploy.yaml
name: Multi-Tenant Deploy

on:
  push:
    branches:
      - 'tenant/*/main'  # Pattern: tenant/{tenant-id}/main

jobs:
  extract-tenant:
    runs-on: ubuntu-latest
    outputs:
      tenant-id: ${{ steps.extract.outputs.tenant_id }}
      environment: ${{ steps.extract.outputs.environment }}
    steps:
      - name: Extract tenant from branch
        id: extract
        run: |
          BRANCH="${{ github.ref_name }}"
          # Branch format: tenant/{tenant-id}/main
          TENANT_ID=$(echo "$BRANCH" | cut -d'/' -f2)
          echo "tenant_id=$TENANT_ID" >> $GITHUB_OUTPUT
          echo "environment=production" >> $GITHUB_OUTPUT
          echo "Deploying for tenant: $TENANT_ID"

  validate-tenant:
    runs-on: ubuntu-latest
    needs: extract-tenant
    steps:
      - name: Validate tenant exists
        run: |
          TENANT_ID="${{ needs.extract-tenant.outputs.tenant-id }}"

          # ตรวจสอบว่า tenant มีอยู่ใน registry
          TENANT_EXISTS=$(curl -s "$TENANT_REGISTRY_URL/tenants/$TENANT_ID" \
            -H "Authorization: Bearer $TENANT_REGISTRY_TOKEN" \
            | jq -r '.exists')

          if [ "$TENANT_EXISTS" != "true" ]; then
            echo "❌ Tenant $TENANT_ID not found"
            exit 1
          fi

          echo "✅ Tenant $TENANT_ID validated"

      - name: Check tenant permissions
        run: |
          # ตรวจสอบว่า actor มีสิทธิ์ deploy สำหรับ tenant นี้
          TENANT_ID="${{ needs.extract-tenant.outputs.tenant-id }}"

          AUTHORIZED=$(curl -s "$TENANT_REGISTRY_URL/tenants/$TENANT_ID/members/${{ github.actor }}" \
            -H "Authorization: Bearer $TENANT_REGISTRY_TOKEN" \
            | jq -r '.authorized')

          if [ "$AUTHORIZED" != "true" ]; then
            echo "❌ ${{ github.actor }} is not authorized for tenant $TENANT_ID"
            exit 1
          fi

  deploy:
    runs-on: ubuntu-latest
    needs:
      - extract-tenant
      - validate-tenant
    environment: ${{ needs.extract-tenant.outputs.tenant-id }}-production
    steps:
      - uses: actions/checkout@v4

      - name: Get tenant config
        id: tenant-config
        run: |
          TENANT_ID="${{ needs.extract-tenant.outputs.tenant-id }}"

          # ดึง config สำหรับ tenant นี้
          TENANT_CONFIG=$(curl -s "$TENANT_REGISTRY_URL/tenants/$TENANT_ID/config" \
            -H "Authorization: Bearer $TENANT_REGISTRY_TOKEN")

          NAMESPACE=$(echo "$TENANT_CONFIG" | jq -r '.namespace')
          K8S_CLUSTER=$(echo "$TENANT_CONFIG" | jq -r '.k8s_cluster')
          RESOURCE_TIER=$(echo "$TENANT_CONFIG" | jq -r '.resource_tier')

          echo "namespace=$NAMESPACE" >> $GITHUB_OUTPUT
          echo "cluster=$K8S_CLUSTER" >> $GITHUB_OUTPUT
          echo "resource_tier=$RESOURCE_TIER" >> $GITHUB_OUTPUT

      - name: Deploy to tenant namespace
        run: |
          NAMESPACE="${{ steps.tenant-config.outputs.namespace }}"
          RESOURCE_TIER="${{ steps.tenant-config.outputs.resource_tier }}"

          # Deploy ด้วย tenant-specific values
          helm upgrade --install \
            "${{ needs.extract-tenant.outputs.tenant-id }}-app" \
            ./helm/app \
            --namespace "$NAMESPACE" \
            --create-namespace \
            --set "tenant.id=${{ needs.extract-tenant.outputs.tenant-id }}" \
            --set "tenant.namespace=$NAMESPACE" \
            --set "resources.tier=$RESOURCE_TIER" \
            --values "./tenants/${{ needs.extract-tenant.outputs.tenant-id }}/values.yaml" \
            --wait \
            --timeout 10m
```

### Jenkins Multi-tenant Pipeline

```groovy
// Jenkinsfile - Multi-tenant
pipeline {
    agent none

    parameters {
        string(name: 'TENANT_ID', defaultValue: '', description: 'Tenant ID to deploy')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Target environment')
    }

    stages {
        stage('Validate Tenant') {
            agent { label 'build' }
            steps {
                script {
                    def tenantConfig = loadTenantConfig(params.TENANT_ID)
                    if (!tenantConfig) {
                        error("Tenant '${params.TENANT_ID}' not found")
                    }

                    // Check authorization
                    if (!isAuthorized(env.BUILD_USER_ID, params.TENANT_ID)) {
                        error("User ${env.BUILD_USER_ID} is not authorized for tenant ${params.TENANT_ID}")
                    }

                    env.TENANT_NAMESPACE = tenantConfig.namespace
                    env.TENANT_CLUSTER = tenantConfig.cluster
                    env.RESOURCE_TIER = tenantConfig.resourceTier
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    // ใช้ isolated build environment ต่อ tenant
                    image 'build-agent:latest'
                    label 'build'
                    args "-v /tenant-${params.TENANT_ID}:/workspace"
                }
            }
            steps {
                sh 'make build'
            }
        }

        stage('Deploy') {
            agent { label 'deploy' }
            // ใช้ credentials เฉพาะของ tenant
            environment {
                KUBECONFIG = credentials("kubeconfig-${params.TENANT_ID}-${params.ENVIRONMENT}")
            }
            steps {
                sh """
                helm upgrade --install \\
                  "${params.TENANT_ID}-app" \\
                  ./helm/app \\
                  --namespace "${env.TENANT_NAMESPACE}" \\
                  --set tenant.id=${params.TENANT_ID} \\
                  --values "tenants/${params.TENANT_ID}/values.yaml"
                """
            }
        }
    }
}

def loadTenantConfig(tenantId) {
    def response = httpRequest(
        url: "${TENANT_REGISTRY_URL}/tenants/${tenantId}",
        authentication: 'tenant-registry-token',
    )
    return readJSON(text: response.content)
}

def isAuthorized(userId, tenantId) {
    def response = httpRequest(
        url: "${TENANT_REGISTRY_URL}/tenants/${tenantId}/members/${userId}",
        authentication: 'tenant-registry-token',
    )
    return readJSON(text: response.content).authorized
}
```

---

## 79.3 Namespace per Tenant

### Namespace Provisioning

```python
# namespace-provisioner.py
import yaml
from kubernetes import client, config
from typing import Optional

class TenantNamespaceProvisioner:
    def __init__(self, kubeconfig_path: Optional[str] = None):
        if kubeconfig_path:
            config.load_kube_config(config_file=kubeconfig_path)
        else:
            config.load_incluster_config()

        self.core_v1 = client.CoreV1Api()
        self.rbac_v1 = client.RbacAuthorizationV1Api()
        self.networking_v1 = client.NetworkingV1Api()

    def provision_tenant(
        self,
        tenant_id: str,
        team_email: str,
        resource_tier: str,
        admin_users: list,
    ) -> None:
        """สร้าง namespace และ resources สำหรับ tenant ใหม่"""

        namespace = f"tenant-{tenant_id}"

        print(f"Provisioning tenant: {tenant_id} (namespace: {namespace})")

        # 1. สร้าง Namespace
        self._create_namespace(namespace, tenant_id, team_email)

        # 2. ตั้งค่า Resource Quotas
        self._create_resource_quota(namespace, resource_tier)

        # 3. ตั้งค่า LimitRange
        self._create_limit_range(namespace, resource_tier)

        # 4. สร้าง Network Policy
        self._create_network_policy(namespace, tenant_id)

        # 5. สร้าง RBAC
        self._create_rbac(namespace, tenant_id, admin_users)

        # 6. สร้าง ServiceAccount สำหรับ CI/CD
        self._create_cicd_service_account(namespace, tenant_id)

        print(f"✅ Tenant {tenant_id} provisioned successfully")

    def _create_namespace(self, namespace: str, tenant_id: str, team_email: str) -> None:
        ns = client.V1Namespace(
            metadata=client.V1ObjectMeta(
                name=namespace,
                labels={
                    "tenant": tenant_id,
                    "managed-by": "tenant-provisioner",
                },
                annotations={
                    "team-email": team_email,
                    "provisioned-at": "2025-01-01T00:00:00Z",
                },
            )
        )

        try:
            self.core_v1.create_namespace(ns)
            print(f"  Created namespace: {namespace}")
        except client.ApiException as e:
            if e.status == 409:
                print(f"  Namespace {namespace} already exists")
            else:
                raise

    def _create_resource_quota(self, namespace: str, tier: str) -> None:
        """สร้าง ResourceQuota ตาม tier"""

        tiers = {
            "starter": {
                "requests.cpu": "2",
                "requests.memory": "4Gi",
                "limits.cpu": "4",
                "limits.memory": "8Gi",
                "pods": "20",
                "services": "10",
                "persistentvolumeclaims": "5",
            },
            "professional": {
                "requests.cpu": "8",
                "requests.memory": "16Gi",
                "limits.cpu": "16",
                "limits.memory": "32Gi",
                "pods": "50",
                "services": "20",
                "persistentvolumeclaims": "20",
            },
            "enterprise": {
                "requests.cpu": "32",
                "requests.memory": "64Gi",
                "limits.cpu": "64",
                "limits.memory": "128Gi",
                "pods": "200",
                "services": "50",
                "persistentvolumeclaims": "100",
            },
        }

        quota_spec = tiers.get(tier, tiers["starter"])

        quota = client.V1ResourceQuota(
            metadata=client.V1ObjectMeta(
                name="tenant-quota",
                namespace=namespace,
            ),
            spec=client.V1ResourceQuotaSpec(
                hard=quota_spec,
            ),
        )

        self.core_v1.create_namespaced_resource_quota(namespace, quota)
        print(f"  Created ResourceQuota (tier: {tier})")

    def _create_network_policy(self, namespace: str, tenant_id: str) -> None:
        """สร้าง NetworkPolicy เพื่อ isolate tenant"""

        policy = client.V1NetworkPolicy(
            metadata=client.V1ObjectMeta(
                name="tenant-isolation",
                namespace=namespace,
            ),
            spec=client.V1NetworkPolicySpec(
                pod_selector=client.V1LabelSelector(),  # select all pods
                policy_types=["Ingress", "Egress"],
                ingress=[
                    # Allow traffic ภายใน namespace เดียวกัน
                    client.V1NetworkPolicyIngressRule(
                        _from=[
                            client.V1NetworkPolicyPeer(
                                namespace_selector=client.V1LabelSelector(
                                    match_labels={"tenant": tenant_id}
                                )
                            )
                        ]
                    ),
                    # Allow traffic จาก ingress controller
                    client.V1NetworkPolicyIngressRule(
                        _from=[
                            client.V1NetworkPolicyPeer(
                                namespace_selector=client.V1LabelSelector(
                                    match_labels={"kubernetes.io/metadata.name": "ingress-nginx"}
                                )
                            )
                        ]
                    ),
                ],
                egress=[
                    # Allow DNS
                    client.V1NetworkPolicyEgressRule(
                        ports=[
                            client.V1NetworkPolicyPort(port=53, protocol="UDP"),
                            client.V1NetworkPolicyPort(port=53, protocol="TCP"),
                        ]
                    ),
                    # Allow traffic ภายใน namespace
                    client.V1NetworkPolicyEgressRule(
                        _to=[
                            client.V1NetworkPolicyPeer(
                                namespace_selector=client.V1LabelSelector(
                                    match_labels={"tenant": tenant_id}
                                )
                            )
                        ]
                    ),
                    # Allow เข้าถึง shared services
                    client.V1NetworkPolicyEgressRule(
                        _to=[
                            client.V1NetworkPolicyPeer(
                                namespace_selector=client.V1LabelSelector(
                                    match_labels={"kubernetes.io/metadata.name": "shared-services"}
                                )
                            )
                        ]
                    ),
                ],
            ),
        )

        self.networking_v1.create_namespaced_network_policy(namespace, policy)
        print(f"  Created NetworkPolicy")

    def _create_rbac(
        self,
        namespace: str,
        tenant_id: str,
        admin_users: list,
    ) -> None:
        """สร้าง RBAC roles และ bindings"""

        # สร้าง Role
        role = client.V1Role(
            metadata=client.V1ObjectMeta(
                name="tenant-admin",
                namespace=namespace,
            ),
            rules=[
                client.V1PolicyRule(
                    api_groups=["", "apps", "networking.k8s.io"],
                    resources=["pods", "deployments", "services", "ingresses", "configmaps", "secrets"],
                    verbs=["get", "list", "watch", "create", "update", "patch", "delete"],
                ),
                # ไม่อนุญาตให้แก้ไข ResourceQuota
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["resourcequotas"],
                    verbs=["get", "list", "watch"],  # read-only
                ),
            ],
        )

        self.rbac_v1.create_namespaced_role(namespace, role)

        # สร้าง RoleBinding สำหรับ admin users
        for user in admin_users:
            binding = client.V1RoleBinding(
                metadata=client.V1ObjectMeta(
                    name=f"tenant-admin-{user.replace('@', '-').replace('.', '-')}",
                    namespace=namespace,
                ),
                role_ref=client.V1RoleRef(
                    api_group="rbac.authorization.k8s.io",
                    kind="Role",
                    name="tenant-admin",
                ),
                subjects=[
                    client.V1Subject(
                        kind="User",
                        name=user,
                        api_group="rbac.authorization.k8s.io",
                    )
                ],
            )
            self.rbac_v1.create_namespaced_role_binding(namespace, binding)

        print(f"  Created RBAC for {len(admin_users)} admins")

    def _create_cicd_service_account(self, namespace: str, tenant_id: str) -> None:
        """สร้าง ServiceAccount สำหรับ CI/CD pipeline"""

        sa = client.V1ServiceAccount(
            metadata=client.V1ObjectMeta(
                name="cicd-deployer",
                namespace=namespace,
                annotations={
                    "eks.amazonaws.com/role-arn": f"arn:aws:iam::123456789:role/cicd-{tenant_id}",
                },
            ),
        )

        self.core_v1.create_namespaced_service_account(namespace, sa)

        # Role สำหรับ CI/CD
        role = client.V1Role(
            metadata=client.V1ObjectMeta(
                name="cicd-deployer",
                namespace=namespace,
            ),
            rules=[
                client.V1PolicyRule(
                    api_groups=["apps"],
                    resources=["deployments"],
                    verbs=["get", "list", "update", "patch"],
                ),
                client.V1PolicyRule(
                    api_groups=[""],
                    resources=["pods", "pods/log"],
                    verbs=["get", "list", "watch"],
                ),
            ],
        )

        binding = client.V1RoleBinding(
            metadata=client.V1ObjectMeta(name="cicd-deployer", namespace=namespace),
            role_ref=client.V1RoleRef(
                api_group="rbac.authorization.k8s.io",
                kind="Role",
                name="cicd-deployer",
            ),
            subjects=[
                client.V1Subject(
                    kind="ServiceAccount",
                    name="cicd-deployer",
                    namespace=namespace,
                )
            ],
        )

        self.rbac_v1.create_namespaced_role(namespace, role)
        self.rbac_v1.create_namespaced_role_binding(namespace, binding)

        print(f"  Created CI/CD ServiceAccount")
```

---

## 79.4 Tenant-specific Configurations

### Helm Values per Tenant

```yaml
# tenants/acme-corp/values.yaml
tenant:
  id: acme-corp
  name: ACME Corporation
  plan: enterprise

# Override global defaults
replicaCount: 5

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2"
    memory: "2Gi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

# Tenant-specific feature flags
features:
  advancedReporting: true
  customBranding: true
  ssoIntegration: true
  apiRateLimit: 10000  # requests per hour

# Custom domain
ingress:
  hosts:
    - host: app.acme-corp.myproduct.com
  tls:
    - secretName: acme-corp-tls
      hosts:
        - app.acme-corp.myproduct.com

# Database configuration
database:
  host: acme-corp-db.internal
  name: acme_prod
  pool:
    min: 5
    max: 30

# Custom notification settings
notifications:
  slack:
    webhook: "https://hooks.slack.com/services/ACME/CORP"
  email:
    from: noreply@acme-corp.myproduct.com
```

### Tenant Configuration Manager

```python
# config-manager/tenant_config.py
import yaml
import json
from pathlib import Path
from typing import Any, Optional
from dataclasses import dataclass, field

@dataclass
class TenantConfig:
    tenant_id: str
    plan: str
    namespace: str
    k8s_cluster: str
    resource_tier: str
    features: dict = field(default_factory=dict)
    custom_domain: Optional[str] = None
    admin_users: list = field(default_factory=list)
    integrations: dict = field(default_factory=dict)


class TenantConfigManager:
    def __init__(self, config_repo_path: str):
        self.config_repo = Path(config_repo_path)

    def get_tenant_config(self, tenant_id: str) -> TenantConfig:
        """โหลด config สำหรับ tenant"""
        config_file = self.config_repo / "tenants" / tenant_id / "config.yaml"

        if not config_file.exists():
            raise FileNotFoundError(f"Config not found for tenant: {tenant_id}")

        with open(config_file) as f:
            raw_config = yaml.safe_load(f)

        return TenantConfig(
            tenant_id=tenant_id,
            plan=raw_config.get("plan", "starter"),
            namespace=raw_config.get("namespace", f"tenant-{tenant_id}"),
            k8s_cluster=raw_config.get("k8s_cluster", "default"),
            resource_tier=raw_config.get("resource_tier", "starter"),
            features=raw_config.get("features", {}),
            custom_domain=raw_config.get("custom_domain"),
            admin_users=raw_config.get("admin_users", []),
            integrations=raw_config.get("integrations", {}),
        )

    def get_merged_values(self, tenant_id: str) -> dict:
        """Merge default values กับ tenant-specific values"""
        # โหลด default values
        default_file = self.config_repo / "defaults" / "values.yaml"
        with open(default_file) as f:
            defaults = yaml.safe_load(f)

        # โหลด tenant values
        tenant_file = self.config_repo / "tenants" / tenant_id / "values.yaml"
        if tenant_file.exists():
            with open(tenant_file) as f:
                tenant_values = yaml.safe_load(f)
        else:
            tenant_values = {}

        # Deep merge
        return self._deep_merge(defaults, tenant_values)

    def _deep_merge(self, base: dict, override: dict) -> dict:
        """Deep merge two dicts"""
        result = base.copy()
        for key, value in override.items():
            if key in result and isinstance(result[key], dict) and isinstance(value, dict):
                result[key] = self._deep_merge(result[key], value)
            else:
                result[key] = value
        return result

    def validate_tenant_config(self, tenant_id: str) -> list:
        """ตรวจสอบ config ว่าถูกต้องหรือไม่"""
        errors = []

        try:
            config = self.get_tenant_config(tenant_id)
        except FileNotFoundError:
            return [f"Config file not found for tenant: {tenant_id}"]

        # Validate plan
        valid_plans = ["starter", "professional", "enterprise"]
        if config.plan not in valid_plans:
            errors.append(f"Invalid plan: {config.plan}. Must be one of {valid_plans}")

        # Validate namespace format
        import re
        if not re.match(r'^[a-z][a-z0-9-]*$', config.namespace):
            errors.append(f"Invalid namespace format: {config.namespace}")

        # Validate admin users (must be emails)
        for user in config.admin_users:
            if '@' not in user:
                errors.append(f"Invalid admin user format: {user}")

        return errors
```

---

## 79.5 Resource Quotas และ Cost Management

### Cost Tracking per Tenant

```python
# cost-tracking/tenant_cost.py
import boto3
from datetime import datetime, timedelta
from typing import Dict, List
from dataclasses import dataclass

@dataclass
class TenantCost:
    tenant_id: str
    period_start: datetime
    period_end: datetime
    compute_cost: float
    storage_cost: float
    network_cost: float
    total_cost: float
    currency: str = "USD"


class TenantCostTracker:
    def __init__(self, aws_region: str = "ap-southeast-1"):
        self.ce_client = boto3.client('ce', region_name=aws_region)
        self.cw_client = boto3.client('cloudwatch', region_name=aws_region)

    def get_tenant_cost(
        self,
        tenant_id: str,
        start_date: datetime,
        end_date: datetime,
    ) -> TenantCost:
        """ดู cost สำหรับ tenant ใน AWS โดยใช้ tags"""

        response = self.ce_client.get_cost_and_usage(
            TimePeriod={
                'Start': start_date.strftime('%Y-%m-%d'),
                'End': end_date.strftime('%Y-%m-%d'),
            },
            Granularity='MONTHLY',
            Filter={
                'Tags': {
                    'Key': 'tenant',
                    'Values': [tenant_id],
                }
            },
            GroupBy=[
                {'Type': 'DIMENSION', 'Key': 'SERVICE'},
            ],
            Metrics=['BlendedCost'],
        )

        costs = {
            'compute': 0.0,
            'storage': 0.0,
            'network': 0.0,
            'other': 0.0,
        }

        for result in response.get('ResultsByTime', []):
            for group in result.get('Groups', []):
                service = group['Keys'][0]
                amount = float(group['Metrics']['BlendedCost']['Amount'])

                if 'EC2' in service or 'EKS' in service or 'Fargate' in service:
                    costs['compute'] += amount
                elif 'S3' in service or 'RDS' in service or 'EBS' in service:
                    costs['storage'] += amount
                elif 'Transfer' in service or 'CloudFront' in service:
                    costs['network'] += amount
                else:
                    costs['other'] += amount

        total = sum(costs.values())

        return TenantCost(
            tenant_id=tenant_id,
            period_start=start_date,
            period_end=end_date,
            compute_cost=round(costs['compute'], 2),
            storage_cost=round(costs['storage'], 2),
            network_cost=round(costs['network'], 2),
            total_cost=round(total, 2),
        )

    def get_kubernetes_resource_usage(self, namespace: str) -> dict:
        """ดู resource usage จาก Kubernetes metrics"""
        from kubernetes import client, config
        config.load_incluster_config()

        custom_api = client.CustomObjectsApi()

        # ดึง metrics จาก metrics-server
        try:
            pod_metrics = custom_api.list_namespaced_custom_object(
                group="metrics.k8s.io",
                version="v1beta1",
                namespace=namespace,
                plural="pods",
            )

            total_cpu = 0
            total_memory = 0

            for pod in pod_metrics.get('items', []):
                for container in pod.get('containers', []):
                    cpu = container['usage']['cpu']
                    memory = container['usage']['memory']

                    # แปลง units
                    if cpu.endswith('n'):
                        total_cpu += int(cpu[:-1]) / 1e9
                    elif cpu.endswith('m'):
                        total_cpu += int(cpu[:-1]) / 1000

                    if memory.endswith('Ki'):
                        total_memory += int(memory[:-2]) * 1024
                    elif memory.endswith('Mi'):
                        total_memory += int(memory[:-2]) * 1024 * 1024

            return {
                "cpu_cores": round(total_cpu, 3),
                "memory_bytes": total_memory,
                "memory_gb": round(total_memory / (1024**3), 2),
            }
        except Exception as e:
            return {"error": str(e)}


class TenantBillingService:
    """คำนวณค่าใช้จ่ายและสร้าง invoice สำหรับ tenant"""

    PRICING = {
        "starter": {
            "base_price": 99.0,  # USD/month
            "cpu_per_core_hour": 0.05,
            "memory_per_gb_hour": 0.01,
            "storage_per_gb_month": 0.10,
        },
        "professional": {
            "base_price": 499.0,
            "cpu_per_core_hour": 0.04,
            "memory_per_gb_hour": 0.008,
            "storage_per_gb_month": 0.08,
        },
        "enterprise": {
            "base_price": 1999.0,
            "cpu_per_core_hour": 0.03,
            "memory_per_gb_hour": 0.006,
            "storage_per_gb_month": 0.06,
        },
    }

    def calculate_invoice(
        self,
        tenant_id: str,
        plan: str,
        month: datetime,
        usage_metrics: dict,
    ) -> dict:
        """คำนวณ invoice สำหรับเดือนที่ผ่านมา"""

        pricing = self.PRICING.get(plan, self.PRICING["starter"])

        # Base price
        base_price = pricing["base_price"]

        # Compute overage (เกิน included quota)
        included_cpu_hours = {"starter": 200, "professional": 800, "enterprise": 3000}[plan]
        included_memory_gb_hours = {"starter": 400, "professional": 1600, "enterprise": 6000}[plan]

        cpu_hours = usage_metrics.get("cpu_core_hours", 0)
        memory_gb_hours = usage_metrics.get("memory_gb_hours", 0)

        cpu_overage = max(0, cpu_hours - included_cpu_hours)
        memory_overage = max(0, memory_gb_hours - included_memory_gb_hours)

        overage_cost = (
            cpu_overage * pricing["cpu_per_core_hour"] +
            memory_overage * pricing["memory_per_gb_hour"]
        )

        total = base_price + overage_cost

        return {
            "tenant_id": tenant_id,
            "plan": plan,
            "period": month.strftime("%Y-%m"),
            "line_items": [
                {
                    "description": f"Base plan ({plan})",
                    "amount": base_price,
                },
                {
                    "description": f"Compute overage ({cpu_overage:.1f} CPU core hours)",
                    "amount": round(cpu_overage * pricing["cpu_per_core_hour"], 2),
                } if cpu_overage > 0 else None,
                {
                    "description": f"Memory overage ({memory_overage:.1f} GB hours)",
                    "amount": round(memory_overage * pricing["memory_per_gb_hour"], 2),
                } if memory_overage > 0 else None,
            ],
            "subtotal": round(total, 2),
            "tax": round(total * 0.07, 2),  # 7% VAT
            "total": round(total * 1.07, 2),
            "currency": "USD",
        }
```

---

## 79.6 Tenant Onboarding Automation

### Automated Tenant Setup

```bash
#!/bin/bash
# scripts/provision-tenant.sh

set -euo pipefail

TENANT_ID="$1"
PLAN="${2:-starter}"
ADMIN_EMAIL="$3"

echo "🚀 Provisioning tenant: $TENANT_ID (plan: $PLAN)"

# 1. Create tenant config directory
mkdir -p "tenants/$TENANT_ID"

# 2. Generate config files
cat > "tenants/$TENANT_ID/config.yaml" << EOF
tenant_id: $TENANT_ID
plan: $PLAN
namespace: tenant-$TENANT_ID
k8s_cluster: production
resource_tier: $PLAN
admin_users:
  - $ADMIN_EMAIL
created_at: $(date -u +%Y-%m-%dT%H:%M:%SZ)
EOF

# 3. Copy template values
cp "templates/values-$PLAN.yaml" "tenants/$TENANT_ID/values.yaml"

# Customize values for tenant
sed -i "s/TENANT_ID_PLACEHOLDER/$TENANT_ID/g" "tenants/$TENANT_ID/values.yaml"

# 4. Provision Kubernetes namespace
python3 namespace-provisioner.py \
  --tenant-id "$TENANT_ID" \
  --plan "$PLAN" \
  --admin-email "$ADMIN_EMAIL"

# 5. Create ArgoCD Application
cat > "argocd/applications/tenant-$TENANT_ID.yaml" << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tenant-$TENANT_ID
  namespace: argocd
  labels:
    tenant: $TENANT_ID
spec:
  project: tenants
  source:
    repoURL: https://github.com/mycompany/tenant-configs
    targetRevision: HEAD
    path: tenants/$TENANT_ID
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: tenant-$TENANT_ID
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

kubectl apply -f "argocd/applications/tenant-$TENANT_ID.yaml"

# 6. Send welcome email
python3 send-welcome-email.py \
  --tenant-id "$TENANT_ID" \
  --admin-email "$ADMIN_EMAIL" \
  --plan "$PLAN"

echo "✅ Tenant $TENANT_ID provisioned successfully"
echo "Namespace: tenant-$TENANT_ID"
echo "Admin: $ADMIN_EMAIL"
```

---

## 79.7 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Namespace Provisioning

```python
# exercises/provision-namespace.py
# สร้าง namespace สำหรับ 3 tenants ด้วย different plans

tenants_to_provision = [
    {
        "tenant_id": "startup-alpha",
        "plan": "starter",
        "admin_email": "admin@startup-alpha.com",
        "team_email": "team@startup-alpha.com",
    },
    {
        "tenant_id": "medium-corp",
        "plan": "professional",
        "admin_email": "admin@medium-corp.com",
        "team_email": "devops@medium-corp.com",
    },
    {
        "tenant_id": "enterprise-inc",
        "plan": "enterprise",
        "admin_email": "admin@enterprise-inc.com",
        "team_email": "platform@enterprise-inc.com",
    },
]

# TODO: สร้าง namespace และ resources สำหรับแต่ละ tenant
# ใช้ TenantNamespaceProvisioner จากตัวอย่างด้านบน

provisioner = TenantNamespaceProvisioner()

for tenant in tenants_to_provision:
    provisioner.provision_tenant(
        tenant_id=tenant["tenant_id"],
        team_email=tenant["team_email"],
        resource_tier=tenant["plan"],
        admin_users=[tenant["admin_email"]],
    )

# ตรวจสอบว่า provision สำเร็จ
# kubectl get namespace | grep tenant-
# kubectl describe namespace tenant-startup-alpha
# kubectl describe resourcequota -n tenant-startup-alpha
# kubectl describe networkpolicy -n tenant-startup-alpha
```

### แบบฝึกหัดที่ 2: Tenant Isolation Test

```bash
# exercises/test-tenant-isolation.sh
# ทดสอบว่า tenants ไม่สามารถเข้าถึง resources ของกันได้

# 1. สร้าง test pod ใน tenant-a
kubectl run test-pod-a \
  --image=curlimages/curl:latest \
  --namespace=tenant-startup-alpha \
  --command -- sleep infinity

# 2. ลอง access service ใน tenant-b จาก tenant-a
kubectl exec -n tenant-startup-alpha test-pod-a -- \
  curl http://my-service.tenant-medium-corp.svc.cluster.local

# ต้องได้รับ error (connection refused/timeout)
# ถ้า NetworkPolicy ทำงานถูกต้อง

# 3. ทดสอบว่า tenant-a สามารถ access shared services ได้
kubectl exec -n tenant-startup-alpha test-pod-a -- \
  curl http://shared-api.shared-services.svc.cluster.local/health

# ต้อง success

# Cleanup
kubectl delete pod test-pod-a -n tenant-startup-alpha
```

### แบบฝึกหัดที่ 3: Cost Dashboard

```python
# exercises/cost-dashboard.py
# สร้าง cost report สำหรับแต่ละ tenant

from datetime import datetime

tracker = TenantCostTracker()
billing = TenantBillingService()

# ดึง costs สำหรับเดือนที่ผ่านมา
last_month_start = datetime(2025, 8, 1)
last_month_end = datetime(2025, 9, 1)

tenants = ["startup-alpha", "medium-corp", "enterprise-inc"]

print("=" * 70)
print(f"TENANT COST REPORT - {last_month_start.strftime('%B %Y')}")
print("=" * 70)

total_revenue = 0

for tenant_id in tenants:
    # TODO: ดึงข้อมูลจริงจาก AWS Cost Explorer
    # cost = tracker.get_tenant_cost(tenant_id, last_month_start, last_month_end)

    # ใช้ mock data สำหรับฝึก
    plan = {"startup-alpha": "starter", "medium-corp": "professional", "enterprise-inc": "enterprise"}[tenant_id]

    invoice = billing.calculate_invoice(
        tenant_id=tenant_id,
        plan=plan,
        month=last_month_start,
        usage_metrics={
            "cpu_core_hours": 150 if plan == "starter" else 500 if plan == "professional" else 2000,
            "memory_gb_hours": 300 if plan == "starter" else 1000 if plan == "professional" else 4000,
        },
    )

    print(f"\nTenant: {tenant_id} ({plan})")
    for item in invoice["line_items"]:
        if item:
            print(f"  {item['description']}: ${item['amount']:.2f}")
    print(f"  Total: ${invoice['total']:.2f}")

    total_revenue += invoice["total"]

print("\n" + "=" * 70)
print(f"Total Monthly Revenue: ${total_revenue:.2f}")
print("=" * 70)
```

---

## สรุป

Multi-Tenant CI/CD ต้องการการวางแผนที่รอบคอบในหลายมิติ:

1. **Isolation** ต้องเลือก level ที่เหมาะสมกับ security requirements
2. **Namespace-per-Tenant** เป็น balance ที่ดีระหว่าง isolation และ cost
3. **Resource Quotas** ป้องกัน noisy neighbor problem
4. **Tenant Config** ต้องจัดการอย่างเป็นระบบและ version-controlled
5. **Cost Visibility** ทำให้ทั้ง tenant และ platform team รู้ว่าใช้ทรัพยากรเท่าไหร่

การ scale multi-tenant system ต้องการ automation ในทุกขั้นตอน ตั้งแต่ onboarding จนถึง offboarding

### ขั้นตอนถัดไป

- ศึกษา [Part 80: CI/CD for Legacy Systems](./part-80-legacy-systems.md)
- Kubernetes multi-tenancy: https://kubernetes.io/docs/concepts/security/multi-tenancy/
- vCluster (virtual clusters): https://www.vcluster.com

# Part 54: Zero-Trust Security ใน CI/CD

## บทนำ

Zero-Trust Security คือแนวคิดความปลอดภัยที่ "ไม่เชื่อใครทั้งนั้น ตรวจสอบทุกอย่างเสมอ" (Never Trust, Always Verify) ในโลก CI/CD แบบดั้งเดิม เรามักสมมติว่า networks ภายในองค์กรปลอดภัย แต่ Zero-Trust บอกว่าไม่ควรเชื่อเช่นนั้น

บทนี้จะครอบคลุม:
- Zero-Trust Principles
- SPIFFE/SPIRE สำหรับ Workload Identity
- OIDC ใน CI/CD
- mTLS สำหรับ Service-to-Service Communication
- Network Policies
- Workshop และ Exercises

---

## 1. Zero-Trust Principles

### 1.1 หลักการ Zero-Trust

```
Traditional Security Model:
┌─────────────────────────────────────────┐
│  Trusted Network (Inside Firewall)      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Service │──│ Service │──│ Service │ │
│  └─────────┘  └─────────┘  └─────────┘ │
│            Implicit Trust               │
└─────────────────────────────────────────┘

Zero-Trust Security Model:
┌─────────────────────────────────────────┐
│  No Implicit Trust (Everywhere)         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │ Service │──│ Service │──│ Service │ │
│  └────┬────┘  └────┬────┘  └────┬────┘ │
│       │            │            │       │
│  Authenticate + Authorize Every Request │
└─────────────────────────────────────────┘
```

### 1.2 Zero-Trust ใน CI/CD

Zero-Trust ใน CI/CD Pipeline หมายถึง:

1. **ทุก Job ต้องมี Identity** — ไม่ใช่แค่ "runner" แต่ต้องระบุได้ว่าเป็น Job อะไร
2. **Short-lived Credentials** — ไม่มี long-lived secrets
3. **Least Privilege Access** — แต่ละ Job เข้าถึงเฉพาะสิ่งที่จำเป็น
4. **Continuous Verification** — ตรวจสอบทุก request ไม่ใช่แค่ครั้งแรก
5. **Audit Everything** — บันทึกทุกการกระทำ

---

## 2. SPIFFE/SPIRE — Workload Identity

### 2.1 SPIFFE คืออะไร?

**SPIFFE** (Secure Production Identity Framework For Everyone) คือ standard สำหรับ workload identity ประกอบด้วย:
- **SPIFFE ID** — URI format: `spiffe://trust-domain/workload-identifier`
- **SVID** (SPIFFE Verifiable Identity Document) — certificate หรือ JWT ที่พิสูจน์ identity

```
SPIFFE ID Examples:
spiffe://example.com/ns/production/sa/frontend
spiffe://example.com/ns/staging/service/payment-api
spiffe://example.com/region/asia/datacenter/dc1/vm/webserver-01
```

### 2.2 SPIRE Server และ Agent

```yaml
# k8s/spire/spire-server.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: spire

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: spire-server
  namespace: spire
data:
  server.conf: |
    server {
      bind_address = "0.0.0.0"
      bind_port = "8081"
      trust_domain = "example.com"
      data_dir = "/run/spire/data"
      log_level = "DEBUG"
      
      # JWT SVIDs
      jwt_issuer = "spire-server"
      
      # มีอายุสั้น เพื่อ security
      default_x509_svid_ttl = "1h"
      default_jwt_svid_ttl = "5m"
    }

    plugins {
      DataStore "sql" {
        plugin_data {
          database_type = "sqlite3"
          connection_string = "/run/spire/data/datastore.sqlite3"
        }
      }

      KeyManager "disk" {
        plugin_data {
          directory = "/run/spire/data"
        }
      }

      NodeAttestor "k8s_psat" {
        plugin_data {
          clusters = {
            "my-cluster" = {
              service_account_allow_list = ["spire:spire-agent"]
            }
          }
        }
      }

      UpstreamAuthority "disk" {
        plugin_data {
          cert_file_path = "/run/spire/secrets/bootstrap.crt"
          key_file_path = "/run/spire/secrets/bootstrap.key"
        }
      }
    }

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: spire-server
  namespace: spire
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spire-server
  template:
    metadata:
      labels:
        app: spire-server
    spec:
      containers:
        - name: spire-server
          image: ghcr.io/spiffe/spire-server:1.9.0
          args:
            - -config
            - /run/spire/config/server.conf
          ports:
            - containerPort: 8081
          volumeMounts:
            - name: spire-config
              mountPath: /run/spire/config
              readOnly: true
            - name: spire-data
              mountPath: /run/spire/data
      volumes:
        - name: spire-config
          configMap:
            name: spire-server
        - name: spire-data
          emptyDir: {}
```

### 2.3 SPIRE Agent

```yaml
# k8s/spire/spire-agent.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spire-agent
  namespace: spire
data:
  agent.conf: |
    agent {
      data_dir = "/run/spire"
      log_level = "DEBUG"
      server_address = "spire-server"
      server_port = "8081"
      socket_path = "/run/spire/sockets/agent.sock"
      trust_bundle_path = "/run/spire/bundle/bundle.crt"
      trust_domain = "example.com"
    }

    plugins {
      NodeAttestor "k8s_psat" {
        plugin_data {
          # cluster name ต้องตรงกับที่ server กำหนด
          cluster = "my-cluster"
        }
      }

      KeyManager "memory" {
        plugin_data {}
      }

      WorkloadAttestor "k8s" {
        plugin_data {
          skip_kubelet_verification = true
        }
      }
    }

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: spire-agent
  namespace: spire
spec:
  selector:
    matchLabels:
      app: spire-agent
  template:
    metadata:
      labels:
        app: spire-agent
    spec:
      hostPID: true
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      
      initContainers:
        - name: init
          image: ghcr.io/spiffe/spire-agent:1.9.0
          args:
            - -config
            - /run/spire/config/agent.conf
          volumeMounts:
            - name: spire-config
              mountPath: /run/spire/config
              readOnly: true
            - name: spire-bundle
              mountPath: /run/spire/bundle
      
      containers:
        - name: spire-agent
          image: ghcr.io/spiffe/spire-agent:1.9.0
          args:
            - -config
            - /run/spire/config/agent.conf
          volumeMounts:
            - name: spire-config
              mountPath: /run/spire/config
              readOnly: true
            - name: spire-bundle
              mountPath: /run/spire/bundle
            - name: spire-agent-socket
              mountPath: /run/spire/sockets
              readOnly: false
      
      volumes:
        - name: spire-config
          configMap:
            name: spire-agent
        - name: spire-bundle
          configMap:
            name: spire-bundle
        - name: spire-agent-socket
          hostPath:
            path: /run/spire/sockets
            type: DirectoryOrCreate
```

### 2.4 Register Workload Identity

```bash
# Register workload entry สำหรับ frontend service
kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.com/ns/production/sa/frontend \
    -parentID spiffe://example.com/spire/agent/k8s_psat/my-cluster/node1 \
    -selector k8s:ns:production \
    -selector k8s:sa:frontend \
    -ttl 3600

# Register workload สำหรับ payment API
kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.com/ns/production/sa/payment-api \
    -parentID spiffe://example.com/spire/agent/k8s_psat/my-cluster/node1 \
    -selector k8s:ns:production \
    -selector k8s:sa:payment-api \
    -ttl 3600

# List entries
kubectl exec -n spire spire-server-0 -- \
    /opt/spire/bin/spire-server entry show
```

---

## 3. OIDC ใน CI/CD

### 3.1 OIDC Authentication Flow

```
GitHub Actions OIDC Flow:

1. GitHub Actions เริ่ม Job
2. Job ขอ OIDC Token จาก GitHub
3. GitHub ออก JWT Token (short-lived, ~5 min)
4. Job ส่ง Token ไปที่ Cloud Provider (AWS/GCP/Azure)
5. Cloud Provider verify token กับ GitHub OIDC endpoint
6. Cloud Provider ออก short-lived credentials
7. Job ใช้ credentials เพื่อ access resources

┌─────────────────────────────────────────────────────────────┐
│  GitHub Actions Runner                                      │
│                                                             │
│  1. Request OIDC Token ──→ GitHub OIDC Provider            │
│     ← JWT Token ──────────────────────────────             │
│                                                             │
│  2. Exchange Token ──→ AWS STS / GCP STS / Azure AD        │
│     ← Temporary Credentials ──────────────────             │
│                                                             │
│  3. Access Resources ──→ AWS ECR / GCS / ACR               │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 GitHub Actions OIDC Token Claims

```json
{
  "jti": "example-id",
  "sub": "repo:myorg/myapp:ref:refs/heads/main",
  "aud": "https://github.com/myorg",
  "ref": "refs/heads/main",
  "sha": "abc123",
  "repository": "myorg/myapp",
  "repository_owner": "myorg",
  "repository_owner_id": "12345",
  "run_id": "987654",
  "run_number": "42",
  "run_attempt": "1",
  "repository_visibility": "private",
  "repository_id": "67890",
  "actor_id": "11111",
  "actor": "developer",
  "workflow": "Deploy",
  "head_ref": "",
  "base_ref": "",
  "event_name": "push",
  "ref_protected": "true",
  "ref_type": "branch",
  "workflow_ref": "myorg/myapp/.github/workflows/deploy.yml@refs/heads/main",
  "workflow_sha": "def456",
  "job_workflow_ref": "myorg/myapp/.github/workflows/deploy.yml@refs/heads/main",
  "runner_environment": "github-hosted",
  "iss": "https://token.actions.githubusercontent.com",
  "nbf": 1709000000,
  "exp": 1709000300,
  "iat": 1709000000
}
```

### 3.3 OIDC กับ HashiCorp Vault

```hcl
# vault/config/github-jwt-auth.tf
# กำหนด JWT auth method ใน Vault สำหรับ GitHub Actions

resource "vault_jwt_auth_backend" "github_actions" {
  description = "GitHub Actions OIDC"
  path        = "jwt/github"
  
  # GitHub Actions OIDC endpoint
  oidc_discovery_url = "https://token.actions.githubusercontent.com"
  
  bound_issuer = "https://token.actions.githubusercontent.com"
  
  default_role = "default"
}

resource "vault_jwt_auth_backend_role" "github_actions_deploy" {
  backend   = vault_jwt_auth_backend.github_actions.path
  role_name = "deploy-production"
  role_type = "jwt"
  
  # ตรวจสอบว่า token มาจาก repository ที่ถูกต้อง
  bound_claims = {
    repository = "myorg/myapp"
    ref        = "refs/heads/main"
  }
  
  # ต้องเป็น main branch เท่านั้น
  bound_claims_type = "glob"
  
  # SPIFFE-like identity
  user_claim = "sub"
  
  token_policies = ["deploy-production-policy"]
  
  # Token มีอายุสั้น
  token_ttl     = 300    # 5 minutes
  token_max_ttl = 600    # 10 minutes
}

resource "vault_policy" "deploy_production" {
  name = "deploy-production-policy"
  
  policy = <<EOT
# Allow reading production secrets
path "secret/data/production/*" {
  capabilities = ["read"]
}

# Allow reading database credentials
path "database/creds/production-role" {
  capabilities = ["read"]
}

# Deny everything else
path "*" {
  capabilities = ["deny"]
}
EOT
}
```

### 3.4 GitHub Actions กับ Vault

```yaml
# .github/workflows/vault-oidc.yml
name: Deploy with Vault OIDC

on:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write  # จำเป็นสำหรับ OIDC

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Get OIDC Token จาก GitHub
      - name: Get GitHub OIDC Token
        id: oidc
        run: |
          TOKEN=$(curl -sS \
            -H "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=https://vault.example.com" \
            | jq -r '.value')
          echo "::add-mask::$TOKEN"
          echo "token=$TOKEN" >> $GITHUB_OUTPUT

      # Authenticate กับ Vault
      - name: Authenticate with Vault
        id: vault-auth
        run: |
          VAULT_TOKEN=$(curl -sS \
            -X POST \
            "https://vault.example.com/v1/auth/jwt/github/login" \
            -d "{\"jwt\": \"${{ steps.oidc.outputs.token }}\", \"role\": \"deploy-production\"}" \
            | jq -r '.auth.client_token')
          
          echo "::add-mask::$VAULT_TOKEN"
          echo "VAULT_TOKEN=$VAULT_TOKEN" >> $GITHUB_ENV

      # Get Secrets จาก Vault
      - name: Get Secrets
        run: |
          # Get database credentials
          DB_CREDS=$(curl -sS \
            -H "X-Vault-Token: $VAULT_TOKEN" \
            "https://vault.example.com/v1/database/creds/production-role")
          
          DB_USER=$(echo $DB_CREDS | jq -r '.data.username')
          DB_PASS=$(echo $DB_CREDS | jq -r '.data.password')
          
          echo "::add-mask::$DB_USER"
          echo "::add-mask::$DB_PASS"
          echo "DB_USER=$DB_USER" >> $GITHUB_ENV
          echo "DB_PASS=$DB_PASS" >> $GITHUB_ENV

      - name: Deploy
        run: |
          # ใช้ secrets ใน deployment
          echo "Deploying with dynamic database credentials"
          kubectl set env deployment/myapp \
            DB_USER="$DB_USER" \
            DB_PASS="$DB_PASS"

      # Revoke token เมื่อเสร็จ
      - name: Revoke Vault Token
        if: always()
        run: |
          curl -sS -X POST \
            -H "X-Vault-Token: $VAULT_TOKEN" \
            "https://vault.example.com/v1/auth/token/revoke-self"
```

---

## 4. mTLS สำหรับ Service-to-Service

### 4.1 mTLS คืออะไร?

mTLS (Mutual TLS) คือ protocol ที่ทั้ง client และ server ต้องแสดง certificate เพื่อ authenticate กัน

```
Regular TLS:
Client ──────────────────────→ Server
       Server Certificate Only

mTLS:
Client ←──────────────────── Server
       Server Certificate
Client ──────────────────────→ Server
       Client Certificate
              Both Authenticated
```

### 4.2 mTLS กับ SPIFFE/SPIRE

```go
// go/mtls-server.go
// ตัวอย่าง server ที่ใช้ SPIFFE mTLS
package main

import (
	"context"
	"crypto/tls"
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
	"github.com/spiffe/go-spiffe/v2/spiffetls"
	"github.com/spiffe/go-spiffe/v2/spiffetls/tlsconfig"
	"github.com/spiffe/go-spiffe/v2/workloadapi"
)

func main() {
	ctx := context.Background()

	// สร้าง X.509 source จาก SPIRE agent
	source, err := workloadapi.NewX509Source(ctx)
	if err != nil {
		log.Fatalf("Unable to create X509Source: %v", err)
	}
	defer source.Close()

	// กำหนด SPIFFE IDs ที่อนุญาต
	allowedID := spiffeid.RequireIDFromString("spiffe://example.com/ns/production/sa/frontend")

	// สร้าง TLS config สำหรับ mTLS
	tlsConfig := tlsconfig.MTLSServerConfig(
		source,
		source,
		tlsconfig.AuthorizeID(allowedID),
	)

	server := &http.Server{
		Addr:      ":8443",
		TLSConfig: tlsConfig,
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// ดู SPIFFE ID ของ client
			peerID, err := spiffetls.PeerIDFromContext(r.Context())
			if err != nil {
				http.Error(w, "Unauthorized", http.StatusUnauthorized)
				return
			}

			log.Printf("Request from: %s", peerID)
			fmt.Fprintf(w, "Hello, %s!\n", peerID)
		}),
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	log.Printf("Server listening on :8443 with mTLS")
	if err := server.ListenAndServeTLS("", ""); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

```go
// go/mtls-client.go
// ตัวอย่าง client ที่ใช้ SPIFFE mTLS
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"net/http"
	"time"

	"github.com/spiffe/go-spiffe/v2/spiffeid"
	"github.com/spiffe/go-spiffe/v2/spiffetls/tlsconfig"
	"github.com/spiffe/go-spiffe/v2/workloadapi"
)

func main() {
	ctx := context.Background()

	// สร้าง X.509 source
	source, err := workloadapi.NewX509Source(ctx)
	if err != nil {
		log.Fatalf("Unable to create X509Source: %v", err)
	}
	defer source.Close()

	// ระบุ server SPIFFE ID
	serverID := spiffeid.RequireIDFromString(
		"spiffe://example.com/ns/production/sa/payment-api",
	)

	// สร้าง HTTP client ที่ใช้ mTLS
	client := &http.Client{
		Transport: &http.Transport{
			TLSClientConfig: tlsconfig.MTLSClientConfig(
				source,
				source,
				tlsconfig.AuthorizeID(serverID),
			),
		},
		Timeout: 10 * time.Second,
	}

	// ทำ request
	resp, err := client.Get("https://payment-api:8443/process")
	if err != nil {
		log.Fatalf("Request failed: %v", err)
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Printf("Response: %s\n", body)
}
```

---

## 5. Network Policies สำหรับ Zero-Trust

### 5.1 Default Deny-All Policy

```yaml
# k8s/network-policies/default-deny.yaml
# ใช้ Default Deny-All เป็น baseline

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # ใช้กับทุก pods
  policyTypes:
    - Ingress
    - Egress
  # ไม่มี ingress/egress rules = deny all

---
# อนุญาต DNS เพื่อให้ resolve hostnames ได้
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 5.2 Microservice Network Policies

```yaml
# k8s/network-policies/frontend-policy.yaml
# Frontend สามารถรับ traffic จาก Ingress เท่านั้น
# และส่ง traffic ไปที่ Backend API เท่านั้น

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: frontend
      tier: web
  
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # รับจาก Ingress Controller เท่านั้น
    - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: ingress-nginx
        podSelector:
          matchLabels:
            app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080

  egress:
    # ส่งไปที่ Backend API เท่านั้น
    - to:
      - podSelector:
          matchLabels:
            app: backend-api
            tier: api
      ports:
        - protocol: TCP
          port: 3000
    
    # DNS
    - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53

---
# k8s/network-policies/backend-api-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-api-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend-api
      tier: api
  
  policyTypes:
    - Ingress
    - Egress
  
  ingress:
    # รับจาก Frontend เท่านั้น
    - from:
      - podSelector:
          matchLabels:
            app: frontend
            tier: web
      ports:
        - protocol: TCP
          port: 3000
    
    # รับจาก Admin tools (ใน namespace เดียวกัน)
    - from:
      - podSelector:
          matchLabels:
            role: admin-tool
      ports:
        - protocol: TCP
          port: 3000
  
  egress:
    # ส่งไปที่ Database เท่านั้น
    - to:
      - podSelector:
          matchLabels:
            app: postgres
            tier: database
      ports:
        - protocol: TCP
          port: 5432
    
    # ส่งไปที่ Redis Cache
    - to:
      - podSelector:
          matchLabels:
            app: redis
            tier: cache
      ports:
        - protocol: TCP
          port: 6379
    
    # DNS
    - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## 6. Zero-Trust Pipeline Implementation

### 6.1 Pipeline Identity สำหรับทุก Stage

```yaml
# .github/workflows/zero-trust-pipeline.yml
name: Zero-Trust CI/CD Pipeline

on:
  push:
    branches: [main]

# ไม่มี implicit permissions
permissions: {}

jobs:
  # Job 1: Build — identity สำหรับ build process
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read          # อ่าน code
      packages: write         # push image
      id-token: write         # OIDC token สำหรับ cloud auth
    
    outputs:
      image-digest: ${{ steps.push.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4

      # Authenticate กับ Registry ด้วย OIDC (ไม่มี long-lived credentials)
      - name: Authenticate to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        id: push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          provenance: true

  # Job 2: Security Scan — identity สำหรับ security scanning
  security-scan:
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: read
      security-events: write  # upload SARIF
      id-token: write         # OIDC สำหรับ security tools
    
    steps:
      - uses: actions/checkout@v4

      # Authenticate ด้วย OIDC สำหรับ security tools
      - name: Configure AWS for Security Scan
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/SecurityScanRole
          aws-region: ap-southeast-1

      - name: Run Security Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ github.repository }}:${{ github.sha }}
          format: sarif
          output: trivy.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy.sarif

  # Job 3: Deploy to Staging — identity สำหรับ staging deployment
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    permissions:
      contents: read
      id-token: write   # OIDC สำหรับ staging cluster
    
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - uses: actions/checkout@v4

      # Authenticate กับ staging cluster ด้วย OIDC
      - name: Authenticate to Staging Cluster
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/StagingDeployRole
          aws-region: ap-southeast-1
          # จำกัด session ให้สั้นที่สุด
          role-duration-seconds: 900  # 15 minutes

      - name: Update Kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name staging-cluster \
            --region ap-southeast-1

      - name: Deploy to Staging
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }} \
            -n staging
          
          kubectl rollout status deployment/myapp -n staging --timeout=5m

  # Job 4: Integration Tests
  integration-tests:
    runs-on: ubuntu-latest
    needs: deploy-staging
    permissions:
      contents: read
      id-token: write
    
    steps:
      - uses: actions/checkout@v4

      # Authenticate ด้วย OIDC สำหรับ test resources
      - name: Authenticate for Tests
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/IntegrationTestRole
          aws-region: ap-southeast-1
          role-duration-seconds: 1800  # 30 minutes สำหรับ tests

      - name: Run Integration Tests
        run: |
          npm run test:integration -- \
            --baseUrl https://staging.example.com \
            --timeout 30000

  # Job 5: Deploy to Production — most restricted identity
  deploy-production:
    runs-on: ubuntu-latest
    needs: integration-tests
    permissions:
      contents: read
      id-token: write
    
    environment:
      name: production
      url: https://www.example.com
    
    steps:
      - uses: actions/checkout@v4

      # Authenticate ด้วย OIDC สำหรับ production — มีข้อจำกัดสูงสุด
      - name: Authenticate to Production
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ProductionDeployRole
          aws-region: ap-southeast-1
          role-duration-seconds: 600  # 10 minutes เท่านั้น

      - name: Update Kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name production-cluster \
            --region ap-southeast-1

      # Verify image signature ก่อน deploy
      - name: Verify Image Signature
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign verify \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            "ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }}"

      - name: Deploy to Production
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}@${{ needs.build.outputs.image-digest }} \
            -n production
          
          kubectl rollout status deployment/myapp -n production --timeout=10m
```

---

## 7. Workload Identity ใน Kubernetes

### 7.1 Kubernetes Service Account Token Projection

```yaml
# k8s/workload-identity/payment-api.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-api
  namespace: production
  annotations:
    # AWS: Bind service account กับ IAM Role ด้วย IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/PaymentAPIRole
    # GCP: Bind กับ GSA
    iam.gke.io/gcp-service-account: payment-api@my-project.iam.gserviceaccount.com

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-api
  template:
    metadata:
      labels:
        app: payment-api
    spec:
      serviceAccountName: payment-api
      
      # ปิดการใช้ default service account token
      automountServiceAccountToken: false
      
      containers:
        - name: payment-api
          image: ghcr.io/myorg/payment-api:latest
          
          # Mount projected service account token
          volumeMounts:
            - name: aws-token
              mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
              readOnly: true
            - name: spire-socket
              mountPath: /tmp/spire-agent/public
              readOnly: true
          
          # Security context
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            runAsGroup: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          
          # Limit resources
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
      
      volumes:
        # AWS service account token
        - name: aws-token
          projected:
            sources:
              - serviceAccountToken:
                  audience: sts.amazonaws.com
                  expirationSeconds: 3600
                  path: token
        
        # SPIRE agent socket
        - name: spire-socket
          hostPath:
            path: /run/spire/sockets
            type: Directory
```

### 7.2 AWS IAM Roles for Service Accounts (IRSA)

```hcl
# terraform/irsa-payment-api.tf

# OIDC provider สำหรับ EKS cluster
data "aws_eks_cluster" "production" {
  name = "production-cluster"
}

data "aws_iam_openid_connect_provider" "eks" {
  url = data.aws_eks_cluster.production.identity[0].oidc[0].issuer
}

# IAM Role สำหรับ payment-api
resource "aws_iam_role" "payment_api" {
  name = "PaymentAPIRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = data.aws_iam_openid_connect_provider.eks.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud" = "sts.amazonaws.com"
            # จำกัดเฉพาะ service account นี้ใน namespace นี้เท่านั้น
            "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub" = "system:serviceaccount:production:payment-api"
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "payment_api" {
  name   = "PaymentAPIPolicy"
  role   = aws_iam_role.payment_api.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        # Stripe API key ใน Secrets Manager
        Effect   = "Allow"
        Action   = ["secretsmanager:GetSecretValue"]
        Resource = "arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:payment-api/stripe-*"
      },
      {
        # DynamoDB สำหรับ payment records
        Effect = "Allow"
        Action = [
          "dynamodb:PutItem",
          "dynamodb:GetItem",
          "dynamodb:UpdateItem",
          "dynamodb:Query"
        ]
        Resource = "arn:aws:dynamodb:ap-southeast-1:123456789012:table/PaymentRecords"
      }
    ]
  })
}
```

---

## 8. Workshop: Zero-Trust Lab

### Lab 1: OIDC Setup สำหรับ AWS

```bash
#!/bin/bash
# workshop/lab1-aws-oidc.sh
# Setup OIDC Provider และ IAM Role สำหรับ GitHub Actions

set -euo pipefail

# Variables
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
GITHUB_ORG="myorg"
GITHUB_REPO="myapp"
ROLE_NAME="GitHubActionsRole"
REGION="ap-southeast-1"

echo "Setting up OIDC for GitHub Actions..."
echo "Account: ${AWS_ACCOUNT_ID}"
echo "Repository: ${GITHUB_ORG}/${GITHUB_REPO}"

# 1. สร้าง OIDC Provider (ถ้ายังไม่มี)
OIDC_URL="https://token.actions.githubusercontent.com"
THUMBPRINT=$(curl -sS https://token.actions.githubusercontent.com/.well-known/openid-configuration | \
    jq -r '.jwks_uri' | \
    xargs curl -sS | \
    jq -r '.keys[0].x5c[0]' | \
    base64 -d | \
    openssl x509 -noout -fingerprint -sha1 | \
    cut -d'=' -f2 | \
    tr -d ':' | \
    tr '[:upper:]' '[:lower:]')

echo "OIDC Thumbprint: ${THUMBPRINT}"

# Check ถ้า provider มีอยู่แล้ว
EXISTING_PROVIDER=$(aws iam list-open-id-connect-providers | \
    jq -r ".OpenIDConnectProviderList[] | select(.Arn | contains(\"token.actions.githubusercontent.com\")) | .Arn" || echo "")

if [ -z "${EXISTING_PROVIDER}" ]; then
    echo "Creating OIDC Provider..."
    aws iam create-open-id-connect-provider \
        --url "${OIDC_URL}" \
        --client-id-list "sts.amazonaws.com" \
        --thumbprint-list "${THUMBPRINT}"
    echo "✅ OIDC Provider created"
else
    echo "✅ OIDC Provider already exists: ${EXISTING_PROVIDER}"
fi

# 2. สร้าง Trust Policy
PROVIDER_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"

TRUST_POLICY=$(cat << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "${PROVIDER_ARN}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GITHUB_ORG}/${GITHUB_REPO}:*"
        }
      }
    }
  ]
}
EOF
)

# 3. สร้าง IAM Role
echo "Creating IAM Role: ${ROLE_NAME}..."
aws iam create-role \
    --role-name "${ROLE_NAME}" \
    --assume-role-policy-document "${TRUST_POLICY}" || {
    echo "Role exists, updating trust policy..."
    aws iam update-assume-role-policy \
        --role-name "${ROLE_NAME}" \
        --policy-document "${TRUST_POLICY}"
}

# 4. Attach permissions
aws iam attach-role-policy \
    --role-name "${ROLE_NAME}" \
    --policy-arn "arn:aws:iam::aws:policy/AmazonECR-FullAccess"

echo "✅ Setup complete!"
echo ""
echo "Use this in your workflow:"
echo "  role-to-assume: arn:aws:iam::${AWS_ACCOUNT_ID}:role/${ROLE_NAME}"
```

### Lab 2: Implement mTLS ระหว่าง Services

```yaml
# workshop/lab2-mtls/docker-compose.yml
version: '3.8'

services:
  # SPIRE Server
  spire-server:
    image: ghcr.io/spiffe/spire-server:1.9.0
    hostname: spire-server
    command: -config /run/spire/config/server.conf
    volumes:
      - ./spire-server.conf:/run/spire/config/server.conf:ro
      - spire-data:/run/spire/data
    ports:
      - "8081:8081"

  # Service A (Frontend)
  frontend:
    build:
      context: ./frontend
    depends_on:
      - spire-server
    volumes:
      - ./spire-agent-frontend.conf:/run/spire/config/agent.conf:ro
      - spire-sockets:/run/spire/sockets
    environment:
      - SPIFFE_ENDPOINT_SOCKET=unix:///run/spire/sockets/agent.sock
      - BACKEND_URL=https://backend:8443

  # Service B (Backend)
  backend:
    build:
      context: ./backend
    depends_on:
      - spire-server
    volumes:
      - ./spire-agent-backend.conf:/run/spire/config/agent.conf:ro
      - spire-sockets:/run/spire/sockets
    environment:
      - SPIFFE_ENDPOINT_SOCKET=unix:///run/spire/sockets/agent.sock
      - PORT=8443

volumes:
  spire-data:
  spire-sockets:
```

---

## 9. Zero-Trust Monitoring

### 9.1 Security Events ที่ต้อง Monitor

```yaml
# monitoring/zero-trust-alerts.yaml
groups:
  - name: zero-trust-alerts
    rules:
      # Alert: Authentication failures
      - alert: HighAuthenticationFailureRate
        expr: |
          rate(authentication_failures_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High authentication failure rate"
          description: "มี authentication failures สูงกว่าปกติ — อาจมีการ attack"

      # Alert: Privilege escalation attempts
      - alert: PrivilegeEscalationAttempt
        expr: |
          rate(kubernetes_audit_events_total{verb="create", objectRef_resource="clusterrolebindings"}[5m]) > 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Privilege escalation attempt detected"
          description: "มีการพยายามสร้าง ClusterRoleBinding — อาจเป็นการโจมตี"

      # Alert: Unusual API access
      - alert: UnusualAPIAccess
        expr: |
          rate(kubernetes_audit_events_total{
            user_username!~"system:.*",
            verb=~"create|update|delete",
            objectRef_namespace="kube-system"
          }[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusual kube-system access"
          description: "Non-system user accessing kube-system namespace"

      # Alert: Expired certificate renewal failure
      - alert: CertificateRenewalFailure
        expr: |
          (certmanager_certificate_expiration_timestamp_seconds - time()) < 86400  # 24 hours
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "Certificate expiring soon"
          description: "Certificate {{ $labels.name }} จะหมดอายุใน 24 ชั่วโมง"
```

---

## 10. สรุป Zero-Trust ใน CI/CD

### Implementation Checklist

```markdown
## Zero-Trust CI/CD Checklist

### Identity
- [ ] ทุก pipeline job มี unique identity (OIDC)
- [ ] ใช้ short-lived credentials เท่านั้น
- [ ] Service accounts มีการ bind กับ specific workloads
- [ ] Implement SPIFFE/SPIRE สำหรับ workload identity

### Access Control
- [ ] Default deny-all สำหรับ network policies
- [ ] Least privilege สำหรับทุก service account
- [ ] Mutual authentication สำหรับ service-to-service communication
- [ ] Regular access reviews และ cleanup

### Monitoring
- [ ] Log ทุก authentication และ authorization event
- [ ] Alert สำหรับ unusual access patterns
- [ ] Certificate expiry monitoring
- [ ] Automated anomaly detection

### Pipeline
- [ ] แต่ละ stage ใช้ OIDC credentials แยกกัน
- [ ] Verify artifacts ก่อน deploy
- [ ] Audit trail สำหรับทุก deployment
- [ ] Automated rollback ถ้าพบ suspicious activity
```

---

## อ้างอิง

- [Zero Trust Architecture (NIST SP 800-207)](https://doi.org/10.6028/NIST.SP.800-207)
- [SPIFFE/SPIRE Documentation](https://spiffe.io/docs/latest/)
- [Google BeyondCorp Enterprise](https://cloud.google.com/beyondcorp)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [HashiCorp Vault JWT Auth](https://developer.hashicorp.com/vault/docs/auth/jwt)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

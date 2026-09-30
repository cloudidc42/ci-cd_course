# Part 46: Microservices CI/CD

## สารบัญ
1. [Challenges ของ Microservices](#challenges-ของ-microservices)
2. [Independent Deployment](#independent-deployment)
3. [Service Versioning](#service-versioning)
4. [API Compatibility](#api-compatibility)
5. [Contract Testing](#contract-testing)
6. [Deployment Order](#deployment-order)
7. [Exercises](#exercises)

---

## Challenges ของ Microservices

### ปัญหาหลักในการทำ CI/CD สำหรับ Microservices

```
Traditional Monolith:
────────────────────
App → Build → Test → Deploy

Microservices (10+ services):
─────────────────────────────
Service A ─┐
Service B ─┤
Service C ─┤── How to coordinate? ──> Production
Service D ─┤
...        ─┘
```

### ความท้าทาย 6 ประการ

```
1. Independent builds สำหรับแต่ละ service
2. Versioning และ API compatibility
3. Integration testing ระหว่าง services
4. Deployment ordering (dependencies)
5. Environment management (หลาย services)
6. Monitoring และ debugging ระหว่าง services
```

### โครงสร้าง Repository สำหรับ Microservices

#### Option 1: Polyrepo (แต่ละ service = 1 repo)
```
github.com/myorg/
├── service-auth/       # ทีม auth
├── service-payment/    # ทีม payment
├── service-user/       # ทีม user
├── service-order/      # ทีม order
└── service-notification/ # ทีม notification
```

#### Option 2: Monorepo (ทุก services ใน 1 repo)
```
github.com/myorg/microservices/
├── services/
│   ├── auth/
│   ├── payment/
│   ├── user/
│   ├── order/
│   └── notification/
└── shared/
    ├── proto/           # gRPC definitions
    └── libraries/       # Shared code
```

---

## Independent Deployment

### Pipeline ต่อ Service

```yaml
# services/auth/.github/workflows/ci-cd.yaml
name: Auth Service CI/CD

on:
  push:
    branches: [main]
    paths:
    - 'services/auth/**'
    - '.github/workflows/auth*.yaml'
  pull_request:
    paths:
    - 'services/auth/**'

defaults:
  run:
    working-directory: services/auth

env:
  SERVICE_NAME: auth
  DOCKER_IMAGE: ghcr.io/myorg/service-auth

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Go
      uses: actions/setup-go@v5
      with:
        go-version-file: services/auth/go.mod
        cache-dependency-path: services/auth/go.sum
    
    - name: Run unit tests
      run: go test ./... -v -count=1 -race
    
    - name: Run integration tests
      run: go test ./... -tags=integration -v
      env:
        DATABASE_URL: postgresql://test:test@localhost:5432/testdb
    
    - name: Generate coverage report
      run: |
        go test ./... -coverprofile=coverage.out
        go tool cover -func=coverage.out | grep total

  build:
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.DOCKER_IMAGE }}
        tags: |
          type=sha,format=long,prefix=
          type=raw,value=latest
          type=semver,pattern={{version}}
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: services/auth
        file: services/auth/Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://auth.staging.mycompany.com
    
    steps:
    - name: Deploy auth service to staging
      run: |
        helm upgrade --install auth-service ./helm \
          --namespace staging \
          --set image.tag=${{ needs.build.outputs.image-tag }} \
          --set replicaCount=2 \
          --values ./helm/values.staging.yaml

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://auth.mycompany.com
    
    steps:
    - name: Deploy auth service to production
      run: |
        helm upgrade --install auth-service ./helm \
          --namespace production \
          --set image.tag=${{ needs.build.outputs.image-tag }} \
          --set replicaCount=5 \
          --values ./helm/values.production.yaml
```

### Helm Chart ต่อ Service

```yaml
# services/auth/helm/Chart.yaml
apiVersion: v2
name: auth-service
description: Authentication microservice
type: application
version: 0.1.0
appVersion: "1.0"

dependencies:
- name: postgresql
  version: "13.x.x"
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
```

```yaml
# services/auth/helm/values.yaml
replicaCount: 1

image:
  repository: ghcr.io/myorg/service-auth
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false

resources:
  limits:
    cpu: 500m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 64Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

env:
  DATABASE_URL: ""
  JWT_SECRET: ""
  TOKEN_EXPIRY: "24h"

postgresql:
  enabled: true
  auth:
    postgresPassword: "changeme"
    database: "authdb"
```

---

## Service Versioning

### Semantic Versioning สำหรับ APIs

```
MAJOR.MINOR.PATCH

MAJOR: Breaking changes (v1 → v2)
MINOR: New features (backward compatible)
PATCH: Bug fixes (backward compatible)

ตัวอย่าง:
v1.0.0 - Initial release
v1.1.0 - เพิ่ม endpoint ใหม่
v1.1.1 - Bug fix
v2.0.0 - Breaking changes (ใช้ /v2/ prefix)
```

### URL Versioning

```go
// services/auth/main.go
package main

import (
    "github.com/gin-gonic/gin"
    v1 "myapp/handlers/v1"
    v2 "myapp/handlers/v2"
)

func main() {
    r := gin.Default()
    
    // V1 API (maintained for backward compatibility)
    v1Group := r.Group("/api/v1")
    {
        v1Group.POST("/login", v1.Login)
        v1Group.POST("/logout", v1.Logout)
        v1Group.GET("/profile", v1.GetProfile)
    }
    
    // V2 API (new version)
    v2Group := r.Group("/api/v2")
    {
        v2Group.POST("/login", v2.Login)   // Enhanced login
        v2Group.POST("/logout", v2.Logout)
        v2Group.GET("/profile", v2.GetProfile)
        v2Group.POST("/refresh", v2.RefreshToken)  // New endpoint
    }
    
    r.Run(":8080")
}
```

### Header Versioning

```go
// Middleware สำหรับ version routing
func VersionMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        version := c.GetHeader("Accept-Version")
        if version == "" {
            version = "v1"  // default
        }
        c.Set("api-version", version)
        c.Next()
    }
}
```

### Container Image Tagging Strategy

```bash
# Strategy 1: Semver
ghcr.io/myorg/service-auth:1.2.3
ghcr.io/myorg/service-auth:1.2
ghcr.io/myorg/service-auth:1
ghcr.io/myorg/service-auth:latest

# Strategy 2: Git SHA (immutable, ดีสำหรับ traceability)
ghcr.io/myorg/service-auth:abc123def456

# Strategy 3: Combined (best)
ghcr.io/myorg/service-auth:1.2.3-abc123de

# แต่ละ service มี version ของตัวเอง
ghcr.io/myorg/service-auth:2.1.0
ghcr.io/myorg/service-payment:1.5.2
ghcr.io/myorg/service-user:3.0.1
```

---

## API Compatibility

### Backward Compatibility Rules

```
สิ่งที่ SAFE (backward compatible):
✅ เพิ่ม field ใหม่ใน response
✅ เพิ่ม endpoint ใหม่
✅ เพิ่ม optional query parameter
✅ เพิ่ม optional request body field

สิ่งที่ BREAKING:
❌ ลบ field ออกจาก response
❌ เปลี่ยน type ของ field
❌ ลบ endpoint
❌ เปลี่ยน required parameters เป็น optional
❌ เปลี่ยน response status codes
```

### OpenAPI Spec Compatibility Check

```yaml
# .github/workflows/api-compatibility.yaml
name: API Compatibility Check

on:
  pull_request:
    paths:
    - 'services/*/api/**'
    - 'services/*/*.yaml'
    - 'services/*/openapi.yaml'

jobs:
  check-compatibility:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Setup openapi-diff
      run: |
        npm install -g openapi-diff
        # หรือใช้ oasdiff
        brew install tufin/tufin/oasdiff  # macOS
    
    - name: Check API compatibility
      run: |
        for service_dir in services/*/; do
          SERVICE=$(basename $service_dir)
          SPEC_FILE="$service_dir/openapi.yaml"
          
          if [ ! -f "$SPEC_FILE" ]; then
            continue
          fi
          
          # ดึง spec จาก main branch
          git show origin/main:$SPEC_FILE > /tmp/old-spec.yaml 2>/dev/null || continue
          
          echo "Checking $SERVICE API compatibility..."
          
          # ตรวจสอบ breaking changes
          oasdiff breaking /tmp/old-spec.yaml $SPEC_FILE || {
            echo "❌ Breaking changes detected in $SERVICE!"
            exit 1
          }
          
          echo "✅ $SERVICE API is backward compatible"
        done
```

### API Contract Definition (OpenAPI)

```yaml
# services/auth/openapi.yaml
openapi: 3.0.0
info:
  title: Auth Service API
  version: 2.1.0
  description: Authentication and authorization service

paths:
  /api/v1/login:
    post:
      summary: User login
      operationId: loginV1
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
              - email
              - password
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
                  minLength: 8
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  token:
                    type: string
                  expires_at:
                    type: string
                    format: date-time
        '401':
          description: Invalid credentials
  
  /api/v2/login:
    post:
      summary: User login (v2 - enhanced)
      operationId: loginV2
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required:
              - email
              - password
              properties:
                email:
                  type: string
                password:
                  type: string
                mfa_code:        # ✅ new optional field
                  type: string
                  description: MFA code (optional)
      responses:
        '200':
          description: Login successful
          content:
            application/json:
              schema:
                type: object
                properties:
                  token:
                    type: string
                  refresh_token:   # ✅ new field in response
                    type: string
                  expires_at:
                    type: string
                  mfa_required:    # ✅ new field in response
                    type: boolean
```

---

## Contract Testing

Contract Testing ช่วยให้ services ทำงานร่วมกันได้โดยไม่ต้องทำ end-to-end integration tests บ่อยๆ

### Pact (Consumer-Driven Contract Testing)

```
Consumer-Driven Contract Testing:
─────────────────────────────────

1. Consumer (frontend/other service) สร้าง "pact" = contract
2. Contract บอกว่า consumer คาดหวังอะไรจาก provider
3. Provider ต้อง verify ว่าตัวเองตรง contract

Consumer (web) ──defines contract──> Pact Broker
Provider (auth) ──verifies contract── Pact Broker
```

#### Consumer Side (Node.js example)

```typescript
// services/web/src/tests/auth.consumer.pact.ts
import { Pact } from '@pact-foundation/pact';
import { AuthClient } from '../clients/auth-client';

const provider = new Pact({
  consumer: 'web-frontend',
  provider: 'auth-service',
  log: path.resolve(process.cwd(), 'logs', 'pact.log'),
  dir: path.resolve(process.cwd(), 'pacts'),
  logLevel: 'warn',
});

describe('Auth Service Contract', () => {
  beforeAll(() => provider.setup());
  afterAll(() => provider.finalize());
  afterEach(() => provider.verify());
  
  describe('POST /api/v1/login', () => {
    it('should return token on successful login', async () => {
      // กำหนด interaction
      await provider.addInteraction({
        state: 'user exists with valid credentials',
        uponReceiving: 'a request to login',
        withRequest: {
          method: 'POST',
          path: '/api/v1/login',
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            email: 'user@example.com',
            password: 'password123',
          },
        },
        willRespondWith: {
          status: 200,
          headers: {
            'Content-Type': 'application/json',
          },
          body: {
            token: like('eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'),
            expires_at: like('2024-01-15T10:00:00Z'),
          },
        },
      });
      
      // ทดสอบ
      const client = new AuthClient(`http://localhost:${provider.opts.port}`);
      const result = await client.login('user@example.com', 'password123');
      
      expect(result.token).toBeDefined();
      expect(result.expires_at).toBeDefined();
    });
    
    it('should return 401 on invalid credentials', async () => {
      await provider.addInteraction({
        state: 'user does not exist',
        uponReceiving: 'a request to login with invalid credentials',
        withRequest: {
          method: 'POST',
          path: '/api/v1/login',
          body: {
            email: 'notexist@example.com',
            password: 'wrongpassword',
          },
        },
        willRespondWith: {
          status: 401,
        },
      });
      
      const client = new AuthClient(`http://localhost:${provider.opts.port}`);
      await expect(client.login('notexist@example.com', 'wrongpassword'))
        .rejects.toThrow('Unauthorized');
    });
  });
});
```

#### Provider Side (Go example)

```go
// services/auth/pact_test.go
package main

import (
    "fmt"
    "testing"
    
    "github.com/pact-foundation/pact-go/v2/provider"
)

func TestPactProvider(t *testing.T) {
    verifier := provider.HTTPVerifier{}
    
    verifier.VerifyProvider(t, provider.VerifyRequest{
        ProviderBaseURL: "http://localhost:8080",
        BrokerURL:       "https://pact-broker.mycompany.com",
        BrokerToken:     os.Getenv("PACT_BROKER_TOKEN"),
        Provider:        "auth-service",
        
        // State handlers
        StateHandlers: provider.StateHandlers{
            "user exists with valid credentials": func(setup bool, s provider.ProviderStateV3) (provider.ProviderStateV3Response, error) {
                if setup {
                    // Create test user in database
                    db.Exec("INSERT INTO users (email, password_hash) VALUES ($1, $2)",
                        "user@example.com",
                        hashPassword("password123"),
                    )
                } else {
                    // Cleanup
                    db.Exec("DELETE FROM users WHERE email = $1", "user@example.com")
                }
                return provider.ProviderStateV3Response{}, nil
            },
            "user does not exist": func(setup bool, s provider.ProviderStateV3) (provider.ProviderStateV3Response, error) {
                // No setup needed, user doesn't exist
                return provider.ProviderStateV3Response{}, nil
            },
        },
        
        // Filter contracts by consumer
        ConsumerVersionSelectors: []provider.ConsumerVersionSelector{
            {MainBranch: true},
        },
        
        PublishVerificationResults: true,
        ProviderVersion:            os.Getenv("VERSION"),
    })
}
```

### Pact Broker Setup

```yaml
# docker-compose.pact.yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: pact
      POSTGRES_PASSWORD: pact
      POSTGRES_DB: pact
  
  pact-broker:
    image: pactfoundation/pact-broker:latest
    depends_on:
    - postgres
    ports:
    - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: "postgres://pact:pact@postgres/pact"
      PACT_BROKER_BASIC_AUTH_USERNAME: admin
      PACT_BROKER_BASIC_AUTH_PASSWORD: admin
```

### CI/CD Integration กับ Pact

```yaml
# .github/workflows/pact.yaml
name: Pact Contract Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  # Consumer: Web Frontend
  consumer-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Run consumer pact tests
      run: |
        cd services/web
        npm ci
        npm run test:pact
      env:
        PACT_BROKER_URL: https://pact-broker.mycompany.com
        PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
    
    - name: Publish pacts to broker
      run: |
        npx pact-broker publish ./pacts \
          --broker-base-url https://pact-broker.mycompany.com \
          --broker-token ${{ secrets.PACT_BROKER_TOKEN }} \
          --consumer-app-version ${{ github.sha }} \
          --branch ${{ github.ref_name }}

  # Provider: Auth Service
  provider-verification:
    runs-on: ubuntu-latest
    needs: consumer-tests
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
        - 5432:5432
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Start auth service
      run: |
        cd services/auth
        go build -o auth-server .
        DATABASE_URL=postgresql://test:test@localhost:5432/testdb ./auth-server &
        sleep 5
    
    - name: Verify pacts
      run: |
        cd services/auth
        go test -run TestPactProvider -v ./...
      env:
        PACT_BROKER_URL: https://pact-broker.mycompany.com
        PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
        VERSION: ${{ github.sha }}

  # Can-I-Deploy check
  can-i-deploy:
    needs: provider-verification
    runs-on: ubuntu-latest
    steps:
    - name: Can I deploy auth service?
      run: |
        npx pact-broker can-i-deploy \
          --broker-base-url https://pact-broker.mycompany.com \
          --broker-token ${{ secrets.PACT_BROKER_TOKEN }} \
          --pacticipant auth-service \
          --version ${{ github.sha }} \
          --to-environment production
```

---

## Deployment Order

### Dependency Management

```
Services dependency graph:
─────────────────────────
notification-service
    └─ depends on: user-service, email-service

order-service
    └─ depends on: user-service, payment-service, inventory-service

payment-service
    └─ depends on: user-service

user-service
    └─ no dependencies (deploy first)

email-service
    └─ no dependencies (deploy first)

inventory-service
    └─ no dependencies (deploy first)
```

### Deployment Orchestration

```yaml
# .github/workflows/deploy-all.yaml
name: Deploy All Services

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]

jobs:
  # Wave 1: Foundation services (no dependencies)
  deploy-user-service:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
    - name: Deploy user-service
      run: |
        helm upgrade --install user-service ./services/user/helm \
          --namespace ${{ github.event.inputs.environment }}

  deploy-email-service:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
    - name: Deploy email-service
      run: |
        helm upgrade --install email-service ./services/email/helm \
          --namespace ${{ github.event.inputs.environment }}

  deploy-inventory-service:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
    - name: Deploy inventory-service
      run: |
        helm upgrade --install inventory-service ./services/inventory/helm \
          --namespace ${{ github.event.inputs.environment }}

  # Wave 2: Services ที่ depends on Wave 1
  deploy-payment-service:
    needs: deploy-user-service
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
    - name: Verify user-service is ready
      run: |
        kubectl rollout status deployment/user-service \
          -n ${{ github.event.inputs.environment }} \
          --timeout=5m
    
    - name: Deploy payment-service
      run: |
        helm upgrade --install payment-service ./services/payment/helm \
          --namespace ${{ github.event.inputs.environment }}

  # Wave 3: Services ที่ depends on Wave 1 และ Wave 2
  deploy-order-service:
    needs: [deploy-user-service, deploy-payment-service, deploy-inventory-service]
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
    - name: Deploy order-service
      run: |
        helm upgrade --install order-service ./services/order/helm \
          --namespace ${{ github.event.inputs.environment }}

  deploy-notification-service:
    needs: [deploy-user-service, deploy-email-service]
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    steps:
    - name: Deploy notification-service
      run: |
        helm upgrade --install notification-service ./services/notification/helm \
          --namespace ${{ github.event.inputs.environment }}
```

### Health Check Verification

```bash
#!/bin/bash
# scripts/wait-for-service.sh

SERVICE_NAME=$1
NAMESPACE=$2
TIMEOUT=${3:-300}  # default 5 minutes

echo "Waiting for $SERVICE_NAME to be ready in $NAMESPACE..."

# รอ deployment ready
kubectl rollout status deployment/$SERVICE_NAME \
  -n $NAMESPACE \
  --timeout=${TIMEOUT}s

# รอ pods healthy
READY=0
ELAPSED=0
while [ $ELAPSED -lt $TIMEOUT ]; do
  READY=$(kubectl get deployment $SERVICE_NAME \
    -n $NAMESPACE \
    -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo 0)
  
  DESIRED=$(kubectl get deployment $SERVICE_NAME \
    -n $NAMESPACE \
    -o jsonpath='{.spec.replicas}' 2>/dev/null || echo 1)
  
  if [ "$READY" = "$DESIRED" ]; then
    echo "✅ $SERVICE_NAME is ready ($READY/$DESIRED)"
    exit 0
  fi
  
  echo "Waiting... $READY/$DESIRED ready ($ELAPSED/$TIMEOUT seconds)"
  sleep 10
  ELAPSED=$((ELAPSED + 10))
done

echo "❌ Timeout waiting for $SERVICE_NAME"
exit 1
```

### Database Migration Strategy

```yaml
# Deployment order สำหรับ database migrations
deploy-with-migration:
  runs-on: ubuntu-latest
  steps:
  
  # Step 1: Deploy backward-compatible migration
  - name: Run database migration
    run: |
      # Migration ต้อง backward compatible
      # (ไม่ลบ columns ที่ยังใช้อยู่)
      kubectl run migration \
        --image=myapp:${{ github.sha }} \
        --restart=Never \
        --rm \
        --attach \
        -- ./migrate up
  
  # Step 2: Deploy new version
  - name: Deploy new version
    run: |
      helm upgrade --install myapp ./helm \
        --set image.tag=${{ github.sha }}
  
  # Step 3: Verify
  - name: Verify deployment
    run: |
      kubectl rollout status deployment/myapp
      curl -f https://api.mycompany.com/health
  
  # Step 4: (หลังจากทุก services upgrade) cleanup old columns
  # เฉพาะหลังจาก deploy สำเร็จทั้งหมด
```

---

## Blue-Green สำหรับ Microservices

```yaml
# Kubernetes Blue-Green deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service-blue
  labels:
    app: auth-service
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
      version: blue
  template:
    metadata:
      labels:
        app: auth-service
        version: blue
    spec:
      containers:
      - name: auth-service
        image: ghcr.io/myorg/service-auth:1.2.3
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service-green
  labels:
    app: auth-service
    version: green
spec:
  replicas: 0  # เริ่มจาก 0
  selector:
    matchLabels:
      app: auth-service
      version: green
  template:
    metadata:
      labels:
        app: auth-service
        version: green
    spec:
      containers:
      - name: auth-service
        image: ghcr.io/myorg/service-auth:1.3.0  # new version
---
apiVersion: v1
kind: Service
metadata:
  name: auth-service
spec:
  selector:
    app: auth-service
    version: blue   # switch between blue/green
  ports:
  - port: 8080
```

```bash
# Blue-Green switch script
#!/bin/bash
# scripts/blue-green-switch.sh

NAMESPACE=$1
SERVICE=$2
NEW_VERSION=$3  # blue or green
OLD_VERSION=${NEW_VERSION == "blue" ? "green" : "blue"}

echo "Switching $SERVICE from $OLD_VERSION to $NEW_VERSION"

# Scale up new version
kubectl scale deployment ${SERVICE}-${NEW_VERSION} \
  --replicas=3 \
  -n $NAMESPACE

# Wait for new version ready
kubectl rollout status deployment/${SERVICE}-${NEW_VERSION} \
  -n $NAMESPACE \
  --timeout=5m

# Switch traffic
kubectl patch service $SERVICE \
  -n $NAMESPACE \
  -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_VERSION\"}}}"

echo "Traffic switched to $NEW_VERSION"

# Wait to verify
sleep 30

# ตรวจสอบ error rate
ERROR_RATE=$(curl -s http://prometheus:9090/api/v1/query?query=sum(rate(http_requests_total{service="$SERVICE",status=~"5.."}[1m]))/sum(rate(http_requests_total{service="$SERVICE"}[1m])) | jq '.data.result[0].value[1]')

if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
  echo "❌ High error rate detected ($ERROR_RATE), rolling back..."
  kubectl patch service $SERVICE \
    -n $NAMESPACE \
    -p "{\"spec\":{\"selector\":{\"version\":\"$OLD_VERSION\"}}}"
  exit 1
fi

# Scale down old version
kubectl scale deployment ${SERVICE}-${OLD_VERSION} \
  --replicas=0 \
  -n $NAMESPACE

echo "✅ Blue-green switch successful!"
```

---

## Exercises

### Exercise 1: สร้าง Microservices Pipeline

```yaml
# .github/workflows/microservices-ci.yaml
name: Microservices CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  detect-services:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.detect.outputs.services }}
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - id: detect
      run: |
        # หา services ที่เปลี่ยนแปลง
        CHANGED_SERVICES=$(git diff --name-only origin/${{ github.base_ref || 'main' }}...HEAD \
          | grep '^services/' \
          | cut -d/ -f2 \
          | sort -u \
          | jq -R -s -c 'split("\n") | map(select(length > 0))')
        
        echo "services=$CHANGED_SERVICES" >> $GITHUB_OUTPUT
        echo "Changed services: $CHANGED_SERVICES"

  build-and-test:
    needs: detect-services
    if: needs.detect-services.outputs.services != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJSON(needs.detect-services.outputs.services) }}
      fail-fast: false  # ไม่ stop ทุก services เมื่อ 1 fail
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Build and test ${{ matrix.service }}
      run: |
        cd services/${{ matrix.service }}
        
        # Detect language and run appropriate commands
        if [ -f "package.json" ]; then
          npm ci
          npm test
          npm run build
        elif [ -f "go.mod" ]; then
          go test ./...
          go build -o app .
        elif [ -f "pom.xml" ]; then
          mvn test package
        elif [ -f "requirements.txt" ]; then
          pip install -r requirements.txt
          pytest
        fi
    
    - name: Build Docker image
      if: github.event_name == 'push'
      run: |
        docker build \
          -t ghcr.io/myorg/service-${{ matrix.service }}:${{ github.sha }} \
          services/${{ matrix.service }}/
        
        echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
        docker push ghcr.io/myorg/service-${{ matrix.service }}:${{ github.sha }}
```

### Exercise 2: Contract Testing Setup

```bash
#!/bin/bash
# exercise-2-pact-setup.sh

# ติดตั้ง Pact Broker ด้วย Docker
docker-compose -f docker/pact-broker.yaml up -d

# รอ Broker ready
sleep 10

# ตรวจสอบ
curl -u admin:admin http://localhost:9292

# Run consumer tests
cd services/web
npm install
npm run test:pact

# Publish pacts
npx pact-broker publish ./pacts \
  --broker-base-url http://localhost:9292 \
  --broker-username admin \
  --broker-password admin \
  --consumer-app-version 1.0.0

# Run provider tests
cd ../auth
go test -run TestPactProvider -v ./...

# Check can-i-deploy
npx pact-broker can-i-deploy \
  --broker-base-url http://localhost:9292 \
  --broker-username admin \
  --broker-password admin \
  --pacticipant auth-service \
  --version 1.0.0 \
  --to-environment staging
```

### Exercise 3: Deployment Order

```yaml
# exercise-3-orchestrated-deploy.yaml
name: Orchestrated Microservices Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        default: staging

jobs:
  # Wave 1
  deploy-database-services:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-db, product-db, order-db]
    steps:
    - name: Deploy ${{ matrix.service }}
      run: echo "Deploying ${{ matrix.service }}"
  
  # Wave 2
  deploy-core-services:
    needs: deploy-database-services
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [user-service, product-service]
    steps:
    - name: Deploy ${{ matrix.service }}
      run: echo "Deploying ${{ matrix.service }}"
  
  # Wave 3
  deploy-dependent-services:
    needs: deploy-core-services
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [order-service, notification-service, api-gateway]
    steps:
    - name: Deploy ${{ matrix.service }}
      run: echo "Deploying ${{ matrix.service }}"
  
  # Wave 4: Integration tests
  integration-tests:
    needs: deploy-dependent-services
    runs-on: ubuntu-latest
    steps:
    - name: Run integration tests
      run: echo "Running integration tests across all services"
```

---

## สรุป

Microservices CI/CD ต้องการ:

1. **Independent pipelines** - แต่ละ service มี pipeline ของตัวเอง
2. **Semantic versioning** - ชัดเจนเรื่อง API compatibility
3. **Contract testing** - ตรวจสอบ compatibility ระหว่าง services
4. **Deployment orchestration** - จัดการ dependency ระหว่าง services
5. **Blue-green/Canary** - deploy safely ด้วย minimal downtime
6. **Health checks** - ตรวจสอบว่า services พร้อมก่อน switch traffic

Key principle: **"Each service should be deployable independently"**

---

*ส่วนต่อไป: Part 47 - API Gateway & Service Mesh*

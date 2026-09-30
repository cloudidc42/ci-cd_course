# Part 93: Case Study — E-commerce Platform CI/CD

## บทนำ

ในบทนี้เราจะศึกษา case study จริงของการ implement CI/CD สำหรับ e-commerce platform ขนาดใหญ่ ชื่อ **"ShopThai"** — แพลตฟอร์ม marketplace ที่มีผู้ใช้ 5 ล้านคน และมูลค่าธุรกรรม ฿2 พันล้าน/เดือน

---

## 93.1 Background และ Requirements

### ข้อมูลองค์กร

```yaml
organization:
  name: "ShopThai Marketplace"
  scale:
    monthly_active_users: 5_000_000
    transactions_per_day: 200_000
    gmv_per_month_thb: 2_000_000_000
    
  engineering:
    total_engineers: 85
    product_teams: 12
    microservices: 48
    
  current_pain_points:
    - "Release cycle 4 สัปดาห์ ทำให้ช้ากว่าคู่แข่ง"
    - "Production incidents บ่อย โดยเฉพาะวัน mega sales"
    - "Deploy แต่ละครั้ง downtime 15-30 นาที"
    - "ทีม QA เป็น bottleneck ต้องทดสอบ manual ทั้งหมด"
    - "5 ทีมสร้าง pipeline แตกต่างกัน"
    
  goals:
    - "Deploy ทุกวัน (multiple times ถ้าเป็นไปได้)"
    - "Zero downtime deployment"
    - "Automated testing ที่เชื่อถือได้"
    - "รองรับ traffic spike ช่วง mega sales (10x normal)"
```

### Business Requirements

```markdown
## Non-Functional Requirements (NFR) สำหรับ CI/CD

### Availability
- Production uptime: 99.9% (8.7 ชั่วโมง downtime/ปี)
- Deployment: Zero-downtime
- Rollback: ใน 5 นาที

### Performance
- Build time: < 15 นาที (per service)
- Full pipeline: < 30 นาที
- Deployment to prod: < 10 นาที

### Security
- Code scanning ทุก PR
- Container scanning ก่อน deploy
- No secrets ใน code

### Compliance
- PCI-DSS Level 1 (เพราะรับชำระเงิน)
- PDPA compliance
- SOC 2 Type II

### Scalability
- Pipeline ต้อง scale รองรับ 12 teams
- รองรับ 50 concurrent builds
```

---

## 93.2 Architecture Overview

### System Architecture

```
ShopThai Microservices Architecture
════════════════════════════════════

                    ┌─────────────────┐
                    │   CloudFront    │
                    │     (CDN)       │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │
                    │  (Kong / AWS)   │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
   ┌──────▼──────┐  ┌────────▼───────┐  ┌──────▼──────┐
   │  Product    │  │  Order Service │  │  User       │
   │  Service    │  │                │  │  Service    │
   └──────┬──────┘  └────────┬───────┘  └──────┬──────┘
          │                  │                  │
   ┌──────▼──────┐  ┌────────▼───────┐  ┌──────▼──────┐
   │  Inventory  │  │  Payment       │  │  Search     │
   │  Service    │  │  Service       │  │  Service    │
   └─────────────┘  └────────────────┘  └─────────────┘

Infrastructure:
- Kubernetes (EKS)
- 3 clusters: dev, staging, production
- Production: 3 availability zones
- Auto-scaling: HPA + KEDA
```

### CI/CD Infrastructure

```
CI/CD Infrastructure Stack
═══════════════════════════

Source Control:    GitHub Enterprise
CI Platform:       GitHub Actions (with self-hosted runners)
Registry:          Amazon ECR
CD/GitOps:         ArgoCD
Secrets:           AWS Secrets Manager + HashiCorp Vault
Environments:      
  - Development (shared)
  - Staging (per-feature)
  - Production (blue/green)

Observability:
  - Metrics: Prometheus + Grafana
  - Logs: ELK Stack
  - Tracing: Jaeger
  - Alerting: PagerDuty

Quality Gates:
  - SonarQube (code quality)
  - Snyk (dependencies)
  - Trivy (containers)
  - OWASP ZAP (DAST)
  - k6 (performance)
```

---

## 93.3 Pipeline Design

### Standard Microservice Pipeline

```yaml
# .github/workflows/microservice-pipeline.yml

name: Microservice CI/CD Pipeline

on:
  push:
    branches:
      - main
      - 'feature/**'
      - 'release/**'
  pull_request:
    branches:
      - main
      - develop

env:
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com
  SERVICE_NAME: ${{ github.event.repository.name }}
  AWS_REGION: ap-southeast-1

jobs:
  # ===== STAGE 1: CODE QUALITY =====
  code-quality:
    name: Code Quality Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for SonarQube

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint -- --format=@microsoft/eslint-formatter-sarif --output-file eslint-results.sarif
        continue-on-error: true

      - name: Upload ESLint results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: eslint-results.sarif

      - name: SonarQube Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        with:
          args: >
            -Dsonar.qualitygate.wait=true
            -Dsonar.coverage.exclusions=**/*.test.ts,**/*.spec.ts

  # ===== STAGE 2: SECURITY SCANNING =====
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: code-quality
    steps:
      - uses: actions/checkout@v4

      - name: Secret Scanning (GitLeaks)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Dependency Vulnerability Scan (Snyk)
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: License Check
        run: |
          npx license-checker --production --failOn "GPL-2.0;GPL-3.0;AGPL-3.0"

  # ===== STAGE 3: TEST =====
  test:
    name: Automated Tests
    runs-on: ubuntu-latest
    needs: code-quality

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run Unit Tests
        run: |
          npm run test:unit -- \
            --coverage \
            --coverageReporters=lcov \
            --coverageThreshold='{"global":{"lines":70,"branches":70}}'
        env:
          NODE_ENV: test

      - name: Run Integration Tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379

      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info

      - name: Publish Test Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Jest Tests
          path: 'test-results/*.xml'
          reporter: jest-junit

  # ===== STAGE 4: BUILD =====
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [security-scan, test]
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/heads/release/')
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: |
            network=host

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Generate Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.ECR_REGISTRY }}/${{ env.SERVICE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}
            VERSION=${{ github.ref_name }}

  # ===== STAGE 5: CONTAINER SECURITY =====
  container-security:
    name: Container Security Scan
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ needs.build.outputs.image-tag }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  # ===== STAGE 6: DEPLOY TO DEVELOPMENT =====
  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: container-security
    environment:
      name: development
      url: https://dev.shopthai.internal/${{ env.SERVICE_NAME }}

    steps:
      - uses: actions/checkout@v4

      - name: Update ArgoCD Application
        run: |
          # Update image tag ใน GitOps repo
          git clone https://github.com/shopthai/k8s-gitops.git
          cd k8s-gitops
          
          # ใช้ yq update image tag
          yq e ".spec.template.spec.containers[0].image = \"${{ needs.build.outputs.image-tag }}\"" \
            -i "environments/development/services/${{ env.SERVICE_NAME }}/deployment.yaml"
          
          git config user.email "cicd@shopthai.com"
          git config user.name "ShopThai CI/CD"
          git add .
          git commit -m "chore: update ${{ env.SERVICE_NAME }} to ${{ github.sha }}"
          git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITOPS_TOKEN }}

      - name: Wait for ArgoCD Sync
        run: |
          argocd app wait ${{ env.SERVICE_NAME }}-dev \
            --health \
            --timeout 300
        env:
          ARGOCD_SERVER: ${{ secrets.ARGOCD_SERVER }}
          ARGOCD_AUTH_TOKEN: ${{ secrets.ARGOCD_AUTH_TOKEN }}

      - name: Run Smoke Tests
        run: |
          npm run test:smoke -- \
            --baseUrl=https://dev.shopthai.internal/${{ env.SERVICE_NAME }}

  # ===== STAGE 7: DEPLOY TO STAGING =====
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    environment:
      name: staging
      url: https://staging.shopthai.internal/${{ env.SERVICE_NAME }}

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Staging
        run: |
          # Similar to dev deployment แต่ staging namespace
          git clone https://github.com/shopthai/k8s-gitops.git
          cd k8s-gitops
          yq e ".spec.template.spec.containers[0].image = \"${{ needs.build.outputs.image-tag }}\"" \
            -i "environments/staging/services/${{ env.SERVICE_NAME }}/deployment.yaml"
          git add . && git commit -m "chore: staging update ${{ env.SERVICE_NAME }}" && git push
        env:
          GITHUB_TOKEN: ${{ secrets.GITOPS_TOKEN }}

      - name: Run Integration Tests
        run: |
          npm run test:integration:staging -- \
            --baseUrl=https://staging.shopthai.internal

      - name: Run Performance Tests
        run: |
          k6 run tests/performance/load-test.js \
            --env BASE_URL=https://staging.shopthai.internal/${{ env.SERVICE_NAME }} \
            --out json=k6-results.json
          
          # Parse results และ fail ถ้า p95 > 500ms
          node scripts/check-performance.js k6-results.json

      - name: Run DAST Scan
        run: |
          docker run --rm owasp/zap2docker-stable zap-baseline.py \
            -t https://staging.shopthai.internal/${{ env.SERVICE_NAME }} \
            -J zap-results.json
          
          node scripts/check-zap-results.js zap-results.json

  # ===== STAGE 8: DEPLOY TO PRODUCTION =====
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://www.shopthai.com

    steps:
      - uses: actions/checkout@v4

      - name: Pre-deployment Health Check
        run: |
          # ตรวจสอบว่า production health ดีก่อน deploy
          ./scripts/check-production-health.sh

      - name: Blue/Green Deployment
        run: |
          ./scripts/blue-green-deploy.sh \
            --service ${{ env.SERVICE_NAME }} \
            --image ${{ needs.build.outputs.image-tag }} \
            --namespace production

      - name: Production Smoke Tests
        run: |
          npm run test:smoke:prod -- \
            --baseUrl=https://www.shopthai.com

      - name: Notify Success
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: "✅ ${{ env.SERVICE_NAME }} deployed to production successfully!\nImage: ${{ needs.build.outputs.image-tag }}"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
```

---

## 93.4 Deployment Strategy

### Blue/Green Deployment Script

```bash
#!/bin/bash
# blue-green-deploy.sh
# Blue/Green deployment สำหรับ ShopThai

set -euo pipefail

SERVICE_NAME=$1
IMAGE_TAG=$2
NAMESPACE=${3:-production}
SWITCH_TRAFFIC_WEIGHT=10  # เริ่มส่ง traffic 10% ไป green

echo "🚀 Starting Blue/Green deployment for $SERVICE_NAME"
echo "   Image: $IMAGE_TAG"
echo "   Namespace: $NAMESPACE"

# 1. Determine current active slot (blue หรือ green)
CURRENT_SLOT=$(kubectl get service $SERVICE_NAME \
    -n $NAMESPACE \
    -o jsonpath='{.spec.selector.slot}' 2>/dev/null || echo "blue")

if [ "$CURRENT_SLOT" = "blue" ]; then
    NEW_SLOT="green"
else
    NEW_SLOT="blue"
fi

echo "   Current slot: $CURRENT_SLOT → Deploying to: $NEW_SLOT"

# 2. Deploy ไปยัง inactive slot
kubectl set image deployment/$SERVICE_NAME-$NEW_SLOT \
    $SERVICE_NAME=$IMAGE_TAG \
    -n $NAMESPACE

# 3. รอให้ deployment พร้อม
echo "⏳ Waiting for $NEW_SLOT deployment to be ready..."
kubectl rollout status deployment/$SERVICE_NAME-$NEW_SLOT \
    -n $NAMESPACE \
    --timeout=300s

# 4. ตรวจสอบ health ของ new deployment
echo "🏥 Health checking new deployment..."
NEW_POD=$(kubectl get pods -n $NAMESPACE \
    -l "app=$SERVICE_NAME,slot=$NEW_SLOT" \
    -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n $NAMESPACE $NEW_POD -- \
    curl -sf http://localhost:8080/health || {
    echo "❌ Health check failed!"
    exit 1
}

# 5. ค่อยๆ shift traffic ไป green (canary)
echo "🔀 Gradually shifting traffic to $NEW_SLOT slot..."

for WEIGHT in 10 25 50 75 100; do
    echo "   Traffic to $NEW_SLOT: $WEIGHT%"
    
    # Update Istio VirtualService
    kubectl patch virtualservice $SERVICE_NAME \
        -n $NAMESPACE \
        --type=merge \
        -p "{
            \"spec\": {
                \"http\": [{
                    \"route\": [
                        {\"destination\": {\"host\": \"$SERVICE_NAME\", \"subset\": \"$CURRENT_SLOT\"}, \"weight\": $((100-WEIGHT))},
                        {\"destination\": {\"host\": \"$SERVICE_NAME\", \"subset\": \"$NEW_SLOT\"}, \"weight\": $WEIGHT}
                    ]
                }]
            }
        }"
    
    # รอ 2 นาที และตรวจสอบ error rate
    sleep 120
    
    ERROR_RATE=$(./scripts/check-error-rate.sh $SERVICE_NAME $NAMESPACE)
    if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
        echo "❌ Error rate too high ($ERROR_RATE%)! Rolling back..."
        
        # Rollback to previous slot
        kubectl patch virtualservice $SERVICE_NAME \
            -n $NAMESPACE \
            --type=merge \
            -p "{
                \"spec\": {
                    \"http\": [{
                        \"route\": [
                            {\"destination\": {\"host\": \"$SERVICE_NAME\", \"subset\": \"$CURRENT_SLOT\"}, \"weight\": 100}
                        ]
                    }]
                }
            }"
        
        # Send alert
        ./scripts/send-alert.sh "DEPLOYMENT_ROLLBACK" "$SERVICE_NAME rolled back from $NEW_SLOT due to high error rate"
        exit 1
    fi
    
    echo "   ✅ Error rate OK ($ERROR_RATE%)"
done

# 6. Update active slot label
kubectl patch service $SERVICE_NAME \
    -n $NAMESPACE \
    --type=merge \
    -p "{\"spec\": {\"selector\": {\"slot\": \"$NEW_SLOT\"}}}"

echo "✅ Blue/Green deployment complete!"
echo "   Active slot: $NEW_SLOT"
echo "   Image: $IMAGE_TAG"
```

### Feature Flag Integration

```typescript
// feature-flags.ts
// Feature flag integration สำหรับ safe releases

import { LaunchDarkly } from 'launchdarkly-node-server-sdk';

interface FeatureFlagContext {
  userId?: string;
  userTier?: 'free' | 'premium' | 'vip';
  region?: string;
}

class FeatureFlagService {
  private client: any;
  
  constructor(sdkKey: string) {
    this.client = LaunchDarkly.init(sdkKey);
  }
  
  async isEnabled(flagKey: string, context: FeatureFlagContext): Promise<boolean> {
    const user = {
      key: context.userId || 'anonymous',
      custom: {
        tier: context.userTier,
        region: context.region
      }
    };
    
    return await this.client.variation(flagKey, user, false);
  }
}

// ตัวอย่างการใช้งานใน product service
class ProductController {
  constructor(
    private productService: ProductService,
    private featureFlags: FeatureFlagService
  ) {}
  
  async getProductRecommendations(userId: string, productId: string) {
    const user = await this.getUserContext(userId);
    
    // Feature flag สำหรับ new recommendation algorithm
    const useNewAlgo = await this.featureFlags.isEnabled(
      'new-recommendation-algorithm',
      { userId, userTier: user.tier }
    );
    
    if (useNewAlgo) {
      return await this.productService.getRecommendationsV2(productId);
    } else {
      return await this.productService.getRecommendationsV1(productId);
    }
  }
}
```

---

## 93.5 Testing Strategy

### Test Pyramid สำหรับ ShopThai

```
                    ┌───────┐
                    │  E2E  │  (~50 tests, 20 min)
                   /│ Tests │\
                  / └───────┘ \
                 /   (Manual + \
                /   Automated)  \
               ─────────────────
              /  Integration    \  (~200 tests, 8 min)
             /     Tests         \
            ─────────────────────
           /    Unit Tests        \  (~2000 tests, 3 min)
          ───────────────────────
```

### Performance Test ด้วย k6

```javascript
// tests/performance/load-test.js
// Performance test สำหรับ product listing endpoint

import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('error_rate');
const checkoutDuration = new Trend('checkout_duration');

// Test configuration
export const options = {
  scenarios: {
    // Scenario 1: Normal load
    normal_load: {
      executor: 'constant-vus',
      vus: 100,
      duration: '5m',
      tags: { scenario: 'normal' }
    },
    
    // Scenario 2: Peak load (mega sales simulation)
    peak_load: {
      executor: 'ramping-vus',
      startVUs: 100,
      stages: [
        { duration: '2m', target: 500 },
        { duration: '5m', target: 1000 },  // 10x normal
        { duration: '2m', target: 100 },
      ],
      tags: { scenario: 'peak' }
    },
    
    // Scenario 3: Spike test
    spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 2000 },  // Sudden spike
        { duration: '1m', target: 2000 },
        { duration: '30s', target: 0 },
      ],
      tags: { scenario: 'spike' }
    }
  },
  
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% ต้อง < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
    error_rate: ['rate<0.01'],
  }
};

// Base URL จาก environment
const BASE_URL = __ENV.BASE_URL || 'https://staging.shopthai.com';

// Test scenarios
export default function() {
  const scenario = __ENV.K6_SCENARIO || 'browse';
  
  if (scenario === 'browse') {
    browseProducts();
  } else if (scenario === 'checkout') {
    checkoutFlow();
  }
}

function browseProducts() {
  // Homepage
  let res = http.get(`${BASE_URL}/api/v1/products/featured`);
  check(res, {
    'homepage status 200': (r) => r.status === 200,
    'homepage response time < 300ms': (r) => r.timings.duration < 300,
  });
  errorRate.add(res.status !== 200);
  
  sleep(1);
  
  // Search products
  res = http.get(`${BASE_URL}/api/v1/products/search?q=smartphone&page=1&limit=20`);
  check(res, {
    'search status 200': (r) => r.status === 200,
    'search has results': (r) => JSON.parse(r.body).data.length > 0,
  });
  errorRate.add(res.status !== 200);
  
  sleep(2);
  
  // Product detail
  const productId = getRandomProductId();
  res = http.get(`${BASE_URL}/api/v1/products/${productId}`);
  check(res, {
    'product detail status 200': (r) => r.status === 200,
  });
  
  sleep(1);
}

function checkoutFlow() {
  const startTime = Date.now();
  
  // Add to cart
  let res = http.post(
    `${BASE_URL}/api/v1/cart/items`,
    JSON.stringify({
      productId: getRandomProductId(),
      quantity: 1
    }),
    { headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${getTestToken()}` } }
  );
  
  check(res, { 'add to cart 201': (r) => r.status === 201 });
  
  sleep(1);
  
  // Get cart
  res = http.get(
    `${BASE_URL}/api/v1/cart`,
    { headers: { 'Authorization': `Bearer ${getTestToken()}` } }
  );
  
  check(res, { 'get cart 200': (r) => r.status === 200 });
  
  sleep(2);
  
  // Checkout
  res = http.post(
    `${BASE_URL}/api/v1/orders`,
    JSON.stringify({
      paymentMethod: 'promptpay',
      deliveryAddress: { /* ... */ }
    }),
    { headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${getTestToken()}` } }
  );
  
  check(res, { 'checkout 201': (r) => r.status === 201 });
  
  const duration = Date.now() - startTime;
  checkoutDuration.add(duration);
}

function getRandomProductId() {
  const ids = ['prod-001', 'prod-002', 'prod-003', 'prod-100'];
  return ids[Math.floor(Math.random() * ids.length)];
}

function getTestToken() {
  return __ENV.TEST_USER_TOKEN || 'test-token';
}
```

---

## 93.6 Monitoring และ Observability

### Grafana Dashboard Configuration

```json
{
  "title": "ShopThai - Service Health Dashboard",
  "panels": [
    {
      "title": "Deployment Frequency",
      "type": "stat",
      "targets": [
        {
          "expr": "count(deployment_timestamp{environment='production'}[24h])",
          "legendFormat": "Deployments/day"
        }
      ],
      "fieldConfig": {
        "thresholds": {
          "steps": [
            {"value": 0, "color": "red"},
            {"value": 1, "color": "yellow"},
            {"value": 3, "color": "green"}
          ]
        }
      }
    },
    {
      "title": "Error Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status=~'5..'}[5m])) / sum(rate(http_requests_total[5m])) * 100",
          "legendFormat": "Error Rate %"
        }
      ]
    },
    {
      "title": "P95 Response Time",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
          "legendFormat": "{{service}} p95"
        }
      ]
    }
  ]
}
```

### Alerting Rules

```yaml
# prometheus-alerts.yml

groups:
  - name: shopthai-sla
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Error rate สูงเกิน 1%"
          description: "Service {{ $labels.service }} มี error rate {{ $value | humanizePercentage }}"
          runbook_url: "https://runbooks.shopthai.internal/high-error-rate"
          
      - alert: SlowResponseTime
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Response time สูง"
          description: "{{ $labels.service }} p95 = {{ $value }}s"
          
      - alert: DeploymentFailed
        expr: |
          kube_deployment_status_replicas_unavailable > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Deployment ไม่สำเร็จ"
          description: "{{ $labels.deployment }} มี {{ $value }} unavailable replicas"
```

---

## 93.7 Security Implementation

### PCI-DSS Pipeline Controls

```yaml
# pci-dss-controls.yml
# Security controls สำหรับ PCI-DSS compliance ใน CI/CD

controls:
  req_6_3:
    name: "Develop software securely"
    pipeline_controls:
      - name: "Code Review Required"
        implementation: "GitHub branch protection: required reviewers"
        verification: "PR cannot merge without approval"
        
      - name: "SAST Scanning"
        implementation: "SonarQube + custom rules for payment data"
        verification: "Quality gate must pass"
        failure_action: "Block deployment"
        
  req_6_4:
    name: "Protect against common vulnerabilities"
    pipeline_controls:
      - name: "Dependency Scanning"
        implementation: "Snyk with CVSS threshold 7.0"
        verification: "No HIGH/CRITICAL vulnerabilities"
        
      - name: "Container Scanning"
        implementation: "Trivy scanning before push"
        verification: "No exploitable vulnerabilities"
        
  req_8_3:
    name: "Secure authentication"
    pipeline_controls:
      - name: "No hardcoded credentials"
        implementation: "GitLeaks + custom patterns"
        verification: "Zero findings"
        failure_action: "Block commit"
        
      - name: "Secrets management"
        implementation: "AWS Secrets Manager + Vault"
        verification: "Audit trail for all secret access"
        
  req_10:
    name: "Log and monitor all access"
    pipeline_controls:
      - name: "Pipeline audit logs"
        implementation: "GitHub Actions audit log retention 90 days"
        verification: "All actions logged with user + timestamp"
        
      - name: "Deployment records"
        implementation: "ArgoCD deployment history"
        verification: "Who deployed what, when"
```

---

## 93.8 Disaster Recovery และ Rollback

### Automated Rollback

```python
# rollback_manager.py
# ระบบ automated rollback เมื่อ deployment มีปัญหา

import boto3
import subprocess
from datetime import datetime

class RollbackManager:
    def __init__(self, service_name: str, namespace: str = "production"):
        self.service_name = service_name
        self.namespace = namespace
        self.argocd_server = "https://argocd.shopthai.internal"
        
    def check_health(self) -> dict:
        """ตรวจสอบ health metrics หลัง deployment"""
        metrics = self.fetch_prometheus_metrics()
        
        issues = []
        
        if metrics['error_rate'] > 1.0:
            issues.append(f"Error rate สูง: {metrics['error_rate']:.2f}%")
        
        if metrics['p95_response_time_ms'] > 1000:
            issues.append(f"Response time สูง: {metrics['p95_response_time_ms']:.0f}ms")
        
        if metrics['pod_availability'] < 0.8:
            issues.append(f"Pod availability ต่ำ: {metrics['pod_availability']:.0%}")
        
        return {
            'healthy': len(issues) == 0,
            'issues': issues,
            'metrics': metrics
        }
    
    def get_previous_stable_version(self) -> str:
        """หา version ก่อนหน้าที่ stable"""
        result = subprocess.run([
            'kubectl', 'rollout', 'history',
            f'deployment/{self.service_name}',
            '-n', self.namespace,
            '--output=json'
        ], capture_output=True, text=True)
        
        history = result.stdout
        # Parse และ return previous revision
        return self.parse_previous_revision(history)
    
    def rollback(self, reason: str = "") -> bool:
        """ทำการ rollback"""
        print(f"🔄 Initiating rollback for {self.service_name}")
        print(f"   Reason: {reason}")
        
        try:
            # Rollback ด้วย kubectl
            result = subprocess.run([
                'kubectl', 'rollout', 'undo',
                f'deployment/{self.service_name}',
                '-n', self.namespace
            ], capture_output=True, text=True, check=True)
            
            # รอให้ rollback สำเร็จ
            subprocess.run([
                'kubectl', 'rollout', 'status',
                f'deployment/{self.service_name}',
                '-n', self.namespace,
                '--timeout=120s'
            ], check=True)
            
            # Record rollback event
            self.record_rollback_event(reason)
            
            # Send notification
            self.send_rollback_notification(reason)
            
            print(f"✅ Rollback completed for {self.service_name}")
            return True
            
        except subprocess.CalledProcessError as e:
            print(f"❌ Rollback failed: {e}")
            self.escalate_incident()
            return False
    
    def auto_rollback_if_unhealthy(self, check_duration_minutes: int = 10):
        """ตรวจสอบ health และ auto-rollback ถ้าจำเป็น"""
        import time
        
        print(f"🔍 Monitoring {self.service_name} for {check_duration_minutes} minutes...")
        
        start_time = time.time()
        check_interval = 60  # ตรวจทุก 1 นาที
        
        while (time.time() - start_time) < (check_duration_minutes * 60):
            health = self.check_health()
            
            if not health['healthy']:
                print(f"⚠️ Health check failed:")
                for issue in health['issues']:
                    print(f"   - {issue}")
                
                self.rollback(reason=f"Automated rollback: {'; '.join(health['issues'])}")
                return False
            
            print(f"✅ Health check passed at {datetime.now().strftime('%H:%M:%S')}")
            time.sleep(check_interval)
        
        print(f"✅ {self.service_name} remained healthy for {check_duration_minutes} minutes")
        return True
    
    def fetch_prometheus_metrics(self) -> dict:
        """ดึง metrics จาก Prometheus"""
        # Implementation จริงจะ query Prometheus API
        return {
            'error_rate': 0.5,
            'p95_response_time_ms': 250,
            'pod_availability': 1.0
        }
    
    def record_rollback_event(self, reason: str):
        """บันทึก rollback event สำหรับ audit"""
        pass
    
    def send_rollback_notification(self, reason: str):
        """ส่ง notification ไปยัง Slack"""
        pass
    
    def escalate_incident(self):
        """Escalate ไปยัง on-call engineer"""
        pass

# Usage
rollback_mgr = RollbackManager('order-service')
rollback_mgr.auto_rollback_if_unhealthy(check_duration_minutes=10)
```

---

## 93.9 Lessons Learned

### สิ่งที่ ShopThai เรียนรู้

```markdown
# Lessons Learned: ShopThai CI/CD Journey

## ✅ สิ่งที่ทำถูก

### 1. เริ่มจาก Pain Points จริง
- ฟัง developer feedback ก่อน design pipeline
- Quick wins เพื่อสร้าง momentum
- ผลลัพธ์: Developer adoption rate สูง

### 2. Feature Flags ก่อน Automated Deployment
- Deploy code ก่อน แต่ enable ด้วย flag
- ลด risk ของ deployment อย่างมาก
- ผลลัพธ์: Zero deployment-related incidents ใน 6 เดือน

### 3. Invest in Test Infrastructure
- ลงทุนใน staging environment ที่ resemble production
- Test data management strategy ชัดเจน
- ผลลัพธ์: 85% bug detection ก่อน production

### 4. GitOps จาก Day 1
- ArgoCD เป็น single source of truth
- ทุก production change มี PR
- ผลลัพธ์: Full audit trail, rollback ง่าย

## ❌ สิ่งที่ควรทำต่างออกไป

### 1. Test Coverage ควรเริ่มทำตั้งแต่ต้น
- ปัจจุบัน legacycode บางส่วนยาก test
- ถ้าเริ่มใหม่: TDD ตั้งแต่แรก
- บทเรียน: Technical debt ใน testing แพงมาก

### 2. หลีกเลี่ยง Pipeline-as-a-Snowflake
- ช่วงแรกแต่ละทีมสร้าง pipeline เอง
- ต้องใช้เวลา 3 เดือนมาตรฐาน
- บทเรียน: Standard templates ตั้งแต่แรก

### 3. Performance Testing ควรอยู่ใน Pipeline ตั้งแต่แรก
- เพิ่มเข้ามาทีหลัง ทำให้ยากขึ้น
- บทเรียน: ใส่ performance test ใน pipeline ตั้งแต่วันแรก

### 4. Secret Rotation Strategy
- ควรวางแผน secret rotation ตั้งแต่ต้น
- ทำทีหลังยากมาก
- บทเรียน: Day 1 secret management

## 📊 ผลลัพธ์หลัง 12 เดือน

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Deployment Frequency | 1/month | 5/day | +15,000% |
| Lead Time | 4 weeks | 2 hours | -99% |
| MTTR | 8 hours | 20 minutes | -96% |
| Change Failure Rate | 35% | 3% | -91% |
| Developer Satisfaction | 3.2/10 | 8.5/10 | +166% |
```

---

## 93.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Pipeline Design

**โจทย์:** ออกแบบ CI/CD pipeline สำหรับ "Product Service" ของ ShopThai

**Requirements:**
- Node.js 18, TypeScript
- PostgreSQL + Redis dependencies
- ต้องผ่าน 70% test coverage
- Container scanning required (PCI-DSS)
- Blue/green deployment to production

**Template:**
```yaml
# .github/workflows/product-service.yml
name: Product Service Pipeline

on:
  push:
    branches: [main, 'feature/**']
  pull_request:
    branches: [main]

jobs:
  # TODO: เพิ่ม jobs ทั้งหมด
  # 1. Code quality (lint + SonarQube)
  # 2. Security scan (secrets + dependencies)
  # 3. Tests (unit + integration)
  # 4. Build Docker image
  # 5. Container security scan
  # 6. Deploy to dev
  # 7. Deploy to staging (with performance test)
  # 8. Deploy to production (with blue/green)
  
  placeholder:
    runs-on: ubuntu-latest
    steps:
      - run: echo "TODO: implement pipeline"
```

### แบบฝึกหัดที่ 2: Rollback Strategy

**โจทย์:** เขียน rollback automation script

```python
# TODO: implement rollback_manager.py
# Requirements:
# 1. ตรวจสอบ error rate จาก Prometheus (mock data ก็ได้)
# 2. ถ้า error rate > 2% ใน 5 นาทีแรก ให้ auto-rollback
# 3. บันทึก rollback event
# 4. ส่ง Slack notification (mock)

class RollbackManager:
    def __init__(self, service_name: str):
        self.service_name = service_name
    
    def check_error_rate(self) -> float:
        # TODO: implement (ใช้ mock data)
        pass
    
    def rollback(self) -> bool:
        # TODO: implement
        pass
    
    def monitor_and_rollback(self, duration_minutes: int = 5):
        # TODO: implement monitoring loop
        pass
```

### แบบฝึกหัดที่ 3: Performance Test

**โจทย์:** เขียน k6 performance test สำหรับ Cart API

```javascript
// tests/performance/cart-load-test.js
// TODO: implement performance test สำหรับ
// 1. Add item to cart
// 2. Update cart item quantity  
// 3. Remove item from cart
// 4. Get cart total

// Requirements:
// - Normal load: 50 concurrent users
// - Peak load: 500 concurrent users (mega sales)
// - Threshold: p95 < 300ms, error rate < 0.5%

import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  // TODO: define scenarios
};

export default function() {
  // TODO: implement test scenarios
}
```

---

## สรุป

ShopThai case study แสดงให้เห็น:

1. **ความสำคัญของ Quick Wins** - เริ่มจาก pain points จริง สร้าง momentum
2. **Feature Flags** - กุญแจสำคัญของ safe continuous deployment
3. **Blue/Green Deployment** - Zero downtime สำหรับ e-commerce
4. **Testing Strategy** - Test pyramid ที่ balance ระหว่าง speed กับ coverage
5. **Observability** - Monitor ทุก step ของ pipeline และ production
6. **GitOps** - Single source of truth สำหรับ infrastructure state
7. **PCI-DSS** - Security controls ที่ integrate ใน pipeline

---

**ต่อไป:** Part 94 - Case Study: FinTech CI/CD

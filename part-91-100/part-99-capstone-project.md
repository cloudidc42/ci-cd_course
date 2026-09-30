# Part 99: Capstone Project — Full CI/CD Pipeline สำหรับ Microservices Application

## บทนำ

Capstone project นี้รวมทุกอย่างที่เรียนมาตลอดหลักสูตร คุณจะสร้าง production-ready CI/CD pipeline ครบถ้วนสำหรับ microservices application ชื่อ **"ShopMini"** — simple e-commerce platform ที่ประกอบด้วย 4 microservices

**สิ่งที่คุณจะสร้าง:**
- Git Workflow ที่ถูกต้อง
- Full CI/CD Pipeline สำหรับแต่ละ service
- Docker containerization
- Kubernetes deployment
- GitOps ด้วย ArgoCD
- Monitoring และ Observability
- Security scanning
- Automated testing

---

## 99.1 Project Overview

### ShopMini Architecture

```
ShopMini - E-commerce Microservices
═══════════════════════════════════════

Services:
┌─────────────────────────────────────────────┐
│                                             │
│  ┌──────────────┐    ┌──────────────────┐  │
│  │  Product     │    │  Order Service   │  │
│  │  Service     │    │  (Node.js)       │  │
│  │  (Node.js)   │    │                  │  │
│  └──────┬───────┘    └──────┬───────────┘  │
│         │                   │              │
│  ┌──────▼───────┐    ┌──────▼───────────┐  │
│  │  Inventory   │    │  Notification    │  │
│  │  Service     │    │  Service         │  │
│  │  (Python)    │    │  (Python)        │  │
│  └──────────────┘    └──────────────────┘  │
│                                             │
└─────────────────────────────────────────────┘

Databases:
- PostgreSQL (Product + Order data)
- Redis (Cache + Session)
- MongoDB (Product catalog)

Infrastructure:
- Kubernetes (EKS/GKE)
- GitHub Actions (CI)
- ArgoCD (CD/GitOps)
- Prometheus + Grafana (Monitoring)
```

### Repository Structure

```
shopmini/
├── services/
│   ├── product-service/        # Node.js
│   ├── order-service/          # Node.js
│   ├── inventory-service/      # Python
│   └── notification-service/   # Python
├── infrastructure/
│   ├── kubernetes/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── development/
│   │       ├── staging/
│   │       └── production/
│   ├── terraform/
│   └── helm/
├── .github/
│   └── workflows/
│       ├── product-service.yml
│       ├── order-service.yml
│       ├── inventory-service.yml
│       ├── notification-service.yml
│       └── infrastructure.yml
└── monitoring/
    ├── prometheus/
    ├── grafana/
    └── alerts/
```

---

## 99.2 Product Service — Node.js

### Application Code

```typescript
// services/product-service/src/index.ts
import express from 'express';
import { createClient } from 'redis';
import { Pool } from 'pg';
import { logger } from './logger';
import { productRouter } from './routes/products';
import { healthRouter } from './routes/health';
import { metricsRouter } from './routes/metrics';

const app = express();

// Middleware
app.use(express.json());

// Database connections
const pgPool = new Pool({
  connectionString: process.env.DATABASE_URL
});

const redisClient = createClient({
  url: process.env.REDIS_URL
});

// Routes
app.use('/health', healthRouter);
app.use('/metrics', metricsRouter);
app.use('/api/v1/products', productRouter(pgPool, redisClient));

// Error handling
app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
  logger.error('Unhandled error', { error: err.message, stack: err.stack });
  res.status(500).json({ error: 'Internal server error' });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  logger.info(`Product service started on port ${PORT}`);
});
```

```typescript
// services/product-service/src/routes/products.ts
import { Router, Request, Response } from 'express';
import { Pool } from 'pg';
import { createClient } from 'redis';
import { ProductRepository } from '../repositories/product';

export function productRouter(db: Pool, cache: any): Router {
  const router = Router();
  const repo = new ProductRepository(db, cache);
  
  // GET /api/v1/products
  router.get('/', async (req: Request, res: Response) => {
    try {
      const { category, page = '1', limit = '20' } = req.query;
      
      const products = await repo.findAll({
        category: category as string,
        page: parseInt(page as string),
        limit: parseInt(limit as string)
      });
      
      res.json({
        data: products.items,
        pagination: {
          page: products.page,
          total: products.total,
          pages: Math.ceil(products.total / products.limit)
        }
      });
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch products' });
    }
  });
  
  // GET /api/v1/products/:id
  router.get('/:id', async (req: Request, res: Response) => {
    try {
      const product = await repo.findById(req.params.id);
      
      if (!product) {
        return res.status(404).json({ error: 'Product not found' });
      }
      
      res.json(product);
    } catch (error) {
      res.status(500).json({ error: 'Failed to fetch product' });
    }
  });
  
  return router;
}
```

### Unit Tests

```typescript
// services/product-service/tests/unit/products.test.ts
import { ProductRepository } from '../../src/repositories/product';

describe('ProductRepository', () => {
  let repo: ProductRepository;
  let mockDb: any;
  let mockCache: any;
  
  beforeEach(() => {
    mockDb = {
      query: jest.fn()
    };
    
    mockCache = {
      get: jest.fn(),
      setEx: jest.fn()
    };
    
    repo = new ProductRepository(mockDb, mockCache);
  });
  
  describe('findById', () => {
    it('returns product from cache when available', async () => {
      const cachedProduct = { id: '1', name: 'Test Product', price: 100 };
      mockCache.get.mockResolvedValue(JSON.stringify(cachedProduct));
      
      const result = await repo.findById('1');
      
      expect(result).toEqual(cachedProduct);
      expect(mockDb.query).not.toHaveBeenCalled();
    });
    
    it('fetches from database when cache miss', async () => {
      mockCache.get.mockResolvedValue(null);
      mockDb.query.mockResolvedValue({
        rows: [{ id: '1', name: 'DB Product', price: 200 }]
      });
      
      const result = await repo.findById('1');
      
      expect(result).toEqual({ id: '1', name: 'DB Product', price: 200 });
      expect(mockCache.setEx).toHaveBeenCalled();
    });
    
    it('returns null when product not found', async () => {
      mockCache.get.mockResolvedValue(null);
      mockDb.query.mockResolvedValue({ rows: [] });
      
      const result = await repo.findById('nonexistent');
      
      expect(result).toBeNull();
    });
  });
  
  describe('findAll', () => {
    it('applies pagination correctly', async () => {
      mockDb.query.mockResolvedValueOnce({ rows: [{ count: '100' }] });
      mockDb.query.mockResolvedValueOnce({
        rows: Array(20).fill(null).map((_, i) => ({
          id: String(i + 1),
          name: `Product ${i + 1}`,
          price: 100 * (i + 1)
        }))
      });
      
      const result = await repo.findAll({ page: 1, limit: 20 });
      
      expect(result.items).toHaveLength(20);
      expect(result.total).toBe(100);
    });
  });
});
```

### Dockerfile

```dockerfile
# services/product-service/Dockerfile
# syntax=docker/dockerfile:1.6

# ===== Stage 1: Dependencies =====
FROM node:18-alpine AS deps
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install with cache mount
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# ===== Stage 2: Build =====
FROM node:18-alpine AS build
WORKDIR /app

COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

COPY . .
RUN npm run build

# Run tests in build stage
RUN npm test -- --passWithNoTests

# ===== Stage 3: Production =====
FROM node:18-alpine AS production

# Security: Non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 --ingroup nodejs nodejs

WORKDIR /app

# Copy production files
COPY --from=deps --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package.json ./

USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### CI/CD Pipeline

```yaml
# .github/workflows/product-service.yml

name: Product Service Pipeline

on:
  push:
    branches: [main, develop]
    paths:
      - 'services/product-service/**'
      - '.github/workflows/product-service.yml'
  pull_request:
    branches: [main, develop]
    paths:
      - 'services/product-service/**'

defaults:
  run:
    working-directory: services/product-service

env:
  SERVICE_NAME: product-service
  ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
  NODE_VERSION: '18'

jobs:
  # ===== CODE QUALITY =====
  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: services/product-service/package-lock.json
      
      - name: Install dependencies
        run: npm ci
      
      - name: TypeScript type check
        run: npx tsc --noEmit
      
      - name: ESLint
        run: npm run lint -- --format=@microsoft/eslint-formatter-sarif --output-file ../../eslint-results.sarif
        continue-on-error: true
      
      - name: Upload ESLint SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: eslint-results.sarif

  # ===== SECURITY =====
  security:
    name: Security Scanning
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: GitLeaks - Secret Detection
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Dependency Audit
        run: |
          npm audit --audit-level=high
          
      - name: Snyk Dependency Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
        continue-on-error: true

  # ===== TESTS =====
  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: code-quality
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: shopmini_test
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass123
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: services/product-service/package-lock.json
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run database migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://testuser:testpass123@localhost:5432/shopmini_test
      
      - name: Run unit tests
        run: |
          npm run test:unit -- \
            --coverage \
            --coverageReporters=lcov,text,cobertura \
            --forceExit
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://testuser:testpass123@localhost:5432/shopmini_test
          REDIS_URL: redis://localhost:6379
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://testuser:testpass123@localhost:5432/shopmini_test
          REDIS_URL: redis://localhost:6379
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: services/product-service/coverage/lcov.info
          flags: product-service
      
      - name: Publish test results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Product Service Tests
          path: services/product-service/test-results/*.xml
          reporter: jest-junit

  # ===== BUILD =====
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [security, test]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
      
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.ECR_REGISTRY }}/shopmini/${{ env.SERVICE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: services/product-service
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha,scope=${{ env.SERVICE_NAME }}
          cache-to: type=gha,mode=max,scope=${{ env.SERVICE_NAME }}
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            GIT_COMMIT=${{ github.sha }}
      
      - name: Container security scan
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest \
            image \
            --exit-code 1 \
            --severity HIGH,CRITICAL \
            --no-progress \
            ${{ env.ECR_REGISTRY }}/shopmini/${{ env.SERVICE_NAME }}:sha-${{ github.sha }}
      
      - name: Generate SBOM
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest \
            image \
            --format cyclonedx \
            --output sbom.json \
            ${{ env.ECR_REGISTRY }}/shopmini/${{ env.SERVICE_NAME }}:sha-${{ github.sha }}
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom-${{ env.SERVICE_NAME }}
          path: sbom.json

  # ===== DEPLOY DEVELOPMENT =====
  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: development
      url: https://dev.shopmini.internal/api/products
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Update GitOps repo
        run: |
          git clone https://x-access-token:${{ secrets.GITOPS_TOKEN }}@github.com/company/shopmini-k8s.git
          cd shopmini-k8s
          
          yq e ".spec.template.spec.containers[0].image = \"${{ env.ECR_REGISTRY }}/shopmini/${{ env.SERVICE_NAME }}:sha-${{ github.sha }}\"" \
            -i "environments/development/services/${{ env.SERVICE_NAME }}/deployment.yaml"
          
          git config user.email "cicd@shopmini.com"
          git config user.name "ShopMini CI/CD"
          git add .
          git commit -m "chore(dev): deploy ${{ env.SERVICE_NAME }}@${{ github.sha }}"
          git push

  # ===== DEPLOY STAGING =====
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.shopmini.com/api/products
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Update staging in GitOps
        run: |
          git clone https://x-access-token:${{ secrets.GITOPS_TOKEN }}@github.com/company/shopmini-k8s.git
          cd shopmini-k8s
          
          yq e ".spec.template.spec.containers[0].image = \"${{ env.ECR_REGISTRY }}/shopmini/${{ env.SERVICE_NAME }}:sha-${{ github.sha }}\"" \
            -i "environments/staging/services/${{ env.SERVICE_NAME }}/deployment.yaml"
          
          git config user.email "cicd@shopmini.com"
          git config user.name "ShopMini CI/CD"
          git add .
          git commit -m "chore(staging): deploy ${{ env.SERVICE_NAME }}@${{ github.sha }}"
          git push
      
      - name: Run smoke tests on staging
        run: |
          sleep 60  # รอให้ deployment propagate
          curl -sf https://staging.shopmini.com/api/v1/products/health
          curl -sf https://staging.shopmini.com/api/v1/products?limit=1

  # ===== DEPLOY PRODUCTION =====
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://www.shopmini.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Production (with manual approval)
        run: |
          git clone https://x-access-token:${{ secrets.GITOPS_TOKEN }}@github.com/company/shopmini-k8s.git
          cd shopmini-k8s
          
          yq e ".spec.template.spec.containers[0].image = \"${{ env.ECR_REGISTRY }}/shopmini/${{ env.SERVICE_NAME }}:sha-${{ github.sha }}\"" \
            -i "environments/production/services/${{ env.SERVICE_NAME }}/deployment.yaml"
          
          git config user.email "cicd@shopmini.com"
          git config user.name "ShopMini CI/CD"
          git add .
          git commit -m "chore(prod): deploy ${{ env.SERVICE_NAME }}@${{ github.sha }}"
          git push
      
      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: "🚀 ${{ env.SERVICE_NAME }} deployed to production!\nCommit: ${{ github.sha }}"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 99.3 Kubernetes Configuration

### Base Configuration

```yaml
# infrastructure/kubernetes/base/product-service/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  labels:
    app: product-service
    version: "1.0"

spec:
  replicas: 2
  selector:
    matchLabels:
      app: product-service
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  template:
    metadata:
      labels:
        app: product-service
        version: "1.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "3000"
    
    spec:
      # Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      
      # Service Account
      serviceAccountName: product-service
      
      containers:
        - name: product-service
          image: registry.company.com/shopmini/product-service:latest
          imagePullPolicy: IfNotPresent
          
          ports:
            - name: http
              containerPort: 3000
          
          # Environment variables from secrets
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: product-service-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: product-service-secrets
                  key: redis-url
          
          # Resource limits
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          
          # Health checks
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          
          # Startup probe สำหรับ slow starts
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            failureThreshold: 30
            periodSeconds: 10
          
          # Security
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: [ALL]
          
          # Writable tmp volume
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: tmp
          emptyDir: {}
      
      # Pod topology
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: product-service

---
apiVersion: v1
kind: Service
metadata:
  name: product-service
spec:
  selector:
    app: product-service
  ports:
    - name: http
      port: 80
      targetPort: http
  type: ClusterIP

---
# HPA - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Kustomize Overlays

```yaml
# infrastructure/kubernetes/overlays/production/kustomization.yaml

apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

bases:
  - ../../base

patchesStrategicMerge:
  - product-service-patch.yaml

# Production: 3 replicas minimum
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: product-service
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 3
      - op: replace
        path: /spec/template/spec/containers/0/resources/requests/memory
        value: "256Mi"
      - op: replace
        path: /spec/template/spec/containers/0/resources/limits/memory
        value: "512Mi"
```

---

## 99.4 GitOps Setup

### ArgoCD Application

```yaml
# argocd/applications/shopmini.yaml
# App of Apps pattern

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shopmini
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io

spec:
  project: shopmini
  
  source:
    repoURL: https://github.com/company/shopmini-k8s.git
    targetRevision: HEAD
    path: argocd/environments/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

---
# Individual service application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: product-service
  namespace: argocd

spec:
  project: shopmini
  
  source:
    repoURL: https://github.com/company/shopmini-k8s.git
    targetRevision: HEAD
    path: environments/production/services/product-service
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # HPA manages this
```

---

## 99.5 Monitoring Setup

### Prometheus Rules

```yaml
# monitoring/prometheus/rules/shopmini-alerts.yaml

groups:
  - name: shopmini-sla
    interval: 30s
    rules:
      # Product Service Error Rate
      - alert: ProductServiceHighErrorRate
        expr: |
          sum(rate(http_requests_total{service="product-service", status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="product-service"}[5m]))
          > 0.01
        for: 5m
        labels:
          severity: critical
          service: product-service
          team: backend
        annotations:
          summary: "Product Service error rate สูงกว่า 1%"
          description: "Error rate = {{ $value | humanizePercentage }}"
          dashboard: "https://grafana.shopmini.com/d/product-service"
          runbook: "https://runbooks.shopmini.com/product-service-errors"
      
      # Deployment Success
      - alert: DeploymentNotReady
        expr: |
          kube_deployment_status_replicas_unavailable{namespace="production"} > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.deployment }} มี unavailable replicas"
          description: "{{ $value }} replicas unavailable"
```

### Grafana Dashboard

```json
{
  "title": "ShopMini - Product Service",
  "uid": "product-service",
  "tags": ["shopmini", "product"],
  "panels": [
    {
      "title": "Request Rate",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
      "targets": [{
        "expr": "sum(rate(http_requests_total{service=\"product-service\"}[5m]))",
        "legendFormat": "req/s"
      }]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
      "targets": [{
        "expr": "sum(rate(http_requests_total{service=\"product-service\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{service=\"product-service\"}[5m])) * 100",
        "legendFormat": "error %"
      }],
      "fieldConfig": {
        "thresholds": {
          "steps": [
            {"value": 0, "color": "green"},
            {"value": 0.5, "color": "yellow"},
            {"value": 1, "color": "red"}
          ]
        }
      }
    },
    {
      "title": "Response Time P95",
      "type": "graph",
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4},
      "targets": [{
        "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{service=\"product-service\"}[5m])) by (le)) * 1000",
        "legendFormat": "p95 (ms)"
      }]
    }
  ]
}
```

---

## 99.6 Security Scanning

### GitHub Actions Security Workflow

```yaml
# .github/workflows/security-scan.yml
# Comprehensive security scan สำหรับทุก service

name: Security Scan

on:
  schedule:
    - cron: '0 2 * * 1'  # ทุกวันจันทร์ 09:00 ICT
  workflow_dispatch:

jobs:
  dependency-audit:
    name: Dependency Audit
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [product-service, order-service, inventory-service, notification-service]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Audit ${{ matrix.service }}
        working-directory: services/${{ matrix.service }}
        run: |
          if [ -f "package.json" ]; then
            npm audit --json > audit-report.json || true
            python3 ../../scripts/evaluate-npm-audit.py audit-report.json
          elif [ -f "requirements.txt" ]; then
            pip install safety
            safety check -r requirements.txt --json > safety-report.json || true
            python3 ../../scripts/evaluate-safety.py safety-report.json
          fi
      
      - name: Upload Audit Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: audit-${{ matrix.service }}
          path: services/${{ matrix.service }}/*-report.json

  container-scan:
    name: Container Security Scan
    runs-on: ubuntu-latest
    needs: dependency-audit
    
    steps:
      - name: Scan all service images
        run: |
          for service in product-service order-service inventory-service notification-service; do
            echo "Scanning ${service}..."
            trivy image \
              --format sarif \
              --output "trivy-${service}.sarif" \
              ${{ secrets.ECR_REGISTRY }}/shopmini/${service}:latest
          done
```

---

## 99.7 Project Deliverables

### สิ่งที่ต้องส่ง

```markdown
# Capstone Project Checklist

## Git Workflow ✅
☐ Repository สร้างแล้ว
☐ Branch protection rules configured
☐ .gitignore ครบถ้วน
☐ PR template configured
☐ CODEOWNERS กำหนดแล้ว

## CI Pipeline ✅
☐ Code quality checks (lint + type check)
☐ Secret detection (GitLeaks)
☐ Dependency scanning
☐ Unit tests (>70% coverage)
☐ Integration tests
☐ Docker build + push to registry
☐ Container security scan (Trivy)

## CD Pipeline ✅
☐ Deploy to Development (automated)
☐ Deploy to Staging (automated, after dev)
☐ Deploy to Production (manual approval)
☐ Smoke tests after each deployment
☐ Rollback mechanism ทำงานได้

## Kubernetes ✅
☐ Deployment manifests ครบถ้วน
☐ Service manifests
☐ HPA configured
☐ Resource limits กำหนด
☐ Non-root security context
☐ Liveness + Readiness probes
☐ Pod disruption budget

## GitOps ✅
☐ GitOps repository แยกต่างหาก
☐ ArgoCD configured
☐ Auto-sync enabled
☐ Drift detection working

## Monitoring ✅
☐ Prometheus metrics exposed
☐ Grafana dashboards created
☐ Alert rules configured
☐ Runbook links ใน alerts

## Security ✅
☐ SAST configured
☐ Secrets ไม่มีใน code
☐ Container scanning
☐ SBOM generated
☐ Audit logging

## Documentation ✅
☐ README ครบถ้วน
☐ Architecture diagram
☐ Runbook สำหรับ common scenarios
☐ Development guide สำหรับ new team members
```

---

## 99.8 Grading Rubric

```markdown
# Capstone Project Grading (100 คะแนน)

## CI Pipeline (30 คะแนน)
- Basic pipeline (build + test): 10 คะแนน
- Security scanning: 10 คะแนน
- Code quality + coverage: 10 คะแนน

## CD Pipeline (25 คะแนน)
- Multi-environment deployment: 10 คะแนน
- Zero-downtime deployment: 5 คะแนน
- Rollback capability: 10 คะแนน

## Kubernetes (20 คะแนน)
- Correct manifests: 10 คะแนน
- Security best practices: 5 คะแนน
- Auto-scaling: 5 คะแนน

## Monitoring (15 คะแนน)
- Metrics and dashboards: 8 คะแนน
- Alert rules: 7 คะแนน

## GitOps (10 คะแนน)
- ArgoCD setup: 5 คะแนน
- Separate GitOps repo: 5 คะแนน

## Bonus (10 คะแนน extra)
- AI integration ใน pipeline: 5 คะแนน
- Performance testing: 3 คะแนน
- Supply chain security (SBOM + signing): 2 คะแนน

เกณฑ์ผ่าน: 70 คะแนน (70%)
เกรด A: 90+ คะแนน
เกรด B: 80-89 คะแนน
เกรด C: 70-79 คะแนน
```

---

## สรุป

Capstone project นี้รวมเนื้อหาทั้งหมดจากหลักสูตร:

1. **Git Workflow** — Feature branches, PR reviews, branch protection
2. **CI/CD Pipeline** — Full automated pipeline ด้วย GitHub Actions
3. **Docker** — Multi-stage builds, security best practices
4. **Kubernetes** — Deployment, HPA, security contexts
5. **GitOps** — ArgoCD, separate config repository
6. **Monitoring** — Prometheus, Grafana, alerting
7. **Security** — Scanning at every stage, SBOM
8. **Testing** — Unit, integration, smoke tests

การทำ project นี้ให้สำเร็จหมายความว่าคุณพร้อมที่จะทำงาน CI/CD ใน environment จริง!

---

**ต่อไป:** Part 100 - สอบและรับใบรับรอง

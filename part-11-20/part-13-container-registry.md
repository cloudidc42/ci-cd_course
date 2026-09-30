# Part 13: Container Registry & Image Management

## สารบัญ

1. [Container Registry คืออะไร?](#container-registry-คืออะไร)
2. [Docker Hub](#docker-hub)
3. [GitHub Container Registry (GHCR)](#github-container-registry-ghcr)
4. [AWS Elastic Container Registry (ECR)](#aws-elastic-container-registry-ecr)
5. [Google Artifact Registry](#google-artifact-registry)
6. [Image Tagging Strategies](#image-tagging-strategies)
7. [Semantic Versioning](#semantic-versioning)
8. [latest Tag Pitfalls](#latest-tag-pitfalls)
9. [Push และ Pull จาก Registry](#push-และ-pull-จาก-registry)
10. [Private Registries](#private-registries)
11. [Image Scanning ด้วย Trivy](#image-scanning-ด้วย-trivy)
12. [Image Size Optimization](#image-size-optimization)
13. [Multi-arch Builds ด้วย buildx](#multi-arch-builds-ด้วย-buildx)
14. [GitHub Actions Workflow](#github-actions-workflow)
15. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Container Registry คืออะไร?

Container Registry คือ **repository สำหรับเก็บ Docker images** เปรียบได้กับ GitHub/GitLab แต่สำหรับ Docker images

### Workflow พื้นฐาน

```
Developer                Registry              Server
    │                        │                    │
    ├─── docker build ───┐   │                    │
    │                    │   │                    │
    ├─── docker tag  ───►│   │                    │
    │                    │   │                    │
    └─── docker push ───►│   │                    │
                         │   │                    │
                         └───┼──► docker pull ───►│
                             │                    │
                             │                 docker run
```

### ประเภทของ Container Registry

| Registry | Provider | ราคา | ความเป็นส่วนตัว |
|----------|----------|------|-----------------|
| Docker Hub | Docker | Free tier / Paid | Public/Private |
| GHCR | GitHub | Free (public) / ขึ้นอยู่กับ storage | Public/Private |
| ECR | AWS | ขึ้นอยู่กับ storage/transfer | Private |
| Artifact Registry | Google | ขึ้นอยู่กับ storage/transfer | Private |
| Azure Container Registry | Microsoft | ขึ้นอยู่กับ tier | Private |
| Quay.io | Red Hat | Free tier / Paid | Public/Private |
| Harbor | Open Source | Free (self-hosted) | Public/Private |

---

## Docker Hub

Docker Hub คือ container registry สาธารณะที่ใหญ่ที่สุด ใช้ได้ฟรีสำหรับ public images

### สร้าง Account และ Login

```bash
# Login ผ่าน CLI
docker login
# Username: yourusername
# Password: yourpassword

# Login ด้วย token (แนะนำกว่า password)
# สร้าง Access Token ที่ https://hub.docker.com/settings/security
echo "YOUR_ACCESS_TOKEN" | docker login --username yourusername --password-stdin

# ตรวจสอบ credentials
cat ~/.docker/config.json
```

### Push Image ไปยัง Docker Hub

```bash
# Tag image ด้วย Docker Hub username
docker tag myapp:latest yourusername/myapp:latest
docker tag myapp:1.0.0 yourusername/myapp:1.0.0

# Push
docker push yourusername/myapp:latest
docker push yourusername/myapp:1.0.0

# Push หลาย tags พร้อมกัน
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t yourusername/myapp:latest \
    -t yourusername/myapp:1.0.0 \
    --push \
    .
```

### Pull Image จาก Docker Hub

```bash
# Pull public image
docker pull nginx:latest
docker pull ubuntu:22.04

# Pull private image (ต้อง login ก่อน)
docker pull yourusername/private-app:latest

# Pull specific digest (immutable)
docker pull nginx@sha256:abc123...
```

### Docker Hub Rate Limits (2024)

| Account Type | Pull Limit |
|-------------|------------|
| Anonymous | 100 pulls/6 hours (per IP) |
| Free | 200 pulls/6 hours (per user) |
| Pro | Unlimited |
| Team | Unlimited |

**แก้ปัญหา Rate Limit:**

```bash
# Login เพื่อเพิ่ม limit
docker login

# ใช้ mirror ใน Docker daemon config
# /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://mirror.gcr.io",
    "https://registry-1.docker.io"
  ]
}

sudo systemctl restart docker
```

### Docker Hub Automated Builds (deprecated) → GitHub Actions

```yaml
# .github/workflows/dockerhub.yml
name: Push to Docker Hub

on:
  push:
    tags:
      - 'v*'

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/myapp:${{ github.ref_name }}
```

---

## GitHub Container Registry (GHCR)

GHCR เป็น container registry ที่ integrate กับ GitHub ecosystem ได้ดีมาก

### ข้อดีของ GHCR

- Integrate กับ GitHub Actions ได้สะดวก
- ใช้ GitHub Personal Access Token หรือ GITHUB_TOKEN
- Free สำหรับ public repositories
- Visibility linked กับ repository

### Login ไปยัง GHCR

```bash
# ใช้ Personal Access Token (PAT)
# สร้าง PAT ที่ Settings > Developer settings > Personal access tokens
# ต้องการ permission: write:packages, read:packages, delete:packages

echo "YOUR_GITHUB_PAT" | docker login ghcr.io --username YOUR_USERNAME --password-stdin

# หรือตั้ง environment variable
export CR_PAT="YOUR_PAT"
echo $CR_PAT | docker login ghcr.io --username YOUR_USERNAME --password-stdin
```

### Push Image ไปยัง GHCR

```bash
# Format: ghcr.io/OWNER/IMAGE_NAME:TAG
# OWNER = GitHub username หรือ organization name

# Build และ tag
docker build -t ghcr.io/yourusername/myapp:latest .
docker build -t ghcr.io/yourusername/myapp:1.0.0 .

# Push
docker push ghcr.io/yourusername/myapp:latest
docker push ghcr.io/yourusername/myapp:1.0.0

# ตัวอย่าง organization
docker push ghcr.io/myorg/backend:v2.1.0
```

### GitHub Actions ด้วย GHCR

```yaml
# .github/workflows/ghcr.yml
name: Build and Push to GHCR

on:
  push:
    branches: [main]
    tags:
      - 'v*'
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### ตัวอย่าง Tags ที่ metadata-action สร้าง

```
# เมื่อ push ไปที่ main branch
ghcr.io/user/myapp:main
ghcr.io/user/myapp:sha-abc1234
ghcr.io/user/myapp:latest

# เมื่อ push tag v1.2.3
ghcr.io/user/myapp:1.2.3
ghcr.io/user/myapp:1.2
ghcr.io/user/myapp:1
ghcr.io/user/myapp:latest

# เมื่อ open pull request
ghcr.io/user/myapp:pr-42
```

### ตั้งค่า Visibility ของ Package

```bash
# ผ่าน GitHub API
curl -s \
  -X PATCH \
  -H "Authorization: Bearer YOUR_PAT" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/user/packages/container/myapp/visibility \
  -d '{"visibility":"public"}'

# หรือผ่าน GitHub UI:
# Repository > Packages > myapp > Package settings > Change visibility
```

---

## AWS Elastic Container Registry (ECR)

ECR เป็น managed container registry ของ AWS เหมาะสำหรับ workloads บน AWS

### ติดตั้งและตั้งค่า

```bash
# ติดตั้ง AWS CLI
# https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

# ตั้งค่า credentials
aws configure
# AWS Access Key ID: YOUR_KEY
# AWS Secret Access Key: YOUR_SECRET
# Default region name: ap-southeast-1
# Default output format: json

# ตรวจสอบ
aws sts get-caller-identity
```

### สร้าง ECR Repository

```bash
# สร้าง repository
aws ecr create-repository \
    --repository-name myapp \
    --region ap-southeast-1

# ผลลัพธ์
# {
#     "repository": {
#         "repositoryUri": "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/myapp",
#         ...
#     }
# }

# ดู repositories
aws ecr describe-repositories --region ap-southeast-1

# ตั้งค่า lifecycle policy (ลบ images เก่า)
aws ecr put-lifecycle-policy \
    --repository-name myapp \
    --lifecycle-policy-text file://lifecycle-policy.json \
    --region ap-southeast-1
```

**lifecycle-policy.json:**
```json
{
    "rules": [
        {
            "rulePriority": 1,
            "description": "Keep last 10 tagged images",
            "selection": {
                "tagStatus": "tagged",
                "tagPrefixList": ["v"],
                "countType": "imageCountMoreThan",
                "countNumber": 10
            },
            "action": {
                "type": "expire"
            }
        },
        {
            "rulePriority": 2,
            "description": "Remove untagged images older than 7 days",
            "selection": {
                "tagStatus": "untagged",
                "countType": "sinceImagePushed",
                "countUnit": "days",
                "countNumber": 7
            },
            "action": {
                "type": "expire"
            }
        }
    ]
}
```

### Login และ Push ไปยัง ECR

```bash
# Get login password และ pipe ไปยัง docker login
aws ecr get-login-password --region ap-southeast-1 | \
    docker login \
    --username AWS \
    --password-stdin \
    123456789.dkr.ecr.ap-southeast-1.amazonaws.com

# Build และ tag
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-southeast-1
ECR_REPO="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/myapp"

docker build -t myapp .
docker tag myapp:latest ${ECR_REPO}:latest
docker tag myapp:latest ${ECR_REPO}:1.0.0

# Push
docker push ${ECR_REPO}:latest
docker push ${ECR_REPO}:1.0.0
```

### ECR ใน GitHub Actions

```yaml
# .github/workflows/ecr.yml
name: Build and Push to ECR

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build, tag, and push image to Amazon ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker buildx build \
            --platform linux/amd64 \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
            -t $ECR_REGISTRY/$ECR_REPOSITORY:latest \
            --push \
            .
      
      - name: Deploy to ECS (optional)
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: ${{ github.sha }}
        run: |
          aws ecs update-service \
            --cluster my-cluster \
            --service my-service \
            --force-new-deployment
```

### ECR ด้วย OIDC (ไม่ต้องใช้ Access Keys)

```yaml
# .github/workflows/ecr-oidc.yml
name: Deploy to ECR (OIDC)

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: ap-southeast-1
      
      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/myapp:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/myapp:latest
```

---

## Google Artifact Registry

Google Artifact Registry (GAR) เป็น successor ของ Google Container Registry (GCR)

### ตั้งค่า Google Cloud

```bash
# ติดตั้ง gcloud CLI
# https://cloud.google.com/sdk/docs/install

# Login
gcloud auth login

# ตั้งค่า project
gcloud config set project YOUR_PROJECT_ID

# Enable Artifact Registry API
gcloud services enable artifactregistry.googleapis.com
```

### สร้าง Repository ใน Artifact Registry

```bash
# สร้าง Docker repository
gcloud artifacts repositories create myapp-repo \
    --repository-format=docker \
    --location=asia-southeast1 \
    --description="My application Docker images"

# ดู repositories
gcloud artifacts repositories list --location=asia-southeast1

# ตั้งค่า authentication
gcloud auth configure-docker asia-southeast1-docker.pkg.dev
```

### Push Image ไปยัง Artifact Registry

```bash
# Format: LOCATION-docker.pkg.dev/PROJECT_ID/REPOSITORY/IMAGE:TAG
PROJECT_ID=$(gcloud config get-value project)
REGION=asia-southeast1
REPO=myapp-repo
IMAGE=myapp

# Build และ tag
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:latest .
docker build -t ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:1.0.0 .

# Push
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:latest
docker push ${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}:1.0.0
```

### Artifact Registry ใน GitHub Actions

```yaml
# .github/workflows/gar.yml
name: Build and Push to Google Artifact Registry

on:
  push:
    branches: [main]

env:
  PROJECT_ID: my-gcp-project
  GAR_LOCATION: asia-southeast1
  REPOSITORY: myapp-repo
  IMAGE: myapp

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      id-token: write  # สำหรับ Workload Identity Federation
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-provider
          service_account: github-actions@${{ env.PROJECT_ID }}.iam.gserviceaccount.com
      
      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2
      
      - name: Configure Docker
        run: gcloud auth configure-docker ${{ env.GAR_LOCATION }}-docker.pkg.dev
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push
        env:
          IMAGE_URI: ${{ env.GAR_LOCATION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPOSITORY }}/${{ env.IMAGE }}
        run: |
          docker buildx build \
            --platform linux/amd64,linux/arm64 \
            -t ${IMAGE_URI}:${{ github.sha }} \
            -t ${IMAGE_URI}:latest \
            --push \
            .
```

---

## Image Tagging Strategies

การ tag images อย่างมีระบบช่วยให้จัดการ versions ได้ง่าย

### Tagging Strategies ที่นิยมใช้

#### 1. Semantic Versioning (แนะนำ)

```bash
# Full semantic version
docker tag myapp:latest myapp:1.2.3

# Major.Minor version
docker tag myapp:latest myapp:1.2

# Major version only
docker tag myapp:latest myapp:1

# Pre-release versions
docker tag myapp:latest myapp:1.2.3-beta.1
docker tag myapp:latest myapp:1.2.3-rc.1
docker tag myapp:latest myapp:1.2.3-alpha.1
```

#### 2. Git Commit SHA

```bash
# Full SHA
GIT_SHA=$(git rev-parse HEAD)
docker tag myapp:latest myapp:${GIT_SHA}

# Short SHA (7 characters)
GIT_SHA_SHORT=$(git rev-parse --short HEAD)
docker tag myapp:latest myapp:${GIT_SHA_SHORT}

# ตัวอย่าง
docker tag myapp:latest myapp:a1b2c3d
```

#### 3. Branch Name

```bash
# สำหรับ feature branches
BRANCH=$(git branch --show-current | sed 's/[^a-zA-Z0-9-]/-/g')
docker tag myapp:latest myapp:${BRANCH}

# ตัวอย่าง
docker tag myapp:latest myapp:feature-user-auth
docker tag myapp:latest myapp:main
docker tag myapp:latest myapp:develop
```

#### 4. Timestamp

```bash
# Unix timestamp
TIMESTAMP=$(date +%s)
docker tag myapp:latest myapp:${TIMESTAMP}

# Human-readable
DATE=$(date +%Y%m%d-%H%M%S)
docker tag myapp:latest myapp:${DATE}

# ตัวอย่าง
docker tag myapp:latest myapp:20240115-143022
```

#### 5. Combined Strategy (Best Practice)

```bash
# สร้างหลาย tags พร้อมกัน
VERSION="1.2.3"
GIT_SHA=$(git rev-parse --short HEAD)
BUILD_DATE=$(date -u +%Y%m%d)

docker buildx build \
    -t myapp:latest \
    -t myapp:${VERSION} \
    -t myapp:${VERSION}-${GIT_SHA} \
    -t myapp:${BUILD_DATE} \
    --push \
    .
```

### Tagging สำหรับ Multi-Environment

```bash
# Development
docker tag myapp:abc123 myapp:dev

# Staging
docker tag myapp:abc123 myapp:staging

# Production (ใช้ semver)
docker tag myapp:abc123 myapp:1.2.3
docker tag myapp:abc123 myapp:latest
```

---

## Semantic Versioning

Semantic Versioning (SemVer) คือ convention สำหรับการกำหนด version ในรูปแบบ `MAJOR.MINOR.PATCH`

### SemVer Format

```
v1.2.3-beta.1+build.123
│ │ │  │         │
│ │ │  │         └─ Build metadata (optional)
│ │ │  └─────────── Pre-release version (optional)
│ │ └────────────── PATCH: bug fixes
│ └──────────────── MINOR: new features (backward compatible)
└────────────────── MAJOR: breaking changes
```

### กฎของ SemVer

- **MAJOR** เพิ่มเมื่อมี breaking changes
- **MINOR** เพิ่มเมื่อเพิ่ม features ที่ backward compatible
- **PATCH** เพิ่มเมื่อแก้ bugs ที่ backward compatible

### ตัวอย่าง

```
1.0.0   - Initial release
1.0.1   - Bug fix
1.1.0   - New feature (backward compatible)
1.1.1   - Bug fix สำหรับ feature ใหม่
2.0.0   - Breaking change (เปลี่ยน API)
2.0.0-alpha.1   - Pre-release alpha
2.0.0-beta.1    - Pre-release beta
2.0.0-rc.1      - Release candidate
```

### Docker Tags ตาม SemVer

```bash
# สำหรับ version 1.2.3
docker buildx build \
    -t myapp:1.2.3 \       # exact version (immutable)
    -t myapp:1.2 \          # minor version (อัปเดตเมื่อ patch เปลี่ยน)
    -t myapp:1 \            # major version (อัปเดตเมื่อ minor เปลี่ยน)
    -t myapp:latest \       # latest stable
    --push \
    .
```

### Automated Versioning ด้วย Git Tags

```bash
# สร้าง Git tag
git tag -a v1.2.3 -m "Release version 1.2.3"
git push origin v1.2.3

# ดู tags
git tag -l
git tag -l "v*"

# ดู latest tag
git describe --tags --abbrev=0

# สร้าง Docker tag จาก Git tag
GIT_TAG=$(git describe --tags --abbrev=0)
VERSION=${GIT_TAG#v}  # ลบ "v" prefix

docker buildx build \
    -t myapp:${VERSION} \
    -t myapp:latest \
    --push \
    .
```

---

## latest Tag Pitfalls

`latest` tag เป็นหนึ่งในปัญหาที่พบบ่อยที่สุดใน container management

### ปัญหาของ latest Tag

```bash
# ❌ ปัญหา 1: ไม่รู้ว่า latest คือ version อะไร
docker pull myapp:latest
docker run myapp:latest  # version อะไร?

# ❌ ปัญหา 2: latest ใน environments ต่างกัน
# Dev environment: latest = v1.2.3
# Prod environment: latest = v1.1.0 (เพราะ pull ช้ากว่า)
# ผลลัพธ์: behavior ต่างกัน!

# ❌ ปัญหา 3: Rollback ยาก
docker pull myapp:latest  # ได้ version ใหม่ที่มี bug
# จะ rollback ไป version ไหน? ไม่รู้!

# ❌ ปัญหา 4: Cache ทำให้ไม่ได้ version ล่าสุด
docker run myapp:latest  # อาจใช้ cached image เก่า
```

### Best Practices สำหรับ latest

```bash
# ✅ ใช้ specific version ใน production
docker run myapp:1.2.3

# ✅ Pin version ใน docker-compose.yml
services:
  webapp:
    image: myapp:1.2.3  # ไม่ใช้ latest

# ✅ ถ้าต้องใช้ latest ใช้ --pull เสมอ
docker pull myapp:latest && docker run myapp:latest

# ✅ ใช้ digest แทน tag ใน production
docker pull myapp@sha256:abc123def456...
```

### Immutable Tags

```bash
# Digest คือ immutable identifier ของ image
docker pull nginx:alpine
docker inspect nginx:alpine | grep -A2 '"RepoDigests"'
# "sha256:abc123..."

# ใช้ digest เพื่อ immutability
docker run nginx@sha256:abc123def456789...

# ใน docker-compose.yml
services:
  nginx:
    image: nginx@sha256:abc123def456789...
```

### Tagging Policy ที่แนะนำ

```yaml
# Tagging matrix สำหรับ git events

# เมื่อ push ไปที่ feature branch
tags:
  - myapp:feature-user-auth-abc1234

# เมื่อ push ไปที่ develop branch
tags:
  - myapp:develop
  - myapp:develop-abc1234

# เมื่อ push ไปที่ main branch
tags:
  - myapp:main
  - myapp:main-abc1234
  - myapp:latest   # latest = main

# เมื่อ tag release v1.2.3
tags:
  - myapp:1.2.3         # immutable
  - myapp:1.2           # floating minor
  - myapp:1             # floating major
  - myapp:latest        # latest stable
```

---

## Push และ Pull จาก Registry

### Authentication Methods

```bash
# Method 1: docker login (interactive)
docker login registry.example.com
docker login -u username -p password registry.example.com

# Method 2: stdin (non-interactive, safer)
echo "password" | docker login -u username --password-stdin registry.example.com

# Method 3: credentials store (recommended)
# macOS: uses macOS Keychain
# Linux: install docker-credential-helpers
# Windows: uses Windows Credential Manager

# Method 4: Environment variable (CI/CD)
DOCKER_REGISTRY_TOKEN="token"
echo "${DOCKER_REGISTRY_TOKEN}" | docker login ghcr.io -u username --password-stdin
```

### Push Operations

```bash
# Basic push
docker push yourusername/myapp:1.0.0

# Push หลาย tags
docker push yourusername/myapp:1.0.0
docker push yourusername/myapp:1.0
docker push yourusername/myapp:1
docker push yourusername/myapp:latest

# Push ทุก tags ที่มีชื่อเดียวกัน
docker push --all-tags yourusername/myapp

# Push ด้วย digest verification
DIGEST=$(docker push yourusername/myapp:1.0.0 | grep "digest: sha256" | awk '{print $3}')
echo "Pushed: ${DIGEST}"
```

### Pull Operations

```bash
# Basic pull
docker pull yourusername/myapp:1.0.0

# Pull specific digest (immutable)
docker pull yourusername/myapp@sha256:abc123...

# Pull เสมอ (ไม่ใช้ cache)
docker pull yourusername/myapp:latest
docker run --pull=always yourusername/myapp:latest

# Pull แล้ว run
docker run --pull=always yourusername/myapp:1.0.0

# Pull options
docker pull --platform linux/amd64 yourusername/myapp:latest
docker pull --disable-content-trust yourusername/myapp:latest
```

---

## Private Registries

### Self-hosted Registry ด้วย Docker Registry

```bash
# รัน local registry
docker run -d \
    --name local-registry \
    -p 5000:5000 \
    --restart=always \
    -v registry_data:/var/lib/registry \
    registry:2

# Tag และ push ไปยัง local registry
docker tag myapp:latest localhost:5000/myapp:latest
docker push localhost:5000/myapp:latest

# Pull จาก local registry
docker pull localhost:5000/myapp:latest
```

### Self-hosted Registry พร้อม TLS

```bash
# สร้าง self-signed certificates
mkdir -p certs
openssl req \
    -newkey rsa:4096 \
    -nodes \
    -sha256 \
    -keyout certs/registry.key \
    -x509 \
    -days 365 \
    -out certs/registry.crt \
    -subj "/CN=registry.example.com"

# รัน registry ด้วย TLS
docker run -d \
    --name secure-registry \
    -p 443:443 \
    -v registry_data:/var/lib/registry \
    -v $(pwd)/certs:/certs:ro \
    -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key \
    registry:2
```

### Harbor - Enterprise Registry

```yaml
# docker-compose.yml สำหรับ Harbor (simplified)
# ดู https://github.com/goharbor/harbor สำหรับ full setup

version: "3.9"

services:
  harbor-core:
    image: goharbor/harbor-core:v2.10.0
    # ... configuration
  
  harbor-portal:
    image: goharbor/harbor-portal:v2.10.0
    # ... configuration
  
  nginx:
    image: goharbor/nginx-photon:v2.10.0
    ports:
      - "80:8080"
      - "443:8443"
```

### Nexus Repository Manager

```yaml
# docker-compose.yml สำหรับ Nexus
version: "3.9"

services:
  nexus:
    image: sonatype/nexus3:latest
    ports:
      - "8081:8081"    # Web UI
      - "8082:8082"    # Docker hosted
      - "8083:8083"    # Docker proxy
    volumes:
      - nexus_data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2703m -Xmx2703m
    restart: unless-stopped

volumes:
  nexus_data:
```

---

## Image Scanning ด้วย Trivy

Trivy คือ open-source vulnerability scanner สำหรับ container images

### ติดตั้ง Trivy

```bash
# Ubuntu/Debian
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
    sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb \
    $(lsb_release -sc) main" | \
    sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# macOS
brew install trivy

# Docker (ไม่ต้องติดตั้ง)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy:latest image nginx:alpine

# ตรวจสอบ
trivy --version
```

### Scan Docker Image

```bash
# Scan image พื้นฐาน
trivy image nginx:alpine

# Scan local image
trivy image myapp:latest

# แสดงเฉพาะ HIGH และ CRITICAL vulnerabilities
trivy image --severity HIGH,CRITICAL nginx:alpine

# Scan และ fail ถ้าพบ CRITICAL
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Scan ด้วย format ต่างๆ
trivy image --format json -o report.json myapp:latest
trivy image --format table myapp:latest
trivy image --format sarif -o report.sarif myapp:latest

# Scan image ที่ push ขึ้น registry แล้ว
trivy image ghcr.io/username/myapp:latest

# Scan ด้วย ignore file
trivy image --ignorefile .trivyignore myapp:latest
```

### .trivyignore File

```
# .trivyignore
# Ignore specific CVEs

# CVE ที่ยังไม่มี fix
CVE-2023-12345

# CVE ที่ acceptable ใน use case นี้
CVE-2023-67890

# Format: CVE_ID [expiration_date] [comment]
CVE-2023-11111 exp:2024-12-31 # Will fix by end of year
```

### Trivy ใน GitHub Actions

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # ทุกวันจันทร์ 6am

jobs:
  scan:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      security-events: write  # สำหรับ upload SARIF
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t myapp:test .
      
      - name: Scan image (table format)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:test'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
      
      - name: Scan image (SARIF for GitHub Security)
        uses: aquasecurity/trivy-action@master
        if: always()  # รันแม้ step ก่อนหน้า fail
        with:
          image-ref: 'myapp:test'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

### Scan Dockerfile (Misconfigurations)

```bash
# Scan Dockerfile สำหรับ best practices
trivy config .
trivy config --severity HIGH,CRITICAL .

# Scan specific file
trivy config Dockerfile

# Scan directory
trivy config ./k8s/

# Scan แบบ recursive
trivy config --recursive .
```

### ตัวอย่างผลลัพธ์ Trivy

```
myapp:latest (alpine 3.18.4)
================================
Total: 5 (HIGH: 3, CRITICAL: 2)

┌────────────────┬───────────────┬──────────┬───────────────────┬───────────────────┬────────────────────────────────────────────────────┐
│    Library     │ Vulnerability │ Severity │ Installed Version │    Fixed Version  │                        Title                        │
├────────────────┼───────────────┼──────────┼───────────────────┼───────────────────┼────────────────────────────────────────────────────┤
│ openssl        │ CVE-2023-5678 │ CRITICAL │ 3.1.3-r0          │ 3.1.4-r0          │ OpenSSL: Potential PKCS#1 buffer overflow           │
│ curl           │ CVE-2023-9876 │ HIGH     │ 8.3.0-r0          │ 8.4.0-r0          │ curl: HSTS long file name clears contents          │
└────────────────┴───────────────┴──────────┴───────────────────┴───────────────────┴────────────────────────────────────────────────────┘
```

---

## Image Size Optimization

### วิธีลดขนาด Image

#### 1. เลือก Base Image ที่เหมาะสม

```dockerfile
# ขนาดของ Node.js base images
FROM node:18        # ~900MB
FROM node:18-slim   # ~220MB
FROM node:18-alpine # ~170MB

# สำหรับ Go/Rust (compiled): ใช้ scratch หรือ distroless
FROM scratch        # 0MB (ต้อง static binary)
FROM gcr.io/distroless/static-debian12  # ~2MB
```

#### 2. Multi-Stage Builds

```dockerfile
# Build stage: ใหญ่ได้
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# Runtime stage: เล็กที่สุด
FROM node:18-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --only=production

CMD ["node", "dist/index.js"]
```

#### 3. ลบ Cache และ Temp Files

```dockerfile
# ❌ Cache ยังอยู่
RUN apt-get update && apt-get install -y curl

# ✅ ลบ cache ในคำสั่งเดียวกัน
RUN apt-get update && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*

# Alpine: ใช้ --no-cache
RUN apk add --no-cache curl

# npm: ใช้ ci แทน install
RUN npm ci --only=production && npm cache clean --force

# pip: ใช้ --no-cache-dir
RUN pip install --no-cache-dir -r requirements.txt
```

#### 4. .dockerignore

```
# .dockerignore
node_modules
.git
dist
coverage
*.log
.env
.env.*
docker-compose*
Dockerfile*
README.md
docs/
tests/
.github/
```

#### 5. Squash Layers (ใช้ BuildKit)

```bash
# Squash แบบ manual ด้วย --squash
docker build --squash -t myapp:optimized .

# หรือใช้ Docker export/import
docker create --name temp myapp:latest
docker export temp | docker import - myapp:flat
docker rm temp
```

#### 6. Distroless Images

```dockerfile
# Python distroless
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM gcr.io/distroless/python3-debian12 AS runtime
COPY --from=builder /root/.local /root/.local
COPY . /app
WORKDIR /app
CMD ["app.py"]
```

### วิธีวิเคราะห์ขนาด Image

```bash
# ดูขนาด image
docker images myapp

# ดู layers โดยละเอียด
docker history myapp:latest
docker history --no-trunc myapp:latest

# ใช้ dive (third-party tool)
# https://github.com/wagoodman/dive
dive myapp:latest

# ตรวจสอบด้วย docker buildx imagetools
docker buildx imagetools inspect myapp:latest
```

### ตัวอย่าง: Optimize Node.js Image

```dockerfile
# syntax=docker/dockerfile:1
# แบบ non-optimized: ~900MB
FROM node:18
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "src/index.js"]

# แบบ optimized: ~150MB
FROM node:18-alpine AS production
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY src/ ./src/
RUN chown -R appuser:appgroup /app
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s \
    CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "src/index.js"]
```

---

## Multi-arch Builds ด้วย buildx

### ทำไมต้อง Multi-arch?

- **Apple Silicon (M1/M2)** ใช้ ARM64 architecture
- **AWS Graviton** ใช้ ARM64 (ประหยัดค่าใช้จ่าย ~20%)
- **Raspberry Pi** ใช้ ARM32/ARM64
- **Standard servers** ใช้ AMD64

ถ้า build แค่ AMD64 จะไม่ทำงานบน ARM64 (หรือช้ามากเพราะต้อง emulate)

### ตั้งค่า Docker Buildx

```bash
# ตรวจสอบ buildx
docker buildx version

# ดู builders
docker buildx ls

# สร้าง builder ที่รองรับ multi-platform
docker buildx create \
    --name mybuilder \
    --driver docker-container \
    --use

# Bootstrap builder
docker buildx inspect --bootstrap

# ตรวจสอบ platforms ที่รองรับ
docker buildx inspect mybuilder | grep Platforms
# Platforms: linux/amd64, linux/arm64, linux/arm/v7, linux/arm/v6
```

### Build Multi-arch Image

```bash
# Build สำหรับ AMD64 และ ARM64
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t yourusername/myapp:latest \
    --push \
    .

# Build สำหรับ platforms หลายๆ
docker buildx build \
    --platform linux/amd64,linux/arm64,linux/arm/v7 \
    -t yourusername/myapp:1.0.0 \
    -t yourusername/myapp:latest \
    --push \
    .

# Build local (ใช้ --load แทน --push)
docker buildx build \
    --platform linux/amd64 \
    -t myapp:test \
    --load \
    .
```

### Multi-arch Dockerfile

```dockerfile
# syntax=docker/dockerfile:1

# ARG ที่ buildx inject ให้อัตโนมัติ
ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

FROM --platform=$BUILDPLATFORM golang:1.21-alpine AS builder

ARG TARGETOS
ARG TARGETARCH

WORKDIR /app
COPY . .

# Cross-compile สำหรับ target platform
RUN GOOS=${TARGETOS} GOARCH=${TARGETARCH} \
    go build -o /app/binary ./cmd/main.go

# Runtime image สำหรับ target platform
FROM --platform=$TARGETPLATFORM alpine:3.18

COPY --from=builder /app/binary /binary
ENTRYPOINT ["/binary"]
```

### QEMU สำหรับ ARM emulation

```bash
# ติดตั้ง QEMU emulators
docker run --privileged --rm \
    tonistiigi/binfmt \
    --install all

# หรือ
docker run --rm --privileged \
    multiarch/qemu-user-static \
    --reset -p yes

# ตรวจสอบ
ls /proc/sys/fs/binfmt_misc/
```

### GitHub Actions Multi-arch Build

```yaml
# .github/workflows/multi-arch.yml
name: Multi-arch Build

on:
  push:
    tags:
      - 'v*'

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up QEMU (for ARM emulation)
        uses: docker/setup-qemu-action@v3
        with:
          platforms: all
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          platforms: linux/amd64,linux/arm64,linux/arm/v7
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=raw,value=latest
      
      - name: Build and push multi-arch
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64,linux/arm/v7
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Inspect image
        run: |
          docker buildx imagetools inspect \
            ghcr.io/${{ github.repository }}:latest
```

### ตรวจสอบ Multi-arch Image

```bash
# ดู platforms ที่รองรับ
docker buildx imagetools inspect yourusername/myapp:latest

# ผลลัพธ์
# Name:      docker.io/yourusername/myapp:latest
# MediaType: application/vnd.docker.distribution.manifest.list.v2+json
# Digest:    sha256:abc123...
#
# Manifests:
#   Name:      docker.io/yourusername/myapp:latest@sha256:def456...
#   MediaType: application/vnd.docker.distribution.manifest.v2+json
#   Platform:  linux/amd64
#
#   Name:      docker.io/yourusername/myapp:latest@sha256:ghi789...
#   MediaType: application/vnd.docker.distribution.manifest.v2+json
#   Platform:  linux/arm64
```

---

## GitHub Actions Workflow

### Complete Build, Test, Push, and Deploy Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - develop
      - 'release/**'
      - 'feature/**'
  pull_request:
    branches:
      - main
      - develop
  release:
    types: [published]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ===== JOB 1: Lint and Test =====
  test:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Run unit tests
        run: npm test -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  # ===== JOB 2: Build Image =====
  build:
    runs-on: ubuntu-latest
    needs: test
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-tags: ${{ steps.meta.outputs.tags }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable={{is_default_branch}}
          labels: |
            org.opencontainers.image.title=My Application
            org.opencontainers.image.description=My awesome application
            org.opencontainers.image.vendor=MyCompany
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.repository.updated_at }}
            VCS_REF=${{ github.sha }}
  
  # ===== JOB 3: Security Scan =====
  security-scan:
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name != 'pull_request'
    
    permissions:
      contents: read
      security-events: write
      packages: read
    
    steps:
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Fail on CRITICAL vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          exit-code: '1'
          severity: 'CRITICAL'
  
  # ===== JOB 4: Deploy to Staging =====
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - name: Deploy to staging
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tags }} to staging"
          # kubectl, helm, ecs, atau deployment script
  
  # ===== JOB 5: Deploy to Production =====
  deploy-production:
    runs-on: ubuntu-latest
    needs: [build, security-scan]
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://example.com
    
    steps:
      - name: Deploy to production
        run: |
          echo "Deploying ${{ needs.build.outputs.image-tags }} to production"
          # Deploy commands
```

### Reusable Workflow สำหรับ Docker Build

```yaml
# .github/workflows/docker-build-reusable.yml
name: Reusable Docker Build

on:
  workflow_call:
    inputs:
      registry:
        required: true
        type: string
      image-name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: Dockerfile
      context:
        required: false
        type: string
        default: .
      platforms:
        required: false
        type: string
        default: linux/amd64,linux/arm64
    outputs:
      digest:
        description: Image digest
        value: ${{ jobs.build.outputs.digest }}
      tags:
        description: Image tags
        value: ${{ jobs.build.outputs.tags }}
    secrets:
      registry-token:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      digest: ${{ steps.build.outputs.digest }}
      tags: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/setup-qemu-action@v3
      
      - uses: docker/setup-buildx-action@v3
      
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ${{ inputs.registry }}
          username: ${{ github.actor }}
          password: ${{ secrets.registry-token }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ inputs.registry }}/${{ inputs.image-name }}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: ${{ inputs.context }}
          file: ${{ inputs.dockerfile }}
          platforms: ${{ inputs.platforms }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**การใช้ Reusable Workflow:**

```yaml
# .github/workflows/main.yml
name: Main CI/CD

on:
  push:
    branches: [main]

jobs:
  build-backend:
    uses: ./.github/workflows/docker-build-reusable.yml
    with:
      registry: ghcr.io
      image-name: ${{ github.repository }}/backend
      context: ./backend
    secrets:
      registry-token: ${{ secrets.GITHUB_TOKEN }}
  
  build-frontend:
    uses: ./.github/workflows/docker-build-reusable.yml
    with:
      registry: ghcr.io
      image-name: ${{ github.repository }}/frontend
      context: ./frontend
      dockerfile: ./frontend/Dockerfile.prod
    secrets:
      registry-token: ${{ secrets.GITHUB_TOKEN }}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Push Image ไปยัง GHCR

ทดสอบ push image ไปยัง GitHub Container Registry:

```bash
# 1. Login
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_USERNAME --password-stdin

# 2. Build image
docker build -t ghcr.io/YOUR_USERNAME/test-app:v1.0.0 .

# 3. Push
docker push ghcr.io/YOUR_USERNAME/test-app:v1.0.0
docker push ghcr.io/YOUR_USERNAME/test-app:latest

# 4. Pull และทดสอบ
docker pull ghcr.io/YOUR_USERNAME/test-app:v1.0.0
docker run --rm ghcr.io/YOUR_USERNAME/test-app:v1.0.0

# 5. ดู image ที่ https://github.com/YOUR_USERNAME?tab=packages
```

### แบบฝึกหัดที่ 2: Image Scanning

```bash
# 1. ติดตั้ง Trivy
# macOS: brew install trivy
# Linux: ดูวิธีข้างต้น

# 2. Scan public image
trivy image --severity HIGH,CRITICAL nginx:latest

# 3. Scan image ของเราเอง
trivy image myapp:latest

# 4. สร้าง report
trivy image --format json -o scan-report.json myapp:latest

# 5. Scan Dockerfile
trivy config Dockerfile

# 6. ลอง fix vulnerabilities
# - อัปเดต base image version
# - อัปเดต packages ใน Dockerfile
# docker build -t myapp:patched .
# trivy image --exit-code 1 --severity CRITICAL myapp:patched
```

### แบบฝึกหัดที่ 3: Multi-arch Build

```bash
# 1. ตั้งค่า buildx
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# 2. Build multi-arch image
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t YOUR_USERNAME/multiarch-test:latest \
    --push \
    .

# 3. ตรวจสอบ
docker buildx imagetools inspect YOUR_USERNAME/multiarch-test:latest

# 4. ทดสอบ pull บน platform ต่างๆ
docker pull --platform linux/arm64 YOUR_USERNAME/multiarch-test:latest
docker run --platform linux/arm64 YOUR_USERNAME/multiarch-test:latest uname -m
```

### แบบฝึกหัดที่ 4: Complete CI/CD Workflow

สร้าง GitHub Actions workflow ที่:
1. Build Docker image
2. Scan ด้วย Trivy
3. Push ไปยัง GHCR ถ้าผ่าน scan
4. Tag ตาม git events

```yaml
# .github/workflows/exercise.yml
# ให้เติม steps ต่อไปนี้:

name: Exercise - Build and Push

on:
  push:
    branches: [main]
    tags:
      - 'v*'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
      security-events: write
    
    steps:
      # TODO: 1. Checkout code
      
      # TODO: 2. Set up QEMU for multi-arch
      
      # TODO: 3. Set up Docker Buildx
      
      # TODO: 4. Login to GHCR
      
      # TODO: 5. Extract metadata for tags
      
      # TODO: 6. Build image (don't push yet)
      
      # TODO: 7. Scan with Trivy
      
      # TODO: 8. Push if scan passes
```

### เฉลย แบบฝึกหัดที่ 4

```yaml
# .github/workflows/exercise-solution.yml
name: Exercise Solution - Build and Push

on:
  push:
    branches: [main]
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
      security-events: write
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build image (local, for scanning)
        uses: docker/build-push-action@v5
        with:
          context: .
          load: true  # ไม่ push, แค่ load เข้า local docker
          tags: myapp:scan-target
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Scan with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:scan-target
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      - name: Upload SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Build and push multi-arch image
        if: success()  # รันแค่ถ้า scan ผ่าน
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## สรุป

ในบทนี้คุณได้เรียนรู้:

1. **Container Registry** - ประเภทและการเลือกใช้
2. **Docker Hub** - Public registry, rate limits, การ push/pull
3. **GHCR** - Integration กับ GitHub, GITHUB_TOKEN authentication
4. **AWS ECR** - Managed registry บน AWS, lifecycle policies, OIDC
5. **Google Artifact Registry** - GAR setup, Workload Identity
6. **Image Tagging** - Strategies ต่างๆ: SemVer, Git SHA, Branch
7. **latest Pitfalls** - ปัญหาและวิธีหลีกเลี่ยง
8. **Trivy** - Image scanning สำหรับ vulnerabilities
9. **Multi-arch** - buildx, QEMU, cross-compilation
10. **GitHub Actions** - Complete CI/CD workflow

### Tagging Cheat Sheet

```
ประเภท Event       → Tags ที่สร้าง
─────────────────────────────────────────────
push main          → latest, main, sha-abc123
push develop       → develop, sha-abc123
push feature/foo   → feature-foo, sha-abc123
tag v1.2.3        → 1.2.3, 1.2, 1, latest
PR #42             → pr-42 (ไม่ push ถ้า sensitive)
```

### Security Checklist

- [ ] Scan images ด้วย Trivy ก่อน push
- [ ] ใช้ specific version tags ใน production
- [ ] ใช้ non-root user ใน containers
- [ ] ตั้ง lifecycle policies เพื่อลบ images เก่า
- [ ] ใช้ OIDC แทน Access Keys ใน CI/CD
- [ ] Scan Dockerfile สำหรับ misconfigurations
- [ ] ใช้ multi-stage builds เพื่อลด attack surface

---

*จบ Part 13 - Container Registry & Image Management*

*บทถัดไป: Part 14: Kubernetes Fundamentals*

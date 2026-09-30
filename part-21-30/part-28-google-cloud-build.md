# Part 28: Cloud CI/CD — Google Cloud Build

## สารบัญ
1. [Cloud Build Overview](#cloud-build-overview)
2. [ติดตั้งและตั้งค่า gcloud CLI](#ติดตั้งและตั้งค่า-gcloud-cli)
3. [cloudbuild.yaml Syntax](#cloudbuildyaml-syntax)
4. [Build Steps และ Images](#build-steps-และ-images)
5. [Triggers](#triggers)
6. [Substitution Variables](#substitution-variables)
7. [Build Artifacts ใน Google Cloud Storage](#build-artifacts-ใน-google-cloud-storage)
8. [Deployment ไปยัง Cloud Run/GKE/App Engine](#deployment-ไปยัง-cloud-rungkeapp-engine)
9. [Build Notifications](#build-notifications)
10. [Cloud Build Private Pools](#cloud-build-private-pools)
11. [Integration กับ Container Registry และ Artifact Registry](#integration-กับ-container-registry-และ-artifact-registry)
12. [Build Security และ Secret Management](#build-security-และ-secret-management)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Cloud Build Overview

**Google Cloud Build** เป็น fully managed CI/CD platform ของ Google Cloud ที่รัน build ใน Docker containers บน Google infrastructure

### สิ่งที่ Cloud Build ทำได้

```
┌─────────────────────────────────────────────────────────────────┐
│                    Cloud Build Workflow                        │
│                                                                 │
│  Source Code     Trigger      Build Steps       Deploy        │
│  ┌─────────┐    ┌────────┐    ┌──────────┐    ┌──────────┐   │
│  │ GitHub  │───▶│ Push/  │───▶│ Step 1:  │───▶│Cloud Run │   │
│  │ GitLab  │    │ PR     │    │ Test     │    │ GKE      │   │
│  │ CSR     │    │ Manual │    │ Step 2:  │    │App Engine│   │
│  └─────────┘    │ Tag    │    │ Build    │    └──────────┘   │
│                 └────────┘    │ Step 3:  │                   │
│                               │ Push     │                   │
│                               └──────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

### ข้อดีของ Cloud Build

- **Serverless**: ไม่ต้องจัดการ infrastructure
- **Docker-native**: ทุก step รันใน Docker container
- **Parallel execution**: รัน steps พร้อมกันได้
- **Built-in integrations**: GKE, Cloud Run, App Engine, Artifact Registry
- **Free tier**: 120 build-minutes/day ฟรี
- **Pay-per-use**: จ่ายตามจริง
- **Security**: รัน build ใน isolated environment

### เปรียบเทียบกับ GitHub Actions

| Feature | Cloud Build | GitHub Actions |
|---------|-------------|----------------|
| Runner | Google-managed | GitHub-managed |
| Config | cloudbuild.yaml | .github/workflows/*.yaml |
| Free tier | 120 min/day | 2000 min/month |
| Private network | Cloud Build private pools | Self-hosted runners |
| Secret management | Secret Manager (native) | GitHub Secrets |
| Artifact storage | Cloud Storage (native) | GitHub Artifacts |
| Integration | GCP services native | Community marketplace |

---

## ติดตั้งและตั้งค่า gcloud CLI

### ติดตั้ง gcloud CLI

```bash
# Linux/macOS
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

# macOS ด้วย Homebrew
brew install google-cloud-sdk

# Windows
# ดาวน์โหลด installer จาก https://cloud.google.com/sdk/docs/install
```

### ตั้งค่า Project

```bash
# Login
gcloud auth login

# Set project
gcloud config set project my-project-id

# ดู config ปัจจุบัน
gcloud config list

# Enable Cloud Build API
gcloud services enable cloudbuild.googleapis.com

# Enable Container Registry API (เก่า)
gcloud services enable containerregistry.googleapis.com

# Enable Artifact Registry API (ใหม่ แนะนำ)
gcloud services enable artifactregistry.googleapis.com

# Enable Cloud Run API
gcloud services enable run.googleapis.com
```

### ตั้งค่า IAM สำหรับ Cloud Build

```bash
# ดู Cloud Build Service Account
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format='value(projectNumber)')
CLOUD_BUILD_SA="${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"

echo "Cloud Build SA: $CLOUD_BUILD_SA"

# เพิ่ม roles ที่จำเป็น
# Cloud Run Admin (deploy to Cloud Run)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member "serviceAccount:$CLOUD_BUILD_SA" \
  --role "roles/run.admin"

# Kubernetes Engine Developer (deploy to GKE)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member "serviceAccount:$CLOUD_BUILD_SA" \
  --role "roles/container.developer"

# Artifact Registry Writer (push images)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member "serviceAccount:$CLOUD_BUILD_SA" \
  --role "roles/artifactregistry.writer"

# Storage Object Admin (artifacts in GCS)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member "serviceAccount:$CLOUD_BUILD_SA" \
  --role "roles/storage.objectAdmin"

# Service Account User (สำหรับ deploy)
gcloud iam service-accounts add-iam-policy-binding \
  my-service-account@$PROJECT_ID.iam.gserviceaccount.com \
  --member "serviceAccount:$CLOUD_BUILD_SA" \
  --role "roles/iam.serviceAccountUser"
```

### รัน Build แรก

```bash
# รัน build ง่ายๆ
gcloud builds submit \
  --config cloudbuild.yaml \
  .

# รัน build โดยไม่มี config file (build Docker image)
gcloud builds submit \
  --tag gcr.io/$PROJECT_ID/my-app:v1.0 \
  .

# ดู build logs
gcloud builds log BUILD_ID

# List builds
gcloud builds list

# ยกเลิก build
gcloud builds cancel BUILD_ID
```

---

## cloudbuild.yaml Syntax

### โครงสร้างพื้นฐาน

```yaml
# cloudbuild.yaml
steps:
  # Step แรก
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/my-app', '.']
  
  # Step ที่สอง
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/my-app']

# Images ที่ต้องการ push ไปยัง registry
images:
  - 'gcr.io/$PROJECT_ID/my-app'

# Artifacts สำหรับ download
artifacts:
  objects:
    location: 'gs://my-bucket/artifacts'
    paths: ['*.zip', '*.tar.gz']

# Timeout ของทั้ง build
timeout: '1200s'

# Options
options:
  machineType: 'E2_MEDIUM'
  diskSizeGb: 100
  logging: CLOUD_LOGGING_ONLY
```

### Build Step Fields

```yaml
steps:
  - name: 'gcr.io/cloud-builders/npm'        # Docker image ที่จะรัน
    id: 'install-dependencies'               # ชื่อ step (ใช้ reference ได้)
    entrypoint: 'bash'                       # Override entrypoint
    args:                                    # Arguments
      - '-c'
      - 'npm ci && echo "Dependencies installed"'
    env:                                     # Environment variables
      - 'NODE_ENV=production'
      - 'CI=true'
    secretEnv:                               # Secrets จาก Secret Manager
      - 'DB_PASSWORD'
    dir: 'backend'                           # Working directory
    timeout: '300s'                          # Timeout สำหรับ step นี้
    waitFor:                                 # รอ steps เหล่านี้ก่อน
      - 'other-step-id'
    volumes:                                 # Shared volumes ระหว่าง steps
      - name: 'my-data'
        path: '/workspace/data'
```

### Full Example: Node.js Application

```yaml
# cloudbuild.yaml
steps:
  # Step 1: Install dependencies
  - name: 'node:18-alpine'
    id: 'install'
    entrypoint: 'npm'
    args: ['ci']
    env:
      - 'NODE_ENV=development'

  # Step 2: Lint
  - name: 'node:18-alpine'
    id: 'lint'
    entrypoint: 'npm'
    args: ['run', 'lint']
    waitFor: ['install']

  # Step 3: Type check
  - name: 'node:18-alpine'
    id: 'typecheck'
    entrypoint: 'npm'
    args: ['run', 'typecheck']
    waitFor: ['install']

  # Step 4: Test (รัน parallel กับ lint และ typecheck)
  - name: 'node:18-alpine'
    id: 'test'
    entrypoint: 'npm'
    args: ['run', 'test', '--', '--coverage', '--ci']
    env:
      - 'CI=true'
    waitFor: ['install']

  # Step 5: Build (รอทุก checks ผ่าน)
  - name: 'node:18-alpine'
    id: 'build'
    entrypoint: 'npm'
    args: ['run', 'build']
    env:
      - 'NODE_ENV=production'
    waitFor: ['lint', 'typecheck', 'test']

  # Step 6: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'docker-build'
    args:
      - 'build'
      - '--build-arg'
      - 'NODE_VERSION=18'
      - '-t'
      - '$_REGION-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:$SHORT_SHA'
      - '-t'
      - '$_REGION-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:latest'
      - '.'
    waitFor: ['build']

  # Step 7: Push Docker image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'docker-push'
    args:
      - 'push'
      - '--all-tags'
      - '$_REGION-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app'
    waitFor: ['docker-build']

  # Step 8: Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'my-app'
      - '--image=$_REGION-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:$SHORT_SHA'
      - '--region=$_REGION'
      - '--platform=managed'
      - '--allow-unauthenticated'
      - '--memory=512Mi'
      - '--cpu=1'
      - '--min-instances=1'
      - '--max-instances=10'
    waitFor: ['docker-push']

images:
  - '$_REGION-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:$SHORT_SHA'
  - '$_REGION-docker.pkg.dev/$PROJECT_ID/my-app-repo/my-app:latest'

substitutions:
  _REGION: 'asia-southeast1'

timeout: '1800s'

options:
  machineType: 'E2_MEDIUM'
  logging: CLOUD_LOGGING_ONLY
```

---

## Build Steps และ Images

### Cloud Build Built-in Builders

Google Cloud Build มี pre-built images สำหรับ tools ยอดนิยม:

```yaml
# Docker
- name: 'gcr.io/cloud-builders/docker'

# gcloud CLI
- name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'

# kubectl
- name: 'gcr.io/cloud-builders/kubectl'

# git
- name: 'gcr.io/cloud-builders/git'

# Maven
- name: 'gcr.io/cloud-builders/mvn'

# Gradle
- name: 'gcr.io/cloud-builders/gradle'

# npm
- name: 'gcr.io/cloud-builders/npm'

# gsutil (Google Cloud Storage)
- name: 'gcr.io/cloud-builders/gsutil'

# Helm
- name: 'gcr.io/cloud-builders/helm'
```

### ใช้ Community Images

```yaml
steps:
  # Node.js official image
  - name: 'node:18'
    args: ['npm', 'test']

  # Python official image
  - name: 'python:3.11'
    args: ['pip', 'install', '-r', 'requirements.txt']

  # Golang official image
  - name: 'golang:1.21'
    args: ['go', 'test', './...']

  # Terraform
  - name: 'hashicorp/terraform:1.6'
    args: ['plan']
    dir: 'infrastructure'

  # Trivy (security scanner)
  - name: 'aquasec/trivy:latest'
    args:
      - 'image'
      - '--exit-code'
      - '1'
      - '--severity'
      - 'CRITICAL,HIGH'
      - '$_IMAGE_NAME'
```

### Parallel Steps

```yaml
steps:
  # Step เหล่านี้รันพร้อมกัน (ต่างจาก default ที่รันต่อกัน)
  - name: 'node:18'
    id: 'test-unit'
    args: ['npm', 'run', 'test:unit']
    waitFor: ['-']  # '-' หมายถึงรันทันทีโดยไม่รอ step ก่อนหน้า

  - name: 'node:18'
    id: 'test-integration'
    args: ['npm', 'run', 'test:integration']
    waitFor: ['-']

  - name: 'node:18'
    id: 'lint'
    args: ['npm', 'run', 'lint']
    waitFor: ['-']

  # Step นี้รอให้ทุก step ด้านบนเสร็จก่อน
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/my-app', '.']
    waitFor: ['test-unit', 'test-integration', 'lint']
```

### Volumes (Shared Data Between Steps)

```yaml
steps:
  # Download dependencies และเก็บใน volume
  - name: 'node:18'
    id: 'install'
    args: ['npm', 'ci']
    volumes:
      - name: 'node-modules'
        path: '/workspace/node_modules'

  # ใช้ node_modules จาก volume ใน step ถัดไป
  - name: 'node:18'
    id: 'test'
    args: ['npm', 'test']
    volumes:
      - name: 'node-modules'
        path: '/workspace/node_modules'
    waitFor: ['install']
```

### Custom Build Step Image

```dockerfile
# custom-builders/my-tool/Dockerfile
FROM alpine:3.18
RUN apk add --no-cache curl jq bash

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/bash
# custom-builders/my-tool/entrypoint.sh
echo "Running custom tool..."
# custom logic here
```

```yaml
# Build และ push custom builder
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'gcr.io/$PROJECT_ID/my-custom-tool:latest'
      - './custom-builders/my-tool'
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'push'
      - 'gcr.io/$PROJECT_ID/my-custom-tool:latest'
```

---

## Triggers

### ประเภทของ Triggers

Cloud Build รองรับ triggers หลายประเภท:

1. **Push to branch** - เมื่อ push ไปยัง branch
2. **Push to tag** - เมื่อ push tag
3. **Pull request** - เมื่อสร้าง/อัปเดต PR (GitHub/GitLab เท่านั้น)
4. **Manual** - รัน manually
5. **Pub/Sub** - trigger จาก Pub/Sub message
6. **Webhook** - trigger จาก HTTP request

### สร้าง Trigger ผ่าน Console

1. เปิด Cloud Build > Triggers
2. คลิก "Create Trigger"
3. กำหนด:
   - Name: my-app-trigger
   - Event: Push to a branch
   - Source: Connect to GitHub (หรือ Bitbucket, GitLab)
   - Repository: myorg/my-app
   - Branch: main
   - Build configuration: Cloud Build configuration file
   - Cloud Build configuration file location: cloudbuild.yaml

### สร้าง Trigger ด้วย gcloud CLI

```bash
# Trigger สำหรับ push to main branch
gcloud builds triggers create github \
  --name="push-to-main" \
  --repo-name="my-app" \
  --repo-owner="myorg" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml" \
  --substitutions="_ENVIRONMENT=production,_REGION=asia-southeast1"

# Trigger สำหรับ PR
gcloud builds triggers create github \
  --name="pull-request-check" \
  --repo-name="my-app" \
  --repo-owner="myorg" \
  --pull-request-pattern="^main$" \
  --build-config="cloudbuild-pr.yaml" \
  --comment-control="COMMENTS_ENABLED_FOR_EXTERNAL_CONTRIBUTORS_ONLY"

# Trigger สำหรับ tag
gcloud builds triggers create github \
  --name="release-tag" \
  --repo-name="my-app" \
  --repo-owner="myorg" \
  --tag-pattern="^v[0-9]+\.[0-9]+\.[0-9]+$" \
  --build-config="cloudbuild-release.yaml"

# ดู triggers ที่มีอยู่
gcloud builds triggers list

# รัน trigger manually
gcloud builds triggers run push-to-main \
  --branch=main

# ลบ trigger
gcloud builds triggers delete push-to-main
```

### cloudbuild.yaml สำหรับ Pull Request

```yaml
# cloudbuild-pr.yaml (สำหรับ PR checks)
steps:
  # Install dependencies
  - name: 'node:18'
    id: 'install'
    args: ['npm', 'ci']

  # Lint
  - name: 'node:18'
    id: 'lint'
    args: ['npm', 'run', 'lint']
    waitFor: ['install']

  # Type check
  - name: 'node:18'
    id: 'typecheck'
    args: ['npm', 'run', 'typecheck']
    waitFor: ['install']

  # Unit tests
  - name: 'node:18'
    id: 'test'
    args: ['npm', 'test', '--', '--ci', '--coverage']
    env:
      - 'CI=true'
    waitFor: ['install']

  # Build (ตรวจสอบว่า build ผ่าน)
  - name: 'node:18'
    id: 'build'
    args: ['npm', 'run', 'build']
    waitFor: ['lint', 'typecheck', 'test']

  # Security scan
  - name: 'node:18'
    id: 'security-scan'
    entrypoint: 'bash'
    args:
      - '-c'
      - 'npm audit --audit-level=high'
    waitFor: ['install']

timeout: '600s'

options:
  logging: CLOUD_LOGGING_ONLY
```

### cloudbuild.yaml สำหรับ Release Tag

```yaml
# cloudbuild-release.yaml
steps:
  # Build production Docker image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-image'
    args:
      - 'build'
      - '--build-arg'
      - 'VERSION=$TAG_NAME'
      - '-t'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-app/app:$TAG_NAME'
      - '-t'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-app/app:latest'
      - '.'

  # Push image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'push-image'
    args:
      - 'push'
      - '--all-tags'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-app/app'
    waitFor: ['build-image']

  # Scan image for vulnerabilities
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'scan-image'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        gcloud artifacts docker images scan \
          asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-app/app:$TAG_NAME \
          --format='value(response.scan)' > scan_id.txt
        
        SCAN_ID=$(cat scan_id.txt)
        echo "Scan ID: $SCAN_ID"
        
        # ตรวจสอบผลลัพธ์
        CRITICAL=$(gcloud artifacts docker images list-vulnerabilities $SCAN_ID \
          --filter='vulnerability.effectiveSeverity=CRITICAL' \
          --format='value(name)' | wc -l)
        
        if [ "$CRITICAL" -gt "0" ]; then
          echo "Found $CRITICAL CRITICAL vulnerabilities!"
          exit 1
        fi
    waitFor: ['push-image']

  # Deploy to production Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy-production'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'my-app'
      - '--image=asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-app/app:$TAG_NAME'
      - '--region=asia-southeast1'
      - '--platform=managed'
    waitFor: ['scan-image']

  # Create GitHub Release
  - name: 'curlimages/curl:latest'
    id: 'create-release'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        curl -X POST \
          -H "Authorization: token $$GITHUB_TOKEN" \
          -H "Accept: application/vnd.github.v3+json" \
          https://api.github.com/repos/myorg/my-app/releases \
          -d "{
            \"tag_name\": \"$TAG_NAME\",
            \"name\": \"Release $TAG_NAME\",
            \"body\": \"Automated release by Cloud Build\",
            \"draft\": false,
            \"prerelease\": false
          }"
    secretEnv: ['GITHUB_TOKEN']
    waitFor: ['deploy-production']

availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_ID/secrets/github-token/versions/latest
      env: 'GITHUB_TOKEN'

images:
  - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-app/app:$TAG_NAME'
  - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-app/app:latest'

timeout: '1800s'
```

---

## Substitution Variables

### Default Substitution Variables

Cloud Build มี built-in substitutions ที่ใช้ได้เสมอ:

```yaml
# Built-in substitutions
$PROJECT_ID          # Google Cloud project ID
$BUILD_ID            # Cloud Build build ID
$LOCATION            # Region ของ build (เมื่อใช้ regional pools)
$TRIGGER_NAME        # ชื่อ trigger ที่ trigger build นี้
$TRIGGER_BUILD_CONFIG_PATH  # Path ของ config file

# Git substitutions (เมื่อ build trigger จาก source control)
$COMMIT_SHA          # Full SHA ของ commit
$SHORT_SHA           # 7 characters แรกของ commit SHA
$REPO_NAME           # ชื่อ repository
$BRANCH_NAME         # ชื่อ branch (เมื่อ trigger จาก branch)
$TAG_NAME            # ชื่อ tag (เมื่อ trigger จาก tag)
$REF_NAME            # Branch หรือ tag name
$REVISION_ID         # Commit SHA (เหมือน $COMMIT_SHA)
```

### Custom Substitutions

```yaml
# cloudbuild.yaml
steps:
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    args:
      - 'run'
      - 'deploy'
      - 'my-app'
      - '--image=$_IMAGE_URI'
      - '--region=$_DEPLOY_REGION'
      - '--set-env-vars=ENV=$_ENV'

# กำหนด default values สำหรับ custom substitutions
substitutions:
  _DEPLOY_REGION: 'asia-southeast1'
  _ENV: 'staging'
  _IMAGE_URI: ''  # ต้องกำหนดเมื่อรัน

# Validation สำหรับ substitutions
options:
  substitution_option: ALLOW_LOOSE  # ALLOW_LOOSE หรือ MUST_MATCH
```

### รัน Build ด้วย Custom Substitutions

```bash
# กำหนด substitutions ตอนรัน
gcloud builds submit \
  --config cloudbuild.yaml \
  --substitutions \
    _DEPLOY_REGION=us-central1,\
    _ENV=production,\
    _IMAGE_URI=gcr.io/my-project/my-app:v1.0 \
  .

# หรือใน trigger configuration
gcloud builds triggers create github \
  --name="deploy-production" \
  --substitutions "_ENV=production,_DEPLOY_REGION=asia-southeast1" \
  ...
```

### ใช้ Substitutions ใน Bash

```yaml
steps:
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        # ใช้ substitutions ใน bash script
        FULL_IMAGE="$_REGISTRY/$PROJECT_ID/$_REPO_NAME:$SHORT_SHA"
        echo "Deploying image: $FULL_IMAGE"
        
        # Dynamic tag
        if [ "$BRANCH_NAME" = "main" ]; then
          DEPLOY_TAG="latest"
        else
          DEPLOY_TAG="$BRANCH_NAME-$SHORT_SHA"
        fi
        
        echo "Deploy tag: $DEPLOY_TAG"
        
        gcloud run deploy my-app \
          --image="$FULL_IMAGE" \
          --region="$_REGION"

substitutions:
  _REGISTRY: 'asia-southeast1-docker.pkg.dev'
  _REPO_NAME: 'my-app'
  _REGION: 'asia-southeast1'
```

---

## Build Artifacts ใน Google Cloud Storage

### กำหนด Artifacts ใน cloudbuild.yaml

```yaml
# cloudbuild.yaml
steps:
  - name: 'node:18'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        npm ci
        npm run build
        npm run test -- --coverage
        zip -r dist.zip dist/
        zip -r coverage.zip coverage/

artifacts:
  # Upload ไปยัง GCS
  objects:
    location: 'gs://my-artifacts-bucket/$BUILD_ID'
    paths:
      - 'dist.zip'
      - 'coverage.zip'
      - 'test-results.xml'

  # Images ที่จะ push ไปยัง registry
  images:
    - 'gcr.io/$PROJECT_ID/my-app:$SHORT_SHA'

  # Maven artifacts
  mavenArtifacts:
    - repository: 'https://asia-southeast1-maven.pkg.dev/$PROJECT_ID/my-maven-repo'
      path: '/workspace/app/target/app.jar'
      artifactId: 'my-app'
      groupId: 'com.example'
      version: '1.0.0'

  # NPM packages
  npmPackages:
    - packagePath: '/workspace'
      repository: 'https://asia-southeast1-npm.pkg.dev/$PROJECT_ID/my-npm-repo'

  # Python packages
  pythonPackages:
    - paths: ['dist/*.whl', 'dist/*.tar.gz']
      repository: 'https://asia-southeast1-python.pkg.dev/$PROJECT_ID/my-pypi-repo'
```

### จัดการ GCS Bucket สำหรับ Artifacts

```bash
# สร้าง bucket สำหรับ artifacts
gsutil mb -l asia-southeast1 gs://my-project-build-artifacts

# ตั้งค่า lifecycle (ลบ artifacts เก่ากว่า 30 วัน)
cat > lifecycle.json << 'EOF'
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "Delete"},
        "condition": {
          "age": 30
        }
      }
    ]
  }
}
EOF

gsutil lifecycle set lifecycle.json gs://my-project-build-artifacts

# ดู artifacts จาก build
gsutil ls gs://my-project-build-artifacts/$BUILD_ID/

# Download artifact
gsutil cp gs://my-project-build-artifacts/$BUILD_ID/dist.zip .
```

### Upload Artifacts ใน Build Step

```yaml
steps:
  - name: 'node:18'
    id: 'build'
    entrypoint: 'npm'
    args: ['run', 'build']

  - name: 'gcr.io/cloud-builders/gsutil'
    id: 'upload-artifacts'
    args:
      - '-m'
      - 'cp'
      - '-r'
      - 'dist/'
      - 'gs://my-artifacts-bucket/$SHORT_SHA/'
    waitFor: ['build']

  - name: 'gcr.io/cloud-builders/gsutil'
    id: 'upload-coverage'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        gsutil -m cp -r coverage/ gs://my-artifacts-bucket/$SHORT_SHA/coverage/
        echo "Coverage report: https://storage.googleapis.com/my-artifacts-bucket/$SHORT_SHA/coverage/lcov-report/index.html"
    waitFor: ['build']
```

---

## Deployment ไปยัง Cloud Run/GKE/App Engine

### Deploy ไปยัง Cloud Run

```yaml
# cloudbuild-cloudrun.yaml
steps:
  # Build and push image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build'
    args:
      - 'build'
      - '-t'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '.'

  - name: 'gcr.io/cloud-builders/docker'
    id: 'push'
    args:
      - 'push'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
    waitFor: ['build']

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - 'my-app'
      - '--image=asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '--region=asia-southeast1'
      - '--platform=managed'
      - '--allow-unauthenticated'
      - '--port=8080'
      - '--memory=512Mi'
      - '--cpu=1'
      - '--concurrency=80'
      - '--min-instances=1'
      - '--max-instances=10'
      - '--set-env-vars=NODE_ENV=production'
      - '--set-secrets=DB_PASSWORD=db-password:latest'
    waitFor: ['push']

  # ตรวจสอบ deployment
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'verify'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        # ดู service URL
        SERVICE_URL=$(gcloud run services describe my-app \
          --region=asia-southeast1 \
          --format='value(status.url)')
        
        echo "Service URL: $SERVICE_URL"
        
        # ทดสอบ health endpoint
        HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $SERVICE_URL/health)
        
        if [ "$HTTP_STATUS" = "200" ]; then
          echo "Deployment successful! Service is healthy."
        else
          echo "Health check failed with status: $HTTP_STATUS"
          exit 1
        fi
    waitFor: ['deploy']
```

### Deploy ไปยัง GKE

```yaml
# cloudbuild-gke.yaml
steps:
  # Build and push image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build'
    args:
      - 'build'
      - '-t'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
      - '.'

  - name: 'gcr.io/cloud-builders/docker'
    id: 'push'
    args:
      - 'push'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA'
    waitFor: ['build']

  # Get GKE credentials
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'get-credentials'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        gcloud container clusters get-credentials my-cluster \
          --region=asia-southeast1 \
          --project=$PROJECT_ID
    waitFor: ['push']

  # Update Helm values
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'update-values'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        # Update image tag ใน values file
        sed -i "s/tag: .*/tag: $SHORT_SHA/" kubernetes/values-production.yaml
        cat kubernetes/values-production.yaml
    waitFor: ['get-credentials']

  # Deploy ด้วย Helm
  - name: 'alpine/helm:3.14'
    id: 'helm-deploy'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        # Copy kubeconfig
        cp -r /root/.kube /root/.kube.bak || true
        
        helm upgrade --install my-app ./kubernetes/my-app \
          --namespace production \
          --create-namespace \
          -f kubernetes/values-production.yaml \
          --set image.repository=asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app \
          --set image.tag=$SHORT_SHA \
          --wait \
          --timeout 10m \
          --atomic
    volumes:
      - name: 'kube-config'
        path: '/root/.kube'
    waitFor: ['update-values']

  # Verify deployment
  - name: 'gcr.io/cloud-builders/kubectl'
    id: 'verify'
    args:
      - 'rollout'
      - 'status'
      - 'deployment/my-app'
      - '-n'
      - 'production'
    env:
      - 'CLOUDSDK_COMPUTE_REGION=asia-southeast1'
      - 'CLOUDSDK_CONTAINER_CLUSTER=my-cluster'
    waitFor: ['helm-deploy']
```

### Deploy ไปยัง App Engine

```yaml
# cloudbuild-appengine.yaml
steps:
  # Install dependencies
  - name: 'node:18'
    id: 'install'
    args: ['npm', 'ci', '--production']

  # Build
  - name: 'node:18'
    id: 'build'
    args: ['npm', 'run', 'build']
    waitFor: ['install']

  # Deploy to App Engine
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy'
    entrypoint: 'gcloud'
    args:
      - 'app'
      - 'deploy'
      - 'app.yaml'
      - '--version=$SHORT_SHA'
      - '--no-promote'  # ยังไม่ส่ง traffic ไปยัง version ใหม่
      - '--quiet'
    waitFor: ['build']

  # Run smoke tests บน new version
  - name: 'curlimages/curl:latest'
    id: 'smoke-test'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        VERSION_URL="https://$SHORT_SHA-dot-$PROJECT_ID.uc.r.appspot.com"
        
        echo "Testing: $VERSION_URL"
        
        HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" $VERSION_URL/health)
        
        if [ "$HTTP_STATUS" = "200" ]; then
          echo "Smoke test passed"
        else
          echo "Smoke test failed: HTTP $HTTP_STATUS"
          # Rollback
          gcloud app versions delete $SHORT_SHA --quiet
          exit 1
        fi
    waitFor: ['deploy']

  # Migrate traffic ไปยัง new version
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'migrate-traffic'
    entrypoint: 'gcloud'
    args:
      - 'app'
      - 'services'
      - 'set-traffic'
      - 'default'
      - '--splits=$SHORT_SHA=1'
      - '--split-by=random'
    waitFor: ['smoke-test']

timeout: '900s'
```

---

## Build Notifications

### Pub/Sub Notifications

```bash
# Enable Pub/Sub ใน Cloud Build
gcloud builds triggers update my-trigger \
  --pubsub-config-topic=projects/$PROJECT_ID/topics/cloud-builds

# หรือ ตั้งค่า default Pub/Sub topic
gcloud alpha builds settings set ENABLE_PUBSUB=true
```

### Cloud Build ส่ง notifications ไปยัง Pub/Sub topic:
```
projects/{project_id}/topics/cloud-builds
```

### สร้าง Cloud Function สำหรับ Slack Notifications

```python
# functions/notify-slack/main.py
import json
import base64
import functions_framework
from google.cloud import secretmanager
import urllib.request

def get_secret(secret_name: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/my-project/secrets/{secret_name}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

def get_status_color(status: str) -> str:
    colors = {
        'SUCCESS': '#2EB886',
        'FAILURE': '#CC0000',
        'INTERNAL_ERROR': '#CC0000',
        'TIMEOUT': '#FFA500',
        'CANCELLED': '#808080',
        'WORKING': '#439FE0',
        'QUEUED': '#808080'
    }
    return colors.get(status, '#808080')

@functions_framework.cloud_event
def notify_slack(cloud_event):
    # Decode Pub/Sub message
    pubsub_message = base64.b64decode(cloud_event.data["message"]["data"])
    build = json.loads(pubsub_message)
    
    # เฉพาะ status ที่สนใจ
    status = build.get('status', 'UNKNOWN')
    if status not in ['SUCCESS', 'FAILURE', 'TIMEOUT', 'CANCELLED']:
        return

    project_id = build.get('projectId', '')
    build_id = build.get('id', '')
    trigger_name = build.get('buildTriggerId', 'manual')
    
    # หา commit info
    substitutions = build.get('substitutions', {})
    branch = substitutions.get('BRANCH_NAME', 'unknown')
    commit_sha = substitutions.get('SHORT_SHA', 'unknown')
    repo_name = substitutions.get('REPO_NAME', 'unknown')
    
    log_url = f"https://console.cloud.google.com/cloud-build/builds/{build_id}?project={project_id}"
    
    message = {
        'attachments': [
            {
                'color': get_status_color(status),
                'title': f'Cloud Build: {repo_name}',
                'title_link': log_url,
                'fields': [
                    {'title': 'Status', 'value': status, 'short': True},
                    {'title': 'Branch', 'value': branch, 'short': True},
                    {'title': 'Commit', 'value': commit_sha, 'short': True},
                    {'title': 'Trigger', 'value': trigger_name, 'short': True},
                    {'title': 'Build ID', 'value': build_id[:8], 'short': True}
                ],
                'footer': 'Google Cloud Build'
            }
        ]
    }
    
    # ดึง Slack webhook URL จาก Secret Manager
    webhook_url = get_secret('slack-webhook-url')
    
    data = json.dumps(message).encode('utf-8')
    req = urllib.request.Request(
        webhook_url,
        data=data,
        headers={'Content-Type': 'application/json'}
    )
    
    with urllib.request.urlopen(req) as response:
        print(f"Slack response: {response.status}")
```

```yaml
# functions/notify-slack/requirements.txt
functions-framework==3.*
google-cloud-secret-manager
```

```bash
# Deploy Cloud Function
gcloud functions deploy notify-slack \
  --gen2 \
  --runtime=python311 \
  --region=asia-southeast1 \
  --source=./functions/notify-slack \
  --entry-point=notify_slack \
  --trigger-topic=cloud-builds \
  --service-account=cloud-function-sa@$PROJECT_ID.iam.gserviceaccount.com

# Grant Secret Manager access
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:cloud-function-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

### Email Notifications ด้วย SendGrid

```python
# functions/email-notify/main.py
import json
import base64
import functions_framework
import sendgrid
from sendgrid.helpers.mail import Mail

@functions_framework.cloud_event
def send_email_notification(cloud_event):
    pubsub_message = base64.b64decode(cloud_event.data["message"]["data"])
    build = json.loads(pubsub_message)
    
    status = build.get('status', 'UNKNOWN')
    if status not in ['SUCCESS', 'FAILURE']:
        return
    
    project_id = build.get('projectId', '')
    build_id = build.get('id', '')
    substitutions = build.get('substitutions', {})
    
    sg = sendgrid.SendGridAPIClient(api_key=get_secret('sendgrid-api-key'))
    
    subject = f"[Cloud Build] {status}: {substitutions.get('REPO_NAME', 'unknown')}"
    
    html_content = f"""
    <h2>Build {status}</h2>
    <p>Project: {project_id}</p>
    <p>Repository: {substitutions.get('REPO_NAME', 'unknown')}</p>
    <p>Branch: {substitutions.get('BRANCH_NAME', 'unknown')}</p>
    <p>Commit: {substitutions.get('SHORT_SHA', 'unknown')}</p>
    <p><a href="https://console.cloud.google.com/cloud-build/builds/{build_id}">View Build Logs</a></p>
    """
    
    message = Mail(
        from_email='noreply@example.com',
        to_emails='team@example.com',
        subject=subject,
        html_content=html_content
    )
    
    sg.send(message)
```

---

## Cloud Build Private Pools

Private pools ให้คุณ run builds ในทรัพยากรที่ควบคุมเองได้ใน VPC ของคุณ

### สร้าง Private Pool

```bash
# สร้าง Private Pool
gcloud builds worker-pools create my-private-pool \
  --region=asia-southeast1 \
  --worker-count=5 \
  --worker-machine-type=e2-standard-4 \
  --worker-disk-size=100 \
  --no-public-egress  # ไม่มี public internet access

# List pools
gcloud builds worker-pools list \
  --region=asia-southeast1

# ดูรายละเอียด pool
gcloud builds worker-pools describe my-private-pool \
  --region=asia-southeast1

# ลบ pool
gcloud builds worker-pools delete my-private-pool \
  --region=asia-southeast1
```

### ใช้ Private Pool ใน Build

```yaml
# cloudbuild.yaml
steps:
  - name: 'node:18'
    args: ['npm', 'test']

options:
  # ใช้ private pool
  pool:
    name: 'projects/$PROJECT_ID/locations/asia-southeast1/workerPools/my-private-pool'
```

### Private Pool กับ VPC Peering

```bash
# สร้าง VPC network สำหรับ private pool
gcloud compute networks create cloud-build-network \
  --subnet-mode=custom

gcloud compute networks subnets create cloud-build-subnet \
  --network=cloud-build-network \
  --region=asia-southeast1 \
  --range=10.0.0.0/24

# สร้าง private pool พร้อม VPC peering
gcloud builds worker-pools create my-vpc-pool \
  --region=asia-southeast1 \
  --worker-count=3 \
  --peered-network=projects/$PROJECT_ID/global/networks/cloud-build-network \
  --peered-network-ip-range=10.0.0.0/24
```

---

## Integration กับ Container Registry และ Artifact Registry

### ตั้งค่า Artifact Registry

```bash
# สร้าง Artifact Registry repository สำหรับ Docker images
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=asia-southeast1 \
  --description="Docker images for my app"

# สร้าง repository สำหรับ npm packages
gcloud artifacts repositories create my-npm-repo \
  --repository-format=npm \
  --location=asia-southeast1

# สร้าง repository สำหรับ Maven
gcloud artifacts repositories create my-maven-repo \
  --repository-format=maven \
  --location=asia-southeast1

# สร้าง repository สำหรับ Python
gcloud artifacts repositories create my-pypi-repo \
  --repository-format=python \
  --location=asia-southeast1

# List repositories
gcloud artifacts repositories list \
  --location=asia-southeast1

# Configure Docker auth
gcloud auth configure-docker asia-southeast1-docker.pkg.dev
```

### Push Images ไปยัง Artifact Registry

```yaml
# cloudbuild.yaml
steps:
  # Build multi-platform image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-amd64'
    args:
      - 'buildx'
      - 'build'
      - '--platform=linux/amd64'
      - '-t'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA-amd64'
      - '--push'
      - '.'

  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-arm64'
    args:
      - 'buildx'
      - 'build'
      - '--platform=linux/arm64'
      - '-t'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA-arm64'
      - '--push'
      - '.'

  # Create multi-arch manifest
  - name: 'gcr.io/cloud-builders/docker'
    id: 'create-manifest'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        docker manifest create \
          asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA \
          asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA-amd64 \
          asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA-arm64
        
        docker manifest push \
          asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA
    waitFor: ['build-amd64', 'build-arm64']

images:
  - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA'
```

### Vulnerability Scanning

```yaml
steps:
  - name: 'gcr.io/cloud-builders/docker'
    id: 'push'
    args:
      - 'push'
      - 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA'

  # Scan image (ต้องเปิด Container Analysis API)
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'scan-vulnerabilities'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        echo "Waiting for vulnerability scan to complete..."
        
        # รอ 30 วินาที เพื่อให้ scan เสร็จ
        sleep 30
        
        # ดูผลลัพธ์
        CRITICAL_COUNT=$(gcloud artifacts docker images list-vulnerabilities \
          asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA \
          --filter="vulnerability.effectiveSeverity=CRITICAL" \
          --format="value(name)" | wc -l)
        
        HIGH_COUNT=$(gcloud artifacts docker images list-vulnerabilities \
          asia-southeast1-docker.pkg.dev/$PROJECT_ID/my-docker-repo/my-app:$SHORT_SHA \
          --filter="vulnerability.effectiveSeverity=HIGH" \
          --format="value(name)" | wc -l)
        
        echo "Critical vulnerabilities: $CRITICAL_COUNT"
        echo "High vulnerabilities: $HIGH_COUNT"
        
        if [ "$CRITICAL_COUNT" -gt "0" ]; then
          echo "Build failed: Found CRITICAL vulnerabilities!"
          exit 1
        fi
        
        echo "No CRITICAL vulnerabilities found"
    waitFor: ['push']
```

---

## Build Security และ Secret Management

### ใช้ Secret Manager

```yaml
# cloudbuild.yaml
steps:
  - name: 'node:18'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        # ใช้ secret ใน script
        echo "Connecting to database: $$DB_HOST"
        npm run db:migrate
        npm run seed
    secretEnv: ['DB_HOST', 'DB_PASSWORD', 'API_KEY']

# กำหนด secrets ที่จะ inject เป็น env vars
availableSecrets:
  secretManager:
    - versionName: projects/$PROJECT_ID/secrets/db-host/versions/latest
      env: 'DB_HOST'
    - versionName: projects/$PROJECT_ID/secrets/db-password/versions/latest
      env: 'DB_PASSWORD'
    - versionName: projects/$PROJECT_ID/secrets/api-key/versions/latest
      env: 'API_KEY'
```

### สร้าง Secrets

```bash
# สร้าง secret
echo -n "my-database-password" | \
  gcloud secrets create db-password --data-file=-

# เพิ่ม version ใหม่
echo -n "new-password" | \
  gcloud secrets versions add db-password --data-file=-

# ดู secret (ไม่แสดง value)
gcloud secrets describe db-password

# Grant Cloud Build access
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:$PROJECT_NUMBER@cloudbuild.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

### Workload Identity Federation (GitHub Actions -> Cloud Build)

```yaml
# .github/workflows/deploy.yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # สำหรับ Workload Identity

    steps:
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider'
          service_account: 'github-actions@my-project.iam.gserviceaccount.com'

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2

      - name: Submit Cloud Build
        run: |
          gcloud builds submit \
            --config cloudbuild.yaml \
            --substitutions _ENV=production \
            .
```

```bash
# ตั้งค่า Workload Identity Federation
# สร้าง pool
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --description="Pool for GitHub Actions"

# สร้าง provider
gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"

# Grant access สำหรับ specific repository
gcloud iam service-accounts add-iam-policy-binding \
  github-actions@$PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/$PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/myorg/my-app"
```

---

## แบบฝึกหัด

### Exercise 1: Setup Cloud Build สำหรับ Node.js App

สร้าง complete Cloud Build pipeline สำหรับ Node.js application:

1. สร้าง `cloudbuild.yaml`:
```yaml
steps:
  - name: 'node:18'
    id: 'install'
    args: ['npm', 'ci']

  - name: 'node:18'
    id: 'test'
    args: ['npm', 'test']
    waitFor: ['install']

  - name: 'gcr.io/cloud-builders/docker'
    id: 'build'
    args: ['build', '-t', 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:$SHORT_SHA', '.']
    waitFor: ['test']

  - name: 'gcr.io/cloud-builders/docker'
    id: 'push'
    args: ['push', 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:$SHORT_SHA']
    waitFor: ['build']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy'
    entrypoint: 'gcloud'
    args:
      - 'run', 'deploy', 'my-app'
      - '--image=asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:$SHORT_SHA'
      - '--region=asia-southeast1'
      - '--platform=managed'
    waitFor: ['push']
```

2. ตั้งค่า trigger:
```bash
gcloud builds triggers create github \
  --name="my-app-ci-cd" \
  --repo-owner="myorg" \
  --repo-name="my-app" \
  --branch-pattern="^main$" \
  --build-config="cloudbuild.yaml"
```

3. Test โดย push code ไปยัง repository

### Exercise 2: Multi-Environment Pipeline

สร้าง pipeline ที่ deploy ไปยัง staging และ production:

```yaml
# cloudbuild-staging.yaml (trigger เมื่อ push ไปยัง develop branch)
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:staging-$SHORT_SHA', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:staging-$SHORT_SHA']
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args: ['run', 'deploy', 'my-app-staging', '--image=asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:staging-$SHORT_SHA', '--region=asia-southeast1', '--no-allow-unauthenticated']

# cloudbuild-production.yaml (trigger เมื่อ push tag v*)
steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:$TAG_NAME', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:$TAG_NAME']
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args: ['run', 'deploy', 'my-app-prod', '--image=asia-southeast1-docker.pkg.dev/$PROJECT_ID/apps/my-app:$TAG_NAME', '--region=asia-southeast1']
```

### Exercise 3: ตั้งค่า Slack Notifications

1. สร้าง Slack App และรับ Webhook URL
2. เก็บ URL ใน Secret Manager:
```bash
echo -n "https://hooks.slack.com/services/..." | \
  gcloud secrets create slack-webhook-url --data-file=-
```

3. Deploy Cloud Function จาก [section Notifications](#build-notifications)

4. ทดสอบ โดย trigger build

---

## สรุป

Google Cloud Build เป็น CI/CD platform ที่:

1. **Serverless** → ไม่ต้องจัดการ server
2. **Docker-native** → ทุก step รันใน container
3. **Parallel execution** → เพิ่มความเร็ว build
4. **Deep GCP integration** → Cloud Run, GKE, App Engine
5. **Security-first** → Secret Manager, VPC, IAM
6. **Artifact Registry** → Modern container and package registry
7. **Private Pools** → สำหรับ enterprise requirements

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **Azure DevOps** ซึ่งเป็น CI/CD platform ของ Microsoft Azure

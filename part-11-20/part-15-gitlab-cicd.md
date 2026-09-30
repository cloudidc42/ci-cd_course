# Part 15: GitLab CI/CD

## บทนำ

GitLab CI/CD เป็นระบบ continuous integration และ continuous delivery ที่ built-in อยู่ใน GitLab platform มันมีความยืดหยุ่นสูง รองรับทั้ง cloud-hosted และ self-managed และมีฟีเจอร์ครบครัน ในบทนี้เราจะเรียนรู้ตั้งแต่พื้นฐานไปจนถึงเทคนิคขั้นสูง

---

## 15.1 .gitlab-ci.yml พื้นฐาน

### โครงสร้างพื้นฐาน

`.gitlab-ci.yml` คือไฟล์ configuration หลักที่อยู่ที่ root ของ repository

```yaml
# .gitlab-ci.yml

# กำหนด image เริ่มต้นสำหรับทุก job
image: node:20-alpine

# กำหนด stages และลำดับการทำงาน
stages:
  - install
  - test
  - build
  - security
  - deploy

# กำหนด variables ระดับ global
variables:
  NODE_ENV: test
  DOCKER_DRIVER: overlay2
  FF_USE_FASTZIP: "true"

# Cache สำหรับทุก job (default)
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/

# Job: ติดตั้ง dependencies
install:
  stage: install
  script:
    - npm ci --cache .npm --prefer-offline
  artifacts:
    paths:
      - node_modules/
    expire_in: 1 hour

# Job: รัน unit tests
unit-test:
  stage: test
  needs: [install]
  script:
    - npm run test:unit
    - npm run test:coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    when: always
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
    paths:
      - coverage/

# Job: รัน linting
lint:
  stage: test
  needs: [install]
  script:
    - npm run lint
    - npm run type-check
  artifacts:
    reports:
      codequality: gl-code-quality-report.json

# Job: Build application
build:
  stage: build
  needs: [unit-test, lint]
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
```

---

## 15.2 Stages, Jobs, และ Scripts

### การกำหนด Stages

```yaml
stages:
  - validate      # ตรวจสอบ code quality
  - test          # รัน tests
  - build         # build artifacts
  - package       # สร้าง Docker images
  - review        # deploy review apps
  - staging       # deploy to staging
  - production    # deploy to production
  - cleanup       # ลบ review environments
```

### Job Configuration ขั้นสูง

```yaml
# Job พร้อมทุก options
comprehensive-job:
  stage: test
  image:
    name: python:3.11
    entrypoint: [""]     # override entrypoint
  
  services:
    - name: postgres:15
      alias: database
    - name: redis:7
      alias: cache
  
  variables:
    DATABASE_URL: "postgresql://test:test@database/testdb"
    REDIS_URL: "redis://cache:6379"
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
  
  before_script:
    - pip install -r requirements.txt
    - python manage.py migrate
  
  script:
    - python manage.py test --parallel
  
  after_script:
    - echo "Tests completed"
    - python manage.py coverage_report
  
  artifacts:
    when: always        # เก็บ artifacts ไม่ว่าจะ success หรือ fail
    expire_in: 30 days
    paths:
      - test-results/
      - coverage/
    reports:
      junit: test-results/junit.xml
  
  coverage: '/TOTAL.+?(\d+\.?\d*)%$/'
  
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
  
  timeout: 30 minutes
  
  interruptible: true   # อนุญาตให้ cancel ได้เมื่อมี newer pipeline
  
  allow_failure:
    exit_codes: [137]   # อนุญาตให้ fail เฉพาะ exit code 137 (OOM)
  
  tags:
    - docker
    - linux
    - high-memory
```

### Multi-line Scripts

```yaml
complex-script:
  stage: build
  script:
    - |
      echo "=== Starting build ==="
      export BUILD_TIME=$(date +%Y%m%d_%H%M%S)
      
      # Build application
      npm run build
      
      # Tag Docker image
      docker build \
        --label "build.time=$BUILD_TIME" \
        --label "build.commit=$CI_COMMIT_SHA" \
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA \
        -t $CI_REGISTRY_IMAGE:latest \
        .
      
      echo "=== Build completed ==="
    
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
```

---

## 15.3 GitLab Runners

### ประเภทของ Runners

**1. Shared Runners** - ใช้ร่วมกันทั่วทั้ง GitLab instance  
**2. Group Runners** - ใช้ร่วมกันในกลุ่ม  
**3. Project Runners** - เฉพาะ project

### ติดตั้ง GitLab Runner

```bash
# ติดตั้งบน Ubuntu/Debian
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# ติดตั้งด้วย Docker
docker run -d --name gitlab-runner --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest

# Register runner
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.com/" \
  --registration-token "YOUR_REGISTRATION_TOKEN" \
  --executor "docker" \
  --docker-image "alpine:latest" \
  --description "My Docker Runner" \
  --tag-list "docker,linux" \
  --run-untagged="true" \
  --locked="false"
```

### Runner Configuration (`/etc/gitlab-runner/config.toml`)

```toml
concurrent = 10
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "Production Runner"
  url = "https://gitlab.com/"
  token = "RUNNER_TOKEN"
  executor = "docker"
  
  [runners.custom_build_dir]
  
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      AccessKey = "AWS_ACCESS_KEY"
      SecretKey = "AWS_SECRET_KEY"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "ap-southeast-1"
      Insecure = false
    
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = true         # จำเป็นสำหรับ Docker-in-Docker
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = [
      "/certs/client",
      "/cache",
      "/var/run/docker.sock:/var/run/docker.sock"
    ]
    shm_size = 0
    
    # Resource limits
    cpus = "2"
    memory = "4g"
    memory_swap = "4g"
```

### Kubernetes Executor

```toml
[[runners]]
  name = "Kubernetes Runner"
  url = "https://gitlab.com/"
  token = "RUNNER_TOKEN"
  executor = "kubernetes"
  
  [runners.kubernetes]
    host = "https://kubernetes.default.svc"
    namespace = "gitlab-runner"
    
    [runners.kubernetes.pod_labels]
      "gitlab.com/runner" = "true"
    
    [[runners.kubernetes.volumes.empty_dir]]
      name = "docker-certs"
      mount_path = "/certs/client"
      medium = "Memory"
```

---

## 15.4 Variables ใน GitLab CI/CD

### ประเภทของ Variables

```yaml
variables:
  # Static variables
  APP_NAME: "myapp"
  APP_VERSION: "1.0.0"
  
  # Dynamic variables (ใช้ predefined variables)
  IMAGE_TAG: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"
  
  # Variables ที่คำนวณจากหลาย parts
  FULL_IMAGE: "$CI_REGISTRY_IMAGE/$APP_NAME:$CI_COMMIT_REF_SLUG"
```

### Predefined Variables ที่ใช้บ่อย

```yaml
show-variables:
  script:
    - echo "Project: $CI_PROJECT_NAME"
    - echo "Project ID: $CI_PROJECT_ID"
    - echo "Project Path: $CI_PROJECT_PATH"
    - echo "Project URL: $CI_PROJECT_URL"
    - echo "Pipeline ID: $CI_PIPELINE_ID"
    - echo "Pipeline URL: $CI_PIPELINE_URL"
    - echo "Job ID: $CI_JOB_ID"
    - echo "Job Name: $CI_JOB_NAME"
    - echo "Job Stage: $CI_JOB_STAGE"
    - echo "Commit SHA: $CI_COMMIT_SHA"
    - echo "Short SHA: $CI_COMMIT_SHORT_SHA"
    - echo "Branch: $CI_COMMIT_REF_NAME"
    - echo "Tag: $CI_COMMIT_TAG"
    - echo "Registry: $CI_REGISTRY"
    - echo "Registry Image: $CI_REGISTRY_IMAGE"
    - echo "Merge Request ID: $CI_MERGE_REQUEST_IID"
    - echo "Runner Tags: $CI_RUNNER_TAGS"
```

### Masked Variables

```yaml
# ตั้งค่าใน GitLab UI:
# Settings > CI/CD > Variables
# ✅ Protected (ใช้ได้เฉพาะ protected branches/tags)
# ✅ Masked (ซ่อนค่าใน job logs)

deploy:
  script:
    - echo "$DATABASE_PASSWORD"  # จะแสดงเป็น [MASKED]
    - export DB_URL="postgresql://user:$DATABASE_PASSWORD@host/db"
```

### Variable Expansion

```yaml
variables:
  BASE_URL: "https://api.myapp.com"
  AUTH_ENDPOINT: "$BASE_URL/auth"    # ใช้ variable อื่น
  FULL_PATH: "${BASE_URL}/v2/items"  # ใช้ ${} syntax
```

---

## 15.5 Artifacts และ Cache

### Artifacts

```yaml
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - npm run build
  artifacts:
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
    paths:
      - dist/
      - public/
    exclude:
      - dist/**/*.map    # ไม่เก็บ source maps
    expire_in: 1 week
    when: on_success

test:
  stage: test
  needs:
    - job: build
      artifacts: true    # ดาวน์โหลด artifacts จาก build job
  script:
    - ls dist/          # ไฟล์จาก build job พร้อมใช้งาน
    - npm test

# Artifact reports
test-with-reports:
  stage: test
  script:
    - npm run test:coverage
    - npm run test:e2e
  artifacts:
    when: always
    reports:
      junit:
        - junit.xml
        - e2e-results.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      sast: gl-sast-report.json
      dependency_scanning: gl-dependency-scanning-report.json
```

### Cache Strategy

```yaml
# Cache พื้นฐาน
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/

# Cache ตาม hash ของ lockfile
cache:
  key:
    files:
      - package-lock.json
    prefix: npm-v2
  paths:
    - node_modules/
    - .npm/

# Multiple cache (GitLab 15.0+)
job-with-multiple-caches:
  cache:
    - key:
        files:
          - package-lock.json
        prefix: npm
      paths:
        - node_modules/
    - key:
        files:
          - requirements.txt
        prefix: pip
      paths:
        - .venv/
    - key: docker-$CI_COMMIT_REF_SLUG
      paths:
        - .docker/

# Cache policy
build:
  cache:
    key: build-cache
    paths:
      - dist/
    policy: push    # เฉพาะ push (ไม่ pull)

test:
  cache:
    key: build-cache
    paths:
      - dist/
    policy: pull    # เฉพาะ pull (ไม่ push)
```

---

## 15.6 Environments และ Deployments

```yaml
# กำหนด environments สำหรับ deployment tracking

deploy-staging:
  stage: staging
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.myapp.com
    on_stop: stop-staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

stop-staging:
  stage: staging
  script:
    - ./stop-environment.sh staging
  environment:
    name: staging
    action: stop
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual

deploy-production:
  stage: production
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://myapp.com
    deployment_tier: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # ต้อง approve ก่อน
  
  # ป้องกัน environment โดยใช้ protected environment settings
  # Settings > CI/CD > Protected environments
```

---

## 15.7 Include และ Extends

### Include

```yaml
# .gitlab-ci.yml หลัก
include:
  # Include จาก project เดียวกัน
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/test.yml'
  
  # Include จาก repository อื่น
  - project: 'mygroup/ci-templates'
    ref: main
    file: '/templates/docker.yml'
  
  # Include จาก URL (public เท่านั้น)
  - remote: 'https://example.com/ci-template.yml'
  
  # Include template ของ GitLab
  - template: 'SAST.gitlab-ci.yml'
  - template: 'Dependency-Scanning.gitlab-ci.yml'
  - template: 'Container-Scanning.gitlab-ci.yml'

stages:
  - test
  - build
  - deploy
```

### ไฟล์ที่ถูก include (`.gitlab/ci/build.yml`)

```yaml
.docker-build-template:
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build-image:
  extends: .docker-build-template
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

### Extends

```yaml
# .gitlab-ci.yml

# กำหนด template job (ขึ้นต้นด้วย .)
.base-deploy:
  image: alpine:3.18
  before_script:
    - apk add --no-cache curl
    - curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.28.0/bin/linux/amd64/kubectl
    - chmod +x kubectl && mv kubectl /usr/local/bin/
  after_script:
    - echo "Deployment finished at $(date)"

.notify-on-failure:
  after_script:
    - |
      if [ "$CI_JOB_STATUS" == "failed" ]; then
        curl -X POST $SLACK_WEBHOOK \
          -d '{"text":"❌ Deployment ล้มเหลว: '$CI_PROJECT_NAME' - '$CI_JOB_NAME'"}'
      fi

# Job ที่ extends หลาย templates
deploy-staging:
  extends:
    - .base-deploy
    - .notify-on-failure
  stage: staging
  variables:
    ENVIRONMENT: staging
    NAMESPACE: myapp-staging
  script:
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA -n $NAMESPACE
    - kubectl rollout status deployment/myapp -n $NAMESPACE

deploy-production:
  extends:
    - .base-deploy
    - .notify-on-failure
  stage: production
  variables:
    ENVIRONMENT: production
    NAMESPACE: myapp-production
  script:
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA -n $NAMESPACE
    - kubectl rollout status deployment/myapp -n $NAMESPACE
  when: manual
  allow_failure: false
```

---

## 15.8 Rules vs Only/Except

### Rules (วิธีใหม่ที่แนะนำ)

```yaml
# rules ให้ความยืดหยุ่นสูงกว่า only/except

deploy-staging:
  script:
    - ./deploy.sh staging
  rules:
    # Deploy เมื่อ push ไปยัง main branch
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_success
    
    # Deploy เมื่อ tag ขึ้นต้นด้วย v
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+/
      when: on_success
    
    # รัน manual เมื่อเป็น develop branch
    - if: $CI_COMMIT_BRANCH == "develop"
      when: manual
      allow_failure: true
    
    # ไม่รันในกรณีอื่น
    - when: never

# Rules ที่ซับซ้อน
advanced-rules-job:
  script:
    - ./run-tests.sh
  rules:
    # รัน unit tests เสมอ ยกเว้นเมื่อเป็น docs changes เท่านั้น
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      changes:
        - "**/*.js"
        - "**/*.ts"
        - package.json
      when: on_success
    
    # รันเสมอบน main
    - if: $CI_COMMIT_BRANCH == "main"
      when: on_success
    
    # ไม่รันถ้าเป็น docs เท่านั้น
    - if: $CI_COMMIT_BRANCH
      changes:
        - "docs/**/*"
        - "*.md"
      when: never
    
    # Default: รัน
    - when: on_success

# Rules กับ variables
conditional-with-variables:
  script:
    - ./deploy.sh
  rules:
    - if: $CI_COMMIT_BRANCH == "main" && $DEPLOY_ENABLED == "true"
      variables:
        ENVIRONMENT: production
    - if: $CI_COMMIT_BRANCH == "develop"
      variables:
        ENVIRONMENT: staging
    - when: never
```

### Only/Except (วิธีเก่า)

```yaml
# ยังใช้งานได้ แต่ rules แนะนำมากกว่า
test-old-style:
  script:
    - npm test
  only:
    - main
    - develop
    - /^feature\/.*/
    - merge_requests
  except:
    - tags
    - schedules
```

---

## 15.9 Merge Request Pipelines

### กำหนด Pipeline สำหรับ MR เท่านั้น

```yaml
# MR Pipeline - รันเฉพาะเมื่อมี Merge Request
mr-lint:
  script:
    - npm run lint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

mr-unit-test:
  script:
    - npm test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

mr-security-scan:
  script:
    - snyk test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

# Review App สำหรับ MR
deploy-review:
  stage: review
  script:
    - ./deploy-review.sh $CI_MERGE_REQUEST_IID
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://mr-$CI_MERGE_REQUEST_IID.review.myapp.com
    on_stop: stop-review
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: on_success

stop-review:
  stage: review
  script:
    - ./stop-review.sh $CI_MERGE_REQUEST_IID
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
```

### MR Pipeline กับ Branch Pipeline

```yaml
# ป้องกัน pipeline รันซ้ำ
workflow:
  rules:
    # รัน Pipeline สำหรับ Merge Requests
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    # รัน Pipeline บน main หรือ develop
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"
    # รัน Pipeline สำหรับ tags
    - if: $CI_COMMIT_TAG
    # ไม่รัน pipeline สำหรับ branch อื่น (เพื่อหลีกเลี่ยง duplicate)
    - when: never
```

---

## 15.10 Protected Branches

```yaml
# ใช้ protected branches ร่วมกับ CI/CD

# กฎสำหรับ main branch (protected)
deploy-production:
  stage: production
  script:
    - ./deploy.sh production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  # protected branch ต้องการ maintainer permission จึงจะ push ได้

# Security scan เฉพาะ protected branches
security-audit:
  stage: security
  script:
    - npm audit --audit-level=high
    - snyk test --severity-threshold=high
  rules:
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"

# Protected variables (ใช้ได้เฉพาะ protected branches)
# Settings > CI/CD > Variables > Protected: ✅
deploy-with-secrets:
  stage: production
  script:
    - echo "Deploying with protected secret: $PROD_API_KEY"
  rules:
    - if: $CI_COMMIT_BRANCH == "main"  # Protected branch เท่านั้น
```

---

## 15.11 GitLab Container Registry

```yaml
# ใช้ GitLab Container Registry

variables:
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

build-docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    # Build image หลาย tags
    - docker build \
        --label "org.opencontainers.image.revision=$CI_COMMIT_SHA" \
        --label "org.opencontainers.image.created=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
        -t $IMAGE_NAME:$IMAGE_TAG \
        -t $IMAGE_NAME:$CI_COMMIT_REF_SLUG \
        .
    
    # Push ทุก tags
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:$CI_COMMIT_REF_SLUG
    
    # Push latest เฉพาะ main branch
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $IMAGE_NAME:$IMAGE_TAG $IMAGE_NAME:latest
        docker push $IMAGE_NAME:latest
      fi
  
  # Container Scanning
  container-scanning:
    extends: .container-scanning  # ใช้ GitLab template
    variables:
      CS_IMAGE: $IMAGE_NAME:$IMAGE_TAG
    needs:
      - build-docker

# Cleanup old images
cleanup-registry:
  stage: cleanup
  image: python:3.11-alpine
  script:
    - pip install python-gitlab
    - python scripts/cleanup-registry.py --keep-last=10 --project-id=$CI_PROJECT_ID
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

---

## 15.12 GitLab Pages

```yaml
# Deploy GitLab Pages

pages:
  stage: deploy
  image: node:20-alpine
  script:
    - npm ci
    - npm run build:docs
    - mv public_docs public   # ต้องเก็บไว้ใน public/ directory
  artifacts:
    paths:
      - public
    expire_in: 1 day
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# Pages + Custom domain
# 1. เพิ่ม custom domain ใน Settings > Pages
# 2. เพิ่ม DNS records
# 3. Enable HTTPS

# สร้าง documentation จาก code
docs-build:
  stage: build
  script:
    - npm run generate-docs
    - mv docs public
  artifacts:
    paths:
      - public/

# Deploy multiple sites
pages:
  stage: deploy
  script:
    - mkdir -p public
    - cp -r dist/* public/
    - cp -r docs/* public/docs/
    - cp -r coverage/* public/coverage/
  artifacts:
    paths:
      - public
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

---

## 15.13 Auto DevOps

```yaml
# เปิด Auto DevOps ใน Settings > CI/CD > Auto DevOps
# หรือกำหนดในไฟล์

include:
  - template: Auto-DevOps.gitlab-ci.yml

# Override specific stages
variables:
  AUTO_DEVOPS_DOMAIN: myapp.com
  KUBE_INGRESS_BASE_DOMAIN: myapp.com
  POSTGRES_ENABLED: "true"
  POSTGRES_USER: myapp
  POSTGRES_DB: myapp_production
  
  # Disable stages ที่ไม่ต้องการ
  TEST_DISABLED: "true"          # ปิด built-in test
  CODE_QUALITY_DISABLED: "true"  # ปิด code quality

# Override build job
build:
  extends: .auto-build
  variables:
    BUILD_SECRET_DETECT_DISABLED: "true"
```

---

## 15.14 Review Apps

```yaml
# สร้าง Review Apps สำหรับ MR

variables:
  REVIEW_APP_DOMAIN: review.myapp.com

deploy-review-app:
  stage: review
  image: bitnami/kubectl:latest
  script:
    - |
      # สร้าง namespace สำหรับ MR
      kubectl create namespace review-mr-$CI_MERGE_REQUEST_IID --dry-run=client -o yaml | kubectl apply -f -
      
      # Deploy application
      helm upgrade --install \
        review-mr-$CI_MERGE_REQUEST_IID \
        ./helm/myapp \
        --namespace review-mr-$CI_MERGE_REQUEST_IID \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --set ingress.host=mr-$CI_MERGE_REQUEST_IID.$REVIEW_APP_DOMAIN \
        --wait
      
      # แสดง URL
      echo "Review app: https://mr-$CI_MERGE_REQUEST_IID.$REVIEW_APP_DOMAIN"
  
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://mr-$CI_MERGE_REQUEST_IID.$REVIEW_APP_DOMAIN
    on_stop: stop-review-app
    auto_stop_in: 1 week
  
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"

stop-review-app:
  stage: review
  image: bitnami/kubectl:latest
  script:
    - kubectl delete namespace review-mr-$CI_MERGE_REQUEST_IID --ignore-not-found
    - helm uninstall review-mr-$CI_MERGE_REQUEST_IID --namespace review-mr-$CI_MERGE_REQUEST_IID || true
  
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    action: stop
  
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
      when: manual
  
  allow_failure: true
```

---

## 15.15 DAG Pipelines (Directed Acyclic Graph)

### ใช้ `needs` สำหรับ DAG

```yaml
stages:
  - build
  - test
  - deploy

# DAG ทำให้ jobs รันโดยไม่ต้องรอ stage ก่อนหน้าเสร็จทั้งหมด

build-frontend:
  stage: build
  script:
    - npm run build:frontend
  artifacts:
    paths:
      - dist/frontend/

build-backend:
  stage: build
  script:
    - go build -o bin/api ./cmd/api
  artifacts:
    paths:
      - bin/

build-worker:
  stage: build
  script:
    - go build -o bin/worker ./cmd/worker
  artifacts:
    paths:
      - bin/

# test-frontend เริ่มทันทีที่ build-frontend เสร็จ
# ไม่ต้องรอ build-backend และ build-worker
test-frontend:
  stage: test
  needs:
    - job: build-frontend
      artifacts: true
  script:
    - npm test

test-backend:
  stage: test
  needs:
    - job: build-backend
      artifacts: true
  script:
    - go test ./...

test-worker:
  stage: test
  needs:
    - job: build-worker
      artifacts: true
  script:
    - go test ./internal/worker/...

# Integration test รอทั้ง backend และ worker
integration-test:
  stage: test
  needs:
    - job: test-backend
    - job: test-worker
    - job: build-frontend
      artifacts: true
  script:
    - npm run test:integration

# Deploy frontend รอแค่ test-frontend
deploy-frontend:
  stage: deploy
  needs:
    - job: test-frontend
    - job: build-frontend
      artifacts: true
  script:
    - ./deploy-frontend.sh

# Deploy backend รอ integration test
deploy-backend:
  stage: deploy
  needs:
    - job: integration-test
    - job: build-backend
      artifacts: true
  script:
    - ./deploy-backend.sh
```

---

## 15.16 Downstream Pipelines

### Multi-project Pipelines

```yaml
# Pipeline หลักที่ trigger pipelines อื่น

trigger-microservices:
  stage: deploy
  trigger:
    project: mygroup/user-service
    branch: main
    strategy: depend   # รอให้ downstream pipeline เสร็จก่อน
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - "user-service/**/*"

trigger-payment:
  stage: deploy
  trigger:
    project: mygroup/payment-service
    branch: main
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      changes:
        - "payment-service/**/*"

# ส่ง variables ไปยัง downstream
trigger-with-variables:
  stage: deploy
  variables:
    PARENT_PIPELINE_ID: $CI_PIPELINE_ID
    DEPLOY_VERSION: $CI_COMMIT_SHORT_SHA
  trigger:
    project: mygroup/myservice
    strategy: depend
```

### Child Pipelines (Dynamic)

```yaml
# Parent pipeline
generate-child-pipelines:
  stage: build
  script:
    - python generate-pipelines.py > child-pipeline.yml
  artifacts:
    paths:
      - child-pipeline.yml

run-child-pipeline:
  stage: test
  needs:
    - generate-child-pipelines
  trigger:
    include:
      - artifact: child-pipeline.yml
        job: generate-child-pipelines
    strategy: depend
```

---

## 15.17 Complete Pipeline Example

```yaml
# .gitlab-ci.yml - Complete Production Pipeline

image: node:20-alpine

stages:
  - validate
  - test
  - build
  - security
  - staging
  - production
  - cleanup

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  KUBECONFIG: /tmp/kubeconfig

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"
    - if: $CI_COMMIT_TAG =~ /^v\d+/

# ===== VALIDATE =====

lint:
  stage: validate
  script:
    - npm ci
    - npm run lint
    - npm run type-check
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/

# ===== TEST =====

unit-test:
  stage: test
  needs: []
  script:
    - npm ci
    - npm run test:unit -- --coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    when: always
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

integration-test:
  stage: test
  needs: []
  services:
    - postgres:15
    - redis:7
  variables:
    DATABASE_URL: "postgresql://test:test@postgres/testdb"
    REDIS_URL: "redis://redis:6379"
    POSTGRES_DB: testdb
    POSTGRES_USER: test
    POSTGRES_PASSWORD: test
  script:
    - npm ci
    - npm run test:integration
  artifacts:
    when: always
    reports:
      junit: integration-results.xml

# ===== BUILD =====

build-image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  needs:
    - unit-test
    - integration-test
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build
        --label "org.opencontainers.image.revision=$CI_COMMIT_SHA"
        --label "org.opencontainers.image.created=$(date -u +'%Y-%m-%dT%H:%M:%SZ')"
        -t $IMAGE_TAG
        -t $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG
        .
    - docker push $IMAGE_TAG
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

# ===== SECURITY =====

sast:
  stage: security
  needs: []
  include:
    - template: Security/SAST.gitlab-ci.yml

container-scanning:
  stage: security
  needs:
    - build-image
  include:
    - template: Security/Container-Scanning.gitlab-ci.yml
  variables:
    CS_IMAGE: $IMAGE_TAG

# ===== STAGING =====

deploy-staging:
  stage: staging
  image: bitnami/kubectl:latest
  needs:
    - build-image
    - sast
  before_script:
    - echo "$STAGING_KUBECONFIG" > $KUBECONFIG
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n myapp-staging
    - kubectl rollout status deployment/myapp -n myapp-staging --timeout=5m
  environment:
    name: staging
    url: https://staging.myapp.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"

smoke-test-staging:
  stage: staging
  needs:
    - deploy-staging
  script:
    - npm run test:smoke -- --url=https://staging.myapp.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"

# ===== PRODUCTION =====

deploy-production:
  stage: production
  image: bitnami/kubectl:latest
  needs:
    - smoke-test-staging
    - container-scanning
  before_script:
    - echo "$PROD_KUBECONFIG" > $KUBECONFIG
  script:
    - kubectl set image deployment/myapp myapp=$IMAGE_TAG -n myapp-production
    - kubectl rollout status deployment/myapp -n myapp-production --timeout=10m
    
    # Tag as latest
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker pull $IMAGE_TAG
    - docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  environment:
    name: production
    url: https://myapp.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
      allow_failure: false
  
rollback-production:
  stage: production
  image: bitnami/kubectl:latest
  before_script:
    - echo "$PROD_KUBECONFIG" > $KUBECONFIG
  script:
    - kubectl rollout undo deployment/myapp -n myapp-production
    - kubectl rollout status deployment/myapp -n myapp-production --timeout=5m
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  allow_failure: true
```

---

## แบบฝึกหัด Part 15

### แบบฝึกหัดที่ 1: Multi-stage Pipeline

สร้าง pipeline สำหรับ Node.js application ที่มี:
1. Lint และ type-check
2. Unit tests พร้อม coverage report
3. Integration tests กับ PostgreSQL
4. Docker build และ push ไปยัง GitLab Registry
5. Deploy ไปยัง staging อัตโนมัติ
6. Deploy ไปยัง production แบบ manual approval

### แบบฝึกหัดที่ 2: Review Apps

สร้างระบบ Review Apps ที่:
1. สร้าง environment ใหม่ทุกครั้งที่เปิด MR
2. แสดง URL ใน MR description
3. ลบ environment อัตโนมัติเมื่อ MR merge
4. รองรับ multiple MRs พร้อมกัน

### แบบฝึกหัดที่ 3: Security Pipeline

สร้าง security pipeline ที่รวม:
1. SAST (Static Application Security Testing)
2. DAST (Dynamic Application Security Testing)
3. Dependency scanning
4. Container scanning
5. Secret detection
6. สร้าง security report สรุปผล

### แบบฝึกหัดที่ 4: DAG Optimization

ปรับปรุง pipeline ที่ช้าด้วย DAG:
1. วิเคราะห์ pipeline dependencies
2. ระบุ jobs ที่รัน parallel ได้
3. ใช้ `needs` เพื่อเพิ่มความเร็ว
4. วัดผลก่อนและหลัง

### แบบฝึกหัดที่ 5: Template Library

สร้าง CI template library ใน separate project:
1. Template สำหรับ Node.js projects
2. Template สำหรับ Python projects
3. Template สำหรับ Docker build
4. Template สำหรับ Kubernetes deployment
5. Documentation และ versioning

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- **GitLab CI/CD syntax**: โครงสร้างพื้นฐานและ job configuration
- **Runners**: การติดตั้งและ configure runners ประเภทต่างๆ
- **Variables**: การใช้ predefined, custom, และ protected variables
- **Artifacts & Cache**: การจัดการ build artifacts และ cache
- **Environments**: Deployment tracking และ review apps
- **Include/Extends**: Code reuse สำหรับ CI configuration
- **Rules**: การควบคุม pipeline execution อย่างละเอียด
- **MR Pipelines**: Pipeline เฉพาะสำหรับ merge requests
- **DAG Pipelines**: การเพิ่ม performance ด้วย job dependencies
- **Downstream Pipelines**: Multi-project pipeline orchestration

ในบทถัดไป เราจะมาดู Jenkins ซึ่งเป็น CI/CD tool ที่มีมานานและยังคงใช้งานอยู่ในหลายองค์กร

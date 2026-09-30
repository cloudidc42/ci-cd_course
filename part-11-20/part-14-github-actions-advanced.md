# Part 14: GitHub Actions Advanced

## บทนำ

ในบทนี้เราจะเรียนรู้เทคนิคขั้นสูงของ GitHub Actions ที่จะทำให้ pipeline ของคุณมีประสิทธิภาพ ยืดหยุ่น และปลอดภัยมากยิ่งขึ้น เราจะครอบคลุมตั้งแต่ reusable workflows, composite actions, custom actions ไปจนถึง OIDC authentication และ deployment environments

---

## 14.1 Reusable Workflows (workflow_call)

### แนวคิด Reusable Workflows

Reusable Workflows ช่วยให้คุณสร้าง workflow ที่สามารถเรียกใช้จาก workflow อื่นได้ เหมือนกับการสร้าง function ที่ใช้ซ้ำได้ในโปรแกรมมิ่ง

**ประโยชน์:**
- ลดการซ้ำซ้อนของ YAML
- จัดการ pipeline ได้ง่ายขึ้น
- ทีมสามารถแชร์ workflow ได้
- บังคับใช้ standards เดียวกันทั่วทั้งองค์กร

### การสร้าง Reusable Workflow

สร้างไฟล์ `.github/workflows/reusable-deploy.yml`:

```yaml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
        description: 'สภาพแวดล้อมที่จะ deploy'
      image_tag:
        required: true
        type: string
        description: 'Docker image tag'
      registry:
        required: false
        type: string
        default: 'ghcr.io'
        description: 'Container registry'
    secrets:
      DEPLOY_KEY:
        required: true
        description: 'SSH key สำหรับ deploy'
      KUBECONFIG:
        required: false
        description: 'Kubernetes config'
    outputs:
      deployment_url:
        description: 'URL ของ deployment'
        value: ${{ jobs.deploy.outputs.url }}

jobs:
  deploy:
    name: Deploy to ${{ inputs.environment }}
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.deploy.outputs.url }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG }}" > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Deploy to Kubernetes
        id: deploy
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ inputs.registry }}/${{ github.repository }}:${{ inputs.image_tag }} \
            --namespace=${{ inputs.environment }}
          
          kubectl rollout status deployment/myapp \
            --namespace=${{ inputs.environment }} \
            --timeout=300s
          
          # Get service URL
          URL=$(kubectl get service myapp \
            --namespace=${{ inputs.environment }} \
            --output=jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          echo "url=https://$URL" >> $GITHUB_OUTPUT

      - name: Notify deployment
        if: always()
        run: |
          if [ "${{ job.status }}" == "success" ]; then
            echo "✅ Deploy สำเร็จ: ${{ steps.deploy.outputs.url }}"
          else
            echo "❌ Deploy ล้มเหลว"
          fi
```

### การเรียกใช้ Reusable Workflow

สร้างไฟล์ `.github/workflows/main-pipeline.yml`:

```yaml
name: Main CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.build.outputs.tag }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and push Docker image
        id: build
        run: |
          TAG="${{ github.sha }}"
          docker build -t ghcr.io/${{ github.repository }}:$TAG .
          docker push ghcr.io/${{ github.repository }}:$TAG
          echo "tag=$TAG" >> $GITHUB_OUTPUT

  deploy-staging:
    name: Deploy to Staging
    needs: build
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      image_tag: ${{ needs.build.outputs.image_tag }}
      registry: ghcr.io
    secrets:
      DEPLOY_KEY: ${{ secrets.STAGING_DEPLOY_KEY }}
      KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}

  deploy-production:
    name: Deploy to Production
    needs: [build, deploy-staging]
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      image_tag: ${{ needs.build.outputs.image_tag }}
      registry: ghcr.io
    secrets:
      DEPLOY_KEY: ${{ secrets.PROD_DEPLOY_KEY }}
      KUBECONFIG: ${{ secrets.PROD_KUBECONFIG }}
    if: github.ref == 'refs/heads/main'
```

### การใช้ Reusable Workflow จาก Repository อื่น

```yaml
jobs:
  call-external-workflow:
    uses: myorg/shared-workflows/.github/workflows/security-scan.yml@main
    with:
      scan_type: 'full'
    secrets:
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

---

## 14.2 Composite Actions

### Composite Actions คืออะไร?

Composite Actions ช่วยให้คุณรวมหลาย steps เข้าด้วยกันเป็น action เดียว เหมาะสำหรับกลุ่มของ steps ที่ใช้บ่อย

### สร้าง Composite Action

สร้างไฟล์ `.github/actions/setup-project/action.yml`:

```yaml
name: 'Setup Project'
description: 'ติดตั้ง dependencies และ configure environment สำหรับ project'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  python-version:
    description: 'Python version'
    required: false
    default: '3.11'
  cache-key-prefix:
    description: 'Prefix สำหรับ cache key'
    required: false
    default: 'v1'

outputs:
  cache-hit:
    description: 'บอกว่า cache ถูก restore หรือไม่'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: composite
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}

    - name: Cache dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: |
          node_modules
          ~/.npm
          ~/.cache/pip
        key: ${{ inputs.cache-key-prefix }}-${{ runner.os }}-${{ hashFiles('**/package-lock.json', '**/requirements.txt') }}
        restore-keys: |
          ${{ inputs.cache-key-prefix }}-${{ runner.os }}-

    - name: Install Node dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      run: npm ci
      shell: bash

    - name: Install Python dependencies
      if: steps.cache.outputs.cache-hit != 'true'
      run: pip install -r requirements.txt
      shell: bash

    - name: Setup environment variables
      run: |
        echo "APP_ENV=ci" >> $GITHUB_ENV
        echo "NODE_ENV=test" >> $GITHUB_ENV
      shell: bash
```

### การใช้งาน Composite Action ใน Workflow

```yaml
name: CI Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup project
        uses: ./.github/actions/setup-project
        with:
          node-version: '20'
          python-version: '3.11'
          cache-key-prefix: 'v2'
      
      - name: Run tests
        run: npm test
```

---

## 14.3 Custom Actions (JavaScript)

### JavaScript Action

JavaScript Actions ทำงานเร็วกว่า Docker Actions เพราะไม่ต้อง build container

สร้างโครงสร้าง:
```
.github/actions/notify-slack/
├── action.yml
├── index.js
└── package.json
```

#### `action.yml`

```yaml
name: 'Notify Slack'
description: 'ส่ง notification ไปยัง Slack'

inputs:
  webhook-url:
    description: 'Slack Incoming Webhook URL'
    required: true
  message:
    description: 'ข้อความที่จะส่ง'
    required: true
  status:
    description: 'สถานะ (success/failure/warning)'
    required: false
    default: 'info'
  channel:
    description: 'Slack channel'
    required: false
    default: '#deployments'

outputs:
  message-id:
    description: 'ID ของข้อความที่ส่ง'

runs:
  using: 'node20'
  main: 'dist/index.js'
```

#### `index.js`

```javascript
const core = require('@actions/core');
const github = require('@actions/github');
const https = require('https');

async function run() {
  try {
    const webhookUrl = core.getInput('webhook-url', { required: true });
    const message = core.getInput('message', { required: true });
    const status = core.getInput('status') || 'info';
    const channel = core.getInput('channel');

    const context = github.context;
    
    // กำหนดสีตาม status
    const colors = {
      success: '#36a64f',
      failure: '#ff0000',
      warning: '#ffcc00',
      info: '#0078d7'
    };

    const icons = {
      success: '✅',
      failure: '❌',
      warning: '⚠️',
      info: 'ℹ️'
    };

    const payload = {
      channel: channel,
      attachments: [{
        color: colors[status] || colors.info,
        blocks: [
          {
            type: 'section',
            text: {
              type: 'mrkdwn',
              text: `${icons[status]} *${message}*`
            }
          },
          {
            type: 'section',
            fields: [
              {
                type: 'mrkdwn',
                text: `*Repository:*\n<${context.payload.repository?.html_url}|${context.repo.owner}/${context.repo.repo}>`
              },
              {
                type: 'mrkdwn',
                text: `*Branch:*\n${context.ref.replace('refs/heads/', '')}`
              },
              {
                type: 'mrkdwn',
                text: `*Commit:*\n<${context.payload.head_commit?.url}|\`${context.sha.substring(0, 7)}\`>`
              },
              {
                type: 'mrkdwn',
                text: `*Workflow:*\n${context.workflow}`
              }
            ]
          },
          {
            type: 'actions',
            elements: [{
              type: 'button',
              text: {
                type: 'plain_text',
                text: 'View Run'
              },
              url: `${context.payload.repository?.html_url}/actions/runs/${context.runId}`
            }]
          }
        ]
      }]
    };

    const messageId = await sendSlackMessage(webhookUrl, payload);
    core.setOutput('message-id', messageId);
    
    core.info(`✅ ส่ง Slack notification สำเร็จ`);
  } catch (error) {
    core.setFailed(`Action ล้มเหลว: ${error.message}`);
  }
}

function sendSlackMessage(webhookUrl, payload) {
  return new Promise((resolve, reject) => {
    const data = JSON.stringify(payload);
    const url = new URL(webhookUrl);
    
    const options = {
      hostname: url.hostname,
      path: url.pathname,
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Content-Length': data.length
      }
    };

    const req = https.request(options, (res) => {
      let responseData = '';
      res.on('data', (chunk) => { responseData += chunk; });
      res.on('end', () => {
        if (res.statusCode === 200) {
          resolve(Date.now().toString());
        } else {
          reject(new Error(`Slack API ตอบกลับด้วย status ${res.statusCode}: ${responseData}`));
        }
      });
    });

    req.on('error', reject);
    req.write(data);
    req.end();
  });
}

run();
```

#### `package.json`

```json
{
  "name": "notify-slack",
  "version": "1.0.0",
  "description": "GitHub Action สำหรับส่ง Slack notification",
  "main": "dist/index.js",
  "scripts": {
    "build": "ncc build index.js -o dist",
    "package": "npm run build"
  },
  "dependencies": {
    "@actions/core": "^1.10.0",
    "@actions/github": "^6.0.0"
  },
  "devDependencies": {
    "@vercel/ncc": "^0.38.0"
  }
}
```

---

## 14.4 Custom Actions (Docker)

### Docker Action

Docker Actions เหมาะสำหรับงานที่ต้องการ environment เฉพาะหรือ dependencies ที่ซับซ้อน

สร้างโครงสร้าง:
```
.github/actions/code-quality/
├── action.yml
├── Dockerfile
└── entrypoint.sh
```

#### `action.yml`

```yaml
name: 'Code Quality Check'
description: 'ตรวจสอบคุณภาพโค้ดด้วย multiple tools'

inputs:
  path:
    description: 'Path ที่จะตรวจสอบ'
    required: false
    default: '.'
  fail-on-error:
    description: 'Fail workflow ถ้าพบ error'
    required: false
    default: 'true'

outputs:
  report:
    description: 'Path ของ report file'

runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.path }}
    - ${{ inputs.fail-on-error }}
```

#### `Dockerfile`

```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir \
    pylint \
    flake8 \
    black \
    bandit \
    safety

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

#### `entrypoint.sh`

```bash
#!/bin/bash
set -e

PATH_TO_CHECK=${1:-.}
FAIL_ON_ERROR=${2:-true}
REPORT_FILE="/tmp/quality-report.txt"

echo "=== Code Quality Report ===" > $REPORT_FILE
echo "Date: $(date)" >> $REPORT_FILE
echo "Path: $PATH_TO_CHECK" >> $REPORT_FILE
echo "" >> $REPORT_FILE

ERRORS=0

# Black - formatting check
echo "### Black Formatting ###" >> $REPORT_FILE
if black --check $PATH_TO_CHECK --quiet 2>&1 >> $REPORT_FILE; then
    echo "✅ Black: Pass" >> $REPORT_FILE
else
    echo "❌ Black: Fail" >> $REPORT_FILE
    ERRORS=$((ERRORS + 1))
fi

# Flake8 - linting
echo "### Flake8 Linting ###" >> $REPORT_FILE
if flake8 $PATH_TO_CHECK --max-line-length=100 2>&1 >> $REPORT_FILE; then
    echo "✅ Flake8: Pass" >> $REPORT_FILE
else
    echo "❌ Flake8: Fail" >> $REPORT_FILE
    ERRORS=$((ERRORS + 1))
fi

# Bandit - security
echo "### Bandit Security ###" >> $REPORT_FILE
if bandit -r $PATH_TO_CHECK -q 2>&1 >> $REPORT_FILE; then
    echo "✅ Bandit: Pass" >> $REPORT_FILE
else
    echo "❌ Bandit: Security issues found" >> $REPORT_FILE
    ERRORS=$((ERRORS + 1))
fi

# Safety - dependency check
echo "### Safety Dependency Check ###" >> $REPORT_FILE
if safety check 2>&1 >> $REPORT_FILE; then
    echo "✅ Safety: Pass" >> $REPORT_FILE
else
    echo "❌ Safety: Vulnerabilities found" >> $REPORT_FILE
    ERRORS=$((ERRORS + 1))
fi

cat $REPORT_FILE
echo "report=$REPORT_FILE" >> $GITHUB_OUTPUT

if [ "$FAIL_ON_ERROR" = "true" ] && [ $ERRORS -gt 0 ]; then
    echo "❌ พบปัญหา $ERRORS อย่าง - Failing workflow"
    exit 1
fi

echo "✅ Quality check เสร็จสิ้น"
```

---

## 14.5 Matrix Strategy

### การใช้ Matrix สำหรับ Multi-platform Testing

```yaml
name: Matrix Testing

on: [push, pull_request]

jobs:
  test:
    name: Test (${{ matrix.os }}, Node ${{ matrix.node }})
    runs-on: ${{ matrix.os }}
    
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: ['18', '20', '21']
        include:
          # เพิ่ม experimental flag สำหรับ Node 21
          - node: '21'
            experimental: true
          # configuration พิเศษสำหรับ Ubuntu + Node 20
          - os: ubuntu-latest
            node: '20'
            coverage: true
        exclude:
          # ไม่ test Windows + Node 18
          - os: windows-latest
            node: '18'
    
    continue-on-error: ${{ matrix.experimental == true }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Run tests with coverage
        if: matrix.coverage
        run: npm run test:coverage
      
      - name: Upload coverage
        if: matrix.coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  # Matrix สำหรับ Docker build หลาย platform
  docker-build:
    name: Build Docker (${{ matrix.platform }})
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        platform:
          - linux/amd64
          - linux/arm64
          - linux/arm/v7
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build for ${{ matrix.platform }}
        uses: docker/build-push-action@v5
        with:
          platforms: ${{ matrix.platform }}
          push: false
          tags: myapp:test
```

### Dynamic Matrix

```yaml
jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate matrix from changed files
        id: set-matrix
        run: |
          # หาว่า services ไหนมีการเปลี่ยนแปลง
          CHANGED_SERVICES=()
          
          if git diff --name-only HEAD~1 | grep -q "^services/auth/"; then
            CHANGED_SERVICES+=("auth")
          fi
          
          if git diff --name-only HEAD~1 | grep -q "^services/api/"; then
            CHANGED_SERVICES+=("api")
          fi
          
          if git diff --name-only HEAD~1 | grep -q "^services/frontend/"; then
            CHANGED_SERVICES+=("frontend")
          fi
          
          # สร้าง JSON matrix
          MATRIX=$(printf '%s\n' "${CHANGED_SERVICES[@]}" | jq -R . | jq -sc '{service: .}')
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT
  
  build-services:
    needs: generate-matrix
    if: ${{ needs.generate-matrix.outputs.matrix != '{"service":[]}' }}
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.generate-matrix.outputs.matrix) }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build ${{ matrix.service }}
        run: |
          cd services/${{ matrix.service }}
          docker build -t ${{ matrix.service }}:${{ github.sha }} .
```

---

## 14.6 Concurrency Control

### ป้องกันการ Deploy ซ้อนกัน

```yaml
name: Deploy Pipeline

on:
  push:
    branches: [main]

# ยกเลิก runs เก่าถ้ามี run ใหม่เข้ามา
concurrency:
  group: deploy-production
  cancel-in-progress: false  # ไม่ยกเลิก run ที่กำลัง deploy อยู่

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

### Concurrency ตาม Branch

```yaml
name: CI Pipeline

on:
  push:
  pull_request:

# แต่ละ branch มี concurrency group ของตัวเอง
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true  # ยกเลิก run เก่าเมื่อมี push ใหม่

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  deploy-preview:
    needs: test
    if: github.event_name == 'pull_request'
    concurrency:
      group: preview-${{ github.event.pull_request.number }}
      cancel-in-progress: true
    runs-on: ubuntu-latest
    steps:
      - name: Deploy preview
        run: echo "Deploy PR preview..."
```

---

## 14.7 Environment Protection Rules

### การตั้งค่า Deployment Environments

```yaml
name: Production Deployment

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        env:
          DEPLOY_URL: ${{ secrets.STAGING_URL }}
          API_KEY: ${{ secrets.STAGING_API_KEY }}
        run: |
          ./scripts/deploy.sh staging

  approve-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production-approval  # Environment ที่ต้อง approve ก่อน
    
    steps:
      - name: Wait for approval
        run: echo "Approved! Proceeding to production deploy..."

  deploy-production:
    needs: approve-production
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        env:
          DEPLOY_URL: ${{ secrets.PROD_URL }}
          API_KEY: ${{ secrets.PROD_API_KEY }}
        run: |
          ./scripts/deploy.sh production
      
      - name: Run smoke tests
        run: |
          ./scripts/smoke-test.sh https://myapp.com
      
      - name: Notify success
        if: success()
        uses: ./.github/actions/notify-slack
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          message: "Production deployment สำเร็จ! 🚀"
          status: success
```

### การตั้งค่า Environment ใน GitHub UI

```
Settings > Environments > New environment

staging:
  - Required reviewers: ไม่จำเป็น
  - Wait timer: 0 นาที
  - Deployment branches: main, develop
  - Secrets: STAGING_URL, STAGING_API_KEY

production:
  - Required reviewers: @tech-lead, @devops-team
  - Wait timer: 5 นาที (สำหรับ deployment window)
  - Deployment branches: main เท่านั้น
  - Secrets: PROD_URL, PROD_API_KEY
```

---

## 14.8 OIDC Authentication

### ทำไมต้องใช้ OIDC?

OIDC (OpenID Connect) ช่วยให้คุณ authenticate กับ cloud providers โดยไม่ต้องเก็บ long-lived credentials ใน GitHub Secrets

**ข้อดีของ OIDC:**
- ไม่มี static credentials ที่อาจ leak
- Tokens มีอายุสั้น (short-lived)
- สามารถกำหนด scope ได้ละเอียด
- Audit trail ดีกว่า

### OIDC กับ AWS

```yaml
name: Deploy to AWS with OIDC

on:
  push:
    branches: [main]

permissions:
  id-token: write   # จำเป็นสำหรับ OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          role-session-name: GitHubActionsSession
          aws-region: ap-southeast-1
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: myapp
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service myapp \
            --force-new-deployment

  # การตั้งค่า AWS IAM Role (ต้องทำใน AWS Console หรือ Terraform)
  # Trust Policy:
  # {
  #   "Version": "2012-10-17",
  #   "Statement": [{
  #     "Effect": "Allow",
  #     "Principal": {
  #       "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
  #     },
  #     "Action": "sts:AssumeRoleWithWebIdentity",
  #     "Condition": {
  #       "StringEquals": {
  #         "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
  #         "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
  #       }
  #     }
  #   }]
  # }
```

### OIDC กับ GCP

```yaml
name: Deploy to GCP with OIDC

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
      
      - name: Authenticate to GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456789/locations/global/workloadIdentityPools/my-pool/providers/my-provider'
          service_account: 'github-actions@myproject.iam.gserviceaccount.com'
      
      - name: Setup gcloud
        uses: google-github-actions/setup-gcloud@v2
      
      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy myapp \
            --image gcr.io/myproject/myapp:${{ github.sha }} \
            --region asia-southeast1 \
            --platform managed
```

### OIDC กับ Azure

```yaml
name: Deploy to Azure with OIDC

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
      
      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Deploy to Azure Container Apps
        run: |
          az containerapp update \
            --name myapp \
            --resource-group mygroup \
            --image myregistry.azurecr.io/myapp:${{ github.sha }}
```

---

## 14.9 Caching Strategies

### Cache ขั้นสูงด้วย actions/cache

```yaml
name: Advanced Caching

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # Cache Node modules - multi-layer
      - name: Cache npm dependencies
        uses: actions/cache@v4
        id: npm-cache
        with:
          path: |
            node_modules
            ~/.npm
          key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            npm-${{ runner.os }}-
      
      # Cache Python virtualenv
      - name: Cache Python dependencies
        uses: actions/cache@v4
        id: pip-cache
        with:
          path: ~/.cache/pip
          key: pip-${{ runner.os }}-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            pip-${{ runner.os }}-
      
      # Cache Docker layers
      - name: Cache Docker layers
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: buildx-${{ runner.os }}-${{ github.sha }}
          restore-keys: |
            buildx-${{ runner.os }}-
      
      # Cache Go modules
      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: |
            ~/go/pkg/mod
            ~/.cache/go-build
          key: go-${{ runner.os }}-${{ hashFiles('go.sum') }}
          restore-keys: |
            go-${{ runner.os }}-
      
      - name: Install dependencies (if not cached)
        if: steps.npm-cache.outputs.cache-hit != 'true'
        run: npm ci
      
      - name: Build with Docker layer caching
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max
          tags: myapp:${{ github.sha }}
      
      # ย้าย cache เพื่อป้องกัน cache สะสมขนาดใหญ่
      - name: Move Docker cache
        run: |
          rm -rf /tmp/.buildx-cache
          mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

### Cache Registry (GitHub Actions Cache)

```yaml
      # ใช้ GitHub Actions cache เป็น Docker registry
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build with registry cache
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 14.10 Job Dependencies และ Artifacts

### ส่ง Artifacts ระหว่าง Jobs

```yaml
name: Multi-job Pipeline with Artifacts

on: [push]

jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.set-artifact.outputs.name }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install and build
        run: |
          npm ci
          npm run build
      
      - name: Set artifact name
        id: set-artifact
        run: echo "name=build-${{ github.run_id }}" >> $GITHUB_OUTPUT
      
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ steps.set-artifact.outputs.name }}
          path: dist/
          retention-days: 7
          compression-level: 9
  
  test-unit:
    name: Unit Tests
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: dist/
      
      - name: Run unit tests
        run: npm test
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: unit-test-results
          path: test-results/
  
  test-e2e:
    name: E2E Tests
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: dist/
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload screenshots on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-screenshots
          path: cypress/screenshots/
  
  security-scan:
    name: Security Scan
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: dist/
      
      - name: Run security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  
  deploy:
    name: Deploy
    needs: [test-unit, test-e2e, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && success()
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
          path: dist/
      
      - name: Deploy to production
        run: ./scripts/deploy.sh
```

---

## 14.11 Secrets vs Variables

### ความแตกต่างระหว่าง Secrets และ Variables

| คุณสมบัติ | Secrets | Variables |
|---------|---------|-----------|
| การซ่อนใน logs | ✅ ซ่อนอัตโนมัติ | ❌ แสดงใน logs |
| เหมาะสำหรับ | Passwords, API keys, tokens | URLs, feature flags, config |
| การ inherit | Repository > Environment | Repository > Environment |
| การเข้าถึง | `${{ secrets.NAME }}` | `${{ vars.NAME }}` |

### การใช้งาน Secrets

```yaml
name: Secrets Usage Example

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Login to database
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          DB_HOST: ${{ vars.DB_HOST }}          # Variable (ไม่ sensitive)
          DB_NAME: ${{ vars.DB_NAME }}          # Variable (ไม่ sensitive)
        run: |
          # ใช้ secret ผ่าน environment variable
          psql "postgresql://$DB_HOST/$DB_NAME" \
            -U admin \
            -W "$DB_PASSWORD" \
            -c "SELECT version();"
      
      - name: Deploy with API key
        run: |
          curl -X POST https://api.myapp.com/deploy \
            -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"environment": "production", "version": "${{ github.sha }}"}'
      
      # ข้อควรระวัง: อย่า echo secrets โดยตรง
      - name: WRONG - อย่าทำแบบนี้
        run: |
          echo "${{ secrets.SECRET_KEY }}"  # ❌ อย่าทำ! 
      
      - name: CORRECT - ใช้ผ่าน env variable
        env:
          MY_SECRET: ${{ secrets.SECRET_KEY }}
        run: |
          # ใช้ $MY_SECRET แทน
          ./script-that-uses-secret.sh
```

### Organization Secrets

```yaml
# Organization-level secrets สามารถใช้ได้ใน repositories ที่อนุญาต
# Settings > Secrets and variables > Actions > Organization secrets

jobs:
  use-org-secret:
    runs-on: ubuntu-latest
    steps:
      - name: Use organization secret
        env:
          ORG_DEPLOY_KEY: ${{ secrets.ORG_SHARED_DEPLOY_KEY }}
        run: |
          echo "Using organization secret..."
```

---

## 14.12 GitHub Packages

### Publish Docker Image ไปยัง GitHub Container Registry

```yaml
name: Publish to GitHub Packages

on:
  release:
    types: [published]
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to GitHub Container Registry
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
            type=sha
      
      - name: Build and push Docker image
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true
          sbom: true
      
      - name: Generate artifact attestation
        uses: actions/attest-build-provenance@v1
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.push.outputs.digest }}
          push-to-registry: true

  publish-npm:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://npm.pkg.github.com'
          scope: '@myorg'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build package
        run: npm run build
      
      - name: Publish to GitHub Packages
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 14.13 Self-hosted Runners

### การติดตั้ง Self-hosted Runner

```bash
# 1. ไปที่ Settings > Actions > Runners > New self-hosted runner
# 2. เลือก OS และ Architecture

# ตัวอย่างสำหรับ Linux x64
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure runner
./config.sh --url https://github.com/myorg/myrepo --token YOUR_TOKEN

# Install as service
sudo ./svc.sh install
sudo ./svc.sh start
```

### Docker สำหรับ Self-hosted Runner

```dockerfile
# Dockerfile.runner
FROM ubuntu:22.04

ARG RUNNER_VERSION="2.311.0"

RUN apt-get update && apt-get install -y \
    curl \
    git \
    jq \
    sudo \
    docker.io \
    && rm -rf /var/lib/apt/lists/*

# สร้าง user สำหรับ runner
RUN useradd -m -s /bin/bash runner && \
    usermod -aG docker runner && \
    echo "runner ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

USER runner
WORKDIR /home/runner

# ดาวน์โหลด runner
RUN curl -o actions-runner-linux-x64.tar.gz -L \
    "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz" && \
    tar xzf actions-runner-linux-x64.tar.gz && \
    rm actions-runner-linux-x64.tar.gz

COPY entrypoint.sh /home/runner/entrypoint.sh
ENTRYPOINT ["/home/runner/entrypoint.sh"]
```

```bash
# entrypoint.sh
#!/bin/bash
set -e

./config.sh \
  --url "$REPO_URL" \
  --token "$RUNNER_TOKEN" \
  --name "${RUNNER_NAME:-docker-runner}" \
  --labels "${RUNNER_LABELS:-self-hosted,linux,x64}" \
  --unattended \
  --replace

cleanup() {
    ./config.sh remove --unattended --token "$RUNNER_TOKEN"
}

trap 'cleanup; exit 130' INT
trap 'cleanup; exit 143' TERM

./run.sh &
wait $!
```

### Kubernetes Runner (actions-runner-controller)

```yaml
# runner-deployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: myapp-runners
  namespace: github-actions
spec:
  replicas: 3
  template:
    spec:
      repository: myorg/myrepo
      labels:
        - self-hosted
        - linux
        - x64
        - k8s
      resources:
        requests:
          cpu: "500m"
          memory: "512Mi"
        limits:
          cpu: "2"
          memory: "2Gi"
      env:
        - name: DOCKER_HOST
          value: tcp://localhost:2376
      sidecarContainers:
        - name: dind
          image: docker:23-dind
          securityContext:
            privileged: true
          volumeMounts:
            - name: work
              mountPath: /workspace

---
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: myapp-runners-autoscaler
spec:
  scaleTargetRef:
    name: myapp-runners
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
      repositoryNames:
        - myrepo
```

### ใช้งาน Self-hosted Runner ใน Workflow

```yaml
jobs:
  build-on-self-hosted:
    runs-on: [self-hosted, linux, x64, high-memory]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build large application
        run: |
          # งานที่ต้องใช้ resources มาก
          make build-all
  
  gpu-training:
    runs-on: [self-hosted, gpu]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Train ML model
        run: |
          python train.py --gpu=0
```

---

## 14.14 Monitoring Workflows

### ดู Workflow Metrics

```yaml
name: Workflow with Monitoring

on: [push]

jobs:
  monitored-build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Start timing
        id: start
        run: echo "start=$(date +%s)" >> $GITHUB_OUTPUT
      
      - name: Build application
        run: |
          time npm run build
      
      - name: Calculate duration
        if: always()
        run: |
          END=$(date +%s)
          START=${{ steps.start.outputs.start }}
          DURATION=$((END - START))
          echo "Build took ${DURATION} seconds"
          
          # ส่ง metric ไปยัง monitoring system
          curl -X POST https://metrics.myapp.com/api/metrics \
            -H "Content-Type: application/json" \
            -d "{
              \"metric\": \"github_actions_build_duration\",
              \"value\": $DURATION,
              \"tags\": {
                \"repo\": \"${{ github.repository }}\",
                \"workflow\": \"${{ github.workflow }}\",
                \"branch\": \"${{ github.ref_name }}\"
              }
            }"
      
      - name: Report workflow status
        if: always()
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_TITLE: "Build ${{ job.status }}"
          SLACK_MESSAGE: "Repository: ${{ github.repository }}"
          SLACK_COLOR: ${{ job.status }}
```

### GitHub Actions Dashboard

```yaml
name: Collect Workflow Metrics

on:
  workflow_run:
    workflows: ['*']
    types: [completed]

jobs:
  collect-metrics:
    runs-on: ubuntu-latest
    
    steps:
      - name: Get workflow run details
        id: details
        uses: octokit/request-action@v2.x
        with:
          route: GET /repos/${{ github.repository }}/actions/runs/${{ github.event.workflow_run.id }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Send metrics to DataDog
        env:
          DD_API_KEY: ${{ secrets.DATADOG_API_KEY }}
          RUN_DATA: ${{ steps.details.outputs.data }}
        run: |
          DURATION=$(echo $RUN_DATA | jq '.run_duration_ms // 0')
          STATUS=$(echo $RUN_DATA | jq -r '.conclusion')
          WORKFLOW=$(echo $RUN_DATA | jq -r '.name')
          
          curl -X POST "https://api.datadoghq.com/api/v1/series" \
            -H "DD-API-KEY: $DD_API_KEY" \
            -H "Content-Type: application/json" \
            -d "{
              \"series\": [{
                \"metric\": \"github.workflow.duration\",
                \"points\": [[$(date +%s), $DURATION]],
                \"tags\": [
                  \"workflow:$WORKFLOW\",
                  \"status:$STATUS\",
                  \"repo:${{ github.repository }}\"
                ]
              }]
            }"
```

---

## 14.15 Advanced Workflow Patterns

### Fan-out / Fan-in Pattern

```yaml
name: Fan-out Fan-in Pattern

on: [push]

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      test-chunks: ${{ steps.chunk.outputs.chunks }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Split tests into chunks
        id: chunk
        run: |
          # หาทุก test files และแบ่งเป็น 4 กลุ่ม
          TEST_FILES=$(find . -name "*.test.js" | sort)
          CHUNKS=$(echo "$TEST_FILES" | split -l $(( $(echo "$TEST_FILES" | wc -l) / 4 + 1 )) - /tmp/chunk-)
          
          MATRIX='{"chunk":['
          for i in 1 2 3 4; do
            MATRIX+="$i,"
          done
          MATRIX="${MATRIX%,}]}"
          
          echo "chunks=$MATRIX" >> $GITHUB_OUTPUT
  
  test-parallel:
    needs: prepare
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.prepare.outputs.test-chunks) }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run test chunk ${{ matrix.chunk }}
        run: |
          # รัน test chunk ที่ ${{ matrix.chunk }}
          npm test -- --shard=${{ matrix.chunk }}/4
      
      - name: Upload chunk results
        uses: actions/upload-artifact@v4
        with:
          name: test-results-chunk-${{ matrix.chunk }}
          path: test-results/
  
  merge-results:
    needs: test-parallel
    runs-on: ubuntu-latest
    
    steps:
      - name: Download all results
        uses: actions/download-artifact@v4
        with:
          pattern: test-results-chunk-*
          merge-multiple: true
          path: all-results/
      
      - name: Merge test reports
        run: |
          npm run merge-reports all-results/
      
      - name: Publish test summary
        uses: dorny/test-reporter@v1
        with:
          name: 'Test Results'
          path: 'merged-results/*.xml'
          reporter: jest-junit
```

---

## แบบฝึกหัด Part 14

### แบบฝึกหัดที่ 1: สร้าง Reusable Workflow

สร้าง reusable workflow สำหรับ Docker build และ push ที่รองรับ:
1. หลาย registry (GHCR, ECR, Docker Hub)
2. Multi-platform builds (amd64, arm64)
3. Automatic versioning จาก git tags
4. SBOM generation

### แบบฝึกหัดที่ 2: Custom JavaScript Action

สร้าง custom action ที่:
1. ตรวจสอบ PR title format (Conventional Commits)
2. ตรวจสอบว่ามี Jira ticket number ใน branch name
3. Comment กลับ PR ถ้าไม่ผ่าน

### แบบฝึกหัดที่ 3: OIDC Authentication

ตั้งค่า OIDC authentication กับ AWS:
1. สร้าง IAM OIDC Provider
2. สร้าง IAM Role พร้อม trust policy
3. สร้าง workflow ที่ deploy ไปยัง S3 โดยใช้ OIDC
4. จำกัด access เฉพาะ `main` branch

### แบบฝึกหัดที่ 4: Matrix Build Optimization

ออกแบบ matrix strategy ที่:
1. Build 3 ภาษา (Python, Node.js, Go) บน 3 OS
2. Skip การ test บน OS ที่ไม่จำเป็น
3. เพิ่ม experimental versions
4. รวม test results จากทุก matrix

### แบบฝึกหัดที่ 5: Production Deployment Pipeline

สร้าง complete deployment pipeline ที่มี:
1. Build stage พร้อม artifact caching
2. Parallel testing (unit, integration, e2e)
3. Security scanning
4. Staging deployment พร้อม smoke tests
5. Manual approval สำหรับ production
6. Production deployment พร้อม rollback capability
7. Slack notifications ทุก stage

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- **Reusable Workflows**: สร้าง workflow ที่ใช้ซ้ำได้ทั่วทั้งองค์กร
- **Composite Actions**: รวม steps เป็น action เดียว
- **Custom Actions**: สร้าง JavaScript และ Docker actions
- **Matrix Strategy**: test หลาย platform พร้อมกัน
- **Concurrency Control**: ป้องกัน deployment ซ้อนกัน
- **OIDC Authentication**: authenticate กับ cloud โดยไม่ใช้ static credentials
- **Caching**: เพิ่มความเร็ว pipeline ด้วย cache ที่ดี
- **Artifacts**: ส่งข้อมูลระหว่าง jobs
- **GitHub Packages**: publish packages และ Docker images
- **Self-hosted Runners**: รัน workflow บน hardware ของตัวเอง

ในบทถัดไป เราจะมาดู GitLab CI/CD ซึ่งมีความสามารถที่คล้ายกันแต่มีแนวทางที่แตกต่างออกไป

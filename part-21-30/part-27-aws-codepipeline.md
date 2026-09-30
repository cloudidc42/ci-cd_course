# Part 27: Cloud CI/CD — AWS CodePipeline

## สารบัญ
1. [AWS CI/CD Ecosystem Overview](#aws-cicd-ecosystem-overview)
2. [AWS CodeCommit](#aws-codecommit)
3. [AWS CodeBuild](#aws-codebuild)
4. [AWS CodeDeploy](#aws-codedeploy)
5. [AWS CodePipeline](#aws-codepipeline)
6. [สร้าง Pipeline ใน AWS Console](#สร้าง-pipeline-ใน-aws-console)
7. [buildspec.yml](#buildspecyml)
8. [Deployment ไปยัง EC2/ECS/Lambda](#deployment-ไปยัง-ec2ecslambda)
9. [IAM Roles สำหรับ CI/CD](#iam-roles-สำหรับ-cicd)
10. [Environment Variables ใน CodeBuild](#environment-variables-ใน-codebuild)
11. [Artifacts ใน S3](#artifacts-ใน-s3)
12. [Notifications ด้วย SNS](#notifications-ด้วย-sns)
13. [เปรียบเทียบกับ GitHub Actions](#เปรียบเทียบกับ-github-actions)
14. [Infrastructure as Code ด้วย CloudFormation](#infrastructure-as-code-ด้วย-cloudformation)
15. [แบบฝึกหัด](#แบบฝึกหัด)

---

## AWS CI/CD Ecosystem Overview

Amazon Web Services (AWS) มีชุดบริการ CI/CD ที่ครบวงจรเรียกว่า **AWS Developer Tools** ซึ่งประกอบด้วย:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AWS CI/CD Ecosystem                         │
│                                                                 │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐          │
│  │ CodeCommit  │   │  CodeBuild  │   │ CodeDeploy  │          │
│  │             │   │             │   │             │          │
│  │ Source Code │──▶│ Build/Test  │──▶│   Deploy    │          │
│  │ Repository  │   │             │   │             │          │
│  └─────────────┘   └─────────────┘   └─────────────┘          │
│         │                 │                 │                  │
│         └─────────────────┴─────────────────┘                  │
│                           │                                     │
│                    ┌──────▼──────┐                             │
│                    │CodePipeline │                             │
│                    │             │                             │
│                    │ Orchestrate │                             │
│                    │ Everything  │                             │
│                    └─────────────┘                             │
│                                                                 │
│  Other Services:                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │CodeArtifact│ │CodeGuru │ │ CodeStar │ │ CloudWatch│      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### บริการหลัก

| บริการ | หน้าที่ |
|--------|---------|
| **CodeCommit** | Git repository hosting (คล้าย GitHub/GitLab) |
| **CodeBuild** | Build and test service (คล้าย GitHub Actions runner) |
| **CodeDeploy** | Automated deployment service |
| **CodePipeline** | Orchestration และ automation ของ pipeline |
| **CodeArtifact** | Artifact repository (npm, Maven, pip packages) |
| **CodeGuru** | Code review และ performance recommendations |

### เปรียบเทียบกับ GitHub Actions

```
GitHub Actions:
  GitHub Repository → GitHub Actions Runner → Deploy Target

AWS CodePipeline:
  CodeCommit/GitHub → CodeBuild → CodeDeploy → Deploy Target
         Source         Build         Deploy
```

---

## AWS CodeCommit

> **หมายเหตุ**: AWS CodeCommit ถูก deprecated ในปี 2024 สำหรับลูกค้าใหม่ แต่ยังใช้ได้กับลูกค้าเก่า ปัจจุบัน AWS แนะนำให้ใช้ GitHub, GitLab หรือ Bitbucket แทน

### สร้าง CodeCommit Repository

```bash
# สร้าง repository
aws codecommit create-repository \
  --repository-name my-app \
  --repository-description "My application repository"

# แสดงข้อมูล
aws codecommit get-repository \
  --repository-name my-app

# List repositories
aws codecommit list-repositories

# ลบ repository
aws codecommit delete-repository \
  --repository-name my-app
```

### Clone Repository

```bash
# ติดตั้ง git-remote-codecommit
pip install git-remote-codecommit

# Clone
git clone codecommit::ap-southeast-1://my-app

# หรือใช้ HTTPS URL
# ต้องตั้งค่า Git credentials ใน IAM ก่อน
git clone https://git-codecommit.ap-southeast-1.amazonaws.com/v1/repos/my-app
```

### Trigger สำหรับ CodePipeline

```json
// cloudformation/codecommit-trigger.json
{
  "repositoryName": "my-app",
  "triggers": [
    {
      "name": "trigger-main-branch",
      "destinationArn": "arn:aws:codepipeline:ap-southeast-1:123456789:my-pipeline",
      "branches": ["main"],
      "events": ["push"]
    }
  ]
}
```

---

## AWS CodeBuild

CodeBuild เป็น fully managed build service ที่รัน build commands ใน container environment

### Create Build Project ด้วย AWS CLI

```bash
# สร้าง build project
aws codebuild create-project \
  --name my-app-build \
  --source type=CODECOMMIT,location=https://git-codecommit.ap-southeast-1.amazonaws.com/v1/repos/my-app \
  --environment type=LINUX_CONTAINER,computeType=BUILD_GENERAL1_SMALL,image=aws/codebuild/standard:7.0 \
  --service-role arn:aws:iam::123456789:role/CodeBuildServiceRole \
  --artifacts type=NO_ARTIFACTS

# Start build manually
aws codebuild start-build \
  --project-name my-app-build

# ดู build logs
aws codebuild batch-get-builds \
  --ids build-id

# List builds
aws codebuild list-builds-for-project \
  --project-name my-app-build
```

### Compute Types

| Compute Type | vCPU | Memory | Disk |
|---|---|---|---|
| BUILD_GENERAL1_SMALL | 3 | 4 GB | 64 GB |
| BUILD_GENERAL1_MEDIUM | 7 | 16 GB | 128 GB |
| BUILD_GENERAL1_LARGE | 36 | 72 GB | 128 GB |
| BUILD_GENERAL1_XLARGE | 72 | 144 GB | 128 GB |

### Pre-built Images ที่ AWS มีให้

```
aws/codebuild/standard:7.0          # Ubuntu 22.04 (Python, Node.js, Java, Ruby, Go)
aws/codebuild/amazonlinux2-x86_64-standard:5.0  # Amazon Linux 2
aws/codebuild/amazonlinux-aarch64-standard:3.0   # ARM64
aws/codebuild/windows-base:2022-1.0              # Windows Server 2022
```

---

## buildspec.yml

`buildspec.yml` คือไฟล์ที่บอก CodeBuild ว่าต้องทำอะไรในแต่ละขั้นตอน

### โครงสร้าง buildspec.yml

```yaml
# buildspec.yml
version: 0.2

# กำหนด environment variables
env:
  variables:
    DOCKER_BUILDKIT: "1"
    NODE_ENV: "production"
  parameter-store:
    DB_PASSWORD: /myapp/prod/db-password
    API_KEY: /myapp/prod/api-key
  secrets-manager:
    GITHUB_TOKEN: myapp/github-token:token

# กำหนด phases
phases:
  install:
    runtime-versions:
      nodejs: 18
      java: corretto17
    commands:
      - echo "Installing dependencies..."
      - npm install -g yarn
      - pip install awscli --upgrade

  pre_build:
    commands:
      - echo "Running pre-build steps..."
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/my-app
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:=latest}
      - echo "Image tag: $IMAGE_TAG"

  build:
    commands:
      - echo "Building the application..."
      - yarn install --frozen-lockfile
      - yarn test
      - yarn build
      - echo "Building Docker image..."
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$IMAGE_TAG

  post_build:
    commands:
      - echo "Running post-build steps..."
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      - echo Writing image definitions file...
      - printf '[{"name":"my-app","imageUri":"%s"}]' $REPOSITORY_URI:$IMAGE_TAG > imagedefinitions.json
      - cat imagedefinitions.json

# Artifacts ที่ต้องการส่งออก
artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json
  discard-paths: yes

# Cache สำหรับ เร็วขึ้นใน build ถัดไป
cache:
  paths:
    - node_modules/**/*
    - /root/.cache/pip/**/*

# Reports สำหรับ test results
reports:
  jest-reports:
    files:
      - "coverage/lcov.info"
      - "test-results.xml"
    file-format: JUNITXML
    base-directory: coverage
```

### buildspec.yml สำหรับ Node.js API

```yaml
# buildspec-nodejs.yml
version: 0.2

env:
  variables:
    NODE_VERSION: "18"
  parameter-store:
    NPM_TOKEN: /myapp/npm-token

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "Node version $(node -v)"
      - echo "NPM version $(npm -v)"
      - echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > ~/.npmrc

  pre_build:
    commands:
      - echo "Installing dependencies..."
      - npm ci

  build:
    commands:
      - echo "Running tests..."
      - npm test -- --coverage --ci
      - echo "Building application..."
      - npm run build
      - echo "Running security audit..."
      - npm audit --audit-level=high

  post_build:
    commands:
      - echo "Build completed on `date`"

artifacts:
  files:
    - dist/**/*
    - package.json
    - package-lock.json
  base-directory: .

reports:
  test-coverage:
    files:
      - "coverage/clover.xml"
    file-format: CLOVERXML
  unit-tests:
    files:
      - "test-results/junit.xml"
    file-format: JUNITXML

cache:
  paths:
    - node_modules/**/*
```

### buildspec.yml สำหรับ Python

```yaml
# buildspec-python.yml
version: 0.2

env:
  variables:
    PYTHON_VERSION: "3.11"

phases:
  install:
    runtime-versions:
      python: 3.11
    commands:
      - pip install --upgrade pip
      - pip install poetry

  pre_build:
    commands:
      - poetry install

  build:
    commands:
      - echo "Running linting..."
      - poetry run flake8 src/ --max-line-length=120
      - poetry run black --check src/
      - echo "Running type checking..."
      - poetry run mypy src/
      - echo "Running tests..."
      - poetry run pytest tests/ -v --cov=src --cov-report=xml --cov-report=html
      - echo "Building package..."
      - poetry build

  post_build:
    commands:
      - echo "Build completed"
      - ls -la dist/

artifacts:
  files:
    - dist/*.whl
    - dist/*.tar.gz
  base-directory: .

reports:
  pytest-reports:
    files:
      - "pytest-report.xml"
    file-format: JUNITXML
  coverage-reports:
    files:
      - "coverage.xml"
    file-format: COBERTURAXML

cache:
  paths:
    - /root/.cache/pip/**/*
    - .venv/**/*
```

### buildspec.yml สำหรับ Docker Build & Push to ECR

```yaml
# buildspec-docker.yml
version: 0.2

env:
  variables:
    AWS_DEFAULT_REGION: ap-southeast-1
    APP_NAME: my-app
  exported-variables:
    - IMAGE_TAG
    - REPOSITORY_URI

phases:
  pre_build:
    commands:
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | \
          docker login --username AWS --password-stdin \
          $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      
      - echo "Setting up variables..."
      - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$APP_NAME
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - BUILD_DATE=$(date +%Y%m%d%H%M%S)
      - IMAGE_TAG=${COMMIT_HASH}-${BUILD_DATE}
      - echo "Repository URI: $REPOSITORY_URI"
      - echo "Image tag: $IMAGE_TAG"

  build:
    commands:
      - echo "Build started on `date`"
      - echo "Building Docker image..."
      
      # Multi-stage build
      - |
        docker build \
          --build-arg NODE_VERSION=18 \
          --build-arg BUILD_DATE=$(date -u +'%Y-%m-%dT%H:%M:%SZ') \
          --build-arg VCS_REF=$CODEBUILD_RESOLVED_SOURCE_VERSION \
          --label "org.opencontainers.image.created=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
          --label "org.opencontainers.image.revision=$CODEBUILD_RESOLVED_SOURCE_VERSION" \
          -t $REPOSITORY_URI:latest \
          -t $REPOSITORY_URI:$IMAGE_TAG \
          .
      
      - echo "Running container structure tests..."
      - |
        docker run --rm \
          -v /var/run/docker.sock:/var/run/docker.sock \
          gcr.io/gcp-runtimes/container-structure-test:latest \
          test \
          --image $REPOSITORY_URI:$IMAGE_TAG \
          --config container-structure-test.yaml

  post_build:
    commands:
      - echo "Pushing Docker image..."
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$IMAGE_TAG
      
      - echo "Creating deployment artifacts..."
      - |
        printf '[{"name":"%s","imageUri":"%s"}]' \
          $APP_NAME \
          $REPOSITORY_URI:$IMAGE_TAG \
          > imagedefinitions.json
      
      - echo "Creating task definition..."
      - |
        cat > taskdef.json << EOF
        {
          "family": "$APP_NAME",
          "containerDefinitions": [
            {
              "name": "$APP_NAME",
              "image": "<IMAGE1_NAME>",
              "cpu": 256,
              "memory": 512,
              "portMappings": [
                {
                  "containerPort": 3000,
                  "protocol": "tcp"
                }
              ]
            }
          ],
          "requiresCompatibilities": ["FARGATE"],
          "networkMode": "awsvpc",
          "cpu": "256",
          "memory": "512"
        }
        EOF
      
      - echo "Build completed successfully"

artifacts:
  files:
    - imagedefinitions.json
    - taskdef.json
    - appspec.yaml
```

---

## Deployment ไปยัง EC2/ECS/Lambda

### Deploy ไปยัง EC2 ด้วย CodeDeploy

**appspec.yml สำหรับ EC2:**

```yaml
# appspec.yml (สำหรับ EC2)
version: 0.0
os: linux

files:
  - source: /app
    destination: /home/ec2-user/app
  - source: /config
    destination: /home/ec2-user/app/config

hooks:
  BeforeInstall:
    - location: scripts/stop_application.sh
      timeout: 60
      runas: ec2-user
  
  Install:
    - location: scripts/install_dependencies.sh
      timeout: 120
      runas: ec2-user
  
  AfterInstall:
    - location: scripts/configure_application.sh
      timeout: 60
      runas: ec2-user
  
  ApplicationStart:
    - location: scripts/start_application.sh
      timeout: 60
      runas: ec2-user
  
  ValidateService:
    - location: scripts/validate_service.sh
      timeout: 60
      runas: ec2-user
```

**Deploy scripts:**

```bash
# scripts/stop_application.sh
#!/bin/bash
echo "Stopping application..."
if systemctl is-active --quiet myapp; then
  systemctl stop myapp
  echo "Application stopped"
else
  echo "Application was not running"
fi

# scripts/install_dependencies.sh
#!/bin/bash
echo "Installing dependencies..."
cd /home/ec2-user/app
npm ci --production
echo "Dependencies installed"

# scripts/start_application.sh
#!/bin/bash
echo "Starting application..."
cd /home/ec2-user/app
systemctl start myapp
sleep 5

# ตรวจสอบว่า start สำเร็จ
if systemctl is-active --quiet myapp; then
  echo "Application started successfully"
  exit 0
else
  echo "Failed to start application"
  exit 1
fi

# scripts/validate_service.sh
#!/bin/bash
echo "Validating service..."

# รอให้ service พร้อม
sleep 10

# ตรวจสอบ health endpoint
for i in {1..5}; do
  HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)
  
  if [ "$HTTP_STATUS" = "200" ]; then
    echo "Service is healthy (HTTP $HTTP_STATUS)"
    exit 0
  fi
  
  echo "Health check failed (HTTP $HTTP_STATUS), retrying in 10s..."
  sleep 10
done

echo "Service health check failed after multiple attempts"
exit 1
```

### Deploy ไปยัง ECS ด้วย CodeDeploy Blue/Green

**appspec.yml สำหรับ ECS:**

```yaml
# appspec.yml (สำหรับ ECS Blue/Green)
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "<TASK_DEFINITION>"
        LoadBalancerInfo:
          ContainerName: "my-app"
          ContainerPort: 3000
        PlatformVersion: "LATEST"

Hooks:
  - BeforeInstall: "arn:aws:lambda:ap-southeast-1:123456789:function:pre-deploy-check"
  - AfterInstall: "arn:aws:lambda:ap-southeast-1:123456789:function:smoke-test"
  - AfterAllowTestTraffic: "arn:aws:lambda:ap-southeast-1:123456789:function:integration-test"
  - BeforeAllowTraffic: "arn:aws:lambda:ap-southeast-1:123456789:function:final-check"
  - AfterAllowTraffic: "arn:aws:lambda:ap-southeast-1:123456789:function:post-deploy"
```

**ECS Task Definition:**

```json
// taskdef.json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<IMAGE1_NAME>",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:myapp/db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### Deploy ไปยัง Lambda

**buildspec.yml สำหรับ Lambda:**

```yaml
# buildspec-lambda.yml
version: 0.2

env:
  variables:
    FUNCTION_NAME: my-lambda-function
    S3_BUCKET: my-lambda-artifacts

phases:
  install:
    runtime-versions:
      python: 3.11
    commands:
      - pip install --upgrade pip

  build:
    commands:
      - echo "Running tests..."
      - python -m pytest tests/ -v
      
      - echo "Creating deployment package..."
      - mkdir -p dist
      - cp -r src/ dist/
      - pip install -r requirements.txt -t dist/
      - cd dist && zip -r ../function.zip . && cd ..
      - echo "Package size: $(du -sh function.zip)"

  post_build:
    commands:
      - echo "Uploading to S3..."
      - S3_KEY=lambda/${CODEBUILD_RESOLVED_SOURCE_VERSION}/function.zip
      - aws s3 cp function.zip s3://$S3_BUCKET/$S3_KEY
      
      - echo "Updating Lambda function..."
      - |
        aws lambda update-function-code \
          --function-name $FUNCTION_NAME \
          --s3-bucket $S3_BUCKET \
          --s3-key $S3_KEY
      
      - echo "Waiting for update..."
      - aws lambda wait function-updated --function-name $FUNCTION_NAME
      
      - echo "Publishing new version..."
      - VERSION=$(aws lambda publish-version --function-name $FUNCTION_NAME --query Version --output text)
      - echo "Published version: $VERSION"
      
      - echo "Updating alias to new version..."
      - |
        aws lambda update-alias \
          --function-name $FUNCTION_NAME \
          --name production \
          --function-version $VERSION

artifacts:
  files:
    - function.zip
```

---

## สร้าง Pipeline ใน AWS Console

### ขั้นตอนการสร้าง Pipeline ผ่าน Console

1. **เข้าไปที่ CodePipeline Console**
   - เปิด AWS Console > Developer Tools > CodePipeline
   - คลิก "Create pipeline"

2. **Step 1: Pipeline settings**
   ```
   Pipeline name: my-app-pipeline
   Pipeline type: V2 (แนะนำ - รองรับ triggers ที่ยืดหยุ่น)
   Execution mode: Superseded
   Service role: New service role (หรือ existing)
   Role name: AWSCodePipelineServiceRole-ap-southeast-1-my-app-pipeline
   Advanced settings:
     Artifact store: Default location (S3 bucket สร้างอัตโนมัติ)
     Encryption key: Default AWS Managed Key
   ```

3. **Step 2: Source Stage**
   ```
   Source provider: GitHub (Version 2) / AWS CodeCommit
   Connection: สร้าง connection ใหม่หรือเลือกที่มีอยู่
   Repository name: myorg/my-app
   Branch name: main
   Change detection options: GitHub webhooks (แนะนำ)
   Output artifact format: CodePipeline default
   ```

4. **Step 3: Build Stage**
   ```
   Build provider: AWS CodeBuild
   Region: Asia Pacific (Singapore)
   Project name: my-app-build (สร้างใหม่หรือเลือกที่มีอยู่)
   Build type: Single build
   ```

5. **Step 4: Deploy Stage**
   ```
   Deploy provider: Amazon ECS (Blue/Green)
   Region: Asia Pacific (Singapore)
   Application name: my-app (CodeDeploy application)
   Deployment group: my-app-prod
   Amazon ECS task definition: taskdef.json (artifact path)
   AWS CodeDeploy AppSpec file: appspec.yaml (artifact path)
   Dynamically update task definition image: IMAGE1_NAME = BuildArtifact::imagedefinitions.json
   ```

### สร้าง Pipeline ด้วย AWS CLI

```bash
# สร้าง pipeline configuration
cat > pipeline.json << 'EOF'
{
  "pipeline": {
    "name": "my-app-pipeline",
    "roleArn": "arn:aws:iam::123456789:role/AWSCodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-codepipeline-artifacts-bucket"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeStarSourceConnection",
              "version": "1"
            },
            "configuration": {
              "ConnectionArn": "arn:aws:codestar-connections:ap-southeast-1:123456789:connection/abc123",
              "FullRepositoryId": "myorg/my-app",
              "BranchName": "main",
              "DetectionMode": "EVENTS"
            },
            "outputArtifacts": [
              {"name": "SourceArtifact"}
            ],
            "runOrder": 1
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "Build",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "my-app-build"
            },
            "inputArtifacts": [
              {"name": "SourceArtifact"}
            ],
            "outputArtifacts": [
              {"name": "BuildArtifact"}
            ],
            "runOrder": 1
          }
        ]
      },
      {
        "name": "Deploy",
        "actions": [
          {
            "name": "Deploy",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeployToECS",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "my-app",
              "DeploymentGroupName": "my-app-prod",
              "TaskDefinitionTemplateArtifact": "BuildArtifact",
              "TaskDefinitionTemplatePath": "taskdef.json",
              "AppSpecTemplateArtifact": "BuildArtifact",
              "AppSpecTemplatePath": "appspec.yaml",
              "Image1ArtifactName": "BuildArtifact",
              "Image1ContainerName": "IMAGE1_NAME"
            },
            "inputArtifacts": [
              {"name": "BuildArtifact"}
            ],
            "runOrder": 1
          }
        ]
      }
    ]
  }
}
EOF

# สร้าง pipeline
aws codepipeline create-pipeline --cli-input-json file://pipeline.json

# Start pipeline manually
aws codepipeline start-pipeline-execution --name my-app-pipeline

# ดู pipeline state
aws codepipeline get-pipeline-state --name my-app-pipeline

# ดู execution history
aws codepipeline list-pipeline-executions --pipeline-name my-app-pipeline
```

### Multi-Stage Pipeline ที่ซับซ้อน

```bash
cat > advanced-pipeline.json << 'EOF'
{
  "pipeline": {
    "name": "my-app-advanced-pipeline",
    "roleArn": "arn:aws:iam::123456789:role/AWSCodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-codepipeline-artifacts"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "AppSource",
            "actionTypeId": {
              "category": "Source",
              "owner": "ThirdParty",
              "provider": "GitHub",
              "version": "1"
            },
            "configuration": {
              "Owner": "myorg",
              "Repo": "my-app",
              "Branch": "main",
              "OAuthToken": "{{resolve:secretsmanager:github-token:SecretString:token}}"
            },
            "outputArtifacts": [{"name": "SourceArtifact"}]
          }
        ]
      },
      {
        "name": "UnitTest",
        "actions": [
          {
            "name": "RunTests",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "my-app-test"
            },
            "inputArtifacts": [{"name": "SourceArtifact"}],
            "outputArtifacts": [{"name": "TestArtifact"}]
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "BuildImage",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "configuration": {
              "ProjectName": "my-app-build"
            },
            "inputArtifacts": [{"name": "SourceArtifact"}],
            "outputArtifacts": [{"name": "BuildArtifact"}]
          }
        ]
      },
      {
        "name": "DeployStaging",
        "actions": [
          {
            "name": "DeployToStaging",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeployToECS",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "my-app",
              "DeploymentGroupName": "my-app-staging"
            },
            "inputArtifacts": [{"name": "BuildArtifact"}]
          }
        ]
      },
      {
        "name": "ApprovalForProduction",
        "actions": [
          {
            "name": "ManualApproval",
            "actionTypeId": {
              "category": "Approval",
              "owner": "AWS",
              "provider": "Manual",
              "version": "1"
            },
            "configuration": {
              "NotificationArn": "arn:aws:sns:ap-southeast-1:123456789:deploy-approvals",
              "CustomData": "Please review and approve deployment to production",
              "ExternalEntityLink": "https://staging.example.com"
            }
          }
        ]
      },
      {
        "name": "DeployProduction",
        "actions": [
          {
            "name": "DeployToProduction",
            "actionTypeId": {
              "category": "Deploy",
              "owner": "AWS",
              "provider": "CodeDeployToECS",
              "version": "1"
            },
            "configuration": {
              "ApplicationName": "my-app",
              "DeploymentGroupName": "my-app-production"
            },
            "inputArtifacts": [{"name": "BuildArtifact"}]
          }
        ]
      }
    ]
  }
}
EOF

aws codepipeline create-pipeline --cli-input-json file://advanced-pipeline.json
```

---

## IAM Roles สำหรับ CI/CD

### CodePipeline Service Role

```json
// iam/codepipeline-service-role-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetObjectVersion",
        "s3:GetBucketVersioning",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-codepipeline-artifacts",
        "arn:aws:s3:::my-codepipeline-artifacts/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:BatchGetBuilds",
        "codebuild:StartBuild",
        "codebuild:StopBuild"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codedeploy:CreateDeployment",
        "codedeploy:GetApplication",
        "codedeploy:GetApplicationRevision",
        "codedeploy:GetDeployment",
        "codedeploy:GetDeploymentConfig",
        "codedeploy:RegisterApplicationRevision"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeServices",
        "ecs:UpdateService"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "*",
      "Condition": {
        "StringEqualsIfExists": {
          "iam:PassedToService": [
            "ecs-tasks.amazonaws.com"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "codestar-connections:UseConnection"
      ],
      "Resource": "arn:aws:codestar-connections:*:*:connection/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:ap-southeast-1:123456789:*"
    }
  ]
}
```

### CodeBuild Service Role

```json
// iam/codebuild-service-role-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:GetObjectVersion",
        "s3:GetBucketVersioning",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-codepipeline-artifacts",
        "arn:aws:s3:::my-codepipeline-artifacts/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:CompleteLayerUpload",
        "ecr:GetAuthorizationToken",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:ap-southeast-1:123456789:parameter/myapp/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:123456789:secret:myapp/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "codebuild:CreateReportGroup",
        "codebuild:CreateReport",
        "codebuild:UpdateReport",
        "codebuild:BatchPutTestCases",
        "codebuild:BatchPutCodeCoverages"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:ap-southeast-1:123456789:key/*"
    }
  ]
}
```

### สร้าง IAM Role ด้วย CLI

```bash
# สร้าง trust policy สำหรับ CodeBuild
cat > codebuild-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "codebuild.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# สร้าง IAM role
aws iam create-role \
  --role-name CodeBuildServiceRole \
  --assume-role-policy-document file://codebuild-trust-policy.json

# Attach policies
aws iam put-role-policy \
  --role-name CodeBuildServiceRole \
  --policy-name CodeBuildPolicy \
  --policy-document file://iam/codebuild-service-role-policy.json

# Attach managed policies
aws iam attach-role-policy \
  --role-name CodeBuildServiceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
```

---

## Environment Variables ใน CodeBuild

### ประเภทของ Environment Variables

```yaml
# buildspec.yml
env:
  # 1. Plaintext variables (สำหรับค่าที่ไม่ sensitive)
  variables:
    NODE_ENV: production
    APP_NAME: my-app
    AWS_DEFAULT_REGION: ap-southeast-1

  # 2. Parameter Store (สำหรับค่าที่ต้องการ encrypt)
  parameter-store:
    DB_HOST: /myapp/prod/db-host
    DB_PORT: /myapp/prod/db-port
    REDIS_URL: /myapp/prod/redis-url

  # 3. Secrets Manager (สำหรับ sensitive secrets)
  secrets-manager:
    # Format: secret-name:json-key
    DB_PASSWORD: myapp/prod/database:password
    API_KEY: myapp/prod/api:key
    JWT_SECRET: myapp/prod/jwt:secret

  # 4. Exported variables (ส่งต่อระหว่าง phases)
  exported-variables:
    - IMAGE_TAG
    - DEPLOY_VERSION
```

### เพิ่ม Parameter ใน Systems Manager

```bash
# เพิ่ม parameter (SecureString สำหรับ sensitive data)
aws ssm put-parameter \
  --name /myapp/prod/db-password \
  --value "my-secure-password" \
  --type SecureString \
  --key-id alias/aws/ssm

# เพิ่ม parameter ปกติ
aws ssm put-parameter \
  --name /myapp/prod/db-host \
  --value "my-db.cluster.ap-southeast-1.rds.amazonaws.com" \
  --type String

# List parameters
aws ssm get-parameters-by-path \
  --path /myapp/prod \
  --recursive

# ดู parameter
aws ssm get-parameter \
  --name /myapp/prod/db-password \
  --with-decryption
```

### เพิ่ม Secret ใน Secrets Manager

```bash
# สร้าง secret
aws secretsmanager create-secret \
  --name myapp/prod/database \
  --description "Database credentials for production" \
  --secret-string '{"username":"admin","password":"secretpassword"}'

# Update secret value
aws secretsmanager update-secret \
  --secret-id myapp/prod/database \
  --secret-string '{"username":"admin","password":"newpassword"}'

# ดู secret value
aws secretsmanager get-secret-value \
  --secret-id myapp/prod/database

# List secrets
aws secretsmanager list-secrets
```

### CodeBuild Environment Variables Override

```bash
# Override environment variables เมื่อ start build
aws codebuild start-build \
  --project-name my-app-build \
  --environment-variables-override \
    name=DEPLOY_ENV,value=staging,type=PLAINTEXT \
    name=DB_HOST,value=/myapp/staging/db-host,type=PARAMETER_STORE
```

---

## Artifacts ใน S3

### ตั้งค่า S3 สำหรับ Artifacts

```bash
# สร้าง S3 bucket สำหรับ artifacts
aws s3api create-bucket \
  --bucket my-codepipeline-artifacts-123456789 \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket my-codepipeline-artifacts-123456789 \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket my-codepipeline-artifacts-123456789 \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms"
        }
      }
    ]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket my-codepipeline-artifacts-123456789 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Lifecycle policy (ลบ artifacts เก่า)
cat > lifecycle-policy.json << 'EOF'
{
  "Rules": [
    {
      "ID": "delete-old-artifacts",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "codepipeline/"
      },
      "Expiration": {
        "Days": 30
      },
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-codepipeline-artifacts-123456789 \
  --lifecycle-configuration file://lifecycle-policy.json
```

### buildspec.yml ที่ใช้ S3 สำหรับ Artifacts

```yaml
# buildspec.yml
version: 0.2

phases:
  build:
    commands:
      - npm run build
      - echo "BUILD_VERSION=${CODEBUILD_BUILD_NUMBER}" >> version.txt

  post_build:
    commands:
      # Upload artifacts โดยตรงไปยัง S3
      - S3_PATH=s3://my-artifacts-bucket/builds/${CODEBUILD_BUILD_NUMBER}
      - aws s3 sync dist/ ${S3_PATH}/dist/
      - aws s3 cp version.txt ${S3_PATH}/

      # Create manifest file
      - |
        cat > manifest.json << MANIFEST
        {
          "buildNumber": "${CODEBUILD_BUILD_NUMBER}",
          "commitHash": "${CODEBUILD_RESOLVED_SOURCE_VERSION}",
          "buildTime": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
          "artifactPath": "${S3_PATH}"
        }
        MANIFEST
      - aws s3 cp manifest.json ${S3_PATH}/

# Artifacts สำหรับส่งต่อไปยัง CodePipeline
artifacts:
  files:
    - dist/**/*
    - manifest.json
  name: my-app-build-$(date +%Y%m%d)
  discard-paths: no

secondary-artifacts:
  # ส่ง test reports แยก
  test-reports:
    files:
      - coverage/**/*
      - test-results.xml
    name: my-app-test-$(date +%Y%m%d)
```

---

## Notifications ด้วย SNS

### ตั้งค่า SNS Topic

```bash
# สร้าง SNS topic
aws sns create-topic \
  --name codepipeline-notifications

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-southeast-1:123456789:codepipeline-notifications \
  --protocol email \
  --notification-endpoint team@example.com

# Subscribe Slack (ผ่าน Lambda)
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-southeast-1:123456789:codepipeline-notifications \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:ap-southeast-1:123456789:function:slack-notifier
```

### Lambda สำหรับ Slack Notifications

```python
# lambda/slack-notifier.py
import json
import os
import urllib.request
import urllib.parse

SLACK_WEBHOOK_URL = os.environ.get('SLACK_WEBHOOK_URL')

def get_color(state):
    colors = {
        'STARTED': '#439FE0',
        'SUCCEEDED': '#2EB886',
        'FAILED': '#CC0000',
        'CANCELED': '#FFA500',
        'RESUMED': '#9B59B6',
        'SUPERSEDED': '#808080'
    }
    return colors.get(state, '#808080')

def get_emoji(state):
    emojis = {
        'STARTED': ':rocket:',
        'SUCCEEDED': ':white_check_mark:',
        'FAILED': ':x:',
        'CANCELED': ':no_entry:',
    }
    return emojis.get(state, ':grey_question:')

def lambda_handler(event, context):
    message = json.loads(event['Records'][0]['Sns']['Message'])
    detail = message.get('detail', {})
    
    pipeline = detail.get('pipeline', 'Unknown')
    state = detail.get('state', 'Unknown')
    execution_id = detail.get('execution-id', 'Unknown')[:8]
    
    region = message.get('region', 'ap-southeast-1')
    account = message.get('account', '')
    
    pipeline_url = (f"https://{region}.console.aws.amazon.com/codesuite/codepipeline/"
                   f"pipelines/{pipeline}/view")
    
    color = get_color(state)
    emoji = get_emoji(state)
    
    payload = {
        'attachments': [
            {
                'color': color,
                'title': f'{emoji} CodePipeline: {pipeline}',
                'title_link': pipeline_url,
                'fields': [
                    {
                        'title': 'Status',
                        'value': state,
                        'short': True
                    },
                    {
                        'title': 'Execution ID',
                        'value': execution_id,
                        'short': True
                    },
                    {
                        'title': 'Region',
                        'value': region,
                        'short': True
                    }
                ],
                'footer': 'AWS CodePipeline',
                'footer_icon': 'https://aws.amazon.com/favicon.ico',
                'ts': int(context.get_remaining_time_in_millis() / 1000)
            }
        ]
    }
    
    data = json.dumps(payload).encode('utf-8')
    req = urllib.request.Request(
        SLACK_WEBHOOK_URL,
        data=data,
        headers={'Content-Type': 'application/json'}
    )
    
    with urllib.request.urlopen(req) as response:
        print(f"Slack response: {response.read().decode('utf-8')}")
    
    return {'statusCode': 200, 'body': 'Notification sent'}
```

### CodeStar Notifications (วิธีที่ง่ายกว่า)

```bash
# สร้าง notification rule สำหรับ CodePipeline
aws codestar-notifications create-notification-rule \
  --name my-app-pipeline-notifications \
  --event-type-ids \
    codepipeline-pipeline-pipeline-execution-failed \
    codepipeline-pipeline-pipeline-execution-succeeded \
    codepipeline-pipeline-pipeline-execution-started \
    codepipeline-pipeline-manual-approval-needed \
  --resource arn:aws:codepipeline:ap-southeast-1:123456789:my-app-pipeline \
  --targets \
    "Id=arn:aws:sns:ap-southeast-1:123456789:codepipeline-notifications,Type=SNS" \
  --detail-type FULL \
  --status ENABLED
```

### CloudWatch Events สำหรับ Pipeline

```json
// cloudwatch-events-rule.json
{
  "source": ["aws.codepipeline"],
  "detail-type": ["CodePipeline Pipeline Execution State Change"],
  "detail": {
    "pipeline": ["my-app-pipeline"],
    "state": ["FAILED", "SUCCEEDED", "STARTED"]
  }
}
```

```bash
# สร้าง EventBridge rule
aws events put-rule \
  --name codepipeline-state-change \
  --event-pattern file://cloudwatch-events-rule.json \
  --state ENABLED

# Add SNS target
aws events put-targets \
  --rule codepipeline-state-change \
  --targets \
    "Id=1,Arn=arn:aws:sns:ap-southeast-1:123456789:codepipeline-notifications,\
     InputTransformer={InputPathsMap={pipeline='$.detail.pipeline',state='$.detail.state'},\
     InputTemplate='\"Pipeline <pipeline> changed state to <state>\"\'"
```

---

## เปรียบเทียบกับ GitHub Actions

### ตาราง Feature Comparison

| Feature | AWS CodePipeline | GitHub Actions |
|---------|-----------------|----------------|
| **Trigger** | S3, CodeCommit, GitHub, ECR | Push, PR, Schedule, API |
| **Runner Environment** | CodeBuild (managed) | GitHub-hosted / Self-hosted |
| **Language** | JSON/YAML (Pipeline) + YAML (buildspec) | YAML |
| **Secrets Management** | SSM/Secrets Manager (native) | GitHub Secrets |
| **Artifacts** | S3 (native) | GitHub Artifacts (90 days) |
| **Approval Gates** | Manual Approval action | Environment protection rules |
| **Pricing** | $1/pipeline/month + build minutes | 2000 min free/month |
| **Integration** | AWS services native | Community marketplace |
| **Multi-region** | Native | Matrix strategy |
| **Audit** | CloudTrail | GitHub Audit Log |

### เมื่อใดควรใช้อะไร

**ใช้ AWS CodePipeline เมื่อ:**
- ต้องการ integrate กับ AWS services (ECS, Lambda, EC2) อย่างใกล้ชิด
- ต้องการ audit log ใน CloudTrail
- ทีมใช้ AWS เป็นหลัก
- ต้องการ multi-region deployment
- Security requirements สูง (AWS native IAM/KMS)

**ใช้ GitHub Actions เมื่อ:**
- Code อยู่ใน GitHub แล้ว
- ต้องการ flexibility และ community actions มากมาย
- ทีมคุ้นเคยกับ YAML workflow
- ต้องการ cross-platform (AWS + GCP + Azure)
- Cost-effective สำหรับ public repos

### Hybrid Approach

```yaml
# .github/workflows/deploy.yaml
# ใช้ GitHub Actions สำหรับ CI แล้ว trigger CodeDeploy สำหรับ CD

name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  ci:
    name: Build and Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests
        run: |
          npm ci
          npm test
      
      - name: Build Docker image
        run: docker build -t my-app .
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Push to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker tag my-app $ECR_REGISTRY/my-app:$IMAGE_TAG
          docker push $ECR_REGISTRY/my-app:$IMAGE_TAG
      
      - name: Start CodePipeline
        run: |
          aws codepipeline start-pipeline-execution \
            --name my-app-deploy-pipeline
```

---

## Infrastructure as Code ด้วย CloudFormation

### CloudFormation Template สำหรับ Pipeline

```yaml
# cloudformation/pipeline.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CI/CD Pipeline for my-app'

Parameters:
  AppName:
    Type: String
    Default: my-app
  
  GitHubOwner:
    Type: String
    Default: myorg
  
  GitHubRepo:
    Type: String
    Default: my-app
  
  GitHubBranch:
    Type: String
    Default: main
  
  GitHubConnectionArn:
    Type: String
    Description: CodeStar connection ARN for GitHub

Resources:
  # S3 bucket สำหรับ artifacts
  ArtifactBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub '${AppName}-codepipeline-artifacts-${AWS::AccountId}'
      VersioningConfiguration:
        Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
      LifecycleConfiguration:
        Rules:
          - Id: DeleteOldArtifacts
            Status: Enabled
            ExpirationInDays: 30

  # CodeBuild project
  BuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Name: !Sub '${AppName}-build'
      ServiceRole: !GetAtt CodeBuildRole.Arn
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/standard:7.0
        PrivilegedMode: true
        EnvironmentVariables:
          - Name: AWS_ACCOUNT_ID
            Value: !Ref AWS::AccountId
          - Name: APP_NAME
            Value: !Ref AppName
      Source:
        Type: CODEPIPELINE
        BuildSpec: buildspec.yml
      Artifacts:
        Type: CODEPIPELINE
      LogsConfig:
        CloudWatchLogs:
          Status: ENABLED
          GroupName: !Sub '/codebuild/${AppName}'

  # CodePipeline
  Pipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name: !Sub '${AppName}-pipeline'
      RoleArn: !GetAtt PipelineRole.Arn
      ArtifactStore:
        Type: S3
        Location: !Ref ArtifactBucket
      Stages:
        - Name: Source
          Actions:
            - Name: Source
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: CodeStarSourceConnection
                Version: '1'
              Configuration:
                ConnectionArn: !Ref GitHubConnectionArn
                FullRepositoryId: !Sub '${GitHubOwner}/${GitHubRepo}'
                BranchName: !Ref GitHubBranch
                DetectionMode: EVENTS
              OutputArtifacts:
                - Name: SourceArtifact
        
        - Name: Build
          Actions:
            - Name: Build
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              Configuration:
                ProjectName: !Ref BuildProject
              InputArtifacts:
                - Name: SourceArtifact
              OutputArtifacts:
                - Name: BuildArtifact
        
        - Name: DeployStaging
          Actions:
            - Name: Deploy
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CodeDeployToECS
                Version: '1'
              Configuration:
                ApplicationName: !Sub '${AppName}-staging'
                DeploymentGroupName: !Sub '${AppName}-staging-group'
                TaskDefinitionTemplateArtifact: BuildArtifact
                AppSpecTemplateArtifact: BuildArtifact
              InputArtifacts:
                - Name: BuildArtifact
        
        - Name: ProductionApproval
          Actions:
            - Name: Approve
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              Configuration:
                NotificationArn: !Ref DeployNotificationTopic
                CustomData: 'Please review staging and approve production deployment'
        
        - Name: DeployProduction
          Actions:
            - Name: Deploy
              ActionTypeId:
                Category: Deploy
                Owner: AWS
                Provider: CodeDeployToECS
                Version: '1'
              Configuration:
                ApplicationName: !Sub '${AppName}-production'
                DeploymentGroupName: !Sub '${AppName}-production-group'
                TaskDefinitionTemplateArtifact: BuildArtifact
                AppSpecTemplateArtifact: BuildArtifact
              InputArtifacts:
                - Name: BuildArtifact

  # SNS Topic สำหรับ notifications
  DeployNotificationTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${AppName}-deploy-notifications'
      Subscription:
        - Protocol: email
          Endpoint: team@example.com

  # IAM Roles
  PipelineRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codepipeline.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AWSCodePipelineFullAccess
      Policies:
        - PolicyName: PipelinePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                  - s3:GetBucketVersioning
                Resource:
                  - !GetAtt ArtifactBucket.Arn
                  - !Sub '${ArtifactBucket.Arn}/*'
              - Effect: Allow
                Action:
                  - codebuild:StartBuild
                  - codebuild:BatchGetBuilds
                Resource: !GetAtt BuildProject.Arn
              - Effect: Allow
                Action:
                  - codestar-connections:UseConnection
                Resource: !Ref GitHubConnectionArn
              - Effect: Allow
                Action:
                  - sns:Publish
                Resource: !Ref DeployNotificationTopic

  CodeBuildRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: codebuild.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser
      Policies:
        - PolicyName: CodeBuildPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                Resource: '*'
              - Effect: Allow
                Action:
                  - s3:GetObject
                  - s3:PutObject
                Resource:
                  - !GetAtt ArtifactBucket.Arn
                  - !Sub '${ArtifactBucket.Arn}/*'
              - Effect: Allow
                Action:
                  - ssm:GetParameter
                  - ssm:GetParameters
                Resource: !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:parameter/${AppName}/*'

Outputs:
  PipelineUrl:
    Description: URL ของ CodePipeline
    Value: !Sub 'https://${AWS::Region}.console.aws.amazon.com/codesuite/codepipeline/pipelines/${Pipeline}/view'
  
  ArtifactBucketName:
    Description: ชื่อ S3 bucket สำหรับ artifacts
    Value: !Ref ArtifactBucket
```

```bash
# Deploy CloudFormation stack
aws cloudformation create-stack \
  --stack-name my-app-pipeline \
  --template-body file://cloudformation/pipeline.yaml \
  --parameters \
    ParameterKey=GitHubConnectionArn,ParameterValue=arn:aws:codestar-connections:... \
    ParameterKey=GitHubOwner,ParameterValue=myorg \
  --capabilities CAPABILITY_NAMED_IAM

# Update stack
aws cloudformation update-stack \
  --stack-name my-app-pipeline \
  --template-body file://cloudformation/pipeline.yaml \
  --capabilities CAPABILITY_NAMED_IAM

# Monitor deployment
aws cloudformation wait stack-create-complete \
  --stack-name my-app-pipeline

# ดู outputs
aws cloudformation describe-stacks \
  --stack-name my-app-pipeline \
  --query 'Stacks[0].Outputs'
```

---

## แบบฝึกหัด

### Exercise 1: สร้าง Pipeline พื้นฐาน

สร้าง CodePipeline ที่ build Node.js application และ deploy ไปยัง ECS:

1. สร้าง ECR repository:
```bash
aws ecr create-repository \
  --repository-name my-nodejs-app \
  --region ap-southeast-1

# ดู repository URI
aws ecr describe-repositories \
  --repository-names my-nodejs-app \
  --query 'repositories[0].repositoryUri' \
  --output text
```

2. สร้าง `buildspec.yml`:
```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - REPO=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/my-nodejs-app
      - TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-7)
  build:
    commands:
      - npm ci
      - npm test
      - docker build -t $REPO:$TAG .
  post_build:
    commands:
      - docker push $REPO:$TAG
      - printf '[{"name":"my-nodejs-app","imageUri":"%s"}]' $REPO:$TAG > imagedefinitions.json
artifacts:
  files:
    - imagedefinitions.json
```

3. สร้าง Pipeline ผ่าน AWS Console ตามขั้นตอนใน [section ด้านบน](#สร้าง-pipeline-ใน-aws-console)

### Exercise 2: เพิ่ม Approval Gate

เพิ่ม manual approval stage ระหว่าง staging และ production:

```bash
# สร้าง SNS topic สำหรับ approvals
aws sns create-topic --name pipeline-approvals
aws sns subscribe \
  --topic-arn $(aws sns list-topics --query 'Topics[?contains(TopicArn, `pipeline-approvals`)].TopicArn' --output text) \
  --protocol email \
  --notification-endpoint your-email@example.com

# Update pipeline เพิ่ม Approval stage (แก้ไข pipeline.json แล้ว update)
aws codepipeline update-pipeline --cli-input-json file://pipeline.json
```

### Exercise 3: Setup Notifications

ตั้งค่า Slack notifications เมื่อ pipeline สำเร็จ/ล้มเหลว:

1. สร้าง Slack App และ Incoming Webhook URL
2. สร้าง Lambda function จากโค้ดใน [section Notifications](#notifications-ด้วย-sns)
3. Subscribe Lambda กับ SNS topic
4. ทดสอบโดย push code ไปยัง repository

### Exercise 4: Deploy ด้วย CloudFormation

ใช้ CloudFormation template จาก [section ด้านบน](#infrastructure-as-code-ด้วย-cloudformation) เพื่อสร้าง complete pipeline infrastructure

---

## สรุป

AWS CodePipeline เป็นเครื่องมือ CI/CD ที่ทรงพลังสำหรับ AWS ecosystem:

1. **CodeCommit** → Source code repository (deprecated สำหรับลูกค้าใหม่)
2. **CodeBuild** → Managed build environment พร้อม buildspec.yml
3. **CodeDeploy** → Automated deployment ไปยัง EC2/ECS/Lambda
4. **CodePipeline** → Orchestration ของทุกส่วน
5. **IAM** → Security ด้วย least-privilege roles
6. **SSM/Secrets Manager** → Secure secrets management
7. **S3** → Artifact storage ที่น่าเชื่อถือ
8. **SNS/EventBridge** → Notifications และ monitoring
9. **CloudFormation** → Infrastructure as Code

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **Google Cloud Build** ซึ่งเป็น CI/CD service ของ Google Cloud Platform

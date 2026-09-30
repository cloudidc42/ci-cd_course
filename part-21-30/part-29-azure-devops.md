# Part 29: Cloud CI/CD — Azure DevOps

## สารบัญ
1. [Azure DevOps Overview](#azure-devops-overview)
2. [Azure DevOps Components](#azure-devops-components)
3. [azure-pipelines.yml](#azure-pipelinesyml)
4. [Stages, Jobs, and Steps](#stages-jobs-and-steps)
5. [Agent Pools](#agent-pools)
6. [Environments](#environments)
7. [Service Connections](#service-connections)
8. [Variable Groups](#variable-groups)
9. [Deployment ไปยัง App Service/AKS/Functions](#deployment-ไปยัง-app-serviceaksfunctions)
10. [Release Gates และ Approval Workflows](#release-gates-และ-approval-workflows)
11. [Bicep Integration](#bicep-integration)
12. [Azure Boards Integration](#azure-boards-integration)
13. [Azure Artifacts](#azure-artifacts)
14. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Azure DevOps Overview

**Azure DevOps** เป็นชุดบริการ DevOps ของ Microsoft ที่ครอบคลุมทุกขั้นตอนของ software development lifecycle

### Azure DevOps Suite

```
┌─────────────────────────────────────────────────────────────────┐
│                       Azure DevOps                             │
│                                                                 │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌─────────────────┐  │
│  │ Boards  │  │  Repos   │  │Pipelines│  │    Artifacts    │  │
│  │         │  │          │  │         │  │                 │  │
│  │ Work    │  │ Git repos│  │ CI/CD   │  │ Package feeds   │  │
│  │ Tracking│  │          │  │ Build   │  │ (NuGet, npm,    │  │
│  │ Sprints │  │ Code     │  │ Release │  │  Maven, etc.)   │  │
│  │ Backlogs│  │ Review   │  │         │  │                 │  │
│  └─────────┘  └──────────┘  └─────────┘  └─────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Test Plans                           │  │
│  │  Manual testing, automated testing, test case management │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### ข้อดีของ Azure DevOps

- **Complete platform**: Boards, Repos, Pipelines, Artifacts, Test Plans ใน platform เดียว
- **Enterprise features**: Advanced security, compliance, audit logs
- **Microsoft ecosystem**: Native integration กับ Azure services
- **Flexible pricing**: Free tier พร้อม free parallel jobs
- **Hybrid support**: Azure-hosted และ self-hosted agents
- **YAML-first**: Pipeline as code ด้วย YAML

### เปรียบเทียบกับ GitHub Actions

| Feature | Azure DevOps | GitHub Actions |
|---------|-------------|----------------|
| Repository | Azure Repos / GitHub | GitHub |
| Pipeline config | azure-pipelines.yml | .github/workflows/*.yaml |
| Free tier | 1 parallel job (public: 10) | 2000 min/month |
| Approvals | Native environments | Environment protection rules |
| Work tracking | Azure Boards (built-in) | GitHub Issues/Projects |
| Package registry | Azure Artifacts | GitHub Packages |
| Test management | Test Plans (built-in) | Third-party tools |

---

## Azure DevOps Components

### Azure Repos

```bash
# Clone repository
git clone https://dev.azure.com/myorg/myproject/_git/myrepo

# หรือผ่าน SSH
git clone git@ssh.dev.azure.com:v3/myorg/myproject/myrepo
```

### Azure Boards Integration

```yaml
# azure-pipelines.yml
# เมื่อ pipeline สำเร็จ อัปเดต work items
- task: UpdateWorkItems@1
  displayName: 'Update Work Items'
  condition: succeeded()
  inputs:
    workItemType: 'Bug'
    updateRules: |
      [
        {
          "ruleType": "ChangeStateTo",
          "value": "Closed"
        }
      ]
```

---

## azure-pipelines.yml

### โครงสร้างพื้นฐาน

```yaml
# azure-pipelines.yml
# ชื่อ pipeline (optional)
name: '$(Date:yyyyMMdd)$(Rev:.r)'

# Trigger
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    exclude:
      - docs/**
      - '*.md'

# PR trigger
pr:
  branches:
    include:
      - main
  paths:
    exclude:
      - docs/**

# กำหนด pool สำหรับทั้ง pipeline (override ได้ในแต่ละ job)
pool:
  vmImage: 'ubuntu-latest'

# Variables ระดับ pipeline
variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.0.x'

# Stages
stages:
  - stage: Build
    displayName: 'Build and Test'
    jobs:
      - job: BuildJob
        steps:
          - script: echo "Building..."

  - stage: Deploy
    displayName: 'Deploy'
    dependsOn: Build
    jobs:
      - deployment: DeployJob
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying..."
```

### Full Pipeline สำหรับ .NET Application

```yaml
# azure-pipelines.yml
name: '$(Date:yyyyMMdd).$(Rev:r)'

trigger:
  branches:
    include:
      - main
      - feature/*
      - hotfix/*
  paths:
    exclude:
      - 'docs/**'
      - '**/*.md'
      - '.gitignore'

pr:
  branches:
    include:
      - main
      - develop

variables:
  - group: 'common-variables'  # Variable group
  - name: buildConfiguration
    value: 'Release'
  - name: dotnetVersion
    value: '8.0.x'
  - name: projectName
    value: 'MyApp'

stages:
  # ===== STAGE 1: Build =====
  - stage: Build
    displayName: 'Build and Test'
    pool:
      vmImage: 'ubuntu-latest'
    
    jobs:
      - job: BuildAndTest
        displayName: 'Build and Run Tests'
        timeoutInMinutes: 30
        
        steps:
          - checkout: self
            fetchDepth: 0  # Full history สำหรับ SonarQube
          
          - task: UseDotNet@2
            displayName: 'Install .NET SDK'
            inputs:
              packageType: 'sdk'
              version: '$(dotnetVersion)'
          
          - task: DotNetCoreCLI@2
            displayName: 'Restore packages'
            inputs:
              command: 'restore'
              projects: '**/*.csproj'
              feedsToUse: 'config'
              nugetConfigPath: 'NuGet.config'
          
          - task: DotNetCoreCLI@2
            displayName: 'Build'
            inputs:
              command: 'build'
              projects: '**/*.csproj'
              arguments: '--configuration $(buildConfiguration) --no-restore'
          
          - task: DotNetCoreCLI@2
            displayName: 'Run Unit Tests'
            inputs:
              command: 'test'
              projects: '**/*Tests/*.csproj'
              arguments: >
                --configuration $(buildConfiguration)
                --no-build
                --collect:"XPlat Code Coverage"
                --results-directory $(Agent.TempDirectory)/TestResults
                --logger "trx;LogFileName=TestResults.trx"
          
          - task: PublishTestResults@2
            displayName: 'Publish Test Results'
            condition: always()
            inputs:
              testResultsFormat: 'VSTest'
              testResultsFiles: '$(Agent.TempDirectory)/TestResults/**/*.trx'
              mergeTestResults: true
          
          - task: PublishCodeCoverageResults@2
            displayName: 'Publish Code Coverage'
            inputs:
              summaryFileLocation: '$(Agent.TempDirectory)/TestResults/**/coverage.cobertura.xml'
          
          - task: DotNetCoreCLI@2
            displayName: 'Publish application'
            inputs:
              command: 'publish'
              publishWebProjects: true
              arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
              zipAfterPublish: true
          
          - task: PublishBuildArtifacts@1
            displayName: 'Publish Build Artifacts'
            inputs:
              pathToPublish: '$(Build.ArtifactStagingDirectory)'
              artifactName: 'drop'
              publishLocation: 'Container'

  # ===== STAGE 2: Deploy to Staging =====
  - stage: DeployStaging
    displayName: 'Deploy to Staging'
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    
    jobs:
      - deployment: DeployToStaging
        displayName: 'Deploy to Staging Environment'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                
                - task: AzureWebApp@1
                  displayName: 'Deploy to Azure App Service (Staging)'
                  inputs:
                    azureSubscription: 'Azure-Service-Connection'
                    appType: 'webApp'
                    appName: 'my-app-staging'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
                    deploymentMethod: 'auto'
  
  # ===== STAGE 3: Integration Tests =====
  - stage: IntegrationTests
    displayName: 'Integration Tests'
    dependsOn: DeployStaging
    
    jobs:
      - job: RunIntegrationTests
        displayName: 'Run Integration Tests'
        pool:
          vmImage: 'ubuntu-latest'
        
        steps:
          - task: DotNetCoreCLI@2
            displayName: 'Run Integration Tests'
            inputs:
              command: 'test'
              projects: '**/*IntegrationTests/*.csproj'
              arguments: '--configuration $(buildConfiguration)'
            env:
              APP_URL: 'https://my-app-staging.azurewebsites.net'
  
  # ===== STAGE 4: Deploy to Production =====
  - stage: DeployProduction
    displayName: 'Deploy to Production'
    dependsOn: IntegrationTests
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    
    jobs:
      - deployment: DeployToProduction
        displayName: 'Deploy to Production Environment'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'  # Requires approval
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                
                - task: AzureWebApp@1
                  displayName: 'Deploy to Azure App Service (Production)'
                  inputs:
                    azureSubscription: 'Azure-Service-Connection'
                    appType: 'webApp'
                    appName: 'my-app-production'
                    package: '$(Pipeline.Workspace)/drop/**/*.zip'
                    deploymentMethod: 'auto'
                
                - task: AzureAppServiceManage@0
                  displayName: 'Swap Slots'
                  inputs:
                    azureSubscription: 'Azure-Service-Connection'
                    Action: 'Swap Slots'
                    WebAppName: 'my-app-production'
                    ResourceGroupName: 'my-app-rg'
                    SourceSlot: 'staging'
```

---

## Stages, Jobs, and Steps

### Hierarchy

```
Pipeline
└── Stage (แยก environment หรือ phase)
    └── Job (รันบน agent หนึ่งตัว)
        └── Step (task หรือ script ที่รันในลำดับ)
```

### Stages

```yaml
stages:
  - stage: Build
    displayName: 'Build Stage'
    
    # รัน stage นี้เมื่อไร
    condition: always()
    
    # ต้องรอ stage อะไรก่อน
    dependsOn: []
    
    # Variables เฉพาะ stage นี้
    variables:
      stageName: 'build'
    
    jobs:
      - job: ...

  - stage: Test
    dependsOn: Build
    condition: succeeded('Build')
    
    jobs:
      - job: ...

  - stage: DeployDev
    dependsOn: Test
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - job: ...

  - stage: DeployProd
    dependsOn: DeployDev
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: ...
```

### Jobs

```yaml
jobs:
  # Regular Job
  - job: BuildJob
    displayName: 'Build Application'
    
    # Pool สำหรับ job นี้
    pool:
      vmImage: 'ubuntu-latest'
      # หรือ self-hosted:
      # name: 'my-agent-pool'
    
    # Timeout
    timeoutInMinutes: 60
    cancelTimeoutInMinutes: 5
    
    # Retry
    retryCountOnTaskFailure: 2
    
    # ขึ้นกับ job อื่น
    dependsOn: []
    
    # Variables เฉพาะ job
    variables:
      jobVariable: 'value'
    
    # Workspace options
    workspace:
      clean: all  # outputs, resources, all
    
    steps:
      - script: echo "Building"
  
  # Parallel jobs (รันพร้อมกัน)
  - job: Job1
    steps:
      - script: echo "Job 1"
  
  - job: Job2
    steps:
      - script: echo "Job 2"
  
  # Matrix job (รัน job เดียวกันหลาย config)
  - job: MatrixJob
    strategy:
      matrix:
        node16:
          nodeVersion: '16.x'
        node18:
          nodeVersion: '18.x'
        node20:
          nodeVersion: '20.x'
      maxParallel: 3
    steps:
      - task: NodeTool@0
        inputs:
          versionSpec: '$(nodeVersion)'
      - script: npm test
```

### Steps ประเภทต่างๆ

```yaml
steps:
  # Script (bash/cmd)
  - script: |
      echo "Hello from bash"
      npm install
      npm test
    displayName: 'Run Tests'
    workingDirectory: '$(Build.SourcesDirectory)'
    failOnStderr: true
    env:
      CI: 'true'
      NODE_ENV: 'test'
  
  # Bash step (เสมอรัน bash)
  - bash: |
      echo "Platform: $AGENT_OS"
      ls -la
    displayName: 'Show environment'
  
  # PowerShell step
  - pwsh: |
      Write-Host "PowerShell"
      Get-Location
    displayName: 'PowerShell step'
  
  # Task (pre-built tasks)
  - task: NodeTool@0
    displayName: 'Install Node.js'
    inputs:
      versionSpec: '18.x'
  
  # Checkout
  - checkout: self
    clean: true
    fetchDepth: 0
    lfs: false
  
  # Download artifacts
  - download: current
    artifact: 'drop'
    patterns: '**/*.zip'
  
  # Publish artifacts
  - publish: '$(Build.ArtifactStagingDirectory)'
    artifact: 'drop'
    displayName: 'Publish artifacts'
  
  # Template reference
  - template: templates/test-steps.yml
    parameters:
      testProject: 'MyApp.Tests'
  
  # Condition
  - script: echo "Only on main"
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
  
  # Continue on error
  - script: npm audit
    continueOnError: true
```

### Templates

```yaml
# templates/build-steps.yml
parameters:
  - name: buildConfiguration
    type: string
    default: 'Release'
  - name: projectPath
    type: string
    default: '**/*.csproj'

steps:
  - task: DotNetCoreCLI@2
    displayName: 'Restore'
    inputs:
      command: 'restore'
      projects: '${{ parameters.projectPath }}'
  
  - task: DotNetCoreCLI@2
    displayName: 'Build'
    inputs:
      command: 'build'
      projects: '${{ parameters.projectPath }}'
      arguments: '--configuration ${{ parameters.buildConfiguration }}'
```

```yaml
# azure-pipelines.yml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - template: templates/build-steps.yml
            parameters:
              buildConfiguration: 'Release'
              projectPath: 'src/**/*.csproj'
```

---

## Agent Pools

### Microsoft-hosted Agents

```yaml
# Available images
pool:
  vmImage: 'ubuntu-latest'   # Ubuntu 22.04
  # หรือ:
  vmImage: 'ubuntu-22.04'
  vmImage: 'ubuntu-20.04'
  vmImage: 'windows-latest'  # Windows Server 2022
  vmImage: 'windows-2022'
  vmImage: 'windows-2019'
  vmImage: 'macOS-latest'    # macOS 14
  vmImage: 'macOS-14'
  vmImage: 'macOS-13'
  vmImage: 'macOS-12'
```

### Self-hosted Agents

```bash
# ดาวน์โหลดและ configure agent บน Linux
mkdir myagent && cd myagent
wget https://vstsagentpackage.azureedge.net/agent/3.232.3/vsts-agent-linux-x64-3.232.3.tar.gz
tar zxvf vsts-agent-linux-x64-3.232.3.tar.gz

./config.sh \
  --url https://dev.azure.com/myorg \
  --auth PAT \
  --token MY_PAT_TOKEN \
  --pool my-self-hosted-pool \
  --agent my-agent-name \
  --work /home/ubuntu/agent-work \
  --runAsService

# Start agent
./run.sh

# Run as service (systemd)
sudo ./svc.sh install
sudo ./svc.sh start
```

### Docker Container Agent

```dockerfile
# Dockerfile.agent
FROM ubuntu:22.04

ARG TARGETARCH=amd64
ARG AGENT_VERSION=3.232.3

RUN apt-get update && apt-get install -y \
    curl \
    git \
    jq \
    libicu70 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /azp

# Download agent
RUN if [ "$TARGETARCH" = "amd64" ]; then \
      AZP_AGENTPACKAGE_URL="https://vstsagentpackage.azureedge.net/agent/${AGENT_VERSION}/vsts-agent-linux-x64-${AGENT_VERSION}.tar.gz"; \
    else \
      AZP_AGENTPACKAGE_URL="https://vstsagentpackage.azureedge.net/agent/${AGENT_VERSION}/vsts-agent-linux-arm64-${AGENT_VERSION}.tar.gz"; \
    fi; \
    curl -LsS "$AZP_AGENTPACKAGE_URL" | tar -xz

COPY start.sh .
RUN chmod +x start.sh

ENTRYPOINT ["/azp/start.sh"]
```

```bash
#!/bin/bash
# start.sh
set -e

if [ -z "$AZP_URL" ]; then
  echo "error: missing AZP_URL environment variable"
  exit 1
fi

if [ -z "$AZP_TOKEN" ]; then
  echo "error: missing AZP_TOKEN environment variable"
  exit 1
fi

./config.sh --unattended \
  --url "$AZP_URL" \
  --auth "PAT" \
  --token "$AZP_TOKEN" \
  --pool "${AZP_POOL:-Default}" \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --work "${AZP_WORK:-/azp/_work}" \
  --replace

./run.sh "$@"
```

```bash
# รัน agent ใน Docker
docker run -e AZP_URL="https://dev.azure.com/myorg" \
  -e AZP_TOKEN="MY_PAT_TOKEN" \
  -e AZP_POOL="docker-pool" \
  my-azure-agent:latest
```

### Scale Set Agents (Kubernetes)

```yaml
# k8s/azure-pipelines-agent.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: azure-pipelines-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: azure-pipelines-agent
  template:
    metadata:
      labels:
        app: azure-pipelines-agent
    spec:
      containers:
        - name: agent
          image: my-azure-agent:latest
          env:
            - name: AZP_URL
              value: "https://dev.azure.com/myorg"
            - name: AZP_TOKEN
              valueFrom:
                secretKeyRef:
                  name: azure-devops-token
                  key: token
            - name: AZP_POOL
              value: "k8s-pool"
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2
              memory: 2Gi
          volumeMounts:
            - name: docker-socket
              mountPath: /var/run/docker.sock
      volumes:
        - name: docker-socket
          hostPath:
            path: /var/run/docker.sock
```

---

## Environments

Environments ใน Azure DevOps ช่วยจัดการ deployment targets พร้อม approval gates

### สร้าง Environment

1. ไปที่ **Pipelines > Environments**
2. คลิก **New environment**
3. กำหนด:
   - Name: production
   - Resource: Kubernetes / Virtual machines / None
4. ตั้งค่า **Approvals and checks**:
   - Approvals: เลือกผู้อนุมัติ
   - Branch control: เฉพาะ branches ที่กำหนด
   - Business hours: deploy เฉพาะเวลาทำงาน

### ใช้ Environment ใน Pipeline

```yaml
stages:
  - stage: DeployProduction
    jobs:
      # Deployment job (ไม่ใช่ regular job)
      - deployment: DeployApp
        displayName: 'Deploy to Production'
        
        # Pool สำหรับ deployment
        pool:
          vmImage: 'ubuntu-latest'
        
        # Environment name
        environment: 'production'
        # หรือ environment กับ resource:
        # environment: 'production.my-server'
        
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to production..."
            
            on:
              failure:
                steps:
                  - script: echo "Deployment failed, rolling back..."
              success:
                steps:
                  - script: echo "Deployment succeeded!"
```

### Kubernetes Environment

```yaml
# เพิ่ม Kubernetes resource ใน Environment
# ผ่าน UI: Environments > production > Add resource > Kubernetes

stages:
  - stage: Deploy
    jobs:
      - deployment: DeployToK8s
        environment: 'production.my-namespace'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  displayName: 'Deploy to Kubernetes'
                  inputs:
                    action: 'deploy'
                    manifests: |
                      kubernetes/deployment.yaml
                      kubernetes/service.yaml
                    containers: |
                      myregistry.azurecr.io/my-app:$(Build.BuildId)
```

---

## Service Connections

Service connections ใช้ connect Azure DevOps กับ external services

### ประเภท Service Connections

- **Azure Resource Manager** - connect กับ Azure subscription
- **Docker Registry** - push/pull Docker images
- **Kubernetes** - deploy ไปยัง Kubernetes
- **GitHub** - access GitHub repositories
- **AWS** - deploy ไปยัง AWS

### สร้าง Azure Resource Manager Connection

1. ไปที่ **Project Settings > Service connections**
2. คลิก **New service connection**
3. เลือก **Azure Resource Manager**
4. เลือก **Service principal (automatic)**
5. เลือก Subscription และ Resource Group
6. ตั้งชื่อ: `Azure-Production-Connection`

### ใช้ Service Connection

```yaml
# ใช้ Azure RM service connection
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'Azure-Production-Connection'
    appName: 'my-app'
    package: '$(Build.ArtifactStagingDirectory)/**/*.zip'

# ใช้ Docker registry connection
- task: Docker@2
  inputs:
    containerRegistry: 'my-docker-registry'
    repository: 'my-app'
    command: 'buildAndPush'
    tags: '$(Build.BuildId)'

# ใช้ Kubernetes connection
- task: KubernetesManifest@1
  inputs:
    kubernetesServiceConnection: 'my-k8s-cluster'
    action: 'deploy'
    manifests: 'k8s/deployment.yaml'
```

### สร้าง Service Connection ด้วย Azure CLI

```bash
# Login
az login
az devops configure --defaults organization=https://dev.azure.com/myorg project=MyProject

# สร้าง service connection
az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id "APP_ID" \
  --azure-rm-subscription-id "SUBSCRIPTION_ID" \
  --azure-rm-subscription-name "My Subscription" \
  --azure-rm-tenant-id "TENANT_ID" \
  --name "Azure-Production"

# List service connections
az devops service-endpoint list
```

---

## Variable Groups

Variable Groups ใช้สำหรับเก็บ variables ที่ใช้ร่วมกันระหว่าง pipelines

### สร้าง Variable Group

1. ไปที่ **Pipelines > Library**
2. คลิก **+ Variable group**
3. ตั้งชื่อและเพิ่ม variables

### ใช้ Variable Group

```yaml
# azure-pipelines.yml
variables:
  # Reference variable group
  - group: 'production-variables'
  
  # Inline variables
  - name: buildConfiguration
    value: 'Release'

stages:
  - stage: Build
    variables:
      - group: 'build-variables'
    jobs:
      - job: BuildJob
        steps:
          - script: echo "$(MY_API_KEY)"
```

### Variable Group กับ Azure Key Vault

```yaml
# Link variable group กับ Azure Key Vault
# ใน UI: Library > Variable group > Link secrets from Azure Key Vault

variables:
  - group: 'keyvault-variables'  # linked to Key Vault

steps:
  - script: |
      echo "DB_PASSWORD is set: $DB_PASSWORD"
      # ใช้ secret ได้เลย (masked ใน logs)
    env:
      DB_PASSWORD: $(DB-PASSWORD)  # Key Vault secret name จะมี dash แทน underscore
```

### สร้าง Variable Group ด้วย Azure CLI

```bash
# สร้าง variable group
az pipelines variable-group create \
  --name 'production-variables' \
  --variables \
    APP_URL=https://myapp.azurewebsites.net \
    APP_ENV=production

# เพิ่ม secret variable
az pipelines variable-group variable create \
  --group-id 1 \
  --name DB_PASSWORD \
  --value "secretpassword" \
  --secret true

# List variable groups
az pipelines variable-group list
```

---

## Deployment ไปยัง App Service/AKS/Functions

### Deploy ไปยัง Azure App Service

```yaml
# azure-pipelines.yml
stages:
  - stage: Build
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '18.x'
          
          - script: |
              npm ci
              npm run build
              npm test
          
          - task: ArchiveFiles@2
            inputs:
              rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/app.zip'
              replaceExistingArchive: true
          
          - publish: '$(Build.ArtifactStagingDirectory)/app.zip'
            artifact: 'drop'
  
  - stage: DeployStaging
    dependsOn: Build
    jobs:
      - deployment: Deploy
        environment: 'staging'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: 'drop'
                
                # Deploy ไปยัง staging slot
                - task: AzureWebApp@1
                  displayName: 'Deploy to Staging Slot'
                  inputs:
                    azureSubscription: 'Azure-Service-Connection'
                    appType: 'webAppLinux'
                    appName: 'my-app'
                    deployToSlotOrASE: true
                    resourceGroupName: 'my-app-rg'
                    slotName: 'staging'
                    package: '$(Pipeline.Workspace)/drop/app.zip'
                    runtimeStack: 'NODE|18-lts'
                    startUpCommand: 'node server.js'
                
                # Health check
                - bash: |
                    sleep 30
                    HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://my-app-staging.azurewebsites.net/health)
                    if [ "$HTTP_STATUS" = "200" ]; then
                      echo "Health check passed"
                    else
                      echo "Health check failed: $HTTP_STATUS"
                      exit 1
                    fi
                  displayName: 'Health Check'
  
  - stage: DeployProduction
    dependsOn: DeployStaging
    jobs:
      - deployment: Deploy
        environment: 'production'  # requires approval
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                # Swap staging slot ไปยัง production
                - task: AzureAppServiceManage@0
                  displayName: 'Swap Slots'
                  inputs:
                    azureSubscription: 'Azure-Service-Connection'
                    Action: 'Swap Slots'
                    WebAppName: 'my-app'
                    ResourceGroupName: 'my-app-rg'
                    SourceSlot: 'staging'
                    SwapWithProduction: true
```

### Deploy ไปยัง AKS

```yaml
# azure-pipelines.yml
variables:
  dockerRegistry: 'myregistry.azurecr.io'
  imageRepository: 'my-app'
  imageName: '$(dockerRegistry)/$(imageRepository)'
  k8sNamespace: 'production'

stages:
  - stage: Build
    jobs:
      - job: BuildImage
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: Docker@2
            displayName: 'Build and Push Docker Image'
            inputs:
              containerRegistry: 'ACR-Service-Connection'
              repository: '$(imageRepository)'
              command: 'buildAndPush'
              Dockerfile: 'Dockerfile'
              tags: |
                $(Build.BuildId)
                latest
          
          - publish: 'kubernetes'
            artifact: 'k8s-manifests'
  
  - stage: DeployAKS
    dependsOn: Build
    jobs:
      - deployment: Deploy
        displayName: 'Deploy to AKS'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'aks-production.$(k8sNamespace)'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: 'k8s-manifests'
                
                # สร้าง imagePullSecret
                - task: KubernetesManifest@1
                  displayName: 'Create imagePullSecret'
                  inputs:
                    action: 'createSecret'
                    kubernetesServiceConnection: 'AKS-Service-Connection'
                    namespace: '$(k8sNamespace)'
                    secretType: 'dockerRegistry'
                    secretName: 'acr-secret'
                    dockerRegistryEndpoint: 'ACR-Service-Connection'
                
                # Deploy manifests
                - task: KubernetesManifest@1
                  displayName: 'Deploy to Kubernetes'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'AKS-Service-Connection'
                    namespace: '$(k8sNamespace)'
                    manifests: |
                      $(Pipeline.Workspace)/k8s-manifests/deployment.yaml
                      $(Pipeline.Workspace)/k8s-manifests/service.yaml
                      $(Pipeline.Workspace)/k8s-manifests/ingress.yaml
                    imagePullSecrets: 'acr-secret'
                    containers: |
                      $(imageName):$(Build.BuildId)
                
                # ตรวจสอบ deployment
                - task: Kubernetes@1
                  displayName: 'Check Rollout Status'
                  inputs:
                    connectionType: 'Kubernetes Service Connection'
                    kubernetesServiceEndpoint: 'AKS-Service-Connection'
                    namespace: '$(k8sNamespace)'
                    command: 'rollout'
                    arguments: 'status deployment/my-app'
```

### Deploy ไปยัง Azure Functions

```yaml
# azure-pipelines.yml
stages:
  - stage: Build
    jobs:
      - job: BuildFunction
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: '18.x'
          
          - bash: |
              npm ci
              npm run build
            displayName: 'Install and Build'
          
          - task: ArchiveFiles@2
            displayName: 'Archive Function App'
            inputs:
              rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/function.zip'
          
          - publish: '$(Build.ArtifactStagingDirectory)/function.zip'
            artifact: 'function'
  
  - stage: DeployFunction
    dependsOn: Build
    jobs:
      - deployment: Deploy
        environment: 'functions-production'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: 'function'
                
                - task: AzureFunctionApp@2
                  displayName: 'Deploy Azure Function'
                  inputs:
                    connectedServiceNameARM: 'Azure-Service-Connection'
                    appType: 'functionAppLinux'
                    appName: 'my-function-app'
                    package: '$(Pipeline.Workspace)/function/function.zip'
                    runtimeStack: 'NODE|18'
                    deploymentMethod: 'zipDeploy'
```

---

## Release Gates และ Approval Workflows

### Approval Gates ใน Environment

```yaml
# ตั้งค่า approvals ผ่าน UI: Environments > production > Approvals and checks

# หรือผ่าน REST API:
POST https://dev.azure.com/myorg/myproject/_apis/pipelines/checks/configurations?api-version=7.1-preview.1
{
  "type": {
    "id": "8c6f20a7-a545-4486-9777-f762fafe0d4d"
  },
  "settings": {
    "approvers": [
      {
        "id": "user-id-guid"
      }
    ],
    "requiredApproverCount": 1,
    "allowApproversToSelfApprove": false,
    "instructions": "Please review deployment and approve"
  },
  "timeout": 43200,
  "resource": {
    "type": "environment",
    "id": "1"
  }
}
```

### Branch Control Check

```yaml
# เพิ่ม Branch Control ใน Environment settings:
# Environments > production > Approvals and checks > Branch control
# Allowed branches: refs/heads/main, refs/heads/release/*
```

### Business Hours Check

```yaml
# Time zone: (UTC+07:00) Bangkok, Hanoi, Jakarta
# Days: Monday - Friday
# Hours: 09:00 - 17:00
```

### Custom Gate ด้วย Azure Function

```python
# azure-functions/deployment-gate/function_app.py
import azure.functions as func
import json
import urllib.request

app = func.FunctionApp()

@app.function_name(name="deployment-gate")
@app.route(route="gate", methods=["POST"])
def deployment_gate(req: func.HttpRequest) -> func.HttpResponse:
    """Azure DevOps deployment gate"""
    
    try:
        request_body = req.get_json()
        
        # ตรวจสอบ conditions ต่างๆ
        checks = []
        
        # Check 1: ตรวจสอบว่าไม่มี active incidents
        incident_check = check_incidents()
        checks.append(incident_check)
        
        # Check 2: ตรวจสอบ load ของ production
        load_check = check_production_load()
        checks.append(load_check)
        
        # Check 3: ตรวจสอบ error rate
        error_check = check_error_rate()
        checks.append(error_check)
        
        # ถ้าทุก check ผ่าน
        all_passed = all(c['passed'] for c in checks)
        
        if all_passed:
            return func.HttpResponse(
                json.dumps({
                    "status": "succeeded",
                    "message": "All pre-deployment checks passed",
                    "checks": checks
                }),
                status_code=200,
                mimetype="application/json"
            )
        else:
            failed_checks = [c for c in checks if not c['passed']]
            return func.HttpResponse(
                json.dumps({
                    "status": "failed",
                    "message": "Pre-deployment checks failed",
                    "failedChecks": failed_checks
                }),
                status_code=200,  # ต้อง return 200 ให้ Azure DevOps
                mimetype="application/json"
            )
    except Exception as e:
        return func.HttpResponse(
            json.dumps({"status": "failed", "message": str(e)}),
            status_code=200,
            mimetype="application/json"
        )

def check_incidents():
    # ตรวจสอบ PagerDuty หรือ Azure Monitor alerts
    return {"name": "No Active Incidents", "passed": True, "message": "No active incidents found"}

def check_production_load():
    # ตรวจสอบ CPU/Memory ของ production
    return {"name": "Production Load", "passed": True, "message": "Load is within normal range"}

def check_error_rate():
    # ตรวจสอบ Application Insights error rate
    return {"name": "Error Rate", "passed": True, "message": "Error rate is below threshold"}
```

### Invoke REST API Check

```yaml
# เพิ่ม Invoke REST API check ใน Environment
# Checks & gates > Invoke REST API
# URL: https://my-function-app.azurewebsites.net/api/gate
# Method: POST
# Headers: Content-Type: application/json
# Success criteria: eq(root['status'], 'succeeded')
```

---

## Bicep Integration

**Bicep** เป็น DSL สำหรับ Azure Infrastructure as Code ที่ง่ายกว่า ARM templates

### ตัวอย่าง Bicep สำหรับ App Service

```bicep
// infrastructure/main.bicep
@description('Environment name')
param environment string = 'production'

@description('Location for all resources')
param location string = resourceGroup().location

@description('App Service Plan SKU')
@allowed(['F1', 'B1', 'B2', 'S1', 'S2', 'P1v3'])
param skuName string = 'B1'

@description('Container image')
param containerImage string

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-${environment}'
  location: location
  sku: {
    name: skuName
    tier: skuName == 'F1' ? 'Free' : 'Basic'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

// App Service
resource webApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'app-${environment}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOCKER|${containerImage}'
      appSettings: [
        {
          name: 'WEBSITE_ENABLE_SYNC_UPDATE_SITE'
          value: 'true'
        }
        {
          name: 'NODE_ENV'
          value: environment
        }
      ]
      healthCheckPath: '/health'
    }
    httpsOnly: true
  }
}

// Output
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
output webAppName string = webApp.name
```

### Deploy Bicep ใน Pipeline

```yaml
# azure-pipelines.yml
stages:
  - stage: Infrastructure
    displayName: 'Deploy Infrastructure'
    jobs:
      - job: DeployBicep
        displayName: 'Deploy Azure Infrastructure'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          # Validate Bicep
          - task: AzureCLI@2
            displayName: 'Validate Bicep'
            inputs:
              azureSubscription: 'Azure-Service-Connection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep version
                az bicep build --file infrastructure/main.bicep
                az deployment group validate \
                  --resource-group my-app-rg \
                  --template-file infrastructure/main.bicep \
                  --parameters environment=production containerImage=$(dockerImage)
          
          # What-if (Preview changes)
          - task: AzureCLI@2
            displayName: 'Preview Infrastructure Changes'
            inputs:
              azureSubscription: 'Azure-Service-Connection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az deployment group what-if \
                  --resource-group my-app-rg \
                  --template-file infrastructure/main.bicep \
                  --parameters environment=production containerImage=$(dockerImage)
          
          # Deploy Bicep
          - task: AzureResourceManagerTemplateDeployment@3
            displayName: 'Deploy Bicep Template'
            inputs:
              deploymentScope: 'Resource Group'
              azureResourceManagerConnection: 'Azure-Service-Connection'
              subscriptionId: '$(subscriptionId)'
              action: 'Create Or Update Resource Group'
              resourceGroupName: 'my-app-rg'
              location: 'Southeast Asia'
              templateLocation: 'Linked artifact'
              csmFile: 'infrastructure/main.bicep'
              overrideParameters: >
                -environment production
                -containerImage $(dockerImage)
              deploymentMode: 'Incremental'
              deploymentOutputs: 'infraOutputs'
          
          # ดึง outputs จาก deployment
          - bash: |
              echo "Infrastructure outputs: $(infraOutputs)"
              WEB_APP_URL=$(echo '$(infraOutputs)' | jq -r '.webAppUrl.value')
              echo "##vso[task.setvariable variable=webAppUrl;isOutput=true]$WEB_APP_URL"
            name: ParseOutputs
            displayName: 'Parse Infrastructure Outputs'
```

### Bicep Modules

```bicep
// modules/networking.bicep
@description('VNet name')
param vnetName string

@description('Location')
param location string = resourceGroup().location

resource vnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'app-subnet'
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
      {
        name: 'db-subnet'
        properties: {
          addressPrefix: '10.0.2.0/24'
          serviceEndpoints: [
            {
              service: 'Microsoft.Sql'
            }
          ]
        }
      }
    ]
  }
}

output vnetId string = vnet.id
output appSubnetId string = vnet.properties.subnets[0].id
```

```bicep
// infrastructure/main.bicep
module networking 'modules/networking.bicep' = {
  name: 'networking'
  params: {
    vnetName: 'vnet-${environment}'
    location: location
  }
}

// ใช้ output จาก module
resource appService 'Microsoft.Web/sites@2023-01-01' = {
  name: 'app-${environment}'
  location: location
  properties: {
    virtualNetworkSubnetId: networking.outputs.appSubnetId
    // ...
  }
}
```

---

## Azure Boards Integration

### Link Pipeline ไปยัง Work Items

```yaml
# azure-pipelines.yml
steps:
  - task: WorkItemQuery@0
    displayName: 'Get related work items'
    inputs:
      queryType: 'wiql'
      wiql: |
        SELECT [System.Id] 
        FROM WorkItems 
        WHERE [System.TeamProject] = @project 
        AND [System.State] != 'Closed'
        AND [System.Tags] CONTAINS 'deploy'
```

### Auto-update Work Items เมื่อ Deploy

```yaml
steps:
  - task: AzureCLI@2
    displayName: 'Update Work Items'
    inputs:
      azureSubscription: 'Azure-Service-Connection'
      scriptType: 'bash'
      scriptLocation: 'inlineScript'
      inlineScript: |
        # ดึง work item IDs จาก commit messages
        COMMIT_MSG=$(git log --pretty=format:"%s" -10)
        WORK_ITEM_IDS=$(echo "$COMMIT_MSG" | grep -oP '(?<=#)\d+' | sort -u)
        
        for ID in $WORK_ITEM_IDS; do
          echo "Updating work item #$ID"
          
          # อัปเดต state เป็น Deployed
          az boards work-item update \
            --id $ID \
            --fields "System.State=Deployed" \
            --discussion "Deployed in build $(Build.BuildNumber)"
        done
```

---

## Azure Artifacts

Azure Artifacts เป็น package manager สำหรับ npm, NuGet, Maven, Python, Universal packages

### สร้าง Feed

```bash
az artifacts universal download \
  --organization https://dev.azure.com/myorg \
  --project myproject \
  --scope project \
  --feed my-feed \
  --name my-package \
  --version 1.0.0 \
  --path ./downloaded
```

### ใช้ Artifacts ใน Pipeline

```yaml
# azure-pipelines.yml
# ตั้งค่า npm registry ให้ใช้ Azure Artifacts
steps:
  - task: npmAuthenticate@0
    inputs:
      workingFile: '.npmrc'

  # .npmrc ใน project
  # registry=https://pkgs.dev.azure.com/myorg/myproject/_packaging/my-feed/npm/registry/

  - task: Npm@1
    displayName: 'npm install'
    inputs:
      command: 'install'
      customEndpoint: 'Azure-Artifacts-Connection'

  # Publish npm package
  - task: Npm@1
    displayName: 'npm publish'
    inputs:
      command: 'publish'
      publishRegistry: 'useFeed'
      publishFeed: 'my-feed'
```

```yaml
# NuGet
- task: NuGetAuthenticate@1

- task: NuGetCommand@2
  displayName: 'Restore NuGet packages'
  inputs:
    command: 'restore'
    feedsToUse: 'select'
    vstsFeed: 'my-project/my-feed'

- task: NuGetCommand@2
  displayName: 'Push to Azure Artifacts'
  inputs:
    command: 'push'
    packagesToPush: '$(Build.ArtifactStagingDirectory)/**/*.nupkg'
    nuGetFeedType: 'internal'
    publishVstsFeed: 'my-project/my-feed'
```

---

## แบบฝึกหัด

### Exercise 1: สร้าง CI Pipeline สำหรับ Node.js

สร้าง `azure-pipelines.yml` ที่:
1. Trigger เมื่อ push ไปยัง main หรือ feature branches
2. Install dependencies
3. Run tests พร้อม code coverage
4. Build application
5. Publish build artifacts

```yaml
# โซลูชัน
trigger:
  branches:
    include:
      - main
      - feature/*

pool:
  vmImage: 'ubuntu-latest'

variables:
  nodeVersion: '18.x'

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '$(nodeVersion)'
    displayName: 'Install Node.js $(nodeVersion)'

  - script: npm ci
    displayName: 'Install dependencies'

  - script: npm run lint
    displayName: 'Run ESLint'

  - script: npm test -- --coverage --ci
    displayName: 'Run tests'
    env:
      CI: 'true'

  - task: PublishTestResults@2
    condition: always()
    inputs:
      testResultsFormat: 'JUnit'
      testResultsFiles: 'coverage/junit.xml'

  - task: PublishCodeCoverageResults@2
    inputs:
      summaryFileLocation: 'coverage/cobertura-coverage.xml'

  - script: npm run build
    displayName: 'Build'

  - task: ArchiveFiles@2
    inputs:
      rootFolderOrFile: 'dist'
      archiveFile: '$(Build.ArtifactStagingDirectory)/app.zip'

  - publish: '$(Build.ArtifactStagingDirectory)/app.zip'
    artifact: 'drop'
```

### Exercise 2: Multi-Stage Deployment Pipeline

ต่อยอดจาก Exercise 1 โดยเพิ่ม deployment stages:

```yaml
# เพิ่มใน azure-pipelines.yml
stages:
  - stage: Build
    # ... (จาก Exercise 1)
  
  - stage: DeployDev
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Deploy
        environment: 'development'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: 'drop'
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'YOUR-SERVICE-CONNECTION'
                    appName: 'my-app-dev'
                    package: '$(Pipeline.Workspace)/drop/app.zip'
  
  - stage: DeployProd
    dependsOn: DeployDev
    jobs:
      - deployment: Deploy
        environment: 'production'  # เพิ่ม approval ใน Azure DevOps
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: 'drop'
                - task: AzureWebApp@1
                  inputs:
                    azureSubscription: 'YOUR-SERVICE-CONNECTION'
                    appName: 'my-app-prod'
                    package: '$(Pipeline.Workspace)/drop/app.zip'
```

### Exercise 3: Infrastructure as Code ด้วย Bicep

สร้าง Bicep template สำหรับ deploy App Service พร้อม pipeline:

1. สร้าง `infrastructure/app.bicep`
2. สร้าง `azure-pipelines-infra.yml`
3. Test ด้วย what-if mode
4. Deploy ด้วย Incremental mode

---

## สรุป

Azure DevOps เป็น complete DevOps platform ที่:

1. **Azure Boards** → Work item tracking, sprint planning, Kanban
2. **Azure Repos** → Git repository hosting พร้อม code review
3. **Azure Pipelines** → CI/CD ด้วย YAML pipeline as code
4. **Azure Artifacts** → Package management (npm, NuGet, Maven, pip)
5. **Test Plans** → Test case management และ manual testing
6. **Environments** → Deployment targets พร้อม approval gates
7. **Service Connections** → Secure external service integration
8. **Variable Groups** → Shared configuration และ secrets
9. **Bicep** → Infrastructure as Code สำหรับ Azure
10. **Release Gates** → Automated quality gates

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ **Monitoring & Observability** ซึ่งเป็นสิ่งสำคัญหลัง deploy application เสร็จแล้ว

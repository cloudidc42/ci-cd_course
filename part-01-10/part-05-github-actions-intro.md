# Part 05: Introduction to GitHub Actions

## บทนำ

GitHub Actions เป็น platform CI/CD ที่ทรงพลังและยืดหยุ่น ซึ่ง integrate โดยตรงกับ GitHub repository ช่วยให้เราสามารถ automate workflows ตั้งแต่การ test code ไปจนถึงการ deploy application โดยไม่ต้องออกจาก GitHub

**สิ่งที่จะได้เรียนรู้:**
- GitHub Actions คืออะไรและทำงานอย่างไร
- สถาปัตยกรรม: Workflows, Jobs, Steps, Actions
- YAML syntax สำหรับการเขียน workflows
- Triggers ประเภทต่างๆ
- GitHub-hosted vs Self-hosted Runners
- GitHub Actions Marketplace
- Context variables และ expressions
- การ debug workflows
- สร้าง Hello World workflow

---

## 5.1 GitHub Actions คืออะไร?

GitHub Actions เป็นบริการ CI/CD (Continuous Integration/Continuous Deployment) ที่ built-in อยู่ใน GitHub ช่วยให้คุณสามารถ:

- **Automate** workflows ต่างๆ ใน repository
- **Build, test, and deploy** code อัตโนมัติ
- **React to events** ใน GitHub เช่น push, pull request, issue
- **Schedule tasks** ด้วย cron expressions
- **Integrate** กับ services ภายนอกผ่าน Actions marketplace

### GitHub Actions vs Traditional CI/CD

| Feature | Traditional CI/CD | GitHub Actions |
|---------|-------------------|----------------|
| Integration | ต้องตั้งค่าแยก | Built-in กับ GitHub |
| Configuration | XML/Groovy (Jenkins) | YAML |
| Marketplace | Limited | 20,000+ Actions |
| Cost | Self-hosted มีค่าใช้จ่าย | ฟรีสำหรับ public repos |
| Setup | Complex | Simple |
| Runners | ต้องดูแลเอง | Managed by GitHub |

### ประโยชน์ของ GitHub Actions

1. **Zero infrastructure** - ไม่ต้องตั้งค่า CI server
2. **Native GitHub integration** - เข้าถึง repo context ได้โดยตรง
3. **Rich marketplace** - Actions สำเร็จรูปกว่า 20,000 รายการ
4. **Matrix builds** - ทดสอบบนหลาย environments พร้อมกัน
5. **Reusable workflows** - แชร์ workflows ระหว่าง repositories

---

## 5.2 สถาปัตยกรรมของ GitHub Actions

### ภาพรวม

```
GitHub Repository
└── .github/
    └── workflows/
        ├── ci.yml          ← Workflow file
        ├── cd.yml
        └── release.yml
```

### Components หลัก

```
Workflow
├── Trigger (on:)
│   ├── push
│   ├── pull_request
│   ├── schedule
│   └── workflow_dispatch
│
└── Jobs (jobs:)
    ├── job-1
    │   ├── runs-on: ubuntu-latest
    │   └── steps:
    │       ├── step 1: actions/checkout@v4
    │       ├── step 2: setup-node@v4
    │       └── step 3: run tests
    │
    └── job-2
        ├── needs: [job-1]      ← dependency
        ├── runs-on: ubuntu-latest
        └── steps:
            └── step 1: deploy
```

### Workflows

Workflow คือ automated process ที่กำหนดไว้ใน YAML file ใน directory `.github/workflows/`

**คุณสมบัติของ Workflow:**
- แต่ละ workflow มีชื่อ (name)
- ถูก trigger ด้วย events หรือ schedule
- ประกอบด้วย jobs อย่างน้อย 1 job
- สามารถมี secrets และ environment variables

### Jobs

Job คือชุดของ steps ที่รันบน runner เดียวกัน

**คุณสมบัติของ Jobs:**
- รันแบบ parallel โดย default
- สามารถกำหนด dependency ระหว่าง jobs ด้วย `needs`
- แต่ละ job มี environment ของตัวเอง
- สามารถ share data ด้วย artifacts

### Steps

Step คือ task แต่ละอย่างภายใน job

**ประเภทของ Steps:**
1. **Run** - รัน shell commands
2. **Uses** - ใช้ action จาก marketplace หรือ repository

### Actions

Action คือ reusable unit ที่ทำงาน specific task

**ประเภทของ Actions:**
1. **JavaScript actions** - รันบน runner โดยตรง
2. **Docker container actions** - รันใน Docker container
3. **Composite actions** - รวม actions หลายๆ อัน

---

## 5.3 YAML Syntax สำหรับ Workflows

### โครงสร้างพื้นฐาน

```yaml
# .github/workflows/basic.yml

# ชื่อ workflow (จะแสดงใน GitHub UI)
name: Basic Workflow

# Trigger - เมื่อไหร่ workflow จะรัน
on:
  push:
    branches:
      - main

# Jobs
jobs:
  # Job name
  my-job:
    # Runner ที่ใช้
    runs-on: ubuntu-latest

    # Steps ใน job
    steps:
      # Step ที่ 1
      - name: Check out code
        uses: actions/checkout@v4

      # Step ที่ 2
      - name: Say Hello
        run: echo "Hello, World!"
```

### YAML ที่ต้องรู้

```yaml
# Comments ใช้ #
# key: value
name: My Workflow

# Lists
on:
  - push
  - pull_request

# Nested objects
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      NODE_ENV: production
      API_URL: https://api.example.com

# Multi-line strings
steps:
  - name: Multiple commands
    run: |
      echo "Line 1"
      echo "Line 2"
      npm install
      npm test

  - name: Single command
    run: echo "Single line"

# Inline list
matrix:
  node: [16, 18, 20]

# Boolean
continue-on-error: true
fail-fast: false

# Numbers
timeout-minutes: 30

# References
steps:
  - name: Setup Node
    uses: actions/setup-node@v4
    with:
      node-version: ${{ matrix.node }}  # Expression
```

---

## 5.4 Triggers (on:)

### Push Trigger

```yaml
on:
  push:
    # เฉพาะ branches เหล่านี้
    branches:
      - main
      - develop
      - 'feature/**'    # pattern matching
      - 'release/*'

    # ยกเว้น branches เหล่านี้
    branches-ignore:
      - 'docs/**'

    # เฉพาะเมื่อ files เหล่านี้เปลี่ยน
    paths:
      - 'src/**'
      - '*.ts'
      - 'package.json'

    # ยกเว้น paths เหล่านี้
    paths-ignore:
      - '**.md'
      - 'docs/**'

    # เฉพาะ tags
    tags:
      - 'v*'
      - 'v[0-9]+.[0-9]+.[0-9]+'
```

### Pull Request Trigger

```yaml
on:
  pull_request:
    branches:
      - main
      - develop

    # PR events ที่ trigger
    types:
      - opened       # เปิด PR ใหม่
      - synchronize  # push commits ใหม่
      - reopened     # เปิด PR ที่ปิดไปแล้ว
      - ready_for_review  # เปลี่ยนจาก draft เป็น ready
      - labeled      # เพิ่ม label

    # ยกเว้น paths เหล่านี้
    paths-ignore:
      - '**.md'
```

### Schedule Trigger

```yaml
on:
  schedule:
    # รัน ทุกวัน เวลา 08:00 UTC
    - cron: '0 8 * * *'

    # รัน ทุก 15 นาที
    - cron: '*/15 * * * *'

    # รัน ทุกวันจันทร์ เวลา 09:00 UTC
    - cron: '0 9 * * 1'

    # รัน ทุกวันทำงาน (จ-ศ) เวลา 07:00 UTC
    - cron: '0 7 * * 1-5'
```

Cron Format: `minute hour day-of-month month day-of-week`
```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of the month (1 - 31)
│ │ │ ┌───────────── month (1 - 12 or JAN-DEC)
│ │ │ │ ┌───────────── day of the week (0 - 6 or SUN-SAT)
│ │ │ │ │
* * * * *
```

### Workflow Dispatch (Manual Trigger)

```yaml
on:
  workflow_dispatch:
    # รับ inputs จาก user
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

      version:
        description: 'Version to deploy'
        required: true
        type: string

      debug:
        description: 'Enable debug mode'
        required: false
        default: false
        type: boolean

      runner:
        description: 'Runner to use'
        required: false
        type: choice
        options:
          - ubuntu-latest
          - windows-latest
          - macos-latest
```

### Multiple Triggers

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:
```

### Release Trigger

```yaml
on:
  release:
    types:
      - published   # เผยแพร่ release
      - created     # สร้าง release
      - edited      # แก้ไข release
```

### Issue Trigger

```yaml
on:
  issues:
    types:
      - opened
      - closed
      - labeled

  issue_comment:
    types:
      - created
```

---

## 5.5 Runners

### GitHub-hosted Runners

GitHub จัดหา runners ให้พร้อมใช้งาน:

```yaml
jobs:
  linux-job:
    runs-on: ubuntu-latest   # Ubuntu (ล่าสุด)
    # หรือระบุ version:
    # runs-on: ubuntu-22.04
    # runs-on: ubuntu-20.04

  windows-job:
    runs-on: windows-latest  # Windows Server
    # runs-on: windows-2022
    # runs-on: windows-2019

  mac-job:
    runs-on: macos-latest    # macOS (ล่าสุด)
    # runs-on: macos-14
    # runs-on: macos-13
```

**Specs ของ GitHub-hosted Runners:**
- 2 CPU cores
- 7 GB RAM
- 14 GB SSD storage
- รัน Ubuntu, Windows, หรือ macOS
- ฟรีสำหรับ public repositories
- Private repos: 2,000 minutes/month (free plan)

### Larger Runners (GitHub Teams/Enterprise)

```yaml
jobs:
  large-job:
    runs-on: ubuntu-latest-4-cores   # 4 cores
    # runs-on: ubuntu-latest-8-cores   # 8 cores
    # runs-on: ubuntu-latest-16-cores  # 16 cores
```

### Self-hosted Runners

```yaml
jobs:
  self-hosted-job:
    runs-on: self-hosted

  # ระบุ labels ของ runner
  labeled-runner:
    runs-on: [self-hosted, linux, x64, gpu]
```

**ติดตั้ง Self-hosted Runner:**

```bash
# ไปที่ GitHub Repository > Settings > Actions > Runners > New self-hosted runner

# Download runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.313.0.tar.gz \
  -L https://github.com/actions/runner/releases/download/v2.313.0/actions-runner-linux-x64-2.313.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.313.0.tar.gz

# Configure runner
./config.sh --url https://github.com/YOUR_ORG/YOUR_REPO \
  --token YOUR_TOKEN

# ติดตั้งเป็น service
sudo ./svc.sh install
sudo ./svc.sh start

# ตรวจสอบ status
sudo ./svc.sh status
```

---

## 5.6 Actions Marketplace

### ค้นหาและใช้ Actions

ไปที่ [GitHub Marketplace](https://github.com/marketplace?type=actions) เพื่อค้นหา actions

### Popular Actions

```yaml
steps:
  # Checkout repository
  - uses: actions/checkout@v4

  # Setup Node.js
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
      cache: 'npm'

  # Setup Python
  - uses: actions/setup-python@v5
    with:
      python-version: '3.12'
      cache: 'pip'

  # Setup Java
  - uses: actions/setup-java@v4
    with:
      java-version: '21'
      distribution: 'temurin'

  # Setup Go
  - uses: actions/setup-go@v5
    with:
      go-version: '1.22'
      cache: true

  # Cache dependencies
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}

  # Upload artifact
  - uses: actions/upload-artifact@v4
    with:
      name: build-output
      path: dist/

  # Download artifact
  - uses: actions/download-artifact@v4
    with:
      name: build-output

  # Create GitHub Release
  - uses: actions/create-release@v1
    with:
      tag_name: ${{ github.ref }}
      release_name: Release ${{ github.ref }}

  # Deploy to GitHub Pages
  - uses: peaceiris/actions-gh-pages@v3
    with:
      github_token: ${{ secrets.GITHUB_TOKEN }}
      publish_dir: ./dist

  # Send Slack notification
  - uses: slackapi/slack-github-action@v1.25.0
    with:
      channel-id: 'C12345'
      slack-message: 'Build completed!'
    env:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### การระบุ Version ของ Actions

```yaml
# ใช้ specific version (recommended สำหรับ production)
- uses: actions/checkout@v4.1.1

# ใช้ major version (รับ bug fixes อัตโนมัติ)
- uses: actions/checkout@v4

# ใช้ branch (ไม่แนะนำ - อาจเปลี่ยนแปลงได้)
- uses: actions/checkout@main

# ใช้ commit SHA (most secure)
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

---

## 5.7 Context Variables และ Expressions

### GitHub Context

```yaml
steps:
  - name: Show context
    run: |
      echo "Repository: ${{ github.repository }}"
      echo "Owner: ${{ github.repository_owner }}"
      echo "Branch: ${{ github.ref_name }}"
      echo "SHA: ${{ github.sha }}"
      echo "Actor: ${{ github.actor }}"
      echo "Event: ${{ github.event_name }}"
      echo "Workflow: ${{ github.workflow }}"
      echo "Run ID: ${{ github.run_id }}"
      echo "Run Number: ${{ github.run_number }}"
```

### Contexts ทั้งหมด

```yaml
# github context - ข้อมูล GitHub repository และ event
github.event_name     # push, pull_request, etc.
github.ref            # refs/heads/main
github.ref_name       # main
github.sha            # commit SHA
github.actor          # ผู้ trigger workflow
github.repository     # owner/repo-name
github.workspace      # path ของ working directory

# env context - environment variables
env.MY_VAR

# vars context - repository/organization variables
vars.MY_VAR

# secrets context - secrets
secrets.MY_SECRET
secrets.GITHUB_TOKEN  # auto-generated token

# runner context - ข้อมูล runner
runner.os             # Linux, Windows, macOS
runner.arch           # X64, ARM, ARM64
runner.temp           # temporary directory path
runner.tool_cache     # tool cache directory

# job context - ข้อมูล current job
job.status            # success, failure, cancelled

# steps context - ข้อมูล steps ที่รันแล้ว
steps.my-step.outputs.my-output
steps.my-step.conclusion  # success, failure, skipped

# matrix context - matrix values
matrix.node
matrix.os
```

### Expressions

```yaml
# Syntax: ${{ expression }}

# Operators
${{ 1 + 2 }}
${{ 'hello' == 'hello' }}
${{ github.event_name != 'push' }}
${{ true && false }}
${{ true || false }}
${{ !true }}

# Functions
${{ contains(github.ref, 'main') }}
${{ startsWith(github.ref, 'refs/tags/') }}
${{ endsWith(github.actor, '-bot') }}
${{ format('Hello {0}!', 'world') }}
${{ join(matrix.os, ', ') }}
${{ toJSON(github.event) }}
${{ fromJSON('{"key": "value"}').key }}
${{ hashFiles('**/package-lock.json') }}

# Conditional expressions
if: ${{ github.event_name == 'push' }}
if: ${{ success() }}
if: ${{ failure() }}
if: ${{ always() }}
if: ${{ cancelled() }}

# Status functions
success()   # เมื่อ step ก่อนหน้าสำเร็จ
failure()   # เมื่อ step ก่อนหน้า fail
always()    # เสมอ
cancelled() # เมื่อถูก cancel
```

### Conditional Steps

```yaml
steps:
  - name: Run only on main branch
    if: ${{ github.ref == 'refs/heads/main' }}
    run: echo "This is main branch"

  - name: Run only on pull request
    if: ${{ github.event_name == 'pull_request' }}
    run: echo "This is a PR"

  - name: Run on failure
    if: ${{ failure() }}
    run: echo "Something went wrong!"

  - name: Run always
    if: ${{ always() }}
    run: echo "I always run"

  - name: Complex condition
    if: |
      ${{ github.event_name == 'push' &&
          github.ref == 'refs/heads/main' &&
          !contains(github.actor, '-bot') }}
    run: echo "Push to main by human"
```

### Environment Variables

```yaml
# Global env (ทุก jobs)
env:
  NODE_ENV: production
  API_URL: https://api.example.com

jobs:
  build:
    # Job-level env
    env:
      BUILD_ENV: production

    steps:
      - name: Run with env
        # Step-level env
        env:
          STEP_VAR: hello
        run: |
          echo "Global: $NODE_ENV"
          echo "Job: $BUILD_ENV"
          echo "Step: $STEP_VAR"
          echo "Secret: $MY_SECRET"
        env:
          MY_SECRET: ${{ secrets.MY_SECRET }}
```

### Setting Output Variables

```yaml
steps:
  - name: Set output
    id: set-version
    run: |
      VERSION=$(cat package.json | jq -r .version)
      echo "version=$VERSION" >> $GITHUB_OUTPUT

  - name: Use output
    run: echo "Version is ${{ steps.set-version.outputs.version }}"
```

### Setting Environment Variables Dynamically

```yaml
steps:
  - name: Set env var
    run: echo "BUILD_DATE=$(date +'%Y-%m-%d')" >> $GITHUB_ENV

  - name: Use env var
    run: echo "Build date: $BUILD_DATE"
```

---

## 5.8 Hello World Workflow

### Workflow แรก

```yaml
# .github/workflows/hello-world.yml
name: Hello World

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  hello:
    name: Say Hello
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Say hello
        run: echo "Hello, World!"

      - name: Show system info
        run: |
          echo "OS: $(uname -a)"
          echo "User: $(whoami)"
          echo "Directory: $(pwd)"
          ls -la

      - name: Show GitHub context
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit: ${{ github.sha }}"
          echo "Actor: ${{ github.actor }}"
          echo "Event: ${{ github.event_name }}"
```

### ขั้นตอนการสร้าง Workflow แรก

**ขั้นตอนที่ 1: สร้าง Directory**
```bash
mkdir -p .github/workflows
```

**ขั้นตอนที่ 2: สร้างไฟล์ workflow**
```bash
cat > .github/workflows/hello-world.yml << 'EOF'
name: Hello World

on:
  push:

jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Hello, World!"
EOF
```

**ขั้นตอนที่ 3: Commit และ Push**
```bash
git add .github/workflows/hello-world.yml
git commit -m "ci: add hello world workflow"
git push
```

**ขั้นตอนที่ 4: ดูผลลัพธ์**
- ไปที่ GitHub Repository
- คลิก tab "Actions"
- จะเห็น workflow กำลังรัน

---

## 5.9 Jobs และ Dependencies

### Parallel Jobs

```yaml
name: Parallel Jobs

on: push

jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running unit tests..."
        
  test-integration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running integration tests..."

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running linter..."

# test-unit, test-integration, lint รันพร้อมกัน
```

### Sequential Jobs (Dependencies)

```yaml
name: Sequential Jobs

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Building..."

  test:
    runs-on: ubuntu-latest
    needs: build    # รอ build ก่อน
    steps:
      - run: echo "Testing..."

  deploy:
    runs-on: ubuntu-latest
    needs: [build, test]    # รอทั้ง build และ test
    steps:
      - run: echo "Deploying..."
```

### Sharing Data Between Jobs

```yaml
name: Share Data

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Get version
        id: get-version
        run: |
          VERSION=$(cat package.json | jq -r .version)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Build
        run: |
          npm ci
          npm run build
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/
      
      - name: Deploy
        run: |
          echo "Deploying version ${{ needs.build.outputs.version }}"
          ls dist/
```

---

## 5.10 Matrix Strategy

Matrix strategy ช่วยให้ test บนหลาย configurations พร้อมกัน

```yaml
name: Matrix Build

on: push

jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 21]
        
      # ถ้า job หนึ่ง fail จะหยุดทั้งหมด (default: true)
      fail-fast: false
      
      # จำนวน parallel jobs (default: ขึ้นอยู่กับ plan)
      max-parallel: 6

    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      
      - run: node --version
      - run: npm test

# สร้าง 9 jobs: 3 OS x 3 Node versions
```

### Matrix Include/Exclude

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    node: [18, 20]
    
    # เพิ่ม combination พิเศษ
    include:
      - os: ubuntu-latest
        node: 21
        experimental: true
    
    # ลบ combination ที่ไม่ต้องการ
    exclude:
      - os: windows-latest
        node: 18
```

---

## 5.11 Secrets และ Variables

### ตั้งค่า Secrets

ไปที่ Repository > Settings > Secrets and variables > Actions

```yaml
steps:
  - name: Use secret
    run: |
      # ใช้ผ่าน env variable (recommended)
      echo "API Key: $API_KEY"
    env:
      API_KEY: ${{ secrets.API_KEY }}
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

  - name: Deploy with secrets
    uses: some/deploy-action@v1
    with:
      token: ${{ secrets.DEPLOY_TOKEN }}
```

**สิ่งที่ต้องระวัง:**
```yaml
# ❌ อย่าทำ - secrets จะถูก redact แต่ยังเป็น bad practice
- run: echo "${{ secrets.MY_SECRET }}"

# ✅ ถูกต้อง - ใช้ env variable
- run: echo "$MY_SECRET"
  env:
    MY_SECRET: ${{ secrets.MY_SECRET }}
```

### Repository Variables

```yaml
# ใช้ Variables (non-secret values)
steps:
  - run: echo "Environment: ${{ vars.ENVIRONMENT }}"
  - run: echo "API URL: ${{ vars.API_URL }}"
```

### GITHUB_TOKEN

GitHub สร้าง token อัตโนมัติสำหรับแต่ละ workflow run

```yaml
steps:
  - name: Create issue comment
    uses: actions/github-script@v7
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      script: |
        github.rest.issues.createComment({
          issue_number: context.issue.number,
          owner: context.repo.owner,
          repo: context.repo.repo,
          body: 'Tests passed! ✅'
        })
```

---

## 5.12 Debugging Workflows

### Enable Debug Logging

```bash
# เพิ่ม secret ในชื่อ:
ACTIONS_RUNNER_DEBUG = true
ACTIONS_STEP_DEBUG = true
```

### Debug ด้วย tmate (SSH into runner)

```yaml
steps:
  - uses: actions/checkout@v4
  
  - name: Setup tmate session
    uses: mxschmitt/action-tmate@v3
    if: ${{ failure() }}  # เปิดเฉพาะเมื่อ fail
    with:
      limit-access-to-actor: true
```

### ใช้ act สำหรับ Local Testing

```bash
# ติดตั้ง act
# macOS
brew install act

# Linux
curl https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# ทดสอบ workflow locally
act

# รัน specific workflow
act -W .github/workflows/ci.yml

# รัน specific job
act -j build

# รัน specific event
act push
act pull_request

# ดู events ที่รองรับ
act --list

# Verbose mode
act -v
```

### ใช้ GitHub CLI สำหรับ Debugging

```bash
# ติดตั้ง GitHub CLI
# macOS
brew install gh

# Ubuntu
sudo apt install gh
gh auth login

# ดู workflow runs
gh run list

# ดู workflow run details
gh run view <run-id>

# ดู logs
gh run view <run-id> --log

# ดู logs ที่ fail
gh run view <run-id> --log-failed

# Re-run workflow
gh run rerun <run-id>

# Re-run failed jobs เท่านั้น
gh run rerun <run-id> --failed
```

---

## 5.13 Reusable Workflows

### สร้าง Reusable Workflow

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
      environment:
        required: false
        type: string
        default: 'test'
    secrets:
      npm-token:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      
      - name: Run tests
        run: npm test
        env:
          NODE_ENV: ${{ inputs.environment }}
```

### เรียกใช้ Reusable Workflow

```yaml
# .github/workflows/ci.yml
name: CI

on: push

jobs:
  test-node18:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '18'
      environment: 'test'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}

  test-node20:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '20'
```

---

## 5.14 Composite Actions

### สร้าง Composite Action

```yaml
# .github/actions/setup-and-test/action.yml
name: Setup and Test
description: Setup Node.js and run tests

inputs:
  node-version:
    description: Node.js version to use
    required: true
    default: '20'
  working-directory:
    description: Working directory
    required: false
    default: '.'

outputs:
  test-result:
    description: Test result
    value: ${{ steps.test.outputs.result }}

runs:
  using: composite
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
        cache-dependency-path: ${{ inputs.working-directory }}/package-lock.json

    - name: Install dependencies
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: npm ci

    - name: Run tests
      id: test
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: |
        npm test
        echo "result=success" >> $GITHUB_OUTPUT
```

### ใช้ Composite Action

```yaml
steps:
  - uses: actions/checkout@v4
  
  - uses: ./.github/actions/setup-and-test
    with:
      node-version: '20'
      working-directory: './app'
```

---

## 5.15 Workflow Permissions

### ตั้งค่า Permissions

```yaml
name: Workflow with permissions

on: push

# Permissions ระดับ workflow
permissions:
  contents: read
  issues: write
  pull-requests: write

jobs:
  build:
    runs-on: ubuntu-latest
    
    # Permissions ระดับ job
    permissions:
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      - run: npm build

  create-pr:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Create PR
        run: gh pr create --title "Auto PR"
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Permission Types

```yaml
permissions:
  actions: read | write | none
  checks: read | write | none
  contents: read | write | none
  deployments: read | write | none
  id-token: write | none
  issues: read | write | none
  discussions: read | write | none
  packages: read | write | none
  pages: read | write | none
  pull-requests: read | write | none
  repository-projects: read | write | none
  security-events: read | write | none
  statuses: read | write | none
```

---

## 5.16 Workshop - สร้าง Workflow จริง

### Workshop 1: Multi-language Hello World

```yaml
# .github/workflows/multi-language.yml
name: Multi-language Hello World

on:
  push:
  workflow_dispatch:

jobs:
  node:
    name: Node.js Hello World
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Say hello
        run: node -e "console.log('Hello from Node.js!')"

  python:
    name: Python Hello World
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Say hello
        run: python -c "print('Hello from Python!')"

  go:
    name: Go Hello World
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      
      - name: Say hello
        run: |
          cat > hello.go << 'EOF'
          package main
          import "fmt"
          func main() {
            fmt.Println("Hello from Go!")
          }
          EOF
          go run hello.go
```

### Workshop 2: Scheduled Report Workflow

```yaml
# .github/workflows/daily-report.yml
name: Daily Report

on:
  schedule:
    - cron: '0 9 * * 1-5'  # ทุกวันทำงาน เวลา 9:00 UTC
  workflow_dispatch:

jobs:
  report:
    runs-on: ubuntu-latest
    
    permissions:
      issues: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate report
        id: report
        run: |
          REPORT="Daily Build Report - $(date '+%Y-%m-%d')"
          echo "report=$REPORT" >> $GITHUB_OUTPUT
          
          echo "## Report Content" >> report.txt
          echo "- Date: $(date)" >> report.txt
          echo "- Repository: ${{ github.repository }}" >> report.txt
          echo "- Branch: ${{ github.ref_name }}" >> report.txt
      
      - name: Create issue
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const date = new Date().toISOString().split('T')[0];
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Daily Report - ${date}`,
              body: `Automated daily report for ${date}\n\nWorkflow: ${context.workflow}\nRun: ${context.runNumber}`,
              labels: ['report', 'automated']
            });
```

---

## 5.17 Best Practices

### 1. Pin Action Versions

```yaml
# ❌ ไม่ดี - อาจเปลี่ยนแปลงได้
- uses: actions/checkout@main

# ✅ ดี - ใช้ major version
- uses: actions/checkout@v4

# ✅ ดีที่สุด - pin ด้วย SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

### 2. ใช้ Least Privilege

```yaml
# ❌ ไม่ดี - permissions กว้างเกินไป
permissions:
  contents: write
  issues: write
  pull-requests: write

# ✅ ดี - ใช้เฉพาะที่จำเป็น
permissions:
  contents: read
```

### 3. Cache Dependencies

```yaml
steps:
  - name: Cache node modules
    uses: actions/cache@v4
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-
```

### 4. ใช้ Environment Protection

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://app.example.com
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to production"
```

### 5. Workflow Concurrency

```yaml
# ยกเลิก runs ที่กำลังรันอยู่เมื่อมี run ใหม่
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### 6. Timeout สำหรับ Jobs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30  # job timeout
    steps:
      - name: Long step
        timeout-minutes: 10  # step timeout
        run: ./long-script.sh
```

---

## 5.18 Exercises

### Exercise 1: Hello World Workflow

**โจทย์:** สร้าง workflow ที่:
1. Trigger บน push ทุก branch
2. Print "Hello from [your name]!"
3. แสดง GitHub context (repository, branch, actor)
4. แสดงข้อมูล runner

**Expected output ใน GitHub Actions:**
```
Hello from John Doe!
Repository: johndoe/my-repo
Branch: main
Actor: johndoe
OS: Linux
```

### Exercise 2: Matrix Build

**โจทย์:** สร้าง workflow ที่ทดสอบบน:
- OS: Ubuntu, macOS
- Node.js: 18, 20

โดยให้ fail-fast เป็น false และแสดง matrix values ใน step

### Exercise 3: Scheduled Cleanup

**โจทย์:** สร้าง workflow ที่:
1. รัน ทุกคืน เวลา 23:00 UTC
2. ลิสต์ไฟล์ที่ไม่ได้แก้ไขนาน 30 วัน
3. บันทึก output เป็น artifact
4. ส่ง summary ไปยัง Job Summary

### Exercise 4: Conditional Deployment

**โจทย์:** สร้าง workflow ที่:
1. Build เสมอ เมื่อมี push
2. Deploy ไป staging เฉพาะเมื่อ push ไป `develop` branch
3. Deploy ไป production เฉพาะเมื่อ push ไป `main` branch

---

## 5.19 สรุป

ในบทนี้เราได้เรียนรู้:

1. **GitHub Actions คืออะไร** - platform CI/CD ที่ integrate กับ GitHub
2. **สถาปัตยกรรม** - Workflows, Jobs, Steps, Actions
3. **YAML Syntax** - วิธีเขียน workflow files
4. **Triggers** - push, pull_request, schedule, workflow_dispatch
5. **Runners** - GitHub-hosted vs Self-hosted
6. **Actions Marketplace** - ใช้ actions สำเร็จรูป
7. **Contexts และ Expressions** - เข้าถึงข้อมูล workflow
8. **Debugging** - วิธี debug workflows
9. **Best Practices** - การเขียน workflow ที่ดี

### Quick Reference

```yaml
# Minimal workflow
name: My Workflow
on: push
jobs:
  my-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Hello!"

# Common patterns
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
```

### แหล่งข้อมูลเพิ่มเติม

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [GitHub Actions Cheatsheet](https://github.com/nickkatsios/Github-Actions-CheatSheet)
- [act - Local GitHub Actions](https://github.com/nektos/act)
- [actionlint - GitHub Actions Linter](https://github.com/rhysd/actionlint)

---

**ต่อไป:** Part 06 - สร้าง CI Pipeline แรก

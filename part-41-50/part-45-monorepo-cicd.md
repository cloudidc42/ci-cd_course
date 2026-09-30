# Part 45: Monorepo CI/CD

## สารบัญ
1. [Monorepo คืออะไร?](#monorepo-คืออะไร)
2. [ปัญหาของ Monorepo ใน CI/CD](#ปัญหาของ-monorepo-ใน-cicd)
3. [Path-Based Triggering](#path-based-triggering)
4. [Nx](#nx)
5. [Turborepo](#turborepo)
6. [Bazel](#bazel)
7. [Caching ใน Monorepo](#caching-ใน-monorepo)
8. [Versioning ใน Monorepo](#versioning-ใน-monorepo)
9. [Exercises](#exercises)

---

## Monorepo คืออะไร?

Monorepo คือการเก็บ code ของหลาย projects/packages ไว้ใน Git repository เดียวกัน

### โครงสร้าง Monorepo ทั่วไป

```
my-monorepo/
├── apps/
│   ├── web/              # Next.js frontend
│   ├── mobile/           # React Native app
│   ├── api/              # Backend API
│   └── admin/            # Admin dashboard
├── packages/
│   ├── ui/               # Shared UI components
│   ├── utils/            # Shared utilities
│   ├── types/            # Shared TypeScript types
│   └── config/           # Shared configurations
├── services/
│   ├── auth/             # Auth microservice
│   ├── payment/          # Payment microservice
│   └── notification/     # Notification service
├── tools/
│   └── scripts/          # Build scripts
├── package.json          # Root package.json
├── turbo.json            # Turborepo config
└── nx.json               # Nx config (alternative)
```

### Polyrepo vs Monorepo

| ด้าน | Polyrepo | Monorepo |
|------|----------|----------|
| Code sharing | ต้องใช้ npm packages | ตรงไปตรงมา |
| Dependency management | ซับซ้อน (cross-repo) | ง่ายกว่า |
| CI/CD | แยกต่างหาก per repo | ซับซ้อนกว่า |
| Code review | แยก per repo | รวมกัน |
| Visibility | จำกัด | เห็นทุก project |
| Scale | ดีสำหรับ large teams | อาจช้าสำหรับ very large |
| Popular users | - | Google, Facebook, Microsoft |

---

## ปัญหาของ Monorepo ใน CI/CD

### ปัญหาหลัก: Build ทุกอย่างทุกครั้ง

```
❌ ปัญหาเดิม:
สมมติ developer แก้ไข apps/web เพียงไฟล์เดียว

CI Pipeline:
├── Build apps/web     ✅ จำเป็น
├── Build apps/mobile  ❌ ไม่จำเป็น
├── Build apps/api     ❌ ไม่จำเป็น
├── Build apps/admin   ❌ ไม่จำเป็น
├── Build packages/ui  ❌ ไม่จำเป็น
└── Build packages/utils ❌ ไม่จำเป็น

เวลา: 30 นาที (ทั้งที่ควรใช้แค่ 5 นาที)
```

### ปัญหาที่ 2: Dependency Graph ซับซ้อน

```
apps/web ──depends on──> packages/ui
          ──depends on──> packages/utils

apps/api ──depends on──> packages/utils
          ──depends on──> services/auth

ถ้า packages/utils เปลี่ยน:
- ต้อง rebuild apps/web
- ต้อง rebuild apps/api
- ต้อง rebuild services/auth (ด้วย?)
```

### ปัญหาที่ 3: CI Time เพิ่มขึ้นเรื่อยๆ

```
ตอนเริ่ม:     5 packages  → CI ใช้เวลา 5 นาที
6 เดือนต่อมา: 20 packages → CI ใช้เวลา 20 นาที
1 ปีต่อมา:    50 packages → CI ใช้เวลา 1+ ชั่วโมง
```

---

## Path-Based Triggering

### GitHub Actions Path Filters

```yaml
# .github/workflows/web-app.yaml
name: Web App CI

on:
  push:
    branches: [main]
    paths:
    - 'apps/web/**'
    - 'packages/ui/**'      # dependency ของ web
    - 'packages/utils/**'   # dependency ของ web
  pull_request:
    paths:
    - 'apps/web/**'
    - 'packages/ui/**'
    - 'packages/utils/**'

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: cd apps/web && npm ci && npm test && npm run build
```

```yaml
# .github/workflows/api.yaml
name: API CI

on:
  push:
    branches: [main]
    paths:
    - 'apps/api/**'
    - 'packages/utils/**'
    - 'services/auth/**'

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: cd apps/api && npm ci && npm test && npm run build
```

### Path Filter ที่ซับซ้อนกว่า

```yaml
# .github/workflows/smart-ci.yaml
name: Smart CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      web: ${{ steps.filter.outputs.web }}
      api: ${{ steps.filter.outputs.api }}
      mobile: ${{ steps.filter.outputs.mobile }}
      shared: ${{ steps.filter.outputs.shared }}
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: dorny/paths-filter@v3
      id: filter
      with:
        filters: |
          web:
            - 'apps/web/**'
            - 'packages/ui/**'
            - 'packages/utils/**'
          api:
            - 'apps/api/**'
            - 'packages/utils/**'
            - 'services/auth/**'
          mobile:
            - 'apps/mobile/**'
            - 'packages/utils/**'
          shared:
            - 'packages/**'

  # Web App - รันเฉพาะเมื่อมีการเปลี่ยนแปลง
  build-web:
    needs: detect-changes
    if: needs.detect-changes.outputs.web == 'true'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build web
      run: |
        cd apps/web
        npm ci
        npm test
        npm run build

  # API - รันเฉพาะเมื่อมีการเปลี่ยนแปลง
  build-api:
    needs: detect-changes
    if: needs.detect-changes.outputs.api == 'true'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build API
      run: |
        cd apps/api
        npm ci
        npm test
        npm run build

  # Mobile - รันเฉพาะเมื่อมีการเปลี่ยนแปลง
  build-mobile:
    needs: detect-changes
    if: needs.detect-changes.outputs.mobile == 'true'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build mobile
      run: |
        cd apps/mobile
        npm ci
        npm test
```

---

## Nx

Nx เป็น build tool ที่ทรงพลังสำหรับ monorepos

### Setup Nx

```bash
# สร้าง Nx workspace ใหม่
npx create-nx-workspace@latest my-workspace \
  --preset=ts \
  --nx-cloud=true

# เพิ่ม apps ในภายหลัง
cd my-workspace
npx nx generate @nx/next:app web
npx nx generate @nx/node:app api
npx nx generate @nx/react-native:app mobile
npx nx generate @nx/js:lib utils
npx nx generate @nx/react:lib ui
```

### nx.json Configuration

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "defaultBase": "main",
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/?(*.)+(spec|test).[jt]s?(x)?(.snap)",
      "!{projectRoot}/tsconfig.spec.json",
      "!{projectRoot}/jest.config.[jt]s",
      "!{projectRoot}/.eslintrc.json"
    ],
    "sharedGlobals": []
  },
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"],
      "cache": true
    },
    "test": {
      "inputs": ["default", "^production", "{workspaceRoot}/jest.preset.js"],
      "cache": true
    },
    "lint": {
      "inputs": ["default", "{workspaceRoot}/.eslintrc.json"],
      "cache": true
    }
  },
  "nxCloudAccessToken": "xxx..."
}
```

### project.json

```json
{
  "name": "web",
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "sourceRoot": "apps/web/src",
  "projectType": "application",
  "targets": {
    "build": {
      "executor": "@nx/next:build",
      "outputs": ["{options.outputPath}"],
      "defaultConfiguration": "production",
      "options": {
        "outputPath": "dist/apps/web"
      },
      "configurations": {
        "development": {
          "outputPath": "dist/apps/web"
        },
        "production": {
          "outputPath": "dist/apps/web",
          "fileReplacements": [
            {
              "replace": "apps/web/src/environments/environment.ts",
              "with": "apps/web/src/environments/environment.prod.ts"
            }
          ]
        }
      }
    },
    "test": {
      "executor": "@nx/jest:jest",
      "outputs": ["{workspaceRoot}/coverage/{projectRoot}"],
      "options": {
        "jestConfig": "apps/web/jest.config.ts",
        "passWithNoTests": true
      }
    },
    "lint": {
      "executor": "@nx/eslint:lint",
      "outputs": ["{options.outputFile}"],
      "options": {
        "lintFilePatterns": ["apps/web/**/*.{ts,tsx,js,jsx}"]
      }
    }
  },
  "tags": ["scope:web", "type:app"]
}
```

### Nx Commands

```bash
# ดู affected projects
npx nx affected:graph

# Build เฉพาะที่ affected
npx nx affected:build --base=main --head=HEAD

# Test เฉพาะที่ affected
npx nx affected:test --base=main --head=HEAD

# Lint เฉพาะที่ affected
npx nx affected:lint --base=main --head=HEAD

# ดู project graph
npx nx graph

# Run ทุก targets ใน projects ทั้งหมด
npx nx run-many --target=build --all

# Run parallel
npx nx run-many --target=test --all --parallel=4

# Print affected projects
npx nx affected:apps --base=main
npx nx affected:libs --base=main
```

### Nx ใน GitHub Actions

```yaml
# .github/workflows/nx-ci.yaml
name: Nx CI

on:
  push:
    branches: [main]
  pull_request:

env:
  NX_CLOUD_ACCESS_TOKEN: ${{ secrets.NX_CLOUD_ACCESS_TOKEN }}

jobs:
  main:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0   # Nx ต้องการ full history เพื่อ detect affected
    
    # Setup Node.js
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    # Install dependencies
    - run: npm ci
    
    # Set base branch สำหรับ affected calculation
    - name: Set SHAs for affected
      uses: nrwl/nx-set-shas@v4
      # จะ set NX_BASE และ NX_HEAD environment variables
    
    # Run affected lint
    - name: Lint affected
      run: npx nx affected:lint --base=$NX_BASE --head=$NX_HEAD --parallel=4
    
    # Run affected tests
    - name: Test affected
      run: npx nx affected:test --base=$NX_BASE --head=$NX_HEAD --parallel=4 --coverage
    
    # Build affected
    - name: Build affected
      run: npx nx affected:build --base=$NX_BASE --head=$NX_HEAD --parallel=2

  # Docker build สำหรับ affected apps เท่านั้น
  build-images:
    runs-on: ubuntu-latest
    needs: main
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - run: npm ci
    
    - uses: nrwl/nx-set-shas@v4
    
    - name: Get affected apps
      id: affected
      run: |
        APPS=$(npx nx affected:apps --base=$NX_BASE --head=$NX_HEAD --plain)
        echo "APPS=$APPS" >> $GITHUB_OUTPUT
    
    - name: Build Docker images for affected apps
      if: steps.affected.outputs.APPS != ''
      run: |
        for APP in ${{ steps.affected.outputs.APPS }}; do
          echo "Building Docker image for: $APP"
          if [ -f "apps/$APP/Dockerfile" ]; then
            docker build \
              -t ghcr.io/myorg/$APP:${{ github.sha }} \
              -f apps/$APP/Dockerfile \
              --build-arg APP_NAME=$APP \
              .
            docker push ghcr.io/myorg/$APP:${{ github.sha }}
          fi
        done
```

---

## Turborepo

Turborepo เป็น high-performance build tool สำหรับ JavaScript/TypeScript monorepos

### Setup Turborepo

```bash
# เพิ่ม Turborepo ใน existing monorepo
npx turbo@latest init

# หรือสร้าง workspace ใหม่
npx create-turbo@latest my-workspace
```

### turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "globalEnv": ["NODE_ENV", "PORT"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**", "build/**"],
      "env": ["NEXT_PUBLIC_API_URL"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"],
      "inputs": ["src/**/*.tsx", "src/**/*.ts", "test/**/*.ts"]
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "deploy": {
      "dependsOn": ["build", "test", "lint"],
      "outputs": []
    }
  }
}
```

### Turborepo Commands

```bash
# Build ทุก packages
turbo build

# Test แบบ parallel
turbo test

# Run ใน specific packages
turbo build --filter=web
turbo test --filter=...api  # api และ dependencies

# Run affected เปรียบกับ branch
turbo build --filter=[origin/main]

# Dry run
turbo build --dry

# Print graph
turbo build --graph

# Profile
turbo build --profile
```

### Turborepo ใน GitHub Actions

```yaml
# .github/workflows/turborepo-ci.yaml
name: Turborepo CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    name: Build and Test
    timeout-minutes: 15
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    # Remote cache (Vercel Remote Cache)
    - name: Build
      run: npx turbo run build
      env:
        TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
        TURBO_TEAM: ${{ vars.TURBO_TEAM }}
    
    - name: Test
      run: npx turbo run test
      env:
        TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
        TURBO_TEAM: ${{ vars.TURBO_TEAM }}
    
    - name: Lint
      run: npx turbo run lint
      env:
        TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
        TURBO_TEAM: ${{ vars.TURBO_TEAM }}
```

---

## Bazel

Bazel เป็น build tool ของ Google เหมาะกับ large-scale monorepos

### Setup Bazel

```bash
# ติดตั้ง Bazel (Linux/macOS)
brew install bazel
# หรือ
apt-get install bazel

# ตรวจสอบ
bazel version
```

### WORKSPACE

```python
# WORKSPACE
workspace(name = "my_monorepo")

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Node.js rules
http_archive(
    name = "build_bazel_rules_nodejs",
    sha256 = "xxx",
    urls = ["https://github.com/bazelbuild/rules_nodejs/releases/download/5.8.3/rules_nodejs-5.8.3.tar.gz"],
)

load("@build_bazel_rules_nodejs//:repositories.bzl", "build_bazel_rules_nodejs_dependencies")
build_bazel_rules_nodejs_dependencies()

# Container rules
http_archive(
    name = "io_bazel_rules_docker",
    sha256 = "xxx",
    urls = ["https://github.com/bazelbuild/rules_docker/releases/download/v0.25.0/rules_docker-v0.25.0.tar.gz"],
)
```

### BUILD file

```python
# apps/api/BUILD
load("@build_bazel_rules_nodejs//:index.bzl", "nodejs_binary", "nodejs_test")
load("@io_bazel_rules_docker//nodejs:image.bzl", "nodejs_image")

nodejs_binary(
    name = "api",
    entry_point = ":src/main.js",
    deps = [
        "//packages/utils:utils",
        "@npm//express",
    ],
)

nodejs_test(
    name = "api_test",
    entry_point = ":src/main.test.js",
    deps = [
        ":api",
        "@npm//jest",
    ],
)

nodejs_image(
    name = "api_image",
    binary = ":api",
    base = "@node_image//image",
)
```

### Bazel Commands

```bash
# Build specific target
bazel build //apps/api:api

# Build ทั้ง monorepo
bazel build //...

# Test
bazel test //apps/api:api_test
bazel test //...

# Build Docker image
bazel run //apps/api:api_image

# Query dependencies
bazel query "deps(//apps/api:api)"

# Query affected (reverse deps)
bazel query "rdeps(//..., //packages/utils:utils)"

# Show affected projects เมื่อ utils เปลี่ยน
bazel query "kind(nodejs_binary, rdeps(//..., //packages/utils:utils))"
```

### Bazel ใน GitHub Actions

```yaml
# .github/workflows/bazel-ci.yaml
name: Bazel CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  affected-targets:
    runs-on: ubuntu-latest
    outputs:
      targets: ${{ steps.find-affected.outputs.targets }}
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Setup Bazel
      uses: bazelbuild/setup-bazelisk@v3
    
    - name: Find affected targets
      id: find-affected
      run: |
        # ดู files ที่เปลี่ยน
        CHANGED_FILES=$(git diff --name-only origin/main...HEAD)
        
        # หา Bazel targets ที่ affected
        AFFECTED_TARGETS=""
        for file in $CHANGED_FILES; do
          TARGET=$(bazel query --keep_going "rdeps(//..., //${file%.*})" 2>/dev/null || true)
          AFFECTED_TARGETS="$AFFECTED_TARGETS $TARGET"
        done
        
        echo "targets=$AFFECTED_TARGETS" >> $GITHUB_OUTPUT
  
  build-and-test:
    needs: affected-targets
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: bazelbuild/setup-bazelisk@v3
    
    - name: Mount Bazel cache
      uses: actions/cache@v4
      with:
        path: ~/.cache/bazel
        key: bazel-${{ runner.os }}-${{ hashFiles('WORKSPACE', 'WORKSPACE.bazel') }}
        restore-keys: bazel-${{ runner.os }}-
    
    - name: Build affected
      run: |
        if [ -n "${{ needs.affected-targets.outputs.targets }}" ]; then
          bazel build ${{ needs.affected-targets.outputs.targets }}
        fi
    
    - name: Test affected
      run: |
        TARGETS=$(echo "${{ needs.affected-targets.outputs.targets }}" | \
          tr ' ' '\n' | grep '_test$' | tr '\n' ' ')
        if [ -n "$TARGETS" ]; then
          bazel test $TARGETS
        fi
```

---

## Caching ใน Monorepo

### npm/pnpm Workspace Caching

```yaml
# Cache strategy สำหรับ pnpm monorepo
- name: Setup pnpm
  uses: pnpm/action-setup@v4
  with:
    version: 9

- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'pnpm'

- name: Get pnpm store directory
  id: pnpm-cache
  run: |
    echo "STORE_PATH=$(pnpm store path)" >> $GITHUB_OUTPUT

- name: Setup pnpm cache
  uses: actions/cache@v4
  with:
    path: ${{ steps.pnpm-cache.outputs.STORE_PATH }}
    key: ${{ runner.os }}-pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
    restore-keys: |
      ${{ runner.os }}-pnpm-

- name: Install dependencies
  run: pnpm install --frozen-lockfile
```

### Turborepo Remote Cache

```bash
# ใช้ Vercel Remote Cache
TURBO_TOKEN=xxx TURBO_TEAM=myteam turbo build

# Self-hosted cache (ducktape)
# ใช้ Docker image: ducktape/ducktape
docker run -p 8080:8080 ducktape/ducktape

# Set environment variables
export TURBO_API="http://localhost:8080"
export TURBO_TOKEN="my-cache-token"
export TURBO_TEAM="myteam"
turbo build
```

### Nx Cloud Cache

```json
// nx.json
{
  "nxCloudAccessToken": "xxx...",
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx-cloud",
      "options": {
        "cacheableOperations": ["build", "test", "lint", "e2e"],
        "accessToken": "xxx..."
      }
    }
  }
}
```

### Docker Layer Caching ใน Monorepo

```dockerfile
# Dockerfile สำหรับ monorepo
# ใช้ multi-stage build เพื่อ cache dependencies

# Stage 1: Base
FROM node:20-alpine AS base
RUN apk add --no-cache libc6-compat
RUN npm install -g turbo

# Stage 2: Pruner - prune dependencies สำหรับ specific app
FROM base AS pruner
WORKDIR /app
COPY . .
RUN turbo prune --scope=api --docker

# Stage 3: Installer - install only necessary dependencies
FROM base AS installer
WORKDIR /app
# Copy pruned package files
COPY --from=pruner /app/out/json/ .
COPY --from=pruner /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
# Install ด้วย cache
RUN npm install --frozen-lockfile

# Stage 4: Builder
FROM base AS builder
WORKDIR /app
COPY --from=installer /app/node_modules ./node_modules
COPY --from=pruner /app/out/full/ .
COPY turbo.json turbo.json

# Build เฉพาะ app ที่ต้องการ
RUN turbo run build --scope=api

# Stage 5: Runner
FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/apps/api/dist ./

EXPOSE 3000
CMD ["node", "main.js"]
```

---

## Versioning ใน Monorepo

### Independent Versioning

```json
// package.json ของแต่ละ package
{
  "name": "@myorg/utils",
  "version": "1.2.3",   // แต่ละ package มี version ของตัวเอง
  "private": false
}
```

### Changesets (แนะนำสำหรับ npm packages)

```bash
# ติดตั้ง changesets
npm install -D @changesets/cli
npx changeset init

# สร้าง changeset เมื่อทำงาน
npx changeset

# คำตอบ:
# Which packages would you like to include? @myorg/utils
# Which packages should have a major bump? (skip)
# Which packages should have a minor bump? @myorg/utils
# Describe changes: Add new utility function
```

```yaml
# .github/workflows/changesets.yaml
name: Changesets

on:
  push:
    branches:
    - main

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        registry-url: 'https://registry.npmjs.org'
    
    - run: npm ci
    
    - name: Create Release Pull Request or Publish
      uses: changesets/action@v1
      with:
        publish: npm run publish
        version: npm run version
        commit: "chore: update package versions"
        title: "chore: update package versions"
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Nx Release (Nx v17+)

```json
// nx.json
{
  "release": {
    "projects": ["packages/*"],
    "version": {
      "conventionalCommits": true
    },
    "changelog": {
      "workspaceChangelog": {
        "createRelease": "github"
      },
      "projectChangelogs": true
    }
  }
}
```

```bash
# Dry run release
npx nx release --dry-run

# Release ทุก packages ที่เปลี่ยนแปลง
npx nx release

# Release specific version
npx nx release --specifier minor
```

---

## Exercises

### Exercise 1: Setup pnpm Monorepo

```bash
#!/bin/bash
# exercise-1-setup-monorepo.sh

mkdir my-monorepo && cd my-monorepo
git init

# สร้าง pnpm workspace
cat > pnpm-workspace.yaml << 'EOF'
packages:
  - 'apps/*'
  - 'packages/*'
EOF

cat > package.json << 'EOF'
{
  "name": "my-monorepo",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "dev": "turbo dev"
  },
  "devDependencies": {
    "turbo": "latest"
  }
}
EOF

# สร้าง shared packages
mkdir -p packages/utils packages/ui
cat > packages/utils/package.json << 'EOF'
{
  "name": "@myorg/utils",
  "version": "0.0.0",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "test": "jest"
  }
}
EOF

# สร้าง apps
mkdir -p apps/web apps/api

cat > apps/web/package.json << 'EOF'
{
  "name": "web",
  "version": "0.0.0",
  "private": true,
  "dependencies": {
    "@myorg/utils": "workspace:*"
  },
  "scripts": {
    "build": "next build",
    "test": "jest"
  }
}
EOF

# สร้าง turbo.json
cat > turbo.json << 'EOF'
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"]
    },
    "lint": {
      "outputs": []
    }
  }
}
EOF

# Install
pnpm install

echo "=== Monorepo setup สำเร็จ! ==="
```

### Exercise 2: Affected Detection

```yaml
# .github/workflows/affected.yaml
name: Monorepo CI

on:
  pull_request:

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      apps: ${{ steps.detect.outputs.apps }}
      packages: ${{ steps.detect.outputs.packages }}
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Detect changed packages
      id: detect
      run: |
        # หา changed files
        CHANGED=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
        
        # หา changed apps
        APPS=$(echo "$CHANGED" | grep '^apps/' | cut -d/ -f2 | sort -u | tr '\n' ' ')
        echo "apps=$APPS" >> $GITHUB_OUTPUT
        
        # หา changed packages
        PKGS=$(echo "$CHANGED" | grep '^packages/' | cut -d/ -f2 | sort -u | tr '\n' ' ')
        echo "packages=$PKGS" >> $GITHUB_OUTPUT
        
        echo "Changed apps: $APPS"
        echo "Changed packages: $PKGS"
  
  build-changed:
    needs: detect-changes
    runs-on: ubuntu-latest
    if: needs.detect-changes.outputs.apps != ''
    
    strategy:
      matrix:
        app: ${{ fromJSON(format('["{0}"]', join(fromJSON('[]'), '","', split(needs.detect-changes.outputs.apps, ' ')))) }}
    
    steps:
    - uses: actions/checkout@v4
    - name: Build ${{ matrix.app }}
      run: |
        cd apps/${{ matrix.app }}
        npm ci
        npm run build
```

### Exercise 3: Nx Affected Pipeline

```bash
#!/bin/bash
# exercise-3-nx-affected.sh

# ติดตั้ง Nx workspace
npx create-nx-workspace@latest nx-demo \
  --preset=ts \
  --nx-cloud=false

cd nx-demo

# เพิ่ม libraries
npx nx generate @nx/js:lib utils --unitTestRunner=jest
npx nx generate @nx/js:lib ui --unitTestRunner=jest

# เพิ่ม apps
npx nx generate @nx/node:app api --unitTestRunner=jest
npx nx generate @nx/node:app web --unitTestRunner=jest

# สร้าง dependency: api depends on utils
cat >> apps/api/src/main.ts << 'EOF'
import { myUtilFunction } from '@nx-demo/utils';
console.log(myUtilFunction());
EOF

# แก้ไข utils
echo "// new feature" >> libs/utils/src/lib/utils.ts

# ดู affected projects
npx nx affected:graph --base=HEAD~1

# Build affected
npx nx affected:build --base=HEAD~1 --head=HEAD

# Test affected
npx nx affected:test --base=HEAD~1 --head=HEAD

echo "=== Nx affected demo สำเร็จ! ==="
```

---

## สรุป

Monorepo CI/CD ต้องการกลยุทธ์ที่ดีเพื่อจัดการ:

1. **Path-based triggering** - build เฉพาะที่เปลี่ยนแปลง
2. **Affected detection** - Nx หรือ Turborepo ช่วย identify affected projects
3. **Caching** - remote cache ลด build time อย่างมาก
4. **Versioning** - Changesets หรือ Nx Release
5. **Parallel builds** - build หลาย packages พร้อมกัน

เลือก tool ตามความต้องการ:
- **Turborepo**: JavaScript/TypeScript, ง่าย, fast
- **Nx**: Full-featured, plugins, ดี enterprise
- **Bazel**: ขนาดใหญ่มาก, polyglot, ซับซ้อน

---

*ส่วนต่อไป: Part 46 - Microservices CI/CD*

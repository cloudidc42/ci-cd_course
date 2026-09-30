# Part 48: Cache Strategy ใน CI/CD

## สารบัญ
1. [ทำไมต้อง Cache?](#ทำไมต้อง-cache)
2. [GitHub Actions Cache](#github-actions-cache)
3. [Layer Caching (Docker)](#layer-caching-docker)
4. [Dependency Caching](#dependency-caching)
5. [Test Result Caching](#test-result-caching)
6. [Cache Invalidation Strategies](#cache-invalidation-strategies)
7. [Cache Storage Options](#cache-storage-options)
8. [Exercises](#exercises)

---

## ทำไมต้อง Cache?

Cache ช่วยลด CI/CD pipeline time ได้อย่างมาก

### ตัวอย่างผลกระทบของ Cache

```
Without Cache:
─────────────
Install dependencies: 3 minutes
Build:               4 minutes
Tests:               8 minutes
Docker build:        6 minutes
Total:               21 minutes

With Cache:
──────────
Restore cache:       30 seconds
Install dependencies: 10 seconds
Build:               2 minutes (incremental)
Tests:               3 minutes (affected only)
Docker build:        1 minute (layer cache hit)
Total:               6.5 minutes

ประหยัดเวลา: 70%!
```

### ประเภทของ Cache ใน CI/CD

```
1. Dependency Cache
   - npm/yarn/pnpm node_modules
   - Go module cache
   - Maven/Gradle cache
   - Python pip cache
   - Ruby gems

2. Build Cache
   - Compilation artifacts
   - Webpack/Vite build cache
   - Go build cache
   - Incremental build results

3. Docker Layer Cache
   - Base image layers
   - Dependency installation layers
   - Build output layers

4. Test Result Cache
   - Jest/pytest cache
   - Test artifacts
   - Coverage reports

5. Tool Cache
   - Terraform plugins
   - Downloaded tools/binaries
```

---

## GitHub Actions Cache

### actions/cache พื้นฐาน

```yaml
- name: Cache node modules
  uses: actions/cache@v4
  with:
    path: node_modules         # path ที่จะ cache
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-  # fallback key
```

### Cache Key Best Practices

```yaml
# BEST: Hash ของ lock file
key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}

# ถ้า cache miss: fallback ไปยัง ล่าสุดที่มี
restore-keys: |
  ${{ runner.os }}-npm-

# สำหรับ Matrix builds
key: ${{ runner.os }}-node${{ matrix.node-version }}-${{ hashFiles('package-lock.json') }}
```

### Cache Hit/Miss Handling

```yaml
- name: Cache node modules
  id: cache-node
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
    restore-keys: ${{ runner.os }}-node-

- name: Install dependencies (only on cache miss)
  if: steps.cache-node.outputs.cache-hit != 'true'
  run: npm ci

# หรือ ใช้ npm ci ทุกครั้ง แต่จะเร็วกว่าถ้า cache hit
- name: Install dependencies
  run: npm ci --prefer-offline  # ใช้ local cache ถ้ามี
```

### Cache Size Limits

```
GitHub Actions Cache Limits:
- Total: 10 GB per repository
- Individual file: ไม่จำกัด
- Eviction: LRU (Least Recently Used) เมื่อเกิน 10 GB
- Retention: 7 วันหลังจาก last access

ถ้า cache เต็ม:
- ลบ cache เก่าที่ไม่ได้ใช้
- ใส่ directory-specific paths
- ใช้ restore-keys เป็น fallback
```

### Manual Cache Invalidation

```bash
# ผ่าน GitHub CLI
gh api repos/myorg/myrepo/actions/caches

# ลบ cache ด้วย key
gh api -X DELETE repos/myorg/myrepo/actions/caches/key/my-cache-key

# ลบ cache ทั้งหมด
gh api repos/myorg/myrepo/actions/caches | \
  jq '.actions_caches[].id' | \
  xargs -I{} gh api -X DELETE repos/myorg/myrepo/actions/caches/{}
```

---

## Layer Caching (Docker)

### Docker Build Cache Fundamentals

```dockerfile
# ❌ ไม่ดี: Copy ทุกอย่างก่อน install
FROM node:20-alpine
WORKDIR /app
COPY . .                    # เมื่อ code เปลี่ยน, layer นี้ invalid
RUN npm ci                  # ต้อง re-run ทุกครั้ง (ช้า!)
RUN npm run build

# ✅ ดี: Copy package files ก่อน install
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./       # เปลี่ยนน้อยกว่า
RUN npm ci                  # cache hit ส่วนใหญ่
COPY . .                    # เมื่อ code เปลี่ยน เฉพาะ layer นี้ invalid
RUN npm run build
```

### Multi-Stage Build Cache

```dockerfile
# Multi-stage สำหรับ Go
# Stage 1: Dependencies (cached)
FROM golang:1.21-alpine AS deps
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download   # cached เมื่อ go.mod ไม่เปลี่ยน

# Stage 2: Builder
FROM deps AS builder
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# Stage 3: Final image (small)
FROM alpine:3.19 AS runner
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

### GitHub Actions Docker Cache

```yaml
# วิธีที่ 1: GitHub Actions Cache backend
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha              # ใช้ GitHub Actions cache
    cache-to: type=gha,mode=max       # cache ทุก layers

# วิธีที่ 2: Registry Cache (ดีกว่าสำหรับ large projects)
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/myorg/myapp:latest
    cache-from: type=registry,ref=ghcr.io/myorg/myapp:cache
    cache-to: type=registry,ref=ghcr.io/myorg/myapp:cache,mode=max

# วิธีที่ 3: Inline Cache (เก็บ cache ใน image)
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/myorg/myapp:latest
    cache-from: type=inline
    cache-to: type=inline
```

### Docker BuildKit Cache Mounts

```dockerfile
# ใช้ BuildKit cache mounts
# syntax=docker/dockerfile:1

FROM node:20-alpine
WORKDIR /app

COPY package*.json ./
# RUN --mount=type=cache,target=/root/.npm npm ci
RUN --mount=type=cache,target=/root/.npm \
    npm ci --cache /root/.npm

COPY . .
RUN --mount=type=cache,target=/tmp/build-cache \
    npm run build

# สำหรับ Go
FROM golang:1.21-alpine
WORKDIR /app
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o server .
```

```yaml
# Enable BuildKit ใน GitHub Actions
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
  with:
    buildkitd-flags: --debug

- name: Build with cache mounts
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
    # BuildKit cache mounts จะถูก cache โดยอัตโนมัติ
```

---

## Dependency Caching

### Node.js / npm

```yaml
# npm
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'   # built-in cache support

# หรือ manual
- uses: actions/cache@v4
  with:
    path: |
      ~/.npm
      node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-node-

- run: npm ci
```

### pnpm

```yaml
- uses: pnpm/action-setup@v4
  with:
    version: 9
    run_install: false

- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'pnpm'  # built-in pnpm cache

- run: pnpm install --frozen-lockfile
```

### Yarn

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'yarn'

- run: yarn install --frozen-lockfile
```

### Python / pip

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.11'
    cache: 'pip'  # built-in pip cache

- run: pip install -r requirements.txt

# หรือ Poetry
- uses: actions/cache@v4
  with:
    path: ~/.cache/pypoetry
    key: ${{ runner.os }}-poetry-${{ hashFiles('poetry.lock') }}
    restore-keys: ${{ runner.os }}-poetry-

- run: poetry install --no-root
```

### Go Modules

```yaml
- uses: actions/setup-go@v5
  with:
    go-version-file: go.mod
    cache-dependency-path: go.sum  # built-in cache

# หรือ manual
- uses: actions/cache@v4
  with:
    path: |
      ~/go/pkg/mod
      ~/.cache/go-build
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: ${{ runner.os }}-go-

- run: go mod download
```

### Java / Maven

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'  # built-in Maven cache

# หรือ Gradle
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'gradle'

- uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
```

### Ruby / Bundler

```yaml
- uses: ruby/setup-ruby@v1
  with:
    ruby-version: '3.2'
    bundler-cache: true  # built-in bundler cache
```

### Rust / Cargo

```yaml
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/registry
      ~/.cargo/git
      target/
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    restore-keys: ${{ runner.os }}-cargo-

- run: cargo build --release
```

---

## Test Result Caching

### Jest Cache

```yaml
- uses: actions/cache@v4
  with:
    path: .jest-cache
    key: jest-${{ runner.os }}-${{ hashFiles('**/*.test.ts', '**/*.spec.ts') }}
    restore-keys: jest-${{ runner.os }}-

- name: Run Jest tests
  run: |
    npx jest \
      --cache \
      --cacheDirectory .jest-cache \
      --ci \
      --coverage
```

### pytest Cache

```yaml
- uses: actions/cache@v4
  with:
    path: .pytest_cache
    key: pytest-${{ runner.os }}-${{ hashFiles('**/*.py') }}
    restore-keys: pytest-${{ runner.os }}-

- name: Run pytest
  run: |
    pytest \
      --cache-dir .pytest_cache \
      -x \
      --cov=src \
      --cov-report=xml
```

### Turborepo Test Cache

```yaml
- run: npx turbo run test
  env:
    TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
    TURBO_TEAM: ${{ vars.TURBO_TEAM }}
  # Turborepo จะ cache test results ใน remote cache
  # และ skip tasks ที่ inputs ไม่เปลี่ยน
```

### Nx Test Cache

```yaml
- run: npx nx affected:test --base=main
  env:
    NX_CLOUD_ACCESS_TOKEN: ${{ secrets.NX_CLOUD_ACCESS_TOKEN }}
  # Nx Cloud จะ cache ผลลัพธ์ test
  # และ replay ถ้า inputs เหมือนกัน
```

---

## Cache Invalidation Strategies

### Strategy 1: Content-Based (Hash)

```yaml
# Cache key ตาม content ของ files
key: ${{ runner.os }}-${{ hashFiles('package-lock.json') }}

# เมื่อ package-lock.json เปลี่ยน → new cache
# เมื่อ package-lock.json ไม่เปลี่ยน → cache hit
```

### Strategy 2: Time-Based

```yaml
# ทำให้ cache หมดอายุทุกวัน
key: ${{ runner.os }}-daily-${{ github.run_number / 100 }}-${{ hashFiles('...') }}

# หรือ ใช้ date
- name: Get date
  id: date
  run: echo "date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT

- uses: actions/cache@v4
  with:
    path: my-cache
    key: daily-${{ steps.date.outputs.date }}-${{ hashFiles('...') }}
```

### Strategy 3: Branch-Based Cache Isolation

```yaml
# แยก cache ต่อ branch ป้องกัน cache poisoning
key: ${{ runner.os }}-${{ github.ref_name }}-node-${{ hashFiles('package-lock.json') }}
restore-keys: |
  ${{ runner.os }}-${{ github.ref_name }}-node-
  ${{ runner.os }}-main-node-     # fallback to main branch cache
  ${{ runner.os }}-node-
```

### Strategy 4: Manual Version Bump

```yaml
# เพิ่ม version number เพื่อ force invalidate cache
key: v3-${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
# v3 → เปลี่ยนเมื่อต้องการ invalidate cache ทั้งหมด
```

### เมื่อไหรที่ควร Invalidate Cache

```
ควร invalidate เมื่อ:
1. Security vulnerability ใน cached dependencies
2. Major version update ที่อาจมี corruption
3. Cache ที่ประกอบด้วยข้อมูลที่เป็น private/sensitive
4. Build environment เปลี่ยน (OS version, tool versions)
5. Cache ทำให้ test ผ่านทั้งที่ควรจะ fail (false positive)
```

---

## Cache Storage Options

### 1. GitHub Actions Cache (Built-in)

```yaml
pros:
- ฟรีมาถึง 10GB ต่อ repository
- ง่ายใน configure
- Integrated กับ workflow

cons:
- จำกัดที่ 10GB
- ช้ากว่า self-hosted cache
- ข้ามระหว่าง repositories ไม่ได้
```

### 2. Self-Hosted Cache Server

```yaml
# Minio (S3-compatible) สำหรับ cache
# docker-compose.yaml
version: '3.8'
services:
  minio:
    image: minio/minio
    ports:
    - "9000:9000"
    - "9001:9001"
    environment:
      MINIO_ROOT_USER: minio-admin
      MINIO_ROOT_PASSWORD: minio-password
    command: server /data --console-address ":9001"
    volumes:
    - minio-data:/data

  cache-server:
    image: ghcr.io/jjst/github-actions-cache-server
    ports:
    - "8080:8080"
    environment:
      STORAGE_BACKEND: s3
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET: actions-cache
      AWS_ACCESS_KEY_ID: minio-admin
      AWS_SECRET_ACCESS_KEY: minio-password
```

```yaml
# GitHub Actions ที่ใช้ self-hosted cache
- uses: actions/cache@v4
  env:
    ACTIONS_CACHE_URL: http://my-cache-server:8080/
    ACTIONS_RUNTIME_TOKEN: ${{ secrets.CACHE_TOKEN }}
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
```

### 3. Turborepo Remote Cache

```bash
# Setup Turborepo remote cache
# Option 1: Vercel (ฟรีสำหรับ personal)
turbo login
turbo link

# Option 2: Self-hosted
docker run -p 8080:8080 \
  -e TURBO_TOKEN=my-secret-token \
  ghcr.io/nicholasgasior/turborepo-remote-cache

# ใช้ cache
TURBO_TOKEN=my-secret-token \
TURBO_API=http://my-cache:8080 \
TURBO_TEAM=my-team \
turbo build
```

### 4. Nx Cloud Cache

```json
// nx.json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx-cloud",
      "options": {
        "accessToken": "xxx..."
      }
    }
  }
}
```

### 5. Docker Registry as Cache

```yaml
# Push cache layers ไปยัง container registry
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: myapp:latest
    cache-from: type=registry,ref=ghcr.io/myorg/myapp:buildcache
    cache-to: type=registry,ref=ghcr.io/myorg/myapp:buildcache,mode=max
```

---

## Advanced Caching Patterns

### Pattern 1: Layered Cache

```yaml
- uses: actions/cache@v4
  with:
    path: node_modules
    key: exact-${{ hashFiles('package-lock.json') }}
    restore-keys: |
      partial-node-${{ hashFiles('packages/*/package.json') }}
      node-${{ runner.os }}-
```

### Pattern 2: Shared Cache ระหว่าง Jobs

```yaml
jobs:
  prepare:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/cache@v4
      with:
        path: node_modules
        key: node-${{ hashFiles('package-lock.json') }}
    - run: npm ci

  test:
    needs: prepare
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/cache@v4
      with:
        path: node_modules
        key: node-${{ hashFiles('package-lock.json') }}
    # node_modules จะถูก restore จาก cache
    - run: npm test

  build:
    needs: prepare
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/cache@v4
      with:
        path: node_modules
        key: node-${{ hashFiles('package-lock.json') }}
    - run: npm run build
```

### Pattern 3: Conditional Install

```yaml
- id: cache
  uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

- name: Install (skip if cache hit)
  if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci

- name: Verify (always)
  run: npm run verify:modules  # ตรวจสอบว่า cache ยัง valid
```

### Pattern 4: Parallel Downloads with Cache

```yaml
# ดาวน์โหลด dependencies แบบ parallel
- name: Cache downloads
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/pip
      ~/.cache/docker
      ~/.npm
      ~/go/pkg/mod
    key: all-deps-${{ runner.os }}-${{ hashFiles('requirements.txt', 'package-lock.json', 'go.sum') }}

- name: Install all dependencies (parallel)
  run: |
    pip install -r requirements.txt &
    npm ci &
    go mod download &
    wait  # รอทุก background jobs
    echo "All dependencies installed!"
```

---

## Cache Monitoring

### GitHub Actions Cache Usage

```bash
#!/bin/bash
# ดู cache usage ผ่าน API

REPO="myorg/myrepo"
TOKEN=$GITHUB_TOKEN

# ดู cache list
curl -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/$REPO/actions/caches" | \
  jq '.actions_caches[] | {id, key, size_in_bytes, last_accessed_at}'

# คำนวณ total usage
TOTAL=$(curl -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/$REPO/actions/caches" | \
  jq '[.actions_caches[].size_in_bytes] | add')

echo "Total cache size: $((TOTAL / 1024 / 1024)) MB"
```

### Cache Hit Rate Tracking

```yaml
- name: Track cache hit rate
  run: |
    if [ "${{ steps.cache.outputs.cache-hit }}" = "true" ]; then
      echo "CACHE_HIT=true" >> $GITHUB_ENV
      echo "✅ Cache hit - saving time!"
    else
      echo "CACHE_HIT=false" >> $GITHUB_ENV
      echo "❌ Cache miss - installing from scratch"
    fi

- name: Report cache metrics
  uses: actions/github-script@v7
  with:
    script: |
      const cacheHit = process.env.CACHE_HIT === 'true';
      core.notice(`Cache hit: ${cacheHit}`);
      
      // ส่งไปยัง metrics system
      // await fetch('https://metrics.mycompany.com/ci-cache', { ... })
```

---

## Exercises

### Exercise 1: เปรียบเทียบ Time กับ/ไม่มี Cache

```yaml
# .github/workflows/cache-benchmark.yaml
name: Cache Benchmark

on: workflow_dispatch

jobs:
  without-cache:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
    
    - name: Measure install time (no cache)
      run: |
        START=$(date +%s)
        npm ci
        END=$(date +%s)
        echo "Install time: $((END-START)) seconds"

  with-cache:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - name: Measure install time (with cache)
      run: |
        START=$(date +%s)
        npm ci
        END=$(date +%s)
        echo "Install time: $((END-START)) seconds"
```

### Exercise 2: Docker Layer Cache Optimization

```dockerfile
# Before: ไม่ optimize (exercise-2-before/Dockerfile)
FROM python:3.11-slim

WORKDIR /app

# ❌ Copy ทุกอย่างก่อน install
COPY . .
RUN pip install -r requirements.txt
RUN python -m pytest

CMD ["python", "app.py"]
```

```dockerfile
# After: optimize layers (exercise-2-after/Dockerfile)
FROM python:3.11-slim

WORKDIR /app

# ✅ Copy only requirements first
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ✅ Copy source after dependencies
COPY . .

CMD ["python", "app.py"]
```

```yaml
# pipeline สำหรับ benchmark
- name: Build Before (no layer optimization)
  uses: docker/build-push-action@v5
  with:
    context: exercise-2-before
    cache-from: type=gha,scope=before
    cache-to: type=gha,scope=before,mode=max

- name: Build After (with layer optimization)
  uses: docker/build-push-action@v5
  with:
    context: exercise-2-after
    cache-from: type=gha,scope=after
    cache-to: type=gha,scope=after,mode=max
```

### Exercise 3: Multi-Language Cache

```yaml
# .github/workflows/multi-lang-cache.yaml
name: Multi-Language with Cache

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    # Node.js Cache
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    # Python Cache
    - uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'
    
    # Go Cache
    - uses: actions/setup-go@v5
      with:
        go-version-file: go.mod
    
    # Custom tool cache
    - uses: actions/cache@v4
      with:
        path: ~/custom-tools
        key: tools-v1-${{ runner.os }}
    
    - name: Install tools (if not cached)
      run: |
        if [ ! -f ~/custom-tools/my-tool ]; then
          mkdir -p ~/custom-tools
          wget -O ~/custom-tools/my-tool https://example.com/my-tool
          chmod +x ~/custom-tools/my-tool
        fi
    
    # Install all dependencies
    - run: npm ci
    - run: pip install -r requirements.txt
    - run: go mod download
    
    # Build everything
    - run: npm run build
    - run: python -m pytest
    - run: go test ./...
```

### Exercise 4: Cache Version Strategy

```yaml
# จัดการ cache versions
env:
  CACHE_VERSION: v2   # เพิ่มเมื่อต้องการ invalidate ทั้งหมด

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/cache@v4
      with:
        path: node_modules
        # Version ใน key บังคับ invalidate เมื่อ CACHE_VERSION เปลี่ยน
        key: ${{ env.CACHE_VERSION }}-${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
        restore-keys: |
          ${{ env.CACHE_VERSION }}-${{ runner.os }}-node-
    
    - run: npm ci
```

---

## สรุป

Cache Strategy ที่ดีต้องคำนึงถึง:

1. **Cache Keys ที่ถูกต้อง** - hash ของ lock files, ไม่ใช่ timestamp
2. **Layer ordering ใน Dockerfile** - dependencies ก่อน source code
3. **Restore Keys** - fallback chains เพื่อ partial cache hit
4. **Cache Scope** - branch-specific vs global
5. **Invalidation** - รู้ว่าเมื่อไหรควร force invalidate
6. **Size Management** - monitor และ prune cache เก่า

ผลลัพธ์ที่ดี:
- Pipeline เร็วขึ้น 50-80%
- ลด network bandwidth
- ลด cost (fewer CI minutes)

---

*ส่วนต่อไป: Part 49 - Parallel & Matrix Builds*

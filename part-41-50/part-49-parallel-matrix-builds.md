# Part 49: Parallel & Matrix Builds

## สารบัญ
1. [ทำไมต้อง Parallel Builds?](#ทำไมต้อง-parallel-builds)
2. [Matrix Strategy ใน GitHub Actions](#matrix-strategy-ใน-github-actions)
3. [Parallel Jobs](#parallel-jobs)
4. [Fan-out/Fan-in Patterns](#fan-outfan-in-patterns)
5. [Test Sharding](#test-sharding)
6. [Dynamic Matrix](#dynamic-matrix)
7. [Exercises](#exercises)

---

## ทำไมต้อง Parallel Builds?

### Sequential vs Parallel Execution

```
Sequential (ช้า):
────────────────
Test Chrome   → 5 min
Test Firefox  → 5 min
Test Safari   → 5 min
Test Node 18  → 3 min
Test Node 20  → 3 min
Total:        → 21 min

Parallel (เร็ว):
────────────────
Test Chrome  ─┐
Test Firefox  ├─ รันพร้อมกัน → 5 min
Test Safari  ─┘
Test Node 18 ─┐
Test Node 20 ─┘─ รันพร้อมกัน → 3 min
Total:        → max(5, 3) = 5 min + overhead

ประหยัด: 16 นาที!
```

### เมื่อไหรที่ควรใช้ Parallel Builds

```
ใช้ parallel เมื่อ:
✅ Tests ที่ใช้เวลานาน (unit tests, e2e tests)
✅ Cross-platform testing (Linux, macOS, Windows)
✅ Multi-version testing (Node 18, 20, 22)
✅ Multi-browser testing
✅ Build หลาย Docker images พร้อมกัน
✅ Security scans (dependency, container, code)

ไม่ควรใช้ parallel เมื่อ:
❌ Jobs มี dependency กัน (ต้อง sequential)
❌ Jobs ใช้ shared resources ที่มี locks
❌ จำนวน jobs น้อยเกินไป (overhead > benefit)
```

---

## Matrix Strategy ใน GitHub Actions

### Basic Matrix

```yaml
jobs:
  test:
    strategy:
      matrix:
        node: [18, 20, 22]
    
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
    
    - run: npm ci
    - run: npm test
    
    # Job name จะเป็น: test (18), test (20), test (22)
```

### Multi-Dimension Matrix

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        # จะสร้าง 3 x 3 = 9 jobs
    
    runs-on: ${{ matrix.os }}
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
    
    - name: Install and test
      run: |
        npm ci
        npm test
```

### Matrix with Include/Exclude

```yaml
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
        
        include:
        # เพิ่ม combination พิเศษ
        - os: ubuntu-latest
          node: 20
          experimental: true
          extra-test: true
        
        exclude:
        # ลบ combination ที่ไม่ต้องการ
        - os: macos-latest
          node: 18   # ไม่ test Node 18 บน macOS
        - os: windows-latest
          node: 18   # ไม่ test Node 18 บน Windows
    
    runs-on: ${{ matrix.os }}
    continue-on-error: ${{ matrix.experimental == true }}
    
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
    
    - run: npm ci
    - run: npm test
    
    - name: Extra tests (only on special combination)
      if: matrix.extra-test == true
      run: npm run test:extra
```

### Matrix with Object Variables

```yaml
jobs:
  deploy:
    strategy:
      matrix:
        environment:
        - name: staging
          url: https://staging.mycompany.com
          replicas: 2
          region: ap-southeast-1
        - name: production-us
          url: https://us.mycompany.com
          replicas: 10
          region: us-east-1
        - name: production-eu
          url: https://eu.mycompany.com
          replicas: 10
          region: eu-west-1
    
    runs-on: ubuntu-latest
    environment:
      name: ${{ matrix.environment.name }}
      url: ${{ matrix.environment.url }}
    
    steps:
    - name: Deploy to ${{ matrix.environment.name }}
      run: |
        helm upgrade --install myapp ./helm \
          --set replicaCount=${{ matrix.environment.replicas }} \
          --set region=${{ matrix.environment.region }}
```

### Fail-Fast Configuration

```yaml
strategy:
  fail-fast: false  # ไม่หยุด jobs อื่นเมื่อ 1 job fail
  matrix:
    node: [18, 20, 22]
    os: [ubuntu-latest, macos-latest]

# ค่า default: fail-fast: true
# เมื่อ 1 job fail → cancel jobs ที่เหลือทั้งหมด
```

### Max Parallel

```yaml
strategy:
  max-parallel: 3   # รันพร้อมกันสูงสุด 3 jobs
  matrix:
    service: [auth, user, order, payment, notification, inventory, search]
  # ถ้าไม่กำหนด: GitHub รันทุก jobs พร้อมกันเลย
```

---

## Parallel Jobs

### Basic Parallel Jobs

```yaml
jobs:
  # รันพร้อมกัน (ไม่มี needs)
  unit-tests:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm run test:unit
  
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm run lint
  
  type-check:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm run type-check
  
  # รอทุก jobs ข้างบน
  build:
    needs: [unit-tests, lint, type-check]
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: npm ci && npm run build
```

### Complex Parallel Pipeline

```yaml
name: Full CI Pipeline

jobs:
  # Phase 1: Fast checks (parallel)
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci && npm run lint

  type-check:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci && npm run type-check

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: npx audit-ci --moderate

  # Phase 2: Tests (parallel, depends on phase 1)
  unit-tests:
    needs: [lint, type-check]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20, 22]
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
        cache: 'npm'
    - run: npm ci && npm run test:unit

  integration-tests:
    needs: [lint, type-check]
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports: ['5432:5432']
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci && npm run test:integration

  # Phase 3: Build (depends on all tests)
  build:
    needs: [unit-tests, integration-tests, security-scan]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - id: meta
      uses: docker/metadata-action@v5
      with:
        images: ghcr.io/myorg/myapp
        tags: type=sha
    - uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  # Phase 4: E2E Tests (parallel browsers)
  e2e-tests:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        browser: [chrome, firefox, webkit]
        shard: [1, 2, 3]
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci && npx playwright install --with-deps ${{ matrix.browser }}
    - run: npx playwright test --browser=${{ matrix.browser }} --shard=${{ matrix.shard }}/3
      env:
        TEST_URL: https://preview.mycompany.com

  # Phase 5: Deploy (depends on e2e)
  deploy:
    needs: e2e-tests
    runs-on: ubuntu-latest
    environment: production
    steps:
    - run: echo "Deploying version ${{ needs.build.outputs.image-tag }}"
```

---

## Fan-out/Fan-in Patterns

### Fan-out Pattern

```
Fan-out: 1 job triggers multiple parallel jobs

         ┌─── Job A ───┐
trigger ─┤─── Job B ───┼─── (independent)
         └─── Job C ───┘
```

```yaml
# Fan-out: trigger multiple independent jobs
jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
    - uses: actions/checkout@v4
    - id: version
      run: echo "version=$(cat VERSION)" >> $GITHUB_OUTPUT

  # Fan-out: build multiple platforms
  build-linux-amd64:
    needs: prepare
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: GOARCH=amd64 GOOS=linux go build -o app-linux-amd64

  build-linux-arm64:
    needs: prepare
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: GOARCH=arm64 GOOS=linux go build -o app-linux-arm64

  build-darwin-amd64:
    needs: prepare
    runs-on: macos-latest
    steps:
    - uses: actions/checkout@v4
    - run: GOARCH=amd64 GOOS=darwin go build -o app-darwin-amd64

  build-windows-amd64:
    needs: prepare
    runs-on: windows-latest
    steps:
    - uses: actions/checkout@v4
    - run: go build -o app-windows-amd64.exe
```

### Fan-in Pattern

```
Fan-in: Multiple jobs complete before one final job

Job A ─┐
Job B ─┼─── Final Job
Job C ─┘
```

```yaml
# Fan-in: รวม artifacts จากหลาย jobs
  build-linux-amd64:
    # ... (จาก fan-out ข้างบน)
    steps:
    - run: go build -o dist/linux-amd64/app
    - uses: actions/upload-artifact@v4
      with:
        name: binary-linux-amd64
        path: dist/linux-amd64/

  build-linux-arm64:
    steps:
    - run: go build -o dist/linux-arm64/app
    - uses: actions/upload-artifact@v4
      with:
        name: binary-linux-arm64
        path: dist/linux-arm64/

  # Fan-in: รอทุก builds แล้วรวม artifacts
  release:
    needs: [build-linux-amd64, build-linux-arm64, build-darwin-amd64, build-windows-amd64]
    runs-on: ubuntu-latest
    steps:
    - name: Download all artifacts
      uses: actions/download-artifact@v4
      with:
        path: dist/
        pattern: binary-*

    - name: List artifacts
      run: ls -la dist/

    - name: Create GitHub Release
      uses: ncipollo/release-action@v1
      with:
        artifacts: "dist/**/*"
        tag: ${{ github.ref_name }}
        generateReleaseNotes: true
```

### Complete Fan-out/Fan-in Pipeline

```yaml
name: Release Pipeline

on:
  push:
    tags: ['v*']

jobs:
  # Fan-out: Build binaries สำหรับทุก platforms
  build:
    strategy:
      matrix:
        include:
        - os: ubuntu-latest
          goos: linux
          goarch: amd64
          artifact: app-linux-amd64
        - os: ubuntu-latest
          goos: linux
          goarch: arm64
          artifact: app-linux-arm64
        - os: macos-latest
          goos: darwin
          goarch: amd64
          artifact: app-darwin-amd64
        - os: macos-latest
          goos: darwin
          goarch: arm64
          artifact: app-darwin-arm64
        - os: windows-latest
          goos: windows
          goarch: amd64
          artifact: app-windows-amd64.exe
    
    runs-on: ${{ matrix.os }}
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-go@v5
      with:
        go-version-file: go.mod
    
    - name: Build
      env:
        GOOS: ${{ matrix.goos }}
        GOARCH: ${{ matrix.goarch }}
      run: |
        go build -ldflags "-X main.version=${{ github.ref_name }}" \
          -o ${{ matrix.artifact }} .
    
    - name: Upload artifact
      uses: actions/upload-artifact@v4
      with:
        name: ${{ matrix.artifact }}
        path: ${{ matrix.artifact }}
        retention-days: 1

  # Fan-out: Test แบบ parallel
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
        go: ['1.21', '1.22']
    
    runs-on: ${{ matrix.os }}
    
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with:
        go-version: ${{ matrix.go }}
    - run: go test ./...

  # Fan-in: รวมทุกอย่างและสร้าง release
  release:
    needs: [build, test]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Download all artifacts
      uses: actions/download-artifact@v4
      with:
        path: dist/
    
    - name: Create checksums
      run: |
        cd dist
        find . -type f -exec sha256sum {} \; > checksums.txt
        cat checksums.txt
    
    - name: Create Release
      uses: ncipollo/release-action@v1
      with:
        artifacts: "dist/**"
        tag: ${{ github.ref_name }}
        generateReleaseNotes: true
        body: |
          ## Release ${{ github.ref_name }}
          
          ### Downloads
          See assets below for pre-built binaries.
          
          ### Checksums
          `sha256sum -c checksums.txt`
```

---

## Test Sharding

Test Sharding แบ่ง test suite ออกเป็น N chunks รันใน parallel

### Playwright Sharding

```yaml
jobs:
  e2e-tests:
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4, 5]  # แบ่งเป็น 5 chunks
    
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    
    - run: npm ci
    - run: npx playwright install --with-deps
    
    - name: Run tests (shard ${{ matrix.shard }}/5)
      run: |
        npx playwright test \
          --shard=${{ matrix.shard }}/5 \
          --reporter=blob
    
    - name: Upload blob report
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: blob-report-${{ matrix.shard }}
        path: blob-report
        retention-days: 1

  # Merge reports จากทุก shards
  merge-reports:
    needs: e2e-tests
    if: always()
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci
    
    - name: Download all blob reports
      uses: actions/download-artifact@v4
      with:
        path: all-blob-reports
        pattern: blob-report-*
        merge-multiple: true
    
    - name: Merge into HTML report
      run: npx playwright merge-reports --reporter html ./all-blob-reports
    
    - name: Upload HTML report
      uses: actions/upload-artifact@v4
      with:
        name: html-report
        path: playwright-report
```

### Jest Sharding

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci
    
    - name: Run Jest shard ${{ matrix.shard }}/4
      run: |
        npx jest \
          --shard=${{ matrix.shard }}/4 \
          --coverage \
          --coverageDirectory=coverage-${{ matrix.shard }}
    
    - name: Upload coverage
      uses: actions/upload-artifact@v4
      with:
        name: coverage-${{ matrix.shard }}
        path: coverage-${{ matrix.shard }}

  coverage-report:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/download-artifact@v4
      with:
        path: coverage
        pattern: coverage-*
    
    - run: npx istanbul merge coverage/*/coverage-final.json --out merged-coverage.json
    
    - uses: codecov/codecov-action@v3
      with:
        files: merged-coverage.json
```

### pytest Sharding

```yaml
jobs:
  test:
    strategy:
      matrix:
        shard: [0, 1, 2, 3]  # pytest shards start from 0
    
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v5
      with:
        python-version: '3.11'
        cache: 'pip'
    
    - run: pip install -r requirements.txt pytest-split
    
    - name: Run pytest shard ${{ matrix.shard }}
      run: |
        pytest \
          --splits=4 \
          --group=${{ matrix.shard }} \
          --store-durations \
          --durations-path=.test_durations \
          -v
    
    - uses: actions/upload-artifact@v4
      with:
        name: test-durations-${{ matrix.shard }}
        path: .test_durations
```

### Cypress Sharding

```yaml
jobs:
  cypress:
    strategy:
      matrix:
        containers: [1, 2, 3, 4, 5, 6]
    
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - uses: cypress-io/github-action@v6
      with:
        record: true
        parallel: true
        group: 'Parallel Group'
        ci-build-id: ${{ github.sha }}-${{ github.workflow }}-${{ github.event_name }}
      env:
        CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Dynamic Matrix

### สร้าง Matrix จาก Script

```yaml
jobs:
  # สร้าง matrix dynamically
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    
    steps:
    - uses: actions/checkout@v4
    
    - id: set-matrix
      run: |
        # สร้าง matrix จาก directories
        SERVICES=$(ls services/ | jq -R -s -c 'split("\n") | map(select(length > 0))')
        echo "matrix={\"service\":$SERVICES}" >> $GITHUB_OUTPUT
        echo "Services: $SERVICES"
  
  # ใช้ dynamic matrix
  build-services:
    needs: generate-matrix
    strategy:
      matrix: ${{ fromJSON(needs.generate-matrix.outputs.matrix) }}
    
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build ${{ matrix.service }}
      run: |
        cd services/${{ matrix.service }}
        docker build -t myapp-${{ matrix.service }}:latest .
```

### Matrix จาก Changed Files

```yaml
jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.detect.outputs.services }}
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 2
    
    - id: detect
      run: |
        # หา changed services
        CHANGED=$(git diff --name-only HEAD~1 HEAD | grep '^services/')
        SERVICES=$(echo "$CHANGED" | cut -d/ -f2 | sort -u | \
          jq -R -s -c 'split("\n") | map(select(length > 0))')
        
        if [ "$SERVICES" = "[]" ]; then
          echo "No service changes detected"
          SERVICES='["placeholder"]'  # dummy เพื่อไม่ให้ matrix ว่าง
        fi
        
        echo "services=$SERVICES" >> $GITHUB_OUTPUT
  
  build-changed:
    needs: detect-changes
    if: needs.detect-changes.outputs.services != '["placeholder"]'
    strategy:
      matrix:
        service: ${{ fromJSON(needs.detect-changes.outputs.services) }}
    
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build ${{ matrix.service }}
      run: |
        cd services/${{ matrix.service }}
        echo "Building ${{ matrix.service }}"
```

### Matrix สำหรับ Database Testing

```yaml
jobs:
  test-databases:
    strategy:
      matrix:
        database:
        - name: postgres-14
          image: postgres:14
          port: 5432
          driver: postgresql
        - name: postgres-15
          image: postgres:15
          port: 5432
          driver: postgresql
        - name: postgres-16
          image: postgres:16
          port: 5432
          driver: postgresql
        - name: mysql-8
          image: mysql:8
          port: 3306
          driver: mysql
        - name: mariadb-10
          image: mariadb:10
          port: 3306
          driver: mariadb
    
    runs-on: ubuntu-latest
    
    services:
      database:
        image: ${{ matrix.database.image }}
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
          MYSQL_ROOT_PASSWORD: testpass
          MYSQL_DATABASE: testdb
        ports:
        - ${{ matrix.database.port }}:${{ matrix.database.port }}
        options: >-
          --health-cmd="pg_isready || mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run tests against ${{ matrix.database.name }}
      run: |
        go test ./... -v \
          -db-driver=${{ matrix.database.driver }} \
          -db-host=localhost \
          -db-port=${{ matrix.database.port }}
```

---

## Advanced Parallel Patterns

### Reusable Workflow สำหรับ Parallel Testing

```yaml
# .github/workflows/reusable-test.yaml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      test-command:
        type: string
        required: true
      node-version:
        type: string
        default: '20'
      coverage:
        type: boolean
        default: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
    - run: npm ci
    - run: ${{ inputs.test-command }}
    
    - name: Upload coverage
      if: inputs.coverage
      uses: codecov/codecov-action@v3
```

```yaml
# .github/workflows/ci.yaml
name: CI

on: push

jobs:
  unit-tests-node18:
    uses: ./.github/workflows/reusable-test.yaml
    with:
      test-command: npm run test:unit
      node-version: '18'

  unit-tests-node20:
    uses: ./.github/workflows/reusable-test.yaml
    with:
      test-command: npm run test:unit
      node-version: '20'
      coverage: true

  integration-tests:
    uses: ./.github/workflows/reusable-test.yaml
    with:
      test-command: npm run test:integration
```

### Timeout Management

```yaml
jobs:
  parallel-tests:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    
    runs-on: ubuntu-latest
    timeout-minutes: 15   # ป้องกัน hung jobs
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run tests with timeout
      timeout-minutes: 10   # step-level timeout
      run: |
        npx jest \
          --shard=${{ matrix.shard }}/4 \
          --testTimeout=5000 \
          --forceExit
```

### Concurrent Job Limits

```yaml
# ใช้ concurrency เพื่อ limit concurrent deployments
jobs:
  deploy:
    concurrency:
      group: deploy-${{ github.ref }}
      cancel-in-progress: true   # cancel เมื่อ new push มา
    
    runs-on: ubuntu-latest
    steps:
    - name: Deploy
      run: echo "Deploying..."

# Pattern: serialize deployments ต่อ environment
  deploy-production:
    concurrency:
      group: production-deploy
      cancel-in-progress: false  # รอแทนที่จะ cancel
```

---

## Exercises

### Exercise 1: Matrix Testing

```yaml
# .github/workflows/exercise-1-matrix.yaml
name: Exercise 1 - Matrix Tests

on: push

jobs:
  test-matrix:
    strategy:
      fail-fast: false
      matrix:
        node: [18, 20, 22]
        os: [ubuntu-latest, macos-latest]
        include:
        - node: 20
          os: ubuntu-latest
          upload-coverage: true
        exclude:
        - node: 18
          os: macos-latest
    
    runs-on: ${{ matrix.os }}
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
        cache: 'npm'
    
    - run: npm ci
    
    - name: Run tests
      run: |
        npm test -- --ci
        echo "Node.js ${{ matrix.node }} on ${{ matrix.os }}: PASSED"
    
    - name: Upload coverage
      if: matrix.upload-coverage == true
      uses: codecov/codecov-action@v3
```

### Exercise 2: Fan-out/Fan-in Build

```yaml
# .github/workflows/exercise-2-fan-out.yaml
name: Exercise 2 - Fan-out/Fan-in

on:
  push:
    branches: [main]

jobs:
  # Fan-out: Build multiple Docker images
  build-images:
    strategy:
      matrix:
        service: [frontend, backend, worker]
        arch: [linux/amd64, linux/arm64]
    
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: docker/setup-buildx-action@v3
    
    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build ${{ matrix.service }} for ${{ matrix.arch }}
      uses: docker/build-push-action@v5
      with:
        context: services/${{ matrix.service }}
        platforms: ${{ matrix.arch }}
        push: true
        tags: ghcr.io/myorg/${{ matrix.service }}:${{ github.sha }}-${{ matrix.arch == 'linux/amd64' && 'amd64' || 'arm64' }}
        cache-from: type=gha,scope=${{ matrix.service }}-${{ matrix.arch }}
        cache-to: type=gha,scope=${{ matrix.service }}-${{ matrix.arch }},mode=max
    
    - name: Save digest
      run: |
        echo "IMAGE=ghcr.io/myorg/${{ matrix.service }}:${{ github.sha }}" >> $GITHUB_ENV
  
  # Fan-in: สร้าง multi-arch manifest
  create-manifests:
    needs: build-images
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [frontend, backend, worker]
    
    steps:
    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Create multi-arch manifest for ${{ matrix.service }}
      run: |
        docker buildx imagetools create \
          --tag ghcr.io/myorg/${{ matrix.service }}:${{ github.sha }} \
          ghcr.io/myorg/${{ matrix.service }}:${{ github.sha }}-amd64 \
          ghcr.io/myorg/${{ matrix.service }}:${{ github.sha }}-arm64
```

### Exercise 3: Dynamic Matrix

```yaml
# .github/workflows/exercise-3-dynamic.yaml
name: Exercise 3 - Dynamic Matrix

on: push

jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.matrix.outputs.value }}
    
    steps:
    - uses: actions/checkout@v4
    
    - id: matrix
      run: |
        # อ่าน services จาก directory หรือ config file
        SERVICES=$(ls -d services/*/ | \
          xargs -I{} basename {} | \
          jq -R -s -c 'split("\n") | map(select(length > 0))')
        
        echo "value={\"service\":$SERVICES}" >> $GITHUB_OUTPUT
        echo "Generated matrix: $SERVICES"
  
  build-dynamic:
    needs: generate-matrix
    strategy:
      matrix: ${{ fromJSON(needs.generate-matrix.outputs.matrix) }}
    
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Build ${{ matrix.service }}
      run: |
        echo "Building service: ${{ matrix.service }}"
        ls services/${{ matrix.service }}
```

### Exercise 4: Test Sharding

```yaml
# .github/workflows/exercise-4-sharding.yaml
name: Exercise 4 - Test Sharding

on: push

jobs:
  test-shards:
    strategy:
      matrix:
        shard: [1, 2, 3, 4, 5]
    
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - run: npm ci
    
    - name: Run tests (shard ${{ matrix.shard }}/5)
      run: |
        npx jest \
          --shard=${{ matrix.shard }}/5 \
          --coverage \
          --coverageDirectory=coverage \
          --ci
    
    - uses: actions/upload-artifact@v4
      with:
        name: coverage-shard-${{ matrix.shard }}
        path: coverage
  
  merge-coverage:
    needs: test-shards
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'
    
    - run: npm ci
    
    - uses: actions/download-artifact@v4
      with:
        path: all-coverage
        pattern: coverage-shard-*
        merge-multiple: true
    
    - name: Merge coverage reports
      run: |
        npx istanbul-combine \
          -d all-coverage/combined \
          all-coverage/*/coverage-final.json
    
    - uses: codecov/codecov-action@v3
      with:
        directory: all-coverage/combined
```

---

## สรุป

Parallel และ Matrix builds ช่วยลด CI/CD time ได้อย่างมาก:

1. **Matrix Strategy** - test หลาย combinations พร้อมกัน (OS, version, etc.)
2. **Parallel Jobs** - แยก independent tasks ให้รันพร้อมกัน
3. **Fan-out/Fan-in** - distribute work และ collect results
4. **Test Sharding** - แบ่ง test suite ใน parallel
5. **Dynamic Matrix** - generate matrix จาก runtime data

Key considerations:
- **Cost**: Parallel jobs ใช้ minutes มากขึ้น (หลาย runners)
- **Dependencies**: บาง jobs ต้อง sequential (ใช้ `needs`)
- **Artifacts**: ใช้ upload/download-artifact สำหรับ data sharing
- **Timeout**: ตั้ง timeout ป้องกัน hung jobs

---

*ส่วนต่อไป: Part 50 - Pipeline Optimization*

# Part 50: Pipeline Optimization

## สารบัญ
1. [ทำไมต้อง Optimize Pipeline?](#ทำไมต้อง-optimize-pipeline)
2. [Measuring Pipeline Speed](#measuring-pipeline-speed)
3. [Bottleneck Identification](#bottleneck-identification)
4. [Optimization Strategies](#optimization-strategies)
5. [Caching & Parallelism](#caching--parallelism)
6. [Incremental Builds](#incremental-builds)
7. [Pipeline Analytics](#pipeline-analytics)
8. [DORA Metrics Improvement](#dora-metrics-improvement)
9. [Exercises](#exercises)

---

## ทำไมต้อง Optimize Pipeline?

### ผลกระทบของ Slow Pipeline

```
Pipeline ช้า → ผลกระทบหลายด้าน:

Developer Experience:
- รอ feedback นาน → ขาด focus
- Context switching สูง
- Developer frustration

Business Impact:
- Deploy ช้า → Features ถึง user ช้า
- Hotfix ช้า → ระยะเวลา outage นานขึ้น
- Cost สูง (CI/CD costs)

Industry Benchmarks (DORA):
- Elite teams: < 10 minutes
- High teams:  < 30 minutes
- Medium teams: < 1 hour
- Low teams:   > 1 day
```

### ต้นทุนของ Pipeline

```
สมมติ:
- Team size: 20 developers
- 5 PRs per developer per week
- Pipeline ใช้เวลา 30 minutes
- CI cost: $0.008 per minute

Weekly cost:
= 20 developers × 5 PRs × 30 min × $0.008
= $24 per week
= ~$1,248 per year

ถ้า optimize เหลือ 10 minutes:
= 20 × 5 × 10 × $0.008
= $8 per week
= ~$416 per year

ประหยัด: $832 per year (66%)

Developer time wasted:
= 20 × 5 × 30 min waiting = 3,000 min/week
Optimize to 10 min:
= 20 × 5 × 10 = 1,000 min/week
Saved: 2,000 min = ~33 hours per week!
```

---

## Measuring Pipeline Speed

### GitHub Actions Analytics

```bash
#!/bin/bash
# scripts/measure-pipeline.sh

REPO="myorg/myrepo"
TOKEN=$GITHUB_TOKEN
WORKFLOW="CI"
DAYS=30

# ดู workflow run durations
gh api "repos/$REPO/actions/workflows" | \
  jq '.workflows[] | select(.name=="'$WORKFLOW'") | .id' | \
  xargs -I{} gh api "repos/$REPO/actions/workflows/{}/runs?per_page=100&status=completed" | \
  jq --arg days $DAYS '
    .workflow_runs |
    map(select(
      .created_at > (now - ($days | tonumber * 86400) | todate)
    )) |
    map({
      name: .name,
      duration: ((.updated_at | fromdate) - (.created_at | fromdate)),
      conclusion: .conclusion,
      created_at: .created_at
    }) |
    group_by(.conclusion) |
    map({
      conclusion: .[0].conclusion,
      count: length,
      avg_duration: (map(.duration) | add / length),
      p50: (sort_by(.duration) | .[length/2 | floor].duration),
      p95: (sort_by(.duration) | .[length * 0.95 | floor].duration)
    })
  '
```

### Custom Timing ใน Workflow

```yaml
# .github/workflows/timed-ci.yaml
jobs:
  measured-build:
    runs-on: ubuntu-latest
    steps:
    - name: Record start time
      id: start
      run: echo "start_time=$(date +%s)" >> $GITHUB_OUTPUT
    
    - uses: actions/checkout@v4
    
    - name: Measure checkout time
      run: |
        echo "Checkout completed at $(date)"
    
    - name: Time - Install dependencies
      id: install-start
      run: echo "start=$(date +%s)" >> $GITHUB_OUTPUT
    
    - name: Install dependencies
      run: npm ci
    
    - name: Measure install time
      run: |
        ELAPSED=$(($(date +%s) - ${{ steps.install-start.outputs.start }}))
        echo "Install took: ${ELAPSED}s"
        echo "INSTALL_TIME=${ELAPSED}" >> $GITHUB_ENV
    
    - name: Time - Run tests
      id: test-start
      run: echo "start=$(date +%s)" >> $GITHUB_OUTPUT
    
    - name: Run tests
      run: npm test
    
    - name: Measure test time
      run: |
        ELAPSED=$(($(date +%s) - ${{ steps.test-start.outputs.start }}))
        echo "Tests took: ${ELAPSED}s"
        echo "TEST_TIME=${ELAPSED}" >> $GITHUB_ENV
    
    - name: Total pipeline time
      run: |
        TOTAL=$(($(date +%s) - ${{ steps.start.outputs.start_time }}))
        echo "## Pipeline Metrics" >> $GITHUB_STEP_SUMMARY
        echo "| Step | Duration |" >> $GITHUB_STEP_SUMMARY
        echo "|------|----------|" >> $GITHUB_STEP_SUMMARY
        echo "| Install | ${INSTALL_TIME}s |" >> $GITHUB_STEP_SUMMARY
        echo "| Tests | ${TEST_TIME}s |" >> $GITHUB_STEP_SUMMARY
        echo "| **Total** | **${TOTAL}s** |" >> $GITHUB_STEP_SUMMARY
```

### Pipeline Dashboard

```yaml
# เก็บ metrics ไว้ใน repository
- name: Save metrics
  run: |
    mkdir -p .metrics
    cat >> .metrics/pipeline-times.json << EOF
    {
      "date": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "commit": "${{ github.sha }}",
      "install_time": ${{ env.INSTALL_TIME }},
      "test_time": ${{ env.TEST_TIME }},
      "total_time": ${{ env.TOTAL_TIME }},
      "branch": "${{ github.ref_name }}"
    }
    EOF

- name: Update metrics file
  run: |
    # Keep only last 100 runs
    python3 -c "
    import json
    with open('.metrics/pipeline-times.json') as f:
        data = [json.loads(line) for line in f if line.strip()]
    data = data[-100:]  # keep last 100
    with open('.metrics/pipeline-times.json', 'w') as f:
        for item in data:
            f.write(json.dumps(item) + '\n')
    "
```

---

## Bottleneck Identification

### Step-Level Timing Analysis

```python
#!/usr/bin/env python3
# scripts/analyze-pipeline.py
"""
Analyzes GitHub Actions workflow runs to find bottlenecks
"""

import json
import subprocess
from datetime import datetime, timedelta
from collections import defaultdict
import statistics

def get_workflow_runs(repo: str, workflow: str, days: int = 30):
    """Fetch workflow runs via GitHub CLI"""
    result = subprocess.run(
        ["gh", "api", f"repos/{repo}/actions/workflows"],
        capture_output=True, text=True
    )
    workflows = json.loads(result.stdout)
    
    workflow_id = None
    for wf in workflows.get("workflows", []):
        if wf["name"] == workflow:
            workflow_id = wf["id"]
            break
    
    if not workflow_id:
        raise ValueError(f"Workflow '{workflow}' not found")
    
    result = subprocess.run(
        ["gh", "api", f"repos/{repo}/actions/workflows/{workflow_id}/runs",
         "--jq", f".workflow_runs | map(select(.status == \"completed\"))"],
        capture_output=True, text=True
    )
    
    return json.loads(result.stdout)

def analyze_jobs(repo: str, run_id: int):
    """Get job durations for a specific run"""
    result = subprocess.run(
        ["gh", "api", f"repos/{repo}/actions/runs/{run_id}/jobs"],
        capture_output=True, text=True
    )
    jobs = json.loads(result.stdout).get("jobs", [])
    
    job_times = {}
    for job in jobs:
        if job["completed_at"] and job["started_at"]:
            start = datetime.fromisoformat(job["started_at"].replace("Z", "+00:00"))
            end = datetime.fromisoformat(job["completed_at"].replace("Z", "+00:00"))
            duration = (end - start).total_seconds()
            job_times[job["name"]] = duration
    
    return job_times

def find_bottlenecks(repo: str, workflow: str, num_runs: int = 20):
    """Identify pipeline bottlenecks"""
    runs = get_workflow_runs(repo, workflow)[:num_runs]
    
    all_job_times = defaultdict(list)
    
    for run in runs:
        job_times = analyze_jobs(repo, run["id"])
        for job_name, duration in job_times.items():
            all_job_times[job_name].append(duration)
    
    print(f"\n{'='*60}")
    print(f"Pipeline Bottleneck Analysis: {workflow}")
    print(f"Based on last {num_runs} runs")
    print(f"{'='*60}\n")
    
    sorted_jobs = sorted(
        all_job_times.items(),
        key=lambda x: statistics.mean(x[1]),
        reverse=True
    )
    
    print(f"{'Job Name':<40} {'Avg':>8} {'P50':>8} {'P95':>8} {'Max':>8}")
    print("-" * 70)
    
    for job_name, times in sorted_jobs:
        avg = statistics.mean(times)
        p50 = statistics.median(times)
        p95 = sorted(times)[int(len(times) * 0.95)]
        max_t = max(times)
        
        print(f"{job_name:<40} {avg:>7.0f}s {p50:>7.0f}s {p95:>7.0f}s {max_t:>7.0f}s")
    
    print("\n🔴 Top Bottlenecks:")
    for i, (job_name, times) in enumerate(sorted_jobs[:3]):
        avg = statistics.mean(times)
        print(f"  {i+1}. {job_name}: avg {avg:.0f}s")
        print(f"     Recommendation: Consider parallelization or optimization")

if __name__ == "__main__":
    find_bottlenecks("myorg/myrepo", "CI Pipeline")
```

### Visual Pipeline Analysis

```bash
#!/bin/bash
# scripts/visualize-pipeline.sh
# สร้าง ASCII timeline ของ pipeline

JOB_DATA='[
  {"name": "checkout", "start": 0, "end": 5},
  {"name": "install", "start": 5, "end": 65},
  {"name": "lint", "start": 65, "end": 90},
  {"name": "test-unit", "start": 65, "end": 110},
  {"name": "test-integration", "start": 65, "end": 145},
  {"name": "build", "start": 145, "end": 200},
  {"name": "docker-build", "start": 200, "end": 260},
  {"name": "deploy", "start": 260, "end": 280}
]'

echo "$JOB_DATA" | python3 - << 'EOF'
import json
import sys

data = json.load(sys.stdin)
max_time = max(j["end"] for j in data)
width = 60

print(f"\nPipeline Timeline ({max_time}s total)")
print("-" * (width + 25))

for job in data:
    start_pos = int(job["start"] / max_time * width)
    end_pos = int(job["end"] / max_time * width)
    bar = " " * start_pos + "█" * (end_pos - start_pos)
    duration = job["end"] - job["start"]
    print(f"{job['name']:<20} |{bar:<{width}}| {duration}s")

print("-" * (width + 25))
print(f"{'Total':<20} |{'█' * width}| {max_time}s")
EOF
```

---

## Optimization Strategies

### Strategy 1: Early Failure Detection

```yaml
# ทำ fast checks ก่อน เพื่อ fail early
jobs:
  # 1. Fastest checks (< 1 min)
  quick-checks:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Check file formatting
      run: |
        npx prettier --check "**/*.{ts,tsx,js,json,css}" || {
          echo "❌ Code not formatted! Run: npm run format"
          exit 1
        }
    - name: Check commit message
      run: |
        COMMIT_MSG=$(git log -1 --format="%s")
        if ! echo "$COMMIT_MSG" | grep -qP "^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .+"; then
          echo "❌ Invalid commit message: $COMMIT_MSG"
          exit 1
        fi

  # 2. Linting (< 2 min)
  lint:
    needs: quick-checks
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci && npm run lint

  # 3. Tests (ใช้เวลานานกว่า, รอ lint ผ่านก่อน)
  tests:
    needs: lint
    # ...
```

### Strategy 2: Selective Testing

```yaml
jobs:
  changed-files:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
      infra: ${{ steps.filter.outputs.infra }}
    steps:
    - uses: actions/checkout@v4
    - uses: dorny/paths-filter@v3
      id: filter
      with:
        filters: |
          frontend:
            - 'frontend/**'
            - 'shared/ui/**'
          backend:
            - 'backend/**'
            - 'shared/lib/**'
          infra:
            - 'k8s/**'
            - 'terraform/**'

  test-frontend:
    needs: changed-files
    if: needs.changed-files.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
    - run: echo "Running frontend tests"

  test-backend:
    needs: changed-files
    if: needs.changed-files.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
    - run: echo "Running backend tests"
```

### Strategy 3: Optimized Docker Builds

```dockerfile
# syntax=docker/dockerfile:1.5
# Stage-based caching with BuildKit

# ─── Dependencies ───────────────────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app

# Cache mount สำหรับ npm
COPY package*.json .
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production

# ─── Development Dependencies ────────────────────────────────
FROM deps AS dev-deps
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# ─── Builder ─────────────────────────────────────────────────
FROM dev-deps AS builder
COPY . .

# Cache mount สำหรับ build cache (Next.js, webpack, etc.)
RUN --mount=type=cache,target=/app/.next/cache \
    npm run build

# ─── Runner ──────────────────────────────────────────────────
FROM node:20-alpine AS runner
WORKDIR /app

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

COPY --from=builder --chown=nextjs:nodejs /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000
CMD ["node", "server.js"]
```

### Strategy 4: Optimized Test Runner

```javascript
// jest.config.js - Optimized Jest configuration
module.exports = {
  // รัน parallel ตาม CPU cores
  maxWorkers: process.env.CI ? '50%' : '25%',
  
  // Cache test results
  cache: true,
  cacheDirectory: '.jest-cache',
  
  // Bail หลัง N failures
  bail: 5,
  
  // Test timeout
  testTimeout: 10000,
  
  // ไม่รัน test ซ้ำถ้า pass แล้ว
  // (ใช้กับ jest-circus)
  
  // Shard configuration
  // รันผ่าน CLI: jest --shard=1/4
  
  // Skip slow tests ใน quick run
  testPathIgnorePatterns: [
    '/node_modules/',
    process.env.QUICK_TEST ? '/e2e/' : null,
  ].filter(Boolean),
  
  // Code coverage settings
  collectCoverage: !!process.env.COVERAGE,
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

---

## Caching & Parallelism

### Optimized GitHub Actions Pipeline

```yaml
# .github/workflows/optimized-ci.yaml
name: Optimized CI Pipeline

on:
  push:
    branches: [main, 'feature/**']
  pull_request:

jobs:
  # ============================================================
  # PHASE 1: Setup (< 30s)
  # ============================================================
  setup:
    runs-on: ubuntu-latest
    outputs:
      cache-key: ${{ steps.cache-key.outputs.key }}
    steps:
    - uses: actions/checkout@v4
    - id: cache-key
      run: |
        echo "key=${{ runner.os }}-node20-${{ hashFiles('package-lock.json') }}" >> $GITHUB_OUTPUT
    
    - name: Pre-warm cache
      uses: actions/cache@v4
      id: cache
      with:
        path: node_modules
        key: ${{ steps.cache-key.outputs.key }}
    
    - name: Install (only on cache miss)
      if: steps.cache.outputs.cache-hit != 'true'
      run: npm ci

  # ============================================================
  # PHASE 2: Parallel fast checks (< 2 min each)
  # ============================================================
  lint:
    needs: setup
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ needs.setup.outputs.cache-key }}
    - run: npm run lint 2>&1 | head -50   # limit output

  type-check:
    needs: setup
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ needs.setup.outputs.cache-key }}
    - run: npm run type-check

  security:
    needs: setup
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npx audit-ci --moderate
    - uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        severity: 'CRITICAL'
        exit-code: '1'

  # ============================================================
  # PHASE 3: Tests (parallel shards)
  # ============================================================
  unit-tests:
    needs: [lint, type-check]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3]
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - uses: actions/cache@v4
      with:
        path: node_modules
        key: ${{ needs.setup.outputs.cache-key }}
    - uses: actions/cache@v4
      with:
        path: .jest-cache
        key: jest-${{ runner.os }}-${{ matrix.shard }}-${{ hashFiles('src/**') }}
        restore-keys: jest-${{ runner.os }}-${{ matrix.shard }}-
    - run: |
        npx jest \
          --shard=${{ matrix.shard }}/3 \
          --cache \
          --cacheDirectory .jest-cache \
          --maxWorkers=2
    - uses: actions/upload-artifact@v4
      with:
        name: coverage-${{ matrix.shard }}
        path: coverage

  # ============================================================
  # PHASE 4: Build (after tests pass)
  # ============================================================
  build:
    needs: [unit-tests, security]
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - id: meta
      uses: docker/metadata-action@v5
      with:
        images: ghcr.io/myorg/myapp
        tags: type=sha,format=short
    - uses: docker/build-push-action@v5
      with:
        context: .
        push: ${{ github.event_name == 'push' }}
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        build-args: |
          BUILDTIME=${{ github.event.head_commit.timestamp }}
          VERSION=${{ github.sha }}
```

---

## Incremental Builds

### Go Build Cache

```yaml
# Optimized Go pipeline
- name: Setup Go with cache
  uses: actions/setup-go@v5
  with:
    go-version-file: go.mod
    cache: true
    cache-dependency-path: go.sum

# Go build cache อยู่ใน ~/.cache/go-build
# actions/setup-go จัดการให้อัตโนมัติ

- name: Build (incremental)
  run: |
    go build -v ./...
    # Go จะ reuse compiled packages จาก cache
    # เฉพาะ packages ที่เปลี่ยนจะ recompile
```

### webpack Incremental Build

```javascript
// webpack.config.js
module.exports = {
  cache: {
    type: 'filesystem',
    cacheDirectory: '.webpack-cache',
    buildDependencies: {
      config: [__filename],
    },
  },
  // snapshot options สำหรับ cache validation
  snapshot: {
    module: {
      timestamp: true,
      hash: true,
    },
  },
};
```

```yaml
- uses: actions/cache@v4
  with:
    path: .webpack-cache
    key: webpack-${{ runner.os }}-${{ hashFiles('webpack.config.js', 'src/**') }}
    restore-keys: webpack-${{ runner.os }}-

- run: npx webpack --config webpack.config.js
```

### Turborepo Incremental

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "inputs": [
        "src/**",
        "public/**",
        "package.json",
        "tsconfig.json"
      ],
      "outputs": [".next/**", "dist/**"],
      "cache": true
    }
  }
}
```

```yaml
- name: Turbo build (incremental)
  run: npx turbo run build
  env:
    TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}
    TURBO_TEAM: ${{ vars.TURBO_TEAM }}
  # Turbo จะ skip packages ที่ inputs ไม่เปลี่ยน
  # และ pull results จาก remote cache
```

---

## Pipeline Analytics

### GitHub Actions Usage API

```python
#!/usr/bin/env python3
# scripts/pipeline-analytics.py

import json
import subprocess
from collections import defaultdict
from datetime import datetime, timedelta

def get_workflow_analytics(repo: str, days: int = 30):
    """Get comprehensive pipeline analytics"""
    
    # Fetch runs
    result = subprocess.run(
        ["gh", "api",
         f"repos/{repo}/actions/runs",
         "--jq", "[.workflow_runs[] | select(.status == \"completed\")]",
         "--paginate"],
        capture_output=True, text=True
    )
    runs = json.loads(result.stdout)
    
    # Filter by date
    cutoff = datetime.utcnow() - timedelta(days=days)
    recent_runs = [
        r for r in runs
        if datetime.strptime(r["created_at"][:19], "%Y-%m-%dT%H:%M:%S") > cutoff
    ]
    
    # Calculate metrics
    total = len(recent_runs)
    success = sum(1 for r in recent_runs if r["conclusion"] == "success")
    failed = sum(1 for r in recent_runs if r["conclusion"] == "failure")
    
    durations = []
    for run in recent_runs:
        if run.get("updated_at") and run.get("created_at"):
            start = datetime.strptime(run["created_at"][:19], "%Y-%m-%dT%H:%M:%S")
            end = datetime.strptime(run["updated_at"][:19], "%Y-%m-%dT%H:%M:%S")
            durations.append((end - start).total_seconds())
    
    avg_duration = sum(durations) / len(durations) if durations else 0
    sorted_durations = sorted(durations)
    p50 = sorted_durations[len(sorted_durations) // 2] if sorted_durations else 0
    p95 = sorted_durations[int(len(sorted_durations) * 0.95)] if sorted_durations else 0
    
    print(f"\n{'='*60}")
    print(f"Pipeline Analytics for {repo}")
    print(f"Period: Last {days} days")
    print(f"{'='*60}")
    print(f"\n📊 Summary:")
    print(f"  Total runs: {total}")
    print(f"  Success rate: {success/total*100:.1f}%")
    print(f"  Failed: {failed} ({failed/total*100:.1f}%)")
    print(f"\n⏱️  Duration:")
    print(f"  Average: {avg_duration/60:.1f} min")
    print(f"  P50:     {p50/60:.1f} min")
    print(f"  P95:     {p95/60:.1f} min")
    print(f"\n💡 Recommendations:")
    
    if avg_duration > 1800:  # > 30 min
        print(f"  🔴 Pipeline is too slow (avg {avg_duration/60:.0f} min)")
        print(f"     → Consider parallelization and caching")
    elif avg_duration > 600:  # > 10 min
        print(f"  🟡 Pipeline could be faster (avg {avg_duration/60:.0f} min)")
        print(f"     → Look for bottlenecks in slow stages")
    else:
        print(f"  🟢 Pipeline speed is good!")
    
    if success/total < 0.9:
        print(f"  🔴 Success rate is low ({success/total*100:.1f}%)")
        print(f"     → Investigate flaky tests")

if __name__ == "__main__":
    get_workflow_analytics("myorg/myrepo")
```

### Grafana Dashboard สำหรับ CI/CD

```json
{
  "title": "CI/CD Pipeline Dashboard",
  "panels": [
    {
      "title": "Pipeline Duration Trend",
      "type": "timeseries",
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(github_workflow_duration_seconds_bucket[1h]))",
        "legendFormat": "P95 Duration"
      }]
    },
    {
      "title": "Success Rate",
      "type": "gauge",
      "targets": [{
        "expr": "sum(rate(github_workflow_runs_total{conclusion='success'}[24h])) / sum(rate(github_workflow_runs_total[24h])) * 100",
        "legendFormat": "Success Rate %"
      }]
    },
    {
      "title": "Flaky Tests",
      "type": "table",
      "targets": [{
        "expr": "topk(10, github_flaky_tests_total)",
        "legendFormat": "{{test_name}}"
      }]
    }
  ]
}
```

---

## DORA Metrics Improvement

### DORA 4 Key Metrics

```
1. Deployment Frequency
   - Elite: Multiple deploys per day
   - High: Between once per day and once per week
   - Medium: Between once per week and once per month
   - Low: Less than once per month

2. Lead Time for Changes
   - Elite: < 1 hour
   - High: Between 1 day and 1 week
   - Medium: Between 1 week and 1 month
   - Low: > 6 months

3. Mean Time To Recovery (MTTR)
   - Elite: < 1 hour
   - High: < 1 day
   - Medium: 1 day to 1 week
   - Low: > 6 months

4. Change Failure Rate
   - Elite: 0-15%
   - High: 16-30%
   - Medium/Low: 31-45%+
```

### Tracking DORA Metrics

```yaml
# .github/workflows/track-dora.yaml
name: Track DORA Metrics

on:
  deployment_status:
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]

jobs:
  track-deployment-frequency:
    if: github.event_name == 'deployment_status' && github.event.deployment_status.state == 'success'
    runs-on: ubuntu-latest
    steps:
    - name: Record deployment
      run: |
        curl -X POST https://metrics.mycompany.com/deployments \
          -H "Content-Type: application/json" \
          -d '{
            "timestamp": "${{ github.event.deployment_status.created_at }}",
            "environment": "${{ github.event.deployment.environment }}",
            "ref": "${{ github.event.deployment.ref }}",
            "sha": "${{ github.sha }}"
          }'

  track-lead-time:
    if: github.event_name == 'deployment_status' && github.event.deployment_status.state == 'success'
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Calculate lead time
      run: |
        # หา commit time ของ first commit ใน this deployment
        FIRST_COMMIT_TIME=$(git log \
          --format="%ct" \
          origin/main..HEAD | tail -1)
        
        DEPLOY_TIME=$(date +%s)
        LEAD_TIME=$((DEPLOY_TIME - FIRST_COMMIT_TIME))
        
        echo "Lead time: $((LEAD_TIME/3600)) hours"
        
        # ส่งไปยัง metrics system
        curl -X POST https://metrics.mycompany.com/lead-time \
          -d "{ \"seconds\": $LEAD_TIME }"
```

### Pipeline Optimization Checklist

```yaml
# .github/workflows/optimization-audit.yaml
name: Pipeline Optimization Audit

on:
  schedule:
    - cron: '0 9 * * 1'  # ทุกวันจันทร์
  workflow_dispatch:

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
    - name: Run optimization audit
      run: |
        echo "## Pipeline Optimization Audit" >> $GITHUB_STEP_SUMMARY
        echo "" >> $GITHUB_STEP_SUMMARY
        
        # Check 1: Pipeline duration
        AVG_DURATION=$(gh api repos/${{ github.repository }}/actions/runs \
          --jq '[.workflow_runs[:50] | .[] |
            ((.updated_at | fromdate) - (.created_at | fromdate))] |
            add/length' 2>/dev/null || echo "0")
        
        echo "### Duration" >> $GITHUB_STEP_SUMMARY
        if (( $(echo "$AVG_DURATION > 1800" | bc -l) )); then
          echo "🔴 Avg duration: $((AVG_DURATION/60)) min (> 30 min - needs optimization)" >> $GITHUB_STEP_SUMMARY
        elif (( $(echo "$AVG_DURATION > 600" | bc -l) )); then
          echo "🟡 Avg duration: $((AVG_DURATION/60)) min (10-30 min - room for improvement)" >> $GITHUB_STEP_SUMMARY
        else
          echo "🟢 Avg duration: $((AVG_DURATION/60)) min (< 10 min - good!)" >> $GITHUB_STEP_SUMMARY
        fi
        
        echo "" >> $GITHUB_STEP_SUMMARY
        echo "### Recommendations" >> $GITHUB_STEP_SUMMARY
        echo "1. Enable caching for dependencies" >> $GITHUB_STEP_SUMMARY
        echo "2. Use parallel jobs where possible" >> $GITHUB_STEP_SUMMARY
        echo "3. Implement test sharding for large test suites" >> $GITHUB_STEP_SUMMARY
        echo "4. Use Docker layer caching" >> $GITHUB_STEP_SUMMARY
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Exercises

### Exercise 1: Baseline Measurement

```bash
#!/bin/bash
# exercise-1-measure-baseline.sh

echo "=== Measuring Pipeline Baseline ==="

# สร้าง test repository ถ้ายังไม่มี
mkdir -p benchmark-app && cd benchmark-app
git init

cat > package.json << 'EOF'
{
  "name": "benchmark-app",
  "scripts": {
    "test": "jest",
    "build": "webpack",
    "lint": "eslint src/"
  }
}
EOF

# สร้าง workflow ที่ measure time
cat > .github/workflows/benchmark.yaml << 'EOF'
name: Benchmark

on: push

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
    - name: Time - start
      id: t0
      run: echo "t=$(date +%s%3N)" >> $GITHUB_OUTPUT
    
    - uses: actions/checkout@v4
    
    - name: Time - after checkout
      run: |
        NOW=$(date +%s%3N)
        echo "Checkout: $((NOW - ${{ steps.t0.outputs.t }}))ms"
    
    - name: Install
      id: t1
      run: |
        echo "t=$(date +%s%3N)" >> $GITHUB_OUTPUT
        npm ci
    
    - name: Time - after install
      run: |
        NOW=$(date +%s%3N)
        echo "Install: $((NOW - ${{ steps.t1.outputs.t }}))ms"
    
    - name: Test
      id: t2
      run: |
        echo "t=$(date +%s%3N)" >> $GITHUB_OUTPUT
        npm test
    
    - name: Time - after test
      run: |
        NOW=$(date +%s%3N)
        TOTAL=$((NOW - ${{ steps.t0.outputs.t }}))
        echo "Tests: $((NOW - ${{ steps.t2.outputs.t }}))ms"
        echo "Total: ${TOTAL}ms"
        
        echo "## Benchmark Results" >> $GITHUB_STEP_SUMMARY
        echo "Total: ${TOTAL}ms" >> $GITHUB_STEP_SUMMARY
EOF
```

### Exercise 2: Apply Optimizations

```yaml
# .github/workflows/optimized.yaml
name: Optimized Pipeline

on: push

jobs:
  # Optimization 1: Use setup actions with caching
  optimized-install:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node (with cache)
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'           # ✅ built-in cache
    
    - run: npm ci
    
  # Optimization 2: Parallel jobs
  parallel-checks:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with: {node-version: '20', cache: 'npm'}
    - run: npm ci
    
    # ✅ รัน lint, type-check, test parallel
    - name: Run all checks in parallel
      run: |
        npm run lint &
        npm run type-check &
        npm test -- --passWithNoTests &
        wait  # รอทั้งหมด

  # Optimization 3: Docker cache
  optimized-docker:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: docker/setup-buildx-action@v3
    - uses: docker/build-push-action@v5
      with:
        context: .
        push: false
        cache-from: type=gha   # ✅ GitHub Actions cache
        cache-to: type=gha,mode=max
```

### Exercise 3: Measure Improvement

```python
#!/usr/bin/env python3
# exercise-3-compare.py
"""
Compare baseline vs optimized pipeline
"""

baseline_times = {
    "install": 180,
    "lint": 60,
    "tests": 300,
    "build": 120,
    "docker": 240,
    "total": 900,  # Sequential
}

optimized_times = {
    "install": 30,    # Cache hit
    "lint": 30,       # Parallel with tests
    "tests": 120,     # Sharded (3x)
    "build": 90,      # Incremental
    "docker": 60,     # Layer cache
    "total": 330,     # Best case parallel
}

print("Pipeline Optimization Results")
print("="*50)
print(f"{'Metric':<20} {'Baseline':>10} {'Optimized':>10} {'Savings':>10}")
print("-"*50)

for key in baseline_times:
    baseline = baseline_times[key]
    optimized = optimized_times[key]
    savings = baseline - optimized
    pct = savings / baseline * 100
    print(f"{key:<20} {baseline:>9}s {optimized:>9}s {pct:>9.0f}%")

total_savings = baseline_times["total"] - optimized_times["total"]
print(f"\nTotal time saved: {total_savings}s ({total_savings/60:.0f} min)")
print(f"Improvement: {total_savings/baseline_times['total']*100:.0f}%")
```

### Exercise 4: DORA Metrics Dashboard

```yaml
# .github/workflows/dora-dashboard.yaml
name: DORA Metrics

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  calculate-dora:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Calculate DORA metrics
      run: |
        echo "## DORA Metrics Report" >> $GITHUB_STEP_SUMMARY
        echo "" >> $GITHUB_STEP_SUMMARY
        
        # Deployment Frequency (commits to main per day)
        COMMITS_30D=$(git log \
          --after="30 days ago" \
          --oneline \
          origin/main | wc -l)
        
        DEPLOY_FREQ=$(echo "scale=2; $COMMITS_30D / 30" | bc)
        echo "### 1. Deployment Frequency" >> $GITHUB_STEP_SUMMARY
        echo "Commits to main per day (last 30 days): $DEPLOY_FREQ" >> $GITHUB_STEP_SUMMARY
        
        if (( $(echo "$DEPLOY_FREQ > 1" | bc -l) )); then
          echo "🟢 Elite: Multiple deployments per day" >> $GITHUB_STEP_SUMMARY
        elif (( $(echo "$DEPLOY_FREQ >= 0.14" | bc -l) )); then
          echo "🟡 High: Once per week to once per day" >> $GITHUB_STEP_SUMMARY
        else
          echo "🔴 Medium/Low: Less than once per week" >> $GITHUB_STEP_SUMMARY
        fi
        
        # Lead Time
        FIRST_COMMIT=$(git log \
          --format="%ct" \
          origin/main \
          | tail -n +2 | head -1)
        
        LAST_COMMIT=$(git log \
          --format="%ct" \
          origin/main \
          | head -1)
        
        echo "" >> $GITHUB_STEP_SUMMARY
        echo "### 2. Recommendations for Improvement" >> $GITHUB_STEP_SUMMARY
        echo "- Implement automated testing for faster feedback" >> $GITHUB_STEP_SUMMARY
        echo "- Use feature flags for safer deployments" >> $GITHUB_STEP_SUMMARY
        echo "- Set up automated rollback on failures" >> $GITHUB_STEP_SUMMARY
        echo "- Monitor error rates post-deployment" >> $GITHUB_STEP_SUMMARY
```

---

## Pipeline Optimization Checklist

```
✅ CACHING
□ Dependencies cached (npm, go, pip, etc.)
□ Docker layers ordered correctly (deps before source)
□ BuildKit cache mounts enabled
□ Test results cached (Jest, pytest)
□ Build artifacts cached where applicable

✅ PARALLELISM
□ Independent jobs run in parallel
□ Test suite sharded across jobs
□ Matrix builds for multi-platform
□ Fan-out/Fan-in pattern used appropriately

✅ EARLY FAILURE
□ Fast checks run before slow ones
□ Lint/format check before tests
□ Security scan early in pipeline
□ PR checks faster than main branch checks

✅ SELECTIVE TESTING
□ Path-based triggering configured
□ Only affected tests run on PR
□ Full suite on main branch

✅ DOCKER OPTIMIZATION
□ Multi-stage builds
□ Layer cache with GHA or registry
□ .dockerignore configured
□ Small base images (alpine, distroless)

✅ MONITORING
□ Pipeline duration tracked
□ Success rate monitored
□ DORA metrics calculated
□ Flaky tests identified and fixed

✅ INFRASTRUCTURE
□ Self-hosted runners for speed (if needed)
□ Concurrent job limits set
□ Runner sizes appropriate for workload
□ ARM64 runners for M1/M2 Macs (if applicable)
```

---

## สรุป Part 50 และ Series

Pipeline Optimization เป็นกระบวนการต่อเนื่อง:

1. **Measure** - วัด baseline ก่อน optimize
2. **Identify Bottlenecks** - หา slow steps
3. **Apply Optimizations** - cache, parallel, incremental
4. **Verify Improvement** - วัดอีกครั้ง
5. **Monitor Continuously** - ติดตาม DORA metrics

### สรุป Series Part 41-50

```
Part 41: GitOps        - Git as single source of truth
Part 42: ArgoCD        - GitOps operator for K8s
Part 43: Flux CD       - Alternative GitOps toolkit
Part 44: Multi-Branch  - Pipeline strategy per branch
Part 45: Monorepo      - CI/CD for single repository
Part 46: Microservices - Independent deployment
Part 47: API Gateway   - Kong, Traefik, Nginx, Istio
Part 48: Cache         - Speed up with caching
Part 49: Parallel      - Matrix and parallel builds
Part 50: Optimization  - Measure and improve
```

---

*จบ Series Part 41-50: Advanced CI/CD Patterns*

*ขอบคุณที่ติดตามคอร์สนี้ หวังว่าจะเป็นประโยชน์ในการนำไปปรับใช้กับ project จริง!*

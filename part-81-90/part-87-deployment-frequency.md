# Part 87: Deployment Frequency & Lead Time Optimization

## บทนำ

Deployment Frequency และ Lead Time for Changes เป็น 2 ใน 4 DORA Metrics ที่สำคัญที่สุด ซึ่งสะท้อนถึงความสามารถขององค์กรในการ deliver software ได้อย่างรวดเร็วและมั่นใจ บทนี้จะลงรายละเอียดเทคนิคและ practices ที่ช่วยให้ deploy ได้บ่อยขึ้นและเร็วขึ้น

## สารบัญ

1. [ทำความเข้าใจ DORA Metrics](#dora-metrics)
2. [Trunk-Based Development](#trunk-based)
3. [Feature Flags ที่ Scale](#feature-flags)
4. [Automated Testing Pyramid](#testing-pyramid)
5. [Deployment Pipeline Optimization](#pipeline-optimization)
6. [Continuous Deployment vs Continuous Delivery](#cd-vs-cd)
7. [Measuring และ Improving Metrics](#measuring)
8. [Case Studies](#case-studies)
9. [แบบฝึกหัด](#exercises)

---

## 1. ทำความเข้าใจ DORA Metrics {#dora-metrics}

### 4 Key DORA Metrics

```
4 DORA Metrics ที่วัด Software Delivery Performance:

1. Deployment Frequency (DF)
   Q: ทีมของคุณ deploy to production บ่อยแค่ไหน?
   
   Elite:  Multiple times per day
   High:   Between once per day and once per week
   Medium: Between once per week and once per month
   Low:    Between once per month and once every 6 months

2. Lead Time for Changes (LTC)
   Q: เวลาตั้งแต่ commit code ถึง production ใช้นานแค่ไหน?
   
   Elite:  Less than 1 hour
   High:   Between 1 day and 1 week
   Medium: Between 1 week and 1 month
   Low:    Between 1 month and 6 months

3. Change Failure Rate (CFR)
   Q: กี่ % ของ deployments ทำให้ service degrade?
   
   Elite:  0-15%
   High:   16-30%
   Medium: 16-30%
   Low:    16-30%

4. Mean Time to Recovery (MTTR)
   Q: ใช้เวลานานแค่ไหนในการ restore service?
   
   Elite:  Less than 1 hour
   High:   Less than 1 day
   Medium: Between 1 day and 1 week
   Low:    More than 1 week
```

### ความสัมพันธ์ระหว่าง DF และ LTC

```
Lead Time = Wait Time + Build Time + Test Time + Deploy Time

Wait Time (ใหญ่ที่สุด):
- รอ code review: 0-48 hours
- รอ CI/CD queue: 0-2 hours
- รอ staging environment: 0-4 hours
- รอ manual approval: 0-48 hours

Build Time:
- Compilation: 1-30 minutes

Test Time:
- Unit tests: 2-15 minutes
- Integration tests: 10-60 minutes
- E2E tests: 30-120 minutes

Deploy Time:
- Actual deployment: 5-30 minutes
```

### ทำไมถึงต้อง deploy บ่อยๆ?

```
Counter-intuitive: deploy บ่อยขึ้น = safer

ทำไม?

1. Batch size เล็กลง
   deploy 1 change ≠ เสี่ยงน้อยกว่า deploy 100 changes
   ถ้า deploy ไม่ผ่าน → ง่ายต่อการหา root cause

2. Feedback loop เร็วขึ้น
   รู้ผลภายใน hours ไม่ใช่ weeks

3. Rollback ง่ายขึ้น
   ย้อน 1 change ง่ายกว่าย้อน 100 changes

4. Pressure ลดลง
   ไม่มี "big bang release" ที่ทุกคนเครียด

5. Team velocity เพิ่มขึ้น
   ไม่มี integration hell ที่เกิดจาก long-lived branches
```

---

## 2. Trunk-Based Development {#trunk-based}

### Trunk-Based Development (TBD) คืออะไร?

```
Traditional (Feature Branches):
main ─────────────────────────────────► 
     └─ feature/big-feature ─────────►  (2 weeks)
                                     └─ merge (chaos)

Trunk-Based Development:
main ─────┬─────┬─────┬─────────────►
          │     │     │
          feat  feat  feat  (hours/days, not weeks)
          │     │     │
          └─────┴─────┘ (frequent small merges)
```

### TBD Practices

**1. Short-lived Branches**
```bash
# หลักการ: branch มีชีวิตไม่เกิน 1-2 วัน

# สร้าง branch
git checkout -b feat/add-payment-method

# code
git add .
git commit -m "feat: add Apple Pay support"

# merge กลับภายใน 1 วัน
git checkout main
git merge feat/add-payment-method
git push origin main

# ลบ branch
git branch -d feat/add-payment-method
```

**2. Feature Flags สำหรับ Incomplete Features**
```python
# แทนที่จะ hold feature ไว้ใน branch
# ใช้ feature flag เพื่อ hide incomplete work

from feature_flags import is_enabled

def process_payment(payment: Payment, user: User):
    if is_enabled("new-payment-flow", user.id):
        return new_payment_processor.process(payment)
    else:
        return legacy_payment_processor.process(payment)
```

**3. Branch by Abstraction**
```python
# สำหรับ large refactoring

# Step 1: สร้าง abstraction layer
class PaymentProcessor:
    def __init__(self, implementation: PaymentImplementation):
        self.impl = implementation
    
    def process(self, payment: Payment) -> Result:
        return self.impl.process(payment)

# Step 2: ใช้ implementation เดิม
processor = PaymentProcessor(LegacyPaymentImpl())

# Step 3: พัฒนา new implementation ขนานกัน (merge บ่อยๆ)
class NewPaymentImpl(PaymentImplementation):
    def process(self, payment: Payment) -> Result:
        # new implementation...
        pass

# Step 4: Switch implementation เมื่อพร้อม
if feature_flags.is_enabled("new-payment-impl"):
    processor = PaymentProcessor(NewPaymentImpl())
else:
    processor = PaymentProcessor(LegacyPaymentImpl())

# Step 5: Remove old implementation เมื่อ stable
```

### TBD กับ Code Review

```
ปัญหาของ TBD: "ถ้า merge บ่อยๆ จะมีเวลา review ไหม?"

Solutions:

1. Pair Programming
   - Review เกิดขึ้นแบบ real-time
   - ไม่ต้องรอ async review

2. Smaller PRs = Faster Reviews
   PR ใหญ่ (500 lines): รอ review 1-2 วัน
   PR เล็ก (50 lines): รอ review 1-2 ชั่วโมง

3. Asynchronous Review Tricks
   - Draft PRs สำหรับ WIP
   - Auto-assign reviewers
   - Review turnaround SLA (e.g., 4 hours)

4. Automated Checks
   - Linting, formatting → ไม่ต้องถามใน review
   - Unit tests → reviewer มั่นใจ
   - Security scan → ลด burden บน reviewers
```

---

## 3. Feature Flags ที่ Scale {#feature-flags}

### Feature Flag Architecture

```
Feature Flag สำหรับ Enterprise:

Simple (Wrong for large scale):
if feature_flags["new-ui"]:
    show_new_ui()

Enterprise-grade (LaunchDarkly/Unleash):
- Targeting rules (% rollout, specific users)
- Real-time updates (no redeploy)
- Analytics integration
- A/B testing
- Kill switch
```

### Implementing Feature Flags

```python
# feature_flags/client.py

from functools import lru_cache
import ldclient
from ldclient.config import Config

class FeatureFlagClient:
    def __init__(self, sdk_key: str):
        ldclient.set_config(Config(sdk_key))
        self.client = ldclient.get()
    
    def is_enabled(
        self, 
        flag_key: str, 
        user_id: str = None,
        context: dict = None
    ) -> bool:
        """ตรวจสอบว่า feature เปิดอยู่หรือเปล่า"""
        
        if not user_id:
            return self.client.variation(flag_key, None, False)
        
        # Build user context
        user = {
            "key": user_id,
            **(context or {})
        }
        
        return self.client.variation(flag_key, user, False)
    
    def get_variation(
        self, 
        flag_key: str, 
        default: any,
        user_id: str = None
    ) -> any:
        """ดึงค่า variation (สำหรับ A/B testing)"""
        
        user = {"key": user_id} if user_id else None
        return self.client.variation(flag_key, user, default)
    
    def track_metric(self, metric_key: str, user_id: str, value: float = None):
        """Track metric สำหรับ feature"""
        user = {"key": user_id}
        self.client.track(metric_key, user, None, value)


# Usage
flags = FeatureFlagClient(sdk_key=os.environ['LAUNCHDARKLY_SDK_KEY'])

# Simple on/off
if flags.is_enabled("new-checkout-flow", user.id):
    response = new_checkout(cart)
else:
    response = old_checkout(cart)

# Percentage rollout
# ใน LaunchDarkly: configure 10% rollout
if flags.is_enabled("beta-search", user.id):
    results = beta_search(query)
else:
    results = standard_search(query)

# A/B testing
checkout_variant = flags.get_variation(
    "checkout-button-color", 
    default="blue",
    user_id=user.id
)
# Returns "blue" or "green" based on rules
```

### Feature Flag Lifecycle Management

```python
# feature_flags/lifecycle.py

FLAG_REGISTRY = {
    "new-checkout-flow": {
        "created": "2024-01-15",
        "owner": "team-checkout",
        "status": "active",  # draft, active, deprecated, removed
        "rollout": 100,  # percentage
        "type": "release",  # release, experiment, ops, permission
        "description": "New checkout flow with improved UX",
        "removal_date": None,  # กำหนดเมื่อถึงเวลา remove
    }
}

def audit_stale_flags() -> list:
    """หา feature flags ที่ค้างอยู่"""
    
    stale_flags = []
    
    for flag_key, flag_info in FLAG_REGISTRY.items():
        created_date = datetime.fromisoformat(flag_info["created"])
        age_days = (datetime.now() - created_date).days
        
        if flag_info["type"] == "release" and age_days > 30 and flag_info["rollout"] == 100:
            stale_flags.append({
                "key": flag_key,
                "age_days": age_days,
                "issue": "100% rolled out for 30+ days, should be removed",
                "owner": flag_info["owner"]
            })
        
        if flag_info["type"] == "experiment" and age_days > 14:
            stale_flags.append({
                "key": flag_key,
                "age_days": age_days,
                "issue": "Experiment running for 14+ days",
                "owner": flag_info["owner"]
            })
    
    return stale_flags
```

### Feature Flag Best Practices

```
DO:
✅ ใช้ flags สำหรับ incomplete features ที่ต้องการ merge บ่อยๆ
✅ กำหนด owner ชัดเจน
✅ ตั้ง removal date ตั้งแต่แรก
✅ Test ทั้ง flag ON และ flag OFF
✅ Monitor metrics เมื่อ enable flag

DON'T:
❌ ใช้ flags แทน proper branching strategy
❌ Stack flags หลายชั้น (flag inside flag)
❌ ทิ้ง flags ไว้หลัง 100% rollout
❌ ใช้ flags สำหรับ business logic ถาวร
❌ Hardcode flag keys ใน code
```

---

## 4. Automated Testing Pyramid {#testing-pyramid}

### Testing Pyramid

```
        ┌─────────────────────┐
        │       E2E Tests      │  (10%)
        │  Slow, expensive     │
        │  High value          │
        └─────────────────────┘
      ┌───────────────────────────┐
      │   Integration Tests        │  (20%)
      │  Medium speed, medium cost │
      └───────────────────────────┘
    ┌─────────────────────────────────┐
    │           Unit Tests             │  (70%)
    │   Fast, cheap, high coverage     │
    └─────────────────────────────────┘

เป้าหมาย:
- Unit tests: < 5 minutes
- Integration tests: < 15 minutes
- E2E tests: < 30 minutes
- Total: < 45 minutes (ideally < 15 min)
```

### Optimizing Test Execution

**1. Test Parallelization**
```yaml
# .github/workflows/parallel-tests.yml

jobs:
  test:
    strategy:
      matrix:
        # แบ่ง tests เป็น 4 groups
        test-group: [1, 2, 3, 4]
    
    runs-on: ubuntu-latest
    
    steps:
      - name: Run Test Shard
        run: |
          pytest \
            --shard-id=${{ matrix.test-group }} \
            --num-shards=4 \
            tests/
```

**2. Test Selection (Run Only What's Affected)**
```python
# scripts/test-selector.py
# ใช้ git diff เพื่อ select tests ที่เกี่ยวข้อง

import subprocess
import ast
import importlib

def get_affected_tests(changed_files: list) -> list:
    """หา tests ที่ควร run เมื่อมี file เปลี่ยนแปลง"""
    
    affected_tests = set()
    
    for changed_file in changed_files:
        # หา modules ที่ import file ที่เปลี่ยน
        importers = find_importers(changed_file)
        
        # หา test files สำหรับ modules นั้น
        for importer in importers:
            test_file = find_test_file(importer)
            if test_file:
                affected_tests.add(test_file)
        
        # Always test ตัว changed file เอง
        test_file = find_test_file(changed_file)
        if test_file:
            affected_tests.add(test_file)
    
    return list(affected_tests)

# ใน pipeline:
CHANGED_FILES=$(git diff --name-only main HEAD)
AFFECTED_TESTS=$(python3 scripts/test-selector.py $CHANGED_FILES)
pytest $AFFECTED_TESTS
```

**3. Flaky Test Detection และ Management**
```python
# scripts/flaky-test-detector.py

class FlakyTestDetector:
    def analyze(self, test_results: list, threshold: float = 0.1) -> list:
        """หา tests ที่ fail บางครั้ง"""
        
        test_stats = {}
        
        for run in test_results:
            for test in run.tests:
                if test.name not in test_stats:
                    test_stats[test.name] = {"pass": 0, "fail": 0}
                
                if test.result == "pass":
                    test_stats[test.name]["pass"] += 1
                else:
                    test_stats[test.name]["fail"] += 1
        
        flaky_tests = []
        for test_name, stats in test_stats.items():
            total = stats["pass"] + stats["fail"]
            fail_rate = stats["fail"] / total
            
            if 0 < fail_rate < threshold:
                flaky_tests.append({
                    "name": test_name,
                    "fail_rate": f"{fail_rate:.1%}",
                    "total_runs": total,
                    "recommendation": self._get_recommendation(fail_rate)
                })
        
        return sorted(flaky_tests, key=lambda x: x["fail_rate"], reverse=True)
```

### Contract Testing สำหรับ Microservices

```python
# contracts/payment_provider_test.py
# Pact Consumer Test

from pact import Consumer, Provider

pact = Consumer('order-service').has_pact_with(Provider('payment-service'))

def test_create_payment():
    """ทดสอบ contract กับ payment service"""
    
    # กำหนด expected interaction
    (pact
        .given("Payment provider is available")
        .upon_receiving("A request to create payment")
        .with_request("POST", "/payments", body={
            "amount": 100.00,
            "currency": "THB",
            "order_id": "ORD-123"
        })
        .will_respond_with(200, body={
            "payment_id": Like("PAY-456"),
            "status": "pending"
        }))
    
    with pact:
        result = order_service.create_payment(
            amount=100.00,
            currency="THB",
            order_id="ORD-123"
        )
    
    assert result.payment_id is not None
    assert result.status == "pending"
```

---

## 5. Deployment Pipeline Optimization {#pipeline-optimization}

### Pipeline Performance Analysis

```python
# scripts/pipeline-analyzer.py

class PipelineAnalyzer:
    def analyze(self, pipeline_runs: list) -> dict:
        """วิเคราะห์ performance ของ pipeline"""
        
        # คำนวณ stats สำหรับแต่ละ stage
        stage_stats = {}
        
        for run in pipeline_runs:
            for stage in run.stages:
                if stage.name not in stage_stats:
                    stage_stats[stage.name] = []
                stage_stats[stage.name].append(stage.duration_seconds)
        
        analysis = {}
        for stage_name, durations in stage_stats.items():
            analysis[stage_name] = {
                "avg_duration": sum(durations) / len(durations),
                "p50": self._percentile(durations, 50),
                "p95": self._percentile(durations, 95),
                "max": max(durations),
                "is_bottleneck": self._is_bottleneck(durations)
            }
        
        # หา top bottlenecks
        bottlenecks = sorted(
            [(k, v) for k, v in analysis.items() if v["is_bottleneck"]],
            key=lambda x: x[1]["avg_duration"],
            reverse=True
        )[:3]
        
        return {
            "stage_analysis": analysis,
            "top_bottlenecks": bottlenecks,
            "total_avg_duration": sum(s["avg_duration"] for s in analysis.values()),
            "optimization_potential": self._calculate_potential(analysis)
        }
    
    def _calculate_potential(self, analysis: dict) -> dict:
        """คำนวณ potential savings"""
        
        current_total = sum(s["avg_duration"] for s in analysis.values())
        
        potential_improvements = {
            "add_caching": current_total * 0.3,  # 30% saving
            "parallelize_tests": current_total * 0.4,  # 40% saving
            "optimize_docker_build": current_total * 0.15,
            "use_faster_runners": current_total * 0.2
        }
        
        return potential_improvements
```

### CI/CD Performance Optimization Techniques

**1. Docker Build Optimization**
```dockerfile
# Bad: ทุกครั้งที่ code เปลี่ยน → ติดตั้ง deps ใหม่
FROM node:18
COPY . .
RUN npm install
RUN npm run build

# Good: cache dependencies layer
FROM node:18 AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18 AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

**2. Pipeline Caching Strategy**
```yaml
# Comprehensive caching strategy

jobs:
  build:
    steps:
      # 1. Maven dependency cache
      - uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: maven-
      
      # 2. Docker layer cache
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/build-push-action@v5
        with:
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      # 3. Test result cache (skip unchanged tests)
      - uses: actions/cache@v4
        with:
          path: .test-cache
          key: tests-${{ hashFiles('tests/**') }}-${{ hashFiles('src/**') }}
```

**3. Stage Parallelization**
```yaml
# แทนที่จะรัน sequential ทุกอย่าง ทำพร้อมกัน

jobs:
  # รันพร้อมกัน
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - run: make test-unit
  
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: make lint
  
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - run: make security-scan
  
  # รอ parallel jobs เสร็จ
  build:
    needs: [unit-tests, lint, security-scan]
    runs-on: ubuntu-latest
    steps:
      - run: make build
  
  # Deploy ต่อจาก build
  deploy-dev:
    needs: build
    steps:
      - run: make deploy-dev
```

---

## 6. Continuous Deployment vs Continuous Delivery {#cd-vs-cd}

### ความแตกต่าง

```
Continuous Integration (CI):
ทุก commit ถูก build และ test อัตโนมัติ
→ ทีมรู้ทันทีว่า code ทำงานหรือไม่

Continuous Delivery (CDel):
ทุก commit พร้อม deploy ได้ตลอดเวลา
แต่ deployment ยังต้องกดปุ่ม (manual trigger)
→ Human decides WHEN to deploy

Continuous Deployment (CDep):
ทุก commit ที่ผ่าน tests จะ deploy อัตโนมัติ
ไม่มี human in the loop
→ ต้องการ confidence สูงใน automated tests
```

### เมื่อไหร่ควรใช้อะไร?

```
Continuous Deployment เหมาะสำหรับ:
✅ SaaS products ที่ release บ่อย
✅ Teams ที่มี test coverage สูง (>85%)
✅ Services ที่ reversible ง่าย
✅ Organizations ที่ mature ใน CI/CD

Continuous Delivery เหมาะสำหรับ:
✅ On-premise software
✅ Regulated industries (SOX, PCI)
✅ Teams ที่กำลัง build confidence
✅ Features ที่ต้องการ coordination

เส้นทางจาก CDel ไป CDep:
1. ให้ test coverage > 80%
2. ให้ deployment success rate > 99%
3. ให้ rollback ใช้เวลา < 5 นาที
4. ให้ monitoring ครอบคลุม
5. Enable automated canary analysis
6. Enable continuous deployment
```

---

## 7. Measuring และ Improving Metrics {#measuring}

### DORA Metrics Collection

```python
# metrics/dora_collector.py

import requests
from datetime import datetime, timedelta

class DORAMetricsCollector:
    def __init__(self, github_token: str, datadog_api_key: str):
        self.github = GitHubClient(github_token)
        self.datadog = DatadogClient(datadog_api_key)
    
    def calculate_deployment_frequency(
        self, 
        repo: str,
        days: int = 30
    ) -> dict:
        """คำนวณ Deployment Frequency"""
        
        deployments = self.github.get_deployments(
            repo=repo,
            environment="production",
            since=datetime.now() - timedelta(days=days)
        )
        
        df = len(deployments) / days
        
        return {
            "deployments_per_day": round(df, 2),
            "total_deployments": len(deployments),
            "period_days": days,
            "elite_threshold": 1.0,  # multiple per day
            "performance_level": self._classify_df(df)
        }
    
    def calculate_lead_time(self, repo: str, days: int = 30) -> dict:
        """คำนวณ Lead Time for Changes"""
        
        prs = self.github.get_merged_prs(
            repo=repo,
            since=datetime.now() - timedelta(days=days)
        )
        
        lead_times = []
        for pr in prs:
            # หา first commit ใน PR
            first_commit = min(
                pr.commits, 
                key=lambda c: c.committed_at
            )
            
            # หา deployment ที่ include PR นี้
            deployment = self._find_deployment_for_pr(pr)
            
            if deployment:
                lead_time_hours = (
                    deployment.created_at - first_commit.committed_at
                ).total_seconds() / 3600
                
                lead_times.append(lead_time_hours)
        
        if lead_times:
            return {
                "median_lead_time_hours": statistics.median(lead_times),
                "p95_lead_time_hours": self._percentile(lead_times, 95),
                "sample_size": len(lead_times),
                "performance_level": self._classify_lt(statistics.median(lead_times))
            }
    
    def _classify_df(self, df_per_day: float) -> str:
        if df_per_day >= 1: return "Elite (≥1/day)"
        if df_per_day >= 1/7: return "High (weekly)"
        if df_per_day >= 1/30: return "Medium (monthly)"
        return "Low (<monthly)"
    
    def _classify_lt(self, hours: float) -> str:
        if hours <= 1: return "Elite (<1 hour)"
        if hours <= 168: return "High (<1 week)"
        if hours <= 720: return "Medium (<1 month)"
        return "Low (>1 month)"
```

### DORA Dashboard

```yaml
# grafana/dora-dashboard.yaml

panels:
  - title: "Deployment Frequency (7-day rolling)"
    type: stat
    query: |
      sum(increase(deployment_total{environment="production"}[7d])) / 7
    thresholds:
      - value: 0
        color: red
        label: Low
      - value: 0.14  # weekly
        color: yellow
        label: Medium
      - value: 1
        color: green
        label: Elite
  
  - title: "Lead Time Distribution"
    type: histogram
    query: |
      histogram_quantile(0.5, rate(deployment_lead_time_hours_bucket[7d]))
  
  - title: "Change Failure Rate"
    type: gauge
    query: |
      (
        sum(deployment_rollback_total{environment="production"}[30d]) /
        sum(deployment_total{environment="production"}[30d])
      ) * 100
    format: percent
    max: 30
  
  - title: "MTTR"
    type: stat
    query: |
      avg(incident_resolution_time_seconds{
        environment="production"
      }) / 3600
    unit: hours
```

---

## 8. Case Studies {#case-studies}

### Case Study 1: E-commerce Platform จาก Weekly ถึง Daily Deploy

**Before:**
```
Deployment Frequency: 1-2 ครั้ง/สัปดาห์
Lead Time: 3-5 วัน
Release process: 2-day freeze + 4-hour deploy window
Change Failure Rate: 25%
MTTR: 4 hours
```

**Root Cause Analysis:**
```
1. Long-lived feature branches (1-3 weeks)
2. ไม่มี feature flags
3. Manual testing required (8-16 hours)
4. Large batch deployments
5. Complex rollback process
```

**Improvements:**
```
Month 1-2:
- Implement feature flags (LaunchDarkly)
- Enforce branch max lifetime: 2 days
- Add automated smoke tests

Month 3-4:
- Implement contract testing
- Improve test coverage: 45% → 75%
- Add automated canary deployment

Month 5-6:
- Enable continuous deployment
- Add comprehensive monitoring
- Create runbooks for common issues
```

**After:**
```
Deployment Frequency: 8-10 ครั้ง/วัน (Elite!)
Lead Time: 4-6 ชั่วโมง (High)
Change Failure Rate: 5%
MTTR: 25 minutes
Developer satisfaction: +40%
Revenue impact: +8% (faster feature delivery)
```

---

## 9. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Calculate DORA Metrics

**Context:** ดูข้อมูล deployment history ของ team:

```python
DEPLOYMENTS = [
    {"date": "2024-01-01", "success": True, "lead_time_hours": 48},
    {"date": "2024-01-03", "success": True, "lead_time_hours": 24},
    {"date": "2024-01-05", "success": False, "lead_time_hours": 36},  # caused incident
    {"date": "2024-01-07", "success": True, "lead_time_hours": 72},
    {"date": "2024-01-14", "success": True, "lead_time_hours": 48},
    {"date": "2024-01-21", "success": True, "lead_time_hours": 36},
    {"date": "2024-01-28", "success": False, "lead_time_hours": 48},  # caused incident
]

INCIDENTS = [
    {"started": "2024-01-05T10:00", "resolved": "2024-01-05T14:30"},  # 4.5 hours
    {"started": "2024-01-28T15:00", "resolved": "2024-01-28T16:00"},  # 1 hour
]
```

**งาน:**
1. คำนวณ Deployment Frequency
2. คำนวณ Mean Lead Time
3. คำนวณ Change Failure Rate
4. คำนวณ MTTR
5. จัดอันดับ performance level
6. ระบุ 3 สิ่งที่ควรปรับปรุง

### แบบฝึกหัดที่ 2: Feature Flag Implementation

**งาน:** implement feature flag สำหรับ use case นี้:
- New checkout flow ที่ยังไม่เสร็จ
- ต้องการ merge บ่อยๆ (TBD)
- จะเปิดให้ 10% users ก่อน

```python
# TODO: implement
class CheckoutService:
    def process(self, cart: Cart, user: User) -> Order:
        # Old implementation
        pass
    
    def new_process(self, cart: Cart, user: User) -> Order:
        # New implementation (incomplete)
        pass
    
    def checkout(self, cart: Cart, user: User) -> Order:
        # TODO: use feature flag here
        pass
```

### แบบฝึกหัดที่ 3: Pipeline Optimization

**Context:** Pipeline ปัจจุบัน:
```
Unit Tests: 12 minutes (sequential)
Integration Tests: 25 minutes (sequential)
E2E Tests: 45 minutes (sequential)
Build: 8 minutes
Docker Push: 3 minutes
Deploy Dev: 5 minutes
Total: 98 minutes
```

**งาน:**
1. Identify optimization opportunities
2. Design parallel execution plan
3. Estimate new total time
4. Write optimized GitHub Actions workflow

---

## สรุป

การปรับปรุง Deployment Frequency และ Lead Time ต้องการการเปลี่ยนแปลงทั้ง process และ technology:

1. **Trunk-Based Development** ลด integration problems
2. **Feature Flags** ให้ merge บ่อยๆ ได้โดยไม่เสี่ยง
3. **Fast Test Suite** ที่ reliable คือ foundation ของทุกอย่าง
4. **Pipeline Optimization** ลด wait time
5. **Measure DORA Metrics** สม่ำเสมอเพื่อ track progress

## อ่านเพิ่มเติม

- "Accelerate" by Forsgren, Humble, Kim
- DORA Research: https://dora.dev/research
- Trunk-Based Development: https://trunkbaseddevelopment.com
- LaunchDarkly Blog: https://launchdarkly.com/blog
- Feature Toggles: https://martinfowler.com/articles/feature-toggles.html

---

*Part 87 จาก 100 | CI/CD Mastery Course*

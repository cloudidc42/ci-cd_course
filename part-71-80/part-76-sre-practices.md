# Part 76: SRE Practices ใน CI/CD

## บทนำ

Site Reliability Engineering (SRE) คือแนวทางที่ Google พัฒนาขึ้น เพื่อสร้างระบบที่มีความน่าเชื่อถือสูงและมีการปรับปรุงอย่างต่อเนื่อง โดยนำหลักการทางวิศวกรรมมาใช้กับ operations

ในบทนี้เราจะเรียนรู้:
- SLOs, SLAs และ SLIs
- Error Budgets
- Toil Reduction
- Reliability in Deployments
- Progressive Rollouts
- แบบฝึกหัดปฏิบัติ

---

## 76.1 SLIs, SLOs และ SLAs

### คำนิยาม

```
SLI (Service Level Indicator)
= ตัวชี้วัดที่วัดประสิทธิภาพจริงของ service
= "สิ่งที่เราวัด"

SLO (Service Level Objective)
= เป้าหมายที่ต้องการบรรลุสำหรับ SLI
= "เป้าหมายที่เราตั้ง"

SLA (Service Level Agreement)
= ข้อตกลงกับ customer เกี่ยวกับ service quality
= "สิ่งที่เรา promise กับ customer"
```

### ตัวอย่าง SLIs

```yaml
# sli-definitions.yaml
slis:
  # Availability
  - name: request_success_rate
    description: "สัดส่วนของ HTTP requests ที่ไม่ใช่ 5xx"
    formula: |
      sum(rate(http_requests_total{status!~"5.."}[5m]))
      /
      sum(rate(http_requests_total[5m]))
    window: 28 days

  # Latency
  - name: request_latency_p99
    description: "99th percentile latency ของ requests"
    formula: |
      histogram_quantile(0.99,
        sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
      )
    window: 28 days

  # Freshness (สำหรับ data pipeline)
  - name: data_freshness
    description: "อายุของข้อมูลล่าสุด"
    formula: |
      time() - max(data_last_updated_timestamp)
    window: 24 hours

  # Throughput
  - name: order_processing_rate
    description: "จำนวน orders ที่ process ได้ต่อนาที"
    formula: |
      sum(rate(orders_processed_total[5m])) * 60
    window: 1 hour
```

### การกำหนด SLOs

```yaml
# slo-definitions.yaml
service: payment-service
team: team-payments

slos:
  - name: payment_availability
    sli: request_success_rate
    target: 99.9  # 99.9%
    window: 28d
    error_budget_policy: "stop_deploys_at_50_percent"
    alerts:
      - severity: page
        threshold: 99.0  # page ถ้าลดลงกว่า 99%
      - severity: ticket
        threshold: 99.5  # สร้าง ticket ถ้าลดลงกว่า 99.5%

  - name: payment_latency
    sli: request_latency_p99
    target: 500  # 500ms
    window: 28d
    comparison: less_than
    alerts:
      - severity: page
        threshold: 2000  # page ถ้า > 2s
      - severity: ticket
        threshold: 1000  # ticket ถ้า > 1s

  - name: checkout_availability
    sli: request_success_rate
    target: 99.95  # checkout สำคัญมาก
    window: 28d
    routes:
      - /api/checkout
      - /api/payment
```

---

## 76.2 Error Budgets

### Error Budget คืออะไร?

```
Error Budget = 100% - SLO target

ตัวอย่าง:
SLO = 99.9% availability (28 วัน)
Error Budget = 0.1% ของ 28 วัน
             = 0.001 × 28 × 24 × 60 นาที
             = 40.32 นาที
```

### Error Budget Tracking

```python
# error-budget.py
from datetime import datetime, timedelta
from dataclasses import dataclass
from typing import Optional
import math

@dataclass
class SLODefinition:
    name: str
    target: float  # e.g., 99.9
    window_days: int

@dataclass
class ErrorBudgetStatus:
    slo_name: str
    target: float
    total_budget_minutes: float
    consumed_minutes: float
    remaining_minutes: float
    remaining_percentage: float
    burn_rate: float  # เท่าไหร่ที่ใช้ไปจากปกติ
    days_until_exhaustion: Optional[float]
    status: str  # healthy, at_risk, critical, exhausted


def calculate_error_budget(
    slo: SLODefinition,
    bad_events: float,
    total_events: float,
) -> ErrorBudgetStatus:
    """
    คำนวณ error budget status

    Args:
        slo: SLO definition
        bad_events: จำนวน events ที่ไม่ตรงตาม SLI
        total_events: จำนวน events ทั้งหมด
    """
    # คำนวณ total budget
    window_minutes = slo.window_days * 24 * 60
    error_rate = 1 - (slo.target / 100)
    total_budget_minutes = window_minutes * error_rate

    # คำนวณ consumed budget
    actual_error_rate = bad_events / total_events if total_events > 0 else 0
    consumed_error_rate = max(0, actual_error_rate - 0)  # errors beyond allowed
    consumed_minutes = window_minutes * actual_error_rate

    # Budget remaining
    remaining_minutes = max(0, total_budget_minutes - consumed_minutes)
    remaining_percentage = (remaining_minutes / total_budget_minutes) * 100 if total_budget_minutes > 0 else 0

    # Burn rate (1.0 = consuming at exactly allowed rate)
    burn_rate = actual_error_rate / error_rate if error_rate > 0 else 0

    # Days until exhaustion
    if burn_rate <= 1:
        days_until_exhaustion = None  # ไม่หมด
    else:
        excess_burn_per_day = (burn_rate - 1) * error_rate
        remaining_as_rate = remaining_percentage / 100 * error_rate
        if excess_burn_per_day > 0:
            days_until_exhaustion = remaining_as_rate / excess_burn_per_day
        else:
            days_until_exhaustion = None

    # Status
    if remaining_percentage <= 0:
        status = "exhausted"
    elif remaining_percentage <= 25:
        status = "critical"
    elif remaining_percentage <= 50:
        status = "at_risk"
    else:
        status = "healthy"

    return ErrorBudgetStatus(
        slo_name=slo.name,
        target=slo.target,
        total_budget_minutes=round(total_budget_minutes, 2),
        consumed_minutes=round(consumed_minutes, 2),
        remaining_minutes=round(remaining_minutes, 2),
        remaining_percentage=round(remaining_percentage, 2),
        burn_rate=round(burn_rate, 2),
        days_until_exhaustion=round(days_until_exhaustion, 1) if days_until_exhaustion else None,
        status=status,
    )


def print_error_budget_report(status: ErrorBudgetStatus) -> None:
    print(f"""
╔══════════════════════════════════════════════════╗
║  Error Budget Report: {status.slo_name:<25} ║
╠══════════════════════════════════════════════════╣
║  SLO Target:     {status.target}%                         ║
║  Total Budget:   {status.total_budget_minutes:>8.2f} minutes               ║
║  Consumed:       {status.consumed_minutes:>8.2f} minutes               ║
║  Remaining:      {status.remaining_minutes:>8.2f} minutes ({status.remaining_percentage:.1f}%)       ║
║  Burn Rate:      {status.burn_rate:>8.2f}x                            ║
║  Status:         {status.status:<12}                        ║
""")
    if status.days_until_exhaustion:
        print(f"║  🚨 Budget exhaustion in: {status.days_until_exhaustion} days           ║")
    print("╚══════════════════════════════════════════════════╝")
```

### Error Budget Policy

```yaml
# error-budget-policy.yaml
service: payment-service

policies:
  # ถ้าเหลือ budget < 50% → หยุด risky deployments
  - name: slow_burn_protection
    trigger:
      remaining_percentage: 50
    actions:
      - type: block_deployments
        except:
          - critical_fixes
          - security_patches
      - type: notify
        channels:
          - slack: "#sre-alerts"
          - email: "team-leads@mycompany.com"
    message: "Error budget ต่ำกว่า 50% - งดเว้น non-critical deployments"

  # ถ้าเหลือ budget < 25% → alert ทีม
  - name: critical_budget_alert
    trigger:
      remaining_percentage: 25
    actions:
      - type: page_oncall
        escalation: immediate
      - type: create_jira_ticket
        priority: P1
        title: "Error Budget Critical: {service}"
    message: "Error budget ต่ำกว่า 25% - ต้องการ immediate action"

  # ถ้า budget หมด → freeze deployments
  - name: budget_exhausted
    trigger:
      remaining_percentage: 0
    actions:
      - type: freeze_deployments
        except:
          - hotfixes
      - type: page_oncall
        escalation: immediate
      - type: create_incident
        severity: P1
    message: "Error budget หมด - frozen deployments"
```

### Prometheus Rules สำหรับ Error Budget

```yaml
# prometheus-slo-rules.yaml
groups:
  - name: slo.payment_service
    rules:
      # SLI: success rate
      - record: slo:payment_service:success_rate_5m
        expr: |
          sum(rate(http_requests_total{job="payment-service",status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total{job="payment-service"}[5m]))

      # SLO: 99.9% over 28 days
      - record: slo:payment_service:error_budget_remaining
        expr: |
          1 - (
            (1 - sum_over_time(slo:payment_service:success_rate_5m[28d]) / count_over_time(slo:payment_service:success_rate_5m[28d]))
            /
            (1 - 0.999)
          )

      # Multi-window burn rate alerts
      - alert: PaymentServiceErrorBudgetBurnRateHigh
        expr: |
          (
            slo:payment_service:burn_rate_1h > 14.4
            and
            slo:payment_service:burn_rate_5m > 14.4
          )
          or
          (
            slo:payment_service:burn_rate_6h > 6
            and
            slo:payment_service:burn_rate_30m > 6
          )
        labels:
          severity: critical
          team: payments
        annotations:
          summary: "High error budget burn rate for payment-service"
          description: |
            Error budget กำลังถูกใช้เร็วเกินไป
            Burn rate (1h): {{ $value }}x
            Budget remaining: {{ with query "slo:payment_service:error_budget_remaining" }}{{ . | first | value | humanizePercentage }}{{ end }}
```

---

## 76.3 Toil Reduction

### Toil คืออะไร?

Toil คืองาน manual, repetitive, ที่ไม่มี lasting value

```
Toil ตัวอย่าง:
- Manual deployment approvals ทุกครั้ง
- ตรวจ alerts ที่ fire ทุกวันแต่ไม่ใช่ปัญหาจริง
- Copy-paste configuration ระหว่าง environments
- Manual rollback procedures
- ตอบ question เดิมๆ ที่ควรมี documentation

Non-toil:
- สร้าง automation ที่ลด toil ในอนาคต
- ออกแบบระบบใหม่
- Mentoring
- Postmortem analysis
```

### วัดและลด Toil

```python
# toil-tracker.py
from dataclasses import dataclass
from datetime import datetime
from typing import List
import json

@dataclass
class ToilEntry:
    date: datetime
    category: str  # manual_deploy, alert_triage, config_copy, etc.
    duration_minutes: float
    description: str
    automatable: bool
    team_member: str

class ToilTracker:
    def __init__(self):
        self.entries: List[ToilEntry] = []

    def add_entry(self, entry: ToilEntry) -> None:
        self.entries.append(entry)

    def calculate_toil_percentage(
        self,
        total_work_hours: float,
        period_days: int = 7,
    ) -> float:
        """คำนวณ % ของเวลาที่ใช้กับ toil"""
        cutoff = datetime.now().timestamp() - (period_days * 86400)
        recent_entries = [
            e for e in self.entries
            if e.date.timestamp() >= cutoff
        ]

        total_toil_minutes = sum(e.duration_minutes for e in recent_entries)
        total_work_minutes = total_work_hours * 60

        return (total_toil_minutes / total_work_minutes) * 100

    def identify_top_toil(self, top_n: int = 5) -> List[dict]:
        """หา toil ที่กินเวลามากที่สุด"""
        category_totals = {}
        for entry in self.entries:
            cat = entry.category
            category_totals[cat] = category_totals.get(cat, 0) + entry.duration_minutes

        sorted_categories = sorted(
            category_totals.items(),
            key=lambda x: x[1],
            reverse=True,
        )

        return [
            {
                "category": cat,
                "total_minutes": mins,
                "total_hours": round(mins / 60, 1),
                "pct_of_total": round(mins / sum(category_totals.values()) * 100, 1),
            }
            for cat, mins in sorted_categories[:top_n]
        ]

    def generate_automation_backlog(self) -> List[dict]:
        """สร้าง backlog ของ toil ที่ automate ได้"""
        automatable = [
            e for e in self.entries
            if e.automatable
        ]

        # จัดกลุ่มและคำนวณ ROI
        category_data = {}
        for entry in automatable:
            cat = entry.category
            if cat not in category_data:
                category_data[cat] = {
                    "category": cat,
                    "total_minutes": 0,
                    "occurrences": 0,
                    "examples": [],
                }
            category_data[cat]["total_minutes"] += entry.duration_minutes
            category_data[cat]["occurrences"] += 1
            if len(category_data[cat]["examples"]) < 3:
                category_data[cat]["examples"].append(entry.description)

        backlog = []
        for cat, data in category_data.items():
            avg_minutes = data["total_minutes"] / data["occurrences"]
            # ROI: ถ้า automate ได้ จะประหยัดเวลา X นาที/เดือน
            monthly_savings = (data["occurrences"] / 4) * avg_minutes  # extrapolate to monthly
            backlog.append({
                **data,
                "avg_duration_minutes": round(avg_minutes, 1),
                "monthly_savings_hours": round(monthly_savings / 60, 1),
                "priority": "high" if monthly_savings > 300 else "medium" if monthly_savings > 60 else "low",
            })

        return sorted(backlog, key=lambda x: x["monthly_savings_hours"], reverse=True)
```

---

## 76.4 Reliability in Deployments

### Deployment Reliability Checklist

```yaml
# deployment-reliability-checklist.yaml
pre_deployment:
  - name: "SLO Check"
    check: "error_budget_remaining > 20%"
    action_if_fail: "block deployment"

  - name: "Test Coverage"
    check: "test_coverage >= 80%"
    action_if_fail: "warn and require approval"

  - name: "Security Scan"
    check: "no_critical_vulnerabilities"
    action_if_fail: "block deployment"

  - name: "Load Testing"
    check: "staging_load_test_passed"
    action_if_fail: "require approval"
    applies_to: "major_changes"

during_deployment:
  - name: "Canary Analysis"
    checks:
      - "canary_error_rate < baseline_error_rate * 1.5"
      - "canary_latency_p99 < baseline_latency_p99 * 1.5"
    interval: "5 minutes"
    auto_rollback: true

  - name: "Health Checks"
    check: "readiness_probe passes"
    required_replicas: "minimum 2"

post_deployment:
  - name: "Smoke Tests"
    check: "all_smoke_tests_pass"
    timeout: "5 minutes"

  - name: "SLO Monitoring"
    monitor: "15 minutes after deployment"
    auto_rollback_if: "slo_breach_detected"

  - name: "Synthetic Monitoring"
    check: "all_synthetic_tests_pass"
    frequency: "every 1 minute"
    duration: "30 minutes"
```

### Automated Rollback สำหรับ SLO Breach

```go
// sre/auto-rollback.go
package sre

import (
    "context"
    "fmt"
    "log"
    "time"

    "github.com/prometheus/client_golang/api"
    v1 "github.com/prometheus/client_golang/api/prometheus/v1"
    "github.com/prometheus/common/model"
)

type AutoRollbackConfig struct {
    Service         string
    Namespace       string
    SLOTarget       float64
    BurnRateWindow  time.Duration
    BurnRateLimit   float64
    CheckInterval   time.Duration
    RollbackEnabled bool
}

type AutoRollbackController struct {
    config     AutoRollbackConfig
    promClient v1.API
    k8sClient  KubernetesClient
    notifier   Notifier
}

func (c *AutoRollbackController) MonitorDeployment(
    ctx context.Context,
    deploymentID string,
    watchDuration time.Duration,
) error {
    startTime := time.Now()
    ticker := time.NewTicker(c.config.CheckInterval)
    defer ticker.Stop()

    log.Printf("Starting deployment monitoring for %s", deploymentID)

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-ticker.C:
            if time.Since(startTime) > watchDuration {
                log.Printf("Deployment %s monitoring completed - no issues", deploymentID)
                return nil
            }

            // ตรวจสอบ SLO
            burnRate, err := c.calculateBurnRate(ctx)
            if err != nil {
                log.Printf("Error calculating burn rate: %v", err)
                continue
            }

            log.Printf("Current burn rate: %.2fx", burnRate)

            if burnRate > c.config.BurnRateLimit {
                msg := fmt.Sprintf(
                    "🚨 SLO Breach Detected!\nService: %s\nDeployment: %s\nBurn Rate: %.2fx (limit: %.2fx)\nInitiating rollback...",
                    c.config.Service, deploymentID, burnRate, c.config.BurnRateLimit,
                )

                c.notifier.SendAlert(msg)

                if c.config.RollbackEnabled {
                    if err := c.initiateRollback(ctx, deploymentID); err != nil {
                        return fmt.Errorf("rollback failed: %w", err)
                    }
                    return fmt.Errorf("deployment rolled back due to SLO breach")
                }
            }
        }
    }
}

func (c *AutoRollbackController) calculateBurnRate(ctx context.Context) (float64, error) {
    query := fmt.Sprintf(`
        sum(rate(http_requests_total{job="%s",status=~"5.."}[%s]))
        /
        sum(rate(http_requests_total{job="%s"}[%s]))
        /
        (1 - %f)
    `,
        c.config.Service,
        c.config.BurnRateWindow,
        c.config.Service,
        c.config.BurnRateWindow,
        c.config.SLOTarget/100,
    )

    result, _, err := c.promClient.Query(ctx, query, time.Now())
    if err != nil {
        return 0, err
    }

    vector, ok := result.(model.Vector)
    if !ok || len(vector) == 0 {
        return 0, nil
    }

    return float64(vector[0].Value), nil
}

func (c *AutoRollbackController) initiateRollback(
    ctx context.Context,
    deploymentID string,
) error {
    log.Printf("Initiating rollback for deployment %s", deploymentID)

    // ใช้ kubectl rollout undo
    err := c.k8sClient.RolloutUndo(ctx, c.config.Namespace, c.config.Service)
    if err != nil {
        return fmt.Errorf("kubectl rollout undo failed: %w", err)
    }

    // รอให้ rollback เสร็จ
    err = c.k8sClient.WaitForRollout(ctx, c.config.Namespace, c.config.Service, 5*time.Minute)
    if err != nil {
        return fmt.Errorf("waiting for rollback failed: %w", err)
    }

    log.Printf("Rollback completed for %s", deploymentID)
    return nil
}
```

---

## 76.5 Progressive Rollouts

### Argo Rollouts Configuration

```yaml
# argo-rollout-with-slo.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 20
  selector:
    matchLabels:
      app: payment-service

  strategy:
    canary:
      canaryService: payment-service-canary
      stableService: payment-service-stable

      # Traffic shaping
      trafficRouting:
        nginx:
          stableIngress: payment-service-stable
          additionalIngressAnnotations:
            canary-by-header: X-Canary

      # Analysis template
      analysis:
        templates:
          - templateName: payment-success-rate
          - templateName: payment-latency
        startingStep: 2
        args:
          - name: service-name
            value: payment-service
          - name: namespace
            value: production

      steps:
        - setWeight: 5
        - pause:
            duration: 5m
        - setWeight: 10
        - pause:
            duration: 10m
        - setWeight: 25
        - pause:
            duration: 15m
        - setWeight: 50
        - pause:
            duration: 30m
        - setWeight: 100
---
# Analysis Template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-success-rate
  namespace: production
spec:
  args:
    - name: service-name
    - name: namespace
  metrics:
    - name: success-rate
      interval: 1m
      count: 5
      successCondition: result[0] >= 0.999
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{
              job="{{ args.service-name }}",
              namespace="{{ args.namespace }}",
              status!~"5.."
            }[5m]))
            /
            sum(rate(http_requests_total{
              job="{{ args.service-name }}",
              namespace="{{ args.namespace }}"
            }[5m]))

    - name: latency-p99
      interval: 1m
      count: 5
      successCondition: result[0] <= 0.5  # 500ms
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            histogram_quantile(0.99,
              sum by (le) (
                rate(http_request_duration_seconds_bucket{
                  job="{{ args.service-name }}",
                  namespace="{{ args.namespace }}"
                }[5m])
              )
            )
---
# Analysis Template สำหรับ Latency
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-latency
  namespace: production
spec:
  args:
    - name: service-name
    - name: namespace
  metrics:
    - name: compare-canary-vs-stable
      interval: 2m
      count: 5
      # canary latency ไม่ควรสูงกว่า stable เกิน 20%
      successCondition: result[0] <= 1.2
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            (
              histogram_quantile(0.95, sum by (le) (
                rate(http_request_duration_seconds_bucket{
                  job="{{ args.service-name }}-canary"
                }[5m])
              ))
            )
            /
            (
              histogram_quantile(0.95, sum by (le) (
                rate(http_request_duration_seconds_bucket{
                  job="{{ args.service-name }}-stable"
                }[5m])
              ))
            )
```

### Feature Flags สำหรับ Progressive Rollout

```typescript
// progressive-rollout/feature-flags.ts
import { OpenFeature, ProviderEvents } from '@openfeature/server-sdk';
import { LaunchDarklyProvider } from '@launchdarkly/openfeature-node-server';

// Setup
await OpenFeature.setProviderAndWait(
  new LaunchDarklyProvider(process.env.LAUNCHDARKLY_SDK_KEY!)
);

const client = OpenFeature.getClient('payment-service');

// ใช้ feature flag ใน payment flow
export async function processPayment(
  userId: string,
  amount: number,
  paymentMethod: string,
): Promise<PaymentResult> {
  const ctx = {
    targetingKey: userId,
    userId,
    email: await getUserEmail(userId),
    plan: await getUserPlan(userId),
  };

  // ใช้ new payment engine สำหรับบาง users
  const useNewPaymentEngine = await client.getBooleanValue(
    'use-new-payment-engine',
    false,
    ctx,
  );

  if (useNewPaymentEngine) {
    return processPaymentV2(userId, amount, paymentMethod);
  } else {
    return processPaymentV1(userId, amount, paymentMethod);
  }
}

// Gradual rollout configuration (LaunchDarkly)
const rolloutConfig = {
  flagKey: 'use-new-payment-engine',
  variations: [
    { value: false, name: 'Old Engine', description: 'Use payment engine v1' },
    { value: true, name: 'New Engine', description: 'Use payment engine v2' },
  ],
  defaultVariation: 0,  // false (old engine)
  rules: [
    // ทดสอบกับ beta users ก่อน
    {
      clauses: [{ attribute: 'plan', op: 'in', values: ['beta', 'internal'] }],
      variation: 1,  // new engine
    },
    // Percentage rollout
    {
      rollout: {
        variations: [
          { variation: 0, weight: 90000 },  // 90% old engine
          { variation: 1, weight: 10000 },  // 10% new engine
        ],
      },
    },
  ],
};
```

---

## 76.6 SRE ใน CI/CD Pipeline

### SRE Gate ใน Pipeline

```yaml
# .github/workflows/sre-gated-deploy.yaml
name: SRE-Gated Deployment

on:
  push:
    branches: [main]

jobs:
  pre-deploy-checks:
    name: Pre-deployment SRE Checks
    runs-on: ubuntu-latest
    outputs:
      deploy-approved: ${{ steps.sre-gate.outputs.approved }}
      budget-remaining: ${{ steps.error-budget.outputs.remaining }}

    steps:
      - name: Check Error Budget
        id: error-budget
        run: |
          # ดึง error budget remaining จาก Prometheus
          BUDGET=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode 'query=slo:payment_service:error_budget_remaining * 100' \
            | jq -r '.data.result[0].value[1]')

          echo "remaining=$BUDGET" >> $GITHUB_OUTPUT
          echo "Error Budget Remaining: ${BUDGET}%"

          # ถ้า budget < 20% → block
          if (( $(echo "$BUDGET < 20" | bc -l) )); then
            echo "⛔ Error budget ต่ำกว่า 20% (${BUDGET}%) - blocking deployment"
            exit 1
          fi

      - name: Check SLO Status
        id: slo-check
        run: |
          # ตรวจสอบว่า SLO ยังเป็นปกติอยู่
          SUCCESS_RATE=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode 'query=slo:payment_service:success_rate_5m * 100' \
            | jq -r '.data.result[0].value[1]')

          echo "Current success rate: ${SUCCESS_RATE}%"

          if (( $(echo "$SUCCESS_RATE < 99" | bc -l) )); then
            echo "⛔ Success rate ต่ำกว่า 99% - blocking deployment"
            exit 1
          fi

      - name: SRE Approval Gate
        id: sre-gate
        run: |
          BUDGET=${{ steps.error-budget.outputs.remaining }}

          if (( $(echo "$BUDGET < 50" | bc -l) )); then
            echo "⚠️ Error budget ต่ำกว่า 50% (${BUDGET}%) - ต้องการ SRE approval"
            echo "approved=needs_approval" >> $GITHUB_OUTPUT
          else
            echo "✅ Error budget OK (${BUDGET}%) - deployment approved"
            echo "approved=true" >> $GITHUB_OUTPUT
          fi

  sre-approval:
    name: SRE Approval (required when budget < 50%)
    runs-on: ubuntu-latest
    needs: pre-deploy-checks
    if: needs.pre-deploy-checks.outputs.deploy-approved == 'needs_approval'
    environment: sre-approval  # ต้องการ manual approval

    steps:
      - name: Request SRE Approval
        run: |
          echo "Deployment requires SRE approval"
          echo "Error Budget Remaining: ${{ needs.pre-deploy-checks.outputs.budget-remaining }}%"
          echo "Please review and approve in GitHub Environments"

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs:
      - pre-deploy-checks
      - sre-approval
    if: |
      always() &&
      (needs.pre-deploy-checks.outputs.deploy-approved == 'true' ||
       needs.sre-approval.result == 'success')

    steps:
      - name: Deploy with Progressive Rollout
        run: |
          # อัพเดท Argo Rollout
          kubectl argo rollouts set image payment-service \
            payment-service=ghcr.io/mycompany/payment-service:${{ github.sha }}

          # Monitor rollout
          kubectl argo rollouts status payment-service --watch --timeout 20m

  post-deploy-monitoring:
    name: Post-deployment Monitoring
    runs-on: ubuntu-latest
    needs: deploy
    if: success()

    steps:
      - name: Monitor SLOs for 30 minutes
        run: |
          START_TIME=$(date +%s)
          MONITOR_DURATION=1800  # 30 minutes

          while [ $(($(date +%s) - START_TIME)) -lt $MONITOR_DURATION ]; do
            SUCCESS_RATE=$(curl -s "http://prometheus:9090/api/v1/query" \
              --data-urlencode 'query=slo:payment_service:success_rate_5m * 100' \
              | jq -r '.data.result[0].value[1]')

            echo "$(date): Success Rate = ${SUCCESS_RATE}%"

            if (( $(echo "$SUCCESS_RATE < 99.5" | bc -l) )); then
              echo "⛔ Success rate ลดลง (${SUCCESS_RATE}%) - initiating rollback"
              kubectl argo rollouts abort payment-service
              kubectl argo rollouts undo payment-service
              exit 1
            fi

            sleep 60
          done

          echo "✅ SLO monitoring completed - deployment successful"
```

---

## 76.7 SLO Dashboard

```yaml
# grafana-slo-dashboard.json (structure)
{
  "title": "SRE SLO Dashboard",
  "panels": [
    {
      "title": "Payment Service - Error Budget",
      "type": "gauge",
      "targets": [{
        "expr": "slo:payment_service:error_budget_remaining * 100"
      }],
      "options": {
        "thresholds": [
          {"color": "red", "value": 0},
          {"color": "yellow", "value": 25},
          {"color": "green", "value": 50}
        ]
      }
    },
    {
      "title": "Availability SLO (28-day window)",
      "type": "stat",
      "targets": [{
        "expr": "avg_over_time(slo:payment_service:success_rate_5m[28d]) * 100"
      }],
      "thresholds": {
        "steps": [
          {"color": "red", "value": 0},
          {"color": "yellow", "value": 99},
          {"color": "green", "value": 99.9}
        ]
      }
    },
    {
      "title": "Burn Rate (Multi-window)",
      "type": "timeseries",
      "targets": [
        {
          "expr": "slo:payment_service:burn_rate_1h",
          "legendFormat": "1h burn rate"
        },
        {
          "expr": "slo:payment_service:burn_rate_6h",
          "legendFormat": "6h burn rate"
        }
      ],
      "thresholds": {
        "steps": [
          {"color": "green", "value": 0},
          {"color": "yellow", "value": 1},
          {"color": "red", "value": 14.4}
        ]
      }
    }
  ]
}
```

---

## 76.8 แบบฝึกหัด

### แบบฝึกหัดที่ 1: กำหนด SLOs

```yaml
# exercises/define-slos.yaml
# กำหนด SLOs สำหรับ service ของคุณ

service_name: "your-service-name"
team: "your-team"

# TODO: กำหนด SLIs และ SLOs
# ต้องมีอย่างน้อย:
# 1. Availability SLO
# 2. Latency SLO (optional แต่แนะนำ)

slos:
  - name: availability
    sli:
      type: availability
      formula: "TODO: เขียน PromQL query"
    target: "TODO: กำหนด % target"
    window: 28d
    rationale: "TODO: ทำไมถึงเลือก target นี้"

  - name: latency_p99
    sli:
      type: latency
      percentile: 99
      formula: "TODO: เขียน PromQL query"
    target: "TODO: กำหนด ms target"
    window: 28d
    rationale: "TODO: ทำไมถึงเลือก target นี้"

error_budget_policy:
  at_50_percent: "TODO: จะทำอะไร?"
  at_25_percent: "TODO: จะทำอะไร?"
  at_0_percent: "TODO: จะทำอะไร?"
```

### แบบฝึกหัดที่ 2: คำนวณ Error Budget

```python
# exercises/calculate-error-budget.py

from datetime import datetime

# SLO สำหรับ service ของคุณ
slo = SLODefinition(
    name="my_service_availability",
    target=99.9,  # 99.9%
    window_days=28,
)

# ข้อมูล metrics จาก Prometheus (ตัวอย่าง)
# TODO: เปลี่ยนเป็นข้อมูลจริงจาก Prometheus ของคุณ
bad_events = 450    # requests ที่ fail
total_events = 500000  # requests ทั้งหมด

# คำนวณ error budget
status = calculate_error_budget(slo, bad_events, total_events)

# แสดงผล
print_error_budget_report(status)

# คำถาม:
# 1. Error budget เหลือกี่ %?
# 2. Burn rate เป็นเท่าไหร่?
# 3. ถ้า burn rate ยังเท่านี้ budget จะหมดในกี่วัน?
# 4. ทีมควรทำอะไร?
```

### แบบฝึกหัดที่ 3: สร้าง Auto-rollback Rule

```yaml
# exercises/auto-rollback-config.yaml
# ออกแบบ auto-rollback policy

rollback_triggers:
  # TODO: กำหนด metric thresholds ที่จะ trigger rollback
  metrics:
    - name: "TODO: ชื่อ metric"
      query: "TODO: PromQL"
      threshold: "TODO: ค่า threshold"
      comparison: "TODO: greater_than หรือ less_than"
      window: "TODO: time window"

  # ตัวอย่าง:
  # - name: error_rate
  #   query: "sum(rate(errors_total[5m])) / sum(rate(requests_total[5m]))"
  #   threshold: 0.05
  #   comparison: greater_than
  #   window: 5m

actions:
  on_trigger:
    - "TODO: ขั้นตอนที่ 1"
    - "TODO: ขั้นตอนที่ 2"
    - "TODO: ขั้นตอนที่ 3"

  notifications:
    - channel: "TODO"
      message: "TODO"

testing:
  # TODO: ทำ chaos test เพื่อทดสอบ auto-rollback
  # 1. Deploy service version ใหม่
  # 2. Inject errors (ใช้ chaos engineering tool)
  # 3. ตรวจสอบว่า auto-rollback ทำงาน
  steps:
    - "TODO"
```

---

## สรุป

SRE practices ช่วยให้ CI/CD pipeline มีความน่าเชื่อถือมากขึ้น:

1. **SLIs/SLOs/SLAs** กำหนดเป้าหมายที่ชัดเจนและวัดได้
2. **Error Budgets** สร้าง balance ระหว่าง reliability และ velocity
3. **Toil Reduction** เพิ่มเวลาสำหรับงานที่มีคุณค่า
4. **Progressive Rollouts** ลดความเสี่ยงของ deployment
5. **Auto-rollback** ลด MTTR เมื่อเกิดปัญหา

องค์กรที่ implement SRE practices ดีจะมี deployment frequency สูงขึ้น ในขณะที่ reliability ก็ดีขึ้นด้วย

### ขั้นตอนถัดไป

- ศึกษา [Part 77: Incident Response Automation](./part-77-incident-response.md)
- Google SRE Book: https://sre.google/sre-book/table-of-contents/
- OpenSLO: https://openslo.com

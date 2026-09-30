# Part 78: Change Management & Rollback Strategy

## บทนำ

Change Management คือกระบวนการควบคุมและจัดการการเปลี่ยนแปลงในระบบ IT เพื่อลดความเสี่ยงและให้มั่นใจว่า business continuity จะไม่ถูกกระทบ

ในบทนี้เราจะเรียนรู้:
- Change Advisory Board (CAB)
- Automated Change Classification
- Rollback Triggers
- Database Rollback
- แบบฝึกหัดปฏิบัติ

---

## 78.1 Change Management Framework

### ประเภทของ Changes

```
1. Standard Changes
   - Low risk, pre-approved
   - ขั้นตอนชัดเจน documented ไว้แล้ว
   - ตัวอย่าง: routine security patches, config updates

2. Normal Changes
   - Medium/high risk
   - ต้องผ่าน CAB approval
   - ตัวอย่าง: new feature releases, infrastructure changes

3. Emergency Changes
   - สำหรับ critical issues ที่ต้องแก้ด่วน
   - ผ่าน expedited process
   - ตัวอย่าง: security vulnerabilities, production outages
```

### Change Risk Matrix

```
┌─────────────────────────────────────────────────────────────┐
│                    Change Risk Matrix                        │
│                                                             │
│         Low Impact    │ Medium Impact │   High Impact       │
│    ────────────────────┼───────────────┼──────────────────  │
│ Low    │  Standard     │  Normal       │  Normal            │
│ Prob   │  (Pre-approve)│  (CAB light)  │  (Full CAB)        │
│    ────────────────────┼───────────────┼──────────────────  │
│ Medium │  Normal       │  Normal       │  Emergency/High     │
│ Prob   │  (CAB light)  │  (Full CAB)   │  (Executive sign)  │
│    ────────────────────┼───────────────┼──────────────────  │
│ High   │  Normal       │  Emergency    │  Emergency/Major    │
│ Prob   │  (Full CAB)   │  (Executive)  │  (Board approval)  │
└─────────────────────────────────────────────────────────────┘
```

---

## 78.2 Automated Change Classification

### ML-based Change Risk Scoring

```python
# change-classifier.py
from dataclasses import dataclass
from typing import List, Optional
import re

@dataclass
class ChangeRequest:
    title: str
    description: str
    services_affected: List[str]
    files_changed: List[str]
    lines_added: int
    lines_deleted: int
    author: str
    team: str
    deployment_time: Optional[str]  # "business_hours", "off_hours", "weekend"
    has_rollback_plan: bool
    has_test_evidence: bool
    environment: str  # "dev", "staging", "prod"


@dataclass
class RiskScore:
    score: float  # 0-100
    category: str  # "low", "medium", "high", "critical"
    factors: List[str]
    recommended_approval: str
    automated_checks_required: List[str]


class ChangeRiskClassifier:
    """Rule-based change risk classifier"""

    # Patterns ที่บ่งบอกความเสี่ยงสูง
    HIGH_RISK_PATTERNS = [
        r'database.*migration',
        r'schema.*change',
        r'authentication',
        r'authorization',
        r'payment',
        r'billing',
        r'security',
        r'encryption',
        r'credentials',
        r'infrastructure',
    ]

    # Services ที่มีความสำคัญสูง
    CRITICAL_SERVICES = {
        'payment-service': 1.5,
        'auth-service': 1.5,
        'user-service': 1.3,
        'order-service': 1.3,
        'api-gateway': 1.4,
        'database': 2.0,
    }

    def classify(self, change: ChangeRequest) -> RiskScore:
        score = 0.0
        factors = []

        # 1. Size of change
        total_lines = change.lines_added + change.lines_deleted
        if total_lines > 1000:
            score += 30
            factors.append(f"Large change: {total_lines} lines modified")
        elif total_lines > 500:
            score += 15
            factors.append(f"Medium change: {total_lines} lines modified")
        elif total_lines > 100:
            score += 5

        # 2. Number of services affected
        if len(change.services_affected) > 3:
            score += 25
            factors.append(f"Many services affected: {len(change.services_affected)}")
        elif len(change.services_affected) > 1:
            score += 10
            factors.append(f"Multiple services: {change.services_affected}")

        # 3. Critical service multiplier
        for service in change.services_affected:
            multiplier = self.CRITICAL_SERVICES.get(service, 1.0)
            if multiplier > 1.0:
                score *= multiplier
                factors.append(f"Critical service: {service} (x{multiplier})")

        # 4. High-risk patterns
        text_to_check = f"{change.title} {change.description}".lower()
        for pattern in self.HIGH_RISK_PATTERNS:
            if re.search(pattern, text_to_check):
                score += 20
                factors.append(f"High-risk pattern: {pattern}")
                break  # Count only once

        # 5. Database files changed
        db_files = [f for f in change.files_changed if 'migration' in f or '.sql' in f]
        if db_files:
            score += 30
            factors.append(f"Database changes: {db_files}")

        # 6. Deployment timing
        if change.deployment_time == 'off_hours' and change.environment == 'prod':
            score += 15
            factors.append("Off-hours production deployment")
        elif change.deployment_time == 'weekend' and change.environment == 'prod':
            score += 25
            factors.append("Weekend production deployment")

        # 7. Safety measures
        if not change.has_rollback_plan:
            score += 20
            factors.append("No rollback plan")
        if not change.has_test_evidence:
            score += 15
            factors.append("No test evidence")

        # Cap at 100
        score = min(100, score)

        # Categorize
        if score >= 75:
            category = "critical"
            approval = "full_cab"
        elif score >= 50:
            category = "high"
            approval = "cab_light"
        elif score >= 25:
            category = "medium"
            approval = "team_lead"
        else:
            category = "low"
            approval = "auto_approved"

        # Required checks
        checks = self.get_required_checks(score, change)

        return RiskScore(
            score=round(score, 1),
            category=category,
            factors=factors,
            recommended_approval=approval,
            automated_checks_required=checks,
        )

    def get_required_checks(self, score: float, change: ChangeRequest) -> List[str]:
        checks = ["unit_tests", "lint"]

        if score >= 25:
            checks.extend(["integration_tests", "security_scan"])

        if score >= 50:
            checks.extend(["load_test", "chaos_test_staging"])

        if any('migration' in f or '.sql' in f for f in change.files_changed):
            checks.extend(["db_migration_dry_run", "db_backup_verified"])

        if "payment" in ' '.join(change.services_affected):
            checks.append("payment_smoke_test")

        return list(set(checks))
```

### Change Management ใน GitHub Actions

```yaml
# .github/workflows/change-management.yaml
name: Change Management Gate

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

jobs:
  classify-change:
    name: Classify Change Risk
    runs-on: ubuntu-latest
    outputs:
      risk-score: ${{ steps.classify.outputs.score }}
      risk-category: ${{ steps.classify.outputs.category }}
      approval-required: ${{ steps.classify.outputs.approval }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed files
        id: changed-files
        run: |
          CHANGED=$(git diff --name-only origin/main...HEAD | tr '\n' ',')
          echo "files=$CHANGED" >> $GITHUB_OUTPUT

          LINES_ADDED=$(git diff --numstat origin/main...HEAD | awk '{sum += $1} END {print sum}')
          LINES_DELETED=$(git diff --numstat origin/main...HEAD | awk '{sum += $2} END {print sum}')
          echo "added=$LINES_ADDED" >> $GITHUB_OUTPUT
          echo "deleted=$LINES_DELETED" >> $GITHUB_OUTPUT

      - name: Classify change risk
        id: classify
        run: |
          python3 << 'EOF'
          import sys
          sys.path.append('.')
          from change_classifier import ChangeRiskClassifier, ChangeRequest

          change = ChangeRequest(
              title="${{ github.event.pull_request.title }}",
              description="${{ github.event.pull_request.body }}",
              services_affected="${{ steps.changed-files.outputs.files }}".split(','),
              files_changed="${{ steps.changed-files.outputs.files }}".split(','),
              lines_added=int("${{ steps.changed-files.outputs.added }}" or 0),
              lines_deleted=int("${{ steps.changed-files.outputs.deleted }}" or 0),
              author="${{ github.actor }}",
              team="engineering",
              deployment_time="business_hours",
              has_rollback_plan="rollback" in "${{ github.event.pull_request.body }}".lower(),
              has_test_evidence=True,
              environment="production",
          )

          classifier = ChangeRiskClassifier()
          risk = classifier.classify(change)

          print(f"Risk Score: {risk.score}")
          print(f"Category: {risk.category}")

          import os
          with open(os.environ['GITHUB_OUTPUT'], 'a') as f:
              f.write(f"score={risk.score}\n")
              f.write(f"category={risk.category}\n")
              f.write(f"approval={risk.recommended_approval}\n")
              f.write(f"factors={'|'.join(risk.factors)}\n")
          EOF

      - name: Post risk assessment comment
        uses: actions/github-script@v7
        with:
          script: |
            const score = '${{ steps.classify.outputs.risk-score }}';
            const category = '${{ steps.classify.outputs.risk-category }}';
            const approval = '${{ steps.classify.outputs.approval-required }}';

            const emoji = {
              'low': '🟢',
              'medium': '🟡',
              'high': '🟠',
              'critical': '🔴',
            }[category] || '⚪';

            const body = `## Change Risk Assessment ${emoji}

            | Metric | Value |
            |--------|-------|
            | Risk Score | **${score}/100** |
            | Category | **${category.toUpperCase()}** |
            | Approval Required | **${approval}** |

            ${category === 'low' ? '✅ Auto-approved: No manual approval needed' : ''}
            ${category === 'medium' ? '⚠️ Team lead approval required' : ''}
            ${category === 'high' ? '🔔 CAB review required' : ''}
            ${category === 'critical' ? '🚨 Full CAB approval + executive sign-off required' : ''}
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body,
            });

  # Block deployment ถ้า high/critical risk
  cab-approval:
    name: CAB Approval (High/Critical)
    runs-on: ubuntu-latest
    needs: classify-change
    if: |
      needs.classify-change.outputs.risk-category == 'high' ||
      needs.classify-change.outputs.risk-category == 'critical'
    environment: cab-approval  # ต้องการ manual approval

    steps:
      - name: CAB Review Required
        run: |
          echo "This change requires CAB approval"
          echo "Risk Score: ${{ needs.classify-change.outputs.risk-score }}"
          echo "Please review and approve in GitHub Environments"
```

---

## 78.3 Database Rollback Strategy

### Database Migration with Flyway

```sql
-- db/migrations/V20250901__add_payment_method.sql
-- Standard migration (forward only)

ALTER TABLE payments ADD COLUMN payment_method_id INTEGER;
ALTER TABLE payments ADD COLUMN payment_gateway VARCHAR(50) DEFAULT 'stripe';
CREATE INDEX idx_payments_payment_method_id ON payments(payment_method_id);
```

```sql
-- db/migrations/U20250901__add_payment_method.sql
-- Undo migration (backward)

DROP INDEX IF EXISTS idx_payments_payment_method_id;
ALTER TABLE payments DROP COLUMN IF EXISTS payment_method_id;
ALTER TABLE payments DROP COLUMN IF EXISTS payment_gateway;
```

### Expand-Contract Pattern

```python
# database/expand_contract.py
"""
Expand-Contract Pattern สำหรับ zero-downtime schema changes

ขั้นตอน:
Phase 1 (Expand): เพิ่ม column ใหม่ (ไม่ลบอันเก่า)
Phase 2 (Migrate): ย้ายข้อมูลจาก old column ไป new column
Phase 3 (Contract): ลบ old column

ทำให้สามารถ rollback ได้ทุก phase
"""

# Phase 1: Expand - เพิ่ม column ใหม่
EXPAND_SQL = """
-- เพิ่ม column ใหม่ (nullable, ไม่ NOT NULL ก่อน)
ALTER TABLE orders ADD COLUMN customer_id_v2 UUID;

-- เพิ่ม index สำหรับ column ใหม่
CREATE INDEX CONCURRENTLY idx_orders_customer_id_v2 ON orders(customer_id_v2);

-- Trigger เพื่อ sync ข้อมูลระหว่าง column เก่าและใหม่
CREATE OR REPLACE FUNCTION sync_customer_id()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.customer_id_v2 IS NULL AND NEW.customer_id IS NOT NULL THEN
    NEW.customer_id_v2 = uuid(NEW.customer_id::text);
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_customer_id_trigger
  BEFORE INSERT OR UPDATE ON orders
  FOR EACH ROW
  EXECUTE FUNCTION sync_customer_id();
"""

# Phase 2: Migrate - ย้ายข้อมูลเป็น batch
MIGRATE_SQL = """
DO $$
DECLARE
  batch_size INT := 1000;
  offset_val INT := 0;
  rows_updated INT;
BEGIN
  LOOP
    UPDATE orders
    SET customer_id_v2 = uuid(customer_id::text)
    WHERE
      customer_id_v2 IS NULL
      AND customer_id IS NOT NULL
    LIMIT batch_size;

    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    EXIT WHEN rows_updated = 0;

    PERFORM pg_sleep(0.1);  -- หน่วงเพื่อไม่ lock table นาน
  END LOOP;
END;
$$;
"""

# Phase 3: Contract - ลบ column เก่า (ทำหลังจาก code ไม่ใช้แล้ว)
CONTRACT_SQL = """
-- ลบ trigger
DROP TRIGGER IF EXISTS sync_customer_id_trigger ON orders;
DROP FUNCTION IF EXISTS sync_customer_id();

-- ลบ column เก่า
ALTER TABLE orders DROP COLUMN customer_id;

-- เพิ่ม NOT NULL constraint บน column ใหม่
ALTER TABLE orders ALTER COLUMN customer_id_v2 SET NOT NULL;

-- Rename column ใหม่
ALTER TABLE orders RENAME COLUMN customer_id_v2 TO customer_id;
"""
```

### Automated Database Backup Before Migration

```bash
#!/bin/bash
# db/pre-migration-backup.sh

set -euo pipefail

DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-mydb}"
DB_USER="${DB_USER:-postgres}"
BACKUP_BUCKET="${BACKUP_BUCKET:-mycompany-db-backups}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="pre_migration_${DB_NAME}_${TIMESTAMP}.sql.gz"

echo "🔒 Creating pre-migration backup..."
echo "Database: $DB_NAME @ $DB_HOST:$DB_PORT"
echo "Backup: s3://$BACKUP_BUCKET/$BACKUP_FILE"

# สร้าง backup
PGPASSWORD="$DB_PASSWORD" pg_dump \
  -h "$DB_HOST" \
  -p "$DB_PORT" \
  -U "$DB_USER" \
  -d "$DB_NAME" \
  --format=custom \
  --compress=9 \
  --file=/tmp/"${BACKUP_FILE%.gz}" \
  --verbose

# Compress
gzip /tmp/"${BACKUP_FILE%.gz}"

# Upload ไปยัง S3
aws s3 cp "/tmp/$BACKUP_FILE" "s3://$BACKUP_BUCKET/$BACKUP_FILE" \
  --storage-class STANDARD_IA \
  --metadata "migration_id=$MIGRATION_ID,environment=$ENVIRONMENT"

# ตรวจสอบขนาด backup
BACKUP_SIZE=$(aws s3 ls "s3://$BACKUP_BUCKET/$BACKUP_FILE" | awk '{print $3}')
echo "✅ Backup created: ${BACKUP_SIZE} bytes"

# บันทึก backup location สำหรับ rollback
echo "BACKUP_S3_PATH=s3://$BACKUP_BUCKET/$BACKUP_FILE" >> "$GITHUB_ENV"

# Cleanup local file
rm "/tmp/$BACKUP_FILE"
```

### Database Rollback Script

```bash
#!/bin/bash
# db/rollback-database.sh

set -euo pipefail

BACKUP_S3_PATH="${1:-$BACKUP_S3_PATH}"
DB_HOST="${DB_HOST}"
DB_NAME="${DB_NAME}"
DB_USER="${DB_USER}"

if [ -z "$BACKUP_S3_PATH" ]; then
  echo "❌ Error: BACKUP_S3_PATH is required"
  exit 1
fi

echo "⚠️  DATABASE ROLLBACK INITIATED"
echo "Restoring from: $BACKUP_S3_PATH"
echo ""
read -p "⛔ This will OVERWRITE the current database! Type 'CONFIRM' to proceed: " confirm

if [ "$confirm" != "CONFIRM" ]; then
  echo "Rollback cancelled"
  exit 0
fi

# Download backup
LOCAL_BACKUP="/tmp/rollback_$(date +%s).sql.gz"
echo "Downloading backup..."
aws s3 cp "$BACKUP_S3_PATH" "$LOCAL_BACKUP"

# Decompress
gunzip "$LOCAL_BACKUP"
LOCAL_SQL="${LOCAL_BACKUP%.gz}"

# Stop application (ถ้าเป็น Kubernetes)
echo "Scaling down application..."
kubectl scale deployment --all -n "$NAMESPACE" --replicas=0

# Restore database
echo "Restoring database..."
PGPASSWORD="$DB_PASSWORD" pg_restore \
  -h "$DB_HOST" \
  -U "$DB_USER" \
  -d "$DB_NAME" \
  --clean \
  --no-owner \
  --jobs=4 \
  --verbose \
  "$LOCAL_SQL"

# Restart application (previous version)
echo "Restarting application with previous image..."
kubectl rollout undo deployment/"$APP_NAME" -n "$NAMESPACE"
kubectl scale deployment/"$APP_NAME" -n "$NAMESPACE" --replicas=3

# Wait for ready
kubectl rollout status deployment/"$APP_NAME" -n "$NAMESPACE" --timeout=300s

# Cleanup
rm "$LOCAL_SQL"

echo "✅ Database rollback completed"
```

---

## 78.4 Rollback Triggers

### Automated Rollback Decision

```python
# rollback/trigger.py
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime, timedelta
import requests

@dataclass
class RollbackTrigger:
    name: str
    prometheus_query: str
    threshold: float
    comparison: str  # "greater_than" or "less_than"
    window: str  # e.g., "5m"
    consecutive_failures: int  # ต้อง fail กี่ครั้งติดกัน


@dataclass
class RollbackDecision:
    should_rollback: bool
    triggered_by: Optional[str]
    current_value: Optional[float]
    threshold: Optional[float]
    reason: str


class RollbackController:
    DEFAULT_TRIGGERS = [
        RollbackTrigger(
            name="error_rate",
            prometheus_query="""
                sum(rate(http_requests_total{{job="{service}",status=~"5.."}}[5m]))
                /
                sum(rate(http_requests_total{{job="{service}"}}[5m]))
            """,
            threshold=0.05,  # 5% error rate
            comparison="greater_than",
            window="5m",
            consecutive_failures=3,
        ),
        RollbackTrigger(
            name="latency_p99",
            prometheus_query="""
                histogram_quantile(0.99,
                  sum by (le) (
                    rate(http_request_duration_seconds_bucket{{job="{service}"}}[5m])
                  )
                )
            """,
            threshold=2.0,  # 2 seconds
            comparison="greater_than",
            window="5m",
            consecutive_failures=3,
        ),
        RollbackTrigger(
            name="success_rate",
            prometheus_query="""
                sum(rate(http_requests_total{{job="{service}",status!~"5.."}}[5m]))
                /
                sum(rate(http_requests_total{{job="{service}"}}[5m]))
            """,
            threshold=0.95,  # Below 95% success rate
            comparison="less_than",
            window="5m",
            consecutive_failures=2,
        ),
    ]

    def __init__(
        self,
        prometheus_url: str,
        service: str,
        triggers: Optional[List[RollbackTrigger]] = None,
    ):
        self.prometheus_url = prometheus_url
        self.service = service
        self.triggers = triggers or self.DEFAULT_TRIGGERS
        self.failure_counts = {}

    def check_should_rollback(self) -> RollbackDecision:
        """ตรวจสอบว่าควร rollback หรือไม่"""

        for trigger in self.triggers:
            query = trigger.prometheus_query.format(service=self.service)

            try:
                value = self._query_prometheus(query)
            except Exception as e:
                print(f"Error querying Prometheus for {trigger.name}: {e}")
                continue

            if value is None:
                continue

            # ตรวจสอบ threshold
            breach = False
            if trigger.comparison == "greater_than" and value > trigger.threshold:
                breach = True
            elif trigger.comparison == "less_than" and value < trigger.threshold:
                breach = True

            if breach:
                key = trigger.name
                self.failure_counts[key] = self.failure_counts.get(key, 0) + 1

                if self.failure_counts[key] >= trigger.consecutive_failures:
                    return RollbackDecision(
                        should_rollback=True,
                        triggered_by=trigger.name,
                        current_value=value,
                        threshold=trigger.threshold,
                        reason=f"{trigger.name} = {value:.4f} (threshold: {trigger.comparison} {trigger.threshold})",
                    )
            else:
                # Reset failure count ถ้า metric กลับมาปกติ
                self.failure_counts.pop(trigger.name, None)

        return RollbackDecision(
            should_rollback=False,
            triggered_by=None,
            current_value=None,
            threshold=None,
            reason="All metrics within acceptable range",
        )

    def _query_prometheus(self, query: str) -> Optional[float]:
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query",
            params={"query": query},
            timeout=10,
        )
        response.raise_for_status()

        data = response.json()
        result = data.get("data", {}).get("result", [])

        if result:
            return float(result[0]["value"][1])
        return None
```

### Rollback GitHub Action

```yaml
# .github/workflows/auto-rollback.yaml
name: Auto-Rollback Monitor

on:
  workflow_dispatch:
    inputs:
      deployment_id:
        description: 'Deployment ID to monitor'
        required: true

jobs:
  monitor-and-rollback:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Monitor deployment for 30 minutes
        id: monitor
        timeout-minutes: 35
        run: |
          DEPLOYMENT_ID="${{ github.event.inputs.deployment_id }}"
          END_TIME=$(($(date +%s) + 1800))  # 30 minutes

          echo "Starting rollback monitor for deployment $DEPLOYMENT_ID"

          while [ $(date +%s) -lt $END_TIME ]; do
            # ตรวจสอบ error rate
            ERROR_RATE=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
              --data-urlencode 'query=sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))' \
              | jq -r '.data.result[0].value[1] // "0"')

            # ตรวจสอบ latency
            P99_LATENCY=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
              --data-urlencode 'query=histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))' \
              | jq -r '.data.result[0].value[1] // "0"')

            echo "$(date): Error Rate = ${ERROR_RATE}, P99 Latency = ${P99_LATENCY}s"

            # ตรวจสอบ threshold
            if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
              echo "🚨 Error rate too high: ${ERROR_RATE}"
              echo "rollback_needed=true" >> $GITHUB_OUTPUT
              echo "reason=error_rate_exceeded" >> $GITHUB_OUTPUT
              exit 0
            fi

            if (( $(echo "$P99_LATENCY > 2.0" | bc -l) )); then
              echo "🚨 Latency too high: ${P99_LATENCY}s"
              echo "rollback_needed=true" >> $GITHUB_OUTPUT
              echo "reason=latency_exceeded" >> $GITHUB_OUTPUT
              exit 0
            fi

            sleep 60
          done

          echo "rollback_needed=false" >> $GITHUB_OUTPUT
          echo "✅ Deployment monitoring complete - no issues detected"

      - name: Execute Rollback
        if: steps.monitor.outputs.rollback_needed == 'true'
        run: |
          REASON="${{ steps.monitor.outputs.reason }}"
          echo "🔄 Initiating rollback due to: $REASON"

          # Rollback Kubernetes deployment
          kubectl rollout undo deployment/"$SERVICE_NAME" -n production

          # Wait for rollback to complete
          kubectl rollout status deployment/"$SERVICE_NAME" -n production --timeout=300s

          echo "✅ Rollback completed"

      - name: Send notification
        if: always()
        run: |
          if [ "${{ steps.monitor.outputs.rollback_needed }}" == "true" ]; then
            MSG="🔄 *Auto-Rollback Executed*\nDeployment: ${{ github.event.inputs.deployment_id }}\nReason: ${{ steps.monitor.outputs.reason }}"
          else
            MSG="✅ *Deployment Stable*\nDeployment: ${{ github.event.inputs.deployment_id }}\nMonitored for 30 minutes - no issues"
          fi

          curl -X POST "$SLACK_WEBHOOK" \
            -H "Content-Type: application/json" \
            -d "{\"text\": \"$MSG\"}"
```

---

## 78.5 Change Advisory Board (CAB) Automation

### Digital CAB Process

```python
# cab/digital-cab.py
from dataclasses import dataclass
from typing import List
from datetime import datetime, timedelta
import requests

@dataclass
class CABRequest:
    title: str
    description: str
    risk_score: float
    risk_category: str
    services_affected: List[str]
    rollback_plan: str
    testing_evidence: str
    deployment_window: str
    requester: str
    team: str
    pr_url: str
    jira_ticket: str


class DigitalCABProcess:
    def __init__(self, slack_webhook: str, jira_url: str, jira_token: str):
        self.slack_webhook = slack_webhook
        self.jira_url = jira_url
        self.jira_token = jira_token

    def submit_cab_request(self, request: CABRequest) -> str:
        """ส่ง CAB request และสร้าง JIRA ticket"""

        # สร้าง JIRA ticket
        jira_key = self.create_jira_ticket(request)

        # แจ้ง CAB members ใน Slack
        self.notify_cab_members(request, jira_key)

        return jira_key

    def create_jira_ticket(self, request: CABRequest) -> str:
        """สร้าง JIRA ticket สำหรับ CAB review"""

        description = f"""
h2. Change Request Summary

*Risk Score:* {request.risk_score}/100 ({request.risk_category.upper()})
*Requester:* {request.requester} ({request.team})
*Services:* {', '.join(request.services_affected)}

h2. Description
{request.description}

h2. Rollback Plan
{request.rollback_plan}

h2. Testing Evidence
{request.testing_evidence}

h2. Deployment Window
{request.deployment_window}

h2. Links
* PR: {request.pr_url}
* JIRA: {request.jira_ticket}

h2. CAB Checklist
* [  ] Risk assessment reviewed
* [  ] Rollback plan verified
* [  ] Testing evidence reviewed
* [  ] Deployment window approved
* [  ] Communication plan reviewed
"""

        payload = {
            "fields": {
                "project": {"key": "CHANGE"},
                "summary": f"[CAB] {request.title}",
                "description": description,
                "issuetype": {"name": "Change Request"},
                "priority": {
                    "name": "Critical" if request.risk_score >= 75 else
                            "High" if request.risk_score >= 50 else
                            "Medium"
                },
                "customfield_risk_score": request.risk_score,
                "customfield_deployment_window": request.deployment_window,
                "labels": ["cab-review", f"risk-{request.risk_category}"],
            }
        }

        response = requests.post(
            f"{self.jira_url}/rest/api/2/issue",
            json=payload,
            headers={
                "Authorization": f"Bearer {self.jira_token}",
                "Content-Type": "application/json",
            },
        )
        response.raise_for_status()
        return response.json()["key"]

    def notify_cab_members(self, request: CABRequest, jira_key: str) -> None:
        """แจ้ง CAB members ผ่าน Slack"""

        # หา emoji ตาม risk
        risk_emoji = {
            "low": "🟢",
            "medium": "🟡",
            "high": "🟠",
            "critical": "🔴",
        }.get(request.risk_category, "⚪")

        message = {
            "blocks": [
                {
                    "type": "header",
                    "text": {
                        "type": "plain_text",
                        "text": f"{risk_emoji} CAB Review Required: {request.title}",
                    },
                },
                {
                    "type": "section",
                    "fields": [
                        {
                            "type": "mrkdwn",
                            "text": f"*Risk Score:*\n{request.risk_score}/100 ({request.risk_category})",
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Team:*\n{request.team}",
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Services:*\n{', '.join(request.services_affected)}",
                        },
                        {
                            "type": "mrkdwn",
                            "text": f"*Deployment Window:*\n{request.deployment_window}",
                        },
                    ],
                },
                {
                    "type": "actions",
                    "elements": [
                        {
                            "type": "button",
                            "text": {"type": "plain_text", "text": "Approve"},
                            "style": "primary",
                            "value": f"approve_{jira_key}",
                        },
                        {
                            "type": "button",
                            "text": {"type": "plain_text", "text": "Reject"},
                            "style": "danger",
                            "value": f"reject_{jira_key}",
                        },
                        {
                            "type": "button",
                            "text": {"type": "plain_text", "text": "View in JIRA"},
                            "url": f"{self.jira_url}/browse/{jira_key}",
                        },
                    ],
                },
            ]
        }

        requests.post(self.slack_webhook, json=message)
```

---

## 78.6 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Change Classification Rules

```python
# exercises/custom-change-classifier.py
# สร้าง classifier ที่เหมาะกับ organization ของคุณ

from change_classifier import ChangeRiskClassifier, ChangeRequest

# TODO: ปรับ rules ให้เหมาะกับ org ของคุณ

class MyOrgChangeClassifier(ChangeRiskClassifier):
    # กำหนด critical services ของ org
    CRITICAL_SERVICES = {
        # TODO: เพิ่ม services สำคัญของ org คุณ
        # 'your-service': risk_multiplier,
    }

    # กำหนด high-risk patterns
    HIGH_RISK_PATTERNS = [
        # TODO: เพิ่ม patterns ที่เกี่ยวข้องกับ business ของคุณ
    ]

    def classify(self, change: ChangeRequest):
        # TODO: เพิ่ม custom rules
        base_result = super().classify(change)
        # ปรับ score ตาม custom rules
        return base_result

# ทดสอบ classifier
test_changes = [
    ChangeRequest(
        title="Update database schema",
        description="Add new column to users table",
        services_affected=["user-service"],
        files_changed=["db/migrations/V001__add_column.sql"],
        lines_added=10,
        lines_deleted=0,
        author="developer@company.com",
        team="platform",
        deployment_time="business_hours",
        has_rollback_plan=True,
        has_test_evidence=True,
        environment="production",
    ),
    # เพิ่ม test cases เพิ่มเติม...
]

classifier = MyOrgChangeClassifier()
for change in test_changes:
    result = classifier.classify(change)
    print(f"Change: {change.title}")
    print(f"Risk: {result.score}/100 ({result.category})")
    print(f"Approval: {result.recommended_approval}")
    print(f"Factors: {result.factors}")
    print()
```

### แบบฝึกหัดที่ 2: Database Migration with Rollback

```sql
-- exercises/migration-with-rollback.sql
-- สร้าง database migration ที่ rollback ได้

-- Forward migration (V001)
-- TODO: สร้าง migration ที่ใช้ Expand-Contract pattern

-- Step 1: Expand
-- เพิ่ม column ใหม่ (nullable)
-- TODO

-- Step 2: Create sync trigger
-- ให้ข้อมูลใน column ใหม่ sync กับ column เก่า
-- TODO

-- Undo migration (U001)
-- TODO: สร้าง undo migration ที่ rollback ได้อย่างปลอดภัย
-- ต้องไม่ทำให้ data loss
-- TODO
```

### แบบฝึกหัดที่ 3: ทดสอบ Rollback Process

```bash
#!/bin/bash
# exercises/test-rollback.sh
# ทดสอบ rollback process ของคุณ

# 1. Deploy version ใหม่ (intentionally broken)
echo "Deploying broken version..."
kubectl set image deployment/test-app \
  test-app=ghcr.io/mycompany/test-app:broken-version

# 2. รอให้ deployment เสร็จ
kubectl rollout status deployment/test-app --timeout=120s

# 3. ตรวจสอบว่า error rate สูงขึ้น
sleep 30
ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[1m])' \
  | jq '.data.result[0].value[1]')
echo "Error rate: $ERROR_RATE"

# 4. Trigger rollback
echo "Triggering rollback..."
kubectl rollout undo deployment/test-app

# 5. ตรวจสอบว่า service กลับมาปกติ
kubectl rollout status deployment/test-app --timeout=120s
sleep 30
ERROR_RATE_AFTER=$(curl -s "http://prometheus:9090/api/v1/query" \
  --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[1m])' \
  | jq '.data.result[0].value[1]')
echo "Error rate after rollback: $ERROR_RATE_AFTER"

# TODO: วัดเวลาที่ใช้
# TODO: ตรวจสอบว่า rollback สำเร็จ
# TODO: บันทึกผลลัพธ์ใน report
```

---

## สรุป

Change Management ที่ดีทำให้ Deploy บ่อยขึ้นโดยไม่เพิ่มความเสี่ยง:

1. **Automated Risk Classification** ช่วย route changes ไปยัง approval process ที่เหมาะสม
2. **Digital CAB** เร็วกว่า traditional CAB แต่ยังมี human oversight
3. **Database Rollback** ต้องวางแผนล่วงหน้าด้วย Expand-Contract pattern
4. **Automated Rollback Triggers** ลด MTTR โดยไม่ต้องรอ human decision
5. **Rollback Testing** ต้องทดสอบ rollback process อย่างสม่ำเสมอ

เป้าหมาย: ทุก change ควร reversible และ measured

### ขั้นตอนถัดไป

- ศึกษา [Part 79: Multi-Tenant CI/CD](./part-79-multi-tenant-cicd.md)
- ITIL Change Management: https://www.axelos.com/certifications/itil
- Database Migration best practices: https://flywaydb.org/documentation

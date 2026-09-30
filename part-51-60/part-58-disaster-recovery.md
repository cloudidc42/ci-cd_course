# Part 58: Disaster Recovery และ Backup Automation

## บทนำ

Disaster Recovery (DR) คือความสามารถขององค์กรในการกู้คืนระบบ IT หลังจากเกิดเหตุการณ์ฉุกเฉิน ในยุค Cloud Native การ backup และ recovery ได้เปลี่ยนโฉมไปอย่างมาก — จากการ backup tape รายสัปดาห์ สู่การ continuous backup และ automated recovery

บทนี้จะครอบคลุม:
- Backup Strategies
- Velero สำหรับ Kubernetes
- RTO/RPO
- Automated DR Testing
- Runbooks
- Chaos + DR Exercises

---

## 1. Backup Strategies

### 1.1 3-2-1 Backup Rule

```
3 — เก็บ backup อย่างน้อย 3 copies
2 — เก็บใน 2 storage types ที่แตกต่างกัน
1 — เก็บ off-site อย่างน้อย 1 copy

ตัวอย่าง:
├── Copy 1: Local SSD (fast recovery)
├── Copy 2: NAS / SAN (medium speed)
└── Copy 3: Cloud Storage S3/GCS (disaster recovery)
```

### 1.2 Backup Types

```
Full Backup:
- Copy ทุกอย่าง
- เวลา backup: นาน
- เวลา restore: เร็ว
- Storage: มาก

Incremental Backup:
- Copy เฉพาะที่เปลี่ยนแปลงตั้งแต่ backup ล่าสุด
- เวลา backup: เร็ว
- เวลา restore: ช้า (ต้อง apply ทุก incremental)
- Storage: น้อย

Differential Backup:
- Copy ทุกอย่างที่เปลี่ยนตั้งแต่ Full backup ล่าสุด
- เวลา backup: ปานกลาง
- เวลา restore: ปานกลาง
- Storage: ปานกลาง

Continuous Backup (CDP):
- Real-time replication
- เวลา backup: continuous
- เวลา restore: เร็วมาก
- Storage: มากที่สุด
```

### 1.3 RPO และ RTO

```
RPO (Recovery Point Objective):
"สูงสุดที่ยอมรับได้คือต้องไม่เสียข้อมูลมากกว่ากี่ชั่วโมง"

ตัวอย่าง RPO:
- Finance: 0 (ไม่ยอมเสียข้อมูลเลย)
- E-commerce: 15 นาที
- Blog: 24 ชั่วโมง

RTO (Recovery Time Objective):
"ระบบต้องกลับมาทำงานได้ภายในกี่ชั่วโมงหลัง disaster"

ตัวอย่าง RTO:
- Critical systems: < 1 ชั่วโมง
- Important systems: < 4 ชั่วโมง
- Normal systems: < 24 ชั่วโมง

Tier Classification:
Tier 1 (Mission Critical): RPO=0, RTO=<15min
Tier 2 (Business Critical): RPO=1h, RTO=<4h
Tier 3 (Important): RPO=24h, RTO=<24h
Tier 4 (Non-critical): RPO=72h, RTO=<72h
```

---

## 2. Velero สำหรับ Kubernetes

### 2.1 ติดตั้ง Velero

```bash
# ติดตั้ง Velero CLI
curl -sLO https://github.com/vmware-tanzu/velero/releases/download/v1.13.0/velero-v1.13.0-linux-amd64.tar.gz
tar xf velero-v1.13.0-linux-amd64.tar.gz
sudo mv velero-v1.13.0-linux-amd64/velero /usr/local/bin/

# สร้าง AWS credentials สำหรับ Velero
cat > credentials-velero << EOF
[default]
aws_access_key_id=<YOUR-ACCESS-KEY>
aws_secret_access_key=<YOUR-SECRET-KEY>
EOF

# ติดตั้ง Velero บน cluster (AWS)
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.9.0 \
    --bucket my-velero-backups \
    --backup-location-config region=ap-southeast-1 \
    --snapshot-location-config region=ap-southeast-1 \
    --secret-file ./credentials-velero \
    --wait

# ตรวจสอบ
kubectl get pods -n velero
velero backup-location get
```

### 2.2 สร้าง Backup ด้วย Velero

```bash
# Backup ทั้ง cluster
velero backup create full-cluster-backup \
    --wait

# Backup เฉพาะ namespace
velero backup create production-backup \
    --include-namespaces production \
    --wait

# Backup พร้อม volume snapshots
velero backup create production-with-volumes \
    --include-namespaces production \
    --snapshot-volumes \
    --volume-snapshot-locations default \
    --wait

# ดู backup list
velero backup get

# ดูรายละเอียด backup
velero backup describe production-backup

# Restore จาก backup
velero restore create \
    --from-backup production-backup \
    --wait

# Restore เฉพาะบาง resources
velero restore create \
    --from-backup production-backup \
    --include-resources deployments,services,configmaps \
    --wait

# ดู restore status
velero restore get
velero restore describe <restore-name>
```

### 2.3 Schedule Backups

```yaml
# velero/schedules/production-schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-daily-backup
  namespace: velero
spec:
  # ทุกวันเวลา 1am Bangkok (18:00 UTC)
  schedule: "0 18 * * *"
  
  template:
    # Backup configuration
    includedNamespaces:
      - production
      - staging
    excludedNamespaces:
      - kube-system
      - velero
    
    # Resources
    includedResources:
      - "*"
    excludedResources:
      - events
      - events.events.k8s.io
    
    # Volume snapshots
    snapshotVolumes: true
    volumeSnapshotLocations:
      - default
    
    # Labels
    labelSelector:
      matchExpressions:
        - key: environment
          operator: In
          values:
            - production
            - staging
    
    # Retention
    ttl: 720h  # เก็บ 30 วัน
    
    # Storage Location
    storageLocation: default
    
    # Hooks
    hooks:
      resources:
        # Pre-backup hook สำหรับ database
        - name: freeze-postgres
          includedNamespaces:
            - production
          labelSelector:
            matchLabels:
              app: postgres
          pre:
            - exec:
                container: postgres
                command:
                  - /bin/sh
                  - -c
                  - psql -U postgres -c "CHECKPOINT;"
                onError: Fail
                timeout: 30s

---
# Hourly backup สำหรับ critical data
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: critical-data-hourly
  namespace: velero
spec:
  schedule: "0 * * * *"
  
  template:
    includedNamespaces:
      - production
    
    includedResources:
      - configmaps
      - secrets
      - persistentvolumeclaims
    
    snapshotVolumes: false  # ไม่ snapshot volumes ทุกชั่วโมง (แพง)
    
    ttl: 168h  # เก็บ 7 วัน
    
    labelSelector:
      matchLabels:
        backup-tier: critical
```

### 2.4 Cross-Region Backup

```bash
#!/bin/bash
# scripts/cross-region-backup.sh
# Copy Velero backups ไปยัง region อื่นเพื่อ DR

set -euo pipefail

SOURCE_BUCKET="my-velero-backups-primary"
DEST_BUCKET="my-velero-backups-dr"
SOURCE_REGION="ap-southeast-1"
DEST_REGION="ap-northeast-1"

echo "=== Cross-Region Backup Sync ==="
echo "Source: s3://${SOURCE_BUCKET} (${SOURCE_REGION})"
echo "Destination: s3://${DEST_BUCKET} (${DEST_REGION})"

# Sync backups
aws s3 sync \
    "s3://${SOURCE_BUCKET}" \
    "s3://${DEST_BUCKET}" \
    --source-region "${SOURCE_REGION}" \
    --region "${DEST_REGION}" \
    --delete \
    --storage-class STANDARD_IA  # ใช้ Infrequent Access สำหรับ DR

echo "✅ Backup sync complete"

# Verify
SOURCE_COUNT=$(aws s3 ls "s3://${SOURCE_BUCKET}/" --recursive | wc -l)
DEST_COUNT=$(aws s3 ls "s3://${DEST_BUCKET}/" --recursive | wc -l)

echo "Source objects: ${SOURCE_COUNT}"
echo "Destination objects: ${DEST_COUNT}"

if [ "${SOURCE_COUNT}" -ne "${DEST_COUNT}" ]; then
    echo "⚠️ Object count mismatch — check for errors"
fi
```

---

## 3. Database Backup Automation

### 3.1 PostgreSQL Backup Pipeline

```yaml
# k8s/cronjobs/postgres-backup.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  # ทุกวันเวลา 2am Bangkok
  schedule: "0 19 * * *"
  
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            job-type: backup
        spec:
          restartPolicy: OnFailure
          
          serviceAccountName: backup-service-account
          
          containers:
            - name: postgres-backup
              image: postgres:15-alpine
              
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-credentials
                      key: password
                
                - name: S3_BUCKET
                  value: "my-database-backups"
                
                - name: AWS_DEFAULT_REGION
                  value: "ap-southeast-1"
              
              command:
                - /bin/sh
                - -c
                - |
                  set -euo pipefail
                  
                  DATE=$(date +%Y%m%d-%H%M%S)
                  BACKUP_FILE="postgres-backup-${DATE}.sql.gz"
                  
                  echo "Starting backup at ${DATE}"
                  
                  # Dump database
                  pg_dump \
                    -h postgres-primary.production.svc.cluster.local \
                    -U postgres \
                    -d myapp \
                    --format=custom \
                    --compress=9 \
                    --no-password \
                    | gzip > /tmp/${BACKUP_FILE}
                  
                  BACKUP_SIZE=$(du -h /tmp/${BACKUP_FILE} | cut -f1)
                  echo "Backup size: ${BACKUP_SIZE}"
                  
                  # Upload ไปที่ S3
                  aws s3 cp /tmp/${BACKUP_FILE} \
                    s3://${S3_BUCKET}/postgres/${BACKUP_FILE} \
                    --storage-class STANDARD_IA
                  
                  echo "✅ Backup uploaded: s3://${S3_BUCKET}/postgres/${BACKUP_FILE}"
                  
                  # Verify backup
                  UPLOADED_SIZE=$(aws s3 ls \
                    s3://${S3_BUCKET}/postgres/${BACKUP_FILE} \
                    | awk '{print $3}')
                  
                  if [ "${UPLOADED_SIZE}" -gt 0 ]; then
                    echo "✅ Backup verified (${UPLOADED_SIZE} bytes)"
                  else
                    echo "❌ Backup verification failed"
                    exit 1
                  fi
                  
                  # Clean up old backups (เก็บแค่ 30 วัน)
                  CUTOFF_DATE=$(date -d '30 days ago' +%Y%m%d)
                  aws s3 ls s3://${S3_BUCKET}/postgres/ \
                    | while read -r line; do
                        FILE_DATE=$(echo "$line" | awk '{print $4}' | grep -oP '\d{8}')
                        FILE_NAME=$(echo "$line" | awk '{print $4}')
                        if [ ! -z "${FILE_DATE}" ] && [ "${FILE_DATE}" -lt "${CUTOFF_DATE}" ]; then
                          echo "Deleting old backup: ${FILE_NAME}"
                          aws s3 rm "s3://${S3_BUCKET}/postgres/${FILE_NAME}"
                        fi
                      done
              
              resources:
                requests:
                  cpu: "100m"
                  memory: "256Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
```

### 3.2 Database Restore Script

```bash
#!/bin/bash
# scripts/restore-postgres.sh
# Restore PostgreSQL จาก backup

set -euo pipefail

S3_BUCKET="${1:-my-database-backups}"
BACKUP_FILE="${2:-}"  # ถ้าไม่ระบุ จะใช้ backup ล่าสุด
TARGET_DB="${3:-myapp_restored}"

echo "=== PostgreSQL Restore ==="

# ถ้าไม่ระบุ backup file ให้ใช้ล่าสุด
if [ -z "${BACKUP_FILE}" ]; then
    echo "Finding latest backup..."
    BACKUP_FILE=$(aws s3 ls "s3://${S3_BUCKET}/postgres/" \
        | sort | tail -1 | awk '{print $4}')
    echo "Using: ${BACKUP_FILE}"
fi

# Download backup
echo "Downloading backup..."
aws s3 cp "s3://${S3_BUCKET}/postgres/${BACKUP_FILE}" /tmp/

# สร้าง database ใหม่
PGPASSWORD="${PGPASSWORD}" psql \
    -h "${PGHOST:-localhost}" \
    -U postgres \
    -c "CREATE DATABASE ${TARGET_DB};" 2>/dev/null || true

# Restore
echo "Restoring database..."
PGPASSWORD="${PGPASSWORD}" pg_restore \
    -h "${PGHOST:-localhost}" \
    -U postgres \
    -d "${TARGET_DB}" \
    --no-owner \
    --no-privileges \
    --verbose \
    /tmp/${BACKUP_FILE}

echo "✅ Database restored to: ${TARGET_DB}"

# Verify restore
TABLE_COUNT=$(PGPASSWORD="${PGPASSWORD}" psql \
    -h "${PGHOST:-localhost}" \
    -U postgres \
    -d "${TARGET_DB}" \
    -t -c "SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';" \
    | tr -d ' ')

echo "Tables restored: ${TABLE_COUNT}"
```

---

## 4. Automated DR Testing

### 4.1 DR Test Pipeline

```yaml
# .github/workflows/dr-test.yml
name: Disaster Recovery Test

on:
  schedule:
    # ทดสอบทุกวันจันทร์ 3am Bangkok
    - cron: "0 20 * * 0"
  workflow_dispatch:
    inputs:
      test-type:
        description: 'DR test type'
        required: true
        default: 'restore-validation'
        type: choice
        options:
          - restore-validation
          - full-dr-simulation
          - database-only

jobs:
  dr-test:
    runs-on: ubuntu-latest
    environment: dr-test
    timeout-minutes: 60

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/DRTestRole
          aws-region: ap-northeast-1  # DR region

      - name: Setup kubectl
        run: |
          aws eks update-kubeconfig \
            --name dr-cluster \
            --region ap-northeast-1

      - name: Validate Latest Backup
        id: validate-backup
        run: |
          # ตรวจสอบว่า backup มีอยู่
          LATEST_BACKUP=$(aws s3 ls s3://my-velero-backups-dr/ \
            | sort | tail -1 | awk '{print $4}')
          
          if [ -z "${LATEST_BACKUP}" ]; then
            echo "❌ No backup found!"
            exit 1
          fi
          
          BACKUP_DATE=$(echo "${LATEST_BACKUP}" | grep -oP '\d{4}-\d{2}-\d{2}')
          TODAY=$(date +%Y-%m-%d)
          
          if [ "${BACKUP_DATE}" != "${TODAY}" ] && \
             [ "$(date -d "${BACKUP_DATE}" +%s)" -lt "$(date -d '2 days ago' +%s)" ]; then
            echo "❌ Latest backup is too old: ${BACKUP_DATE}"
            exit 1
          fi
          
          echo "✅ Backup found: ${LATEST_BACKUP}"
          echo "backup-name=${LATEST_BACKUP}" >> $GITHUB_OUTPUT

      - name: Restore to DR Cluster
        if: github.event.inputs.test-type != 'database-only'
        run: |
          # Restore ด้วย Velero
          velero restore create dr-test-$(date +%Y%m%d-%H%M%S) \
            --from-backup "${{ steps.validate-backup.outputs.backup-name }}" \
            --include-namespaces production \
            --namespace-mappings production:dr-production \
            --wait

      - name: Test Database Restore
        run: |
          # Restore database ไปที่ DR environment
          LATEST_DB_BACKUP=$(aws s3 ls s3://my-database-backups-dr/postgres/ \
            | sort | tail -1 | awk '{print $4}')
          
          echo "Restoring database: ${LATEST_DB_BACKUP}"
          
          aws s3 cp "s3://my-database-backups-dr/postgres/${LATEST_DB_BACKUP}" /tmp/
          
          # สร้าง test database
          PGPASSWORD="${{ secrets.DR_DB_PASSWORD }}" pg_restore \
            -h dr-postgres.example.com \
            -U postgres \
            -d dr_test_myapp \
            --clean --if-exists \
            /tmp/${LATEST_DB_BACKUP}

      - name: Run Smoke Tests
        run: |
          # รัน smoke tests เพื่อตรวจสอบว่า restore ถูกต้อง
          kubectl rollout status deployment -n dr-production --timeout=5m
          
          # Test endpoints
          DR_URL="https://dr.example.com"
          
          # Health check
          curl -f "${DR_URL}/health" || { echo "❌ Health check failed"; exit 1; }
          
          # API check
          RESPONSE=$(curl -sS "${DR_URL}/api/users" | python3 -c "import json,sys; d=json.load(sys.stdin); print(len(d))")
          echo "Users count: ${RESPONSE}"
          
          if [ "${RESPONSE}" -gt 0 ]; then
            echo "✅ Data integrity verified"
          else
            echo "❌ No data found in DR environment"
            exit 1
          fi

      - name: Measure RTO
        id: rto
        run: |
          # คำนวณ actual RTO
          START_TIME="${{ steps.restore.outputs.start-time }}"
          END_TIME=$(date +%s)
          RTO_MINUTES=$(( (END_TIME - START_TIME) / 60 ))
          
          echo "Actual RTO: ${RTO_MINUTES} minutes"
          echo "rto-minutes=${RTO_MINUTES}" >> $GITHUB_OUTPUT
          
          # Fail ถ้า RTO เกิน target
          if [ "${RTO_MINUTES}" -gt 30 ]; then
            echo "❌ RTO ${RTO_MINUTES}min exceeds target 30min"
            exit 1
          fi
          
          echo "✅ RTO ${RTO_MINUTES}min within target 30min"

      - name: Generate DR Test Report
        if: always()
        run: |
          cat > dr-test-report.md << EOF
          # Disaster Recovery Test Report
          
          **Date**: $(date)
          **Test Type**: ${{ github.event.inputs.test-type || 'scheduled' }}
          **Status**: ${{ job.status }}
          
          ## Metrics
          - Backup Age: Valid
          - Restore Status: ${{ steps.restore-cluster.outcome || 'N/A' }}
          - Database Restore: ${{ steps.test-db.outcome }}
          - Smoke Tests: ${{ steps.smoke-tests.outcome }}
          - Actual RTO: ${{ steps.rto.outputs.rto-minutes || 'N/A' }} minutes
          
          ## Backup Information
          - Latest Backup: ${{ steps.validate-backup.outputs.backup-name }}
          
          ## Actions Required
          $([ "${{ job.status }}" != "success" ] && echo "- Review failed steps" || echo "- None")
          EOF

      - name: Upload Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dr-test-report
          path: dr-test-report.md

      - name: Cleanup DR Test Resources
        if: always()
        run: |
          # ลบ DR test resources
          kubectl delete namespace dr-production --ignore-not-found=true || true
          
          # ลบ test database
          PGPASSWORD="${{ secrets.DR_DB_PASSWORD }}" psql \
            -h dr-postgres.example.com \
            -U postgres \
            -c "DROP DATABASE IF EXISTS dr_test_myapp;" || true

      - name: Notify Results
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: "sre-alerts"
          slack-message: |
            *DR Test ${{ job.status == 'success' && '✅ PASSED' || '❌ FAILED' }}*
            RTO: ${{ steps.rto.outputs.rto-minutes || 'N/A' }} minutes
            Report: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## 5. Runbooks

### 5.1 Database Recovery Runbook

```markdown
# Runbook: Database Recovery

## Severity
Critical (P0)

## When to Use
- Database primary node failure
- Data corruption detected
- Accidental data deletion

## Prerequisites
- Access to AWS Console
- kubectl access to production cluster
- Database admin credentials (in Vault)

## Steps

### 1. Assess the Situation (5 min)
```bash
# ตรวจสอบ database status
kubectl get pods -n production -l app=postgres

# ตรวจสอบ database logs
kubectl logs -n production -l app=postgres --previous | tail -50

# ตรวจสอบ error messages
kubectl describe pod -n production -l app=postgres
```

### 2. Notify Stakeholders (2 min)
- แจ้ง Slack channel #incidents: "Database incident detected, investigating"
- แจ้ง on-call manager

### 3. Attempt Quick Fix (10 min)
```bash
# ลอง restart pod ก่อน
kubectl rollout restart deployment/postgres -n production

# รอ 2 นาที
sleep 120

# ตรวจสอบอีกครั้ง
kubectl get pods -n production -l app=postgres
```

### 4. Failover to Replica (5 min)
```bash
# ถ้า primary down ให้ promote replica
kubectl exec -n production postgres-replica-0 -- \
  pg_ctl promote -D /var/lib/postgresql/data

# Update service endpoint
kubectl patch service postgres -n production \
  -p '{"spec":{"selector":{"role":"replica"}}}'
```

### 5. Restore from Backup (ถ้าจำเป็น)
```bash
# หา backup ล่าสุด
LATEST=$(aws s3 ls s3://my-database-backups/postgres/ | sort | tail -1)
echo "Latest backup: ${LATEST}"

# Download และ restore
./scripts/restore-postgres.sh my-database-backups \
  "${LATEST}" \
  myapp
```

## Verification
```bash
# ตรวจสอบ connectivity
psql -h postgres.production.svc.cluster.local -U app -d myapp -c "\dt"

# ตรวจสอบ row count
psql -h postgres.production.svc.cluster.local -U app -d myapp \
  -c "SELECT count(*) FROM users;"
```

## Post-Incident
1. Document timeline ใน incident report
2. Identify root cause
3. Update runbook ถ้าจำเป็น
4. Schedule post-mortem
```

### 5.2 Runbook Automation

```python
# scripts/runbook_executor.py
"""
Automated Runbook Executor
"""
import json
import subprocess
import sys
import time
from dataclasses import dataclass
from typing import List, Callable, Optional


@dataclass
class RunbookStep:
    """Step ใน Runbook"""
    name: str
    description: str
    command: Optional[str] = None
    function: Optional[Callable] = None
    timeout_seconds: int = 60
    on_failure: str = "abort"  # abort, continue, manual


class RunbookExecutor:
    """Execute runbook steps อัตโนมัติ"""
    
    def __init__(self, runbook_name: str, dry_run: bool = False):
        self.runbook_name = runbook_name
        self.dry_run = dry_run
        self.results = []
    
    def run(self, steps: List[RunbookStep]) -> bool:
        """รัน runbook steps"""
        print(f"\n=== Running Runbook: {self.runbook_name} ===")
        if self.dry_run:
            print("DRY RUN MODE — no changes will be made\n")
        
        all_success = True
        
        for i, step in enumerate(steps, 1):
            print(f"\nStep {i}/{len(steps)}: {step.name}")
            print(f"Description: {step.description}")
            
            start_time = time.time()
            success = False
            output = ""
            
            try:
                if self.dry_run:
                    print(f"  [DRY RUN] Would execute: {step.command or 'function'}")
                    success = True
                elif step.command:
                    result = subprocess.run(
                        step.command,
                        shell=True,
                        capture_output=True,
                        text=True,
                        timeout=step.timeout_seconds
                    )
                    output = result.stdout + result.stderr
                    success = result.returncode == 0
                elif step.function:
                    step.function()
                    success = True
            
            except subprocess.TimeoutExpired:
                output = f"Command timed out after {step.timeout_seconds}s"
                success = False
            except Exception as e:
                output = str(e)
                success = False
            
            duration = time.time() - start_time
            
            status = "✅" if success else "❌"
            print(f"  {status} Completed in {duration:.1f}s")
            
            if output and (not success or "ERROR" in output.upper()):
                print(f"  Output: {output[:500]}")
            
            self.results.append({
                "step": step.name,
                "success": success,
                "duration_seconds": duration,
                "output": output[:1000]
            })
            
            if not success:
                all_success = False
                
                if step.on_failure == "abort":
                    print(f"\n❌ Step failed. Aborting runbook.")
                    break
                elif step.on_failure == "manual":
                    print(f"\n⚠️  Step failed. Manual intervention required.")
                    input("Press Enter to continue after manual fix, or Ctrl+C to abort...")
                elif step.on_failure == "continue":
                    print(f"  ⚠️  Step failed but continuing...")
        
        self._save_results()
        return all_success
    
    def _save_results(self):
        """บันทึก results"""
        report = {
            "runbook": self.runbook_name,
            "dry_run": self.dry_run,
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
            "overall_success": all(r["success"] for r in self.results),
            "steps": self.results
        }
        
        with open("runbook-result.json", 'w') as f:
            json.dump(report, f, indent=2)
        
        success_count = sum(1 for r in self.results if r["success"])
        print(f"\n{'='*50}")
        print(f"Runbook Complete: {success_count}/{len(self.results)} steps passed")
        print(f"Results saved to runbook-result.json")


# ตัวอย่าง: Database Recovery Runbook
def run_database_recovery():
    """Execute database recovery runbook"""
    
    executor = RunbookExecutor("Database Recovery", dry_run="--dry-run" in sys.argv)
    
    steps = [
        RunbookStep(
            name="Check Database Status",
            description="ตรวจสอบ status ของ database pods",
            command="kubectl get pods -n production -l app=postgres",
            on_failure="continue"
        ),
        RunbookStep(
            name="Attempt Pod Restart",
            description="ลอง restart database pod",
            command="kubectl rollout restart deployment/postgres -n production",
            on_failure="continue"
        ),
        RunbookStep(
            name="Wait for Recovery",
            description="รอให้ pod ขึ้นมา",
            command="kubectl rollout status deployment/postgres -n production --timeout=120s",
            timeout_seconds=130,
            on_failure="continue"
        ),
        RunbookStep(
            name="Verify Database Connection",
            description="ตรวจสอบว่า database ตอบสนองได้",
            command="""kubectl exec -n production deployment/postgres -- \
                psql -U postgres -c "SELECT 1;" """,
            on_failure="continue"
        ),
        RunbookStep(
            name="Find Latest Backup",
            description="ค้นหา backup ล่าสุดใน S3",
            command="""aws s3 ls s3://my-database-backups/postgres/ | sort | tail -1""",
            on_failure="abort"
        ),
    ]
    
    success = executor.run(steps)
    sys.exit(0 if success else 1)


if __name__ == '__main__':
    run_database_recovery()
```

---

## 6. Multi-Region DR

### 6.1 Active-Passive Setup

```
Region: AP-Southeast-1 (Primary)              Region: AP-Northeast-1 (DR)
┌──────────────────────────────┐              ┌──────────────────────────────┐
│  EKS Cluster (Active)        │              │  EKS Cluster (Passive)       │
│  ├── Production workloads    │              │  ├── Standby only            │
│  ├── Primary RDS             │              │  ├── RDS Read Replica        │
│  └── ElastiCache             │              │  └── Velero backups          │
└──────────────────────────────┘              └──────────────────────────────┘
              │                                             │
              └─────────── Replication ─────────────────────┘
                          (RDS cross-region, S3 replication)
```

### 6.2 DR Failover Script

```bash
#!/bin/bash
# scripts/dr-failover.sh
# Failover ไปยัง DR region

set -euo pipefail

PRIMARY_REGION="ap-southeast-1"
DR_REGION="ap-northeast-1"
PRIMARY_CLUSTER="production"
DR_CLUSTER="production-dr"
DNS_HOSTED_ZONE="Z1234567890"
PRIMARY_ENDPOINT="primary.example.com"
DR_ENDPOINT="dr.example.com"

echo "=== DR FAILOVER INITIATED ==="
echo "From: ${PRIMARY_REGION}/${PRIMARY_CLUSTER}"
echo "To: ${DR_REGION}/${DR_CLUSTER}"
echo ""
echo "⚠️  This will switch all traffic to DR region!"
read -p "Are you sure? (yes/no): " CONFIRM

if [ "${CONFIRM}" != "yes" ]; then
    echo "Failover cancelled"
    exit 0
fi

START_TIME=$(date +%s)

# 1. Promote RDS Read Replica เป็น Primary
echo "Step 1: Promoting RDS Read Replica..."
aws rds promote-read-replica \
    --db-instance-identifier myapp-db-dr \
    --region "${DR_REGION}"

# รอให้ promote เสร็จ (อาจใช้เวลา 5-10 นาที)
echo "Waiting for promotion to complete..."
aws rds wait db-instance-available \
    --db-instance-identifier myapp-db-dr \
    --region "${DR_REGION}"
echo "✅ RDS promoted"

# 2. Update Velero location ใน DR cluster
echo "Step 2: Configuring Velero in DR cluster..."
aws eks update-kubeconfig \
    --name "${DR_CLUSTER}" \
    --region "${DR_REGION}"

# 3. Restore latest backup
echo "Step 3: Restoring latest backup..."
LATEST_BACKUP=$(velero backup get --output json 2>/dev/null | \
    python3 -c "
import json, sys
backups = json.load(sys.stdin)
if backups.get('items'):
    completed = [b for b in backups['items'] if b['status']['phase'] == 'Completed']
    if completed:
        latest = max(completed, key=lambda b: b['metadata']['creationTimestamp'])
        print(latest['metadata']['name'])
")

if [ -z "${LATEST_BACKUP}" ]; then
    echo "❌ No backup found!"
    exit 1
fi

echo "Restoring from: ${LATEST_BACKUP}"
velero restore create dr-failover-$(date +%Y%m%d-%H%M%S) \
    --from-backup "${LATEST_BACKUP}" \
    --wait

# 4. Update environment configs
echo "Step 4: Updating configurations for DR environment..."
kubectl set env deployment -n production \
    DB_HOST="myapp-db-dr.${DR_REGION}.rds.amazonaws.com" \
    REGION="${DR_REGION}"

# 5. Verify services
echo "Step 5: Verifying DR services..."
kubectl rollout status deployment -n production --timeout=10m

# 6. Update DNS (Route53)
echo "Step 6: Updating DNS to point to DR..."
DR_LB=$(kubectl get service -n production istio-ingressgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

aws route53 change-resource-record-sets \
    --hosted-zone-id "${DNS_HOSTED_ZONE}" \
    --change-batch "{
        \"Changes\": [{
            \"Action\": \"UPSERT\",
            \"ResourceRecordSet\": {
                \"Name\": \"${PRIMARY_ENDPOINT}\",
                \"Type\": \"CNAME\",
                \"TTL\": 60,
                \"ResourceRecords\": [{
                    \"Value\": \"${DR_LB}\"
                }]
            }
        }]
    }"

# คำนวณ RTO
END_TIME=$(date +%s)
RTO_MINUTES=$(( (END_TIME - START_TIME) / 60 ))

echo ""
echo "=== FAILOVER COMPLETE ==="
echo "RTO: ${RTO_MINUTES} minutes"
echo ""
echo "Next steps:"
echo "1. Verify all services at DR endpoint"
echo "2. Monitor error rates"
echo "3. Notify stakeholders"
echo "4. Plan failback when primary is restored"
```

---

## 7. Workshop: DR Testing Lab

### Lab 1: Velero Backup และ Restore

```bash
#!/bin/bash
# workshop/lab1-velero.sh

echo "=== Lab 1: Velero Backup and Restore ==="

# 1. Deploy test application
kubectl create namespace dr-test 2>/dev/null || true

cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: dr-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
  namespace: dr-test
data:
  app.conf: |
    key1=value1
    key2=value2
    important_data=this-should-survive-backup
EOF

# 2. สร้าง backup
echo "Creating backup..."
velero backup create lab1-backup \
    --include-namespaces dr-test \
    --wait

velero backup describe lab1-backup

# 3. ลบ namespace
echo "Deleting namespace (simulating disaster)..."
kubectl delete namespace dr-test

# 4. Restore
echo "Restoring from backup..."
velero restore create lab1-restore \
    --from-backup lab1-backup \
    --wait

velero restore describe lab1-restore

# 5. Verify
echo "Verifying restore..."
kubectl get all -n dr-test
kubectl get configmap test-config -n dr-test -o yaml

echo "✅ Lab 1 Complete!"
```

### Lab 2: Automated DR Test

```yaml
# workshop/lab2-dr-test/schedule.yaml
# Schedule automated DR tests ทุกสัปดาห์

apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-dr-test
  namespace: velero
spec:
  schedule: "0 20 * * 0"  # ทุกวันอาทิตย์ 3am Bangkok
  
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: dr-test-sa
          restartPolicy: Never
          containers:
            - name: dr-tester
              image: bitnami/kubectl:latest
              command:
                - /bin/sh
                - -c
                - |
                  echo "=== Weekly DR Test ==="
                  
                  # สร้าง restore ใน namespace แยก
                  LATEST=$(velero backup get -o json | \
                    python3 -c "import json,sys; d=json.load(sys.stdin); print([b['metadata']['name'] for b in d.get('items',[]) if b['status']['phase']=='Completed'][-1])")
                  
                  echo "Latest backup: ${LATEST}"
                  
                  velero restore create \
                    weekly-dr-test-$(date +%Y%m%d) \
                    --from-backup ${LATEST} \
                    --namespace-mappings production:dr-test \
                    --wait
                  
                  # Verify
                  kubectl get pods -n dr-test
                  
                  # Cleanup
                  kubectl delete namespace dr-test
                  
                  echo "✅ DR test complete"
```

---

## 8. สรุปและ Best Practices

### DR Checklist

```markdown
## Disaster Recovery Checklist

### Backup Strategy
- [ ] 3-2-1 backup rule implemented
- [ ] Backup frequency matches RPO
- [ ] Automated backup verification
- [ ] Cross-region backup replication
- [ ] Backup retention policy defined

### Recovery
- [ ] Runbooks documented และ tested
- [ ] RTO validated
- [ ] RPO validated
- [ ] Automated DR testing
- [ ] DR team training

### Kubernetes
- [ ] Velero installed และ configured
- [ ] Scheduled backups
- [ ] Volume snapshots
- [ ] Cross-namespace restore tested
- [ ] Cross-cluster restore tested

### Database
- [ ] Point-in-time recovery enabled
- [ ] Read replica in DR region
- [ ] Automated failover tested
- [ ] Data integrity checks

### CI/CD
- [ ] DR environment deployment automated
- [ ] Health checks after restore
- [ ] Smoke tests automated
- [ ] Notification configured
```

---

## อ้างอิง

- [Velero Documentation](https://velero.io/docs/)
- [AWS Disaster Recovery](https://aws.amazon.com/disaster-recovery/)
- [Google Cloud DR Planning](https://cloud.google.com/architecture/dr-scenarios-planning-guide)
- [NIST SP 800-34: Contingency Planning Guide](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-34r1.pdf)
- [The Site Reliability Workbook](https://sre.google/workbook/table-of-contents/)

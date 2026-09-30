# Part 80: CI/CD for Legacy Systems

## บทนำ

Legacy systems คือระบบเก่าที่ยังใช้งานอยู่ มักจะมีโค้ดที่เขียนนานแล้ว ขาด tests ขาด documentation และยากต่อการ deploy ด้วย modern CI/CD practices อย่างไรก็ตาม การ modernize legacy systems เป็นสิ่งที่จำเป็นสำหรับ digital transformation

ในบทนี้เราจะเรียนรู้:
- Strangler Fig Pattern
- Incremental Modernization
- Testing Legacy Code
- Wrapping Legacy with Containers
- Migration Strategy
- แบบฝึกหัดปฏิบัติ

---

## 80.1 ความท้าทายของ Legacy Systems

### Legacy System คืออะไร?

```
Legacy system มักมีลักษณะ:
├── โค้ดเก่า 10-30+ ปี
├── ขาด unit tests (หรือมีน้อยมาก)
├── Documentation ไม่ครบ
├── Dependencies ที่ outdated
├── Monolithic architecture
├── Deploy โดย manual process
├── รัน on bare metal หรือ VMs
└── ทีมที่รู้จักระบบออกไปแล้ว
```

### ทำไมต้อง Modernize?

```
Business Drivers:
├── Time to market ช้า
├── ต้นทุนการ maintenance สูง
├── ไม่สามารถ scale ได้
├── Security vulnerabilities
└── Developer unhappiness (ยากในการหา talent)

Technical Drivers:
├── ไม่สามารถ deploy บ่อยๆ ได้
├── ไม่มี rollback ที่ reliable
├── Environment inconsistency
└── Difficult to test
```

---

## 80.2 Strangler Fig Pattern

### แนวคิด

```
Strangler Fig Pattern:
1. ไม่ rewrite ทั้งหมดในคราวเดียว (Big Bang rewrite = ล้มเหลวเสมอ)
2. สร้าง functionality ใหม่ alongside legacy
3. ค่อยๆ redirect traffic ไปยัง new system
4. เมื่อ legacy ไม่มี traffic แล้ว → ลบออก

ชื่อมาจาก: strangler fig tree ที่เติบโตรอบต้นไม้เก่า
จนในที่สุด "กิน" ต้นไม้เก่าและแทนที่มัน
```

### ขั้นตอนการ Implement

```
Phase 1: Setup Facade/Proxy
┌────────────┐     ┌─────────────┐     ┌──────────────┐
│  Browser   │────►│  API Proxy  │────►│ Legacy System│
│            │     │  (Nginx)    │     │ (Monolith)   │
└────────────┘     └─────────────┘     └──────────────┘

Phase 2: Extract first feature
┌────────────┐     ┌─────────────┐     ┌──────────────┐
│  Browser   │────►│  API Proxy  │────►│ Legacy System│
│            │     │  (Nginx)    │     │ (Monolith)   │
│            │     │             │     └──────────────┘
│            │     │  /users/*   │────►┌──────────────┐
│            │     │  routes new │     │  New Users   │
│            │     │             │     │  Service     │
└────────────┘     └─────────────┘     └──────────────┘

Phase 3: Extract more features...

Phase N: Legacy is empty → Remove
```

### Nginx Strangler Proxy Configuration

```nginx
# nginx/strangler-proxy.conf
upstream legacy_monolith {
    server legacy-app:8080;
}

upstream new_user_service {
    server user-service:8080;
}

upstream new_payment_service {
    server payment-service:8080;
}

upstream new_product_service {
    server product-service:8080;
}

server {
    listen 80;
    server_name api.mycompany.com;

    # ─── New Microservices ────────────────────────────────────
    # Users API - ย้ายแล้ว
    location /api/v1/users {
        proxy_pass http://new_user_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Logging สำหรับ monitoring migration
        access_log /var/log/nginx/new_service_access.log migration_log;
    }

    # Payments API - ย้ายแล้ว
    location /api/v1/payments {
        proxy_pass http://new_payment_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Products API - ย้ายแล้ว
    location /api/v1/products {
        proxy_pass http://new_product_service;
    }

    # ─── Legacy System (everything else) ─────────────────────
    location / {
        proxy_pass http://legacy_monolith;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # Log legacy requests สำหรับ analysis
        access_log /var/log/nginx/legacy_access.log migration_log;
    }
}

# Log format สำหรับ migration monitoring
log_format migration_log '$remote_addr - $remote_user [$time_local] '
                         '"$request" $status $body_bytes_sent '
                         '"$http_referer" "$http_user_agent" '
                         'upstream=$upstream_addr response_time=$upstream_response_time';
```

### Feature Toggle สำหรับ Gradual Migration

```typescript
// strangler/migration-flags.ts
interface MigrationFlag {
  feature: string;
  targetService: 'legacy' | 'new';
  rolloutPercentage: number;
  enabledForUsers?: string[];  // specific user IDs
  disabledForUsers?: string[];
}

class MigrationRouter {
  private flags: Map<string, MigrationFlag> = new Map();

  async shouldUseNewService(
    feature: string,
    userId: string,
  ): Promise<boolean> {
    const flag = this.flags.get(feature);
    if (!flag) return false;

    // Always use legacy if explicitly disabled
    if (flag.disabledForUsers?.includes(userId)) return false;

    // Always use new if explicitly enabled
    if (flag.enabledForUsers?.includes(userId)) return true;

    // Percentage rollout
    const userBucket = this.hashUserId(userId) % 100;
    return userBucket < flag.rolloutPercentage;
  }

  private hashUserId(userId: string): number {
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      hash = (hash << 5) - hash + userId.charCodeAt(i);
      hash |= 0;
    }
    return Math.abs(hash);
  }
}

// Express middleware
export function migrationMiddleware(router: MigrationRouter) {
  return async (req: Request, res: Response, next: NextFunction) => {
    const userId = req.headers['x-user-id'] as string;
    const feature = extractFeatureFromPath(req.path);

    if (feature && userId) {
      const useNew = await router.shouldUseNewService(feature, userId);
      req.headers['x-use-new-service'] = useNew ? 'true' : 'false';

      // Log for monitoring
      console.log(`migration_router: feature=${feature} user=${userId} use_new=${useNew}`);
    }

    next();
  };
}
```

---

## 80.3 Testing Legacy Code

### Characterization Tests

```python
# tests/characterization/test_legacy_payment.py
"""
Characterization Tests:
- ไม่ได้ทดสอบว่าโค้ดทำสิ่งที่ถูกต้อง
- ทดสอบว่าโค้ดทำสิ่งที่มันทำอยู่ตอนนี้
- เพื่อให้มั่นใจว่า behavior ไม่เปลี่ยนเมื่อ refactor
"""
import pytest
from unittest.mock import patch, MagicMock
from legacy_code.payment_processor import LegacyPaymentProcessor


class TestLegacyPaymentCharacterization:
    """Characterize existing behavior of LegacyPaymentProcessor"""

    @pytest.fixture
    def processor(self):
        return LegacyPaymentProcessor(
            db_config={"host": "localhost", "db": "testdb"},
            gateway_url="http://test-gateway",
        )

    def test_processes_valid_payment(self, processor):
        """Document current behavior for valid payment"""
        with patch.object(processor, '_call_gateway') as mock_gateway:
            mock_gateway.return_value = {"status": "success", "transaction_id": "TXN123"}

            result = processor.process(
                amount=100.00,
                currency="THB",
                card_token="tok_test",
                customer_id="CUST001",
            )

            # Document what the function currently returns
            assert result["success"] is True
            assert result["transaction_id"] == "TXN123"
            assert "timestamp" in result

    def test_handles_declined_card(self, processor):
        """Document current behavior when card is declined"""
        with patch.object(processor, '_call_gateway') as mock_gateway:
            mock_gateway.return_value = {"status": "declined", "code": "CARD_DECLINED"}

            result = processor.process(
                amount=100.00,
                currency="THB",
                card_token="tok_declined",
                customer_id="CUST001",
            )

            # Document current (possibly odd) behavior
            assert result["success"] is False
            assert result["error_code"] == "CARD_DECLINED"
            # Note: Legacy code returns None for message when declined
            # (this might be a bug but we're documenting current behavior)
            assert result.get("message") is None

    def test_logs_to_database(self, processor):
        """Document that processor logs to DB"""
        with patch.object(processor, '_call_gateway') as mock_gateway, \
             patch.object(processor, '_log_to_db') as mock_db:

            mock_gateway.return_value = {"status": "success", "transaction_id": "TXN123"}

            processor.process(100.00, "THB", "tok_test", "CUST001")

            # Document DB logging behavior
            assert mock_db.called
            call_args = mock_db.call_args[0]
            assert call_args[0] == "CUST001"
            assert call_args[1] == 100.00

    def test_retries_on_timeout(self, processor):
        """Document retry behavior"""
        from requests.exceptions import Timeout

        with patch.object(processor, '_call_gateway') as mock_gateway:
            mock_gateway.side_effect = [Timeout(), {"status": "success", "transaction_id": "TXN123"}]

            result = processor.process(100.00, "THB", "tok_test", "CUST001")

            # Document that it retries once on timeout
            assert mock_gateway.call_count == 2
            assert result["success"] is True
```

### Golden Master Testing

```python
# tests/golden-master/test_report_generation.py
"""
Golden Master Tests:
- บันทึก output ของโค้ดเก่า (golden master)
- ทดสอบว่า output ไม่เปลี่ยนเมื่อ refactor
- มีประโยชน์สำหรับ complex functions ที่ยากจะเขียน unit tests
"""
import json
import hashlib
from pathlib import Path
import pytest

GOLDEN_DIR = Path("tests/golden-master/snapshots")
GOLDEN_DIR.mkdir(parents=True, exist_ok=True)


def golden_master(name: str, actual_output: str, update: bool = False) -> bool:
    """
    Compare output กับ golden master snapshot

    Args:
        name: ชื่อของ snapshot
        actual_output: output จริง
        update: ถ้า True ให้อัพเดท snapshot (ใช้เมื่อตั้งใจจะเปลี่ยน behavior)

    Returns:
        True ถ้า match
    """
    snapshot_file = GOLDEN_DIR / f"{name}.snapshot"

    if update or not snapshot_file.exists():
        snapshot_file.write_text(actual_output)
        return True

    expected = snapshot_file.read_text()
    return actual_output == expected


class TestLegacyReportGeneration:
    """Golden master tests สำหรับ legacy report generator"""

    def test_monthly_sales_report(self):
        from legacy_code.reports import generate_monthly_report

        output = generate_monthly_report(
            year=2025,
            month=1,
            data={
                "sales": [
                    {"product": "A", "quantity": 10, "amount": 1000},
                    {"product": "B", "quantity": 5, "amount": 750},
                ]
            }
        )

        assert golden_master("monthly_sales_report_jan_2025", output), \
            "Report output changed! If intentional, run with update=True"

    def test_customer_summary_report(self):
        from legacy_code.reports import generate_customer_summary

        output = generate_customer_summary(
            customer_id="CUST001",
            include_history=True,
        )

        assert golden_master("customer_summary_CUST001", output), \
            "Customer summary format changed!"
```

### Adding Tests to Legacy Code

```python
# src/legacy_code/payment_processor.py (legacy code)
# โค้ดเดิมที่ไม่มี tests

class LegacyPaymentProcessor:
    def __init__(self, db_config, gateway_url):
        self.db_config = db_config
        self.gateway_url = gateway_url
        self._connect_db()

    def process(self, amount, currency, card_token, customer_id):
        # โค้ดเดิมที่ยุ่งเหยิง
        import requests
        import time

        data = {
            'amount': amount,
            'currency': currency,
            'token': card_token,
            'cust': customer_id,
            'ts': int(time.time()),
        }

        try:
            resp = requests.post(self.gateway_url + '/charge', json=data, timeout=5)
            result = resp.json()

            if result.get('status') == 'success':
                self._log_to_db(customer_id, amount, result['transaction_id'])
                return {
                    'success': True,
                    'transaction_id': result['transaction_id'],
                    'timestamp': data['ts'],
                }
            else:
                return {
                    'success': False,
                    'error_code': result.get('code'),
                }
        except requests.Timeout:
            # Retry once
            try:
                resp = requests.post(self.gateway_url + '/charge', json=data, timeout=10)
                result = resp.json()
                if result.get('status') == 'success':
                    self._log_to_db(customer_id, amount, result['transaction_id'])
                    return {'success': True, 'transaction_id': result['transaction_id'], 'timestamp': data['ts']}
            except Exception:
                pass
            return {'success': False, 'error_code': 'TIMEOUT'}
        except Exception as e:
            return {'success': False, 'error_code': 'INTERNAL_ERROR', 'message': str(e)}
```

---

## 80.4 Wrapping Legacy with Containers

### Containerizing Legacy Java App

```dockerfile
# Dockerfile.legacy-java
# Legacy Java 8 application ที่รัน on bare metal

# Stage 1: Build (ถ้ายังไม่มี build artifact)
FROM maven:3.6-jdk-8 AS builder
WORKDIR /app
COPY pom.xml .

# ดึง dependencies ก่อน (cache layer)
RUN mvn dependency:go-offline -q

COPY src/ src/
RUN mvn package -DskipTests -q

# Stage 2: Runtime
FROM openjdk:8-jre-slim

# ติดตั้ง dependencies ที่จำเป็น
RUN apt-get update && apt-get install -y \
    curl \
    netcat \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy artifact จาก build stage
COPY --from=builder /app/target/legacy-app.jar .

# Copy configuration
COPY config/ config/
COPY scripts/start.sh .

# สร้าง non-root user
RUN useradd -r -u 1001 -g root appuser
RUN chown -R appuser:root /app
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Start script
ENTRYPOINT ["./start.sh"]
```

```bash
#!/bin/bash
# scripts/start.sh - Start script สำหรับ legacy app

set -e

# แปลง environment variables เป็น config format ที่ legacy app ใช้
cat > /app/config/database.properties << EOF
db.host=${DB_HOST:-localhost}
db.port=${DB_PORT:-5432}
db.name=${DB_NAME:-mydb}
db.user=${DB_USER:-admin}
db.password=${DB_PASSWORD:-secret}
db.pool.min=${DB_POOL_MIN:-5}
db.pool.max=${DB_POOL_MAX:-20}
EOF

cat > /app/config/app.properties << EOF
app.port=8080
app.log.level=${LOG_LEVEL:-INFO}
app.gateway.url=${GATEWAY_URL:-http://payment-gateway}
app.feature.new_checkout=${FEATURE_NEW_CHECKOUT:-false}
EOF

# รอให้ database พร้อม
echo "Waiting for database..."
until nc -z "$DB_HOST" "${DB_PORT:-5432}"; do
  echo "Database not ready, waiting..."
  sleep 2
done
echo "Database is ready"

# Start application
echo "Starting legacy application..."
exec java \
  -Xmx${JAVA_MAX_HEAP:-512m} \
  -Xms${JAVA_MIN_HEAP:-256m} \
  -XX:+UseG1GC \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -Djava.security.egd=file:/dev/./urandom \
  -Dapp.config.dir=/app/config \
  -jar legacy-app.jar
```

### Kubernetes Deployment สำหรับ Legacy App

```yaml
# k8s/legacy-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: legacy-payment-app
  namespace: legacy
  labels:
    app: legacy-payment-app
    migration-status: "in-progress"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-payment-app
  strategy:
    # ใช้ RollingUpdate แทน Recreate เพื่อ zero downtime
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0  # ห้าม unavailable ระหว่าง update
      maxSurge: 1        # อนุญาตให้มี 1 pod เพิ่มระหว่าง update
  template:
    metadata:
      labels:
        app: legacy-payment-app
    spec:
      # ใช้ Init Container เพื่อรอ dependencies
      initContainers:
        - name: wait-for-db
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              until nc -z $DB_HOST $DB_PORT; do
                echo "Waiting for database..."
                sleep 2
              done
              echo "Database is ready"
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: legacy-db-secret
                  key: host
            - name: DB_PORT
              value: "5432"

      containers:
        - name: legacy-app
          image: ghcr.io/mycompany/legacy-payment:v2.1.0
          ports:
            - containerPort: 8080

          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: legacy-db-secret
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: legacy-db-secret
                  key: password
            - name: JAVA_MAX_HEAP
              value: "1024m"
            - name: LOG_LEVEL
              value: "INFO"

          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "2Gi"

          # Readiness probe - ยาวขึ้นสำหรับ legacy app
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 120  # Legacy app ช้าในการ start
            periodSeconds: 10
            failureThreshold: 6
            timeoutSeconds: 5

          # Liveness probe
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 180
            periodSeconds: 30
            failureThreshold: 3

          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - "sleep 15"  # รอให้ in-flight requests เสร็จ

          volumeMounts:
            - name: logs
              mountPath: /app/logs
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: logs
          emptyDir: {}
        - name: tmp
          emptyDir: {}

      terminationGracePeriodSeconds: 60
```

---

## 80.5 CI/CD Pipeline สำหรับ Legacy

### Pipeline ที่ไม่ Break Legacy

```yaml
# .github/workflows/legacy-cicd.yaml
name: Legacy System CI/CD

on:
  push:
    branches: [main, legacy-maintenance]

jobs:
  # ขั้นตอนที่ 1: Build (อาจนาน)
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java 8
        uses: actions/setup-java@v4
        with:
          java-version: '8'
          distribution: 'temurin'

      - name: Cache Maven packages
        uses: actions/cache@v4
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}

      - name: Build
        run: mvn package -DskipTests

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: legacy-app
          path: target/legacy-app.jar

  # ขั้นตอนที่ 2: Tests (Characterization + new unit tests)
  test:
    runs-on: ubuntu-latest
    needs: build
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s

    steps:
      - uses: actions/checkout@v4

      - name: Setup Java 8
        uses: actions/setup-java@v4
        with:
          java-version: '8'
          distribution: 'temurin'

      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: legacy-app

      - name: Run characterization tests
        run: |
          # รัน tests กับ legacy app จริง (integration tests)
          java -jar legacy-app.jar &
          sleep 30  # รอให้ start

          python -m pytest tests/characterization/ -v \
            --html=report.html \
            --self-contained-html

      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-report
          path: report.html

  # ขั้นตอนที่ 3: Static Analysis
  static-analysis:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java 8
        uses: actions/setup-java@v4
        with:
          java-version: '8'
          distribution: 'temurin'

      # Checkstyle สำหรับ code style
      - name: Checkstyle
        run: mvn checkstyle:check
        continue-on-error: true  # ไม่ fail pipeline เพราะ legacy code อาจ violate rules

      # SpotBugs สำหรับ bug patterns
      - name: SpotBugs
        run: mvn spotbugs:check
        continue-on-error: true

      # OWASP Dependency Check
      - name: Dependency Check
        run: |
          mvn dependency-check:check \
            --fail-cvss-score 9.0 \  # Fail แค่ critical vulnerabilities
            --format HTML

  # ขั้นตอนที่ 4: Docker Build
  docker-build:
    runs-on: ubuntu-latest
    needs: [build, test]
    steps:
      - uses: actions/checkout@v4

      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: legacy-app
          path: target/

      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.legacy-java
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: |
            ghcr.io/mycompany/legacy-payment:latest
            ghcr.io/mycompany/legacy-payment:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ขั้นตอนที่ 5: Deploy (conservative strategy)
  deploy-staging:
    runs-on: ubuntu-latest
    needs: docker-build
    if: github.ref == 'refs/heads/main'
    environment: legacy-staging
    steps:
      - name: Deploy to staging
        run: |
          # Deploy แบบ conservative - canary 10% ก่อน
          kubectl set image deployment/legacy-payment-app \
            legacy-app=ghcr.io/mycompany/legacy-payment:${{ github.sha }} \
            -n legacy-staging

          # รอให้ healthy
          kubectl rollout status deployment/legacy-payment-app \
            -n legacy-staging --timeout=600s

      - name: Run smoke tests
        run: |
          # ทดสอบ endpoints หลัก
          BASE_URL="https://legacy-staging.mycompany.com"

          curl -f "$BASE_URL/health"
          curl -f "$BASE_URL/api/v1/ping"

          # ทดสอบ payment flow
          RESULT=$(curl -s -X POST "$BASE_URL/api/v1/payments/test" \
            -H "Content-Type: application/json" \
            -d '{"amount": 1.00, "currency": "THB"}')

          if echo "$RESULT" | grep -q '"success":true'; then
            echo "✅ Payment smoke test passed"
          else
            echo "❌ Payment smoke test failed: $RESULT"
            exit 1
          fi

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: legacy-production  # ต้องการ manual approval
    steps:
      - name: Deploy to production (rolling update)
        run: |
          kubectl set image deployment/legacy-payment-app \
            legacy-app=ghcr.io/mycompany/legacy-payment:${{ github.sha }} \
            -n legacy-production

          kubectl rollout status deployment/legacy-payment-app \
            -n legacy-production --timeout=900s

      - name: Monitor for 15 minutes
        run: |
          END_TIME=$(($(date +%s) + 900))
          while [ $(date +%s) -lt $END_TIME ]; do
            ERROR_RATE=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
              --data-urlencode 'query=rate(legacy_app_errors_total[5m])' \
              | jq -r '.data.result[0].value[1] // "0"')

            if (( $(echo "$ERROR_RATE > 0.1" | bc -l) )); then
              echo "❌ Error rate high: $ERROR_RATE - rolling back"
              kubectl rollout undo deployment/legacy-payment-app -n legacy-production
              exit 1
            fi
            sleep 60
          done
          echo "✅ Production deployment stable"
```

---

## 80.6 Migration Strategy

### Phased Migration Plan

```markdown
# Legacy to Microservices Migration Plan

## Overview
Timeline: 18 months
Approach: Strangler Fig Pattern
Risk Level: Managed (phased approach)

## Phase 1: Stabilization (Month 1-3)
Goal: ทำให้ legacy system มี CI/CD

Tasks:
- [ ] Containerize legacy app
- [ ] สร้าง CI/CD pipeline พื้นฐาน
- [ ] เพิ่ม characterization tests
- [ ] ตั้งค่า monitoring & alerting
- [ ] สร้าง staging environment

Success Criteria:
- Deploy อัตโนมัติ
- Rollback ได้ใน < 5 นาที
- Test coverage > 30%

## Phase 2: Extract Authentication (Month 4-6)
Goal: แยก auth ออกเป็น microservice แรก

Tasks:
- [ ] Identify auth-related code
- [ ] สร้าง new auth service
- [ ] ตั้งค่า proxy routing
- [ ] Migrate 10% → 50% → 100% traffic
- [ ] Remove auth code จาก monolith

Success Criteria:
- Auth service deployed independently
- Zero downtime migration
- Response time ไม่เพิ่มขึ้น > 10%

## Phase 3: Extract Product Catalog (Month 7-9)
Goal: แยก product catalog ออก

## Phase 4: Extract Order Management (Month 10-12)
Goal: แยก order management ออก

## Phase 5: Extract Payment Processing (Month 13-15)
Goal: แยก payment ออก (critical - ต้องระวังมาก)

## Phase 6: Decommission Legacy (Month 16-18)
Goal: ปิด legacy system

Tasks:
- [ ] ตรวจสอบว่าไม่มี traffic ไปยัง legacy
- [ ] ทำ final data migration
- [ ] Archive legacy code
- [ ] ปิด legacy infrastructure
```

### Database Migration สำหรับ Legacy

```python
# migration/database_migrator.py
"""
Migrator สำหรับย้ายข้อมูลจาก legacy database schema
ไปยัง new microservice databases
"""
import psycopg2
from datetime import datetime
from typing import Iterator
import logging

log = logging.getLogger(__name__)


class LegacyDataMigrator:
    def __init__(
        self,
        legacy_conn_string: str,
        users_service_conn_string: str,
        batch_size: int = 1000,
    ):
        self.legacy_conn = psycopg2.connect(legacy_conn_string)
        self.users_conn = psycopg2.connect(users_service_conn_string)
        self.batch_size = batch_size

    def migrate_users(self) -> dict:
        """ย้าย users จาก legacy ไปยัง new users service"""

        stats = {"migrated": 0, "skipped": 0, "errors": 0}

        log.info("Starting users migration...")

        for batch in self._fetch_users_batch():
            for user in batch:
                try:
                    self._insert_user_to_new_db(user)
                    stats["migrated"] += 1
                except DuplicateKeyError:
                    stats["skipped"] += 1
                except Exception as e:
                    log.error(f"Error migrating user {user['id']}: {e}")
                    stats["errors"] += 1

            self.users_conn.commit()
            log.info(f"Migrated {stats['migrated']} users so far...")

        log.info(f"Migration complete: {stats}")
        return stats

    def _fetch_users_batch(self) -> Iterator[list]:
        """ดึง users จาก legacy DB เป็น batch"""
        cursor = self.legacy_conn.cursor()
        offset = 0

        while True:
            cursor.execute("""
                SELECT
                    id,
                    username,
                    email,
                    password_hash,
                    first_name,
                    last_name,
                    phone,
                    created_at,
                    last_login,
                    status
                FROM users
                WHERE deleted_at IS NULL
                ORDER BY id
                LIMIT %s OFFSET %s
            """, (self.batch_size, offset))

            rows = cursor.fetchall()
            if not rows:
                break

            yield [
                {
                    "id": row[0],
                    "username": row[1],
                    "email": row[2],
                    "password_hash": row[3],
                    "first_name": row[4],
                    "last_name": row[5],
                    "phone": row[6],
                    "created_at": row[7],
                    "last_login": row[8],
                    "status": row[9],
                }
                for row in rows
            ]

            offset += self.batch_size

    def _insert_user_to_new_db(self, user: dict) -> None:
        """Insert user ลงใน new users service DB"""
        cursor = self.users_conn.cursor()

        cursor.execute("""
            INSERT INTO users (
                id, username, email, password_hash,
                first_name, last_name, phone,
                created_at, last_login_at, is_active
            ) VALUES (
                %s, %s, %s, %s,
                %s, %s, %s,
                %s, %s, %s
            )
            ON CONFLICT (id) DO NOTHING
        """, (
            user["id"],
            user["username"],
            user["email"],
            user["password_hash"],
            user["first_name"],
            user["last_name"],
            user["phone"],
            user["created_at"],
            user["last_login"],
            user["status"] == "active",
        ))

    def verify_migration(self) -> dict:
        """ตรวจสอบว่า migration สำเร็จ"""
        legacy_count = self._count_legacy_users()
        new_count = self._count_new_users()

        return {
            "legacy_count": legacy_count,
            "new_count": new_count,
            "match": legacy_count == new_count,
            "migration_rate": new_count / legacy_count * 100 if legacy_count > 0 else 0,
        }
```

---

## 80.7 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Containerize Legacy App

```dockerfile
# exercises/Dockerfile.legacy
# Containerize แอปเก่าต่อไปนี้:
# - Python 2.7 Flask application
# - ต้องการ libpq-dev สำหรับ PostgreSQL
# - Config อยู่ใน /etc/myapp/config.ini
# - Port 5000

# TODO: สร้าง Dockerfile ที่:
# 1. ใช้ python:2.7-slim เป็น base
# 2. ติดตั้ง dependencies
# 3. Copy แอป
# 4. Convert environment variables เป็น config.ini
# 5. มี health check
# 6. รัน แบบ non-root user

FROM python:2.7-slim

# TODO: implement
```

### แบบฝึกหัดที่ 2: Characterization Tests

```python
# exercises/characterization_tests.py
# สร้าง characterization tests สำหรับ legacy function นี้

def calculate_order_total(items, discount_code=None, tax_rate=0.07):
    """
    Legacy function ที่คำนวณ order total
    ไม่มี tests และ logic ซับซ้อน
    """
    subtotal = 0
    for item in items:
        if item.get('on_sale'):
            price = item['price'] * 0.9  # 10% sale discount
        else:
            price = item['price']
        subtotal += price * item['quantity']

    # Apply discount code
    if discount_code:
        if discount_code.startswith('SAVE'):
            discount_pct = int(discount_code[4:]) / 100
            subtotal *= (1 - discount_pct)
        elif discount_code == 'HALFOFF':
            subtotal *= 0.5
        # Note: other discount codes are silently ignored

    # Tax
    tax = subtotal * tax_rate
    total = subtotal + tax

    # Rounding (legacy code rounds differently)
    return round(total * 100) / 100  # round to cents

# TODO: สร้าง characterization tests ที่ document behavior ทั้งหมด
# ทดสอบ:
# 1. Normal order without discount
# 2. Sale item
# 3. SAVE20 discount code
# 4. HALFOFF discount code
# 5. Unknown discount code
# 6. Mixed sale and non-sale items
# 7. Different tax rates

import pytest

class TestCalculateOrderTotal:
    def test_normal_order(self):
        items = [{"price": 100, "quantity": 2, "on_sale": False}]
        result = calculate_order_total(items)
        # TODO: document what the result should be
        assert result == ???  # fill in expected value

    # TODO: เพิ่ม test cases อื่นๆ
```

### แบบฝึกหัดที่ 3: Strangler Fig Implementation

```yaml
# exercises/strangler-nginx.conf
# สร้าง Nginx config สำหรับ Strangler Fig Pattern

# Scenario:
# - Legacy monolith รัน ที่ legacy-app:8080
# - New user service รัน ที่ user-service:3000
# - New product service รัน ที่ product-service:3000
# - ทุก path อื่นๆ ไป legacy

# TODO: สร้าง Nginx config ที่:
# 1. /api/v2/users/* → new user-service
# 2. /api/v2/products/* → new product-service
# 3. ทุกอย่างอื่น → legacy-app

upstream legacy_app {
    server legacy-app:8080;
}

upstream user_service {
    server user-service:3000;
}

upstream product_service {
    server product-service:3000;
}

server {
    listen 80;

    # TODO: เพิ่ม location blocks
}
```

---

## สรุป

CI/CD for Legacy Systems ต้องใช้แนวทางที่แตกต่างจาก greenfield projects:

1. **Stabilize First** ก่อน modernize ต้องทำให้ legacy system มี basic CI/CD ก่อน
2. **Characterization Tests** บันทึก current behavior เพื่อป้องกัน regression
3. **Strangler Fig** เป็น safe pattern สำหรับ incremental migration
4. **Containerization** เป็น first step สำคัญในการ modernize infrastructure
5. **Never Stop Delivering** migration ต้องไม่หยุด business
6. **Patience** legacy modernization ใช้เวลา 1-3 ปี ไม่ใช่ weeks

Key insight: ทุก legacy system เคยเป็น modern system มาก่อน systems ที่เราสร้างวันนี้จะกลายเป็น legacy ในอนาคต

### ขั้นตอนถัดไป

- ศึกษา [Part 81: GitOps Advanced Patterns](../part-81-90/part-81-gitops-advanced.md)
- Working Effectively with Legacy Code (book) โดย Michael Feathers
- Strangler Fig Pattern: https://martinfowler.com/bliki/StranglerFigApplication.html

---

## Appendix: Migration Checklist

```markdown
# Legacy System Migration Checklist

## Before Starting
- [ ] ทำ inventory ของ legacy system ทั้งหมด
- [ ] Document current architecture
- [ ] สัมภาษณ์ทีมที่รู้จักระบบ
- [ ] ประเมิน business risk
- [ ] กำหนด success metrics

## Phase 1: Stabilization
- [ ] Containerize application
- [ ] สร้าง basic CI pipeline
- [ ] ตั้งค่า staging environment
- [ ] สร้าง characterization tests
- [ ] ตั้งค่า monitoring

## Phase 2+: Migration
- [ ] กำหนด feature ที่จะ extract
- [ ] สร้าง new service
- [ ] ตั้งค่า proxy routing
- [ ] Test thoroughly
- [ ] Gradual traffic migration
- [ ] Remove from legacy

## Final Decommission
- [ ] ยืนยันว่าไม่มี traffic ไปยัง legacy
- [ ] Final data migration
- [ ] Archive legacy code & documentation
- [ ] ปิด infrastructure
- [ ] Post-migration review
```

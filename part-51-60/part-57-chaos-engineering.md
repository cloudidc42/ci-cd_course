# Part 57: Chaos Engineering

## บทนำ

Chaos Engineering คือแนวปฏิบัติในการทดสอบความแข็งแกร่งของ system โดยการ "จงใจสร้างความผิดพลาด" ในสภาพแวดล้อมที่ควบคุมได้ Netflix เป็นผู้บุกเบิกแนวคิดนี้ด้วยการสร้าง Chaos Monkey ซึ่ง random terminate production instances

**หลักการสำคัญ**: "ถ้าเราไม่ทดสอบว่า system รับมือกับความผิดพลาดได้ ก็แปลว่าเราไม่รู้ว่ามันทำได้"

บทนี้จะครอบคลุม:
- Chaos Engineering Principles
- Chaos Monkey
- LitmusChaos
- Hypothesis-Driven Experiments
- Blast Radius Management
- CI/CD Integration
- Game Days

---

## 1. Chaos Engineering Principles

### 1.1 หลักการ 5 ข้อของ Chaos Engineering

```
1. Establish a Steady State
   วัด normal behavior ก่อน
   เช่น: 99.9% success rate, p95 latency < 200ms

2. Hypothesize what happens
   ตั้งสมมติฐาน เช่น: "ถ้า database ช้าลง 200ms
   application ยังตอบสนองได้ภายใน 1 วินาที"

3. Introduce real-world events
   Inject failures: node failure, network latency,
   disk full, memory pressure, etc.

4. Disprove the Hypothesis
   วัดว่า steady state เปลี่ยนไปหรือไม่
   ถ้า hypothesis ผิด = ค้นพบ weakness

5. Minimize Blast Radius
   เริ่มด้วย scope เล็ก ๆ
   เพิ่มขึ้นเรื่อย ๆ ตามความมั่นใจ
```

### 1.2 ประเภทของ Chaos Experiments

```
Infrastructure Level:
├── Node failures (kill node/VM)
├── Network partitions
├── Disk I/O failures
├── CPU/Memory exhaustion
└── Power failures

Application Level:
├── Service crashes
├── Slow responses (latency injection)
├── Error responses (HTTP 5xx)
├── Dependency failures
└── Memory leaks

Data Level:
├── Database slow queries
├── Cache misses
├── Data corruption (carefully!)
└── Backup restoration failures

Security Level:
├── Certificate expiry
├── Permission changes
└── Network policy violations
```

---

## 2. Chaos Monkey

### 2.1 Chaos Monkey สำหรับ Kubernetes

```bash
# ติดตั้ง kube-monkey (Chaos Monkey สำหรับ Kubernetes)
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-monkey
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-monkey
  template:
    metadata:
      labels:
        app: kube-monkey
    spec:
      containers:
        - name: kube-monkey
          image: asobti/kube-monkey:v0.3.0
          env:
            - name: NAMESPACE
              value: "production"  # namespace ที่จะทำ chaos
            - name: DRY_RUN
              value: "false"
            - name: TIMEZONE
              value: "Asia/Bangkok"
EOF
```

### 2.2 Opt-in pods สำหรับ Chaos Monkey

```yaml
# ทุก deployment ที่ต้องการ chaos testing ต้องมี labels เหล่านี้
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    kube-monkey/enabled: "enabled"           # เปิดใช้ chaos monkey
    kube-monkey/identifier: "myapp"          # identifier ที่ unique
    kube-monkey/mtbf: "8"                    # Mean Time Between Failures (hours)
    kube-monkey/kill-mode: "fixed"           # fixed หรือ percentage
    kube-monkey/kill-value: "1"              # ฆ่ 1 pod ต่อครั้ง
spec:
  replicas: 3  # ต้องมี replicas มากพอ
  # ...
```

### 2.3 Custom Chaos Script

```bash
#!/bin/bash
# scripts/chaos-pod-killer.sh
# Script สำหรับ kill random pods ใน namespace

set -euo pipefail

NAMESPACE="${1:-staging}"
KILL_COUNT="${2:-1}"
LABEL_SELECTOR="${3:-app}"
DRY_RUN="${4:-false}"

echo "=== Chaos Pod Killer ==="
echo "Namespace: ${NAMESPACE}"
echo "Kill Count: ${KILL_COUNT}"
echo "Dry Run: ${DRY_RUN}"

# ดึงรายชื่อ pods ที่ running
PODS=$(kubectl get pods \
    -n "${NAMESPACE}" \
    -l "${LABEL_SELECTOR}" \
    --field-selector=status.phase=Running \
    -o jsonpath='{.items[*].metadata.name}' \
    2>/dev/null | tr ' ' '\n')

POD_COUNT=$(echo "${PODS}" | wc -l | tr -d ' ')
echo "Found ${POD_COUNT} running pods"

if [ "${POD_COUNT}" -lt 2 ]; then
    echo "❌ Not enough pods (minimum 2 required)"
    exit 1
fi

# Random select pods to kill
PODS_TO_KILL=$(echo "${PODS}" | shuf -n "${KILL_COUNT}")

for pod in ${PODS_TO_KILL}; do
    if [ "${DRY_RUN}" == "true" ]; then
        echo "DRY RUN: Would kill pod ${pod}"
    else
        echo "Killing pod: ${pod}"
        kubectl delete pod "${pod}" -n "${NAMESPACE}" --grace-period=0
        echo "✅ Pod ${pod} killed"
    fi
done

# Wait และตรวจสอบว่า pods ถูก recreate
if [ "${DRY_RUN}" != "true" ]; then
    echo "Waiting for pods to recover..."
    sleep 10
    
    # ตรวจสอบ pod count
    NEW_POD_COUNT=$(kubectl get pods \
        -n "${NAMESPACE}" \
        -l "${LABEL_SELECTOR}" \
        --field-selector=status.phase=Running \
        -o jsonpath='{.items[*].metadata.name}' \
        2>/dev/null | tr ' ' '\n' | wc -l | tr -d ' ')
    
    echo "Running pods after chaos: ${NEW_POD_COUNT}/${POD_COUNT}"
    
    if [ "${NEW_POD_COUNT}" -ge "${POD_COUNT}" ]; then
        echo "✅ System recovered successfully"
    else
        echo "⚠️  System may not have fully recovered"
    fi
fi
```

---

## 3. LitmusChaos

### 3.1 ติดตั้ง LitmusChaos

```bash
# ติดตั้ง LitmusChaos ด้วย Helm
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update

helm install chaos litmuschaos/litmus \
    --namespace litmus \
    --create-namespace \
    --set portal.frontend.service.type=LoadBalancer

# ตรวจสอบ
kubectl get pods -n litmus

# Access LitmusChaos Portal
PORTAL_URL=$(kubectl get service chaos-litmus-frontend-service \
    -n litmus \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Portal: http://${PORTAL_URL}:9091"
```

### 3.2 สร้าง ChaosExperiment

```yaml
# litmus/experiments/pod-delete-experiment.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosExperiment
metadata:
  name: pod-delete
  namespace: production
spec:
  definition:
    scope: Namespaced
    permissions:
      - apiGroups: [""]
        resources: ["pods"]
        verbs: ["create", "delete", "get", "list", "patch", "update", "deletecollection"]
      - apiGroups: ["apps"]
        resources: ["deployments", "replicasets"]
        verbs: ["get", "list", "patch", "update"]
    image: litmuschaos/go-runner:latest
    imagePullPolicy: Always
    args:
      - -c
      - ./experiments -name pod-delete
    command:
      - /bin/bash
    env:
      - name: TOTAL_CHAOS_DURATION
        value: "60"          # ระยะเวลา chaos (วินาที)
      - name: CHAOS_INTERVAL
        value: "10"          # interval ระหว่าง kills (วินาที)
      - name: FORCE
        value: "false"       # graceful kill
      - name: PODS_AFFECTED_PERC
        value: "50"          # ฆ่ 50% ของ pods
```

### 3.3 สร้าง ChaosEngine

```yaml
# litmus/chaos-engine/production-chaos.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: production-chaos-engine
  namespace: production
spec:
  # Target application
  appinfo:
    appns: production
    applabel: "app=myapp"
    appkind: deployment
  
  # ยกเลิก annotation requirement
  annotationCheck: "false"
  
  # Chaos experiments to run
  chaosServiceAccount: chaos-service-account
  
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "false"
        
        # Probe สำหรับตรวจสอบ steady state
        probe:
          - name: check-http-probe
            type: httpProbe
            httpProbe/inputs:
              url: "http://myapp:80/health"
              insecureSkipVerify: false
              method:
                get:
                  criteria: "=="
                  responseCode: "200"
            mode: Continuous
            runProperties:
              probeTimeout: 5
              interval: 5
              attempt: 1
              initialDelaySeconds: 0
          
          - name: check-pod-count
            type: cmdProbe
            cmdProbe/inputs:
              command: "kubectl get pods -n production -l app=myapp --field-selector=status.phase=Running | wc -l"
              comparator:
                type: int
                criteria: ">="
                value: "2"
            mode: Continuous
            runProperties:
              probeTimeout: 10
              interval: 10
              attempt: 1
```

### 3.4 Network Chaos

```yaml
# litmus/experiments/network-latency.yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: network-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=backend-api"
    appkind: deployment
  
  chaosServiceAccount: chaos-service-account
  
  experiments:
    # Inject network latency
    - name: pod-network-latency
      spec:
        components:
          env:
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: NETWORK_LATENCY
              value: "2000"         # 2000ms latency
            - name: JITTER
              value: "200"          # ±200ms jitter
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: PODS_AFFECTED_PERC
              value: "100"          # ใช้กับทุก pods
        
        probe:
          - name: check-response-time
            type: httpProbe
            httpProbe/inputs:
              url: "http://backend-api:3000/api/users"
              method:
                get:
                  criteria: "=="
                  responseCode: "200"
            mode: Continuous
            runProperties:
              probeTimeout: 5000    # 5 วินาที timeout (เพราะมี 2s latency)
              interval: 10

    # Inject network packet loss
    - name: pod-network-loss
      spec:
        components:
          env:
            - name: NETWORK_INTERFACE
              value: "eth0"
            - name: NETWORK_PACKET_LOSS_PERCENTAGE
              value: "30"           # 30% packet loss
            - name: TOTAL_CHAOS_DURATION
              value: "30"
```

---

## 4. Hypothesis-Driven Experiments

### 4.1 Chaos Experiment Template

```markdown
# Chaos Experiment: [ชื่อ Experiment]

## Hypothesis
"ถ้า [เหตุการณ์ที่เกิดขึ้น], ระบบ [ผลลัพธ์ที่คาดหวัง]"

ตัวอย่าง: "ถ้า database node หนึ่งในสามล้มเหลว, 
application ยังคงให้บริการได้ด้วย availability >= 99%"

## Context
- **System**: Production Kubernetes Cluster
- **Service**: [ชื่อ Service]
- **Date**: [วันที่]
- **Team**: [ทีมที่รับผิดชอบ]

## Steady State
วัด metrics ก่อน experiment:
- Success rate: 99.95%
- p95 latency: 180ms
- p99 latency: 450ms
- Error rate: 0.05%

## Experiment Design
- **Type**: [Pod Delete / Network Latency / etc.]
- **Blast Radius**: [กี่ % ของ system]
- **Duration**: [กี่นาที]
- **Rollback Plan**: [วิธี stop ถ้าเกิดปัญหา]

## Method
1. เริ่ม monitoring
2. Inject [ประเภท fault]
3. รอ [X นาที]
4. วัด metrics
5. Stop experiment
6. ตรวจสอบ recovery

## Observations
[บันทึกสิ่งที่พบระหว่าง experiment]

## Results
- Hypothesis: [Confirmed / Rejected]
- สิ่งที่ค้นพบ: [...]
- Impact: [...]

## Follow-up Actions
- [ ] [Action 1]
- [ ] [Action 2]
```

### 4.2 ตัวอย่าง Experiment ใน Python

```python
# scripts/chaos_experiment.py
"""
Framework สำหรับรัน chaos experiments แบบ structured
"""
import json
import time
import subprocess
import statistics
from dataclasses import dataclass, field
from datetime import datetime, timezone
from typing import Callable, List, Optional
import urllib.request


@dataclass
class SteadyState:
    """วัด steady state metrics"""
    success_rate: float
    p95_latency_ms: float
    error_rate: float
    custom_metrics: dict = field(default_factory=dict)


@dataclass
class ExperimentResult:
    """ผลลัพธ์ของ experiment"""
    hypothesis: str
    confirmed: bool
    steady_state_before: SteadyState
    steady_state_after: Optional[SteadyState]
    observations: List[str]
    duration_seconds: float
    timestamp: str


def measure_steady_state(service_url: str, samples: int = 10) -> SteadyState:
    """วัด steady state ของ service"""
    latencies = []
    errors = 0
    
    for _ in range(samples):
        start = time.time()
        try:
            req = urllib.request.Request(f"{service_url}/health")
            with urllib.request.urlopen(req, timeout=10) as resp:
                if resp.status != 200:
                    errors += 1
        except Exception:
            errors += 1
        
        latencies.append((time.time() - start) * 1000)
        time.sleep(0.5)
    
    sorted_latencies = sorted(latencies)
    p95_index = int(len(sorted_latencies) * 0.95)
    
    return SteadyState(
        success_rate=(samples - errors) / samples * 100,
        p95_latency_ms=sorted_latencies[p95_index],
        error_rate=errors / samples * 100
    )


class ChaosExperiment:
    """Chaos Experiment runner"""
    
    def __init__(self, name: str, hypothesis: str, service_url: str):
        self.name = name
        self.hypothesis = hypothesis
        self.service_url = service_url
        self.observations = []
    
    def run_experiment(
        self,
        inject_chaos: Callable,
        stop_chaos: Callable,
        duration_seconds: int = 60,
        verification_interval: int = 10
    ) -> ExperimentResult:
        """รัน chaos experiment"""
        
        print(f"\n=== Chaos Experiment: {self.name} ===")
        print(f"Hypothesis: {self.hypothesis}")
        
        # 1. วัด Steady State ก่อน
        print("\n📊 Measuring steady state...")
        steady_state_before = measure_steady_state(self.service_url)
        print(f"Before: success={steady_state_before.success_rate:.1f}%, "
              f"p95={steady_state_before.p95_latency_ms:.0f}ms")
        
        # 2. Inject Chaos
        print(f"\n💥 Injecting chaos for {duration_seconds}s...")
        start_time = time.time()
        
        try:
            inject_chaos()
            
            # 3. Monitor during chaos
            check_times = []
            success_during_chaos = []
            
            elapsed = 0
            while elapsed < duration_seconds:
                time.sleep(verification_interval)
                elapsed = time.time() - start_time
                
                # Quick health check
                try:
                    req = urllib.request.Request(f"{self.service_url}/health")
                    with urllib.request.urlopen(req, timeout=5) as resp:
                        success = resp.status == 200
                except Exception:
                    success = False
                
                success_during_chaos.append(success)
                status = "✅" if success else "❌"
                obs = f"t={elapsed:.0f}s: {status}"
                self.observations.append(obs)
                print(obs)
        
        finally:
            # 4. Stop Chaos
            print("\n🛑 Stopping chaos...")
            stop_chaos()
        
        # 5. รอให้ system recover
        print("Waiting for system to recover...")
        time.sleep(30)
        
        # 6. วัด Steady State หลัง
        print("\n📊 Measuring steady state after chaos...")
        steady_state_after = measure_steady_state(self.service_url)
        print(f"After: success={steady_state_after.success_rate:.1f}%, "
              f"p95={steady_state_after.p95_latency_ms:.0f}ms")
        
        # 7. ตรวจสอบ Hypothesis
        # Hypothesis: success rate ยังคง >= 95% ระหว่าง chaos
        chaos_success_rate = (
            sum(success_during_chaos) / len(success_during_chaos) * 100
            if success_during_chaos else 0
        )
        
        hypothesis_confirmed = (
            chaos_success_rate >= 95.0 and
            steady_state_after.success_rate >= 99.0
        )
        
        result = ExperimentResult(
            hypothesis=self.hypothesis,
            confirmed=hypothesis_confirmed,
            steady_state_before=steady_state_before,
            steady_state_after=steady_state_after,
            observations=self.observations,
            duration_seconds=time.time() - start_time,
            timestamp=datetime.now(timezone.utc).isoformat()
        )
        
        self._print_results(result, chaos_success_rate)
        return result
    
    def _print_results(self, result: ExperimentResult, chaos_success_rate: float):
        """แสดงผลลัพธ์"""
        print("\n" + "="*50)
        print(f"EXPERIMENT RESULTS: {self.name}")
        print("="*50)
        
        status = "✅ CONFIRMED" if result.confirmed else "❌ REJECTED"
        print(f"Hypothesis: {status}")
        print(f"Success rate during chaos: {chaos_success_rate:.1f}%")
        print(f"Success rate after: {result.steady_state_after.success_rate:.1f}%")
        
        if not result.confirmed:
            print("\n⚠️  System did not meet resilience requirements!")
            print("Actions needed:")
            print("  1. Review circuit breakers")
            print("  2. Check retry policies")
            print("  3. Consider increasing replicas")
        
        # Save results
        with open(f"chaos-result-{self.name.replace(' ', '-')}.json", 'w') as f:
            json.dump({
                "name": self.name,
                "hypothesis": result.hypothesis,
                "confirmed": result.confirmed,
                "chaos_success_rate": chaos_success_rate,
                "steady_state_before": {
                    "success_rate": result.steady_state_before.success_rate,
                    "p95_latency_ms": result.steady_state_before.p95_latency_ms
                },
                "steady_state_after": {
                    "success_rate": result.steady_state_after.success_rate,
                    "p95_latency_ms": result.steady_state_after.p95_latency_ms
                } if result.steady_state_after else None,
                "observations": result.observations,
                "timestamp": result.timestamp
            }, f, indent=2)
        
        print(f"\nResults saved to chaos-result-{self.name.replace(' ', '-')}.json")


# ตัวอย่างการใช้งาน
def inject_pod_failure():
    """Kill random pod"""
    result = subprocess.run(
        ["kubectl", "get", "pods", "-n", "staging",
         "-l", "app=myapp", "--no-headers",
         "-o", "custom-columns=NAME:.metadata.name"],
        capture_output=True, text=True
    )
    pods = result.stdout.strip().split('\n')
    if pods:
        pod_to_kill = pods[0]
        subprocess.run(
            ["kubectl", "delete", "pod", pod_to_kill,
             "-n", "staging", "--grace-period=0"]
        )
        print(f"Killed pod: {pod_to_kill}")


def stop_pod_failure():
    """ไม่ต้องทำอะไร เพราะ Kubernetes จะ recreate pod อัตโนมัติ"""
    print("Waiting for Kubernetes to recreate pods...")


if __name__ == '__main__':
    experiment = ChaosExperiment(
        name="Pod Failure Resilience",
        hypothesis="ถ้า pod หนึ่งถูก kill, system ยังให้บริการได้ด้วย success rate >= 95%",
        service_url="http://myapp-service.staging.svc.cluster.local"
    )
    
    result = experiment.run_experiment(
        inject_chaos=inject_pod_failure,
        stop_chaos=stop_pod_failure,
        duration_seconds=60
    )
```

---

## 5. Blast Radius Management

### 5.1 Progressive Chaos Strategy

```
Start Small → Grow Gradually → Learn → Repeat

Stage 1: Development Environment
├── Aggressive chaos (high blast radius)
├── All types of failures
└── No traffic impact

Stage 2: Staging Environment
├── Moderate chaos
├── Specific failure scenarios
└── Limited real users

Stage 3: Production (Canary)
├── Minimal chaos (1-5% users)
├── Well-defined experiments
└── Full monitoring

Stage 4: Production (Full)
├── Scheduled experiments
├── Automated game days
└── Continuous validation
```

### 5.2 Chaos Limits Configuration

```yaml
# litmus/chaos-limits.yaml
# กำหนด limits เพื่อ control blast radius

apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: controlled-chaos
  namespace: production
spec:
  appinfo:
    appns: production
    applabel: "app=myapp"
    appkind: deployment
  
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            # จำกัด blast radius
            - name: PODS_AFFECTED_PERC
              value: "25"      # สูงสุด 25% ของ pods
            
            - name: TOTAL_CHAOS_DURATION
              value: "30"      # สูงสุด 30 วินาที
            
            # Pause ระหว่าง each kill
            - name: CHAOS_INTERVAL
              value: "15"
        
        # Probe ที่ต้อง pass ตลอด
        probe:
          - name: availability-probe
            type: httpProbe
            httpProbe/inputs:
              url: "http://myapp/health"
              method:
                get:
                  criteria: "=="
                  responseCode: "200"
            mode: Continuous
            runProperties:
              probeTimeout: 5
              interval: 2
              # ถ้า probe fail 3 ครั้งติดกัน ให้หยุด chaos
              attempt: 3
              probePollingInterval: 2
```

---

## 6. CI/CD Integration กับ Chaos

### 6.1 Chaos ใน Deployment Pipeline

```yaml
# .github/workflows/chaos-testing.yml
name: Chaos Testing in Pipeline

on:
  push:
    branches: [main]
  schedule:
    # รัน chaos tests ทุกวันพุธ 2am Bangkok time
    - cron: "0 19 * * 3"  # 19:00 UTC = 02:00 Bangkok

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    outputs:
      deploy-success: ${{ steps.deploy.outcome }}
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        id: deploy
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/myorg/myapp:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/myapp -n staging

  chaos-test:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: needs.deploy-staging.outputs.deploy-success == 'success'
    
    timeout-minutes: 30
    
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install Dependencies
        run: pip install requests prometheus-client kubernetes

      - name: Configure kubectl
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ChaosTestRole
          aws-region: ap-southeast-1

      - name: Run Baseline Measurement
        id: baseline
        run: |
          python scripts/measure_baseline.py \
            --service-url http://myapp.staging.svc.cluster.local \
            --duration 60 \
            --output baseline.json
          
          cat baseline.json

      - name: Run Pod Kill Chaos
        id: chaos-pod
        run: |
          python scripts/chaos_experiment.py \
            --experiment pod-kill \
            --namespace staging \
            --service myapp \
            --duration 60 \
            --output chaos-pod-result.json

      - name: Run Network Latency Chaos
        id: chaos-network
        run: |
          python scripts/chaos_experiment.py \
            --experiment network-latency \
            --namespace staging \
            --service myapp \
            --latency 500 \
            --duration 60 \
            --output chaos-network-result.json

      - name: Evaluate Results
        id: evaluate
        run: |
          python scripts/evaluate_chaos.py \
            --baseline baseline.json \
            --results chaos-pod-result.json chaos-network-result.json \
            --threshold-success-rate 95 \
            --threshold-latency-multiplier 3

      - name: Upload Chaos Reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: chaos-reports
          path: |
            baseline.json
            chaos-pod-result.json
            chaos-network-result.json

      - name: Post to Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: "chaos-results"
          slack-message: |
            *Chaos Test Results* for ${{ github.repository }}
            Status: ${{ job.status }}
            Branch: ${{ github.ref_name }}
            Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### 6.2 Chaos Scorecard

```python
# scripts/chaos_scorecard.py
"""
สร้าง Chaos Engineering Scorecard
"""
import json
import sys
from pathlib import Path


def calculate_resilience_score(results: list) -> dict:
    """คำนวณ Resilience Score จาก chaos results"""
    
    total_experiments = len(results)
    passed = sum(1 for r in results if r.get('confirmed', False))
    
    # คำนวณ scores
    scores = {
        'pod_failure': 0,
        'network_chaos': 0,
        'resource_exhaustion': 0,
        'dependency_failure': 0,
    }
    
    for result in results:
        experiment_type = result.get('type', 'unknown')
        confirmed = result.get('confirmed', False)
        
        if 'pod' in experiment_type.lower():
            scores['pod_failure'] = 100 if confirmed else 0
        elif 'network' in experiment_type.lower():
            scores['network_chaos'] = 100 if confirmed else 0
        elif 'resource' in experiment_type.lower():
            scores['resource_exhaustion'] = 100 if confirmed else 0
        elif 'dependency' in experiment_type.lower():
            scores['dependency_failure'] = 100 if confirmed else 0
    
    overall = (passed / total_experiments * 100) if total_experiments > 0 else 0
    
    return {
        'overall_score': overall,
        'total_experiments': total_experiments,
        'passed': passed,
        'failed': total_experiments - passed,
        'category_scores': scores,
        'grade': get_grade(overall)
    }


def get_grade(score: float) -> str:
    """แปลง score เป็น grade"""
    if score >= 90:
        return "A - Excellent Resilience"
    elif score >= 80:
        return "B - Good Resilience"
    elif score >= 70:
        return "C - Acceptable Resilience"
    elif score >= 60:
        return "D - Poor Resilience"
    else:
        return "F - Critical Issues"


def generate_scorecard(results_files: list) -> None:
    """สร้าง scorecard จาก results files"""
    all_results = []
    
    for file_path in results_files:
        if Path(file_path).exists():
            with open(file_path) as f:
                result = json.load(f)
                all_results.append(result)
    
    scorecard = calculate_resilience_score(all_results)
    
    print("\n" + "="*60)
    print("CHAOS ENGINEERING SCORECARD")
    print("="*60)
    print(f"Overall Score: {scorecard['overall_score']:.1f}%")
    print(f"Grade: {scorecard['grade']}")
    print(f"Experiments: {scorecard['passed']}/{scorecard['total_experiments']} passed")
    
    print("\nCategory Scores:")
    for category, score in scorecard['category_scores'].items():
        bar = "█" * int(score / 10) + "░" * (10 - int(score / 10))
        print(f"  {category:25s}: [{bar}] {score:.0f}%")
    
    print("="*60)
    
    # Save scorecard
    with open('chaos-scorecard.json', 'w') as f:
        json.dump(scorecard, f, indent=2)
    
    # Exit ด้วย error ถ้า score ต่ำ
    if scorecard['overall_score'] < 70:
        print(f"\n❌ Resilience score {scorecard['overall_score']:.1f}% < 70% threshold")
        sys.exit(1)
    
    print(f"\n✅ Resilience score {scorecard['overall_score']:.1f}% passed!")


if __name__ == '__main__':
    generate_scorecard(sys.argv[1:])
```

---

## 7. Game Days

### 7.1 Game Day คืออะไร?

Game Day คือการซ้อมรับมือกับเหตุการณ์ฉุกเฉิน โดยมีทีมงานมาร่วมกันจำลองสถานการณ์ต่าง ๆ

### 7.2 Game Day Template

```markdown
# Game Day Plan: [ชื่อ]

## Objective
[เป้าหมายของ Game Day]
ตัวอย่าง: ทดสอบว่าทีมสามารถรับมือกับ database failure ใน production ได้

## Schedule
| Time | Activity |
|------|----------|
| 09:00 | Briefing และ review runbooks |
| 09:30 | Establish baseline metrics |
| 10:00 | Start chaos experiments |
| 10:30 | Incident response drill |
| 11:00 | Retrospective |
| 11:30 | Action items |

## Scenarios
1. Database primary node failure
2. Network partition ระหว่าง app และ database
3. Sudden 10x traffic spike

## Team Roles
- **Chaos Coordinator**: [ชื่อ] — รัน experiments
- **Observer**: [ชื่อ] — ดู metrics และ behavior
- **Incident Commander**: [ชื่อ] — coordinate response
- **Communication Lead**: [ชื่อ] — communicate to stakeholders

## Success Criteria
- [ ] ทีม detect incident ภายใน 5 นาที
- [ ] ทีม mitigate ภายใน 15 นาที
- [ ] ไม่มี data loss
- [ ] User impact < 2 นาที

## Runbooks ที่ต้องใช้
- [Runbook: Database Failover](link)
- [Runbook: Traffic Scaling](link)
- [Runbook: Rollback Procedure](link)

## Post Game Day
- บันทึก observations
- สร้าง action items
- อัพเดท runbooks
- กำหนด next game day
```

### 7.3 Automated Game Day

```yaml
# .github/workflows/game-day.yml
name: Automated Game Day

on:
  schedule:
    # ทุกวันอาทิตย์ 10am Bangkok time
    - cron: "0 3 * * 0"
  workflow_dispatch:
    inputs:
      scenario:
        description: 'Chaos scenario to run'
        required: true
        default: 'full'
        type: choice
        options:
          - full
          - pod-kill-only
          - network-only
          - resource-only

jobs:
  game-day:
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4

      - name: Notify Team - Game Day Starting
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: "sre-team"
          slack-message: |
            🎮 *Game Day Starting*
            Scenario: ${{ github.event.inputs.scenario || 'full' }}
            Time: $(date)
            Environment: Staging
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

      - name: Setup Chaos Tools
        run: |
          # Install LitmusChaos CLI
          curl -sLO https://github.com/litmuschaos/litmusctl/releases/download/0.27.0/litmusctl-linux-amd64-0.27.0.tar.gz
          tar xf litmusctl-linux-amd64-0.27.0.tar.gz
          sudo mv litmusctl /usr/local/bin/

      - name: Run Chaos Scenario
        id: chaos
        run: |
          SCENARIO="${{ github.event.inputs.scenario || 'full' }}"
          
          case "${SCENARIO}" in
            full)
              python scripts/game_day.py --scenario full
              ;;
            pod-kill-only)
              python scripts/game_day.py --scenario pod-kill
              ;;
            network-only)
              python scripts/game_day.py --scenario network
              ;;
            resource-only)
              python scripts/game_day.py --scenario resource
              ;;
          esac

      - name: Generate Report
        if: always()
        run: |
          python scripts/game_day_report.py \
            --output game-day-report.md

      - name: Upload Report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: game-day-report
          path: game-day-report.md

      - name: Notify Team - Game Day Complete
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: "sre-team"
          slack-message: |
            🎮 *Game Day Complete*
            Status: ${{ job.status }}
            Report: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## 8. Workshop: Chaos Testing Lab

### Lab 1: LitmusChaos Basic Experiment

```bash
#!/bin/bash
# workshop/lab1-litmus.sh

echo "=== Lab 1: LitmusChaos Basic Experiment ==="

# 1. Deploy test application
kubectl create namespace chaos-lab 2>/dev/null || true
kubectl label namespace chaos-lab istio-injection=enabled

cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
  namespace: chaos-lab
spec:
  replicas: 3
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
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: test-app
  namespace: chaos-lab
spec:
  selector:
    app: test-app
  ports:
    - port: 80
      targetPort: 80
EOF

# 2. รอให้ deploy เสร็จ
kubectl rollout status deployment/test-app -n chaos-lab

# 3. วัด baseline
echo "Measuring baseline..."
for i in {1..10}; do
    kubectl exec -n chaos-lab deployment/test-app -- \
        curl -sS -o /dev/null -w "%{http_code}" http://localhost:80/ 2>/dev/null
done | sort | uniq -c

# 4. Run chaos experiment (ตาม ChaosEngine ที่ defined ไว้)
echo "Running chaos experiment..."
cat << EOF | kubectl apply -f -
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: test-app-chaos
  namespace: chaos-lab
spec:
  appinfo:
    appns: chaos-lab
    applabel: "app=test-app"
    appkind: deployment
  annotationCheck: "false"
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "30"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: PODS_AFFECTED_PERC
              value: "33"
EOF

echo "Chaos experiment started!"
echo "Monitor with: kubectl get chaosresult -n chaos-lab -w"
```

---

## 9. สรุปและ Best Practices

### Chaos Engineering Maturity Model

```
Level 1 — Exploratory
├── Manual chaos tests เป็นครั้งคราว
├── Dev/test environments เท่านั้น
└── Basic experiments (pod kill)

Level 2 — Structured
├── Defined experiments ด้วย hypotheses
├── Staging environment regularly
└── Automated measurement

Level 3 — Integrated
├── CI/CD pipeline integration
├── Automated chaos in staging
└── Production experiments (limited)

Level 4 — Continuous
├── Automated game days
├── Production chaos with guardrails
└── Self-healing validation

Level 5 — Optimized
├── AI-driven chaos experiments
├── Full production chaos
└── Chaos as competitive advantage
```

---

## อ้างอิง

- [Chaos Engineering Principles](https://principlesofchaos.org/)
- [LitmusChaos Documentation](https://litmuschaos.io/docs/)
- [Netflix Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)
- [AWS Fault Injection Simulator](https://aws.amazon.com/fis/)
- [Gremlin Chaos Engineering](https://www.gremlin.com/)
- [Chaos Engineering Book](https://www.oreilly.com/library/view/chaos-engineering/9781492051725/)

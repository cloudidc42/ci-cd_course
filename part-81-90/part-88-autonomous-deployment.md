# Part 88: Autonomous Deployment Systems

## บทนำ

Autonomous Deployment คือ goal สูงสุดของ CI/CD evolution - ระบบที่สามารถ analyze, deploy, monitor, และ rollback ได้โดยไม่ต้องมีคนแทรกแซง บทนี้จะครอบคลุม automated canary analysis, progressive delivery automation, Flagger, และการออกแบบ autonomous deployment systems

## สารบัญ

1. [Autonomous Deployment คืออะไร](#autonomous)
2. [Automated Canary Analysis](#canary-analysis)
3. [Progressive Delivery Automation](#progressive-delivery)
4. [Flagger Deep Dive](#flagger)
5. [Automated Rollback Systems](#rollback)
6. [Deployment Decisions Engine](#decision-engine)
7. [Safety Nets และ Override Mechanisms](#safety-nets)
8. [Case Studies](#case-studies)
9. [แบบฝึกหัด](#exercises)

---

## 1. Autonomous Deployment คืออะไร {#autonomous}

### Evolution ของ Deployment Automation

```
Level 0: Manual
  - คนทำทุกอย่างด้วยมือ
  - Slow, error-prone
  - ไม่ reproducible

Level 1: Scripted
  - Deploy scripts
  - Still human-triggered
  - Reproducible แต่ต้องมีคน

Level 2: CI/CD Pipeline
  - Automated build/test
  - Human approves deploy
  - Fast แต่ต้องมีคน approve

Level 3: Continuous Deployment
  - Auto-deploy on tests pass
  - Human monitors
  - Fast, no approval needed

Level 4: Autonomous Deployment
  - Auto-deploy
  - Auto-analyze (canary)
  - Auto-rollback on issues
  - Human เข้ามาเฉพาะ edge cases

Level 5: Self-Healing Systems
  - Autonomous deployment
  - Auto-scale
  - Auto-remediate issues
  - Predictive deployment
```

### เมื่อไหร่ที่ Autonomous Deployment เหมาะสม?

```
Prerequisites สำหรับ Autonomous Deployment:

1. Test Coverage > 85%
   เพราะต้องเชื่อใจ automated tests

2. Comprehensive Monitoring
   เพราะต้องรู้ว่า deployment ดีหรือไม่

3. Fast Rollback < 5 minutes
   เพราะต้องสามารถ undo ได้เร็ว

4. Low Blast Radius
   เพราะต้องจำกัดผลกระทบ

5. Mature On-Call Process
   เพราะยังต้องมีคน handle edge cases
```

---

## 2. Automated Canary Analysis {#canary-analysis}

### Canary Analysis Framework

Automated Canary Analysis ใช้ statistical analysis เพื่อตัดสินว่า deployment ปลอดภัยหรือไม่:

```
Canary Analysis Process:

1. Deploy new version ให้ 5-10% traffic
2. เก็บ metrics 5-15 นาที:
   - Error rate
   - Latency (p50, p95, p99)
   - Success rate
   - Custom business metrics
   
3. Compare กับ baseline (current version)
4. Statistical significance test
5. If good → increase traffic
6. If bad → rollback automatically
```

### Metrics สำหรับ Canary Analysis

```python
# canary/analyzer.py

from scipy import stats
import numpy as np

class CanaryAnalyzer:
    """วิเคราะห์ canary deployment ด้วย statistical methods"""
    
    def __init__(self, metrics_client):
        self.metrics = metrics_client
    
    def analyze(
        self,
        canary_version: str,
        baseline_version: str,
        analysis_interval: int = 300  # 5 minutes
    ) -> AnalysisResult:
        """
        เปรียบเทียบ canary กับ baseline
        Return: pass/fail/inconclusive
        """
        
        # ดึง metrics จาก Prometheus
        canary_data = self._fetch_metrics(canary_version, analysis_interval)
        baseline_data = self._fetch_metrics(baseline_version, analysis_interval)
        
        checks = []
        
        # Check 1: Error rate
        error_check = self._check_error_rate(canary_data, baseline_data)
        checks.append(error_check)
        
        # Check 2: Latency
        latency_check = self._check_latency(canary_data, baseline_data)
        checks.append(latency_check)
        
        # Check 3: Request success rate
        success_check = self._check_success_rate(canary_data, baseline_data)
        checks.append(success_check)
        
        # Check 4: Custom business metrics
        business_checks = self._check_business_metrics(canary_data, baseline_data)
        checks.extend(business_checks)
        
        # Overall decision
        failed_checks = [c for c in checks if c.result == "fail"]
        passed_checks = [c for c in checks if c.result == "pass"]
        
        if len(failed_checks) > 0:
            decision = "fail"
            reason = f"Failed checks: {[c.name for c in failed_checks]}"
        elif len(passed_checks) == len(checks):
            decision = "pass"
            reason = "All checks passed"
        else:
            decision = "inconclusive"
            reason = "Insufficient data"
        
        return AnalysisResult(
            decision=decision,
            reason=reason,
            checks=checks,
            canary_version=canary_version,
            baseline_version=baseline_version
        )
    
    def _check_error_rate(
        self, 
        canary: dict, 
        baseline: dict
    ) -> Check:
        """ตรวจสอบว่า error rate ของ canary ไม่สูงกว่า baseline มาก"""
        
        canary_errors = canary.get("error_rate", [])
        baseline_errors = baseline.get("error_rate", [])
        
        if len(canary_errors) < 10 or len(baseline_errors) < 10:
            return Check(
                name="error_rate",
                result="inconclusive",
                reason="Insufficient data points"
            )
        
        canary_mean = np.mean(canary_errors)
        baseline_mean = np.mean(baseline_errors)
        
        # ถ้า error rate สูงกว่า baseline เกิน 20% → fail
        if canary_mean > baseline_mean * 1.2:
            return Check(
                name="error_rate",
                result="fail",
                reason=f"Canary error rate {canary_mean:.2%} > "
                       f"baseline {baseline_mean:.2%} by 20%+"
            )
        
        # Mann-Whitney U test (non-parametric)
        stat, p_value = stats.mannwhitneyu(
            canary_errors, 
            baseline_errors,
            alternative='greater'
        )
        
        if p_value < 0.05:
            return Check(
                name="error_rate",
                result="fail",
                reason=f"Statistically significant increase in errors (p={p_value:.3f})"
            )
        
        return Check(
            name="error_rate",
            result="pass",
            reason=f"Error rate within acceptable range "
                   f"(canary: {canary_mean:.2%}, baseline: {baseline_mean:.2%})"
        )
    
    def _check_latency(self, canary: dict, baseline: dict) -> Check:
        """ตรวจสอบ latency"""
        
        canary_p99 = np.percentile(canary.get("request_duration", [0]), 99)
        baseline_p99 = np.percentile(baseline.get("request_duration", [0]), 99)
        
        # Threshold: p99 ต้องไม่เกิน 200ms หรือไม่เพิ่มขึ้น 50%
        absolute_threshold = 0.2  # 200ms
        relative_threshold = 1.5  # 50% increase
        
        if canary_p99 > absolute_threshold or canary_p99 > baseline_p99 * relative_threshold:
            return Check(
                name="latency_p99",
                result="fail",
                reason=f"P99 latency {canary_p99*1000:.0f}ms exceeded threshold "
                       f"(baseline: {baseline_p99*1000:.0f}ms)"
            )
        
        return Check(name="latency_p99", result="pass", 
                    reason=f"P99 {canary_p99*1000:.0f}ms within limits")
```

---

## 3. Progressive Delivery Automation {#progressive-delivery}

### Automated Traffic Shifting

```python
# deployment/progressive_delivery.py

import asyncio
import logging

class ProgressiveDeliveryController:
    """Controller สำหรับ automated progressive delivery"""
    
    TRAFFIC_STEPS = [5, 10, 25, 50, 75, 100]
    ANALYSIS_INTERVAL = 300  # 5 minutes
    MIN_DATA_POINTS = 50
    
    def __init__(
        self,
        canary_analyzer: CanaryAnalyzer,
        traffic_router: TrafficRouter,
        notification_service: NotificationService
    ):
        self.analyzer = canary_analyzer
        self.router = traffic_router
        self.notifier = notification_service
    
    async def run_progressive_delivery(
        self,
        service: str,
        canary_version: str
    ) -> DeliveryResult:
        """
        รัน progressive delivery จนเสร็จหรือ rollback
        """
        
        baseline_version = await self.router.get_stable_version(service)
        
        logging.info(f"Starting progressive delivery: {canary_version}")
        self.notifier.send(
            f"🚀 Progressive delivery started for {service}:{canary_version}"
        )
        
        for traffic_pct in self.TRAFFIC_STEPS:
            logging.info(f"Setting canary traffic to {traffic_pct}%")
            
            # Route traffic
            await self.router.set_canary_weight(
                service=service,
                canary=canary_version,
                weight=traffic_pct
            )
            
            # Wait for traffic to stabilize
            await asyncio.sleep(60)
            
            # Analyze
            analysis = await self.analyzer.analyze(
                canary_version=canary_version,
                baseline_version=baseline_version
            )
            
            if analysis.decision == "fail":
                # Rollback
                await self._rollback(service, baseline_version, analysis)
                return DeliveryResult(
                    status="rolled_back",
                    reason=analysis.reason,
                    traffic_at_failure=traffic_pct
                )
            
            elif analysis.decision == "inconclusive":
                logging.info("Analysis inconclusive, waiting for more data...")
                await asyncio.sleep(self.ANALYSIS_INTERVAL)
                
                # Re-analyze
                analysis = await self.analyzer.analyze(
                    canary_version=canary_version,
                    baseline_version=baseline_version
                )
                
                if analysis.decision != "pass":
                    await self._rollback(service, baseline_version, analysis)
                    return DeliveryResult(
                        status="rolled_back",
                        reason="Insufficient confidence after extended analysis"
                    )
            
            logging.info(f"✅ {traffic_pct}% traffic analysis passed")
            
            # Pause between steps
            if traffic_pct < 100:
                await asyncio.sleep(self.ANALYSIS_INTERVAL)
        
        # All steps passed!
        self.notifier.send(
            f"✅ Progressive delivery complete: {service}:{canary_version}"
        )
        
        return DeliveryResult(
            status="completed",
            canary_version=canary_version
        )
    
    async def _rollback(
        self, 
        service: str, 
        baseline_version: str,
        analysis: AnalysisResult
    ):
        """Execute automated rollback"""
        
        logging.warning(f"🔄 Rolling back {service}: {analysis.reason}")
        
        await self.router.set_canary_weight(
            service=service,
            canary=analysis.canary_version,
            weight=0
        )
        
        self.notifier.send(
            f"🔄 ROLLBACK: {service} rolled back\n"
            f"Reason: {analysis.reason}",
            severity="warning"
        )
```

---

## 4. Flagger Deep Dive {#flagger}

### Flagger Architecture

```
Flagger Architecture:

┌──────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                   │
│                                                      │
│  ┌─────────────────────────────────────────────────┐ │
│  │                  Flagger                         │ │
│  │  - Watches Canary objects                       │ │
│  │  - Controls traffic routing                     │ │
│  │  - Runs analysis (Prometheus/Datadog/etc.)      │ │
│  │  - Makes pass/fail decisions                    │ │
│  │  - Executes rollbacks                           │ │
│  └──────────────┬──────────────────────────────────┘ │
│                 │ manages                             │
│  ┌──────────────▼──────────────────────────────────┐ │
│  │  Deployment/Service/VirtualService               │ │
│  │  ┌──────────────┐    ┌──────────────────────┐   │ │
│  │  │   Stable     │    │   Canary             │   │ │
│  │  │   (90-100%)  │    │   (0-10%)            │   │ │
│  │  └──────────────┘    └──────────────────────┘   │ │
│  └─────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
                       │ queries
                       ▼
              ┌─────────────────┐
              │   Prometheus    │
              │   (metrics)     │
              └─────────────────┘
```

### Flagger Configuration

```yaml
# flagger/canary-payment-service.yaml

apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: payment-service
  namespace: production

spec:
  # Target deployment
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  
  # Progress timeout
  progressDeadlineSeconds: 120
  
  # Service configuration
  service:
    port: 80
    targetPort: 8080
    
    # Istio traffic routing
    gateways:
      - company-gateway.istio-system.svc.cluster.local
    hosts:
      - payment.company.com
    
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
    
    # Traffic policy
    trafficPolicy:
      connectionPool:
        http:
          http2MaxRequests: 1000
          h2UpgradePolicy: UPGRADE
  
  # Canary analysis configuration
  analysis:
    # Traffic shift interval
    interval: 5m
    
    # Number of failed checks before rollback
    threshold: 5
    
    # Maximum traffic to route to canary
    maxWeight: 50
    
    # How much to increase traffic each step
    stepWeight: 10
    
    # Metrics to analyze
    metrics:
      - name: request-success-rate
        # From Prometheus
        interval: 1m
        # Minimum success rate threshold
        thresholdRange:
          min: 99
      
      - name: request-duration
        # P99 latency
        interval: 30s
        thresholdRange:
          max: 500  # milliseconds
      
      # Custom business metric
      - name: payment-success-rate
        templateRef:
          name: payment-success-rate
          namespace: flagger-system
        thresholdRange:
          min: 95
        interval: 1m
    
    # Pre-deployment checks
    webhooks:
      - name: acceptance-test
        type: pre-rollout
        url: http://flagger-loadtester.test/
        timeout: 30s
        metadata:
          type: bash
          cmd: |
            curl -sd 'test' http://payment-service-canary/health | grep '"status":"ok"'
      
      # Load test during canary
      - name: load-test
        url: http://flagger-loadtester.test/
        timeout: 5s
        metadata:
          type: cmd
          cmd: |
            hey -z 1m -q 10 -c 2 \
              -H "Host: payment.company.com" \
              http://payment-service-canary/

---
# Custom metric template
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: payment-success-rate
  namespace: flagger-system
spec:
  provider:
    type: prometheus
    address: http://prometheus.monitoring:9090
  
  query: |
    100 - sum(
      rate(
        payment_transactions_failed_total{
          namespace="{{ namespace }}",
          name="{{ target }}"
        }[{{ interval }}]
      )
    ) / sum(
      rate(
        payment_transactions_total{
          namespace="{{ namespace }}",
          name="{{ target }}"
        }[{{ interval }}]
      )
    ) * 100
```

### Flagger Event Handling

```python
# flagger/event_handler.py
# Handle Flagger events และส่ง notifications

import asyncio
import json
from kubernetes import client, watch

class FlaggerEventHandler:
    """Monitor Flagger events และ notify team"""
    
    def __init__(self, slack_client, pagerduty_client):
        self.slack = slack_client
        self.pagerduty = pagerduty_client
    
    async def watch_canary_events(self):
        """Watch Kubernetes events จาก Flagger"""
        
        v1 = client.CoreV1Api()
        w = watch.Watch()
        
        for event in w.stream(
            v1.list_event_for_all_namespaces,
            field_selector="reportingComponent=flagger"
        ):
            await self._handle_event(event['object'])
    
    async def _handle_event(self, event):
        """Process Flagger event"""
        
        service = event.involved_object.name
        reason = event.reason
        message = event.message
        
        if reason == "Synced":
            await self.slack.send(
                channel="#deployments",
                message=f"✅ {service}: {message}"
            )
        
        elif reason == "Progressed":
            # Traffic increasing
            traffic_pct = self._extract_traffic_pct(message)
            await self.slack.send(
                channel="#deployments",
                message=f"📈 {service}: {message} ({traffic_pct}%)"
            )
        
        elif reason == "Failed":
            # Canary failed - critical alert
            await self.slack.send(
                channel="#deployments-critical",
                message=f"🚨 CANARY FAILED: {service}\n{message}"
            )
            
            # Create PagerDuty incident ถ้า production
            if self._is_production(event):
                await self.pagerduty.create_incident(
                    title=f"Canary failure: {service}",
                    details=message,
                    severity="high"
                )
        
        elif reason == "Rollback":
            await self.slack.send(
                channel="#deployments",
                message=f"🔄 ROLLBACK: {service}\n{message}"
            )
        
        elif reason == "Promoted":
            # Canary promoted to stable!
            await self.slack.send(
                channel="#deployments",
                message=f"🎉 PROMOTED: {service}\n{message}"
            )
```

---

## 5. Automated Rollback Systems {#rollback}

### Rollback Triggers

```
Automated Rollback ควร trigger เมื่อ:

1. Error Rate สูงขึ้น
   - HTTP 5xx > 5% (threshold)
   - Exception rate เพิ่มขึ้น > 200%
   
2. Latency เพิ่มขึ้น
   - P99 > 2x baseline
   - P50 > 1.5x baseline
   
3. Business Metrics ลดลง
   - Conversion rate ลดลง > 10%
   - Payment success rate < 95%
   
4. External Signals
   - PagerDuty incident
   - Customer support tickets spike
   - Manual override by engineer
   
5. Infrastructure Issues
   - OOMKilled containers
   - CrashLoopBackOff
   - Health check failures
```

### Multi-Signal Rollback Decision

```python
# rollback/decision_engine.py

class RollbackDecisionEngine:
    """ตัดสินใจว่าควร rollback หรือไม่"""
    
    def __init__(self, threshold_config: dict):
        self.thresholds = threshold_config
        self.signals = []
    
    def evaluate(self, metrics: dict) -> RollbackDecision:
        """ประเมิน metrics และตัดสินใจ"""
        
        violated_signals = []
        warning_signals = []
        
        # Check error rate
        error_check = self._check_error_rate(metrics)
        if error_check.severity == "critical":
            violated_signals.append(error_check)
        elif error_check.severity == "warning":
            warning_signals.append(error_check)
        
        # Check latency
        latency_check = self._check_latency(metrics)
        if latency_check.severity == "critical":
            violated_signals.append(latency_check)
        elif latency_check.severity == "warning":
            warning_signals.append(latency_check)
        
        # Check business metrics
        business_checks = self._check_business_metrics(metrics)
        for check in business_checks:
            if check.severity == "critical":
                violated_signals.append(check)
        
        # Decision logic
        if len(violated_signals) >= 1:
            # Critical violation → immediate rollback
            return RollbackDecision(
                should_rollback=True,
                urgency="immediate",
                reason=f"Critical violations: {[s.name for s in violated_signals]}",
                confidence=0.95
            )
        
        elif len(warning_signals) >= 2:
            # Multiple warnings → rollback after confirmation
            return RollbackDecision(
                should_rollback=True,
                urgency="soon",
                reason=f"Multiple warnings: {[s.name for s in warning_signals]}",
                confidence=0.75
            )
        
        elif len(warning_signals) == 1:
            # Single warning → monitor closely
            return RollbackDecision(
                should_rollback=False,
                urgency="monitor",
                reason=f"Single warning: {warning_signals[0].name}",
                confidence=0.3
            )
        
        else:
            return RollbackDecision(
                should_rollback=False,
                urgency="none",
                reason="All metrics within acceptable range",
                confidence=0.9
            )
    
    def _check_error_rate(self, metrics: dict) -> Signal:
        current = metrics.get('error_rate', 0)
        baseline = metrics.get('baseline_error_rate', 0)
        threshold = self.thresholds['error_rate']
        
        if current > threshold['critical']:
            return Signal(
                name="error_rate",
                severity="critical",
                value=current,
                threshold=threshold['critical'],
                message=f"Error rate {current:.2%} > critical threshold {threshold['critical']:.2%}"
            )
        elif current > threshold['warning'] or current > baseline * 2:
            return Signal(
                name="error_rate",
                severity="warning",
                value=current,
                message=f"Error rate {current:.2%} elevated vs baseline {baseline:.2%}"
            )
        
        return Signal(name="error_rate", severity="ok", value=current)
```

---

## 6. Deployment Decisions Engine {#decision-engine}

### ML-Based Deployment Risk Assessment

```python
# ml/deployment_risk.py

import numpy as np
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler

class DeploymentRiskPredictor:
    """ใช้ ML ทำนายความเสี่ยงของ deployment"""
    
    def __init__(self, model_path: str):
        self.model = self._load_model(model_path)
        self.scaler = self._load_scaler(model_path)
    
    def predict_risk(self, deployment_features: dict) -> RiskPrediction:
        """ทำนาย risk ของ deployment"""
        
        features = self._extract_features(deployment_features)
        features_scaled = self.scaler.transform([features])
        
        # Predict probability ของ failure
        failure_prob = self.model.predict_proba(features_scaled)[0][1]
        
        # Feature importance
        feature_names = self._get_feature_names()
        importances = self.model.feature_importances_
        top_risk_factors = sorted(
            zip(feature_names, importances * features),
            key=lambda x: abs(x[1]),
            reverse=True
        )[:3]
        
        return RiskPrediction(
            failure_probability=failure_prob,
            risk_level=self._classify_risk(failure_prob),
            top_risk_factors=top_risk_factors,
            recommendation=self._get_recommendation(failure_prob)
        )
    
    def _extract_features(self, deployment: dict) -> list:
        """แปลง deployment metadata เป็น features"""
        
        return [
            # Code change metrics
            deployment.get('files_changed', 0),
            deployment.get('lines_added', 0),
            deployment.get('lines_deleted', 0),
            
            # Test metrics
            deployment.get('test_coverage', 0),
            deployment.get('tests_added', 0),
            deployment.get('flaky_test_count', 0),
            
            # Historical metrics
            deployment.get('days_since_last_deploy', 0),
            deployment.get('recent_failure_rate', 0),
            deployment.get('mean_time_between_failures', 0),
            
            # Timing factors
            deployment.get('hour_of_day', 0),
            deployment.get('is_friday', 0),
            deployment.get('is_end_of_month', 0),
            
            # Dependency metrics
            deployment.get('dependencies_changed', 0),
            deployment.get('config_changes', 0)
        ]
    
    def _classify_risk(self, prob: float) -> str:
        if prob < 0.05: return "very_low"
        if prob < 0.15: return "low"
        if prob < 0.30: return "medium"
        if prob < 0.50: return "high"
        return "very_high"
    
    def _get_recommendation(self, prob: float) -> str:
        if prob < 0.15:
            return "Proceed with automated deployment"
        elif prob < 0.30:
            return "Proceed with increased canary monitoring"
        elif prob < 0.50:
            return "Consider deploying during business hours with engineer on standby"
        else:
            return "Strongly recommend delaying deployment and reviewing changes"
```

---

## 7. Safety Nets และ Override Mechanisms {#safety-nets}

### Circuit Breaker Pattern

```python
# safety/circuit_breaker.py

class DeploymentCircuitBreaker:
    """
    ป้องกัน cascading failures จาก bad deployments
    """
    
    def __init__(self):
        self.failure_count = 0
        self.success_count = 0
        self.state = "closed"  # closed, open, half-open
        self.last_failure_time = None
        
        self.FAILURE_THRESHOLD = 3
        self.RECOVERY_TIMEOUT = 300  # 5 minutes
        self.HALF_OPEN_LIMIT = 1
    
    def can_deploy(self) -> tuple[bool, str]:
        """ตรวจสอบว่า deployment ควร proceed หรือไม่"""
        
        if self.state == "open":
            # Check ว่าถึงเวลา recovery แล้วหรือยัง
            if time.time() - self.last_failure_time > self.RECOVERY_TIMEOUT:
                self.state = "half-open"
                self.failure_count = 0
                return True, "Circuit half-open, allowing test deployment"
            
            return False, f"Circuit breaker OPEN: {self.failure_count} failures. Wait {self.RECOVERY_TIMEOUT}s"
        
        elif self.state == "half-open":
            return True, "Circuit half-open, monitoring closely"
        
        return True, "Circuit closed, deployment allowed"
    
    def record_success(self):
        self.success_count += 1
        self.failure_count = 0
        
        if self.state == "half-open":
            self.state = "closed"
    
    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.FAILURE_THRESHOLD:
            self.state = "open"
            # Alert
            self._alert_team("Circuit breaker OPEN")
```

### Human Override Mechanisms

```python
# safety/override.py

class DeploymentOverrideSystem:
    """
    ให้ Engineer override automated decisions
    """
    
    OVERRIDE_TYPES = {
        "force_rollback": {
            "permission": "senior-engineer",
            "reason_required": True,
            "audit_trail": True
        },
        "skip_analysis": {
            "permission": "on-call-engineer",
            "reason_required": True,
            "audit_trail": True
        },
        "emergency_deploy": {
            "permission": "tech-lead",
            "reason_required": True,
            "audit_trail": True,
            "notify_vp": True
        },
        "pause_deployment": {
            "permission": "any-engineer",
            "reason_required": False,
            "audit_trail": True
        }
    }
    
    def execute_override(
        self,
        override_type: str,
        engineer: User,
        reason: str,
        service: str
    ) -> OverrideResult:
        
        config = self.OVERRIDE_TYPES.get(override_type)
        if not config:
            raise ValueError(f"Unknown override type: {override_type}")
        
        # ตรวจสอบ permission
        if not self._has_permission(engineer, config["permission"]):
            return OverrideResult(
                success=False,
                error=f"Insufficient permissions. Required: {config['permission']}"
            )
        
        # ตรวจสอบ reason
        if config["reason_required"] and not reason:
            return OverrideResult(
                success=False,
                error="Reason is required for this override"
            )
        
        # Audit log
        self._create_audit_log(
            override_type=override_type,
            engineer=engineer,
            reason=reason,
            service=service
        )
        
        # Notify ถ้าจำเป็น
        if config.get("notify_vp"):
            self._notify_management(override_type, engineer, service, reason)
        
        # Execute override
        result = self._execute(override_type, service)
        
        return OverrideResult(
            success=True,
            message=f"Override {override_type} executed for {service}"
        )
```

---

## 8. Case Studies {#case-studies}

### Case Study 1: Netflix - Automated Deployment at Scale

**บริบท:**
- Deploy 4,000+ ครั้ง/วัน
- 1,000+ microservices
- ต้องไม่มี downtime

**Approach:**
```
Spinnaker Platform:
- Automated canary analysis (Kayenta)
- Progressive traffic shifting
- Automated rollback
- Multi-cloud deployment

Kayenta (Automated Canary Analysis):
- Statistical comparison ของ metrics
- Mann-Whitney U test
- Custom metric templates
- Pass/Fail scoring (0-100)
```

**ผลลัพธ์:**
```
Deployments ที่ automated rollback ป้องกัน: ~200/เดือน
Mean time saved per rollback: 45 minutes
Engineer intervention required: < 2% of deployments
Production incidents from bad deployments: ลดลง 70%
```

### Case Study 2: Startup ที่ implement Autonomous Deployment

**Starting Point:**
```
Team: 8 developers
Deployments: 2/week
Manual canary: ใช้เวลา 2-3 ชั่วโมงต่อ deployment
Rollbacks: 2-4 ชั่วโมง
On-call burden: High
```

**Implementation (3 เดือน):**
```
Month 1: Foundation
- Comprehensive metrics (Prometheus + Grafana)
- Automated smoke tests
- Flagger installation

Month 2: Canary Automation
- Define canary success metrics
- Implement 5%→100% progressive delivery
- Automated rollback

Month 3: Full Automation
- Remove manual approval gate
- ML-based risk assessment
- Circuit breaker
```

**After:**
```
Deployments: 3-5/day (↑ 10x)
Canary analysis: fully automated
Rollback time: 3 minutes (↓ from 3 hours)
On-call burden: ลดลง 60%
Team focus: features, not deployment
```

---

## 9. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Canary Analysis Implementation

**งาน:** implement simple canary analyzer ที่:
1. Query Prometheus สำหรับ error rate และ latency
2. Compare canary กับ baseline
3. ตัดสินใจ pass/fail
4. สร้าง readable report

```python
# canary_analyzer.py
import requests
import statistics

PROMETHEUS_URL = "http://prometheus:9090"
CANARY_SERVICE = "payment-service-canary"
STABLE_SERVICE = "payment-service"

def query_prometheus(query: str) -> list:
    response = requests.get(
        f"{PROMETHEUS_URL}/api/v1/query",
        params={"query": query}
    )
    # TODO: parse and return values

def analyze_canary() -> dict:
    # TODO: Implement
    # 1. Query error rates
    # 2. Query latency
    # 3. Compare
    # 4. Return decision
    pass

result = analyze_canary()
print(f"Decision: {result['decision']}")
print(f"Reason: {result['reason']}")
```

### แบบฝึกหัดที่ 2: Flagger Configuration

**งาน:** สร้าง Flagger Canary configuration สำหรับ:
- Service: user-service
- Strategy: Canary (5% steps up to 50%)
- Metrics: error rate < 1%, p99 latency < 300ms
- Analysis interval: 3 minutes

### แบบฝึกหัดที่ 3: Rollback Automation

**งาน:** ออกแบบและ implement automated rollback ที่:
1. Monitor metrics ทุก 30 วินาที
2. Trigger rollback ถ้า error rate > 5%
3. Send Slack notification
4. Create incident ticket
5. Log audit trail

---

## สรุป

Autonomous Deployment Systems เป็น engineering achievement ที่ยากแต่ valuable มาก:

1. **Start with Metrics** - ต้องมี observability ก่อน automation
2. **Canary Analysis ต้องสถิติถูกต้อง** - ไม่ใช่แค่ threshold comparison
3. **Safety nets สำคัญกว่า speed** - Circuit breakers, human overrides
4. **Incremental adoption** - จาก manual → semi-auto → full-auto
5. **Trust ต้องสร้างทีละขั้น** - เริ่มด้วย canary analysis, ค่อยๆ เพิ่ม automation

## อ่านเพิ่มเติม

- Flagger Documentation: https://flagger.app
- Argo Rollouts: https://argo-rollouts.readthedocs.io
- Kayenta (Netflix): https://kayenta.io
- Spinnaker: https://spinnaker.io
- "Progressive Delivery" by James Governor

---

*Part 88 จาก 100 | CI/CD Mastery Course*

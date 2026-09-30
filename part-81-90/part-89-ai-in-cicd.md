# Part 89: AI/ML ใน CI/CD

## บทนำ

AI กำลัง transform CI/CD landscape ในทุกด้าน ตั้งแต่ code review อัตโนมัติ ไปจนถึงการ predict pipeline failures ก่อนที่จะเกิดขึ้น บทนี้จะสำรวจการประยุกต์ใช้ AI/ML ใน CI/CD workflow แบบ practical และ forward-looking

## สารบัญ

1. [AI-Assisted Code Review](#ai-code-review)
2. [Intelligent Test Selection](#intelligent-test)
3. [Predictive Pipeline Optimization](#predictive)
4. [AI-Generated CI/CD Configurations](#ai-configs)
5. [Anomaly Detection ใน Deployments](#anomaly)
6. [AI-Powered Root Cause Analysis](#rca)
7. [Future Trends และ Emerging Technologies](#future)
8. [Ethical Considerations](#ethics)
9. [Case Studies](#case-studies)
10. [แบบฝึกหัด](#exercises)

---

## 1. AI-Assisted Code Review {#ai-code-review}

### Code Review ด้วย LLM

```python
# ai_review/code_reviewer.py
# ใช้ Claude API สำหรับ automated code review

import anthropic

class AICodeReviewer:
    """AI-powered code reviewer ที่ integrate กับ GitHub"""
    
    def __init__(self, api_key: str):
        self.client = anthropic.Anthropic(api_key=api_key)
        self.model = "claude-opus-4-5"
    
    def review_pull_request(
        self,
        diff: str,
        language: str,
        context: dict
    ) -> CodeReview:
        """Review PR diff และส่งคำแนะนำ"""
        
        prompt = f"""You are an expert code reviewer specializing in {language}.
        
Review the following code changes and provide:
1. Security vulnerabilities
2. Performance issues  
3. Code quality improvements
4. Missing test cases
5. Documentation gaps

Context:
- Service: {context.get('service_name')}
- Team: {context.get('team_name')}
- Related issues: {context.get('related_issues')}

Code Changes:
{diff}

Respond in JSON format:
{{
  "overall_assessment": "approve|request_changes|comment",
  "security_issues": [{{
    "severity": "critical|high|medium|low",
    "file": "...",
    "line": 0,
    "description": "...",
    "suggestion": "..."
  }}],
  "performance_issues": [...],
  "quality_suggestions": [...],
  "missing_tests": [...],
  "summary": "..."
}}"""
        
        message = self.client.messages.create(
            model=self.model,
            max_tokens=4096,
            messages=[{"role": "user", "content": prompt}]
        )
        
        import json
        review_data = json.loads(message.content[0].text)
        
        return CodeReview(
            assessment=review_data["overall_assessment"],
            security_issues=review_data["security_issues"],
            performance_issues=review_data["performance_issues"],
            quality_suggestions=review_data["quality_suggestions"],
            summary=review_data["summary"]
        )
    
    def generate_test_cases(
        self,
        function_code: str,
        language: str
    ) -> list:
        """Generate test cases สำหรับ function"""
        
        prompt = f"""Generate comprehensive unit tests for this {language} function.
        
Cover: happy path, edge cases, error cases, boundary values.

Function:
{function_code}

Return tests as a list of test cases in {language} syntax."""
        
        message = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return self._parse_test_cases(message.content[0].text, language)
```

### AI Review Integration กับ GitHub

```yaml
# .github/workflows/ai-code-review.yml

name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Get PR Diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > pr_diff.txt
          echo "diff_size=$(wc -l < pr_diff.txt)" >> $GITHUB_OUTPUT
      
      - name: Skip if diff too large
        if: steps.diff.outputs.diff_size > 1000
        run: |
          echo "Diff too large for AI review (>1000 lines). Consider splitting PR."
          exit 0
      
      - name: Run AI Review
        if: steps.diff.outputs.diff_size <= 1000
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python3 scripts/ai-review.py \
            --diff pr_diff.txt \
            --pr-number ${{ github.event.pull_request.number }} \
            --repo ${{ github.repository }}
      
      - name: Post Review to GitHub
        uses: actions/github-script@v7
        with:
          script: |
            const review = JSON.parse(
              require('fs').readFileSync('ai_review_result.json', 'utf8')
            );
            
            // Post review
            await github.rest.pulls.createReview({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
              event: review.assessment === 'approve' ? 'APPROVE' : 'REQUEST_CHANGES',
              body: review.summary,
              comments: review.inline_comments.map(c => ({
                path: c.file,
                line: c.line,
                body: c.comment
              }))
            });
```

### Security-Focused AI Review

```python
# ai_review/security_reviewer.py

SECURITY_PATTERNS_PROMPT = """
Analyze this code for security vulnerabilities. Focus on:

1. SQL Injection
   - String concatenation in SQL queries
   - Missing parameterized queries
   
2. XSS (Cross-Site Scripting)
   - Unescaped user input in HTML
   - innerHTML usage
   
3. Authentication/Authorization
   - Missing access controls
   - Hardcoded credentials
   - Weak password handling
   
4. Cryptography
   - Weak hash algorithms (MD5, SHA1)
   - Hardcoded encryption keys
   - Missing encryption for sensitive data
   
5. Injection Attacks
   - Command injection
   - LDAP injection
   - XML injection
   
6. Information Disclosure
   - Sensitive data in logs
   - Error messages exposing internals
   - Debug information in production

7. OWASP Top 10 patterns

For each finding:
- Severity (Critical/High/Medium/Low)
- CWE ID
- Description
- Code location
- Fix recommendation
"""

def security_review(code: str, language: str) -> SecurityReport:
    client = anthropic.Anthropic()
    
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        system="You are a security expert specializing in code security review. Be precise and actionable.",
        messages=[{
            "role": "user",
            "content": f"{SECURITY_PATTERNS_PROMPT}\n\nCode to review:\n```{language}\n{code}\n```"
        }]
    )
    
    return parse_security_report(response.content[0].text)
```

---

## 2. Intelligent Test Selection {#intelligent-test}

### ML-Based Test Selection

```python
# test_selection/intelligent_selector.py

import numpy as np
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.feature_extraction.text import TfidfVectorizer

class IntelligentTestSelector:
    """ใช้ ML เลือก tests ที่น่าจะ fail เมื่อ code เปลี่ยน"""
    
    def __init__(self, model_path: str):
        self.model = self._load_model(model_path)
        self.vectorizer = TfidfVectorizer(
            analyzer='char_wb',
            ngram_range=(3, 5)
        )
    
    def select_tests(
        self,
        changed_files: list,
        all_tests: list
    ) -> list:
        """เลือก tests ที่ควร run"""
        
        selected = []
        
        for test in all_tests:
            # คำนวณ probability ที่ test จะ fail
            features = self._extract_features(test, changed_files)
            prob = self.model.predict_proba([features])[0][1]
            
            selected.append({
                "test": test,
                "failure_probability": prob,
                "should_run": prob > 0.1  # threshold
            })
        
        # เรียงตาม probability
        selected.sort(key=lambda x: x["failure_probability"], reverse=True)
        
        # รัน tests ที่มี prob > threshold ก่อน
        must_run = [s["test"] for s in selected if s["should_run"]]
        optional_run = [s["test"] for s in selected if not s["should_run"]]
        
        return {
            "priority_tests": must_run,
            "optional_tests": optional_run,
            "skipped_count": len(optional_run),
            "expected_time_saving": self._estimate_time_saving(
                len(optional_run), len(all_tests)
            )
        }
    
    def _extract_features(self, test: str, changed_files: list) -> list:
        """สร้าง feature vector"""
        
        # Feature 1: Semantic similarity ระหว่าง test กับ changed code
        test_tokens = self._tokenize(test)
        changed_tokens = [t for f in changed_files for t in self._tokenize(f)]
        similarity = self._calculate_similarity(test_tokens, changed_tokens)
        
        # Feature 2: Historical failure rate ของ test เมื่อ files เหล่านี้เปลี่ยน
        historical_rate = self._get_historical_failure_rate(test, changed_files)
        
        # Feature 3: Direct dependency ระหว่าง test กับ changed code
        has_direct_dep = self._check_direct_dependency(test, changed_files)
        
        # Feature 4: Number of changed files
        num_changed = len(changed_files)
        
        return [similarity, historical_rate, has_direct_dep, num_changed]
```

### Test Impact Analysis

```python
# test_selection/impact_analysis.py

class TestImpactAnalyzer:
    """วิเคราะห์ว่าการ change code ส่งผลต่อ tests ไหนบ้าง"""
    
    def __init__(self, coverage_data: dict):
        """
        coverage_data: {
          "tests/test_payment.py::test_create": {
            "covered_lines": {
              "src/payment.py": [10, 11, 15, 20, 25]
            }
          }
        }
        """
        self.coverage = coverage_data
        self.impact_graph = self._build_impact_graph()
    
    def _build_impact_graph(self) -> dict:
        """สร้าง mapping จาก source line → tests ที่ cover line นั้น"""
        
        impact = {}  # file_line → [test_names]
        
        for test_name, test_data in self.coverage.items():
            for source_file, lines in test_data["covered_lines"].items():
                for line in lines:
                    key = f"{source_file}:{line}"
                    if key not in impact:
                        impact[key] = []
                    impact[key].append(test_name)
        
        return impact
    
    def get_affected_tests(self, git_diff: str) -> list:
        """หา tests ที่ได้รับผลจาก code changes"""
        
        changed_lines = self._parse_diff(git_diff)
        affected_tests = set()
        
        for file_path, lines in changed_lines.items():
            for line in lines:
                key = f"{file_path}:{line}"
                tests = self.impact_graph.get(key, [])
                affected_tests.update(tests)
        
        return list(affected_tests)
    
    def calculate_test_priority(self, affected_tests: list) -> dict:
        """จัดลำดับความสำคัญของ tests"""
        
        priorities = {}
        
        for test in affected_tests:
            # Priority = จำนวน changed lines ที่ test cover
            priority_score = self._calculate_priority_score(test)
            priorities[test] = priority_score
        
        return dict(sorted(
            priorities.items(),
            key=lambda x: x[1],
            reverse=True
        ))
```

---

## 3. Predictive Pipeline Optimization {#predictive}

### Build Time Prediction

```python
# predictions/build_time_predictor.py

import pandas as pd
from sklearn.ensemble import RandomForestRegressor
from sklearn.preprocessing import LabelEncoder

class BuildTimePredictor:
    """ทำนาย build time ก่อนที่ pipeline จะเริ่ม"""
    
    def __init__(self):
        self.model = RandomForestRegressor(
            n_estimators=100,
            random_state=42
        )
        self.label_encoders = {}
    
    def train(self, historical_runs: pd.DataFrame):
        """Train model จาก historical pipeline runs"""
        
        features = self._extract_features(historical_runs)
        target = historical_runs['duration_seconds']
        
        self.model.fit(features, target)
    
    def predict(self, commit_info: dict) -> dict:
        """ทำนาย build time และ resource requirements"""
        
        features = self._extract_commit_features(commit_info)
        
        predicted_duration = self.model.predict([features])[0]
        confidence_interval = self._calculate_ci(features)
        
        return {
            "predicted_seconds": round(predicted_duration),
            "predicted_minutes": round(predicted_duration / 60, 1),
            "confidence_interval": confidence_interval,
            "runner_recommendation": self._recommend_runner(predicted_duration),
            "cache_strategy": self._recommend_cache(commit_info)
        }
    
    def _extract_commit_features(self, commit: dict) -> list:
        """แปลง commit metadata เป็น features"""
        
        return [
            commit.get('files_changed', 0),
            commit.get('lines_changed', 0),
            commit.get('test_files_changed', 0),
            commit.get('package_json_changed', 0),
            commit.get('go_mod_changed', 0),
            commit.get('dockerfile_changed', 0),
            commit.get('hour_of_day', 12),
            commit.get('day_of_week', 0),
            commit.get('author_historical_avg', 0),
        ]
    
    def _recommend_runner(self, predicted_seconds: float) -> str:
        """แนะนำ runner type ตาม predicted build time"""
        
        if predicted_seconds < 120:
            return "ubuntu-small"  # 2 CPU
        elif predicted_seconds < 600:
            return "ubuntu-medium"  # 4 CPU
        elif predicted_seconds < 1800:
            return "ubuntu-large"  # 8 CPU
        else:
            return "ubuntu-xlarge"  # 16 CPU
```

### Pipeline Failure Prediction

```python
# predictions/failure_predictor.py

class PipelineFailurePredictor:
    """ทำนายว่า pipeline จะ fail หรือไม่ก่อนที่จะ run"""
    
    RISK_FACTORS = {
        "friday_after_4pm": 2.5,       # High risk deployment time
        "large_diff": 1.8,             # Large code changes
        "many_files_changed": 1.5,
        "no_tests_added": 1.3,
        "test_coverage_decreased": 2.0,
        "flaky_test_in_path": 1.7,
        "recent_failed_builds": 1.4,
        "dependency_update": 1.6,
        "database_migration": 2.2,
    }
    
    def calculate_risk_score(self, pipeline_context: dict) -> dict:
        """คำนวณ risk score และ recommendations"""
        
        risk_score = 1.0  # baseline
        active_factors = []
        
        # Check แต่ละ risk factor
        if self._is_friday_afternoon():
            risk_score *= self.RISK_FACTORS["friday_after_4pm"]
            active_factors.append("Deploying Friday afternoon")
        
        files_changed = pipeline_context.get("files_changed", 0)
        if files_changed > 50:
            risk_score *= self.RISK_FACTORS["large_diff"]
            active_factors.append(f"Large diff ({files_changed} files)")
        
        if pipeline_context.get("has_db_migration"):
            risk_score *= self.RISK_FACTORS["database_migration"]
            active_factors.append("Contains database migration")
        
        coverage_change = pipeline_context.get("coverage_change", 0)
        if coverage_change < -5:
            risk_score *= self.RISK_FACTORS["test_coverage_decreased"]
            active_factors.append(f"Test coverage decreased by {abs(coverage_change):.1f}%")
        
        # Normalize to 0-100 scale
        risk_percentage = min(100, (risk_score - 1) / 5 * 100)
        
        return {
            "risk_score": round(risk_percentage, 1),
            "risk_level": self._classify_risk(risk_percentage),
            "active_risk_factors": active_factors,
            "recommendations": self._get_recommendations(active_factors),
            "should_alert_engineer": risk_percentage > 60
        }
    
    def _get_recommendations(self, factors: list) -> list:
        """สร้าง recommendations ตาม risk factors"""
        
        recommendations = []
        
        if "Friday afternoon" in str(factors):
            recommendations.append("Consider deploying Monday morning instead")
        
        if any("migration" in f for f in factors):
            recommendations.append("Run database migration separately with DBA present")
            recommendations.append("Ensure rollback plan is ready")
        
        if any("coverage" in f for f in factors):
            recommendations.append("Add tests before merging")
        
        return recommendations
```

---

## 4. AI-Generated CI/CD Configurations {#ai-configs}

### GitHub Actions Generator

```python
# ai_config/pipeline_generator.py

class CICDPipelineGenerator:
    """Generate CI/CD pipeline configuration จาก project analysis"""
    
    def __init__(self, ai_client):
        self.ai = ai_client
    
    def generate_pipeline(
        self, 
        project_path: str,
        requirements: dict = None
    ) -> str:
        """Generate GitHub Actions workflow จาก project analysis"""
        
        # Analyze project
        project_info = self._analyze_project(project_path)
        
        prompt = f"""Generate a production-ready GitHub Actions CI/CD workflow.

Project Analysis:
{json.dumps(project_info, indent=2)}

Requirements:
{json.dumps(requirements or {}, indent=2)}

The workflow should:
1. Use appropriate build tools for the detected language/framework
2. Include all necessary test stages (unit, integration, e2e if applicable)
3. Include security scanning (SAST, SCA)
4. Include Docker build if Dockerfile exists
5. Include deployment to detected environments
6. Use caching for dependencies
7. Follow security best practices (minimal permissions, no secret hardcoding)
8. Include proper error handling and notifications

Output the complete GitHub Actions YAML workflow."""
        
        response = self.ai.messages.create(
            model="claude-opus-4-5",
            max_tokens=8192,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return self._extract_yaml(response.content[0].text)
    
    def _analyze_project(self, path: str) -> dict:
        """วิเคราะห์ project structure"""
        
        analysis = {
            "languages": [],
            "frameworks": [],
            "has_dockerfile": False,
            "has_docker_compose": False,
            "test_frameworks": [],
            "package_managers": [],
            "environments": []
        }
        
        # Detect language and frameworks
        if os.path.exists(f"{path}/package.json"):
            analysis["languages"].append("javascript/typescript")
            analysis["package_managers"].append("npm")
            
            with open(f"{path}/package.json") as f:
                pkg = json.load(f)
                
            if "react" in pkg.get("dependencies", {}):
                analysis["frameworks"].append("react")
            if "express" in pkg.get("dependencies", {}):
                analysis["frameworks"].append("express")
            if "jest" in pkg.get("devDependencies", {}):
                analysis["test_frameworks"].append("jest")
        
        if os.path.exists(f"{path}/go.mod"):
            analysis["languages"].append("go")
            analysis["package_managers"].append("go modules")
        
        if os.path.exists(f"{path}/requirements.txt"):
            analysis["languages"].append("python")
            analysis["package_managers"].append("pip")
        
        # Check Docker
        analysis["has_dockerfile"] = os.path.exists(f"{path}/Dockerfile")
        analysis["has_docker_compose"] = os.path.exists(f"{path}/docker-compose.yml")
        
        return analysis
```

### Kubernetes Manifest Generator

```python
# ai_config/k8s_generator.py

class KubernetesManifestGenerator:
    """Generate Kubernetes manifests จาก service specification"""
    
    def generate_manifests(self, service_spec: dict) -> dict:
        """Generate complete K8s manifests"""
        
        prompt = f"""Generate production-ready Kubernetes manifests for this service.

Service Specification:
{json.dumps(service_spec, indent=2)}

Generate the following manifests:
1. Deployment with:
   - Proper resource requests and limits
   - Liveness and readiness probes
   - Security context (non-root, read-only filesystem)
   - Environment variable references from secrets/configmaps
   - Pod disruption budget
   
2. Service
3. HorizontalPodAutoscaler
4. PodDisruptionBudget
5. NetworkPolicy (restrict ingress/egress)
6. ServiceAccount with minimal permissions

Output as a JSON object with keys for each manifest type."""
        
        response = self.ai.messages.create(
            model="claude-opus-4-5",
            max_tokens=8192,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return json.loads(self._extract_json(response.content[0].text))
```

---

## 5. Anomaly Detection ใน Deployments {#anomaly}

### ML-Based Anomaly Detection

```python
# anomaly/detector.py

from sklearn.ensemble import IsolationForest
import numpy as np

class DeploymentAnomalyDetector:
    """ตรวจจับ anomalies ใน deployment metrics"""
    
    def __init__(self):
        self.model = IsolationForest(
            contamination=0.05,  # 5% anomaly rate
            random_state=42
        )
        self.feature_history = []
    
    def train_on_history(self, historical_deployments: list):
        """Train model จาก deployment history"""
        
        features = [
            self._extract_features(d) 
            for d in historical_deployments
        ]
        
        self.model.fit(features)
        self.feature_history = features
    
    def is_anomalous(self, current_metrics: dict) -> dict:
        """ตรวจสอบว่า deployment metrics ปกติหรือไม่"""
        
        features = self._extract_features(current_metrics)
        prediction = self.model.predict([features])
        score = self.model.score_samples([features])
        
        is_anomaly = prediction[0] == -1
        anomaly_score = -score[0]  # Higher = more anomalous
        
        if is_anomaly:
            anomalous_features = self._identify_anomalous_features(
                features,
                current_metrics
            )
        else:
            anomalous_features = []
        
        return {
            "is_anomaly": is_anomaly,
            "anomaly_score": float(anomaly_score),
            "confidence": self._calculate_confidence(anomaly_score),
            "anomalous_features": anomalous_features,
            "recommendation": "investigate" if is_anomaly else "normal"
        }
    
    def _extract_features(self, metrics: dict) -> list:
        return [
            metrics.get("error_rate", 0),
            metrics.get("latency_p50", 0),
            metrics.get("latency_p99", 0),
            metrics.get("requests_per_second", 0),
            metrics.get("cpu_usage", 0),
            metrics.get("memory_usage", 0),
            metrics.get("active_connections", 0),
            metrics.get("cache_hit_rate", 0)
        ]
```

---

## 6. AI-Powered Root Cause Analysis {#rca}

### Automated Root Cause Analysis

```python
# rca/analyzer.py

class AutomatedRCASystem:
    """ระบบ Root Cause Analysis อัตโนมัติ"""
    
    def __init__(self, ai_client, metrics_client, log_client):
        self.ai = ai_client
        self.metrics = metrics_client
        self.logs = log_client
    
    async def analyze_incident(
        self,
        incident_id: str,
        service: str,
        start_time: datetime,
        end_time: datetime
    ) -> RCAReport:
        """วิเคราะห์ incident และหา root cause"""
        
        # รวบรวม evidence
        evidence = await self._collect_evidence(
            service, start_time, end_time
        )
        
        # ส่งให้ AI วิเคราะห์
        prompt = f"""Analyze this production incident and provide root cause analysis.

Incident Details:
- Service: {service}
- Start: {start_time}
- End: {end_time}
- Duration: {(end_time - start_time).total_seconds() / 60:.0f} minutes

Evidence Collected:

## Recent Deployments:
{json.dumps(evidence['recent_deployments'], indent=2)}

## Error Logs (sample):
{evidence['error_logs'][:5000]}  # Limit for token count

## Metrics Changes:
{json.dumps(evidence['metric_changes'], indent=2)}

## Infrastructure Events:
{json.dumps(evidence['infra_events'], indent=2)}

Provide:
1. Most likely root cause (with confidence %)
2. Contributing factors
3. Timeline of events
4. Immediate remediation taken
5. Prevention recommendations
6. Similar past incidents (if patterns match)

Format as structured JSON."""
        
        response = await self.ai.messages.create_async(
            model="claude-opus-4-5",
            max_tokens=4096,
            messages=[{"role": "user", "content": prompt}]
        )
        
        rca_data = json.loads(response.content[0].text)
        
        return RCAReport(
            incident_id=incident_id,
            root_cause=rca_data["root_cause"],
            confidence=rca_data["confidence"],
            contributing_factors=rca_data["contributing_factors"],
            timeline=rca_data["timeline"],
            prevention=rca_data["prevention_recommendations"]
        )
```

---

## 7. Future Trends และ Emerging Technologies {#future}

### AI-Native CI/CD: 2025-2030 Vision

```
2025: Current State
✅ AI-assisted code review (available now)
✅ Intelligent test selection (emerging)
✅ Predictive build optimization (early stage)
✅ AI-generated configurations (available)

2026-2027: Near Future
🔮 Self-healing pipelines
   - AI detects and fixes flaky tests automatically
   - Auto-updates dependencies
   - Resolves build cache issues

🔮 Predictive deployment windows
   - AI suggests optimal deploy times
   - Predicts traffic patterns
   - Reduces deployment risk

🔮 Natural language CI/CD
   - "Deploy payment service to production after 6pm"
   - "Roll back everything from the last hour"
   - Voice-controlled deployments

2028-2030: Further Future
🔮 Autonomous software delivery
   - AI writes tests for new code
   - AI optimizes code for performance
   - End-to-end AI-driven SDLC
   
🔮 Multi-modal deployment intelligence
   - AI understands business context
   - Correlates code changes with business metrics
   - Makes deployment decisions based on business impact
```

### Emerging AI CI/CD Tools

```
Current Landscape (2024-2025):

1. GitHub Copilot for CI/CD
   - Suggest workflow configurations
   - Auto-fix failed pipelines
   - Generate test code

2. Harness AI
   - AI-powered deployment verification
   - Intelligent rollback
   - Cost optimization suggestions

3. Amazon CodeGuru
   - Code quality analysis
   - Security vulnerabilities
   - Performance hotspots

4. Tabnine/Cursor
   - Context-aware code completion
   - CI/CD config assistance
   - Test generation

5. Emerging Open Source:
   - AutoDev (automated development)
   - DevOpsGPT (NL to pipeline)
```

---

## 8. Ethical Considerations {#ethics}

### Responsible AI ใน CI/CD

```
ความเสี่ยงของ AI ใน CI/CD:

1. Over-reliance
   - Engineer ไม่เข้าใจ pipeline ที่ AI สร้าง
   - ไม่สามารถ debug เมื่อมีปัญหา
   - "Black box" problem
   
   Mitigation:
   - AI ต้อง explain decisions
   - Human review ก่อน apply
   - Training สำหรับ engineers

2. Bias ใน Decisions
   - AI ที่ train บน historical data
   - อาจ perpetuate past mistakes
   - อาจ discriminate (เช่น บล็อก code จาก certain developers)
   
   Mitigation:
   - Audit AI decisions regularly
   - Diverse training data
   - Human override mechanism

3. Security Risks
   - AI model ถูก poison
   - Adversarial inputs
   - Data leakage ใน AI prompts
   
   Mitigation:
   - ไม่ส่ง secrets ไปยัง AI
   - Validate AI-generated configs
   - Security review ของ AI outputs

4. Over-automation
   - Deploy ไปเร็วเกินไปโดยไม่มี human check
   - ขาด domain context
   
   Mitigation:
   - Keep human in loop สำหรับ critical decisions
   - Clear boundaries ของ autonomous actions
```

---

## 9. Case Studies {#case-studies}

### Case Study 1: Google - AI-Powered Test Selection

**บริบท:**
- Monorepo ขนาดใหญ่
- 100M+ lines of code
- Billions of test runs/day
- 25,000+ engineers

**ปัญหา:**
- ไม่สามารถรัน tests ทั้งหมด ทุก commit ได้
- Naive selection ทำให้ miss failures

**Solution:**
```
ML-based test selection (TAP System):
- Feature extraction จาก code changes
- Historical failure patterns
- Dependency analysis
- Dynamic test prioritization

Results:
- 90% reduction in unnecessary test runs
- Same failure detection rate
- $millions saved in compute costs annually
```

### Case Study 2: Startup ที่ใช้ AI สำหรับ Pipeline Generation

**บริบท:**
- Early-stage startup, 5 developers
- No dedicated DevOps
- Multiple microservices

**Solution:**
```python
# พวกเขาใช้ AI เพื่อ:
1. Generate initial GitHub Actions workflows
2. Suggest optimizations
3. Debug failing pipelines
4. Write test configurations

Prompt ที่ใช้บ่อย:
"Our Node.js service pipeline is failing at the Docker build step.
Here's the error: [error]. Here's our Dockerfile: [dockerfile].
What's wrong and how do we fix it?"

Time saved: 40+ hours ใน setup time
Quality: Production-ready configurations
```

---

## 10. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: AI Code Review Integration

**งาน:** Integrate AI code review เข้ากับ GitHub Actions:
1. สร้าง workflow ที่ trigger บน PR
2. ส่ง diff ไปยัง Claude API
3. Parse response และ post comment
4. Set PR status ตาม review result

```python
# Starter code
import anthropic
import os

def review_code(diff: str) -> dict:
    client = anthropic.Anthropic(
        api_key=os.environ['ANTHROPIC_API_KEY']
    )
    
    # TODO: Implement AI review
    # 1. Create prompt with diff
    # 2. Call Claude API
    # 3. Parse response
    # 4. Return structured review
    pass
```

### แบบฝึกหัดที่ 2: Build Time Prediction

**งาน:** สร้าง simple build time predictor:
1. Collect historical build data (mock data provided)
2. Train sklearn model
3. Predict ใน new commits
4. Evaluate accuracy

```python
HISTORICAL_DATA = [
    {"files_changed": 5, "lines_changed": 100, "has_docker": True, "build_time": 300},
    {"files_changed": 50, "lines_changed": 1000, "has_docker": True, "build_time": 900},
    # ... more data
]

# TODO: Train and evaluate model
```

### แบบฝึกหัดที่ 3: Pipeline Generator

**งาน:** สร้าง simple pipeline generator ด้วย AI:
1. Analyze ไฟล์ใน project (detect language)
2. Generate GitHub Actions YAML
3. Validate generated YAML syntax
4. Test กับ sample projects

---

## สรุป

AI กำลัง transform CI/CD อย่างรวดเร็ว:

1. **AI-Assisted ≠ AI-Controlled** - Engineer ยังต้อง understand และ verify
2. **Start Small** - เริ่มด้วย non-critical use cases
3. **Measure Impact** - วัดว่า AI ช่วย improve metrics ได้จริงหรือไม่
4. **Security First** - ไม่ส่ง secrets หรือ sensitive data ไปยัง AI
5. **Human Override** - สำหรับ critical decisions ต้องมี human in loop เสมอ

## อ่านเพิ่มเติม

- Google ML in CI Research: https://research.google/pubs
- GitHub Copilot: https://github.com/features/copilot
- Anthropic API: https://docs.anthropic.com
- "AI Engineering" by Chip Huyen
- DORA Report on AI in DevOps: https://dora.dev

---

*Part 89 จาก 100 | CI/CD Mastery Course*

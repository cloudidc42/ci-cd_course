# Part 73: CI/CD Metrics & DORA Metrics

## บทนำ

DORA (DevOps Research and Assessment) metrics เป็นตัวชี้วัดสำคัญ 4 ตัวที่ใช้วัดประสิทธิภาพของ software delivery ซึ่งมาจากงานวิจัย Accelerate ที่ศึกษาทีม engineering กว่า 30,000 ทีมทั่วโลก

ในบทนี้เราจะเรียนรู้:
- 4 DORA metrics หลัก
- วิธีวัด Deployment Frequency
- Lead Time for Changes
- Mean Time to Recovery (MTTR)
- Change Failure Rate
- เครื่องมือ (Sleuth, Faros, LinearB)
- กลยุทธ์การปรับปรุง
- แบบฝึกหัดปฏิบัติ

---

## 73.1 DORA Metrics คืออะไร?

### 4 ตัวชี้วัดหลัก

```
┌─────────────────────────────────────────────────────────────┐
│                    DORA Metrics                              │
├────────────────────────┬────────────────────────────────────┤
│  THROUGHPUT             │  STABILITY                        │
│                         │                                   │
│  1. Deployment          │  3. Mean Time to Recovery        │
│     Frequency           │     (MTTR)                       │
│                         │                                   │
│  2. Lead Time for       │  4. Change Failure Rate          │
│     Changes             │     (CFR)                        │
└────────────────────────┴────────────────────────────────────┘
```

### Elite vs. Low Performers

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| Deployment Frequency | On-demand (หลายครั้ง/วัน) | 1x/สัปดาห์ ถึง 1x/เดือน | 1x/เดือน ถึง 1x/6เดือน | น้อยกว่า 1x/6เดือน |
| Lead Time for Changes | < 1 ชั่วโมง | 1 วัน ถึง 1 สัปดาห์ | 1 สัปดาห์ ถึง 1 เดือน | > 6 เดือน |
| MTTR | < 1 ชั่วโมง | < 1 วัน | 1 วัน ถึง 1 สัปดาห์ | > 6 เดือน |
| Change Failure Rate | 0-15% | 0-15% | 16-30% | 16-30% |

---

## 73.2 Deployment Frequency

**คำนิยาม:** ความถี่ที่ทีม deploy code ไปยัง production

### วิธีวัด

```python
# deployment-frequency.py
from datetime import datetime, timedelta
from typing import List
import statistics

def calculate_deployment_frequency(
    deployments: List[datetime],
    period_days: int = 30
) -> dict:
    """
    คำนวณ deployment frequency

    Args:
        deployments: รายการวันเวลาที่ deploy
        period_days: ช่วงเวลาที่วัด (วัน)

    Returns:
        dict ที่มี frequency stats
    """
    if not deployments:
        return {"error": "ไม่มีข้อมูล deployment"}

    # กรองเฉพาะในช่วงเวลาที่กำหนด
    cutoff = datetime.now() - timedelta(days=period_days)
    recent_deployments = [d for d in deployments if d >= cutoff]

    total_deployments = len(recent_deployments)
    avg_per_day = total_deployments / period_days

    # คำนวณ gaps ระหว่าง deployments
    sorted_deps = sorted(recent_deployments)
    gaps = []
    for i in range(1, len(sorted_deps)):
        gap = (sorted_deps[i] - sorted_deps[i-1]).total_seconds() / 3600  # hours
        gaps.append(gap)

    return {
        "total_deployments": total_deployments,
        "period_days": period_days,
        "avg_per_day": round(avg_per_day, 2),
        "avg_per_week": round(avg_per_day * 7, 2),
        "avg_gap_hours": round(statistics.mean(gaps), 2) if gaps else 0,
        "median_gap_hours": round(statistics.median(gaps), 2) if gaps else 0,
        "category": categorize_frequency(avg_per_day),
    }

def categorize_frequency(avg_per_day: float) -> str:
    """จัดหมวดหมู่ตาม DORA standard"""
    if avg_per_day >= 1:
        return "Elite"
    elif avg_per_day >= 1/7:
        return "High"
    elif avg_per_day >= 1/30:
        return "Medium"
    else:
        return "Low"
```

### เก็บข้อมูลจาก GitHub Actions

```python
# collect-deployments-github.py
import requests
from datetime import datetime
from typing import List, Optional

class GitHubDeploymentCollector:
    def __init__(self, org: str, token: str):
        self.org = org
        self.token = token
        self.headers = {
            "Authorization": f"token {token}",
            "Accept": "application/vnd.github.v3+json",
        }
        self.base_url = "https://api.github.com"

    def get_deployments(
        self,
        repo: str,
        environment: str = "production",
        since: Optional[datetime] = None,
    ) -> List[dict]:
        """ดึงข้อมูล deployments จาก GitHub Deployments API"""
        url = f"{self.base_url}/repos/{self.org}/{repo}/deployments"
        params = {
            "environment": environment,
            "per_page": 100,
        }

        deployments = []
        page = 1

        while True:
            params["page"] = page
            response = requests.get(url, headers=self.headers, params=params)
            data = response.json()

            if not data:
                break

            for dep in data:
                created_at = datetime.strptime(
                    dep["created_at"], "%Y-%m-%dT%H:%M:%SZ"
                )

                if since and created_at < since:
                    return deployments

                # ดูว่า deployment สำเร็จหรือไม่
                status = self.get_deployment_status(repo, dep["id"])
                if status == "success":
                    deployments.append({
                        "id": dep["id"],
                        "created_at": created_at,
                        "sha": dep["sha"],
                        "environment": dep["environment"],
                        "creator": dep["creator"]["login"],
                    })

            page += 1

        return deployments

    def get_deployment_status(self, repo: str, deployment_id: int) -> str:
        """ดูสถานะล่าสุดของ deployment"""
        url = f"{self.base_url}/repos/{self.org}/{repo}/deployments/{deployment_id}/statuses"
        response = requests.get(url, headers=self.headers)
        statuses = response.json()

        if statuses:
            return statuses[0]["state"]  # latest status
        return "unknown"

    def get_workflow_runs(
        self,
        repo: str,
        workflow_name: str = "deploy.yaml",
    ) -> List[dict]:
        """ดึงข้อมูลจาก GitHub Actions workflow runs"""
        url = f"{self.base_url}/repos/{self.org}/{repo}/actions/workflows/{workflow_name}/runs"
        params = {
            "status": "success",
            "branch": "main",
            "per_page": 100,
        }

        response = requests.get(url, headers=self.headers, params=params)
        data = response.json()

        return [
            {
                "id": run["id"],
                "created_at": datetime.strptime(run["created_at"], "%Y-%m-%dT%H:%M:%SZ"),
                "run_number": run["run_number"],
                "head_sha": run["head_sha"],
            }
            for run in data.get("workflow_runs", [])
        ]


# ตัวอย่างการใช้งาน
collector = GitHubDeploymentCollector("mycompany", "github_token")
deployments = collector.get_deployments("payment-service")
timestamps = [d["created_at"] for d in deployments]
frequency = calculate_deployment_frequency(timestamps)
print(f"Deployment Frequency: {frequency}")
```

---

## 73.3 Lead Time for Changes

**คำนิยาม:** เวลาตั้งแต่ code commit จนถึง running ใน production

### วิธีวัด

```python
# lead-time.py
from datetime import datetime
from typing import List, Optional
import statistics

def calculate_lead_time(changes: List[dict]) -> dict:
    """
    คำนวณ lead time for changes

    change = {
        "commit_sha": str,
        "commit_timestamp": datetime,
        "deploy_timestamp": datetime,
        "pr_created_at": datetime,  # optional
        "pr_merged_at": datetime,   # optional
    }
    """
    lead_times = []
    breakdown = {
        "coding_time": [],       # commit → PR created
        "review_time": [],       # PR created → PR merged
        "deploy_wait_time": [],  # PR merged → deploy triggered
        "deploy_time": [],       # deploy triggered → deployed
    }

    for change in changes:
        # คำนวณ total lead time (commit → production)
        total_hours = (
            change["deploy_timestamp"] - change["commit_timestamp"]
        ).total_seconds() / 3600

        lead_times.append(total_hours)

        # Breakdown ถ้ามีข้อมูล PR
        if change.get("pr_created_at") and change.get("pr_merged_at"):
            coding_time = (
                change["pr_created_at"] - change["commit_timestamp"]
            ).total_seconds() / 3600
            review_time = (
                change["pr_merged_at"] - change["pr_created_at"]
            ).total_seconds() / 3600
            deploy_wait = (
                change["deploy_timestamp"] - change["pr_merged_at"]
            ).total_seconds() / 3600

            breakdown["coding_time"].append(max(0, coding_time))
            breakdown["review_time"].append(max(0, review_time))
            breakdown["deploy_wait_time"].append(max(0, deploy_wait))

    if not lead_times:
        return {"error": "ไม่มีข้อมูล"}

    return {
        "avg_lead_time_hours": round(statistics.mean(lead_times), 2),
        "p50_lead_time_hours": round(statistics.median(lead_times), 2),
        "p90_lead_time_hours": round(
            sorted(lead_times)[int(len(lead_times) * 0.9)], 2
        ),
        "p99_lead_time_hours": round(
            sorted(lead_times)[int(len(lead_times) * 0.99)], 2
        ),
        "breakdown_avg": {
            k: round(statistics.mean(v), 2) if v else 0
            for k, v in breakdown.items()
        },
        "category": categorize_lead_time(statistics.median(lead_times)),
    }

def categorize_lead_time(hours: float) -> str:
    if hours < 1:
        return "Elite"
    elif hours < 168:  # 1 week
        return "High"
    elif hours < 720:  # 1 month
        return "Medium"
    else:
        return "Low"
```

### เก็บข้อมูลจาก Git และ Deployment

```python
# collect-lead-time.py
import subprocess
import json
from datetime import datetime

class LeadTimeCollector:
    def __init__(self, repo_path: str, deployment_api_url: str):
        self.repo_path = repo_path
        self.deployment_api_url = deployment_api_url

    def get_commits_for_deployment(
        self,
        deploy_sha: str,
        previous_deploy_sha: str,
    ) -> List[dict]:
        """ดึง commits ระหว่าง 2 deployments"""
        result = subprocess.run(
            [
                "git", "log",
                f"{previous_deploy_sha}..{deploy_sha}",
                "--format=%H|%ai|%ae|%s",
                "--no-merges",
            ],
            capture_output=True,
            text=True,
            cwd=self.repo_path,
        )

        commits = []
        for line in result.stdout.strip().split("\n"):
            if not line:
                continue
            parts = line.split("|", 3)
            if len(parts) >= 3:
                commits.append({
                    "sha": parts[0],
                    "timestamp": datetime.fromisoformat(parts[1]),
                    "author": parts[2],
                    "message": parts[3] if len(parts) > 3 else "",
                })

        return commits

    def get_pr_info_from_github(
        self,
        commit_sha: str,
        org: str,
        repo: str,
        token: str,
    ) -> Optional[dict]:
        """ดึงข้อมูล PR ที่ commit นี้อยู่"""
        url = f"https://api.github.com/repos/{org}/{repo}/commits/{commit_sha}/pulls"
        headers = {
            "Authorization": f"token {token}",
            "Accept": "application/vnd.github.v3+json",
        }

        response = requests.get(url, headers=headers)
        prs = response.json()

        if prs:
            pr = prs[0]
            return {
                "number": pr["number"],
                "created_at": datetime.strptime(
                    pr["created_at"], "%Y-%m-%dT%H:%M:%SZ"
                ),
                "merged_at": datetime.strptime(
                    pr["merged_at"], "%Y-%m-%dT%H:%M:%SZ"
                ) if pr.get("merged_at") else None,
            }
        return None
```

---

## 73.4 Mean Time to Recovery (MTTR)

**คำนิยาม:** เวลาเฉลี่ยในการกู้คืนจากเหตุการณ์ production ที่ทำให้ service ล่ม

### วิธีวัด

```python
# mttr.py
from datetime import datetime
from typing import List
import statistics

def calculate_mttr(incidents: List[dict]) -> dict:
    """
    คำนวณ MTTR

    incident = {
        "id": str,
        "started_at": datetime,
        "resolved_at": datetime,
        "severity": str,  # P0, P1, P2, P3
        "root_cause": str,
        "service": str,
    }
    """
    recovery_times = []
    by_severity = {}

    for incident in incidents:
        if not incident.get("resolved_at"):
            continue  # ยังไม่แก้ไข

        recovery_hours = (
            incident["resolved_at"] - incident["started_at"]
        ).total_seconds() / 3600

        recovery_times.append(recovery_hours)

        # แยกตาม severity
        sev = incident.get("severity", "unknown")
        if sev not in by_severity:
            by_severity[sev] = []
        by_severity[sev].append(recovery_hours)

    if not recovery_times:
        return {"error": "ไม่มีข้อมูล"}

    return {
        "overall": {
            "mean_hours": round(statistics.mean(recovery_times), 2),
            "median_hours": round(statistics.median(recovery_times), 2),
            "p90_hours": round(
                sorted(recovery_times)[int(len(recovery_times) * 0.9)], 2
            ),
        },
        "by_severity": {
            sev: {
                "mean_hours": round(statistics.mean(times), 2),
                "count": len(times),
            }
            for sev, times in by_severity.items()
        },
        "category": categorize_mttr(statistics.median(recovery_times)),
    }

def categorize_mttr(hours: float) -> str:
    if hours < 1:
        return "Elite"
    elif hours < 24:
        return "High"
    elif hours < 168:  # 1 week
        return "Medium"
    else:
        return "Low"
```

### เชื่อมกับ PagerDuty

```python
# collect-mttr-pagerduty.py
import requests
from datetime import datetime, timedelta
from typing import List

class PagerDutyMTTRCollector:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.pagerduty.com"
        self.headers = {
            "Authorization": f"Token token={api_key}",
            "Accept": "application/vnd.pagerduty+json;version=2",
        }

    def get_incidents(
        self,
        service_ids: List[str] = None,
        days: int = 30,
        urgency: str = "high",
    ) -> List[dict]:
        """ดึง incidents จาก PagerDuty"""
        since = (datetime.now() - timedelta(days=days)).isoformat()

        params = {
            "since": since,
            "statuses[]": "resolved",
            "urgencies[]": urgency,
            "limit": 100,
        }

        if service_ids:
            params["service_ids[]"] = service_ids

        response = requests.get(
            f"{self.base_url}/incidents",
            headers=self.headers,
            params=params,
        )

        incidents = []
        for incident in response.json().get("incidents", []):
            # คำนวณ time to acknowledge และ time to resolve
            created = datetime.fromisoformat(
                incident["created_at"].replace("Z", "+00:00")
            )
            resolved = None
            if incident.get("resolved_at"):
                resolved = datetime.fromisoformat(
                    incident["resolved_at"].replace("Z", "+00:00")
                )

            incidents.append({
                "id": incident["id"],
                "title": incident["title"],
                "severity": self.map_urgency(incident["urgency"]),
                "started_at": created,
                "resolved_at": resolved,
                "service": incident["service"]["summary"],
                "created_by": incident.get("first_trigger_log_entry", {})
                    .get("agent", {})
                    .get("summary", "unknown"),
            })

        return incidents

    def map_urgency(self, urgency: str) -> str:
        mapping = {"high": "P1", "low": "P2"}
        return mapping.get(urgency, "P3")

    def get_mttr_trend(self, service_id: str, weeks: int = 12) -> List[dict]:
        """ดู trend ของ MTTR รายสัปดาห์"""
        trend = []

        for week in range(weeks):
            end = datetime.now() - timedelta(weeks=week)
            start = end - timedelta(weeks=1)

            incidents = self.get_incidents(
                service_ids=[service_id],
                days=7,
            )

            if incidents:
                result = calculate_mttr(incidents)
                trend.append({
                    "week": f"Week {weeks - week}",
                    "start": start.isoformat(),
                    "end": end.isoformat(),
                    "mttr_hours": result["overall"]["mean_hours"],
                    "incident_count": len(incidents),
                })

        return sorted(trend, key=lambda x: x["start"])
```

---

## 73.5 Change Failure Rate (CFR)

**คำนิยาม:** สัดส่วนของ deployments ที่ทำให้เกิด incident หรือต้องทำ rollback

### วิธีวัด

```python
# change-failure-rate.py
from datetime import datetime, timedelta
from typing import List

def calculate_cfr(
    deployments: List[dict],
    failures: List[dict],
    period_days: int = 30,
) -> dict:
    """
    คำนวณ Change Failure Rate

    deployment = {
        "id": str,
        "deployed_at": datetime,
        "environment": str,
        "service": str,
        "sha": str,
    }

    failure = {
        "deployment_id": str,  # deployment ที่ทำให้เกิดปัญหา
        "detected_at": datetime,
        "type": str,  # incident, rollback, hotfix
    }
    """
    cutoff = datetime.now() - timedelta(days=period_days)

    # กรองเฉพาะ production deployments ในช่วงเวลา
    prod_deployments = [
        d for d in deployments
        if d["environment"] == "production"
        and d["deployed_at"] >= cutoff
    ]

    if not prod_deployments:
        return {"error": "ไม่มีข้อมูล deployment"}

    # หา deployments ที่ fail
    failed_deployment_ids = {f["deployment_id"] for f in failures}
    failed_deployments = [
        d for d in prod_deployments
        if d["id"] in failed_deployment_ids
    ]

    total = len(prod_deployments)
    failed = len(failed_deployments)
    cfr = (failed / total) * 100

    return {
        "total_deployments": total,
        "failed_deployments": failed,
        "cfr_percentage": round(cfr, 2),
        "category": categorize_cfr(cfr),
        "failure_types": _count_failure_types(failures, failed_deployment_ids),
    }

def categorize_cfr(cfr: float) -> str:
    if cfr <= 15:
        return "Elite/High"
    elif cfr <= 30:
        return "Medium"
    else:
        return "Low"

def _count_failure_types(failures: List[dict], relevant_ids: set) -> dict:
    """นับประเภทของ failure"""
    counts = {}
    for failure in failures:
        if failure["deployment_id"] in relevant_ids:
            ftype = failure.get("type", "unknown")
            counts[ftype] = counts.get(ftype, 0) + 1
    return counts
```

---

## 73.6 การเชื่อมทุก Metric เข้าด้วยกัน

### DORA Dashboard

```python
# dora-dashboard.py
from dataclasses import dataclass
from datetime import datetime, timedelta
from typing import Optional

@dataclass
class DORAMetrics:
    deployment_frequency: dict
    lead_time: dict
    mttr: dict
    cfr: dict
    overall_score: str
    calculated_at: datetime

class DORACalculator:
    def __init__(
        self,
        github_token: str,
        pagerduty_token: str,
        org: str,
    ):
        self.github = GitHubDeploymentCollector(org, github_token)
        self.pagerduty = PagerDutyMTTRCollector(pagerduty_token)
        self.org = org

    def calculate_for_service(
        self,
        service: str,
        period_days: int = 30,
    ) -> DORAMetrics:
        """คำนวณ DORA metrics สำหรับ service เดียว"""

        # Deployment Frequency
        deployments = self.github.get_deployments(service)
        dep_timestamps = [d["created_at"] for d in deployments]
        df_metrics = calculate_deployment_frequency(dep_timestamps, period_days)

        # Lead Time (ต้องการข้อมูล commit-deploy mapping)
        changes = self._build_change_data(service, deployments)
        lt_metrics = calculate_lead_time(changes)

        # MTTR
        incidents = self.pagerduty.get_incidents(days=period_days)
        mttr_metrics = calculate_mttr(incidents)

        # Change Failure Rate
        failures = self._get_failures(service, incidents, deployments)
        cfr_metrics = calculate_cfr(deployments, failures, period_days)

        # คำนวณ overall score
        overall = self._calculate_overall_score(
            df_metrics["category"],
            lt_metrics["category"],
            mttr_metrics["category"],
            cfr_metrics["category"],
        )

        return DORAMetrics(
            deployment_frequency=df_metrics,
            lead_time=lt_metrics,
            mttr=mttr_metrics,
            cfr=cfr_metrics,
            overall_score=overall,
            calculated_at=datetime.now(),
        )

    def _calculate_overall_score(self, *categories: str) -> str:
        scores = {"Elite": 4, "High": 3, "Medium": 2, "Low": 1}
        total = sum(scores.get(c, 1) for c in categories)
        avg = total / len(categories)

        if avg >= 3.5:
            return "Elite"
        elif avg >= 2.5:
            return "High"
        elif avg >= 1.5:
            return "Medium"
        else:
            return "Low"

    def generate_report(self, service: str) -> str:
        metrics = self.calculate_for_service(service)

        report = f"""
╔══════════════════════════════════════════════════════════════╗
║              DORA Metrics Report: {service:<26} ║
║              As of: {metrics.calculated_at.strftime('%Y-%m-%d %H:%M'):<38} ║
╠══════════════════════════════════════════════════════════════╣
║  THROUGHPUT METRICS                                          ║
╠══════════════════════════════════════════════════════════════╣
║  Deployment Frequency                                        ║
║    Average per day:  {metrics.deployment_frequency['avg_per_day']:<10.2f}                         ║
║    Total (30 days):  {metrics.deployment_frequency['total_deployments']:<10}                         ║
║    Category:         {metrics.deployment_frequency['category']:<15}                    ║
╠──────────────────────────────────────────────────────────────║
║  Lead Time for Changes                                       ║
║    Median:           {metrics.lead_time.get('p50_lead_time_hours', 0):<10.2f} hours                  ║
║    P90:              {metrics.lead_time.get('p90_lead_time_hours', 0):<10.2f} hours                  ║
║    Category:         {metrics.lead_time.get('category', 'N/A'):<15}                    ║
╠══════════════════════════════════════════════════════════════╣
║  STABILITY METRICS                                           ║
╠══════════════════════════════════════════════════════════════╣
║  MTTR                                                        ║
║    Mean:             {metrics.mttr.get('overall', {}).get('mean_hours', 0):<10.2f} hours                  ║
║    Category:         {metrics.mttr.get('category', 'N/A'):<15}                    ║
╠──────────────────────────────────────────────────────────────║
║  Change Failure Rate                                         ║
║    Rate:             {metrics.cfr.get('cfr_percentage', 0):<10.2f}%                        ║
║    Category:         {metrics.cfr.get('category', 'N/A'):<15}                    ║
╠══════════════════════════════════════════════════════════════╣
║  OVERALL: {metrics.overall_score:<51} ║
╚══════════════════════════════════════════════════════════════╝
"""
        return report
```

---

## 73.7 เครื่องมือ DORA

### Sleuth

Sleuth เป็น SaaS tool ที่วัด DORA metrics อัตโนมัติ

```yaml
# .sleuth/config.yaml
# กำหนด deployment sources
organization: mycompany
project: payment-platform

deployments:
  - name: payment-service
    type: github_pr
    github:
      org: mycompany
      repo: payment-service
      branch: main
    environment: production
    slack_channel: "#deployments-payment"

  - name: user-service
    type: argocd
    argocd:
      server: argocd.mycompany.com
      app_name: user-service-prod
    environment: production

incident_integration:
  type: pagerduty
  service_ids:
    - PABC123  # payment-service PD service
    - PDEF456  # user-service PD service

# Code integration
code_changes:
  - type: github
    org: mycompany
    repos:
      - payment-service
      - user-service
```

### Faros CE (Community Edition)

```yaml
# docker-compose.faros.yaml
version: '3.8'
services:
  faros-ce:
    image: farosai/faros-ce:latest
    ports:
      - "8080:8080"
    environment:
      - GITHUB_TOKEN=${GITHUB_TOKEN}
      - PAGERDUTY_TOKEN=${PAGERDUTY_TOKEN}
      - JIRA_TOKEN=${JIRA_TOKEN}
      - JIRA_URL=${JIRA_URL}
    volumes:
      - ./faros-config:/app/config

  hasura:
    image: hasura/graphql-engine:latest
    ports:
      - "8081:8080"
    environment:
      - HASURA_GRAPHQL_DATABASE_URL=postgres://faros:faros@postgres:5432/faros
      - HASURA_GRAPHQL_ENABLE_CONSOLE=true
    depends_on:
      - postgres

  postgres:
    image: postgres:14
    environment:
      - POSTGRES_USER=faros
      - POSTGRES_PASSWORD=faros
      - POSTGRES_DB=faros
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### Prometheus + Grafana สำหรับ DORA

```yaml
# prometheus-dora-rules.yaml
groups:
  - name: dora_metrics
    rules:
      # Deployment Frequency
      - record: dora:deployment_frequency:rate7d
        expr: |
          sum by (service, environment) (
            rate(deployment_total{environment="production"}[7d])
          ) * 86400  # แปลงเป็น per day

      # สร้าง custom metric จาก deployment events
      - alert: LowDeploymentFrequency
        expr: |
          dora:deployment_frequency:rate7d < 0.143  # น้อยกว่า 1x/week
        for: 7d
        labels:
          severity: warning
        annotations:
          summary: "Deployment frequency ต่ำสำหรับ {{ $labels.service }}"
          description: "{{ $labels.service }} deploy เฉลี่ย {{ $value | humanize }}/วัน"
```

```yaml
# grafana-dora-dashboard.json (excerpt)
{
  "panels": [
    {
      "title": "Deployment Frequency (Elite: ≥1/day)",
      "type": "stat",
      "targets": [
        {
          "expr": "avg_over_time(dora:deployment_frequency:rate7d{service=~\"$service\"}[30d])",
          "legendFormat": "{{service}}"
        }
      ],
      "thresholds": {
        "steps": [
          {"color": "red", "value": 0},
          {"color": "yellow", "value": 0.143},
          {"color": "light-green", "value": 1},
          {"color": "green", "value": 3}
        ]
      }
    },
    {
      "title": "Lead Time for Changes",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, rate(lead_time_seconds_bucket{service=~\"$service\"}[7d])) / 3600",
          "legendFormat": "p50 - {{service}}"
        },
        {
          "expr": "histogram_quantile(0.90, rate(lead_time_seconds_bucket{service=~\"$service\"}[7d])) / 3600",
          "legendFormat": "p90 - {{service}}"
        }
      ]
    }
  ]
}
```

---

## 73.8 กลยุทธ์การปรับปรุง

### ปรับปรุง Deployment Frequency

```markdown
## Practices ที่ช่วยเพิ่ม Deployment Frequency

### 1. Trunk-Based Development
- Commit ตรงไปที่ main branch ทุกวัน
- ใช้ Feature Flags แทน long-lived branches
- เปิดใช้ Feature เมื่อพร้อม ไม่ใช่เมื่อ deploy

### 2. Automated Testing
- Unit test coverage ≥ 80%
- Integration tests อัตโนมัติ
- Contract testing (Pact)

### 3. Small Batch Sizes
- แต่ละ PR ไม่เกิน 400 lines
- ทำ feature ทีละชิ้นเล็กๆ
- Independent deployable components

### 4. Environment Parity
- Dev/Staging/Prod เหมือนกันมากที่สุด
- Infrastructure as Code
- Containerization
```

### ปรับปรุง Lead Time

```markdown
## Practices ที่ลด Lead Time

### 1. Parallelize Pipeline
ก่อน: Lint → Test → Build → Scan → Deploy (sequential)
หลัง: Lint ┐
          ├──→ Build → Scan → Deploy
      Test ┘

### 2. Optimize Build Time
- Docker layer caching
- Dependency caching
- Incremental builds

### 3. Reduce Review Time
- Clear PR guidelines
- Auto-assign reviewers
- Draft PRs สำหรับ early feedback
- Pair programming

### 4. Automate Everything
- Automated testing (ลดเวลา manual QA)
- Automated security scanning
- Automated deployment
```

### ปรับปรุง MTTR

```yaml
# runbook-automation.yaml (Ansible)
# Automate common recovery tasks

- name: Restart failed service
  hosts: "{{ target_hosts }}"
  tasks:
    - name: Check service status
      command: kubectl get pod -n {{ namespace }} -l app={{ service }}
      register: pod_status

    - name: Restart deployment if pods not ready
      command: |
        kubectl rollout restart deployment/{{ service }} -n {{ namespace }}
      when: "'0/1' in pod_status.stdout or '0/2' in pod_status.stdout"

    - name: Wait for rollout
      command: |
        kubectl rollout status deployment/{{ service }} -n {{ namespace }} --timeout=300s

    - name: Notify Slack
      uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: "✅ {{ service }} restarted successfully by automation"
```

### ปรับปรุง Change Failure Rate

```yaml
# progressive-delivery-strategy.yaml
# ใช้ Argo Rollouts สำหรับ canary deployment

apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service
spec:
  replicas: 10
  strategy:
    canary:
      canaryService: payment-service-canary
      stableService: payment-service-stable
      analysis:
        templates:
          - templateName: error-rate-analysis
        startingStep: 2
        args:
          - name: service-name
            value: payment-service
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 25
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
```

---

## 73.9 DORA Benchmark Report Generator

```python
# generate-dora-report.py

def generate_team_benchmark_report(
    teams: List[str],
    calculator: DORACalculator,
    period_days: int = 30,
) -> str:
    """สร้าง benchmark report สำหรับทุกทีม"""

    all_metrics = {}
    for team in teams:
        all_metrics[team] = calculator.calculate_for_service(team)

    report_lines = [
        "=" * 80,
        f"DORA METRICS BENCHMARK REPORT",
        f"Period: Last {period_days} days",
        f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M')}",
        "=" * 80,
        "",
        f"{'Team':<25} {'Dep Freq':<15} {'Lead Time':<15} {'MTTR':<15} {'CFR':<10} {'Overall':<10}",
        "-" * 80,
    ]

    for team, metrics in all_metrics.items():
        df = metrics.deployment_frequency.get("category", "N/A")
        lt = metrics.lead_time.get("category", "N/A")
        mttr = metrics.mttr.get("category", "N/A")
        cfr = metrics.cfr.get("category", "N/A")
        overall = metrics.overall_score

        report_lines.append(
            f"{team:<25} {df:<15} {lt:<15} {mttr:<15} {cfr:<10} {overall:<10}"
        )

    report_lines.extend([
        "",
        "LEGEND:",
        "  Elite  = Performance ระดับ top 25%",
        "  High   = Performance ที่ดี",
        "  Medium = ต้องปรับปรุง",
        "  Low    = ต้องปรับปรุงเร่งด่วน",
    ])

    return "\n".join(report_lines)
```

---

## 73.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: คำนวณ DORA Metrics จากข้อมูลจริง

```python
# exercises/calculate-dora.py
# ข้อมูลตัวอย่างสำหรับฝึก

from datetime import datetime

# Deployments ในเดือนที่ผ่านมา
sample_deployments = [
    {"id": "d1", "deployed_at": datetime(2025, 9, 1, 10, 0), "environment": "production", "service": "api"},
    {"id": "d2", "deployed_at": datetime(2025, 9, 3, 14, 0), "environment": "production", "service": "api"},
    {"id": "d3", "deployed_at": datetime(2025, 9, 5, 9, 0), "environment": "production", "service": "api"},
    {"id": "d4", "deployed_at": datetime(2025, 9, 8, 16, 0), "environment": "production", "service": "api"},
    {"id": "d5", "deployed_at": datetime(2025, 9, 10, 11, 0), "environment": "production", "service": "api"},
    # เพิ่มข้อมูลเพิ่มเติม...
]

# Changes สำหรับ Lead Time
sample_changes = [
    {
        "commit_sha": "abc123",
        "commit_timestamp": datetime(2025, 9, 1, 8, 0),
        "deploy_timestamp": datetime(2025, 9, 1, 10, 0),
        "pr_created_at": datetime(2025, 9, 1, 8, 30),
        "pr_merged_at": datetime(2025, 9, 1, 9, 45),
    },
    # เพิ่มข้อมูลเพิ่มเติม...
]

# Incidents
sample_incidents = [
    {
        "id": "i1",
        "started_at": datetime(2025, 9, 2, 3, 0),
        "resolved_at": datetime(2025, 9, 2, 4, 30),
        "severity": "P1",
        "service": "api",
    },
    # เพิ่มข้อมูลเพิ่มเติม...
]

# Failures (deployments ที่ทำให้เกิดปัญหา)
sample_failures = [
    {"deployment_id": "d2", "type": "rollback", "detected_at": datetime(2025, 9, 3, 16, 0)},
    # เพิ่มข้อมูลเพิ่มเติม...
]

# TODO: คำนวณ metrics ทั้ง 4 ตัว
# 1. Deployment Frequency
df = calculate_deployment_frequency([d["deployed_at"] for d in sample_deployments])
print(f"Deployment Frequency: {df}")

# 2. Lead Time
lt = calculate_lead_time(sample_changes)
print(f"Lead Time: {lt}")

# 3. MTTR
mttr = calculate_mttr(sample_incidents)
print(f"MTTR: {mttr}")

# 4. CFR
cfr = calculate_cfr(sample_deployments, sample_failures)
print(f"Change Failure Rate: {cfr}")
```

### แบบฝึกหัดที่ 2: สร้าง Prometheus Metrics

```go
// exercises/dora-exporter/main.go
// สร้าง Prometheus exporter สำหรับ DORA metrics

package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    deploymentTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "deployment_total",
            Help: "จำนวน deployments ทั้งหมด",
        },
        []string{"service", "environment", "status"},
    )

    leadTimeHistogram = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "lead_time_seconds",
            Help:    "Lead time สำหรับแต่ละ change",
            Buckets: []float64{3600, 7200, 14400, 28800, 86400, 604800},
        },
        []string{"service"},
    )

    mttrHistogram = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "mttr_seconds",
            Help:    "Time to recovery สำหรับแต่ละ incident",
            Buckets: []float64{900, 1800, 3600, 7200, 86400},
        },
        []string{"service", "severity"},
    )
)

func main() {
    prometheus.MustRegister(deploymentTotal, leadTimeHistogram, mttrHistogram)

    // TODO: เพิ่ม logic เก็บข้อมูลจาก GitHub และ PagerDuty

    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

### แบบฝึกหัดที่ 3: วิเคราะห์และปรับปรุง

```markdown
# แบบฝึกหัด: วิเคราะห์ DORA ของทีมคุณ

## ขั้นตอน:

1. รวบรวมข้อมูล deployment ของทีม (ใช้ GitHub หรือ CI/CD tool)
2. คำนวณ 4 DORA metrics
3. ระบุว่าอยู่ในระดับไหน (Elite/High/Medium/Low)
4. วิเคราะห์ bottleneck หลัก
5. สร้าง improvement plan

## Template Improvement Plan:

### ปัญหา: Lead Time สูง (Medium tier)
**Root Cause:** PR review ใช้เวลานาน (เฉลี่ย 3 วัน)

**Action Items:**
- [ ] กำหนด PR review SLA: ตอบใน 4 ชั่วโมง
- [ ] ใช้ CODEOWNERS ให้ auto-assign reviewers ที่ถูกต้อง
- [ ] ลดขนาด PR (เป้าหมาย < 200 lines)
- [ ] เพิ่ม automated checks เพื่อลด manual review

**เป้าหมาย:** ลด Lead Time จาก 3 วัน เป็น 1 วัน ภายใน Q1
**วัดผล:** ทุกสัปดาห์ด้วย DORA dashboard
```

---

## สรุป

DORA Metrics ให้ข้อมูลเชิงลึกที่มีค่ามากสำหรับการปรับปรุง software delivery:

1. **Deployment Frequency** วัดความสามารถในการ deliver ได้บ่อย
2. **Lead Time** วัดความเร็วตั้งแต่ code ถึง production
3. **MTTR** วัดความสามารถในการกู้คืนจากปัญหา
4. **CFR** วัดคุณภาพของ deployment process

องค์กรระดับ Elite มีทั้ง throughput สูงและ stability ดี ซึ่งพิสูจน์ว่า speed และ quality ไม่ได้ trade-off กัน

### ขั้นตอนถัดไป

- ศึกษา [Part 74: Continuous Feedback Loops](./part-74-continuous-feedback.md)
- DORA State of DevOps Report: https://dora.dev
- Accelerate book: https://itrevolution.com/accelerate-book/

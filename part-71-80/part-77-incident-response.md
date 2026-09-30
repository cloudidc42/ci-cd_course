# Part 77: Incident Response Automation

## บทนำ

Incident Response Automation คือการนำ automation มาใช้ในกระบวนการจัดการ incidents เพื่อลด MTTR (Mean Time to Recovery) และลด toil ของทีม on-call

ในบทนี้เราจะเรียนรู้:
- Runbooks as Code
- Auto-remediation
- PagerDuty Integration
- Post-mortem Automation
- Chaos Engineering → Incident Drills
- แบบฝึกหัดปฏิบัติ

---

## 77.1 Runbooks as Code

### Runbook คืออะไร?

Runbook คือเอกสารที่อธิบายขั้นตอนการแก้ไขปัญหาที่รู้จักแล้ว

```
Traditional Runbook (Markdown):
1. SSH เข้า server
2. รัน `ps aux | grep app`
3. ถ้า process ตาย ให้รัน `sudo systemctl restart myapp`
4. ตรวจสอบ logs: `tail -f /var/log/myapp/error.log`

Runbook as Code (Ansible/Python):
- Automated
- Consistent
- Auditable
- Can be tested
```

### Ansible Runbook

```yaml
# runbooks/high-memory-usage.yaml
---
- name: High Memory Usage Runbook
  hosts: "{{ target_hosts | default('all') }}"
  gather_facts: true
  vars:
    memory_threshold: 90
    slack_webhook: "{{ lookup('env', 'SLACK_WEBHOOK') }}"
    service_name: "{{ service | default('unknown') }}"

  tasks:
    - name: Get current memory usage
      command: free -m
      register: memory_output

    - name: Parse memory usage
      set_fact:
        memory_used_pct: "{{ ((ansible_memory_mb.real.used | int) / (ansible_memory_mb.real.total | int) * 100) | int }}"

    - name: Report current status
      debug:
        msg: "Memory usage: {{ memory_used_pct }}% ({{ ansible_memory_mb.real.used }}MB / {{ ansible_memory_mb.real.total }}MB)"

    # Step 1: Find memory-hungry processes
    - name: Find top memory consumers
      shell: ps aux --sort=-%mem | head -20
      register: top_processes

    - name: Send initial report to Slack
      uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body:
          text: |
            🔴 *High Memory Alert*
            Host: {{ inventory_hostname }}
            Service: {{ service_name }}
            Memory: {{ memory_used_pct }}%

            *Top processes:*
            ```{{ top_processes.stdout }}```

    # Step 2: Clear caches if memory > 95%
    - name: Clear page cache if critical
      when: memory_used_pct | int >= 95
      block:
        - name: Sync filesystem
          command: sync

        - name: Clear page cache
          shell: echo 3 > /proc/sys/vm/drop_caches
          become: yes

        - name: Check memory after cache clear
          set_fact:
            memory_after_clear: "{{ ((ansible_memory_mb.real.used | int) / (ansible_memory_mb.real.total | int) * 100) | int }}"

        - name: Report cache clear result
          debug:
            msg: "Memory after cache clear: {{ memory_after_clear }}%"

    # Step 3: Restart service if still high
    - name: Restart service if memory still high
      when:
        - memory_used_pct | int >= memory_threshold | int
        - service_name != 'unknown'
      block:
        - name: Create pre-restart snapshot
          command: "kubectl get pod -n {{ namespace }} -l app={{ service_name }} -o yaml"
          register: pod_snapshot
          ignore_errors: true

        - name: Restart deployment
          command: "kubectl rollout restart deployment/{{ service_name }} -n {{ namespace }}"
          register: restart_result

        - name: Wait for rollout
          command: "kubectl rollout status deployment/{{ service_name }} -n {{ namespace }} --timeout=300s"
          register: rollout_status

        - name: Verify memory after restart
          wait_for:
            timeout: 120

        - name: Report result
          uri:
            url: "{{ slack_webhook }}"
            method: POST
            body_format: json
            body:
              text: |
                ✅ *Runbook Completed*
                Service {{ service_name }} restarted
                Result: {{ rollout_status.rc == 0 | ternary('Success', 'Failed') }}
```

### Python Runbook

```python
# runbooks/database-connection-pool-exhausted.py
"""
Runbook: Database Connection Pool Exhausted
Trigger: Alert "PostgreSQL connection pool > 90% utilized"
"""
import subprocess
import sys
import time
import logging
from dataclasses import dataclass
from typing import Optional
import psycopg2
import requests

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
log = logging.getLogger(__name__)

@dataclass
class RunbookContext:
    service_name: str
    namespace: str
    database_host: str
    database_port: int
    database_name: str
    slack_webhook: str
    pagerduty_routing_key: str
    max_connections: int = 100


def get_current_connections(ctx: RunbookContext) -> dict:
    """ดู connections ปัจจุบัน"""
    try:
        conn = psycopg2.connect(
            host=ctx.database_host,
            port=ctx.database_port,
            dbname=ctx.database_name,
            user="readonly_user",
            password="readonly_pass",
        )
        cursor = conn.cursor()

        # นับ connections ทั้งหมด
        cursor.execute("""
            SELECT
                count(*) as total,
                count(*) FILTER (WHERE state = 'active') as active,
                count(*) FILTER (WHERE state = 'idle') as idle,
                count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_txn,
                max(now() - query_start) as longest_query
            FROM pg_stat_activity
            WHERE datname = %s
        """, (ctx.database_name,))

        row = cursor.fetchone()
        cursor.close()
        conn.close()

        return {
            "total": row[0],
            "active": row[1],
            "idle": row[2],
            "idle_in_transaction": row[3],
            "longest_query_seconds": row[4].total_seconds() if row[4] else 0,
        }
    except Exception as e:
        log.error(f"Error getting connections: {e}")
        return {}


def kill_idle_in_transaction(ctx: RunbookContext, max_age_seconds: int = 300) -> int:
    """Kill connections ที่ idle in transaction นานเกินไป"""
    try:
        conn = psycopg2.connect(
            host=ctx.database_host,
            port=ctx.database_port,
            dbname=ctx.database_name,
            user="admin_user",
            password="admin_pass",
        )
        cursor = conn.cursor()

        cursor.execute("""
            SELECT pg_terminate_backend(pid)
            FROM pg_stat_activity
            WHERE
                datname = %s
                AND state = 'idle in transaction'
                AND (now() - state_change) > interval '%s seconds'
        """, (ctx.database_name, max_age_seconds))

        killed = cursor.rowcount
        conn.commit()
        cursor.close()
        conn.close()

        return killed
    except Exception as e:
        log.error(f"Error killing idle connections: {e}")
        return 0


def restart_application_pods(ctx: RunbookContext) -> bool:
    """Restart application pods เพื่อ reset connection pools"""
    try:
        result = subprocess.run(
            ["kubectl", "rollout", "restart",
             f"deployment/{ctx.service_name}",
             "-n", ctx.namespace],
            capture_output=True,
            text=True,
            timeout=30,
        )

        if result.returncode != 0:
            log.error(f"Restart failed: {result.stderr}")
            return False

        # รอให้ restart เสร็จ
        result = subprocess.run(
            ["kubectl", "rollout", "status",
             f"deployment/{ctx.service_name}",
             "-n", ctx.namespace,
             "--timeout=300s"],
            capture_output=True,
            text=True,
            timeout=310,
        )

        return result.returncode == 0
    except Exception as e:
        log.error(f"Error restarting pods: {e}")
        return False


def send_slack_update(webhook: str, message: str) -> None:
    try:
        requests.post(webhook, json={"text": message}, timeout=10)
    except Exception as e:
        log.error(f"Error sending Slack: {e}")


def run_runbook(ctx: RunbookContext) -> int:
    """Execute runbook - returns exit code (0=success, 1=failed, 2=escalate)"""

    log.info(f"Starting runbook for {ctx.service_name}")
    send_slack_update(ctx.slack_webhook,
        f"🔧 Starting automated runbook for {ctx.service_name} (DB connection pool exhausted)")

    # Step 1: Get current state
    connections = get_current_connections(ctx)
    log.info(f"Current connections: {connections}")

    utilization = connections.get("total", 0) / ctx.max_connections * 100
    send_slack_update(ctx.slack_webhook,
        f"📊 Current state:\n"
        f"- Total: {connections.get('total', 0)}/{ctx.max_connections} ({utilization:.1f}%)\n"
        f"- Active: {connections.get('active', 0)}\n"
        f"- Idle in transaction: {connections.get('idle_in_transaction', 0)}\n"
        f"- Longest query: {connections.get('longest_query_seconds', 0):.0f}s"
    )

    # Step 2: Kill idle in transaction connections (> 5 minutes)
    killed = kill_idle_in_transaction(ctx, max_age_seconds=300)
    log.info(f"Killed {killed} idle in transaction connections")

    time.sleep(30)  # รอให้ connections settle

    # ตรวจสอบอีกครั้ง
    connections_after = get_current_connections(ctx)
    utilization_after = connections_after.get("total", 0) / ctx.max_connections * 100

    if utilization_after < 80:
        send_slack_update(ctx.slack_webhook,
            f"✅ Resolved by killing idle connections\n"
            f"Killed: {killed} connections\n"
            f"Utilization: {utilization_after:.1f}%"
        )
        return 0

    # Step 3: Restart application pods
    log.info("Utilization still high, restarting pods...")
    send_slack_update(ctx.slack_webhook,
        f"⚠️ Still high ({utilization_after:.1f}%) after killing idle connections. Restarting {ctx.service_name} pods...")

    success = restart_application_pods(ctx)

    if success:
        time.sleep(60)
        connections_final = get_current_connections(ctx)
        final_utilization = connections_final.get("total", 0) / ctx.max_connections * 100

        if final_utilization < 80:
            send_slack_update(ctx.slack_webhook,
                f"✅ Resolved by restarting pods\n"
                f"Final utilization: {final_utilization:.1f}%"
            )
            return 0

    # Step 4: Escalate
    send_slack_update(ctx.slack_webhook,
        f"🚨 Automated runbook failed to resolve issue. Escalating to on-call engineer.\n"
        f"Current utilization: {utilization_after:.1f}%"
    )
    return 2


if __name__ == "__main__":
    import os
    ctx = RunbookContext(
        service_name=os.environ["SERVICE_NAME"],
        namespace=os.environ["NAMESPACE"],
        database_host=os.environ["DB_HOST"],
        database_port=int(os.environ.get("DB_PORT", "5432")),
        database_name=os.environ["DB_NAME"],
        slack_webhook=os.environ["SLACK_WEBHOOK"],
        pagerduty_routing_key=os.environ["PAGERDUTY_ROUTING_KEY"],
        max_connections=int(os.environ.get("DB_MAX_CONNECTIONS", "100")),
    )

    exit_code = run_runbook(ctx)
    sys.exit(exit_code)
```

---

## 77.2 Auto-remediation

### Kubernetes Operator สำหรับ Auto-remediation

```go
// auto-remediation/operator.go
package operator

import (
    "context"
    "fmt"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/reconcile"
)

// RemediationPolicy defines auto-remediation rules
type RemediationPolicy struct {
    // Restart deployment if pod crash loop count > threshold
    CrashLoopRestartThreshold int `json:"crashLoopRestartThreshold"`

    // Scale up if CPU > threshold%
    CPUScaleUpThreshold int `json:"cpuScaleUpThreshold"`

    // Max replicas during auto-scale
    MaxAutoScaleReplicas int `json:"maxAutoScaleReplicas"`

    // Cooldown between remediation actions
    CooldownDuration time.Duration `json:"cooldownDuration"`
}

type RemediationController struct {
    client.Client
    lastRemediation map[string]time.Time
}

func (r *RemediationController) Reconcile(
    ctx context.Context,
    req reconcile.Request,
) (reconcile.Result, error) {
    // ดึง pods ทั้งหมดใน namespace
    var pods corev1.PodList
    if err := r.List(ctx, &pods, client.InNamespace(req.Namespace)); err != nil {
        return reconcile.Result{}, err
    }

    for _, pod := range pods.Items {
        // ตรวจสอบ CrashLoopBackOff
        for _, cs := range pod.Status.ContainerStatuses {
            if cs.State.Waiting != nil &&
                cs.State.Waiting.Reason == "CrashLoopBackOff" {
                if cs.RestartCount >= 5 {
                    if err := r.handleCrashLoop(ctx, &pod); err != nil {
                        return reconcile.Result{}, err
                    }
                }
            }

            // ตรวจสอบ OOMKilled
            if cs.LastTerminationState.Terminated != nil &&
                cs.LastTerminationState.Terminated.Reason == "OOMKilled" {
                if err := r.handleOOMKilled(ctx, &pod); err != nil {
                    return reconcile.Result{}, err
                }
            }
        }
    }

    return reconcile.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *RemediationController) handleCrashLoop(
    ctx context.Context,
    pod *corev1.Pod,
) error {
    deploymentName := pod.Labels["app"]
    if deploymentName == "" {
        return nil
    }

    // Cooldown check
    key := fmt.Sprintf("%s/%s", pod.Namespace, deploymentName)
    if lastTime, ok := r.lastRemediation[key]; ok {
        if time.Since(lastTime) < 15*time.Minute {
            return nil  // Still in cooldown
        }
    }

    // Restart deployment
    var deployment appsv1.Deployment
    if err := r.Get(ctx, client.ObjectKey{
        Name:      deploymentName,
        Namespace: pod.Namespace,
    }, &deployment); err != nil {
        return err
    }

    // Add restart annotation
    if deployment.Spec.Template.Annotations == nil {
        deployment.Spec.Template.Annotations = map[string]string{}
    }
    deployment.Spec.Template.Annotations["kubectl.kubernetes.io/restartedAt"] =
        time.Now().Format(time.RFC3339)

    if err := r.Update(ctx, &deployment); err != nil {
        return err
    }

    r.lastRemediation[key] = time.Now()

    // Send notification
    r.sendNotification(fmt.Sprintf(
        "🔄 Auto-restarted deployment %s/%s due to CrashLoopBackOff",
        pod.Namespace, deploymentName,
    ))

    return nil
}

func (r *RemediationController) handleOOMKilled(
    ctx context.Context,
    pod *corev1.Pod,
) error {
    // Auto-scale memory limits
    deploymentName := pod.Labels["app"]
    if deploymentName == "" {
        return nil
    }

    var deployment appsv1.Deployment
    if err := r.Get(ctx, client.ObjectKey{
        Name:      deploymentName,
        Namespace: pod.Namespace,
    }, &deployment); err != nil {
        return err
    }

    // เพิ่ม memory limit 50%
    for i := range deployment.Spec.Template.Spec.Containers {
        container := &deployment.Spec.Template.Spec.Containers[i]
        if container.Name == pod.Name {
            currentMem := container.Resources.Limits.Memory().Value()
            newMem := currentMem * 3 / 2  // 150% of current

            container.Resources.Limits[corev1.ResourceMemory] =
                *resource.NewQuantity(newMem, resource.BinarySI)

            r.sendNotification(fmt.Sprintf(
                "⬆️ Auto-increased memory limit for %s/%s from %dMi to %dMi (OOMKilled)",
                pod.Namespace, deploymentName,
                currentMem/1024/1024, newMem/1024/1024,
            ))
        }
    }

    return r.Update(ctx, &deployment)
}
```

### AWS Lambda Auto-remediation

```python
# lambda/auto-remediation.py
import boto3
import json
import os
from datetime import datetime

def lambda_handler(event, context):
    """
    Auto-remediation Lambda triggered by CloudWatch Alarms
    """
    alarm_name = event.get('detail', {}).get('alarmName', '')
    new_state = event.get('detail', {}).get('state', {}).get('value', '')

    print(f"Alarm: {alarm_name}, State: {new_state}")

    if new_state != 'ALARM':
        return {'statusCode': 200, 'body': 'Not in ALARM state, skipping'}

    # Map alarm → remediation action
    remediation_map = {
        'high-cpu-payment-service': remediate_high_cpu,
        'db-connection-exhausted': remediate_db_connections,
        'high-memory-inventory': remediate_high_memory,
        'unhealthy-targets-alb': remediate_unhealthy_targets,
    }

    for alarm_pattern, remediation_func in remediation_map.items():
        if alarm_pattern in alarm_name:
            result = remediation_func(event)
            send_slack_notification(alarm_name, result)
            return result

    return {'statusCode': 200, 'body': f'No remediation for alarm: {alarm_name}'}


def remediate_high_cpu(event):
    """Scale out ECS service"""
    ecs = boto3.client('ecs')
    cluster = os.environ['ECS_CLUSTER']
    service = 'payment-service'

    # Get current desired count
    response = ecs.describe_services(cluster=cluster, services=[service])
    current_count = response['services'][0]['desiredCount']
    max_count = int(os.environ.get('MAX_REPLICAS', '10'))

    if current_count >= max_count:
        return {
            'statusCode': 200,
            'body': f'Already at max capacity ({current_count})',
        }

    new_count = min(current_count + 2, max_count)

    ecs.update_service(
        cluster=cluster,
        service=service,
        desiredCount=new_count,
    )

    return {
        'statusCode': 200,
        'body': f'Scaled {service} from {current_count} to {new_count}',
    }


def remediate_db_connections(event):
    """Kill idle connections in RDS"""
    rds_data = boto3.client('rds-data')

    result = rds_data.execute_statement(
        resourceArn=os.environ['DB_CLUSTER_ARN'],
        secretArn=os.environ['DB_SECRET_ARN'],
        database=os.environ['DB_NAME'],
        sql="""
            SELECT pg_terminate_backend(pid)
            FROM pg_stat_activity
            WHERE
                state = 'idle in transaction'
                AND (now() - state_change) > interval '5 minutes'
                AND pid != pg_backend_pid()
        """,
    )

    killed = len(result.get('records', []))
    return {
        'statusCode': 200,
        'body': f'Killed {killed} idle in transaction connections',
    }


def remediate_unhealthy_targets(event):
    """Deregister and re-register unhealthy ALB targets"""
    alb = boto3.client('elbv2')
    target_group_arn = os.environ['TARGET_GROUP_ARN']

    # Get unhealthy targets
    health = alb.describe_target_health(TargetGroupArn=target_group_arn)
    unhealthy = [
        t['Target']
        for t in health['TargetHealthDescriptions']
        if t['TargetHealth']['State'] == 'unhealthy'
    ]

    if not unhealthy:
        return {'statusCode': 200, 'body': 'No unhealthy targets'}

    # Deregister unhealthy targets
    alb.deregister_targets(
        TargetGroupArn=target_group_arn,
        Targets=unhealthy,
    )

    # TODO: Restart EC2/ECS instances
    # Re-register after restart

    return {
        'statusCode': 200,
        'body': f'Deregistered {len(unhealthy)} unhealthy targets',
    }


def send_slack_notification(alarm_name: str, result: dict) -> None:
    import urllib.request
    webhook = os.environ.get('SLACK_WEBHOOK')
    if not webhook:
        return

    payload = {
        "text": f"🤖 *Auto-remediation executed*\n"
                f"Alarm: {alarm_name}\n"
                f"Action: {result.get('body', 'unknown')}\n"
                f"Time: {datetime.now().isoformat()}",
    }

    data = json.dumps(payload).encode('utf-8')
    req = urllib.request.Request(webhook, data=data, headers={'Content-Type': 'application/json'})
    urllib.request.urlopen(req)
```

---

## 77.3 PagerDuty Integration

### PagerDuty Events API

```python
# pagerduty/client.py
import requests
from datetime import datetime
from typing import Optional, List
import json

class PagerDutyClient:
    EVENTS_URL = "https://events.pagerduty.com/v2/enqueue"
    API_URL = "https://api.pagerduty.com"

    def __init__(self, routing_key: str, api_key: str):
        self.routing_key = routing_key
        self.api_key = api_key
        self.headers = {
            "Authorization": f"Token token={api_key}",
            "Accept": "application/vnd.pagerduty+json;version=2",
            "Content-Type": "application/json",
        }

    def trigger_incident(
        self,
        summary: str,
        source: str,
        severity: str = "critical",
        custom_details: Optional[dict] = None,
        dedup_key: Optional[str] = None,
        links: Optional[List[dict]] = None,
    ) -> str:
        """สร้าง incident ใหม่"""
        payload = {
            "routing_key": self.routing_key,
            "event_action": "trigger",
            "dedup_key": dedup_key or f"{source}-{datetime.now().timestamp()}",
            "payload": {
                "summary": summary,
                "source": source,
                "severity": severity,  # critical, error, warning, info
                "custom_details": custom_details or {},
                "timestamp": datetime.utcnow().isoformat() + "Z",
            },
        }

        if links:
            payload["links"] = links

        response = requests.post(self.EVENTS_URL, json=payload)
        response.raise_for_status()
        return response.json().get("dedup_key", dedup_key or "")

    def resolve_incident(self, dedup_key: str) -> None:
        """Resolve incident"""
        payload = {
            "routing_key": self.routing_key,
            "event_action": "resolve",
            "dedup_key": dedup_key,
        }
        response = requests.post(self.EVENTS_URL, json=payload)
        response.raise_for_status()

    def acknowledge_incident(self, dedup_key: str) -> None:
        """Acknowledge incident"""
        payload = {
            "routing_key": self.routing_key,
            "event_action": "acknowledge",
            "dedup_key": dedup_key,
        }
        response = requests.post(self.EVENTS_URL, json=payload)
        response.raise_for_status()

    def add_note(self, incident_id: str, content: str) -> None:
        """เพิ่ม note ใน incident"""
        response = requests.post(
            f"{self.API_URL}/incidents/{incident_id}/notes",
            headers=self.headers,
            json={"note": {"content": content}},
        )
        response.raise_for_status()

    def get_on_call(self, schedule_id: str) -> Optional[dict]:
        """ดูว่าใคร on-call ตอนนี้"""
        response = requests.get(
            f"{self.API_URL}/oncalls",
            headers=self.headers,
            params={
                "schedule_ids[]": schedule_id,
                "since": datetime.utcnow().isoformat() + "Z",
                "until": datetime.utcnow().isoformat() + "Z",
            },
        )
        response.raise_for_status()

        oncalls = response.json().get("oncalls", [])
        if oncalls:
            return oncalls[0]["user"]
        return None
```

### PagerDuty Webhook Handler

```python
# pagerduty/webhook_handler.py
from flask import Flask, request, jsonify
import hmac
import hashlib
import os

app = Flask(__name__)
pd_client = PagerDutyClient(
    routing_key=os.environ["PAGERDUTY_ROUTING_KEY"],
    api_key=os.environ["PAGERDUTY_API_KEY"],
)

@app.route('/webhooks/pagerduty', methods=['POST'])
def handle_pagerduty_webhook():
    # ตรวจสอบ signature
    signature = request.headers.get('X-PagerDuty-Signature')
    if not verify_signature(request.data, signature):
        return jsonify({"error": "Invalid signature"}), 401

    messages = request.json.get('messages', [])

    for message in messages:
        event_type = message.get('event', '')
        incident = message.get('incident', {})

        if event_type == 'incident.trigger':
            handle_incident_triggered(incident)
        elif event_type == 'incident.acknowledge':
            handle_incident_acknowledged(incident)
        elif event_type == 'incident.resolve':
            handle_incident_resolved(incident)
        elif event_type == 'incident.escalate':
            handle_incident_escalated(incident)

    return jsonify({"status": "ok"})


def handle_incident_triggered(incident: dict) -> None:
    """เมื่อ incident เกิดขึ้น - รัน automatic diagnostics"""
    service_name = incident.get('service', {}).get('name', '')
    incident_id = incident.get('id', '')
    title = incident.get('title', '')

    print(f"Incident triggered: {title} ({incident_id})")

    # รัน automatic diagnostics
    diagnostics = run_diagnostics(service_name)

    # เพิ่ม diagnostic info เป็น note
    pd_client.add_note(
        incident_id,
        f"""🤖 Automatic Diagnostics:

Service: {service_name}
Time: {datetime.now().isoformat()}

{diagnostics}

---
Runbook: https://runbooks.mycompany.com/{service_name}
Dashboard: https://grafana.mycompany.com/d/{service_name}
"""
    )

    # ลอง auto-remediation
    remediation_result = attempt_auto_remediation(service_name, incident)

    if remediation_result.get('resolved'):
        pd_client.resolve_incident(incident.get('dedup_key', ''))
        pd_client.add_note(incident_id,
            f"✅ Auto-remediation successful: {remediation_result.get('action')}")
    else:
        pd_client.add_note(incident_id,
            f"❌ Auto-remediation failed: {remediation_result.get('reason')}\n"
            f"Manual intervention required.")


def run_diagnostics(service_name: str) -> str:
    """รัน diagnostics อัตโนมัติ"""
    lines = []

    # Check pod status
    pod_status = check_kubernetes_pods(service_name)
    lines.append(f"Pod Status:\n{pod_status}")

    # Check recent errors
    recent_errors = get_recent_sentry_errors(service_name)
    lines.append(f"\nRecent Errors (last 5 min):\n{recent_errors}")

    # Check metrics
    metrics = get_prometheus_metrics(service_name)
    lines.append(f"\nKey Metrics:\n{metrics}")

    return "\n".join(lines)


def verify_signature(payload: bytes, signature: str) -> bool:
    secret = os.environ.get("PAGERDUTY_WEBHOOK_SECRET", "")
    computed = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256,
    ).hexdigest()
    return hmac.compare_digest(f"v1={computed}", signature or "")
```

---

## 77.4 Post-mortem Automation

### Auto-generate Post-mortem Template

```python
# postmortem/generator.py
from datetime import datetime, timedelta
from typing import List, Optional
import jinja2
import requests

class PostMortemGenerator:
    def __init__(
        self,
        pagerduty: PagerDutyClient,
        github_token: str,
        github_org: str,
    ):
        self.pd = pagerduty
        self.github_token = github_token
        self.github_org = github_org

    def generate(self, incident_id: str) -> str:
        """สร้าง post-mortem document อัตโนมัติ"""
        # ดึงข้อมูล incident
        incident = self.pd.get_incident(incident_id)
        timeline = self.pd.get_incident_timeline(incident_id)
        alerts = self.pd.get_incident_alerts(incident_id)

        # ดึง deployments ที่เกิดขึ้นก่อน incident
        service = incident['service']['name']
        start_time = datetime.fromisoformat(incident['created_at'].replace('Z', '+00:00'))
        deployments = self.get_recent_deployments(service, start_time)

        # สร้างจาก template
        template = jinja2.Template(POST_MORTEM_TEMPLATE)
        return template.render(
            incident=incident,
            timeline=timeline,
            alerts=alerts,
            deployments=deployments,
            generated_at=datetime.now().isoformat(),
        )

    def create_github_issue(self, incident_id: str) -> str:
        """สร้าง post-mortem ใน GitHub Issues"""
        content = self.generate(incident_id)
        incident = self.pd.get_incident(incident_id)

        headers = {
            "Authorization": f"token {self.github_token}",
            "Accept": "application/vnd.github.v3+json",
        }

        response = requests.post(
            f"https://api.github.com/repos/{self.github_org}/post-mortems/issues",
            headers=headers,
            json={
                "title": f"Post-mortem: {incident['title']} ({incident_id})",
                "body": content,
                "labels": ["post-mortem", f"severity-{incident.get('urgency', 'high')}"],
                "assignees": [incident['assignments'][0]['assignee']['name']] if incident.get('assignments') else [],
            },
        )

        return response.json().get('html_url', '')

    def get_recent_deployments(
        self,
        service: str,
        before: datetime,
        lookback_hours: int = 4,
    ) -> List[dict]:
        """ดู deployments ที่เกิดก่อน incident"""
        since = before - timedelta(hours=lookback_hours)

        # ดึงจาก GitHub Deployments API
        headers = {
            "Authorization": f"token {self.github_token}",
            "Accept": "application/vnd.github.v3+json",
        }

        response = requests.get(
            f"https://api.github.com/repos/{self.github_org}/{service}/deployments",
            headers=headers,
            params={"environment": "production"},
        )

        deployments = []
        for dep in response.json():
            created_at = datetime.fromisoformat(dep['created_at'].replace('Z', '+00:00'))
            if since <= created_at <= before:
                deployments.append({
                    "sha": dep['sha'][:8],
                    "created_at": created_at.isoformat(),
                    "creator": dep['creator']['login'],
                })

        return deployments


POST_MORTEM_TEMPLATE = """
# Post-Mortem: {{ incident.title }}

**Date:** {{ incident.created_at | truncate(10, true, '') }}
**Incident ID:** {{ incident.id }}
**Severity:** {{ incident.urgency | upper }}
**Duration:** {{ incident.duration_hours }}h {{ incident.duration_minutes }}m
**Generated:** {{ generated_at }}

---

## Summary

> ⚠️ TODO: เพิ่มสรุปสั้นๆ ว่าเกิดอะไรขึ้น impact กว้างแค่ไหน และแก้ไขอย่างไร

**Impact:** [TODO: ระบุ users ที่ได้รับผลกระทบ]
**Root Cause:** [TODO: ระบุ root cause]

---

## Timeline

| เวลา | เหตุการณ์ |
|------|----------|
{% for event in timeline %}
| {{ event.time }} | {{ event.description }} |
{% endfor %}

---

## Deployments ก่อนเกิด Incident (4 ชั่วโมง)

{% if deployments %}
| เวลา | Commit | ผู้ deploy |
|------|--------|-----------|
{% for dep in deployments %}
| {{ dep.created_at }} | {{ dep.sha }} | {{ dep.creator }} |
{% endfor %}
{% else %}
ไม่มี deployment ในช่วงเวลานี้
{% endif %}

---

## Root Cause Analysis

### ปัญหาเกิดจาก

[TODO: อธิบาย root cause]

### ทำไมถึงเกิด

[TODO: five whys analysis]

1. ทำไม? →
2. ทำไม? →
3. ทำไม? →
4. ทำไม? →
5. ทำไม? → **Root cause**

---

## Corrective Actions

| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| TODO   | TODO  | TODO     | TODO     |

---

## Lessons Learned

### สิ่งที่ทำได้ดี
- [TODO]

### สิ่งที่ต้องปรับปรุง
- [TODO]

---

*Post-mortem นี้ถูกสร้างอัตโนมัติจาก PagerDuty incident data*
*กรุณาเพิ่มข้อมูลในส่วน [TODO] ก่อน publish*
"""
```

---

## 77.5 Chaos Engineering → Incident Drill

### Chaos Monkey สำหรับ Kubernetes

```yaml
# chaos-mesh-experiments.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: payment-service-pod-kill
  namespace: chaos-testing
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      "app": "payment-service"
  scheduler:
    cron: "@midnight"  # ทดสอบทุกคืน
```

### Incident Drill Script

```python
# incident-drill.py
"""
Automated Incident Drill
เป้าหมาย: ทดสอบกระบวนการ incident response
"""
import time
import subprocess
import requests
from datetime import datetime

class IncidentDrill:
    def __init__(self, service: str, namespace: str, pd: PagerDutyClient):
        self.service = service
        self.namespace = namespace
        self.pd = pd
        self.drill_start = None
        self.metrics = {}

    def run_drill(self, scenario: str) -> dict:
        """รัน incident drill scenario"""
        print(f"\n{'='*60}")
        print(f"🎯 Starting Incident Drill: {scenario}")
        print(f"Service: {self.service}")
        print(f"Time: {datetime.now().isoformat()}")
        print(f"{'='*60}\n")

        self.drill_start = time.time()

        # เริ่ม drill
        dedup_key = self.pd.trigger_incident(
            summary=f"[DRILL] {scenario} - {self.service}",
            source="incident-drill",
            severity="error",
            custom_details={
                "drill": True,
                "scenario": scenario,
                "service": self.service,
            },
            dedup_key=f"drill-{self.service}-{int(time.time())}",
        )

        # Inject failure
        print(f"💥 Injecting failure: {scenario}")
        self.inject_failure(scenario)

        # Wait for team response
        print("⏳ Waiting for on-call response...")
        response_time = self.wait_for_acknowledgment(dedup_key, timeout=600)
        self.metrics['response_time'] = response_time

        if response_time:
            print(f"✅ On-call responded in {response_time:.0f}s")
        else:
            print("❌ No response within 10 minutes")

        # Wait for resolution
        print("⏳ Waiting for resolution...")
        resolution_time = self.wait_for_resolution(dedup_key, timeout=3600)
        self.metrics['resolution_time'] = resolution_time

        # Cleanup
        self.cleanup_failure(scenario)
        self.pd.resolve_incident(dedup_key)

        # Generate report
        return self.generate_report(scenario)

    def inject_failure(self, scenario: str) -> None:
        """Inject failure ตาม scenario"""
        if scenario == "pod_crash":
            subprocess.run([
                "kubectl", "delete", "pod",
                "-n", self.namespace,
                "-l", f"app={self.service}",
                "--all",
            ])
        elif scenario == "network_latency":
            subprocess.run([
                "kubectl", "exec",
                "-n", self.namespace,
                f"deployment/{self.service}",
                "--",
                "tc", "qdisc", "add", "dev", "eth0",
                "root", "netem", "delay", "500ms",
            ])
        elif scenario == "memory_pressure":
            subprocess.run([
                "kubectl", "exec",
                "-n", self.namespace,
                f"deployment/{self.service}",
                "--",
                "stress", "--vm", "1", "--vm-bytes", "500M", "&",
            ])

    def wait_for_acknowledgment(self, dedup_key: str, timeout: int) -> Optional[float]:
        """รอให้ on-call acknowledge"""
        start = time.time()
        while time.time() - start < timeout:
            incident = self.pd.get_incident_by_dedup_key(dedup_key)
            if incident and incident.get('status') == 'acknowledged':
                return time.time() - start
            time.sleep(10)
        return None

    def generate_report(self, scenario: str) -> dict:
        """สรุปผลการ drill"""
        return {
            "scenario": scenario,
            "service": self.service,
            "drill_date": datetime.now().isoformat(),
            "response_time_seconds": self.metrics.get('response_time'),
            "resolution_time_seconds": self.metrics.get('resolution_time'),
            "slo_impact": self.calculate_slo_impact(),
            "passed": (
                self.metrics.get('response_time', 999) <= 300 and  # < 5 min
                self.metrics.get('resolution_time', 9999) <= 1800  # < 30 min
            ),
        }
```

---

## 77.6 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Ansible Runbook

```yaml
# exercises/runbook-template.yaml
# สร้าง runbook สำหรับ service ของคุณ

# TODO: เลือกปัญหาที่เกิดบ่อยที่สุด
# ตัวอย่าง: High error rate, Slow response, Database timeout

- name: "[ชื่อปัญหา] Runbook"
  hosts: localhost
  tasks:
    - name: "Step 1: Check current status"
      # TODO: เพิ่ม task ตรวจสอบสถานะ
      debug:
        msg: "TODO"

    - name: "Step 2: Diagnose"
      # TODO: เพิ่ม task วิเคราะห์ปัญหา
      debug:
        msg: "TODO"

    - name: "Step 3: Remediate"
      # TODO: เพิ่ม task แก้ไขปัญหา
      debug:
        msg: "TODO"

    - name: "Step 4: Verify"
      # TODO: ตรวจสอบว่าแก้ไขสำเร็จ
      debug:
        msg: "TODO"

    - name: "Step 5: Notify"
      # TODO: แจ้ง Slack/PagerDuty
      debug:
        msg: "TODO"
```

### แบบฝึกหัดที่ 2: ตั้งค่า PagerDuty

```bash
# สร้าง PagerDuty integration

# 1. สร้าง service ใน PagerDuty (https://app.pagerduty.com)
# 2. เพิ่ม Integration "Events API v2"
# 3. คัดลอก Integration Key

# ทดสอบส่ง event
ROUTING_KEY="your-integration-key"

curl -X POST https://events.pagerduty.com/v2/enqueue \
  -H "Content-Type: application/json" \
  -d "{
    \"routing_key\": \"$ROUTING_KEY\",
    \"event_action\": \"trigger\",
    \"dedup_key\": \"test-$(date +%s)\",
    \"payload\": {
      \"summary\": \"Test Alert from CI/CD Course\",
      \"source\": \"test-script\",
      \"severity\": \"info\"
    }
  }"

# Resolve test incident
curl -X POST https://events.pagerduty.com/v2/enqueue \
  -H "Content-Type: application/json" \
  -d "{
    \"routing_key\": \"$ROUTING_KEY\",
    \"event_action\": \"resolve\",
    \"dedup_key\": \"test-[YOUR_TIMESTAMP]\"
  }"
```

### แบบฝึกหัดที่ 3: ออกแบบ Post-mortem Process

```markdown
# Post-mortem Process Design

## สำหรับ Incident ของ Service: [ชื่อ service]

### 1. Information ที่ต้องการ
- Timeline ของ incident
- Alert ที่ fire
- On-call response time
- Deployments ก่อน incident
- Logs และ metrics
- Root cause

### 2. Template ที่จะใช้
สร้าง template ที่เหมาะสมกับ service ของคุณ:

```markdown
# Post-mortem: [ชื่อ]

## Impact
- Users affected: [จำนวน]
- Duration: [เวลา]
- Revenue impact: [จำนวน]

## Root Cause
[TODO]

## Timeline
[TODO]

## Action Items
| Priority | Action | Owner | Due |
|----------|--------|-------|-----|
| P1 | | | |
```

### 3. Automation ที่ต้องการ
- [ ] Auto-generate template เมื่อ incident resolve
- [ ] Auto-populate timeline จาก PagerDuty
- [ ] Auto-populate deployments จาก GitHub
- [ ] Send reminder 48h หลัง incident
```

---

## สรุป

Incident Response Automation ช่วยลด MTTR และ toil ของทีม:

1. **Runbooks as Code** ทำให้ขั้นตอนแก้ไขปัญหาเป็น automation และ consistent
2. **Auto-remediation** แก้ปัญหาที่รู้จักโดยอัตโนมัติโดยไม่ต้องรอ human
3. **PagerDuty Integration** เชื่อม alerts กับ on-call และ incident management
4. **Post-mortem Automation** ลด friction ในการทำ post-mortem
5. **Incident Drills** ฝึกซ้อมให้ทีมพร้อมรับมือ incidents จริง

เป้าหมายไม่ใช่แค่แก้ปัญหาเร็ว แต่คือป้องกันไม่ให้เกิดปัญหาซ้ำด้วย

### ขั้นตอนถัดไป

- ศึกษา [Part 78: Change Management & Rollback](./part-78-change-management.md)
- PagerDuty API docs: https://developer.pagerduty.com
- Chaos Mesh: https://chaos-mesh.org/docs

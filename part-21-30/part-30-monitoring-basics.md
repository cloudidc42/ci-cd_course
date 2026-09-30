# Part 30: Monitoring & Observability Basics

## สารบัญ
1. [Observability คืออะไร?](#observability-คืออะไร)
2. [The Three Pillars of Observability](#the-three-pillars-of-observability)
3. [Prometheus Setup](#prometheus-setup)
4. [Grafana Dashboards](#grafana-dashboards)
5. [Alerting Rules และ Alertmanager](#alerting-rules-และ-alertmanager)
6. [Application Metrics Instrumentation](#application-metrics-instrumentation)
7. [Health Check Endpoints](#health-check-endpoints)
8. [SLI, SLO, and SLA Concepts](#sli-slo-and-sla-concepts)
9. [Uptime Monitoring](#uptime-monitoring)
10. [Monitoring ใน CI/CD Pipeline](#monitoring-ใน-cicd-pipeline)
11. [Distributed Tracing](#distributed-tracing)
12. [Log Management](#log-management)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Observability คืออะไร?

**Observability** (การสังเกตได้) คือความสามารถในการเข้าใจ internal state ของระบบจาก external outputs ของมัน

### ทำไม Observability ถึงสำคัญ?

ลองนึกภาพ:
- Production system ทำงานช้าลงอย่างกะทันหัน
- ผู้ใช้รายงาน error แต่ไม่รู้ว่าเกิดที่ไหน
- หลัง deploy ใหม่แล้ว performance แย่ลงหรือเปล่า?

**โดยไม่มี Observability**: คุณต้องเดาสาเหตุ, ลอง restart service ดูว่าจะดีขึ้นไหม, ใช้เวลานานกว่าจะ diagnose ปัญหา

**มี Observability ที่ดี**: คุณสามารถ:
1. ตรวจพบปัญหาก่อนผู้ใช้จะรู้สึก
2. ระบุ root cause ได้รวดเร็ว
3. เข้าใจ impact ของการเปลี่ยนแปลง
4. ตัดสินใจบน data แทนที่จะเดา

### Monitoring vs Observability

```
Monitoring (แบบเก่า):
→ บอกว่า "มีปัญหา" (known unknowns)
→ Dashboard แสดง metrics ที่กำหนดไว้ล่วงหน้า
→ Alert เมื่อค่าเกิน threshold

Observability (แบบใหม่):
→ บอกว่า "ปัญหาคืออะไรและทำไม" (unknown unknowns)
→ สามารถถามคำถามใหม่ๆ เกี่ยวกับระบบได้
→ ค้นหาปัญหาที่ไม่เคยรู้ว่าจะเกิดขึ้น
```

### Observability Maturity Model

```
Level 0: ไม่มี monitoring
Level 1: Infrastructure metrics (CPU, Memory, Disk)
Level 2: Application metrics (request rate, error rate)
Level 3: Distributed tracing
Level 4: Full observability (Logs + Metrics + Traces + กวาดมาใช้ร่วมกัน)
```

---

## The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────┐
│                 Three Pillars of Observability                 │
│                                                                 │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐    │
│  │     Logs      │   │    Metrics    │   │    Traces     │    │
│  │               │   │               │   │               │    │
│  │ Structured    │   │ Numerical     │   │ Request flow  │    │
│  │ events that   │   │ measurements  │   │ through       │    │
│  │ describe what │   │ over time     │   │ distributed   │    │
│  │ happened      │   │               │   │ system        │    │
│  │               │   │               │   │               │    │
│  │ "What?"       │   │ "How much?"   │   │ "Where?"      │    │
│  └───────────────┘   └───────────────┘   └───────────────┘    │
│                                                                 │
│  Tools: ELK,         Tools: Prometheus,  Tools: Jaeger,       │
│         Loki,               Datadog,           Zipkin,         │
│         CloudWatch          VictoriaMetrics    Tempo           │
└─────────────────────────────────────────────────────────────────┘
```

### Logs

Logs คือ records ของ events ที่เกิดขึ้นในระบบ:

```json
// ตัวอย่าง Structured Log
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "traceId": "abc123",
  "spanId": "def456",
  "userId": "user-789",
  "message": "Payment processing failed",
  "error": {
    "type": "PaymentGatewayException",
    "message": "Connection timeout",
    "stack": "..."
  },
  "duration": 5234,
  "requestId": "req-xyz"
}
```

### Metrics

Metrics คือ numerical measurements ที่วัดตลอดเวลา:

```
# ตัวอย่าง Metrics
http_requests_total{method="GET", path="/api/users", status="200"} 12453
http_request_duration_seconds{method="GET", path="/api/users"} 0.125
process_cpu_usage 0.45
memory_usage_bytes 536870912
database_connections_active 15
```

### Traces

Traces แสดงให้เห็น request flow ผ่านระบบ:

```
Request: GET /checkout
│
├── [0ms] API Gateway (5ms)
│   └── Auth check passed
│
├── [5ms] Cart Service (12ms)
│   ├── Get cart items
│   └── Cache hit
│
├── [17ms] Inventory Service (45ms)
│   ├── Check item availability
│   └── Database query (40ms)  ← ช้า!
│
├── [62ms] Payment Service (8ms)
│   └── Validate payment
│
└── [70ms] Order Service (15ms)
    ├── Create order
    └── Send confirmation email
```

---

## Prometheus Setup

**Prometheus** เป็น open-source monitoring system ที่เก็บ time-series metrics

### สถาปัตยกรรม Prometheus

```
┌─────────────────────────────────────────────────────────────────┐
│                    Prometheus Architecture                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Prometheus Server                    │   │
│  │                                                         │   │
│  │  ┌──────────┐    ┌──────────────┐    ┌─────────────┐  │   │
│  │  │  Scrape  │    │ Storage      │    │  Query      │  │   │
│  │  │  Engine  │───▶│ (TSDB)       │◀───│  Engine     │  │   │
│  │  └──────────┘    └──────────────┘    └─────────────┘  │   │
│  │       │                                      │          │   │
│  └───────┼──────────────────────────────────────┼──────────┘   │
│          │                                      │              │
│  ┌───────▼───────┐    ┌──────────────┐    ┌────▼────────┐     │
│  │ Applications  │    │   Alerting   │    │  Grafana    │     │
│  │ (with         │    │   Rules      │    │ (Dashboards)│     │
│  │  /metrics)    │    │              │    └─────────────┘     │
│  └───────────────┘    └──────┬───────┘                        │
│                              │                                  │
│                       ┌──────▼───────┐                        │
│                       │ Alertmanager │                        │
│                       └─────────────┘                        │
└─────────────────────────────────────────────────────────────────┘
```

### ติดตั้ง Prometheus ด้วย Docker Compose

```yaml
# docker-compose.yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secure-password
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    networks:
      - monitoring
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/config.yml:/etc/alertmanager/config.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/config.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring

  # Node Exporter (system metrics)
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  # cAdvisor (container metrics)
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:

networks:
  monitoring:
    driver: bridge
```

### prometheus.yml Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s          # ดึง metrics ทุก 15 วินาที
  evaluation_interval: 15s      # ประเมิน alerting rules ทุก 15 วินาที
  scrape_timeout: 10s

# Alertmanager config
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Rules files
rule_files:
  - "/etc/prometheus/rules/*.yml"

# Scrape configs
scrape_configs:
  # Prometheus ตัวเอง
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Node Exporter (system metrics)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):.*'
        replacement: '$1'

  # cAdvisor (container metrics)
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  # My Application
  - job_name: 'my-app'
    metrics_path: '/metrics'
    scrape_interval: 10s
    static_configs:
      - targets:
          - 'my-app:3000'
        labels:
          environment: 'production'
          team: 'backend'

  # Kubernetes pods (auto-discovery)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name

  # Blackbox Exporter (external checks)
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://example.com
          - https://api.example.com/health
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

### ติดตั้ง Prometheus บน Kubernetes ด้วย Helm

```bash
# เพิ่ม prometheus-community repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager + Exporters)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml

# prometheus-values.yaml
cat > prometheus-values.yaml << 'EOF'
# Prometheus configuration
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

# Grafana configuration
grafana:
  adminPassword: "secure-password"
  persistence:
    enabled: true
    size: 10Gi

# Alertmanager configuration
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

# Default rules
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    k8s: true
    kubeApiserver: true
    node: true
    prometheus: true
EOF
```

---

## Grafana Dashboards

### ตั้งค่า Grafana Data Source

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: '15s'
      queryTimeout: '60s'
    editable: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: true
```

### Provisioning Dashboards อัตโนมัติ

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 60
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
      foldersFromFilesStructure: true
```

### Dashboard JSON (Node.js Application)

```json
// grafana/provisioning/dashboards/nodejs-app.json
{
  "title": "Node.js Application Metrics",
  "uid": "nodejs-app",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 },
      "targets": [
        {
          "expr": "rate(http_requests_total{job='my-app'}[5m])",
          "legendFormat": "{{method}} {{path}} {{status}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "gridPos": { "h": 4, "w": 6, "x": 12, "y": 0 },
      "targets": [
        {
          "expr": "rate(http_requests_total{job='my-app',status=~'5..'}[5m]) / rate(http_requests_total{job='my-app'}[5m]) * 100",
          "legendFormat": "Error Rate %"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 1 },
              { "color": "red", "value": 5 }
            ]
          }
        }
      }
    },
    {
      "title": "P95 Latency",
      "type": "graph",
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 8 },
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job='my-app'}[5m])) by (le, path))",
          "legendFormat": "p95 {{path}}"
        },
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{job='my-app'}[5m])) by (le, path))",
          "legendFormat": "p50 {{path}}"
        }
      ]
    },
    {
      "title": "Active Connections",
      "type": "stat",
      "targets": [
        {
          "expr": "nodejs_active_connections{job='my-app'}"
        }
      ]
    },
    {
      "title": "Memory Usage",
      "type": "graph",
      "targets": [
        {
          "expr": "process_resident_memory_bytes{job='my-app'}",
          "legendFormat": "RSS Memory"
        },
        {
          "expr": "nodejs_heap_size_used_bytes{job='my-app'}",
          "legendFormat": "Heap Used"
        }
      ]
    }
  ]
}
```

### PromQL Query Examples

```promql
# Request rate (requests per second)
rate(http_requests_total[5m])

# Error rate percentage
rate(http_requests_total{status=~"5.."}[5m]) 
  / rate(http_requests_total[5m]) * 100

# P99 latency
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)
)

# CPU usage percentage
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) 
  / node_memory_MemTotal_bytes * 100

# Disk usage
100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) 
  / node_filesystem_size_bytes{mountpoint="/"})

# Container CPU throttling
sum(rate(container_cpu_cfs_throttled_periods_total[5m])) by (container)
  / sum(rate(container_cpu_cfs_periods_total[5m])) by (container)

# Kubernetes pod restarts
sum(kube_pod_container_status_restarts_total) by (pod, namespace)

# Active deployments
kube_deployment_status_replicas_available
  / kube_deployment_spec_replicas

# Number of pods not ready
sum(kube_pod_status_ready{condition="false"}) by (namespace)
```

---

## Alerting Rules และ Alertmanager

### สร้าง Alerting Rules

```yaml
# prometheus/rules/alerts.yml
groups:
  - name: application-alerts
    interval: 30s
    rules:
      # High Error Rate
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{status=~"5.."}[5m])
            / rate(http_requests_total[5m])
          ) * 100 > 5
        for: 5m  # ต้องเป็นจริงนาน 5 นาทีก่อน alert
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: |
            Error rate is {{ $value | humanize }}% 
            on {{ $labels.job }} (threshold: 5%)
          runbook: "https://wiki.example.com/runbooks/high-error-rate"
          dashboard: "https://grafana.example.com/d/nodejs-app"
      
      # High Latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)
          ) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected on {{ $labels.path }}"
          description: "P95 latency is {{ $value | humanizeDuration }} on {{ $labels.path }}"
      
      # Service Down
      - alert: ServiceDown
        expr: up{job="my-app"} == 0
        for: 1m
        labels:
          severity: critical
          pagerduty: 'true'
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."
      
      # Database connection pool exhausted
      - alert: DatabaseConnectionPoolExhausted
        expr: |
          db_pool_connections_idle{job="my-app"} == 0
          and
          db_pool_connections_total{job="my-app"} >= db_pool_max_connections{job="my-app"}
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool exhausted"
          description: "All {{ $value }} database connections are in use"

  - name: infrastructure-alerts
    rules:
      # High CPU Usage
      - alert: HighCPUUsage
        expr: |
          100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}% on {{ $labels.instance }}"
      
      # Low Disk Space
      - alert: LowDiskSpace
        expr: |
          (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Only {{ $value | humanize }}% disk space remaining"
      
      # High Memory Usage
      - alert: HighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}%"

  - name: kubernetes-alerts
    rules:
      # Pod CrashLooping
      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"
          description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} has restarted {{ $value | humanize }} times in 15 minutes"
      
      # Deployment not available
      - alert: DeploymentNotAvailable
        expr: |
          kube_deployment_status_replicas_available / kube_deployment_spec_replicas < 0.5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Deployment {{ $labels.deployment }} has less than 50% replicas available"
```

### Alertmanager Configuration

```yaml
# alertmanager/config.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'app-password'
  slack_api_url: 'https://hooks.slack.com/services/...'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  # Default receiver
  receiver: 'team-notifications'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s          # รอก่อน send alert แรก
  group_interval: 5m       # รอก่อน send alert ใหม่ในกลุ่มเดิม
  repeat_interval: 4h      # ส่ง alert ซ้ำทุก 4 ชั่วโมง
  
  routes:
    # Critical alerts ไปยัง PagerDuty
    - match:
        severity: critical
        pagerduty: 'true'
      receiver: 'pagerduty-critical'
      continue: true  # ยังคง route ต่อไปยัง default receiver ด้วย
    
    # Critical alerts
    - match:
        severity: critical
      receiver: 'slack-critical'
      group_wait: 10s
      repeat_interval: 1h
    
    # Warning alerts
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_wait: 1m
      repeat_interval: 2h
    
    # Backend team alerts
    - match:
        team: backend
      receiver: 'backend-team'
    
    # Silence alerts ระหว่างนอกเวลาทำงาน (สำหรับ non-critical)
    - match_re:
        severity: warning
      receiver: 'slack-warnings'
      active_time_intervals:
        - business-hours

time_intervals:
  - name: business-hours
    time_intervals:
      - times:
          - start_time: '09:00'
            end_time: '17:00'
        weekdays: ['monday:friday']
        location: 'Asia/Bangkok'

# Receivers
receivers:
  - name: 'team-notifications'
    slack_configs:
      - channel: '#alerts'
        title: '{{ .CommonAnnotations.summary }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Severity:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
          {{ end }}

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        color: 'danger'
        title: '🔥 CRITICAL: {{ .CommonAnnotations.summary }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook }}
          *Dashboard:* {{ .Annotations.dashboard }}
          {{ end }}

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#alerts-warnings'
        color: 'warning'
        title: '⚠️ WARNING: {{ .CommonAnnotations.summary }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          {{ end }}

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-integration-key'
        description: '{{ .CommonAnnotations.summary }}'
        severity: critical
        details:
          summary: '{{ .CommonAnnotations.summary }}'
          description: '{{ .CommonAnnotations.description }}'

  - name: 'backend-team'
    email_configs:
      - to: 'backend-team@example.com'
        send_resolved: true
        subject: '[{{ .Status | toUpper }}] {{ .CommonAnnotations.summary }}'
        html: |
          <h2>{{ .CommonAnnotations.summary }}</h2>
          <p>{{ .CommonAnnotations.description }}</p>
          {{ if .CommonAnnotations.runbook }}
          <p><a href="{{ .CommonAnnotations.runbook }}">Runbook</a></p>
          {{ end }}

# Inhibit rules (ยกเลิก alert เมื่อ critical alert เกิดขึ้น)
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

---

## Application Metrics Instrumentation

### Node.js Metrics ด้วย prom-client

```javascript
// src/metrics.js
const client = require('prom-client');

// Default metrics (CPU, memory, etc.)
client.collectDefaultMetrics({
  prefix: 'nodejs_',
  labels: { 
    service: process.env.SERVICE_NAME || 'my-app',
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development'
  }
});

// HTTP request metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'path', 'status'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10]
});

const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'path', 'status']
});

const httpActiveRequests = new client.Gauge({
  name: 'http_active_requests',
  help: 'Number of active HTTP requests',
  labelNames: ['method', 'path']
});

// Business metrics
const ordersTotal = new client.Counter({
  name: 'orders_total',
  help: 'Total number of orders',
  labelNames: ['status', 'payment_method']
});

const orderValue = new client.Histogram({
  name: 'order_value_dollars',
  help: 'Order value in dollars',
  labelNames: ['product_category'],
  buckets: [1, 5, 10, 50, 100, 500, 1000]
});

const activeUsers = new client.Gauge({
  name: 'active_users',
  help: 'Number of currently active users'
});

// Database metrics
const dbQueryDuration = new client.Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1]
});

const dbConnectionPoolSize = new client.Gauge({
  name: 'db_pool_connections_total',
  help: 'Total connections in pool'
});

const dbConnectionsIdle = new client.Gauge({
  name: 'db_pool_connections_idle',
  help: 'Idle connections in pool'
});

// Cache metrics
const cacheHits = new client.Counter({
  name: 'cache_hits_total',
  help: 'Number of cache hits',
  labelNames: ['cache_type']
});

const cacheMisses = new client.Counter({
  name: 'cache_misses_total',
  help: 'Number of cache misses',
  labelNames: ['cache_type']
});

module.exports = {
  registry: client.register,
  httpRequestDuration,
  httpRequestTotal,
  httpActiveRequests,
  ordersTotal,
  orderValue,
  activeUsers,
  dbQueryDuration,
  dbConnectionPoolSize,
  dbConnectionsIdle,
  cacheHits,
  cacheMisses
};
```

```javascript
// src/middleware/metrics.js
const { 
  httpRequestDuration, 
  httpRequestTotal,
  httpActiveRequests
} = require('../metrics');

function metricsMiddleware(req, res, next) {
  const startTime = Date.now();
  const method = req.method;
  const path = req.route?.path || req.path;
  
  // Track active requests
  httpActiveRequests.labels(method, path).inc();
  
  res.on('finish', () => {
    const duration = (Date.now() - startTime) / 1000;
    const status = res.statusCode.toString();
    
    // Record duration
    httpRequestDuration.labels(method, path, status).observe(duration);
    
    // Increment counter
    httpRequestTotal.labels(method, path, status).inc();
    
    // Decrease active requests
    httpActiveRequests.labels(method, path).dec();
  });
  
  next();
}

module.exports = metricsMiddleware;
```

```javascript
// src/app.js
const express = require('express');
const { registry } = require('./metrics');
const metricsMiddleware = require('./middleware/metrics');

const app = express();

// Metrics middleware
app.use(metricsMiddleware);

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', registry.contentType);
  res.send(await registry.metrics());
});

// Business logic routes
app.post('/orders', async (req, res) => {
  const { ordersTotal, orderValue } = require('./metrics');
  
  try {
    const order = await createOrder(req.body);
    
    // Track business metrics
    ordersTotal.labels('completed', order.paymentMethod).inc();
    orderValue.labels(order.category).observe(order.totalValue);
    
    res.json(order);
  } catch (error) {
    ordersTotal.labels('failed', req.body.paymentMethod || 'unknown').inc();
    throw error;
  }
});
```

### Python Metrics ด้วย prometheus-client

```python
# metrics.py
from prometheus_client import (
    Counter, Histogram, Gauge, Summary,
    CollectorRegistry, generate_latest, CONTENT_TYPE_LATEST
)
import time
import functools
from typing import Callable, Any

# สร้าง metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'path', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'path'],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    'http_active_requests',
    'Active HTTP requests',
    ['method', 'path']
)

DB_QUERY_LATENCY = Histogram(
    'db_query_duration_seconds',
    'Database query duration',
    ['operation', 'table']
)

CACHE_HITS = Counter(
    'cache_hits_total',
    'Cache hits',
    ['cache_name']
)

CACHE_MISSES = Counter(
    'cache_misses_total',
    'Cache misses',
    ['cache_name']
)

BUSINESS_EVENTS = Counter(
    'business_events_total',
    'Business events',
    ['event_type', 'status']
)


def track_request(method: str, path: str):
    """Decorator สำหรับ track HTTP requests"""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            ACTIVE_REQUESTS.labels(method=method, path=path).inc()
            start_time = time.time()
            status = 'success'
            
            try:
                result = await func(*args, **kwargs)
                return result
            except Exception as e:
                status = 'error'
                raise
            finally:
                duration = time.time() - start_time
                REQUEST_LATENCY.labels(method=method, path=path).observe(duration)
                ACTIVE_REQUESTS.labels(method=method, path=path).dec()
        
        return wrapper
    return decorator


def track_db_query(operation: str, table: str):
    """Decorator สำหรับ track database queries"""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            start_time = time.time()
            try:
                result = await func(*args, **kwargs)
                return result
            finally:
                duration = time.time() - start_time
                DB_QUERY_LATENCY.labels(operation=operation, table=table).observe(duration)
        
        return wrapper
    return decorator
```

```python
# app.py (FastAPI example)
from fastapi import FastAPI, Response
from prometheus_client import generate_latest, CONTENT_TYPE_LATEST
from metrics import REQUEST_COUNT, REQUEST_LATENCY, BUSINESS_EVENTS, track_request
import time

app = FastAPI()

@app.middleware("http")
async def metrics_middleware(request, call_next):
    method = request.method
    path = request.url.path
    start_time = time.time()
    
    response = await call_next(request)
    
    duration = time.time() - start_time
    status = str(response.status_code)
    
    REQUEST_COUNT.labels(method=method, path=path, status=status).inc()
    REQUEST_LATENCY.labels(method=method, path=path).observe(duration)
    
    return response

@app.get("/metrics")
async def metrics():
    return Response(
        generate_latest(),
        media_type=CONTENT_TYPE_LATEST
    )

@app.post("/orders")
@track_request("POST", "/orders")
async def create_order(order_data: dict):
    try:
        order = await process_order(order_data)
        BUSINESS_EVENTS.labels(event_type="order_created", status="success").inc()
        return order
    except Exception as e:
        BUSINESS_EVENTS.labels(event_type="order_created", status="failure").inc()
        raise
```

---

## Health Check Endpoints

### Node.js Health Check

```javascript
// src/health.js
const express = require('express');
const router = express.Router();

// ตรวจสอบ dependencies
async function checkDatabase() {
  try {
    const pool = require('./db/pool');
    await pool.query('SELECT 1');
    return { status: 'healthy', latency: null };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}

async function checkRedis() {
  try {
    const redis = require('./cache/redis');
    const start = Date.now();
    await redis.ping();
    return { status: 'healthy', latency: Date.now() - start };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}

async function checkExternalApi() {
  try {
    const response = await fetch('https://api.external-service.com/health', {
      timeout: 5000
    });
    return { 
      status: response.ok ? 'healthy' : 'degraded',
      httpStatus: response.status
    };
  } catch (error) {
    return { status: 'unhealthy', error: error.message };
  }
}

// Liveness probe: ตรวจสอบว่า app ยังทำงานอยู่
router.get('/health/live', (req, res) => {
  res.status(200).json({ 
    status: 'alive',
    timestamp: new Date().toISOString()
  });
});

// Readiness probe: ตรวจสอบว่าพร้อมรับ traffic
router.get('/health/ready', async (req, res) => {
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis()
  ]);
  
  const results = {
    database: checks[0].status === 'fulfilled' ? checks[0].value : { status: 'error' },
    redis: checks[1].status === 'fulfilled' ? checks[1].value : { status: 'error' }
  };
  
  const isReady = Object.values(results).every(r => r.status === 'healthy');
  
  res.status(isReady ? 200 : 503).json({
    status: isReady ? 'ready' : 'not ready',
    timestamp: new Date().toISOString(),
    checks: results
  });
});

// Deep health check: ตรวจสอบทุก dependencies
router.get('/health', async (req, res) => {
  const startTime = Date.now();
  
  const [dbCheck, redisCheck, apiCheck] = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkExternalApi()
  ]);
  
  const checks = {
    database: dbCheck.status === 'fulfilled' ? dbCheck.value : { status: 'error' },
    redis: redisCheck.status === 'fulfilled' ? redisCheck.value : { status: 'error' },
    externalApi: apiCheck.status === 'fulfilled' ? apiCheck.value : { status: 'error' }
  };
  
  const allHealthy = Object.values(checks).every(c => c.status === 'healthy');
  const anyUnhealthy = Object.values(checks).some(c => c.status === 'unhealthy');
  
  const overallStatus = anyUnhealthy ? 'unhealthy' : allHealthy ? 'healthy' : 'degraded';
  const httpStatus = anyUnhealthy ? 503 : 200;
  
  res.status(httpStatus).json({
    status: overallStatus,
    version: process.env.APP_VERSION || '1.0.0',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    responseTime: Date.now() - startTime,
    checks
  });
});

module.exports = router;
```

### Kubernetes Probes

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          ports:
            - containerPort: 3000
          
          # Startup probe: ให้ application เริ่มต้นได้
          startupProbe:
            httpGet:
              path: /health/live
              port: 3000
            failureThreshold: 30   # รอได้ถึง 30 * 10s = 5 นาที
            periodSeconds: 10
          
          # Liveness probe: ถ้า fail จะ restart pod
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
            successThreshold: 1
          
          # Readiness probe: ถ้า fail จะเอา pod ออกจาก load balancer
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
            successThreshold: 1
```

---

## SLI, SLO, and SLA Concepts

### นิยาม

```
SLI (Service Level Indicator)
  → Metric ที่ใช้วัด service performance
  → ตัวอย่าง: error rate, latency, throughput, availability

SLO (Service Level Objective)
  → เป้าหมายสำหรับ SLI
  → ตัวอย่าง: error rate < 0.1%, p99 latency < 500ms, availability >= 99.9%

SLA (Service Level Agreement)
  → ข้อตกลงทางธุรกิจกับลูกค้า
  → มักจะต่ำกว่า SLO (เผื่อ buffer)
  → ตัวอย่าง: "เราการันตี 99.5% uptime, ถ้าไม่ได้จะคืนเงิน"
```

### Error Budget

```
Error Budget = 1 - SLO

ตัวอย่าง:
SLO = 99.9% availability
Error Budget = 0.1% downtime per month
= 0.001 × 30 × 24 × 60 minutes
= 43.2 minutes downtime allowed per month
```

### ตัวอย่าง SLI และ SLO

```yaml
# slo-config.yaml
slos:
  # Availability SLO
  - name: "API Availability"
    sli:
      metric: "http_requests_total"
      good_events: "http_requests_total{status!~'5..'}"
      total_events: "http_requests_total"
    objectives:
      - target: 0.999     # 99.9%
        window: 30d       # ในช่วง 30 วัน

  # Latency SLO
  - name: "API Latency"
    sli:
      metric: "http_request_duration_seconds"
      good_events: "http_request_duration_seconds_bucket{le='0.5'}"  # < 500ms
      total_events: "http_request_duration_seconds_count"
    objectives:
      - target: 0.95      # 95% ของ requests ต้องเร็วกว่า 500ms
        window: 30d

  # Error Rate SLO
  - name: "API Error Rate"
    sli:
      type: "error_rate"
      bad_events: "http_requests_total{status=~'5..'}"
      total_events: "http_requests_total"
    objectives:
      - target: 0.999     # error rate < 0.1%
        window: 7d
```

### PromQL สำหรับ SLO Tracking

```promql
# Availability SLI (30 วัน)
sum(rate(http_requests_total{status!~"5.."}[30d]))
  /
sum(rate(http_requests_total[30d]))

# Error Budget remaining (percentage)
(
  1 - (
    sum(rate(http_requests_total{status=~"5.."}[30d]))
      /
    sum(rate(http_requests_total[30d]))
  )
) / (1 - 0.999) * 100

# Error Budget burn rate
(
  rate(http_requests_total{status=~"5.."}[1h]) /
  rate(http_requests_total[1h])
) / (1 - 0.999)

# Alert เมื่อ burn rate สูง (จะหมด budget เร็ว)
# ถ้า burn rate > 14.4x จะหมด budget ใน 2 ชั่วโมง
# ถ้า burn rate > 1 จะหมด budget ก่อน 30 วัน
```

---

## Uptime Monitoring

### Blackbox Exporter (Prometheus)

```yaml
# blackbox/config.yml
modules:
  # HTTP Check
  http_2xx:
    prober: http
    timeout: 10s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200, 201, 204]
      method: GET
      follow_redirects: true
      fail_if_not_ssl: false
      tls_config:
        insecure_skip_verify: false

  # HTTPS Check with content validation
  https_content_check:
    prober: http
    timeout: 10s
    http:
      valid_status_codes: [200]
      fail_if_not_ssl: true
      fail_if_body_not_matches_regexp:
        - "healthy"

  # TCP Check
  tcp_connect:
    prober: tcp
    timeout: 5s

  # DNS Check
  dns_check:
    prober: dns
    timeout: 5s
    dns:
      query_name: "example.com"
      query_type: "A"
```

```yaml
# prometheus.yml scrape config สำหรับ blackbox
- job_name: 'blackbox-http'
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://example.com
        - https://api.example.com/health
        - https://app.example.com/login
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115

- job_name: 'blackbox-tcp'
  metrics_path: /probe
  params:
    module: [tcp_connect]
  static_configs:
    - targets:
        - postgres:5432
        - redis:6379
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

### UptimeRobot Alternative (Open Source)

```yaml
# docker-compose.yaml - Uptime Kuma (self-hosted uptime monitoring)
version: '3.8'

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - uptime_kuma_data:/app/data
    environment:
      - UPTIME_KUMA_PORT=3001

volumes:
  uptime_kuma_data:
```

---

## Monitoring ใน CI/CD Pipeline

### ตรวจสอบ Metrics หลัง Deploy

```yaml
# .github/workflows/deploy-with-monitoring.yaml
name: Deploy with Monitoring

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy Application
        run: |
          helm upgrade --install my-app ./charts/my-app \
            --set image.tag=${{ github.sha }} \
            --wait --timeout 5m
      
      - name: Wait for deployment to stabilize
        run: sleep 120  # รอ 2 นาทีก่อนตรวจสอบ
      
      - name: Check error rate post-deploy
        run: |
          PROMETHEUS_URL="https://prometheus.example.com"
          
          # ดึง error rate ใน 5 นาทีที่ผ่านมา
          ERROR_RATE=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
            --data-urlencode 'query=rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100' \
            | jq '.data.result[0].value[1]' -r)
          
          echo "Current error rate: ${ERROR_RATE}%"
          
          # ถ้า error rate สูงกว่า 5% ให้ rollback
          if (( $(echo "$ERROR_RATE > 5" | bc -l) )); then
            echo "Error rate too high! Rolling back..."
            helm rollback my-app
            exit 1
          fi
          
          echo "Error rate is acceptable"
      
      - name: Check P99 latency post-deploy
        run: |
          PROMETHEUS_URL="https://prometheus.example.com"
          
          P99_LATENCY=$(curl -s "$PROMETHEUS_URL/api/v1/query" \
            --data-urlencode 'query=histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))' \
            | jq '.data.result[0].value[1]' -r)
          
          echo "P99 Latency: ${P99_LATENCY}s"
          
          # ถ้า P99 latency สูงกว่า 2 วินาที
          if (( $(echo "$P99_LATENCY > 2" | bc -l) )); then
            echo "Latency too high! Rolling back..."
            helm rollback my-app
            exit 1
          fi
      
      - name: Send deployment notification
        if: success()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          channel-id: 'deployments'
          slack-message: |
            ✅ Deployment successful!
            Version: ${{ github.sha }}
            Error rate: Normal
            Latency: Normal
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Canary Deployment กับ Metrics

```yaml
# .github/workflows/canary-deploy.yaml
name: Canary Deployment

jobs:
  canary:
    runs-on: ubuntu-latest
    steps:
      # Deploy 10% traffic ไปยัง new version
      - name: Deploy Canary (10% traffic)
        run: |
          helm upgrade --install my-app-canary ./charts/my-app \
            --set image.tag=${{ github.sha }} \
            --set replicaCount=1
          
          # Update ingress traffic split
          kubectl patch ingress my-app --type='json' -p='[
            {"op":"replace","path":"/metadata/annotations/nginx.ingress.kubernetes.io~1canary-weight","value":"10"}
          ]'
      
      # รอ 10 นาทีแล้วตรวจสอบ metrics
      - name: Monitor canary metrics (10 min)
        run: |
          sleep 600
          
          # Compare error rates
          STABLE_ERROR=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode 'query=rate(http_requests_total{version="stable",status=~"5.."}[5m]) / rate(http_requests_total{version="stable"}[5m]) * 100' \
            | jq -r '.data.result[0].value[1]')
          
          CANARY_ERROR=$(curl -s "http://prometheus:9090/api/v1/query" \
            --data-urlencode 'query=rate(http_requests_total{version="canary",status=~"5.."}[5m]) / rate(http_requests_total{version="canary"}[5m]) * 100' \
            | jq -r '.data.result[0].value[1]')
          
          echo "Stable error rate: $STABLE_ERROR%"
          echo "Canary error rate: $CANARY_ERROR%"
          
          # ถ้า canary error rate สูงกว่า stable เกิน 2x ให้ rollback
          THRESHOLD=$(echo "$STABLE_ERROR * 2" | bc -l)
          
          if (( $(echo "$CANARY_ERROR > $THRESHOLD" | bc -l) )); then
            echo "Canary has too many errors, rolling back"
            kubectl patch ingress my-app --type='json' -p='[
              {"op":"replace","path":"/metadata/annotations/nginx.ingress.kubernetes.io~1canary-weight","value":"0"}
            ]'
            helm uninstall my-app-canary
            exit 1
          fi
      
      # Promote canary ถ้าผ่าน
      - name: Promote canary to 100%
        run: |
          helm upgrade my-app ./charts/my-app \
            --set image.tag=${{ github.sha }}
          
          helm uninstall my-app-canary
          
          kubectl patch ingress my-app --type='json' -p='[
            {"op":"remove","path":"/metadata/annotations/nginx.ingress.kubernetes.io~1canary-weight"}
          ]'
```

---

## Distributed Tracing

### OpenTelemetry Setup สำหรับ Node.js

```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-trace-otlp-http
```

```javascript
// tracing.js - ต้อง import ก่อน application code
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://tempo:4318/v1/traces',
    headers: {}
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      '@opentelemetry/instrumentation-express': { enabled: true },
      '@opentelemetry/instrumentation-http': { enabled: true },
      '@opentelemetry/instrumentation-pg': { enabled: true },
      '@opentelemetry/instrumentation-redis': { enabled: true },
      '@opentelemetry/instrumentation-mongoose': { enabled: true }
    })
  ]
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch((error) => console.log('Error terminating tracing', error))
    .finally(() => process.exit(0));
});
```

```javascript
// Custom spans
const { trace, context } = require('@opentelemetry/api');

const tracer = trace.getTracer('my-app');

async function processPayment(orderId, amount) {
  // สร้าง custom span
  return await tracer.startActiveSpan('processPayment', async (span) => {
    try {
      span.setAttribute('order.id', orderId);
      span.setAttribute('payment.amount', amount);
      span.setAttribute('payment.currency', 'THB');
      
      // Nested span
      const result = await tracer.startActiveSpan('validatePayment', async (childSpan) => {
        try {
          const validation = await validatePaymentGateway(amount);
          childSpan.setAttribute('validation.result', validation.status);
          return validation;
        } catch (error) {
          childSpan.recordException(error);
          childSpan.setStatus({ code: SpanStatusCode.ERROR });
          throw error;
        } finally {
          childSpan.end();
        }
      });
      
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.recordException(error);
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### Grafana Tempo สำหรับ Tracing

```yaml
# docker-compose.yaml - เพิ่ม Tempo
services:
  tempo:
    image: grafana/tempo:latest
    container_name: tempo
    ports:
      - "3200:3200"   # Tempo HTTP
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
    volumes:
      - ./tempo/tempo.yaml:/etc/tempo.yaml:ro
      - tempo_data:/tmp/tempo
    command: ["-config.file=/etc/tempo.yaml"]
    networks:
      - monitoring
```

```yaml
# tempo/tempo.yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318

storage:
  trace:
    backend: local
    local:
      path: /tmp/tempo/blocks

compactor:
  compaction:
    block_retention: 1h

metrics_generator:
  registry:
    external_labels:
      source: tempo
  storage:
    path: /tmp/tempo/generator/wal
    remote_write:
      - url: http://prometheus:9090/api/v1/write
        send_exemplars: true
```

---

## Log Management

### Loki + Grafana Stack

```yaml
# docker-compose.yaml - เพิ่ม Loki และ Promtail
services:
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yaml
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail/promtail-config.yaml:/etc/promtail/promtail-config.yaml:ro
    command: -config.file=/etc/promtail/promtail-config.yaml
    networks:
      - monitoring
```

```yaml
# promtail/promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Scrape Docker container logs
  - job_name: docker-containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
    pipeline_stages:
      - json:
          expressions:
            level: level
            msg: message
            timestamp: timestamp
      - labels:
          level:
      - timestamp:
          source: timestamp
          format: RFC3339Nano

  # System logs
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

### Structured Logging ในแอปพลิเคชัน

```javascript
// src/logger.js
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()  // Structured JSON logs
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'my-app',
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development'
  },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

// เพิ่ม trace context ใน logs
function withTrace(req) {
  const { trace, context } = require('@opentelemetry/api');
  const span = trace.getActiveSpan();
  
  if (span) {
    const spanContext = span.spanContext();
    return logger.child({
      traceId: spanContext.traceId,
      spanId: spanContext.spanId,
      requestId: req.headers['x-request-id']
    });
  }
  
  return logger;
}

module.exports = { logger, withTrace };
```

```javascript
// ใช้ logger ใน application
const { withTrace } = require('./logger');

app.post('/orders', async (req, res) => {
  const log = withTrace(req);
  
  log.info('Order creation started', {
    userId: req.user.id,
    items: req.body.items.length
  });
  
  try {
    const order = await createOrder(req.body);
    
    log.info('Order created successfully', {
      orderId: order.id,
      totalAmount: order.totalAmount,
      userId: req.user.id
    });
    
    res.json(order);
  } catch (error) {
    log.error('Order creation failed', {
      error: error.message,
      stack: error.stack,
      userId: req.user.id,
      input: req.body
    });
    
    throw error;
  }
});
```

---

## แบบฝึกหัด

### Exercise 1: ติดตั้ง Monitoring Stack

```bash
# 1. สร้าง docker-compose.yaml พร้อม Prometheus, Grafana, Alertmanager
# (ใช้ config จาก section "Prometheus Setup")

# 2. Start stack
docker compose up -d

# 3. ตรวจสอบ
docker compose ps
docker compose logs prometheus

# 4. เปิด Prometheus: http://localhost:9090
# ลอง query: up

# 5. เปิด Grafana: http://localhost:3000
# Login: admin/secure-password
# เพิ่ม Prometheus datasource

# 6. Import dashboard ยอดนิยม
# Grafana Dashboard IDs:
# - 1860: Node Exporter Full
# - 3662: Prometheus 2.0 Overview
# - 13978: Docker Containers
```

### Exercise 2: Instrument Node.js Application

1. สร้าง Express application พร้อม metrics:

```javascript
// app.js
const express = require('express');
const promClient = require('prom-client');

const app = express();
promClient.collectDefaultMetrics();

const httpReqs = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status']
});

const httpLatency = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'path'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
});

app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpReqs.labels(req.method, req.path, res.statusCode.toString()).inc();
    httpLatency.labels(req.method, req.path).observe(duration);
  });
  next();
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.send(await promClient.register.metrics());
});

app.get('/health/live', (req, res) => res.json({ status: 'alive' }));
app.get('/health/ready', (req, res) => res.json({ status: 'ready' }));

app.get('/', (req, res) => res.json({ message: 'Hello World' }));

app.listen(3000, () => console.log('Server on port 3000'));
```

2. เพิ่ม scrape config ใน prometheus.yml
3. สร้าง Grafana dashboard
4. ตั้งค่า alert เมื่อ error rate สูง

### Exercise 3: ตั้งค่า Alerting

สร้าง alerting rules และ Alertmanager config:

```yaml
# prometheus/rules/my-app.yml
groups:
  - name: my-app
    rules:
      - alert: AppDown
        expr: up{job="my-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
      
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])
            /
          rate(http_requests_total[5m]) * 100 > 5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "High error rate: {{ $value }}%"
```

```yaml
# alertmanager/config.yml
route:
  receiver: slack
  group_wait: 30s

receivers:
  - name: slack
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        channel: '#alerts'
        title: '{{ .CommonAnnotations.summary }}'
```

### Exercise 4: SLO Dashboard ใน Grafana

สร้าง Grafana dashboard ที่แสดง:
1. Current availability (uptime %)
2. Error budget remaining
3. Error budget burn rate
4. Request rate, error rate, latency (RED metrics)

ใช้ queries จาก [section SLI/SLO](#sli-slo-and-sla-concepts)

---

## สรุป

Monitoring & Observability เป็นสิ่งจำเป็นสำหรับ production systems:

1. **Three Pillars**: Logs, Metrics, Traces ต้องมีทั้งสาม
2. **Prometheus**: Time-series database สำหรับ metrics
3. **Grafana**: Visualization platform ที่ทรงพลัง
4. **Alertmanager**: จัดการ alerts อย่างชาญฉลาด
5. **Instrumentation**: Code ต้อง expose metrics ที่มีความหมาย
6. **Health Checks**: Liveness, Readiness, Startup probes
7. **SLI/SLO/SLA**: วัดและ track service reliability
8. **Uptime Monitoring**: ตรวจสอบจากภายนอก
9. **CI/CD Integration**: ตรวจสอบ metrics หลัง deploy อัตโนมัติ
10. **Distributed Tracing**: เข้าใจ request flow ในระบบ microservices

**ยินดีด้วย!** คุณได้เรียนรู้ครบ Part 21-30 ของหลักสูตร CI/CD แล้ว ซึ่งครอบคลุม:
- Part 26: Helm Charts สำหรับ Kubernetes
- Part 27: AWS CodePipeline
- Part 28: Google Cloud Build
- Part 29: Azure DevOps
- Part 30: Monitoring & Observability

ในบทต่อๆ ไปจะครอบคลุม advanced topics เช่น security scanning, compliance, GitOps workflows และอื่นๆ

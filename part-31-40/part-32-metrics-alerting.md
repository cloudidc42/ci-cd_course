# Part 32: Metrics & Alerting ด้วย Prometheus และ Grafana

## บทนำ

Metrics เป็นตัวชี้วัดสุขภาพของระบบที่วัดค่าได้เป็นตัวเลข เช่น จำนวน requests, response time, error rate, CPU usage การมี metrics ที่ดีช่วยให้ทีมสามารถตรวจจับปัญหาได้ก่อนที่ users จะได้รับผลกระทบ ในบทนี้เราจะเรียนรู้การตั้งค่า Prometheus ecosystem อย่างครบวงจร

## วัตถุประสงค์การเรียนรู้

- เข้าใจ Prometheus metric types และ data model
- เขียน PromQL queries สำหรับ monitoring
- ตั้งค่า Grafana dashboards
- สร้าง alert rules ที่มีประสิทธิภาพ
- Integrate กับ PagerDuty และ Slack
- ออกแบบ SLO-based alerting

---

## 32.1 Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Prometheus Ecosystem                      │
│                                                             │
│  Applications  ──(metrics)──►  Prometheus  ──►  Grafana    │
│  (with /metrics endpoint)         │                         │
│                                   │                         │
│  Pushgateway ◄─(push)── Batch    ▼                         │
│                            Alertmanager ──► Slack/PD/Email  │
│                                                             │
│  Service Discovery:                                         │
│  Kubernetes, Consul, EC2, File-based                       │
└─────────────────────────────────────────────────────────────┘
```

### Prometheus Metric Types

| Type | คำอธิบาย | ตัวอย่าง |
|------|---------|---------|
| **Counter** | ค่าที่เพิ่มขึ้นเท่านั้น | จำนวน requests, errors |
| **Gauge** | ค่าที่ขึ้นลงได้ | CPU usage, memory, active connections |
| **Histogram** | กระจาย observations เป็น buckets | request duration, response size |
| **Summary** | คล้าย Histogram แต่คำนวณ quantiles ใน client | latency percentiles |

---

## 32.2 Instrument Applications

### Python - ใช้ prometheus_client

```python
# requirements.txt
# prometheus-client==0.19.0
# flask==3.0.0

from flask import Flask, request, jsonify
from prometheus_client import (
    Counter, Histogram, Gauge, Summary, Info,
    generate_latest, CONTENT_TYPE_LATEST,
    multiprocess, CollectorRegistry
)
import time
import os

app = Flask(__name__)

# === Metrics Definition ===

# Counter: นับ requests
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

# Histogram: วัด request duration
http_request_duration_seconds = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

# Gauge: วัด active connections
http_active_connections = Gauge(
    'http_active_connections',
    'Number of active HTTP connections'
)

# Gauge: Application info
app_info = Info(
    'app',
    'Application information'
)
app_info.info({
    'version': os.getenv('APP_VERSION', '0.0.0'),
    'environment': os.getenv('APP_ENV', 'development'),
    'service': os.getenv('SERVICE_NAME', 'unknown')
})

# Counter: Business metrics
orders_total = Counter(
    'orders_total',
    'Total orders processed',
    ['status', 'payment_method']
)

orders_amount = Histogram(
    'orders_amount_baht',
    'Order amount in Thai Baht',
    buckets=[100, 500, 1000, 5000, 10000, 50000]
)

# Gauge: Queue depth
queue_depth = Gauge(
    'order_queue_depth',
    'Number of orders in processing queue',
    ['queue_name']
)

# === Middleware ===

@app.before_request
def before_request():
    request.start_time = time.time()
    http_active_connections.inc()

@app.after_request
def after_request(response):
    duration = time.time() - request.start_time
    http_active_connections.dec()
    
    # Record metrics
    http_requests_total.labels(
        method=request.method,
        endpoint=request.endpoint or 'unknown',
        status_code=response.status_code
    ).inc()
    
    http_request_duration_seconds.labels(
        method=request.method,
        endpoint=request.endpoint or 'unknown'
    ).observe(duration)
    
    return response

# === Metrics Endpoint ===

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}

# === Business Logic ===

@app.route('/orders', methods=['POST'])
def create_order():
    data = request.json
    
    # Business logic
    order = process_order(data)
    
    # Record business metrics
    orders_total.labels(
        status='success',
        payment_method=data.get('payment_method', 'unknown')
    ).inc()
    
    orders_amount.observe(float(order.get('total', 0)))
    
    return jsonify(order), 201

def process_order(data):
    # Simulate processing
    time.sleep(0.1)
    return {'id': 'ORD-001', 'total': 1500.00, 'status': 'created'}

# === Health Check ===

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'timestamp': time.time()})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### Node.js - ใช้ prom-client

```javascript
// metrics.js
const client = require('prom-client');
const express = require('express');

// ตั้งค่า default metrics
const register = new client.Registry();
client.collectDefaultMetrics({
  register,
  labels: {
    service: process.env.SERVICE_NAME || 'unknown',
    version: process.env.APP_VERSION || '0.0.0',
    environment: process.env.NODE_ENV || 'development'
  }
});

// === Custom Metrics ===

// HTTP Request Counter
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register]
});

// HTTP Request Duration Histogram
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [register]
});

// Database Query Duration
const dbQueryDuration = new client.Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duration of database queries',
  labelNames: ['query_type', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register]
});

// Active Connections Gauge
const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  labelNames: ['type'],
  registers: [register]
});

// Business Metrics
const ordersProcessed = new client.Counter({
  name: 'orders_processed_total',
  help: 'Total orders processed',
  labelNames: ['status', 'payment_method'],
  registers: [register]
});

const revenueTotal = new client.Counter({
  name: 'revenue_total_baht',
  help: 'Total revenue in Thai Baht',
  labelNames: ['payment_method'],
  registers: [register]
});

// === Middleware ===
const metricsMiddleware = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route ? req.route.path : req.path;
    
    httpRequestsTotal.inc({
      method: req.method,
      route: route,
      status_code: res.statusCode
    });
    
    httpRequestDuration.observe(
      { method: req.method, route: route },
      duration
    );
  });
  
  next();
};

// === DB Query Wrapper ===
async function timedQuery(queryType, table, queryFn) {
  const end = dbQueryDuration.startTimer({ query_type: queryType, table });
  try {
    const result = await queryFn();
    end();
    return result;
  } catch (error) {
    end();
    throw error;
  }
}

// === Setup Express ===
function setupMetrics(app) {
  app.use(metricsMiddleware);
  
  app.get('/metrics', async (req, res) => {
    res.set('Content-Type', register.contentType);
    res.send(await register.metrics());
  });
}

module.exports = {
  register,
  httpRequestsTotal,
  httpRequestDuration,
  dbQueryDuration,
  activeConnections,
  ordersProcessed,
  revenueTotal,
  metricsMiddleware,
  timedQuery,
  setupMetrics
};
```

### Go - ใช้ prometheus/client_golang

```go
package metrics

import (
    "net/http"
    "strconv"
    "time"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    // HTTP Metrics
    HTTPRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status_code"},
    )

    HTTPRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )

    // Database Metrics
    DBQueryDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "db_query_duration_seconds",
            Help:    "Database query duration in seconds",
            Buckets: []float64{.001, .005, .01, .05, .1, .5, 1, 5},
        },
        []string{"query_type", "table", "status"},
    )

    DBConnectionPool = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "db_connection_pool",
            Help: "Database connection pool stats",
        },
        []string{"state"},
    )

    // Business Metrics
    OrdersTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "orders_total",
            Help: "Total orders",
        },
        []string{"status", "payment_method"},
    )

    OrderAmount = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "order_amount_baht",
            Help:    "Order amount in Thai Baht",
            Buckets: []float64{100, 500, 1000, 5000, 10000, 50000, 100000},
        },
        []string{"payment_method"},
    )
)

// HTTPMetricsMiddleware วัด HTTP metrics
func HTTPMetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)
        
        duration := time.Since(start).Seconds()
        statusCode := strconv.Itoa(wrapped.statusCode)
        
        HTTPRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            statusCode,
        ).Inc()
        
        HTTPRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

// MetricsHandler return prometheus handler
func MetricsHandler() http.Handler {
    return promhttp.Handler()
}

// RecordDBQuery บันทึก DB query metrics
func RecordDBQuery(queryType, table string, fn func() error) error {
    start := time.Now()
    err := fn()
    duration := time.Since(start).Seconds()
    
    status := "success"
    if err != nil {
        status = "error"
    }
    
    DBQueryDuration.WithLabelValues(queryType, table, status).Observe(duration)
    return err
}
```

---

## 32.3 Prometheus Configuration

### prometheus.yml

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    region: 'ap-southeast-1'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load rule files
rule_files:
  - "rules/*.yml"

# Scrape configurations
scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  
  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets:
          - 'node-exporter:9100'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+)(?::\d+)?'
        replacement: '${1}'
  
  # Kubernetes pods (auto-discovery)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # เก็บเฉพาะ pods ที่มี annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # ใช้ port จาก annotation
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      
      # เพิ่ม labels จาก Kubernetes
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod
  
  # Kubernetes services
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
  
  # Application services
  - job_name: 'order-service'
    metrics_path: '/metrics'
    static_configs:
      - targets:
          - 'order-service:8080'
        labels:
          service: 'order-service'
          environment: 'production'
    
  - job_name: 'payment-service'
    metrics_path: '/metrics'
    static_configs:
      - targets:
          - 'payment-service:8080'
        labels:
          service: 'payment-service'
```

---

## 32.4 PromQL Queries

### พื้นฐาน PromQL

```promql
# === Counter Queries ===

# Rate ของ requests ต่อวินาที (ช่วง 5 นาที)
rate(http_requests_total[5m])

# Total requests per endpoint
sum by (endpoint) (rate(http_requests_total[5m]))

# Error rate (4xx + 5xx)
sum(rate(http_requests_total{status_code=~"[45].."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# === Histogram Queries ===

# P50, P95, P99 latency
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Average request duration
rate(http_request_duration_seconds_sum[5m]) /
rate(http_request_duration_seconds_count[5m])

# === Gauge Queries ===

# Current memory usage percentage
(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 
node_memory_MemTotal_bytes * 100

# CPU usage percentage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Disk usage
(1 - node_filesystem_free_bytes / node_filesystem_size_bytes) * 100

# === Business Metrics ===

# Orders per minute
rate(orders_total[1m]) * 60

# Success rate
sum(rate(orders_total{status="success"}[5m])) /
sum(rate(orders_total[5m])) * 100

# Revenue per hour
sum(increase(revenue_total_baht[1h]))

# Average order value
rate(order_amount_baht_sum[5m]) / rate(order_amount_baht_count[5m])
```

### Advanced PromQL

```promql
# === SLO Calculations ===

# Error budget remaining (99.9% SLO)
1 - (
  sum(rate(http_requests_total{status_code=~"5.."}[30d])) /
  sum(rate(http_requests_total[30d]))
) / (1 - 0.999)

# Availability SLO
avg_over_time(
  (
    1 - (
      sum(rate(http_requests_total{status_code=~"5.."}[5m])) /
      sum(rate(http_requests_total[5m]))
    )
  )[30d:5m]
)

# === Anomaly Detection ===

# Unusual traffic spike (> 3x average)
rate(http_requests_total[5m]) > 
3 * avg_over_time(rate(http_requests_total[5m])[1h:5m])

# === Multi-dimension Analysis ===

# Top 5 slowest endpoints
topk(5, 
  histogram_quantile(0.95, 
    sum by (endpoint, le) (
      rate(http_request_duration_seconds_bucket[5m])
    )
  )
)

# Error rate by service and endpoint
sum by (service, endpoint) (
  rate(http_requests_total{status_code=~"5.."}[5m])
) /
sum by (service, endpoint) (
  rate(http_requests_total[5m])
)

# === Recording Rules (สำหรับ performance) ===
# บันทึกผลลัพธ์ที่คำนวณบ่อยๆ

# rules/recording.yml
groups:
  - name: http_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
      
      - record: job:http_error_rate:rate5m
        expr: |
          sum by (job) (rate(http_requests_total{status_code=~"5.."}[5m])) /
          sum by (job) (rate(http_requests_total[5m]))
      
      - record: job:http_request_duration_p95:rate5m
        expr: |
          histogram_quantile(0.95, 
            sum by (job, le) (rate(http_request_duration_seconds_bucket[5m]))
          )
```

---

## 32.5 Alert Rules

### Prometheus Alert Rules

```yaml
# rules/alerts.yml
groups:
  - name: availability
    rules:
      # Service Down
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} has been down for more than 1 minute"
          runbook: "https://wiki.example.com/runbooks/service-down"
          
      # High Error Rate
      - alert: HighErrorRate
        expr: |
          (
            sum by (service) (rate(http_requests_total{status_code=~"5.."}[5m])) /
            sum by (service) (rate(http_requests_total[5m]))
          ) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 1%)"
          
      - alert: CriticalErrorRate
        expr: |
          (
            sum by (service) (rate(http_requests_total{status_code=~"5.."}[5m])) /
            sum by (service) (rate(http_requests_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

  - name: latency
    rules:
      # High Latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, 
            sum by (service, le) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High P95 latency on {{ $labels.service }}"
          description: "P95 latency is {{ $value | humanizeDuration }} (threshold: 500ms)"
          
      - alert: CriticalLatency
        expr: |
          histogram_quantile(0.99, 
            sum by (service, le) (
              rate(http_request_duration_seconds_bucket[5m])
            )
          ) > 2
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Critical P99 latency on {{ $labels.service }}"

  - name: resources
    rules:
      # High CPU Usage
      - alert: HighCPUUsage
        expr: |
          100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}%"
          
      # High Memory Usage
      - alert: HighMemoryUsage
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanize }}%"
          
      # Disk Space Low
      - alert: DiskSpaceLow
        expr: |
          (1 - node_filesystem_free_bytes{fstype!="tmpfs"} / 
          node_filesystem_size_bytes{fstype!="tmpfs"}) * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space low on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} is {{ $value | humanize }}% full"

  - name: slo
    rules:
      # SLO Error Budget Burning Fast
      - alert: SLOErrorBudgetBurningFast
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[1h])) /
            sum(rate(http_requests_total[1h]))
          ) > (1 - 0.999) * 14.4
        for: 2m
        labels:
          severity: critical
          category: slo
        annotations:
          summary: "SLO error budget burning too fast"
          description: "Error budget will be exhausted in less than 1 hour at current rate"
          
      - alert: SLOErrorBudgetLow
        expr: |
          (
            1 - (
              sum(rate(http_requests_total{status_code=~"5.."}[30d])) /
              sum(rate(http_requests_total[30d]))
            )
          ) < 0.2
        labels:
          severity: warning
          category: slo
        annotations:
          summary: "SLO error budget below 20%"
          description: "Error budget remaining: {{ $value | humanizePercentage }}"

  - name: business
    rules:
      # Order Processing Stopped
      - alert: OrderProcessingStopped
        expr: rate(orders_total[5m]) == 0
        for: 10m
        labels:
          severity: critical
          category: business
        annotations:
          summary: "Order processing has stopped"
          description: "No orders have been processed in the last 10 minutes"
          
      # Low Order Success Rate
      - alert: LowOrderSuccessRate
        expr: |
          sum(rate(orders_total{status="success"}[10m])) /
          sum(rate(orders_total[10m])) < 0.95
        for: 5m
        labels:
          severity: warning
          category: business
        annotations:
          summary: "Low order success rate"
          description: "Order success rate is {{ $value | humanizePercentage }}"
```

---

## 32.6 Alertmanager Configuration

### alertmanager.yml

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  
  # Slack configuration
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
  
  # PagerDuty
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# Templates สำหรับ notifications
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree
route:
  # Default receiver
  receiver: 'default'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10m
  repeat_interval: 12h
  
  routes:
    # Critical alerts ไปยัง PagerDuty
    - match:
        severity: critical
      receiver: pagerduty-critical
      continue: true  # ส่งไปยัง receivers อื่นด้วย
      
    # Critical alerts ไปยัง Slack #alerts-critical
    - match:
        severity: critical
      receiver: slack-critical
      
    # Warning alerts ไปยัง Slack #alerts-warning
    - match:
        severity: warning
      receiver: slack-warning
      group_wait: 30s
      
    # Business alerts ไปยัง team channel
    - match:
        category: business
      receiver: slack-business
      
    # SLO alerts ไปยัง SRE team
    - match:
        category: slo
      receiver: slack-sre
      group_interval: 1h

# Inhibition rules (ระงับ alerts บางอย่างเมื่อมี alerts อื่น)
inhibit_rules:
  # ระงับ warning เมื่อมี critical สำหรับ service เดียวกัน
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'service']
  
  # ระงับ alerts ทั้งหมดเมื่อ service down
  - source_match:
      alertname: 'ServiceDown'
    target_match_re:
      alertname: '(High.*|Low.*|Critical.*)'
    equal: ['service']

# Receivers
receivers:
  - name: 'default'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_ROUTING_KEY'
        description: '{{ template "pagerduty.default.description" . }}'
        severity: '{{ if eq .GroupLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'

  - name: 'slack-critical'
    slack_configs:
      - channel: '#alerts-critical'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '🚨 CRITICAL: {{ .GroupLabels.alertname }}'
        text: |
          *Alert:* {{ .GroupLabels.alertname }}
          *Severity:* {{ .GroupLabels.severity }}
          *Service:* {{ .GroupLabels.service | default "N/A" }}
          *Environment:* {{ .GroupLabels.cluster }}
          
          {{ range .Alerts }}
          *Summary:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook | default "N/A" }}
          *Started:* {{ .StartsAt | since }}
          {{ end }}
        actions:
          - type: button
            text: 'View in Grafana'
            url: '{{ .CommonAnnotations.grafana_url | default "http://grafana:3000" }}'
          - type: button
            text: 'View Runbook'
            url: '{{ .CommonAnnotations.runbook | default "#" }}'

  - name: 'slack-warning'
    slack_configs:
      - channel: '#alerts-warning'
        send_resolved: true
        color: 'warning'
        title: '⚠️ WARNING: {{ .GroupLabels.alertname }}'
        text: |
          *Service:* {{ .GroupLabels.service | default "N/A" }}
          {{ range .Alerts }}
          {{ .Annotations.description }}
          {{ end }}

  - name: 'slack-business'
    slack_configs:
      - channel: '#team-business-alerts'
        send_resolved: true
        title: '📊 Business Alert: {{ .GroupLabels.alertname }}'

  - name: 'slack-sre'
    slack_configs:
      - channel: '#sre-alerts'
        send_resolved: true
        title: '🎯 SLO Alert: {{ .GroupLabels.alertname }}'
```

### Custom Alert Templates

```
{{/* templates/custom.tmpl */}}

{{ define "slack.default.title" }}
{{ if .Status | eq "firing" }}🔥{{ else }}✅{{ end }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
{{ end }}

{{ define "slack.default.text" }}
{{ range .Alerts }}
*Summary:* {{ .Annotations.summary }}
*Description:* {{ .Annotations.description }}
*Labels:*
{{ range .Labels.SortedPairs }}  - {{ .Name }}: {{ .Value }}
{{ end }}
{{ end }}
{{ end }}

{{ define "pagerduty.default.description" }}
[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }} - {{ .CommonAnnotations.summary }}
{{ end }}
```

---

## 32.7 Grafana Dashboards

### Grafana Dashboard ด้วย JSON

```json
{
  "dashboard": {
    "id": null,
    "title": "Service Overview",
    "tags": ["service", "production"],
    "timezone": "Asia/Bangkok",
    "schemaVersion": 38,
    "version": 1,
    "refresh": "30s",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m]))",
            "legendFormat": "req/s"
          }
        ],
        "options": {
          "reduceOptions": {"calcs": ["lastNotNull"]},
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 1000},
              {"color": "red", "value": 5000}
            ]
          }
        }
      },
      {
        "id": 2,
        "title": "Error Rate",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m])) * 100",
            "legendFormat": "Error %"
          }
        ],
        "options": {
          "unit": "percent",
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 1},
              {"color": "red", "value": 5}
            ]
          }
        }
      },
      {
        "id": 3,
        "title": "P95 Latency",
        "type": "stat",
        "gridPos": {"h": 4, "w": 6, "x": 12, "y": 0},
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m]))) * 1000",
            "legendFormat": "P95 ms"
          }
        ],
        "options": {
          "unit": "ms",
          "thresholds": {
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 200},
              {"color": "red", "value": 500}
            ]
          }
        }
      },
      {
        "id": 4,
        "title": "Request Rate Over Time",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 4},
        "targets": [
          {
            "expr": "sum by (endpoint) (rate(http_requests_total[5m]))",
            "legendFormat": "{{endpoint}}"
          }
        ],
        "options": {
          "tooltip": {"mode": "multi"},
          "legend": {"displayMode": "table"}
        }
      }
    ]
  }
}
```

### Grafana Provisioning

```yaml
# grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo
          
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      derivedFields:
        - datasourceUid: tempo
          matcherRegex: "trace_id=(\\w+)"
          name: TraceID
          url: "$${__value.raw}"

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
```

---

## 32.8 SLO-Based Alerting

### SLO Definition และ Error Budget

```yaml
# slo-config.yml
slos:
  - name: api-availability
    description: "API should be available 99.9% of the time"
    
    # SLI (Service Level Indicator)
    sli:
      type: ratio
      good_events:
        selector: 'http_requests_total{status_code!~"5.."}'
      total_events:
        selector: 'http_requests_total'
    
    # SLO Target
    target: 0.999
    
    # Windows
    windows:
      - name: 30d
        duration: 30d
      - name: 7d
        duration: 7d
    
    # Error budget alerting
    alert:
      # Alert เมื่อ error budget burn rate สูงเกิน
      burn_rate_windows:
        - short: 1h
          long: 6h
          burn_rate: 14.4  # จะหมด budget ใน 2 วัน
          severity: critical
        - short: 6h
          long: 3d
          burn_rate: 1
          severity: warning
```

### PromQL สำหรับ SLO Monitoring

```promql
# Error Budget Calculation
# error_budget_remaining = 1 - (actual_error_rate / (1 - slo_target))

# ตัวอย่างสำหรับ 99.9% SLO
(
  1 - (
    sum(rate(http_requests_total{status_code=~"5.."}[30d])) /
    sum(rate(http_requests_total[30d]))
  )
) / (1 - 0.999)

# Error Budget Burn Rate
# burn_rate = current_error_rate / (1 - slo_target)
sum(rate(http_requests_total{status_code=~"5.."}[1h])) /
sum(rate(http_requests_total[1h])) /
(1 - 0.999)

# Multi-window multi-burn-rate alerting
# Alert เมื่อ burn rate สูงทั้งใน short และ long window
(
  sum(rate(http_requests_total{status_code=~"5.."}[1h])) /
  sum(rate(http_requests_total[1h])) 
) > (14.4 * (1 - 0.999))
AND
(
  sum(rate(http_requests_total{status_code=~"5.."}[6h])) /
  sum(rate(http_requests_total[6h]))
) > (14.4 * (1 - 0.999))
```

---

## 32.9 Infrastructure as Code สำหรับ Monitoring

### Terraform สำหรับ Grafana

```hcl
# main.tf
terraform {
  required_providers {
    grafana = {
      source  = "grafana/grafana"
      version = "~> 2.0"
    }
  }
}

provider "grafana" {
  url  = var.grafana_url
  auth = var.grafana_api_key
}

# Data Source
resource "grafana_data_source" "prometheus" {
  name = "Prometheus"
  type = "prometheus"
  url  = var.prometheus_url

  json_data_encoded = jsonencode({
    httpMethod = "POST"
  })
}

# Alert Contact Points
resource "grafana_contact_point" "slack" {
  name = "Slack"

  slack {
    url   = var.slack_webhook_url
    title = "{{ template \"slack.default.title\" . }}"
    text  = "{{ template \"slack.default.text\" . }}"
  }
}

resource "grafana_contact_point" "pagerduty" {
  name = "PagerDuty"

  pagerduty {
    integration_key = var.pagerduty_integration_key
    severity        = "critical"
  }
}

# Notification Policy
resource "grafana_notification_policy" "main" {
  group_by      = ["alertname", "service"]
  contact_point = grafana_contact_point.slack.name

  policy {
    matcher {
      label = "severity"
      match = "="
      value = "critical"
    }
    contact_point   = grafana_contact_point.pagerduty.name
    continue        = true
    group_wait      = "10s"
    group_interval  = "5m"
    repeat_interval = "4h"
  }
}

# Dashboard
resource "grafana_dashboard" "service_overview" {
  config_json = file("${path.module}/dashboards/service-overview.json")
  folder      = grafana_folder.production.id
}

resource "grafana_folder" "production" {
  title = "Production"
  uid   = "production"
}

# Alert Rule
resource "grafana_rule_group" "high_error_rate" {
  name             = "High Error Rate"
  folder_uid       = grafana_folder.production.uid
  interval_seconds = 60

  rule {
    name           = "High Error Rate"
    condition      = "C"
    for            = "5m"
    no_data_state  = "NoData"
    exec_err_state = "Alerting"

    data {
      ref_id = "A"
      datasource_uid = grafana_data_source.prometheus.uid
      query_type = ""
      
      model = jsonencode({
        expr = "sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m])) * 100"
        intervalMs = 1000
        maxDataPoints = 43200
        refId = "A"
      })
    }

    data {
      ref_id = "C"
      datasource_uid = "__expr__"
      
      model = jsonencode({
        conditions = [{
          evaluator = {
            params = [1]
            type   = "gt"
          }
          operator = { type = "and" }
          query    = { params = ["A"] }
          reducer  = { params = [], type = "last" }
          type     = "query"
        }]
        refId = "C"
        type  = "classic_conditions"
      })
    }

    labels = {
      severity = "warning"
      team     = "platform"
    }

    annotations = {
      summary     = "High error rate detected"
      description = "Error rate is {{ $values.A.Value | humanizePercentage }}"
    }
  }
}
```

---

## 32.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Instrument Application

สร้าง REST API ด้วย Python/Flask ที่มี:
- Custom metrics: request count, duration, error rate
- Business metrics: order count, revenue
- `/metrics` endpoint สำหรับ Prometheus

```python
# แบบฝึกหัด: เพิ่ม metrics เข้าไปใน service นี้
from flask import Flask, request, jsonify
from prometheus_client import Counter, Histogram, generate_latest

app = Flask(__name__)

# TODO: สร้าง metrics:
# 1. http_requests_total (Counter) พร้อม labels: method, endpoint, status_code
# 2. http_request_duration_seconds (Histogram)
# 3. orders_total (Counter) พร้อม labels: status
# 4. Middleware ที่บันทึก metrics อัตโนมัติ

@app.route('/orders', methods=['POST'])
def create_order():
    # TODO: บันทึก business metrics ที่นี่
    return jsonify({'id': 'ORD-001', 'status': 'created'}), 201

@app.route('/metrics')
def metrics():
    # TODO: Return Prometheus metrics
    pass
```

### แบบฝึกหัดที่ 2: สร้าง Alert Rules

สร้าง alert rules สำหรับ:
1. Service down มากกว่า 1 นาที
2. Error rate > 5% ใน 5 นาที
3. P95 latency > 500ms ใน 10 นาที
4. CPU usage > 90% ใน 15 นาที
5. Disk space < 10% remaining

### แบบฝึกหัดที่ 3: Grafana Dashboard

สร้าง Grafana dashboard ที่แสดง:
- Overview panel: request rate, error rate, P95 latency, active users
- Time series: requests/errors/latency over time
- Heatmap: request duration distribution
- Table: top 10 slowest endpoints
- Business metrics: orders, revenue

### แบบฝึกหัดที่ 4: SLO Dashboard

สร้าง SLO dashboard แสดง:
- Current SLO compliance (%)
- Error budget remaining
- Burn rate ใน 1h, 6h, 1d, 7d
- Error budget forecast

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Prometheus metric types**: Counter, Gauge, Histogram, Summary
- **Application instrumentation** ใน Python, Node.js, Go
- **PromQL** สำหรับ querying และ alerting
- **Alert rules** สำหรับ availability, latency, resources
- **Alertmanager** การ route notifications ไปยัง PagerDuty, Slack
- **Grafana dashboards** สำหรับ visualization
- **SLO-based alerting** ด้วย error budget burn rate
- **Infrastructure as Code** ด้วย Terraform + Grafana Provider

บทต่อไป (Part 33) เราจะเรียนรู้เรื่อง **Integration Testing ใน Pipeline**

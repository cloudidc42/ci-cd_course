# Part 31: Logging & Log Aggregation ใน CI/CD Pipeline

## บทนำ

Logging เป็นหัวใจสำคัญของการ observability ในระบบ production การมี log ที่ดีช่วยให้ทีม DevOps สามารถ debug ปัญหา, monitor system health, และ audit security events ได้อย่างมีประสิทธิภาพ ในบทนี้เราจะเรียนรู้การสร้าง logging strategy ที่ครอบคลุมตั้งแต่ application logging ไปจนถึง centralized log aggregation

## วัตถุประสงค์การเรียนรู้

- เข้าใจหลักการ structured logging และ log levels
- สร้าง centralized logging ด้วย ELK Stack
- ใช้ Fluentd และ Fluent Bit สำหรับ log collection
- ตั้งค่า Loki + Grafana สำหรับ log visualization
- กำหนด log retention policy
- Integrate logging เข้ากับ CI/CD pipeline

---

## 31.1 หลักการ Structured Logging

### ทำไมต้อง Structured Logging?

Unstructured log (แบบเดิม):
```
2024-01-15 10:23:45 ERROR User authentication failed for user john@example.com from IP 192.168.1.100
```

Structured log (JSON format):
```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "message": "User authentication failed",
  "user_email": "john@example.com",
  "client_ip": "192.168.1.100",
  "request_id": "req-abc123",
  "service": "auth-service",
  "version": "1.2.3",
  "environment": "production"
}
```

ข้อดีของ Structured Logging:
- Parse ได้ง่ายด้วย log aggregation tools
- ค้นหาและ filter ได้แม่นยำ
- สร้าง metrics จาก logs ได้
- ง่ายต่อการ correlate กับ traces

### Log Levels มาตรฐาน

| Level | ค่า | ใช้เมื่อ |
|-------|-----|---------|
| TRACE | 5 | ข้อมูล debugging ละเอียดมาก |
| DEBUG | 10 | ข้อมูล debugging ทั่วไป |
| INFO | 20 | การทำงานปกติของระบบ |
| WARN | 30 | เหตุการณ์ที่ควรสังเกต แต่ไม่ใช่ error |
| ERROR | 40 | Error ที่ต้องการการแก้ไข |
| FATAL | 50 | Error ที่ทำให้ระบบหยุดทำงาน |

---

## 31.2 Structured Logging ใน Applications

### Python - ใช้ structlog

```python
# requirements.txt
# structlog==23.2.0
# python-json-logger==2.0.7

import structlog
import logging
import sys
from datetime import datetime

def configure_logging(service_name: str, environment: str, log_level: str = "INFO"):
    """ตั้งค่า structured logging"""
    
    # ตั้งค่า processors
    processors = [
        structlog.contextvars.merge_contextvars,
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        # เพิ่ม context fields
        add_service_context(service_name, environment),
        structlog.processors.JSONRenderer()
    ]
    
    structlog.configure(
        processors=processors,
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )
    
    # ตั้งค่า standard logging
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, log_level.upper())
    )

def add_service_context(service_name: str, environment: str):
    """Processor เพิ่ม service context"""
    def processor(logger, method, event_dict):
        event_dict["service"] = service_name
        event_dict["environment"] = environment
        return event_dict
    return processor

# การใช้งาน
configure_logging("order-service", "production")
logger = structlog.get_logger()

# Log พื้นฐาน
logger.info("Order created", order_id="ORD-001", user_id="USR-123", amount=1500.00)

# Log พร้อม context
with structlog.contextvars.bound_contextvars(request_id="req-xyz789"):
    logger.info("Processing payment", payment_method="credit_card")
    logger.warning("Payment gateway slow", response_time_ms=2500)
    
# Log exception
try:
    raise ValueError("Invalid order amount")
except ValueError as e:
    logger.error("Order processing failed", 
                 order_id="ORD-002",
                 exc_info=True)
```

### Node.js - ใช้ Pino

```javascript
// package.json dependencies:
// "pino": "^8.0.0",
// "pino-pretty": "^10.0.0"

const pino = require('pino');
const { v4: uuidv4 } = require('uuid');

// สร้าง logger configuration
const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
    bindings: (bindings) => ({
      service: process.env.SERVICE_NAME || 'unknown',
      version: process.env.APP_VERSION || '0.0.0',
      environment: process.env.NODE_ENV || 'development',
      pid: bindings.pid,
      hostname: bindings.hostname
    })
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  serializers: {
    err: pino.stdSerializers.err,
    req: (req) => ({
      method: req.method,
      url: req.url,
      headers: {
        'content-type': req.headers['content-type'],
        'user-agent': req.headers['user-agent']
      },
      remoteAddress: req.remoteAddress
    }),
    res: (res) => ({
      statusCode: res.statusCode
    })
  }
});

// Middleware สำหรับ Express
function requestLogger(req, res, next) {
  const requestId = req.headers['x-request-id'] || uuidv4();
  req.requestId = requestId;
  
  const childLogger = logger.child({ requestId });
  req.log = childLogger;
  
  const startTime = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    childLogger.info({
      req,
      res,
      duration_ms: duration,
      msg: 'Request completed'
    });
  });
  
  childLogger.info({ req, msg: 'Request received' });
  next();
}

// การใช้งานใน service
class OrderService {
  constructor() {
    this.log = logger.child({ component: 'OrderService' });
  }
  
  async createOrder(userId, items) {
    const orderId = uuidv4();
    
    this.log.info({
      orderId,
      userId,
      itemCount: items.length,
      msg: 'Creating order'
    });
    
    try {
      // Business logic
      const order = await this.processOrder(userId, items);
      
      this.log.info({
        orderId,
        userId,
        totalAmount: order.total,
        msg: 'Order created successfully'
      });
      
      return order;
    } catch (error) {
      this.log.error({
        orderId,
        userId,
        err: error,
        msg: 'Failed to create order'
      });
      throw error;
    }
  }
}

module.exports = { logger, requestLogger, OrderService };
```

### Go - ใช้ zerolog

```go
package main

import (
    "context"
    "os"
    "time"
    
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "github.com/google/uuid"
)

type contextKey string

const requestIDKey contextKey = "requestID"

func initLogger() zerolog.Logger {
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnixMs
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
    
    logger := zerolog.New(os.Stdout).
        With().
        Timestamp().
        Str("service", getEnv("SERVICE_NAME", "unknown")).
        Str("version", getEnv("APP_VERSION", "0.0.0")).
        Str("environment", getEnv("APP_ENV", "development")).
        Logger()
    
    return logger
}

func getEnv(key, defaultVal string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return defaultVal
}

// ContextLogger เพิ่ม context ให้ logger
func ContextLogger(ctx context.Context, logger zerolog.Logger) zerolog.Logger {
    if requestID, ok := ctx.Value(requestIDKey).(string); ok {
        return logger.With().Str("request_id", requestID).Logger()
    }
    return logger
}

// OrderService ตัวอย่างการใช้ logger
type OrderService struct {
    logger zerolog.Logger
    db     Database
}

func NewOrderService(logger zerolog.Logger, db Database) *OrderService {
    return &OrderService{
        logger: logger.With().Str("component", "OrderService").Logger(),
        db:     db,
    }
}

func (s *OrderService) CreateOrder(ctx context.Context, userID string, items []Item) (*Order, error) {
    orderID := uuid.New().String()
    log := ContextLogger(ctx, s.logger).With().
        Str("order_id", orderID).
        Str("user_id", userID).
        Logger()
    
    log.Info().
        Int("item_count", len(items)).
        Msg("Creating order")
    
    start := time.Now()
    
    order, err := s.db.InsertOrder(ctx, orderID, userID, items)
    if err != nil {
        log.Error().
            Err(err).
            Dur("duration", time.Since(start)).
            Msg("Failed to create order")
        return nil, err
    }
    
    log.Info().
        Float64("total_amount", order.Total).
        Dur("duration", time.Since(start)).
        Msg("Order created successfully")
    
    return order, nil
}

func main() {
    logger := initLogger()
    log.Logger = logger
    
    logger.Info().Msg("Service starting")
    
    // สร้าง context พร้อม request ID
    ctx := context.WithValue(context.Background(), requestIDKey, uuid.New().String())
    
    contextLogger := ContextLogger(ctx, logger)
    contextLogger.Debug().
        Str("feature", "new_checkout").
        Bool("enabled", true).
        Msg("Feature flag checked")
}
```

### Java - ใช้ Logback + SLF4J

```xml
<!-- logback.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"order-service","environment":"${APP_ENV:-development}"}</customFields>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <version>[ignore]</version>
                <levelValue>[ignore]</levelValue>
            </fieldNames>
        </encoder>
    </appender>
    
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/app/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/app/application.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy 
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

```java
// OrderService.java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import net.logstash.logback.argument.StructuredArguments;

public class OrderService {
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    public Order createOrder(String userId, List<Item> items) {
        String orderId = UUID.randomUUID().toString();
        
        // เพิ่ม context ด้วย MDC
        MDC.put("order_id", orderId);
        MDC.put("user_id", userId);
        
        try {
            logger.info("Creating order",
                StructuredArguments.kv("item_count", items.size()),
                StructuredArguments.kv("component", "OrderService"));
            
            Order order = processOrder(userId, items);
            
            logger.info("Order created successfully",
                StructuredArguments.kv("total_amount", order.getTotal()),
                StructuredArguments.kv("payment_method", order.getPaymentMethod()));
            
            return order;
        } catch (Exception e) {
            logger.error("Failed to create order",
                StructuredArguments.kv("error_type", e.getClass().getSimpleName()),
                e);
            throw e;
        } finally {
            MDC.remove("order_id");
            MDC.remove("user_id");
        }
    }
}
```

---

## 31.3 ELK Stack (Elasticsearch, Logstash, Kibana)

### Architecture Overview

```
Applications --> Filebeat --> Logstash --> Elasticsearch --> Kibana
                    |                           |
                    +-- Direct Ingest ----------+
```

### Docker Compose สำหรับ ELK Stack

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - node.name=elasticsearch
      - cluster.name=ci-cd-logs
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - elk
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_URL: http://elasticsearch:9200
      ELASTICSEARCH_HOSTS: '["http://elasticsearch:9200"]'
    networks:
      - elk
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - elk
    depends_on:
      - logstash

networks:
  elk:
    driver: bridge

volumes:
  elasticsearch_data:
    driver: local
```

### Logstash Pipeline Configuration

```ruby
# logstash/pipeline/logstash.conf

input {
  beats {
    port => 5044
  }
  
  # รับ logs จาก TCP (สำหรับ applications)
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  # Parse JSON logs
  if [message] =~ /^\{/ {
    json {
      source => "message"
      target => "parsed"
      remove_field => ["message"]
    }
    
    # Flatten nested JSON
    ruby {
      code => '
        parsed = event.get("parsed")
        if parsed.is_a?(Hash)
          parsed.each { |k, v| event.set(k, v) }
          event.remove("parsed")
        end
      '
    }
  }
  
  # Parse timestamp
  date {
    match => ["timestamp", "ISO8601", "yyyy-MM-dd'T'HH:mm:ss.SSSZ"]
    target => "@timestamp"
    remove_field => ["timestamp"]
  }
  
  # เพิ่ม geo information สำหรับ IP
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geoip"
    }
  }
  
  # Normalize log levels
  mutate {
    uppercase => ["level"]
  }
  
  # เพิ่ม tags ตาม log level
  if [level] == "ERROR" or [level] == "FATAL" {
    mutate {
      add_tag => ["error", "alert"]
    }
  }
  
  # กรอง sensitive fields
  mutate {
    remove_field => ["password", "credit_card", "ssn", "token"]
  }
  
  # เพิ่ม environment tag
  if ![environment] {
    mutate {
      add_field => { "environment" => "unknown" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[environment]}-%{[service]}-%{+YYYY.MM.dd}"
    
    # ILM Policy
    ilm_enabled => true
    ilm_rollover_alias => "logs"
    ilm_pattern => "{now/d}-000001"
    ilm_policy => "logs-policy"
  }
  
  # Output errors ไปยัง separate index
  if "error" in [tags] {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
  }
  
  # Debug output (ปิดใน production)
  # stdout { codec => rubydebug }
}
```

### Filebeat Configuration

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
  # Docker container logs
  - type: container
    paths:
      - /var/lib/docker/containers/*/*.log
    processors:
      - add_docker_metadata:
          host: "unix:///var/run/docker.sock"
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true
    
  # Application log files
  - type: filestream
    id: app-logs
    paths:
      - /var/log/app/*.log
    parsers:
      - ndjson:
          target: ""
          overwrite_keys: true

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_fields:
      target: ''
      fields:
        pipeline: filebeat

output.logstash:
  hosts: ["logstash:5044"]
  loadbalance: true
  bulk_max_size: 2048

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0640
```

---

## 31.4 Fluentd สำหรับ Kubernetes

### Fluentd DaemonSet Configuration

```yaml
# fluentd-daemonset.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - namespaces
    verbs:
      - get
      - list
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: fluentd
    namespace: kube-system

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
      version: v1
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd-kubernetes-daemonset:v1.16-debian-elasticsearch8-1
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch.logging.svc.cluster.local"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
            - name: FLUENT_ELASTICSEARCH_SCHEME
              value: "http"
            - name: FLUENTD_SYSTEMD_CONF
              value: disable
            - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
              value: /var/log/containers/fluent*
            - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
              value: /^(?<time>.+) (?<stream>stdout|stderr)( (?<logtag>.))? (?<log>.*)$/
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: dockercontainerlogdirectory
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: fluentd-config
              mountPath: /fluentd/etc
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: dockercontainerlogdirectory
          hostPath:
            path: /var/lib/docker/containers
        - name: fluentd-config
          configMap:
            name: fluentd-config
```

### Fluentd Configuration (fluent.conf)

```yaml
# fluentd-config.yaml (ConfigMap)
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: kube-system
data:
  fluent.conf: |
    @include kubernetes.conf
    @include containers.conf
    
    # Match all kubernetes logs
    <match kubernetes.**>
      @type elasticsearch
      @id out_es
      @log_level info
      include_tag_key true
      
      host "#{ENV['FLUENT_ELASTICSEARCH_HOST']}"
      port "#{ENV['FLUENT_ELASTICSEARCH_PORT']}"
      scheme "#{ENV['FLUENT_ELASTICSEARCH_SCHEME']}"
      
      logstash_format true
      logstash_prefix kubernetes
      
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>

  kubernetes.conf: |
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
      kubernetes_url "https://#{ENV.fetch('KUBERNETES_SERVICE_HOST')}:#{ENV.fetch('KUBERNETES_SERVICE_PORT')}/api"
      verify_ssl false
      ca_file /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      skip_labels false
      skip_container_metadata false
      skip_master_url false
      skip_namespace_metadata false
    </filter>
    
    # Parse JSON logs
    <filter kubernetes.**>
      @type parser
      key_name log
      reserve_data true
      remove_key_name_field true
      <parse>
        @type multi_format
        <pattern>
          format json
          time_key timestamp
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </pattern>
        <pattern>
          format none
        </pattern>
      </parse>
    </filter>
```

---

## 31.5 Loki + Grafana สำหรับ Log Visualization

### Loki Stack Setup ด้วย Helm

```bash
# เพิ่ม Grafana Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# สร้าง namespace
kubectl create namespace monitoring

# ติดตั้ง Loki Stack
helm install loki-stack grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set prometheus.enabled=true \
  --set prometheus.alertmanager.persistentVolume.enabled=false \
  --set prometheus.server.persistentVolume.enabled=false \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi
```

### Loki Configuration

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

limits_config:
  retention_period: 30d
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_query_parallelism: 32
```

### Promtail Configuration

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Kubernetes pod logs
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_promtail_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_promtail_io_path]
        action: replace
        target_label: __path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod
      - source_labels: [__meta_kubernetes_container_name]
        action: replace
        target_label: container
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: replace
        target_label: app
    
    pipeline_stages:
      - docker: {}
      - json:
          expressions:
            level: level
            message: message
            service: service
      - labels:
          level:
          service:
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
          __path__: /var/log/*log
```

### LogQL Queries ใน Grafana

```
# ดู logs ทั้งหมดจาก service
{app="order-service"}

# Filter ตาม log level
{app="order-service"} |= "level=ERROR"

# Parse JSON logs
{app="order-service"} | json | level="ERROR"

# นับ error rate ต่อนาที
sum(rate({app="order-service"} |= "ERROR" [1m])) by (pod)

# หา top 10 error messages
topk(10, sum by (message) (count_over_time({app="order-service"} | json | level="ERROR" [1h])))

# ดู logs ของ specific request ID
{namespace="production"} |= "req-abc123"

# Pattern matching
{app="payment-service"} | pattern `<_> level=<level> <_> msg="<message>" <_>` | level="ERROR"

# Log volume ต่อ service
sum by (service) (
  rate(
    {namespace="production"} | json | line_format "{{.service}}" [5m]
  )
)
```

---

## 31.6 Log Retention Policy

### Elasticsearch Index Lifecycle Management (ILM)

```json
// PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_primary_shard_size": "50gb",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {
            "delete_searchable_snapshot": true
          }
        }
      }
    }
  }
}
```

### Log Archival Script

```bash
#!/bin/bash
# log-archival.sh - Archive logs ก่อน retention period

set -euo pipefail

ELASTICSEARCH_URL="${ELASTICSEARCH_URL:-http://localhost:9200}"
S3_BUCKET="${S3_BUCKET:-my-log-archive}"
ARCHIVE_AFTER_DAYS="${ARCHIVE_AFTER_DAYS:-30}"
LOG_DIR="/var/log/archive"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

# Archive Elasticsearch indices
archive_elasticsearch_indices() {
    log "Archiving Elasticsearch indices older than ${ARCHIVE_AFTER_DAYS} days..."
    
    # ดึง indices ที่ต้องการ archive
    OLD_INDICES=$(curl -s "${ELASTICSEARCH_URL}/_cat/indices/logs-*?h=index,creation.date" | \
        awk -v days="${ARCHIVE_AFTER_DAYS}" '
        {
            split($2, d, ".");
            creation_ts = d[1]/1000;
            now = systime();
            age_days = (now - creation_ts) / 86400;
            if (age_days > days) print $1;
        }')
    
    if [ -z "$OLD_INDICES" ]; then
        log "No indices to archive"
        return
    fi
    
    for INDEX in $OLD_INDICES; do
        log "Archiving index: $INDEX"
        
        # Snapshot ไปยัง S3
        curl -s -X PUT "${ELASTICSEARCH_URL}/_snapshot/s3-backup/${INDEX}" \
            -H 'Content-Type: application/json' \
            -d "{
                \"indices\": \"${INDEX}\",
                \"ignore_unavailable\": true,
                \"include_global_state\": false
            }"
        
        # รอให้ snapshot เสร็จ
        while true; do
            STATUS=$(curl -s "${ELASTICSEARCH_URL}/_snapshot/s3-backup/${INDEX}/_status" | \
                jq -r '.snapshots[0].state // "UNKNOWN"')
            
            if [ "$STATUS" == "SUCCESS" ]; then
                log "Snapshot completed for $INDEX"
                break
            elif [ "$STATUS" == "FAILED" ]; then
                log "ERROR: Snapshot failed for $INDEX"
                exit 1
            fi
            
            sleep 10
        done
        
        # ลบ index เดิม
        curl -s -X DELETE "${ELASTICSEARCH_URL}/${INDEX}"
        log "Deleted index: $INDEX"
    done
}

# Archive local log files
archive_local_logs() {
    log "Archiving local log files..."
    
    find /var/log/app -name "*.log" -mtime "+${ARCHIVE_AFTER_DAYS}" | while read -r logfile; do
        ARCHIVE_PATH="${LOG_DIR}/$(basename "$logfile").$(date '+%Y%m%d').gz"
        
        gzip -c "$logfile" > "$ARCHIVE_PATH"
        
        # Upload ไปยัง S3
        aws s3 cp "$ARCHIVE_PATH" \
            "s3://${S3_BUCKET}/logs/$(date '+%Y/%m/')$(basename "$ARCHIVE_PATH")" \
            --storage-class GLACIER
        
        rm "$logfile"
        log "Archived: $logfile -> S3"
    done
}

main() {
    log "Starting log archival process"
    archive_elasticsearch_indices
    archive_local_logs
    log "Log archival completed"
}

main
```

---

## 31.7 Integration กับ CI/CD Pipeline

### GitHub Actions - Logging Setup

```yaml
# .github/workflows/deploy-with-logging.yml
name: Deploy with Logging

on:
  push:
    branches: [main]

env:
  SERVICE_NAME: order-service
  ELASTICSEARCH_URL: ${{ secrets.ELASTICSEARCH_URL }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure Build Logger
        run: |
          cat > /tmp/log_helper.sh << 'EOF'
          #!/bin/bash
          log_event() {
            local level=$1
            local message=$2
            shift 2
            local extra_fields="$*"
            
            curl -s -X POST "${ELASTICSEARCH_URL}/ci-logs/_doc" \
              -H "Content-Type: application/json" \
              -d "{
                \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",
                \"level\": \"${level}\",
                \"message\": \"${message}\",
                \"service\": \"${SERVICE_NAME}\",
                \"build_id\": \"${GITHUB_RUN_ID}\",
                \"commit_sha\": \"${GITHUB_SHA}\",
                \"branch\": \"${GITHUB_REF_NAME}\",
                ${extra_fields}
                \"pipeline\": \"github-actions\"
              }"
          }
          EOF
          chmod +x /tmp/log_helper.sh
      
      - name: Build Application
        run: |
          source /tmp/log_helper.sh
          log_event "INFO" "Starting build" "\"stage\": \"build\","
          
          if docker build -t ${{ env.SERVICE_NAME }}:${{ github.sha }} .; then
            log_event "INFO" "Build successful" "\"stage\": \"build\", \"duration_seconds\": \"$SECONDS\","
          else
            log_event "ERROR" "Build failed" "\"stage\": \"build\","
            exit 1
          fi
      
      - name: Run Tests
        run: |
          source /tmp/log_helper.sh
          log_event "INFO" "Starting tests" "\"stage\": \"test\","
          
          # Capture test output
          TEST_OUTPUT=$(docker run --rm ${{ env.SERVICE_NAME }}:${{ github.sha }} npm test 2>&1)
          TEST_EXIT_CODE=$?
          
          if [ $TEST_EXIT_CODE -eq 0 ]; then
            log_event "INFO" "Tests passed" "\"stage\": \"test\", \"test_output\": \"$(echo $TEST_OUTPUT | tail -1)\","
          else
            log_event "ERROR" "Tests failed" "\"stage\": \"test\","
            echo "$TEST_OUTPUT"
            exit $TEST_EXIT_CODE
          fi
      
      - name: Deploy to Production
        run: |
          source /tmp/log_helper.sh
          log_event "INFO" "Starting deployment" "\"stage\": \"deploy\", \"environment\": \"production\","
          
          # Deploy commands here
          kubectl set image deployment/${{ env.SERVICE_NAME }} \
            ${{ env.SERVICE_NAME }}=${{ env.SERVICE_NAME }}:${{ github.sha }} \
            --namespace=production
          
          kubectl rollout status deployment/${{ env.SERVICE_NAME }} \
            --namespace=production \
            --timeout=5m
          
          log_event "INFO" "Deployment successful" "\"stage\": \"deploy\", \"environment\": \"production\","
```

### Centralized Build Log Aggregation

```python
#!/usr/bin/env python3
# log_aggregator.py - รวบรวม CI/CD build logs

import json
import requests
from datetime import datetime
from typing import Dict, Any, Optional

class CICDLogAggregator:
    def __init__(self, elasticsearch_url: str, index_prefix: str = "ci-cd-logs"):
        self.es_url = elasticsearch_url
        self.index_prefix = index_prefix
    
    def log_build_event(
        self,
        event_type: str,
        message: str,
        metadata: Optional[Dict[str, Any]] = None
    ) -> bool:
        """ส่ง build event ไปยัง Elasticsearch"""
        
        doc = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "event_type": event_type,
            "message": message,
            "metadata": metadata or {}
        }
        
        index_name = f"{self.index_prefix}-{datetime.utcnow().strftime('%Y.%m.%d')}"
        
        try:
            response = requests.post(
                f"{self.es_url}/{index_name}/_doc",
                json=doc,
                timeout=5
            )
            response.raise_for_status()
            return True
        except requests.exceptions.RequestException as e:
            print(f"Failed to log event: {e}")
            return False
    
    def log_pipeline_summary(
        self,
        pipeline_id: str,
        repository: str,
        branch: str,
        commit_sha: str,
        stages: list,
        overall_status: str,
        duration_seconds: int
    ) -> bool:
        """บันทึกสรุป pipeline execution"""
        
        summary = {
            "timestamp": datetime.utcnow().isoformat() + "Z",
            "event_type": "pipeline_summary",
            "pipeline_id": pipeline_id,
            "repository": repository,
            "branch": branch,
            "commit_sha": commit_sha,
            "stages": stages,
            "overall_status": overall_status,
            "duration_seconds": duration_seconds,
            "stage_count": len(stages),
            "failed_stages": [s for s in stages if s.get("status") == "failed"]
        }
        
        index_name = f"pipeline-summaries-{datetime.utcnow().strftime('%Y.%m')}"
        
        try:
            response = requests.post(
                f"{self.es_url}/{index_name}/_doc/{pipeline_id}",
                json=summary,
                timeout=5
            )
            response.raise_for_status()
            return True
        except requests.exceptions.RequestException as e:
            print(f"Failed to log pipeline summary: {e}")
            return False

# ตัวอย่างการใช้งาน
if __name__ == "__main__":
    aggregator = CICDLogAggregator(
        elasticsearch_url="http://elasticsearch:9200"
    )
    
    # Log build events
    aggregator.log_build_event(
        event_type="build_started",
        message="Building Docker image",
        metadata={
            "service": "order-service",
            "commit_sha": "abc123",
            "branch": "main"
        }
    )
    
    # Log pipeline summary
    aggregator.log_pipeline_summary(
        pipeline_id="build-12345",
        repository="my-org/order-service",
        branch="main",
        commit_sha="abc123",
        stages=[
            {"name": "build", "status": "success", "duration": 120},
            {"name": "test", "status": "success", "duration": 60},
            {"name": "deploy", "status": "success", "duration": 45}
        ],
        overall_status="success",
        duration_seconds=225
    )
```

---

## 31.8 Log Analysis และ Alerting

### Kibana Watcher สำหรับ Log Alerts

```json
// PUT _watcher/watch/high-error-rate
{
  "trigger": {
    "schedule": {
      "interval": "5m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["logs-production-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"term": {"level": "ERROR"}},
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m"
                    }
                  }
                }
              ]
            }
          },
          "aggs": {
            "error_count": {
              "value_count": {
                "field": "_id"
              }
            },
            "by_service": {
              "terms": {
                "field": "service",
                "size": 10
              }
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.aggregations.error_count.value": {
        "gt": 100
      }
    }
  },
  "actions": {
    "send_slack": {
      "webhook": {
        "scheme": "https",
        "host": "hooks.slack.com",
        "port": 443,
        "method": "post",
        "path": "/services/YOUR/SLACK/WEBHOOK",
        "params": {},
        "headers": {
          "Content-Type": "application/json"
        },
        "body": "{\"text\": \"High Error Rate Alert: {{ctx.payload.aggregations.error_count.value}} errors in the last 5 minutes\", \"channel\": \"#alerts\"}"
      }
    }
  }
}
```

---

## 31.9 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ตั้งค่า Structured Logging

1. สร้าง Node.js application ที่ใช้ Pino สำหรับ structured logging
2. ตั้งค่า log levels ตาม environment (DEBUG ใน dev, INFO ใน prod)
3. เพิ่ม request ID middleware
4. เพิ่ม log sanitization สำหรับ sensitive fields

```javascript
// แบบฝึกหัด: เติมโค้ดให้ครบ
const pino = require('pino');

// TODO: สร้าง logger ที่:
// 1. ใช้ JSON format
// 2. เพิ่ม service name จาก env var
// 3. Sanitize fields: password, token, apiKey
// 4. ตั้งค่า log level จาก LOG_LEVEL env var

const logger = pino({
  // TODO: เติม configuration
  redact: {
    paths: ['password', 'token', 'apiKey', '*.password', '*.token'],
    censor: '[REDACTED]'
  }
});
```

### แบบฝึกหัดที่ 2: Deploy ELK Stack

1. Clone โปรเจค: `git clone https://github.com/deviantony/docker-elk`
2. แก้ไข Logstash pipeline ให้ parse JSON logs
3. สร้าง Kibana dashboard แสดง:
   - Error rate ต่อ service
   - Log volume ตามเวลา
   - Top 10 error messages

### แบบฝึกหัดที่ 3: Log Correlation

สร้าง system ที่ correlate logs ระหว่าง services โดยใช้ trace ID:

```python
# ใช้ OpenTelemetry สำหรับ distributed tracing
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
import structlog

# TODO: 
# 1. ตั้งค่า OpenTelemetry tracer
# 2. Inject trace ID เข้าไปใน logs
# 3. สร้าง request ที่ผ่านหลาย services
# 4. ดู correlated logs ใน Kibana
```

### แบบฝึกหัดที่ 4: Log Retention Automation

สร้าง script ที่:
1. ตรวจสอบ disk usage
2. Archive logs ที่เก่ากว่า 30 วัน ไปยัง S3
3. ลบ logs ที่เก่ากว่า 90 วัน
4. ส่ง report ไปยัง Slack

---

## 31.10 Best Practices สรุป

### Do's ✅

1. **ใช้ structured logging เสมอ** - JSON format ทำให้ parse และ search ได้ง่าย
2. **เพิ่ม context ที่จำเป็น** - service name, version, environment, request ID
3. **ใช้ log levels อย่างถูกต้อง** - ไม่ log ทุกอย่างด้วย INFO
4. **ตั้งค่า log rotation** - ป้องกัน disk full
5. **Sanitize sensitive data** - ไม่ log passwords, tokens, PII
6. **ทำ centralized logging** - รวม logs จากทุก services ไว้ที่เดียว
7. **ตั้งค่า retention policy** - ลบ logs เก่าอย่างอัตโนมัติ
8. **สร้าง alerts จาก logs** - ตรวจจับ errors โดยอัตโนมัติ

### Don'ts ❌

1. **อย่า log sensitive information** - passwords, credit cards, personal data
2. **อย่า log ทุกอย่าง** - เลือก log เฉพาะที่มีประโยชน์
3. **อย่า ignore log errors** - ตรวจสอบว่า logs ส่งถึง aggregator
4. **อย่าใช้ synchronous logging ใน critical path** - ใช้ async logging แทน
5. **อย่า hardcode log levels** - ใช้ environment variables

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- หลักการ **Structured Logging** และการ implement ใน Python, Node.js, Go, Java
- การตั้งค่า **ELK Stack** สำหรับ centralized log management
- การใช้ **Fluentd** สำหรับ log collection ใน Kubernetes
- การตั้งค่า **Loki + Grafana** เป็นทางเลือกที่ lightweight กว่า
- การกำหนด **Log Retention Policy** ด้วย ILM
- การ integrate logging เข้ากับ **CI/CD pipeline**

บทต่อไป (Part 32) เราจะเรียนรู้เรื่อง **Metrics & Alerting** ด้วย Prometheus และ Grafana

---

*หมายเหตุ: ตรวจสอบ versions ของ tools ที่ใช้งานเพื่อให้ compatible กับระบบของคุณ*

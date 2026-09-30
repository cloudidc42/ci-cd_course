# Part 75: Distributed Tracing

## บทนำ

Distributed Tracing คือเทคนิคที่ใช้ติดตาม request ที่เดินทางผ่าน microservices หลายตัว ช่วยให้เข้าใจ latency, ค้นหา bottleneck และ debug ปัญหาที่เกิดใน distributed systems

ในบทนี้เราจะเรียนรู้:
- OpenTelemetry มาตรฐาน
- การตั้งค่า Jaeger
- การตั้งค่า Zipkin
- Trace Propagation
- CI/CD Pipeline Instrumentation
- Performance Profiling
- แบบฝึกหัดปฏิบัติ

---

## 75.1 แนวคิด Distributed Tracing

### Trace และ Span

```
Trace: การเดินทางของ request ผ่านระบบทั้งหมด
  └─ Span: หน่วยงานแต่ละชิ้น

ตัวอย่าง:
GET /checkout [Trace ID: abc123]
├─ api-gateway: validate request (5ms)
├─ cart-service: get cart items (15ms)
│   └─ database: query cart (10ms)
├─ inventory-service: check stock (25ms)
│   └─ database: query inventory (20ms)
└─ payment-service: process payment (150ms)
    ├─ fraud-check: detect fraud (30ms)
    └─ payment-gateway: charge card (100ms)

Total: 200ms
Bottleneck: payment-gateway
```

### ข้อมูลใน Span

```
Span {
  trace_id: "abc123def456",
  span_id: "span789",
  parent_span_id: "span456",  // parent span
  operation_name: "payment.process",
  service_name: "payment-service",
  start_time: 1700000000000,
  end_time:   1700000000150,
  duration: 150ms,
  status: "OK",
  attributes: {
    "http.method": "POST",
    "http.url": "/api/payment",
    "http.status_code": 200,
    "payment.amount": 1500.00,
    "payment.currency": "THB",
    "db.system": "postgresql",
    "db.statement": "SELECT ...",
  },
  events: [
    { name: "fraud_check_started", timestamp: ... },
    { name: "fraud_check_passed", timestamp: ... },
  ],
  links: [],
}
```

---

## 75.2 OpenTelemetry

OpenTelemetry คือ vendor-neutral framework สำหรับ observability (traces, metrics, logs)

### OpenTelemetry Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Application                               │
│  ┌──────────────┐  ┌─────────────────────────────────────────┐  │
│  │ Instrumented │  │            OpenTelemetry SDK             │  │
│  │     Code     │→ │  Tracer  │  Meter  │  Logger           │  │
│  └──────────────┘  └─────────────────────────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ OTLP (gRPC/HTTP)
                    ┌──────────────▼──────────────────┐
                    │    OpenTelemetry Collector       │
                    │  ┌──────────┐  ┌──────────────┐ │
                    │  │ Receiver │→ │  Processor   │ │
                    │  └──────────┘  └──────┬───────┘ │
                    │                       │          │
                    │               ┌───────▼──────┐  │
                    │               │   Exporter   │  │
                    └───────────────┴──────┬───────┴──┘
                                           │
              ┌────────────────────────────┴────────────────┐
              │                    │                        │
       ┌──────▼──────┐    ┌───────▼──────┐    ┌───────────▼─────┐
       │   Jaeger    │    │   Zipkin     │    │  Prometheus     │
       │  (Tracing)  │    │  (Tracing)   │    │  (Metrics)      │
       └─────────────┘    └─────────────┘    └─────────────────┘
```

### ติดตั้ง OpenTelemetry Collector

```yaml
# otel-collector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:latest
          ports:
            - containerPort: 4317  # gRPC
            - containerPort: 4318  # HTTP
            - containerPort: 8888  # Metrics
          volumeMounts:
            - name: config
              mountPath: /etc/otelcol
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

      # รับ metrics จาก Prometheus
      prometheus:
        config:
          scrape_configs:
            - job_name: 'otel-collector'
              static_configs:
                - targets: ['localhost:8888']

    processors:
      batch:
        timeout: 10s
        send_batch_size: 1024

      memory_limiter:
        check_interval: 1s
        limit_mib: 400
        spike_limit_mib: 100

      # เพิ่ม resource attributes
      resource:
        attributes:
          - key: deployment.environment
            value: ${ENVIRONMENT}
            action: upsert

      # กรอง spans ที่ไม่สำคัญ
      filter:
        error_mode: ignore
        traces:
          span:
            - 'name == "health_check"'
            - 'attributes["http.url"] == "/health"'

    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true

      zipkin:
        endpoint: http://zipkin:9411/api/v2/spans

      prometheus:
        endpoint: "0.0.0.0:8889"

      logging:
        loglevel: info

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, resource, filter, batch]
          exporters: [jaeger, zipkin]

        metrics:
          receivers: [otlp, prometheus]
          processors: [memory_limiter, batch]
          exporters: [prometheus]

        logs:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [logging]
```

---

## 75.3 Instrumentation ใน Go

### การ Setup Tracer

```go
// tracing/setup.go
package tracing

import (
    "context"
    "fmt"
    "os"
    "time"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/sdk/resource"
    sdktrace "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

// InitTracer สร้างและ configure trace provider
func InitTracer(serviceName, serviceVersion string) (*sdktrace.TracerProvider, error) {
    ctx := context.Background()

    // สร้าง OTLP exporter
    otlpEndpoint := os.Getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
    if otlpEndpoint == "" {
        otlpEndpoint = "localhost:4317"
    }

    conn, err := grpc.DialContext(ctx, otlpEndpoint,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
        grpc.WithBlock(),
        grpc.WithTimeout(5*time.Second),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to connect to OTLP: %w", err)
    }

    exporter, err := otlptracegrpc.New(ctx, otlptracegrpc.WithGRPCConn(conn))
    if err != nil {
        return nil, fmt.Errorf("failed to create exporter: %w", err)
    }

    // Resource attributes
    r, err := resource.Merge(
        resource.Default(),
        resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion(serviceVersion),
            attribute.String("deployment.environment",
                os.Getenv("ENVIRONMENT")),
            attribute.String("k8s.pod.name",
                os.Getenv("POD_NAME")),
            attribute.String("k8s.namespace.name",
                os.Getenv("NAMESPACE")),
        ),
    )
    if err != nil {
        return nil, err
    }

    // สร้าง TracerProvider
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exporter,
            sdktrace.WithBatchTimeout(5*time.Second),
            sdktrace.WithMaxExportBatchSize(512),
        ),
        sdktrace.WithResource(r),
        sdktrace.WithSampler(
            sdktrace.ParentBased(
                sdktrace.TraceIDRatioBased(getSamplingRate()),
            ),
        ),
    )

    // ตั้งเป็น global tracer provider
    otel.SetTracerProvider(tp)

    // ตั้งค่า propagation (W3C Trace Context + B3)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))

    return tp, nil
}

func getSamplingRate() float64 {
    env := os.Getenv("ENVIRONMENT")
    switch env {
    case "production":
        return 0.1  // 10% sampling
    case "staging":
        return 0.5  // 50% sampling
    default:
        return 1.0  // 100% sampling
    }
}
```

### HTTP Server Instrumentation

```go
// tracing/http.go
package tracing

import (
    "net/http"

    "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("payment-service")

// InstrumentedHandler wraps HTTP handler ด้วย tracing
func InstrumentedHandler(handler http.Handler) http.Handler {
    return otelhttp.NewHandler(handler, "http.request",
        otelhttp.WithSpanNameFormatter(func(operation string, r *http.Request) string {
            return fmt.Sprintf("%s %s", r.Method, r.URL.Path)
        }),
    )
}

// Middleware สำหรับ custom attributes
func TracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        span := trace.SpanFromContext(r.Context())

        // เพิ่ม custom attributes
        span.SetAttributes(
            attribute.String("user.id", r.Header.Get("X-User-ID")),
            attribute.String("request.id", r.Header.Get("X-Request-ID")),
            attribute.String("team", "payments"),
        )

        // Wrapped ResponseWriter เพื่อ capture status code
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}
        next.ServeHTTP(wrapped, r)

        // เพิ่ม response attributes
        span.SetAttributes(
            attribute.Int("http.response.status_code", wrapped.statusCode),
        )

        if wrapped.statusCode >= 500 {
            span.SetStatus(codes.Error, "Internal server error")
        }
    })
}

// Payment Handler ตัวอย่าง
func ProcessPaymentHandler(w http.ResponseWriter, r *http.Request) {
    ctx, span := tracer.Start(r.Context(), "payment.process",
        trace.WithAttributes(
            attribute.String("payment.method", "credit_card"),
        ),
    )
    defer span.End()

    // Parse request
    var req PaymentRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, "invalid request body")
        http.Error(w, "Invalid request", http.StatusBadRequest)
        return
    }

    span.SetAttributes(
        attribute.Float64("payment.amount", req.Amount),
        attribute.String("payment.currency", req.Currency),
        attribute.String("payment.order_id", req.OrderID),
    )

    // Validate
    _, validateSpan := tracer.Start(ctx, "payment.validate")
    if err := validatePayment(req); err != nil {
        validateSpan.RecordError(err)
        validateSpan.SetStatus(codes.Error, err.Error())
        validateSpan.End()

        span.RecordError(err)
        span.SetStatus(codes.Error, "validation failed")
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    validateSpan.End()

    // Process
    _, processSpan := tracer.Start(ctx, "payment.charge",
        trace.WithAttributes(
            attribute.String("payment.gateway", "stripe"),
        ),
    )
    chargeID, err := chargePaymentGateway(ctx, req)
    if err != nil {
        processSpan.RecordError(err)
        processSpan.SetStatus(codes.Error, "charge failed")
        processSpan.End()

        span.RecordError(err)
        span.SetStatus(codes.Error, "payment processing failed")
        http.Error(w, "Payment failed", http.StatusInternalServerError)
        return
    }
    processSpan.SetAttributes(attribute.String("payment.charge_id", chargeID))
    processSpan.End()

    span.SetAttributes(
        attribute.String("payment.charge_id", chargeID),
        attribute.Bool("payment.success", true),
    )

    json.NewEncoder(w).Encode(map[string]string{
        "charge_id": chargeID,
        "status": "success",
    })
}
```

### Database Instrumentation

```go
// tracing/database.go
package tracing

import (
    "context"
    "database/sql"
    "fmt"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/attribute"
    "go.opentelemetry.io/otel/codes"
    semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
    "go.opentelemetry.io/otel/trace"
)

// InstrumentedDB wraps sql.DB ด้วย tracing
type InstrumentedDB struct {
    db     *sql.DB
    tracer trace.Tracer
    dbName string
}

func NewInstrumentedDB(db *sql.DB, dbName string) *InstrumentedDB {
    return &InstrumentedDB{
        db:     db,
        tracer: otel.Tracer("database"),
        dbName: dbName,
    }
}

func (idb *InstrumentedDB) QueryContext(
    ctx context.Context,
    query string,
    args ...interface{},
) (*sql.Rows, error) {
    ctx, span := idb.tracer.Start(ctx, "db.query",
        trace.WithAttributes(
            semconv.DBSystemPostgreSQL,
            semconv.DBName(idb.dbName),
            semconv.DBStatement(sanitizeQuery(query)),
            attribute.String("db.operation", "SELECT"),
        ),
        trace.WithSpanKind(trace.SpanKindClient),
    )
    defer span.End()

    rows, err := idb.db.QueryContext(ctx, query, args...)
    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, fmt.Sprintf("query failed: %v", err))
        return nil, err
    }

    return rows, nil
}

func sanitizeQuery(query string) string {
    // ลบ sensitive data จาก query ก่อน log
    // TODO: implement proper sanitization
    if len(query) > 200 {
        return query[:200] + "..."
    }
    return query
}
```

---

## 75.4 Instrumentation ใน TypeScript/Node.js

### Setup OpenTelemetry

```typescript
// tracing/setup.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-node';
import { ParentBasedSampler, TraceIdRatioBasedSampler } from '@opentelemetry/sdk-trace-base';

const otlpExporter = new OTLPTraceExporter({
  url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://localhost:4317',
});

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME || 'node-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
    'deployment.environment': process.env.NODE_ENV || 'development',
    'k8s.pod.name': process.env.POD_NAME || 'unknown',
  }),

  spanProcessor: new BatchSpanProcessor(otlpExporter, {
    maxQueueSize: 2048,
    maxExportBatchSize: 512,
    scheduledDelayMillis: 5000,
    exportTimeoutMillis: 30000,
  }),

  sampler: new ParentBasedSampler({
    root: new TraceIdRatioBasedSampler(
      process.env.NODE_ENV === 'production' ? 0.1 : 1.0
    ),
  }),

  instrumentations: [
    getNodeAutoInstrumentations({
      // HTTP instrumentation
      '@opentelemetry/instrumentation-http': {
        ignoreIncomingRequestHook: (req) => {
          // ไม่ trace health check endpoints
          return req.url === '/health' || req.url === '/readiness';
        },
        requestHook: (span, request) => {
          span.setAttribute('http.request_id', request.headers['x-request-id'] as string);
        },
        responseHook: (span, response) => {
          span.setAttribute('http.response_size', response.headers['content-length'] as string);
        },
      },

      // Express instrumentation
      '@opentelemetry/instrumentation-express': {
        ignoreLayers: ['/health', '/metrics'],
      },

      // PostgreSQL instrumentation
      '@opentelemetry/instrumentation-pg': {
        dbStatementSerializer: (operation, queryConfig) => {
          return queryConfig?.text?.substring(0, 200);
        },
      },

      // Redis instrumentation
      '@opentelemetry/instrumentation-redis': {},
    }),
  ],
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown()
    .then(() => console.log('Tracing terminated'))
    .catch(console.error)
    .finally(() => process.exit(0));
});
```

### Custom Spans

```typescript
// tracing/custom-spans.ts
import { trace, context, SpanStatusCode, SpanKind } from '@opentelemetry/api';

const tracer = trace.getTracer('order-service', '1.0.0');

export async function processOrder(orderId: string, items: OrderItem[]): Promise<string> {
  // Span หลัก
  return tracer.startActiveSpan(
    'order.process',
    {
      attributes: {
        'order.id': orderId,
        'order.item_count': items.length,
        'order.total': items.reduce((sum, item) => sum + item.price * item.quantity, 0),
      },
    },
    async (span) => {
      try {
        // Validate inventory
        const inventoryOk = await tracer.startActiveSpan(
          'inventory.check',
          { kind: SpanKind.CLIENT },
          async (inventorySpan) => {
            inventorySpan.setAttribute('inventory.items_checked', items.length);
            const result = await checkInventory(items);
            inventorySpan.setAttribute('inventory.all_available', result.allAvailable);
            inventorySpan.end();
            return result.allAvailable;
          }
        );

        if (!inventoryOk) {
          span.setStatus({ code: SpanStatusCode.ERROR, message: 'Items out of stock' });
          throw new Error('Items out of stock');
        }

        // Reserve inventory
        const reservationId = await tracer.startActiveSpan(
          'inventory.reserve',
          async (reserveSpan) => {
            const id = await reserveInventory(items);
            reserveSpan.setAttribute('reservation.id', id);
            reserveSpan.end();
            return id;
          }
        );

        span.setAttribute('order.reservation_id', reservationId);

        // Create order record
        const order = await createOrderInDB(orderId, items, reservationId);

        span.addEvent('order.created', {
          'order.database_id': order.id,
        });

        span.setStatus({ code: SpanStatusCode.OK });
        return order.id;

      } catch (error) {
        span.recordException(error as Error);
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: (error as Error).message,
        });
        throw error;
      } finally {
        span.end();
      }
    }
  );
}
```

---

## 75.5 Jaeger Setup

### ติดตั้ง Jaeger ด้วย Helm

```bash
# ติดตั้ง Jaeger Operator
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm install jaeger-operator jaegertracing/jaeger-operator \
  --namespace observability \
  --create-namespace

# สร้าง Jaeger instance
kubectl apply -f - << 'EOF'
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production

  collector:
    replicas: 2
    resources:
      requests:
        memory: 256Mi
        cpu: 100m
      limits:
        memory: 512Mi
        cpu: 500m

  query:
    replicas: 2
    resources:
      requests:
        memory: 128Mi
        cpu: 50m

  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        index-prefix: jaeger
    esIndexCleaner:
      enabled: true
      numberOfDays: 7
      schedule: "55 23 * * *"

  ingress:
    enabled: true
    annotations:
      kubernetes.io/ingress.class: nginx
    hosts:
      - jaeger.mycompany.com
EOF
```

### Jaeger Query API

```python
# jaeger-query.py
import requests
from datetime import datetime, timedelta

class JaegerClient:
    def __init__(self, base_url: str):
        self.base_url = base_url

    def get_traces(
        self,
        service: str,
        operation: str = None,
        limit: int = 20,
        lookback: str = '1h',
        min_duration: str = None,
        tags: dict = None,
    ) -> list:
        """ดึง traces จาก Jaeger"""
        params = {
            'service': service,
            'limit': limit,
            'lookback': lookback,
        }

        if operation:
            params['operation'] = operation
        if min_duration:
            params['minDuration'] = min_duration
        if tags:
            for k, v in tags.items():
                params[f'tags'] = f'{k}={v}'

        response = requests.get(
            f'{self.base_url}/api/traces',
            params=params,
        )

        return response.json().get('data', [])

    def get_trace(self, trace_id: str) -> dict:
        """ดึง trace เดียว"""
        response = requests.get(f'{self.base_url}/api/traces/{trace_id}')
        return response.json()

    def get_services(self) -> list:
        """ดึงรายชื่อ services"""
        response = requests.get(f'{self.base_url}/api/services')
        return response.json().get('data', [])

    def find_slow_traces(
        self,
        service: str,
        threshold_ms: int = 1000,
        limit: int = 10,
    ) -> list:
        """หา traces ที่ช้าเกิน threshold"""
        return self.get_traces(
            service=service,
            min_duration=f'{threshold_ms}ms',
            limit=limit,
        )

    def analyze_trace(self, trace: dict) -> dict:
        """วิเคราะห์ trace หา bottleneck"""
        spans = trace.get('spans', [])

        if not spans:
            return {}

        # หา total duration
        root_span = next(
            (s for s in spans if not s.get('references')),
            spans[0]
        )
        total_duration = root_span['duration']  # microseconds

        # หา bottleneck spans
        bottlenecks = sorted(spans, key=lambda s: s['duration'], reverse=True)[:5]

        return {
            'trace_id': trace['traceID'],
            'total_duration_ms': total_duration / 1000,
            'span_count': len(spans),
            'services_involved': list(set(s['processID'] for s in spans)),
            'bottleneck_spans': [
                {
                    'operation': s['operationName'],
                    'service': s['processID'],
                    'duration_ms': s['duration'] / 1000,
                    'pct_of_total': s['duration'] / total_duration * 100,
                }
                for s in bottlenecks
            ],
        }


# ตัวอย่างการใช้งาน
client = JaegerClient('http://jaeger.mycompany.com')

# หา slow traces
slow_traces = client.find_slow_traces('payment-service', threshold_ms=2000)

for trace in slow_traces:
    analysis = client.analyze_trace(trace)
    print(f"Trace: {analysis['trace_id']}")
    print(f"Duration: {analysis['total_duration_ms']:.2f}ms")
    print(f"Top bottlenecks:")
    for bn in analysis['bottleneck_spans']:
        print(f"  - {bn['operation']}: {bn['duration_ms']:.2f}ms ({bn['pct_of_total']:.1f}%)")
```

---

## 75.6 Trace Propagation

### W3C Trace Context

```
HTTP Headers สำหรับ Trace Propagation:

traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              ^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^  ^^
              version  trace-id (128-bit hex)        parent-id (64-bit)  flags

tracestate: vendor1=value1,vendor2=value2
```

### Manual Propagation ใน Go

```go
// propagation/manual.go
package propagation

import (
    "context"
    "net/http"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
)

// Inject context เข้า HTTP request (outgoing)
func InjectHTTPContext(ctx context.Context, req *http.Request) {
    otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
}

// Extract context จาก HTTP request (incoming)
func ExtractHTTPContext(ctx context.Context, req *http.Request) context.Context {
    return otel.GetTextMapPropagator().Extract(ctx, propagation.HeaderCarrier(req.Header))
}

// HTTP Client ที่ propagate context
type TracedHTTPClient struct {
    client *http.Client
}

func (c *TracedHTTPClient) Do(ctx context.Context, req *http.Request) (*http.Response, error) {
    // Inject trace context ลงใน request headers
    InjectHTTPContext(ctx, req)

    return c.client.Do(req.WithContext(ctx))
}

// ตัวอย่าง: เรียก downstream service
func CallInventoryService(ctx context.Context, orderID string) (*InventoryResponse, error) {
    tracer := otel.Tracer("payment-service")
    ctx, span := tracer.Start(ctx, "inventory.check",
        trace.WithSpanKind(trace.SpanKindClient),
        trace.WithAttributes(
            attribute.String("rpc.system", "http"),
            attribute.String("rpc.service", "inventory-service"),
            attribute.String("rpc.method", "CheckInventory"),
        ),
    )
    defer span.End()

    req, _ := http.NewRequestWithContext(ctx,
        "GET",
        fmt.Sprintf("http://inventory-service/api/check?order_id=%s", orderID),
        nil,
    )

    // Context จะถูก inject อัตโนมัติเพราะใช้ otelhttp client
    client := &TracedHTTPClient{client: &http.Client{}}
    resp, err := client.Do(ctx, req)

    if err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return nil, err
    }

    span.SetAttributes(attribute.Int("http.status_code", resp.StatusCode))

    var result InventoryResponse
    json.NewDecoder(resp.Body).Decode(&result)
    return &result, nil
}
```

### Kafka Message Propagation

```go
// propagation/kafka.go
package propagation

import (
    "context"

    "github.com/Shopify/sarama"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
)

// KafkaHeaderCarrier ใช้ Kafka headers สำหรับ propagation
type KafkaHeaderCarrier []sarama.RecordHeader

func (c KafkaHeaderCarrier) Get(key string) string {
    for _, header := range c {
        if string(header.Key) == key {
            return string(header.Value)
        }
    }
    return ""
}

func (c *KafkaHeaderCarrier) Set(key, val string) {
    *c = append(*c, sarama.RecordHeader{
        Key:   []byte(key),
        Value: []byte(val),
    })
}

func (c KafkaHeaderCarrier) Keys() []string {
    keys := make([]string, len(c))
    for i, header := range c {
        keys[i] = string(header.Key)
    }
    return keys
}

// Inject trace context ลง Kafka message
func InjectKafkaContext(ctx context.Context, headers *[]sarama.RecordHeader) {
    carrier := KafkaHeaderCarrier(*headers)
    otel.GetTextMapPropagator().Inject(ctx, &carrier)
    *headers = []sarama.RecordHeader(carrier)
}

// Extract trace context จาก Kafka message
func ExtractKafkaContext(ctx context.Context, headers []sarama.RecordHeader) context.Context {
    carrier := KafkaHeaderCarrier(headers)
    return otel.GetTextMapPropagator().Extract(ctx, carrier)
}
```

---

## 75.7 CI/CD Pipeline Instrumentation

### Tracing GitHub Actions Pipeline

```yaml
# .github/workflows/traced-pipeline.yaml
name: CI/CD with Tracing

on:
  push:
    branches: [main]

env:
  OTEL_EXPORTER_OTLP_ENDPOINT: https://otel-collector.mycompany.com
  SERVICE_NAME: payment-service

jobs:
  pipeline:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start Pipeline Trace
        id: trace
        run: |
          # เริ่ม trace สำหรับ pipeline ทั้งหมด
          TRACE_ID=$(openssl rand -hex 16)
          SPAN_ID=$(openssl rand -hex 8)
          echo "TRACE_ID=$TRACE_ID" >> $GITHUB_ENV
          echo "SPAN_ID=$SPAN_ID" >> $GITHUB_ENV
          echo "PIPELINE_START=$(date +%s%N)" >> $GITHUB_ENV

          # ส่ง span เริ่มต้น
          curl -X POST "$OTEL_EXPORTER_OTLP_ENDPOINT/v1/traces" \
            -H "Content-Type: application/json" \
            -d "{
              \"resourceSpans\": [{
                \"resource\": {
                  \"attributes\": [
                    {\"key\": \"service.name\", \"value\": {\"stringValue\": \"$SERVICE_NAME\"}},
                    {\"key\": \"service.version\", \"value\": {\"stringValue\": \"${{ github.sha }}\"}},
                    {\"key\": \"deployment.environment\", \"value\": {\"stringValue\": \"ci\"}}
                  ]
                },
                \"scopeSpans\": [{
                  \"spans\": [{
                    \"traceId\": \"$TRACE_ID\",
                    \"spanId\": \"$SPAN_ID\",
                    \"name\": \"ci.pipeline\",
                    \"kind\": 1,
                    \"startTimeUnixNano\": \"$(date +%s%N)\",
                    \"attributes\": [
                      {\"key\": \"ci.workflow\", \"value\": {\"stringValue\": \"${{ github.workflow }}\"}},
                      {\"key\": \"ci.run_id\", \"value\": {\"stringValue\": \"${{ github.run_id }}\"}},
                      {\"key\": \"git.commit\", \"value\": {\"stringValue\": \"${{ github.sha }}\"}},
                      {\"key\": \"git.branch\", \"value\": {\"stringValue\": \"${{ github.ref_name }}\"}},
                      {\"key\": \"git.author\", \"value\": {\"stringValue\": \"${{ github.actor }}\"}}
                    ]
                  }]
                }]
              }]
            }"

      - name: Build
        id: build
        run: |
          BUILD_START=$(date +%s%N)
          make build
          BUILD_END=$(date +%s%N)

          # Instrument build span
          BUILD_SPAN_ID=$(openssl rand -hex 8)
          curl -X POST "$OTEL_EXPORTER_OTLP_ENDPOINT/v1/traces" \
            -H "Content-Type: application/json" \
            -d "{
              \"resourceSpans\": [{
                \"scopeSpans\": [{
                  \"spans\": [{
                    \"traceId\": \"$TRACE_ID\",
                    \"spanId\": \"$BUILD_SPAN_ID\",
                    \"parentSpanId\": \"$SPAN_ID\",
                    \"name\": \"ci.build\",
                    \"startTimeUnixNano\": \"$BUILD_START\",
                    \"endTimeUnixNano\": \"$BUILD_END\",
                    \"status\": {\"code\": 1}
                  }]
                }]
              }]
            }"
```

### Python Script สำหรับ Pipeline Tracing

```python
# pipeline-tracer.py
"""
Script สำหรับ trace CI/CD pipeline steps
ใช้ในทุก step ของ pipeline
"""
import os
import sys
import json
import time
import requests
from datetime import datetime

class PipelineTracer:
    def __init__(self, otlp_endpoint: str, service_name: str):
        self.otlp_endpoint = otlp_endpoint
        self.service_name = service_name
        self.trace_id = os.environ.get('TRACE_ID', self._generate_id(32))
        self.pipeline_span_id = os.environ.get('PIPELINE_SPAN_ID', self._generate_id(16))

    def _generate_id(self, length: int) -> str:
        import secrets
        return secrets.token_hex(length // 2)

    def start_step(self, step_name: str) -> str:
        """เริ่ม span สำหรับ step"""
        span_id = self._generate_id(16)
        start_time = time.time_ns()

        # บันทึก span state ลง env
        os.environ[f'SPAN_{step_name}_ID'] = span_id
        os.environ[f'SPAN_{step_name}_START'] = str(start_time)

        return span_id

    def end_step(
        self,
        step_name: str,
        success: bool = True,
        attributes: dict = None,
    ) -> None:
        """จบ span สำหรับ step"""
        span_id = os.environ.get(f'SPAN_{step_name}_ID')
        start_time = int(os.environ.get(f'SPAN_{step_name}_START', 0))
        end_time = time.time_ns()

        if not span_id:
            return

        span_attrs = [
            {"key": "ci.step", "value": {"stringValue": step_name}},
            {"key": "ci.success", "value": {"boolValue": success}},
        ]

        if attributes:
            for k, v in attributes.items():
                span_attrs.append({
                    "key": k,
                    "value": {"stringValue": str(v)},
                })

        payload = {
            "resourceSpans": [{
                "resource": {
                    "attributes": [
                        {"key": "service.name", "value": {"stringValue": self.service_name}},
                    ]
                },
                "scopeSpans": [{
                    "spans": [{
                        "traceId": self.trace_id,
                        "spanId": span_id,
                        "parentSpanId": self.pipeline_span_id,
                        "name": f"ci.{step_name}",
                        "kind": 1,
                        "startTimeUnixNano": str(start_time),
                        "endTimeUnixNano": str(end_time),
                        "attributes": span_attrs,
                        "status": {
                            "code": 1 if success else 2,  # OK or ERROR
                        },
                    }]
                }]
            }]
        }

        requests.post(
            f"{self.otlp_endpoint}/v1/traces",
            json=payload,
            timeout=10,
        )


# ตัวอย่างการใช้งาน
if __name__ == "__main__":
    tracer = PipelineTracer(
        otlp_endpoint=os.environ['OTEL_EXPORTER_OTLP_ENDPOINT'],
        service_name=os.environ['SERVICE_NAME'],
    )

    # เริ่ม step
    tracer.start_step('unit_tests')

    # รัน tests
    import subprocess
    result = subprocess.run(['make', 'test'], capture_output=True)
    success = result.returncode == 0

    # จบ step พร้อม attributes
    tracer.end_step('unit_tests', success=success, attributes={
        'tests.exit_code': result.returncode,
        'tests.output': result.stdout.decode()[:500],
    })

    sys.exit(0 if success else 1)
```

---

## 75.8 Performance Profiling

### Continuous Profiling ด้วย Pyroscope

```yaml
# pyroscope-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pyroscope
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pyroscope
  template:
    spec:
      containers:
        - name: pyroscope
          image: grafana/pyroscope:latest
          ports:
            - containerPort: 4040
          env:
            - name: PYROSCOPE_STORAGE_TYPE
              value: s3
            - name: AWS_S3_BUCKET
              value: mycompany-pyroscope
```

```go
// profiling/setup.go
package profiling

import (
    "os"

    "github.com/grafana/pyroscope-go"
)

func InitProfiling(serviceName string) error {
    return pyroscope.Start(pyroscope.Config{
        ApplicationName: serviceName,
        ServerAddress:   os.Getenv("PYROSCOPE_SERVER_ADDRESS"),
        Logger:          pyroscope.StandardLogger,

        // Profile types
        ProfileTypes: []pyroscope.ProfileType{
            pyroscope.ProfileCPU,
            pyroscope.ProfileAllocObjects,
            pyroscope.ProfileAllocSpace,
            pyroscope.ProfileInuseObjects,
            pyroscope.ProfileInuseSpace,
            pyroscope.ProfileGoroutines,
            pyroscope.ProfileMutexCount,
            pyroscope.ProfileMutexDuration,
        },

        // Tags
        Tags: map[string]string{
            "env":     os.Getenv("ENVIRONMENT"),
            "version": os.Getenv("APP_VERSION"),
            "pod":     os.Getenv("POD_NAME"),
        },
    })
}
```

---

## 75.9 แบบฝึกหัด

### แบบฝึกหัดที่ 1: ติดตั้ง Jaeger Locally

```bash
# รัน Jaeger ด้วย Docker Compose

cat > docker-compose-jaeger.yaml << 'EOF'
version: '3.8'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "6831:6831/udp"   # Jaeger Thrift UDP
      - "6832:6832/udp"   # Jaeger Thrift UDP (binary)
      - "5778:5778"        # Config server
      - "16686:16686"      # Jaeger UI
      - "4317:4317"        # OTLP gRPC
      - "4318:4318"        # OTLP HTTP
      - "14268:14268"      # Jaeger HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
      - JAEGER_DISABLED=false

  otel-demo-app:
    image: node:20-alpine
    working_dir: /app
    volumes:
      - ./demo-app:/app
    command: node server.js
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
      - SERVICE_NAME=demo-service
    depends_on:
      - jaeger
EOF

docker-compose -f docker-compose-jaeger.yaml up -d
echo "Jaeger UI: http://localhost:16686"
```

### แบบฝึกหัดที่ 2: Instrument Node.js App

```typescript
// exercises/instrument-app/server.ts
// สร้าง Express app และ instrument ด้วย OpenTelemetry

import './tracing';  // ต้อง import ก่อน
import express from 'express';
import axios from 'axios';

const app = express();
app.use(express.json());

// TODO: เพิ่ม tracing ให้กับ endpoints เหล่านี้

// 1. GET /users/:id - ดึงข้อมูล user
app.get('/users/:id', async (req, res) => {
  // TODO: เพิ่ม span สำหรับ endpoint นี้
  // TODO: เพิ่ม span สำหรับ database query
  const user = await db.findUser(req.params.id);
  res.json(user);
});

// 2. POST /orders - สร้าง order
app.post('/orders', async (req, res) => {
  // TODO: เพิ่ม spans สำหรับ:
  // - validate request
  // - check inventory (external service call)
  // - save order to database
  // - send confirmation email (external service call)
  res.json({ orderId: 'new-id' });
});

// 3. ดู traces ใน Jaeger UI
// http://localhost:16686
// หา service "demo-service"
// ดู trace สำหรับ /orders endpoint

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
```

### แบบฝึกหัดที่ 3: วิเคราะห์ Trace

```python
# exercises/analyze-traces.py
# ใช้ Jaeger API วิเคราะห์ performance ของ service

client = JaegerClient('http://localhost:16686')

# 1. หา services ทั้งหมด
services = client.get_services()
print("Services:", services)

# 2. หา slow traces (> 1 วินาที)
slow_traces = client.find_slow_traces('demo-service', threshold_ms=1000)
print(f"Found {len(slow_traces)} slow traces")

# 3. วิเคราะห์ trace แรก
if slow_traces:
    analysis = client.analyze_trace(slow_traces[0])
    print("\nTrace Analysis:")
    print(f"Total duration: {analysis['total_duration_ms']:.2f}ms")
    print("\nTop bottlenecks:")
    for bn in analysis['bottleneck_spans']:
        print(f"  {bn['operation']}: {bn['duration_ms']:.2f}ms ({bn['pct_of_total']:.1f}%)")

# TODO: สร้าง report สรุป:
# - Average trace duration
# - P95 trace duration
# - Most common bottleneck
# - Services involved
```

---

## สรุป

Distributed Tracing เป็นเครื่องมือสำคัญสำหรับ microservices:

1. **OpenTelemetry** คือ vendor-neutral standard ที่ควรใช้
2. **Jaeger/Zipkin** เป็น backends ที่นิยมสำหรับ tracing
3. **Trace Propagation** ทำให้ trace ผ่าน services ต่างๆ ได้
4. **CI/CD Instrumentation** ช่วย correlate deployment กับ issues
5. **Continuous Profiling** ช่วยหา performance bottlenecks

การ implement tracing ตั้งแต่ต้นดีกว่าทำทีหลัง เพราะ retrofit เข้าไปใน codebase ที่มีอยู่แล้วยาก

### ขั้นตอนถัดไป

- ศึกษา [Part 76: SRE Practices](./part-76-sre-practices.md)
- OpenTelemetry docs: https://opentelemetry.io/docs
- Jaeger docs: https://www.jaegertracing.io/docs

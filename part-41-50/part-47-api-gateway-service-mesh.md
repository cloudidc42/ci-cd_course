# Part 47: API Gateway & Service Mesh

## สารบัญ
1. [API Gateway คืออะไร?](#api-gateway-คืออะไร)
2. [Kong](#kong)
3. [Traefik](#traefik)
4. [Nginx Ingress](#nginx-ingress)
5. [Service Mesh คืออะไร?](#service-mesh-คืออะไร)
6. [Istio](#istio)
7. [Linkerd](#linkerd)
8. [Traffic Management](#traffic-management)
9. [Canary Deployments กับ Service Mesh](#canary-deployments-กับ-service-mesh)
10. [Observability](#observability)
11. [Exercises](#exercises)

---

## API Gateway คืออะไร?

API Gateway เป็น entry point สำหรับ client requests ทำหน้าที่เป็น reverse proxy ที่รวม cross-cutting concerns ไว้

```
┌────────────────────────────────────────────────────────────┐
│                      API Gateway                           │
│                                                            │
│  Auth → Rate Limit → Load Balance → Routing → Transform   │
│                                                            │
└─────────────────────────────────────────────────────────┬──┘
                                                          │
          ┌───────────────────────────────────────────────┘
          │
   ┌──────▼──────┐    ┌─────────────┐    ┌─────────────┐
   │ User Service│    │Order Service│    │Pay Service  │
   └─────────────┘    └─────────────┘    └─────────────┘
```

### Features ของ API Gateway

```
Authentication & Authorization
Rate Limiting
SSL Termination
Load Balancing
Request/Response Transformation
Caching
Logging & Monitoring
API Versioning
Circuit Breaking
```

---

## Kong

Kong เป็น cloud-native API Gateway ที่ได้รับความนิยมสูง

### ติดตั้ง Kong บน Kubernetes

```bash
# เพิ่ม Helm repo
helm repo add kong https://charts.konghq.com
helm repo update

# ติดตั้ง Kong
helm install kong kong/kong \
  --namespace kong \
  --create-namespace \
  --set ingressController.installCRDs=false
```

### Kong Ingress Configuration

```yaml
# Kong Ingress สำหรับ User Service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: user-service-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: kong
    # Rate limiting
    konghq.com/plugins: rate-limit-plugin,jwt-auth-plugin
spec:
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /api/v1/users
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 8080
      - path: /api/v1/orders
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 8080
```

### Kong Plugins

```yaml
# Rate Limiting Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit-plugin
  namespace: production
config:
  minute: 60
  hour: 1000
  policy: local
  limit_by: consumer
plugin: rate-limiting
---
# JWT Authentication Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: jwt-auth-plugin
  namespace: production
config:
  key_claim_name: iss
  claims_to_verify:
  - exp
  - nbf
plugin: jwt
---
# CORS Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: cors-plugin
  namespace: production
config:
  origins:
  - "https://mycompany.com"
  - "https://app.mycompany.com"
  methods:
  - GET
  - POST
  - PUT
  - DELETE
  - OPTIONS
  headers:
  - Authorization
  - Content-Type
  exposed_headers:
  - X-Request-Id
  credentials: true
  max_age: 3600
plugin: cors
---
# Request Transformer
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-transformer
  namespace: production
config:
  add:
    headers:
    - "X-Request-Source:api-gateway"
    - "X-Service-Version:v1"
  remove:
    headers:
    - "X-Internal-Token"
plugin: request-transformer
```

### Kong Consumer และ Credentials

```yaml
# KongConsumer
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: mobile-app
  namespace: production
  annotations:
    kubernetes.io/ingress.class: kong
username: mobile-app
credentials:
- mobile-jwt-secret
---
# JWT Secret
apiVersion: v1
kind: Secret
metadata:
  name: mobile-jwt-secret
  namespace: production
  labels:
    konghq.com/credential: jwt
type: Opaque
stringData:
  key: "mobile-app"
  algorithm: RS256
  rsa_public_key: |
    -----BEGIN PUBLIC KEY-----
    MIIBIjANBgkqhkiG9...
    -----END PUBLIC KEY-----
```

---

## Traefik

Traefik เป็น modern HTTP reverse proxy และ load balancer ที่ auto-discovers services

### ติดตั้ง Traefik

```bash
helm repo add traefik https://helm.traefik.io/traefik
helm repo update

helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --values traefik-values.yaml
```

```yaml
# traefik-values.yaml
additionalArguments:
- "--api.dashboard=true"
- "--log.level=INFO"
- "--accesslog=true"
- "--metrics.prometheus=true"

ports:
  web:
    port: 80
    redirectTo:
      port: websecure
  websecure:
    port: 443
    tls:
      enabled: true
      certResolver: letsencrypt

ingressRoute:
  dashboard:
    enabled: true
    entryPoints: ["websecure"]
    middlewares:
    - name: auth-middleware

certResolvers:
  letsencrypt:
    email: devops@mycompany.com
    storage: /data/acme.json
    httpChallenge:
      entryPoint: web
```

### Traefik IngressRoute

```yaml
# IngressRoute สำหรับ API
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: api-routes
  namespace: production
spec:
  entryPoints:
  - websecure
  routes:
  - match: Host(`api.mycompany.com`) && PathPrefix(`/api/v1/users`)
    kind: Rule
    services:
    - name: user-service
      port: 8080
      weight: 100
    middlewares:
    - name: jwt-auth
    - name: rate-limit
    - name: headers
  
  - match: Host(`api.mycompany.com`) && PathPrefix(`/api/v1/orders`)
    kind: Rule
    services:
    - name: order-service
      port: 8080
    middlewares:
    - name: jwt-auth
    - name: rate-limit
  
  tls:
    certResolver: letsencrypt
    domains:
    - main: api.mycompany.com
```

### Traefik Middlewares

```yaml
# JWT Authentication Middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: jwt-auth
  namespace: production
spec:
  forwardAuth:
    address: http://auth-service:8080/api/v1/verify
    trustForwardHeader: true
    authResponseHeaders:
    - X-User-Id
    - X-User-Role
---
# Rate Limiting Middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rate-limit
  namespace: production
spec:
  rateLimit:
    average: 100
    burst: 50
    period: 1m
    sourceCriterion:
      requestHeaderName: X-Real-IP
---
# Headers Middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: headers
  namespace: production
spec:
  headers:
    stsSeconds: 31536000
    stsIncludeSubdomains: true
    stsPreload: true
    contentTypeNosniff: true
    browserXssFilter: true
    customRequestHeaders:
      X-Gateway: "traefik"
    customResponseHeaders:
      X-Frame-Options: DENY
---
# Circuit Breaker Middleware
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: circuit-breaker
  namespace: production
spec:
  circuitBreaker:
    expression: ResponseCodeRatio(500, 600, 0, 600) > 0.30 || NetworkErrorRatio() > 0.10
```

---

## Nginx Ingress

Nginx Ingress Controller เป็นตัวเลือกยอดนิยมที่ใช้ Nginx เป็น reverse proxy

### ติดตั้ง Nginx Ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values nginx-values.yaml
```

### Nginx Ingress Configuration

```yaml
# Ingress กับ advanced configurations
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    
    # SSL
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-url: "https://auth.mycompany.com/api/v1/verify"
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.mycompany.com/login"
    nginx.ingress.kubernetes.io/auth-response-headers: "X-User-Id,X-User-Role"
    
    # CORS
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://mycompany.com"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    
    # Timeout
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    
    # Rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /api/v1/$2
    
    # Custom header
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Gateway "nginx" always;
      add_header X-Request-Id $request_id always;

spec:
  tls:
  - hosts:
    - api.mycompany.com
    secretName: api-tls
  
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /v1(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

## Service Mesh คืออะไร?

Service Mesh เป็น infrastructure layer สำหรับ service-to-service communication จัดการ traffic, security, และ observability ระหว่าง services

```
┌─────────────────────────────────────────────────────────┐
│                    Service Mesh Layer                    │
│                                                         │
│  ┌──────────────┐        ┌──────────────┐               │
│  │  Service A   │        │  Service B   │               │
│  │  ┌────────┐  │        │  ┌────────┐  │               │
│  │  │  App   │  │        │  │  App   │  │               │
│  │  └───┬────┘  │        │  └───┬────┘  │               │
│  │      │       │  mTLS  │      │       │               │
│  │  ┌───▼────┐  │◄──────►│  ┌───▼────┐  │               │
│  │  │ Sidecar│  │        │  │ Sidecar│  │               │
│  │  │ Proxy  │  │        │  │ Proxy  │  │               │
│  │  │(Envoy) │  │        │  │(Envoy) │  │               │
│  │  └────────┘  │        │  └────────┘  │               │
│  └──────────────┘        └──────────────┘               │
│                                                         │
│  Control Plane: Istiod/Linkerd Controller               │
└─────────────────────────────────────────────────────────┘
```

### Service Mesh Features

```
Traffic Management:
- Load balancing (Round Robin, Least Connection, etc.)
- Traffic splitting (canary, A/B testing)
- Circuit breaking
- Retry logic
- Timeout

Security:
- Mutual TLS (mTLS)
- Authorization policies
- Certificate rotation

Observability:
- Distributed tracing
- Metrics (latency, error rate, traffic)
- Access logging
```

---

## Istio

Istio เป็น service mesh ที่ครบครันและได้รับความนิยมมากที่สุด

### ติดตั้ง Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# ตรวจสอบ prerequisites
istioctl x precheck

# ติดตั้งด้วย profile
istioctl install --set profile=production -y

# ตรวจสอบ
kubectl get pods -n istio-system

# เปิด sidecar injection สำหรับ namespace
kubectl label namespace production istio-injection=enabled
```

### Istio Core Resources

```yaml
# VirtualService - traffic routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: production
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
---
# DestinationRule - policies per subset
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: production
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 1000
        http2MaxRequests: 1000
    outlierDetection:
      consecutiveErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```

### Istio Security

```yaml
# PeerAuthentication - mTLS mode
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # บังคับ mTLS ทุก connections
---
# AuthorizationPolicy - RBAC
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: order-service-policy
  namespace: production
spec:
  selector:
    matchLabels:
      app: order-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/web-frontend"
        - "cluster.local/ns/production/sa/api-gateway"
    to:
    - operation:
        methods: ["GET", "POST"]
        paths: ["/api/v1/orders*"]
  - from:
    - source:
        principals:
        - "cluster.local/ns/production/sa/admin-service"
    to:
    - operation:
        methods: ["*"]
```

---

## Linkerd

Linkerd เป็น lightweight service mesh ที่เน้น simplicity และ performance

### ติดตั้ง Linkerd

```bash
# ติดตั้ง Linkerd CLI
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
export PATH=$PATH:$HOME/.linkerd2/bin

# ตรวจสอบ cluster prerequisites
linkerd check --pre

# ติดตั้ง CRDs
linkerd install --crds | kubectl apply -f -

# ติดตั้ง Linkerd control plane
linkerd install | kubectl apply -f -

# ตรวจสอบ
linkerd check

# Inject namespace
kubectl annotate namespace production \
  linkerd.io/inject=enabled
```

### Linkerd Configuration

```yaml
# Service Profile - traffic policies per route
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: order-service.production.svc.cluster.local
  namespace: production
spec:
  routes:
  - name: GET /api/v1/orders
    condition:
      method: GET
      pathRegex: /api/v1/orders
    responseClasses:
    - condition:
        status:
          min: 500
          max: 599
      isFailure: true
    retryBudget:
      retryRatio: 0.2
      minRetriesPerSecond: 10
      ttl: 10s
    timeout: 3s
  
  - name: POST /api/v1/orders
    condition:
      method: POST
      pathRegex: /api/v1/orders
    timeout: 5s
    # ไม่ retry POST (non-idempotent)
```

---

## Traffic Management

### Weighted Traffic Splitting

```yaml
# Istio - Canary: 90% v1, 10% v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp-vs
spec:
  hosts:
  - webapp
  http:
  - route:
    - destination:
        host: webapp
        subset: v1
      weight: 90
    - destination:
        host: webapp
        subset: v2
      weight: 10
---
# Traefik - Canary
apiVersion: traefik.io/v1alpha1
kind: TraefikService
metadata:
  name: webapp-canary
spec:
  weighted:
    services:
    - name: webapp-v1
      port: 8080
      weight: 90
    - name: webapp-v2
      port: 8080
      weight: 10
---
# Nginx - Canary with annotations
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-canary
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 10% traffic to canary
spec:
  rules:
  - host: app.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-v2   # canary service
            port:
              number: 8080
```

### Circuit Breaking

```yaml
# Istio Circuit Breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 200
        maxRequestsPerConnection: 10
    outlierDetection:
      # Circuit opens เมื่อ:
      consecutiveGatewayErrors: 5    # 5 errors ติดต่อกัน
      interval: 30s                   # ภายใน 30 วินาที
      baseEjectionTime: 30s          # eject host เป็นเวลา 30 วินาที
      maxEjectionPercent: 50         # eject ได้สูงสุด 50% ของ hosts
      minHealthPercent: 50            # รักษา healthy hosts อย่างน้อย 50%
```

### Retry Policies

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-vs
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
    retries:
      attempts: 3
      perTryTimeout: 5s
      retryOn: "5xx,reset,connect-failure,retriable-4xx"
    timeout: 15s
    fault:
      # Inject fault สำหรับ testing (disable ใน production)
      # delay:
      #   percentage:
      #     value: 10
      #   fixedDelay: 5s
```

---

## Canary Deployments กับ Service Mesh

### Progressive Canary ด้วย Flagger

Flagger automates canary deployments ด้วย Istio/Linkerd

```bash
# ติดตั้ง Flagger
helm repo add flagger https://flagger.app
helm repo update

helm upgrade -i flagger flagger/flagger \
  --namespace istio-system \
  --set crd.create=true \
  --set meshProvider=istio \
  --set metricsServer=http://prometheus:9090
```

```yaml
# Flagger Canary resource
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: webapp
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  
  progressDeadlineSeconds: 120
  
  service:
    port: 80
    targetPort: 8080
    gateways:
    - public-gateway.istio-system.svc.cluster.local
    hosts:
    - app.mycompany.com
    trafficPolicy:
      tls:
        mode: ISTIO_MUTUAL
  
  analysis:
    # Progressive traffic increment
    interval: 1m
    threshold: 5       # ล้มเหลวสูงสุด 5 ครั้ง
    maxWeight: 50      # Traffic สูงสุด 50% ไปยัง canary
    stepWeight: 10     # เพิ่มครั้งละ 10%
    
    # Metrics สำหรับตัดสินใจ
    metrics:
    - name: request-success-rate
      # Istio จะ calculate success rate ให้
      thresholdRange:
        min: 99        # ต้อง >= 99%
      interval: 1m
    
    - name: request-duration
      thresholdRange:
        max: 500       # ต้อง <= 500ms (p99)
      interval: 1m
    
    # Webhook สำหรับ acceptance testing
    webhooks:
    - name: acceptance-test
      type: pre-rollout
      url: http://flagger-loadtester.test/
      timeout: 30s
      metadata:
        type: bash
        cmd: "curl -sd 'test' http://webapp-canary/health | grep ok"
    
    - name: load-test
      type: rollout
      url: http://flagger-loadtester.test/
      timeout: 5s
      metadata:
        type: cmd
        cmd: "hey -z 1m -q 10 -c 2 http://webapp-canary/"
```

### Manual Canary ด้วย Istio

```yaml
# Step 1: Deploy v2 with 0% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
      version: v2
  template:
    metadata:
      labels:
        app: webapp
        version: v2
    spec:
      containers:
      - name: webapp
        image: myapp:2.0.0
---
# Step 2: VirtualService ส่ง 0% ไปยัง v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: webapp
spec:
  http:
  - route:
    - destination:
        host: webapp
        subset: v1
      weight: 100
    - destination:
        host: webapp
        subset: v2
      weight: 0
```

```bash
# Progressive rollout script
#!/bin/bash
NAMESPACE=production
SERVICE=webapp

for WEIGHT in 10 25 50 75 100; do
  echo "Setting canary weight to ${WEIGHT}%..."
  
  # Update VirtualService
  kubectl patch virtualservice $SERVICE -n $NAMESPACE \
    --type merge \
    -p "{
      \"spec\": {
        \"http\": [{
          \"route\": [
            {\"destination\": {\"host\": \"$SERVICE\", \"subset\": \"v1\"}, \"weight\": $((100-WEIGHT))},
            {\"destination\": {\"host\": \"$SERVICE\", \"subset\": \"v2\"}, \"weight\": $WEIGHT}
          ]
        }]
      }
    }"
  
  echo "Waiting 5 minutes to observe metrics..."
  sleep 300
  
  # ตรวจสอบ error rate
  ERROR_RATE=$(kubectl exec -n istio-system deployment/prometheus \
    -- curl -s "http://localhost:9090/api/v1/query?query=sum(rate(istio_requests_total{destination_app=\"$SERVICE\",response_code=~\"5..\"}[5m]))/sum(rate(istio_requests_total{destination_app=\"$SERVICE\"}[5m]))" \
    | jq '.data.result[0].value[1]' -r)
  
  if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
    echo "❌ High error rate! Rolling back..."
    kubectl patch virtualservice $SERVICE -n $NAMESPACE \
      --type merge \
      -p '{"spec":{"http":[{"route":[{"destination":{"host":"'$SERVICE'","subset":"v1"},"weight":100}]}]}}'
    exit 1
  fi
  
  echo "✅ ${WEIGHT}% healthy, continuing..."
done

echo "✅ Canary rollout complete!"
```

---

## Observability

### Distributed Tracing กับ Jaeger

```yaml
# ติดตั้ง Jaeger ด้วย Istio
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    enableTracing: true
    defaultConfig:
      tracing:
        zipkin:
          address: jaeger-collector.monitoring:9411
        sampling: 100   # 100% ใน dev, 1% ใน prod
```

```go
// Go service ด้วย OpenTelemetry tracing
package main

import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/jaeger"
    "go.opentelemetry.io/otel/sdk/trace"
)

func initTracing() func() {
    exp, _ := jaeger.New(jaeger.WithCollectorEndpoint(
        jaeger.WithEndpoint("http://jaeger:14268/api/traces"),
    ))
    
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exp),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceNameKey.String("order-service"),
        )),
    )
    
    otel.SetTracerProvider(tp)
    
    return func() { tp.Shutdown(context.Background()) }
}

func handleOrder(w http.ResponseWriter, r *http.Request) {
    ctx, span := otel.Tracer("order-service").Start(r.Context(), "handleOrder")
    defer span.End()
    
    // Call downstream service
    req, _ := http.NewRequestWithContext(ctx, "GET", "http://payment-service/api/v1/charge", nil)
    // OpenTelemetry จะ inject trace headers โดยอัตโนมัติ
    client.Do(req)
}
```

### Prometheus Metrics กับ Service Mesh

```yaml
# ServiceMonitor สำหรับ Istio metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: istio-service-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: istiod
  namespaceSelector:
    matchNames:
    - istio-system
  endpoints:
  - port: http-monitoring
    interval: 15s
```

```yaml
# Grafana Dashboard สำหรับ Service Mesh
# Dashboard queries
panels:
- title: "Request Rate"
  query: |
    sum(rate(istio_requests_total{
      destination_service="order-service",
      namespace="production"
    }[5m]))
    
- title: "Error Rate"
  query: |
    sum(rate(istio_requests_total{
      destination_service="order-service",
      namespace="production",
      response_code=~"5.."
    }[5m])) /
    sum(rate(istio_requests_total{
      destination_service="order-service",
      namespace="production"
    }[5m]))
    
- title: "P99 Latency"
  query: |
    histogram_quantile(0.99, 
      sum(rate(istio_request_duration_milliseconds_bucket{
        destination_service="order-service",
        namespace="production"
      }[5m])) by (le)
    )
```

### Kiali Dashboard

```bash
# Kiali เป็น observability UI สำหรับ Istio
# ดู topology, traffic flow, health

# ติดตั้ง Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# เข้าถึง Kiali
istioctl dashboard kiali
```

---

## Exercises

### Exercise 1: ติดตั้ง Kong

```bash
#!/bin/bash
# exercise-1-kong.sh

# ติดตั้ง Kong
helm repo add kong https://charts.konghq.com
helm install kong kong/kong \
  --namespace kong \
  --create-namespace \
  --set ingressController.installCRDs=false

# รอ pods ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=kong \
  -n kong \
  --timeout=120s

# Deploy test service
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
      - name: httpbin
        image: kennethreitz/httpbin
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  namespace: default
spec:
  selector:
    app: httpbin
  ports:
  - port: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: httpbin
  namespace: default
  annotations:
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: httpbin
            port:
              number: 80
EOF

# ทดสอบ
KONG_IP=$(kubectl get svc -n kong kong-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl http://$KONG_IP/get
```

### Exercise 2: Istio Traffic Management

```bash
#!/bin/bash
# exercise-2-istio.sh

# ติดตั้ง Istio
istioctl install --set profile=demo -y

# Enable sidecar injection
kubectl label namespace default istio-injection=enabled

# Deploy Bookinfo sample
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml

# รอ pods ready
kubectl wait --for=condition=ready pod \
  -l app=reviews \
  --timeout=120s

# สร้าง Gateway
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/networking/bookinfo-gateway.yaml

# Traffic routing: ส่ง 80% ไป v1, 20% ไป v2
cat > routing.yaml << 'EOF'
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v2
      weight: 20
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
EOF

kubectl apply -f routing.yaml

echo "=== Istio traffic routing configured! ==="
echo "Test with: kubectl get svc istio-ingressgateway -n istio-system"
```

### Exercise 3: Canary with Flagger

```bash
#!/bin/bash
# exercise-3-flagger-canary.sh

# ติดตั้ง Flagger
helm upgrade -i flagger flagger/flagger \
  --namespace istio-system \
  --set crd.create=true \
  --set meshProvider=istio \
  --set metricsServer=http://prometheus:9090

# Deploy initial version
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.24
        ports:
        - containerPort: 80
EOF

# สร้าง Canary resource
kubectl apply -f - << 'EOF'
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: webapp
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  service:
    port: 80
  analysis:
    interval: 30s
    threshold: 3
    maxWeight: 50
    stepWeight: 10
    metrics:
    - name: request-success-rate
      thresholdRange:
        min: 99
      interval: 30s
EOF

# Trigger canary (update image)
kubectl set image deployment/webapp webapp=nginx:1.25 -n production

# Monitor progress
kubectl describe canary webapp -n production
flagger -namespace production get canary webapp
```

---

## สรุป

API Gateway และ Service Mesh ทำงานเสริมกัน:

**API Gateway** (Kong/Traefik/Nginx):
- จัดการ external traffic
- Authentication, Rate Limiting
- SSL Termination
- Routing ตาม path/host

**Service Mesh** (Istio/Linkerd):
- จัดการ internal service-to-service traffic
- mTLS encryption
- Traffic splitting, Canary deployments
- Distributed tracing, Observability

ใน CI/CD pipeline:
- Test อ separately ด้วย contract tests
- Deploy API Gateway configuration เป็น code
- Automate canary deployments ด้วย Flagger
- Monitor ด้วย metrics จาก service mesh

---

*ส่วนต่อไป: Part 48 - Cache Strategy ใน CI/CD*

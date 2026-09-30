# Part 56: Istio Service Mesh

## บทนำ

Istio คือ open-source service mesh ที่ช่วยจัดการ traffic ระหว่าง microservices อย่างมีประสิทธิภาพ โดยไม่ต้องแก้ไข application code Istio ทำงานโดยการ inject sidecar proxy (Envoy) เข้าไปในทุก pod เพื่อ intercept และ manage network traffic

บทนี้จะครอบคลุม:
- การติดตั้ง Istio
- Traffic Management
- Fault Injection
- Circuit Breaking
- mTLS
- Observability ด้วย Kiali และ Jaeger
- Workshop

---

## 1. ติดตั้ง Istio

### 1.1 ติดตั้ง istioctl

```bash
# Download และติดตั้ง Istio
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.20.0 sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# ตรวจสอบ prerequisites
istioctl x precheck

# ติดตั้ง Istio ด้วย default profile
istioctl install --set profile=default -y

# ตรวจสอบ
kubectl get pods -n istio-system
```

### 1.2 Istio Profiles

```bash
# Profiles ที่มี: minimal, default, demo, empty, preview, remote, external

# Demo profile — มีทุก features สำหรับ learning
istioctl install --set profile=demo -y

# Production profile — optimized
istioctl install \
    --set profile=default \
    --set values.global.proxy.resources.requests.cpu=100m \
    --set values.global.proxy.resources.requests.memory=128Mi \
    --set values.global.proxy.resources.limits.cpu=2000m \
    --set values.global.proxy.resources.limits.memory=1024Mi \
    -y

# ดู configuration ของ profile
istioctl profile dump default

# Compare profiles
istioctl profile diff default demo
```

### 1.3 Enable Sidecar Injection

```bash
# Enable injection สำหรับ namespace
kubectl label namespace production istio-injection=enabled

# Verify
kubectl get namespace production --show-labels

# Restart pods เพื่อให้ sidecar ถูก inject
kubectl rollout restart deployment -n production

# ตรวจสอบว่า sidecar ถูก inject
kubectl get pods -n production
# Pods ควรมี 2 containers (app + istio-proxy)

# ดู configuration ของ sidecar
kubectl describe pod <pod-name> -n production | grep -A5 "istio-proxy"
```

### 1.4 IstioOperator Configuration

```yaml
# istio/istio-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: istio-production
spec:
  profile: default
  
  # Components configuration
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: "500m"
            memory: "2Gi"
        hpaSpec:
          minReplicas: 2
          maxReplicas: 5
    
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true
        k8s:
          service:
            type: LoadBalancer
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
          hpaSpec:
            minReplicas: 2
            maxReplicas: 5
  
  # Global values
  values:
    global:
      # mTLS mode
      mtls:
        auto: true
      
      # Tracing
      tracer:
        zipkin:
          address: zipkin.istio-system:9411
      
      # Logging
      proxy:
        logLevel: warning
        accessLogFile: /dev/stdout
```

---

## 2. Traffic Management

### 2.1 VirtualService

VirtualService กำหนดกฎการ routing traffic

```yaml
# istio/virtual-services/frontend-vs.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: frontend
  namespace: production
spec:
  hosts:
    - frontend  # internal name
    - www.example.com  # external hostname
  
  gateways:
    - istio-ingressgateway
    - mesh  # internal mesh traffic
  
  http:
    # Route based on header (A/B Testing)
    - match:
        - headers:
            x-user-group:
              exact: "beta"
      route:
        - destination:
            host: frontend
            subset: v2  # Beta version
          weight: 100
    
    # Route based on path
    - match:
        - uri:
            prefix: "/api/v2"
      rewrite:
        uri: "/api"
      route:
        - destination:
            host: backend-api
            port:
              number: 3000
    
    # Default routing with Canary
    - route:
        - destination:
            host: frontend
            subset: v1  # Stable version
          weight: 90
        - destination:
            host: frontend
            subset: v2  # Canary version
          weight: 10
      
      # Timeout
      timeout: 10s
      
      # Retry policy
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: 5xx,retriable-4xx,connect-failure

---
# Destination Rule กำหนด subsets
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: frontend-dr
  namespace: production
spec:
  host: frontend
  
  # Traffic policy สำหรับทุก subsets
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 100
    
    loadBalancer:
      simple: LEAST_CONN  # ROUND_ROBIN, LEAST_CONN, RANDOM, PASSTHROUGH
    
    # Outlier detection (Circuit Breaker)
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        connectionPool:
          http:
            http2MaxRequests: 200
    
    - name: v2
      labels:
        version: v2
```

### 2.2 Canary Deployment ด้วย Istio

```yaml
# istio/canary/canary-deployment.yaml

# Version 1 (Stable)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v1
  namespace: production
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:v1.0.0
          ports:
            - containerPort: 8080

---
# Version 2 (Canary)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-v2
  namespace: production
  labels:
    app: myapp
    version: v2
spec:
  replicas: 1  # เริ่มด้วย replicas น้อย
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
        - name: myapp
          image: ghcr.io/myorg/myapp:v2.0.0
          ports:
            - containerPort: 8080

---
# Service (รวมทุก versions)
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp  # route ไปทุก versions
  ports:
    - port: 80
      targetPort: 8080

---
# VirtualService สำหรับ Canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-canary
  namespace: production
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10  # 10% traffic ไป canary

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-dr
  namespace: production
spec:
  host: myapp
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
```

### 2.3 Progressive Delivery Script

```bash
#!/bin/bash
# scripts/progressive-delivery.sh
# ค่อย ๆ เพิ่ม traffic ไปที่ canary

set -euo pipefail

NAMESPACE="${1:-production}"
APP_NAME="${2:-myapp}"
FINAL_WEIGHT="${3:-100}"
STEP="${4:-10}"
INTERVAL="${5:-120}"  # seconds between steps

CURRENT_WEIGHT=0

echo "Starting progressive delivery for ${APP_NAME}"
echo "Target: ${FINAL_WEIGHT}%, Step: ${STEP}%, Interval: ${INTERVAL}s"

# Function ตรวจสอบ error rate
check_error_rate() {
    local canary_weight=$1
    
    # Query Prometheus สำหรับ error rate
    ERROR_RATE=$(kubectl exec -n istio-system deployment/prometheus -- \
        curl -sS "http://localhost:9090/api/v1/query" \
        --data-urlencode 'query=sum(rate(istio_requests_total{destination_service_name="myapp",response_code=~"5.*",destination_version="v2"}[2m])) / sum(rate(istio_requests_total{destination_service_name="myapp",destination_version="v2"}[2m]))' \
        | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['data']['result'][0]['value'][1] if d['data']['result'] else '0')" 2>/dev/null || echo "0")
    
    echo "${ERROR_RATE}"
}

# Function update traffic weight
update_weight() {
    local v2_weight=$1
    local v1_weight=$((100 - v2_weight))
    
    kubectl patch virtualservice "${APP_NAME}-canary" \
        -n "${NAMESPACE}" \
        --type=json \
        -p "[
          {\"op\": \"replace\", \"path\": \"/spec/http/0/route/0/weight\", \"value\": ${v1_weight}},
          {\"op\": \"replace\", \"path\": \"/spec/http/0/route/1/weight\", \"value\": ${v2_weight}}
        ]"
    
    echo "Updated weights: v1=${v1_weight}%, v2=${v2_weight}%"
}

# ค่อย ๆ เพิ่ม traffic
while [ "${CURRENT_WEIGHT}" -lt "${FINAL_WEIGHT}" ]; do
    CURRENT_WEIGHT=$((CURRENT_WEIGHT + STEP))
    if [ "${CURRENT_WEIGHT}" -gt "${FINAL_WEIGHT}" ]; then
        CURRENT_WEIGHT="${FINAL_WEIGHT}"
    fi
    
    echo "Setting canary weight to ${CURRENT_WEIGHT}%..."
    update_weight "${CURRENT_WEIGHT}"
    
    if [ "${CURRENT_WEIGHT}" -lt "${FINAL_WEIGHT}" ]; then
        echo "Waiting ${INTERVAL}s before next step..."
        sleep "${INTERVAL}"
        
        # ตรวจสอบ error rate
        ERROR_RATE=$(check_error_rate "${CURRENT_WEIGHT}")
        echo "Current error rate: ${ERROR_RATE}"
        
        # ถ้า error rate สูงกว่า 1% ให้ rollback
        if (( $(echo "${ERROR_RATE} > 0.01" | bc -l 2>/dev/null || echo 0) )); then
            echo "❌ Error rate too high! Rolling back..."
            update_weight 0
            exit 1
        fi
    fi
done

echo "✅ Progressive delivery complete! All traffic on v2"
```

---

## 3. Fault Injection

### 3.1 HTTP Fault Injection

```yaml
# istio/fault-injection/delay.yaml
# ทดสอบว่า application รับมือกับ latency ได้หรือไม่

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-api-fault
  namespace: production
spec:
  hosts:
    - backend-api
  
  http:
    - fault:
        # Inject delay 5 วินาที สำหรับ 10% ของ requests
        delay:
          percentage:
            value: 10
          fixedDelay: 5s
      
      route:
        - destination:
            host: backend-api

---
# istio/fault-injection/abort.yaml
# ทดสอบว่า application รับมือกับ errors ได้หรือไม่

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-fault
  namespace: production
spec:
  hosts:
    - payment-service
  
  http:
    # Inject fault เฉพาะ user ที่เป็น test user
    - match:
        - headers:
            x-test-user:
              exact: "chaos-test"
      fault:
        abort:
          percentage:
            value: 50
          httpStatus: 503
      route:
        - destination:
            host: payment-service
    
    # Normal routing
    - route:
        - destination:
            host: payment-service
```

---

## 4. Circuit Breaking

### 4.1 Circuit Breaker Pattern

```yaml
# istio/circuit-breaking/backend-dr.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: backend-api-cb
  namespace: production
spec:
  host: backend-api
  
  trafficPolicy:
    # Connection Pool — จำกัด connections
    connectionPool:
      tcp:
        maxConnections: 50           # max concurrent connections
        connectTimeout: 3s            # connection timeout
        tcpKeepalive:
          time: 7200s
          interval: 75s
      http:
        http1MaxPendingRequests: 100  # max pending requests
        http2MaxRequests: 200         # max concurrent requests
        maxRequestsPerConnection: 10  # max requests per connection
        maxRetries: 3                 # max retries
        idleTimeout: 90s
        h2UpgradePolicy: UPGRADE
    
    # Outlier Detection — Circuit Breaker
    outlierDetection:
      # Errors threshold
      consecutiveGatewayErrors: 5       # consecutive 502/503/504
      consecutive5xxErrors: 5           # consecutive 5xx
      
      # Analysis window
      interval: 30s                     # how often to analyze
      
      # Ejection settings
      baseEjectionTime: 30s             # minimum ejection time
      maxEjectionPercent: 50            # max % of hosts to eject
      
      # เมื่อถูก eject ครั้งที่ n จะรอ baseEjectionTime * n
      minHealthPercent: 0
```

---

## 5. mTLS กับ Istio

### 5.1 เปิด mTLS

```yaml
# istio/security/mtls-strict.yaml
# เปิด mTLS แบบ Strict Mode สำหรับ namespace

apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # PERMISSIVE, STRICT, DISABLE

---
# ยกเว้นบาง ports จาก mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: myapp-mtls
  namespace: production
spec:
  selector:
    matchLabels:
      app: myapp
  mtls:
    mode: STRICT
  portLevelMtls:
    # Port 8080 ใช้ PERMISSIVE (รองรับทั้ง HTTP และ HTTPS)
    8080:
      mode: PERMISSIVE
    # Port 9090 (metrics) ไม่ใช้ mTLS
    9090:
      mode: DISABLE
```

### 5.2 Authorization Policy

```yaml
# istio/security/authz-policy.yaml

# Allow เฉพาะ frontend ที่จะ access backend-api
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: backend-api-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend-api
  
  action: ALLOW
  
  rules:
    # Allow จาก frontend
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST"]
            paths: ["/api/*"]
    
    # Allow Health checks
    - to:
        - operation:
            methods: ["GET"]
            paths: ["/health", "/ready"]
    
    # Allow จาก monitoring
    - from:
        - source:
            namespaces: ["monitoring"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/metrics"]

---
# Deny ทุก traffic ที่ไม่มี rule
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  {}  # Empty spec = deny all
  # action: DENY (default)
```

---

## 6. Istio Observability

### 6.1 ติดตั้ง Kiali

```bash
# ติดตั้ง Kiali
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/kiali.yaml

# ติดตั้ง Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/prometheus.yaml

# ติดตั้ง Grafana
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/grafana.yaml

# ติดตั้ง Jaeger
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/addons/jaeger.yaml

# เปิด Dashboard
istioctl dashboard kiali
istioctl dashboard grafana
istioctl dashboard jaeger
```

### 6.2 Configure Tracing

```yaml
# istio/telemetry/tracing.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
    - randomSamplingPercentage: 1.0  # 1% ใน production
      customTags:
        environment:
          literal:
            value: production
        version:
          header:
            name: x-app-version

---
# เพิ่ม sampling rate สำหรับ specific service
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: payment-tracing
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  tracing:
    - randomSamplingPercentage: 100.0  # 100% สำหรับ payment (สำคัญมาก)
```

### 6.3 Custom Metrics

```yaml
# istio/telemetry/custom-metrics.yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: production
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        # เพิ่ม custom dimensions
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            user_type:
              value: "request.headers['x-user-type'] | 'unknown'"
            region:
              value: "request.headers['x-region'] | 'unknown'"
        
        # เพิ่ม business metric
        - match:
            metric: REQUEST_DURATION
          tagOverrides:
            payment_method:
              value: "request.headers['x-payment-method'] | 'none'"
```

---

## 7. Istio Gateway

### 7.1 Ingress Gateway

```yaml
# istio/gateway/gateway.yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: main-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  
  servers:
    # HTTP — redirect to HTTPS
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*.example.com"
      tls:
        httpsRedirect: true
    
    # HTTPS
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "api.example.com"
        - "www.example.com"
      tls:
        mode: SIMPLE
        credentialName: example-com-cert  # Secret ที่มี TLS cert

---
# cert-manager สำหรับ auto-renew certificates
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-cert
  namespace: istio-system
spec:
  secretName: example-com-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.example.com
    - www.example.com

---
# VirtualService สำหรับ Gateway
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: main-vs
  namespace: production
spec:
  hosts:
    - "api.example.com"
    - "www.example.com"
  gateways:
    - istio-system/main-gateway
  
  http:
    - match:
        - uri:
            prefix: "/api"
          authority:
            exact: "api.example.com"
      route:
        - destination:
            host: backend-api
            port:
              number: 3000
    
    - match:
        - authority:
            exact: "www.example.com"
      route:
        - destination:
            host: frontend
            port:
              number: 80
```

---

## 8. Istio ใน CI/CD Pipeline

### 8.1 Deploy กับ Istio

```yaml
# .github/workflows/istio-deploy.yml
name: Deploy with Istio Traffic Management

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/DeployRole
          aws-region: ap-southeast-1

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig --name production --region ap-southeast-1

      - name: Deploy Canary
        run: |
          # Deploy new version (canary)
          kubectl set image deployment/myapp-v2 \
            myapp=ghcr.io/myorg/myapp:${{ github.sha }} \
            -n production
          
          kubectl rollout status deployment/myapp-v2 -n production

      - name: Start Canary Traffic
        run: |
          # ส่ง 5% traffic ไปที่ canary
          kubectl patch virtualservice myapp-canary \
            -n production \
            --type=json \
            -p '[
              {"op": "replace", "path": "/spec/http/0/route/0/weight", "value": 95},
              {"op": "replace", "path": "/spec/http/0/route/1/weight", "value": 5}
            ]'

      - name: Wait and Check Metrics
        run: |
          echo "Waiting 5 minutes for canary metrics..."
          sleep 300
          
          # Check error rate via Prometheus
          ERROR_RATE=$(kubectl exec -n istio-system deployment/prometheus -- \
            curl -sS "http://localhost:9090/api/v1/query" \
            --data-urlencode 'query=sum(rate(istio_requests_total{destination_version="v2",response_code=~"5.*"}[5m])) / sum(rate(istio_requests_total{destination_version="v2"}[5m]))' \
            | python3 -c "import json,sys; d=json.load(sys.stdin); r=d['data']['result']; print(r[0]['value'][1] if r else '0')")
          
          echo "Canary error rate: ${ERROR_RATE}"
          
          if (( $(echo "${ERROR_RATE} > 0.02" | bc -l 2>/dev/null || echo 0) )); then
            echo "❌ Error rate too high, rolling back"
            kubectl patch virtualservice myapp-canary \
              -n production \
              --type=json \
              -p '[
                {"op": "replace", "path": "/spec/http/0/route/0/weight", "value": 100},
                {"op": "replace", "path": "/spec/http/0/route/1/weight", "value": 0}
              ]'
            exit 1
          fi

      - name: Progressive Rollout
        run: |
          # ค่อย ๆ เพิ่ม canary traffic
          for weight in 10 25 50 75 100; do
            echo "Setting canary weight to ${weight}%"
            kubectl patch virtualservice myapp-canary \
              -n production \
              --type=json \
              -p "[
                {\"op\": \"replace\", \"path\": \"/spec/http/0/route/0/weight\", \"value\": $((100 - weight))},
                {\"op\": \"replace\", \"path\": \"/spec/http/0/route/1/weight\", \"value\": ${weight}}
              ]"
            
            if [ "${weight}" -lt 100 ]; then
              sleep 120  # รอ 2 นาที
            fi
          done
          
          echo "✅ Full rollout complete!"
```

---

## 9. Workshop: Istio Hands-on

### Lab 1: Traffic Management ด้วย Bookinfo Sample

```bash
#!/bin/bash
# workshop/lab1-bookinfo.sh

echo "=== Lab 1: Bookinfo App with Istio ==="

# Deploy Bookinfo sample application
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/platform/kube/bookinfo.yaml -n production

# Verify deployment
kubectl get pods -n production
kubectl get services -n production

# Apply Gateway
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/networking/bookinfo-gateway.yaml -n production

# Get gateway URL
INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway \
    -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')

echo "Access app at: http://${INGRESS_HOST}:${INGRESS_PORT}/productpage"

# Apply DestinationRules
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.20/samples/bookinfo/networking/destination-rule-all.yaml -n production

# Test 1: Route all traffic to reviews v1 (no stars)
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: production
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
EOF

echo "All traffic → reviews v1 (no stars)"
echo "Test: curl http://${INGRESS_HOST}:${INGRESS_PORT}/productpage"

# Test 2: User-based routing
cat << EOF | kubectl apply -f -
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
        subset: v2  # black stars
  - route:
    - destination:
        host: reviews
        subset: v1  # no stars
EOF

echo "jason → reviews v2 (black stars)"
echo "Others → reviews v1 (no stars)"
```

### Lab 2: Fault Injection Testing

```bash
#!/bin/bash
# workshop/lab2-fault-injection.sh

echo "=== Lab 2: Fault Injection ==="

# Inject delay สำหรับ jason
cat << EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: ratings
  namespace: production
spec:
  hosts:
  - ratings
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    fault:
      delay:
        percentage:
          value: 100.0
        fixedDelay: 7s  # 7 วินาที delay สำหรับ jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
EOF

echo "Injected 7s delay for jason"
echo "Login as jason and observe slow page load"

# ตรวจสอบว่า timeout ทำงาน
echo ""
echo "Testing timeout behavior:"
cat << EOF | kubectl apply -f -
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
    timeout: 0.5s  # 0.5 วินาที timeout
  - route:
    - destination:
        host: reviews
        subset: v1
EOF

echo "Set 0.5s timeout on reviews for jason"
echo "Expected: Unavailable error (เพราะ ratings delay 7s > 0.5s timeout)"
```

---

## 10. สรุปและ Best Practices

### Istio Checklist

```markdown
## Istio Best Practices

### Installation
- [ ] ใช้ IstioOperator configuration สำหรับ production
- [ ] กำหนด resource limits สำหรับ control plane
- [ ] ใช้ HA mode (replicas >= 2)
- [ ] Monitor control plane metrics

### Traffic Management
- [ ] กำหนด timeouts สำหรับทุก services
- [ ] Configure retry policies
- [ ] ใช้ circuit breakers
- [ ] Implement health checks

### Security
- [ ] เปิด mTLS ใน STRICT mode
- [ ] กำหนด AuthorizationPolicies
- [ ] ใช้ RBAC สำหรับ Istio resources
- [ ] Regular policy audits

### Observability
- [ ] Configure tracing sampling rate
- [ ] Set up Kiali สำหรับ visualization
- [ ] Configure Prometheus metrics
- [ ] Create dashboards สำหรับ SLOs

### CI/CD
- [ ] Automate canary deployments
- [ ] Monitor error rates during rollout
- [ ] Implement automated rollback
- [ ] Test fault injection regularly
```

---

## อ้างอิง

- [Istio Documentation](https://istio.io/latest/docs/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Kiali Documentation](https://kiali.io/docs/)
- [Envoy Proxy Documentation](https://www.envoyproxy.io/docs/)
- [Progressive Delivery with Istio](https://istio.io/latest/blog/2019/canary-upgrades/)

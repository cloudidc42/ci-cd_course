# Part 37: Container Security

## บทนำ

Container Security เป็นหัวข้อที่สำคัญมากขึ้นเรื่อยๆ เมื่อองค์กรนำ Kubernetes และ Docker มาใช้งานในระดับ production การ scan container images, ใช้ base images ที่ปลอดภัย, และตั้งค่า runtime security เป็นสิ่งที่ทุกทีม DevSecOps ต้องรู้ ในบทนี้เราจะเรียนรู้การสร้าง container security ที่แข็งแกร่งตั้งแต่ build time จนถึง runtime

## วัตถุประสงค์การเรียนรู้

- Scan container images ด้วย Trivy และ Grype
- เลือกและสร้าง minimal base images
- ใช้ distroless containers
- ตั้งค่า runtime security ด้วย Falco
- กำหนด Network Policies
- ใช้ Pod Security Standards

---

## 37.1 Container Image Scanning

### Trivy - Comprehensive Scanner

```bash
# ติดตั้ง Trivy
brew install aquasecurity/trivy/trivy

# หรือ Docker
docker pull aquasec/trivy

# Scan image
trivy image nginx:latest

# Scan และ output เป็น JSON
trivy image --format json --output trivy-report.json nginx:latest

# Scan เฉพาะ HIGH และ CRITICAL
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan พร้อม secret detection
trivy image --security-checks vuln,secret,config nginx:latest

# Scan Dockerfile
trivy config Dockerfile

# Scan Infrastructure as Code
trivy config ./terraform/

# Scan Kubernetes manifests
trivy config ./k8s/

# Exit code 1 ถ้ามี vulnerabilities
trivy image --exit-code 1 --severity CRITICAL nginx:latest
```

### Trivy Configuration

```yaml
# trivy.yaml
scan:
  security-checks:
    - vuln
    - secret
    - config
  
  severity:
    - CRITICAL
    - HIGH
  
  exit-code: 1

cache:
  no-cache: false
  cache-dir: ~/.cache/trivy

db:
  skip-update: false
  light: false

vulnerability:
  ignore-unfixed: true
  type:
    - os
    - library

secret:
  config: trivy-secret.yaml

report:
  format: table
  output: ""
```

### Trivy ใน Dockerfile

```dockerfile
# Dockerfile สำหรับ scan ระหว่าง build
FROM aquasec/trivy:latest AS scanner

# ขั้นตอน build จริง
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Security scan stage
FROM scanner AS security-check
COPY --from=builder /app /scan-target
RUN trivy filesystem --exit-code 1 --no-progress /scan-target

# Production image
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Grype - Alternative Scanner

```bash
# ติดตั้ง Grype
brew install anchore/grype/grype

# Scan image
grype nginx:latest

# Scan Docker image บน disk
grype docker:nginx:latest

# Output เป็น JSON
grype nginx:latest -o json > grype-report.json

# Fail ถ้า severity สูงกว่า threshold
grype nginx:latest --fail-on high

# Scan OCI image
grype oci:./my-image.tar
```

### Clair สำหรับ Private Registries

```yaml
# clair-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clair
  namespace: security
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clair
  template:
    metadata:
      labels:
        app: clair
    spec:
      containers:
        - name: clair
          image: quay.io/projectquay/clair:4.7.2
          ports:
            - containerPort: 6060
            - containerPort: 6061
          env:
            - name: CLAIR_CONF
              value: /clair/config.yaml
          volumeMounts:
            - name: clair-config
              mountPath: /clair
      volumes:
        - name: clair-config
          configMap:
            name: clair-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: clair-config
  namespace: security
data:
  config.yaml: |
    http_listen_addr: "0.0.0.0:6060"
    introspection_addr: "0.0.0.0:6061"
    log_level: "info"
    
    indexer:
      connstring: "host=postgres port=5432 dbname=clair user=clair password=secret sslmode=disable"
      scanlock_retry: 10
      layer_scan_concurrency: 5
      
    matcher:
      connstring: "host=postgres port=5432 dbname=clair user=clair password=secret sslmode=disable"
      
    updaters:
      config:
        rhel:
          ignore_unpatched: false
```

---

## 37.2 Secure Dockerfile Best Practices

### Multi-stage Build สำหรับ Python

```dockerfile
# ================================================================
# Dockerfile - Python Application (Secure)
# ================================================================

# Stage 1: Dependencies
FROM python:3.11-slim AS dependencies

WORKDIR /deps

# Copy only dependency files
COPY requirements.txt .

# Install deps พร้อม security flags
RUN pip install --no-cache-dir --require-hashes -r requirements.txt \
    && pip check  # ตรวจสอบ dependency conflicts

# Stage 2: Security scan ของ dependencies
FROM aquasec/trivy:latest AS dep-scanner
COPY --from=dependencies /usr/local/lib/python3.11 /scan-target/lib
RUN trivy filesystem \
    --exit-code 0 \
    --severity HIGH,CRITICAL \
    --no-progress \
    /scan-target

# Stage 3: Build application
FROM python:3.11-slim AS builder

WORKDIR /app

# Copy verified dependencies
COPY --from=dependencies /usr/local/lib/python3.11 /usr/local/lib/python3.11

COPY src/ ./src/

# Compile Python files
RUN python -m compileall src/

# Stage 4: Production image (Distroless)
FROM gcr.io/distroless/python3-debian12:nonroot

WORKDIR /app

# Copy compiled application
COPY --from=builder /app/src ./src
COPY --from=dependencies /usr/local/lib/python3.11 /usr/local/lib/python3.11

# Set non-root user (distroless ใช้ nonroot UID 65532 โดย default)

EXPOSE 8080

ENTRYPOINT ["python", "-m", "src.main"]
```

### Distroless สำหรับ Go

```dockerfile
# Dockerfile สำหรับ Go Application (Distroless)
FROM golang:1.21-alpine AS builder

# ติดตั้ง security tools
RUN apk add --no-cache git ca-certificates

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copy source
COPY . .

# Build แบบ static binary
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -ldflags='-w -s -extldflags "-static"' \
    -a \
    -o /go/bin/app \
    ./cmd/server

# Security scan binary
FROM aquasec/trivy:latest AS scanner
COPY --from=builder /go/bin/app /binary
RUN trivy filesystem --exit-code 0 /binary

# Production: Distroless (ไม่มี shell, ไม่มี package manager)
FROM gcr.io/distroless/static-debian12:nonroot

# Copy CA certificates สำหรับ HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary เท่านั้น
COPY --from=builder /go/bin/app /app

EXPOSE 8080

ENTRYPOINT ["/app"]
```

### Node.js Secure Dockerfile

```dockerfile
# Dockerfile สำหรับ Node.js (Secure)

# Build stage
FROM node:20-alpine AS builder

# สร้าง non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

WORKDIR /app

# Copy package files
COPY package.json package-lock.json* ./

# Install dependencies ด้วย exact versions
RUN npm ci \
    --only=production \
    --ignore-scripts \
    && npm cache clean --force

# Copy source
COPY --chown=nextjs:nodejs . .

# Build application
RUN npm run build

# Production stage
FROM node:20-alpine AS production

# Security: อัพเดต OS packages
RUN apk update && \
    apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# สร้าง user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

WORKDIR /app

# Copy เฉพาะ production artifacts
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./

# Security headers
ENV NODE_ENV=production
ENV NODE_OPTIONS="--max-old-space-size=512"

# ไม่รันด้วย root
USER nextjs

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD node healthcheck.js

EXPOSE 3000

# ใช้ dumb-init สำหรับ proper signal handling
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

---

## 37.3 Kubernetes Pod Security

### Pod Security Standards

```yaml
# namespace-security.yaml
# กำหนด Pod Security Standards สำหรับ namespace

apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce: ห้าม Pods ที่ไม่ผ่าน policy ใน namespace นี้
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    
    # Warn: แจ้งเตือนเมื่อ Pod ไม่ผ่าน policy
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    
    # Audit: บันทึก audit log เมื่อ Pod ไม่ผ่าน policy
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
```

### Secure Pod Deployment

```yaml
# secure-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        container.seccomp.security.alpha.kubernetes.io/order-service: runtime/default
    spec:
      # ไม่ใช้ service account โดยไม่จำเป็น
      serviceAccountName: order-service
      automountServiceAccountToken: false
      
      # Security context ระดับ Pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
        fsGroup: 65534
        seccompProfile:
          type: RuntimeDefault
        sysctls: []
      
      # ไม่อนุญาต privilege escalation
      hostPID: false
      hostIPC: false
      hostNetwork: false
      
      containers:
        - name: order-service
          image: registry.example.com/order-service:1.2.3
          
          # Security context ระดับ Container
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 65534
            capabilities:
              drop:
                - ALL
              add: []  # ไม่ add capabilities ใดเลย
            seccompProfile:
              type: RuntimeDefault
          
          # Resources limits (ป้องกัน resource exhaustion)
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          
          # Ports
          ports:
            - containerPort: 8080
              protocol: TCP
          
          # Volume mounts (read-only)
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-config
              mountPath: /app/config
              readOnly: true
          
          # Environment variables (จาก Secrets)
          env:
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: db-password
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: order-service-secrets
                  key: api-key
          
          # Liveness และ Readiness probes
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
      
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
            sizeLimit: 100Mi
        - name: app-config
          configMap:
            name: order-service-config
      
      # Anti-affinity สำหรับ HA
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - order-service
                topologyKey: kubernetes.io/hostname

---
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT_ID:role/order-service-role

---
# RBAC - Minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: order-service-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get"]
    resourceNames: ["order-service-config"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: order-service-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: order-service
    namespace: production
roleRef:
  kind: Role
  name: order-service-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 37.4 Network Policies

### Restrictive Network Policy

```yaml
# network-policies.yaml

# Default deny all ingress และ egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Allow ingress ไปยัง order-service จาก ingress controller เท่านั้น
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-order-service
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        # อนุญาตจาก ingress-nginx namespace
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
    
    # อนุญาต internal services
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080

---
# Allow egress จาก order-service ไปยัง database เท่านั้น
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-order-service-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Egress
  egress:
    # Database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: database
      ports:
        - protocol: TCP
          port: 5432
    
    # Redis Cache
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    
    # DNS (จำเป็นสำหรับ service discovery)
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    
    # External APIs (HTTPS only)
    - to: []
      ports:
        - protocol: TCP
          port: 443
```

---

## 37.5 Runtime Security ด้วย Falco

### Falco Installation

```bash
# Helm
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco-system \
  --create-namespace \
  --set falco.json_output=true \
  --set falco.log_stderr=false \
  --set falco.log_syslog=false
```

### Falco Custom Rules

```yaml
# falco-rules.yaml
# Custom security rules

- rule: Unexpected Process in Container
  desc: A process ran in a container that is not expected
  condition: >
    spawned_process and
    container and
    not proc.name in (node, python, java, sh, bash, ps, ls) and
    not proc.pname in (node, python, java, sh, bash)
  output: >
    Unexpected process in container
    (user=%user.name user_loginuid=%user.loginuid
    command=%proc.cmdline pid=%proc.pid
    container_id=%container.id image=%container.image.repository
    k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
  priority: WARNING
  tags: [process, container, security]

- rule: Write to Sensitive File
  desc: Process writes to sensitive files
  condition: >
    open_write and
    (fd.name startswith /etc or
     fd.name startswith /root or
     fd.name startswith /proc or
     fd.name in (/bin, /sbin, /usr/bin, /usr/sbin))
  output: >
    Sensitive file opened for writing
    (user=%user.name user_loginuid=%user.loginuid
    command=%proc.cmdline pid=%proc.pid
    file=%fd.name container=%container.id
    image=%container.image.repository)
  priority: ERROR
  tags: [filesystem, security]

- rule: Shell Spawned in Container
  desc: Shell was started in a container
  condition: >
    spawned_process and
    container and
    proc.name in (bash, sh, zsh, dash, ksh)
  output: >
    Shell spawned in container
    (user=%user.name user_loginuid=%user.loginuid
    command=%proc.cmdline pid=%proc.pid
    parent=%proc.pname
    container_id=%container.id image=%container.image.repository
    k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
  priority: NOTICE
  tags: [shell, container, security]

- rule: Outbound Connection to External IP
  desc: Container made connection to external IP (not internal)
  condition: >
    outbound and
    container and
    not (fd.sip in (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16))
  output: >
    Container made connection to external host
    (command=%proc.cmdline pid=%proc.pid
    connection=%fd.name container=%container.id
    image=%container.image.repository)
  priority: NOTICE
  tags: [network, container, security]

- rule: Crypto Mining Detected
  desc: Possible crypto mining activity
  condition: >
    spawned_process and
    (proc.name in (minerd, xmrig, cpuminer, bfgminer, ethminer) or
     proc.cmdline contains "stratum+tcp" or
     proc.cmdline contains "mining.pool")
  output: >
    Crypto mining detected!
    (command=%proc.cmdline pid=%proc.pid
    container=%container.id image=%container.image.repository)
  priority: CRITICAL
  tags: [cryptomining, security]

- rule: Privileged Container Started
  desc: Privileged container started
  condition: >
    container_started and
    container.privileged = true
  output: >
    Privileged container started
    (user=%user.name command=%proc.cmdline
    container_id=%container.id image=%container.image.repository
    k8s_ns=%k8s.ns.name k8s_pod=%k8s.pod.name)
  priority: WARNING
  tags: [container, security, privileged]
```

### Falco Alert Integration

```yaml
# falco-values.yaml สำหรับ Helm
falco:
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/falco_rules.local.yaml
    - /etc/falco/custom_rules.yaml
  
  json_output: true
  json_include_output_property: true
  
  outputs:
    rate: 1
    max_burst: 1000
  
  syslog_output:
    enabled: false
  
  file_output:
    enabled: false
  
  stdout_output:
    enabled: true
  
  webserver:
    enabled: true
    listen_port: 8765

# Falcosidekick - ส่ง alerts ไปยัง Slack/PagerDuty
falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
      minimumpriority: "warning"
      messageformat: |
        🚨 Falco Security Alert
        *Rule:* {{ .Rule }}
        *Priority:* {{ .Priority }}
        *Output:* {{ .Output }}
        *Time:* {{ .Time }}
    
    pagerduty:
      routingKey: "YOUR_PAGERDUTY_ROUTING_KEY"
      minimumpriority: "critical"
```

---

## 37.6 Image Signing ด้วย Cosign

```bash
# ติดตั้ง Cosign
brew install sigstore/tap/cosign

# สร้าง key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key registry.example.com/my-app:v1.0.0

# Verify signature
cosign verify --key cosign.pub registry.example.com/my-app:v1.0.0

# Sign ด้วย Keyless (OIDC)
COSIGN_EXPERIMENTAL=1 cosign sign registry.example.com/my-app:v1.0.0

# Verify Keyless signature
COSIGN_EXPERIMENTAL=1 cosign verify \
  --certificate-identity user@example.com \
  --certificate-oidc-issuer https://accounts.google.com \
  registry.example.com/my-app:v1.0.0
```

### Sigstore Policy Controller

```yaml
# cluster-image-policy.yaml
# กำหนด policy ว่า images ต้อง signed เท่านั้น

apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    - glob: "registry.example.com/**"
  authorities:
    - name: sigstore
      keyless:
        url: https://fulcio.sigstore.dev
        trustRootRef: sigstore
        identities:
          - issuer: https://accounts.google.com
            subject: ci-cd@example.com
```

---

## 37.7 Container Scanning ใน CI/CD

```yaml
# .github/workflows/container-security.yml
name: Container Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      security-events: write
      packages: write
      id-token: write  # สำหรับ keyless signing
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: |
            my-app:${{ github.sha }}
            my-app:latest
          load: true
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      # Trivy Scan
      - name: Trivy Vulnerability Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-app:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true
      
      - name: Upload Trivy SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif
      
      # Grype Scan
      - name: Grype Vulnerability Scan
        uses: anchore/scan-action@v3
        id: grype-scan
        with:
          image: my-app:${{ github.sha }}
          fail-build: true
          severity-cutoff: high
          output-format: sarif
      
      - name: Upload Grype SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: ${{ steps.grype-scan.outputs.sarif }}
      
      # Dockerfile Lint
      - name: Hadolint Dockerfile Scan
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          format: sarif
          output-file: hadolint-results.sarif
          no-fail: false
      
      - name: Upload Hadolint SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: hadolint-results.sarif
      
      # Push to registry (only on main)
      - name: Login to Registry
        if: github.ref == 'refs/heads/main'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Push Docker image
        if: github.ref == 'refs/heads/main'
        uses: docker/build-push-action@v5
        id: docker-push
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
      
      # Sign image ด้วย Cosign
      - name: Install Cosign
        if: github.ref == 'refs/heads/main'
        uses: sigstore/cosign-installer@main
      
      - name: Sign Docker image
        if: github.ref == 'refs/heads/main'
        env:
          DIGEST: ${{ steps.docker-push.outputs.digest }}
          TAGS: ghcr.io/${{ github.repository }}:${{ github.sha }}
        run: |
          cosign sign --yes \
            "${TAGS}@${DIGEST}"
      
      # Generate SBOM
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: my-app:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom
          path: sbom.spdx.json
```

---

## 37.8 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Secure Dockerfile

ปรับปรุง Dockerfile ต่อไปนี้ให้ปลอดภัยขึ้น:

```dockerfile
# Dockerfile ที่มีปัญหา - แก้ไขให้ถูกต้อง
FROM ubuntu:latest

# ← ปัญหาที่ 1: ใช้ latest tag
# ← ปัญหาที่ 2: Ubuntu มีขนาดใหญ่

RUN apt-get update && apt-get install -y nodejs npm
# ← ปัญหาที่ 3: ไม่ pin versions

COPY . /app
WORKDIR /app

RUN npm install
# ← ปัญหาที่ 4: ติดตั้ง dev dependencies ด้วย

EXPOSE 3000

CMD ["node", "server.js"]
# ← ปัญหาที่ 5: รันด้วย root
# ← ปัญหาที่ 6: ไม่มี HEALTHCHECK
```

### แบบฝึกหัดที่ 2: Network Policies

สร้าง Network Policies สำหรับ microservices:
- `frontend` ควรรับ traffic จาก ingress เท่านั้น
- `api-service` ควรรับ traffic จาก frontend เท่านั้น
- `database` ควรรับ connection จาก api-service เท่านั้น
- ทุก services ควร deny all เป็น default

### แบบฝึกหัดที่ 3: Falco Rules

เขียน Falco rules สำหรับตรวจจับ:
1. การรัน `kubectl exec` เข้าไปใน production pod
2. การ access secrets ที่ไม่ได้รับอนุญาต
3. การ create privileged container
4. การ change file permissions (`chmod 777`)

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Trivy และ Grype**: Scan container images สำหรับ vulnerabilities
- **Secure Dockerfiles**: Multi-stage builds, distroless images, non-root users
- **Pod Security Standards**: Kubernetes-native security policies
- **Network Policies**: Micro-segmentation สำหรับ container networking
- **Falco**: Runtime security monitoring
- **Cosign**: Image signing และ verification
- **CI Integration**: Automated container security scanning

บทต่อไป (Part 38) เราจะเรียนรู้เรื่อง **Dependency Management และ Security**

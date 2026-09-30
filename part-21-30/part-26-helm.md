# Part 26: Helm Charts สำหรับ Kubernetes

## สารบัญ
1. [Helm คืออะไร?](#helm-คืออะไร)
2. [ทำไมต้องใช้ Helm?](#ทำไมต้องใช้-helm)
3. [ติดตั้ง Helm](#ติดตั้ง-helm)
4. [โครงสร้าง Chart](#โครงสร้าง-chart)
5. [Templates และ values.yaml](#templates-และ-valuesyaml)
6. [คำสั่งพื้นฐาน Helm](#คำสั่งพื้นฐาน-helm)
7. [Built-in Objects](#built-in-objects)
8. [Template Functions](#template-functions)
9. [Helpers และ _helpers.tpl](#helpers-และ-_helperstpl)
10. [Chart Dependencies](#chart-dependencies)
11. [Helm Repositories และ Artifact Hub](#helm-repositories-และ-artifact-hub)
12. [Packaging และ Publishing Charts](#packaging-และ-publishing-charts)
13. [Helmfile](#helmfile)
14. [CI/CD Integration ด้วย GitHub Actions](#cicd-integration-ด้วย-github-actions)
15. [Best Practices](#best-practices)
16. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Helm คืออะไร?

**Helm** คือ Package Manager สำหรับ Kubernetes ที่ช่วยให้การ deploy และจัดการ applications บน Kubernetes เป็นเรื่องง่ายขึ้น Helm ใช้แนวคิดของ "Chart" ซึ่งเป็นชุด template ของ Kubernetes manifests ที่สามารถ configure ได้ผ่าน values

### ปัญหาที่ Helm แก้ไข

เมื่อ deploy application บน Kubernetes โดยไม่ใช้ Helm คุณต้องจัดการ manifest files หลายไฟล์:

```
my-app/
├── deployment.yaml
├── service.yaml
├── configmap.yaml
├── secret.yaml
├── ingress.yaml
├── hpa.yaml
└── serviceaccount.yaml
```

ปัญหาที่พบ:
- **การ versioning**: ยากที่จะติดตามว่า deploy version ไหนอยู่
- **การ configure สำหรับหลาย environment**: dev/staging/prod มีค่าต่างกัน ต้องแก้ไขหลายไฟล์
- **การ rollback**: ยากต้องรู้ว่าเวลา deploy ครั้งก่อนใช้ config อะไร
- **การแชร์**: ยากที่จะแชร์ configuration ระหว่างทีม
- **Dependencies**: application ต้องการ database หรือ cache ต้องติดตั้งแยก

Helm แก้ปัญหาเหล่านี้โดย:
- **Templating**: ใช้ Go templates ใน manifests
- **Versioning**: ติดตาม release history
- **Rollback**: ง่ายด้วยคำสั่งเดียว
- **Packaging**: bundle application เป็น Chart เดียว
- **Dependencies**: จัดการ sub-charts

### Helm Architecture

```
┌─────────────────────────────────────────────────┐
│                   Helm Client                   │
│  (CLI: helm install, upgrade, rollback, etc.)   │
└─────────────────────┬───────────────────────────┘
                      │ Kubernetes API
                      ▼
┌─────────────────────────────────────────────────┐
│              Kubernetes Cluster                 │
│                                                 │
│  ┌──────────────┐  ┌──────────────────────────┐ │
│  │   Secrets    │  │   Deployed Resources     │ │
│  │ (Release     │  │  - Deployments           │ │
│  │  History)    │  │  - Services              │ │
│  └──────────────┘  │  - ConfigMaps            │ │
│                    │  - etc.                  │ │
│                    └──────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

> **หมายเหตุ**: Helm v3 (ปัจจุบัน) ไม่มี Tiller server แล้ว (เคยมีใน Helm v2) ทำให้ปลอดภัยขึ้นมาก

---

## ทำไมต้องใช้ Helm?

### เปรียบเทียบการ deploy โดยไม่ใช้ vs ใช้ Helm

**ไม่ใช้ Helm:**
```bash
# ต้อง apply ทีละไฟล์
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress.yaml

# อยากเปลี่ยน replicas ต้องแก้ deployment.yaml แล้ว apply ใหม่
# อยากดู history ต้องดูจาก git history หรือจำเอง
# อยาก rollback ต้องหา manifest เก่ามา apply
```

**ใช้ Helm:**
```bash
# Install ครั้งเดียว
helm install my-app ./my-chart

# อยากเปลี่ยน replicas
helm upgrade my-app ./my-chart --set replicaCount=3

# ดู history
helm history my-app

# Rollback
helm rollback my-app 1
```

### ประโยชน์หลักของ Helm

1. **Reusability**: Chart เดียว deploy ได้หลาย environment
2. **Maintainability**: แก้ไขที่ template เดียวกระทบทุกที่
3. **Community Charts**: ใช้ Helm charts ที่คนอื่นสร้างแล้ว (nginx, postgresql, redis, etc.)
4. **Release Management**: Helm จัดการ lifecycle ของ release
5. **Testing**: Built-in chart testing

---

## ติดตั้ง Helm

### Linux

```bash
# วิธีที่ 1: Script อัตโนมัติ
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# วิธีที่ 2: ดาวน์โหลด binary โดยตรง
# ดู version ล่าสุดที่ https://github.com/helm/helm/releases
HELM_VERSION="v3.14.0"
wget https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz
tar -zxvf helm-${HELM_VERSION}-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm

# วิธีที่ 3: Snap
sudo snap install helm --classic

# วิธีที่ 4: Homebrew (สำหรับ macOS/Linux)
brew install helm
```

### macOS

```bash
brew install helm
```

### Windows

```powershell
# Chocolatey
choco install kubernetes-helm

# Scoop
scoop install helm

# Winget
winget install Helm.Helm
```

### ตรวจสอบการติดตั้ง

```bash
helm version
# output: version.BuildInfo{Version:"v3.14.0", ...}

helm help
```

### เพิ่ม Shell Completion

```bash
# Bash
echo 'source <(helm completion bash)' >> ~/.bashrc
source ~/.bashrc

# Zsh
echo 'source <(helm completion zsh)' >> ~/.zshrc
source ~/.zshrc

# Fish
helm completion fish > ~/.config/fish/completions/helm.fish
```

---

## โครงสร้าง Chart

### สร้าง Chart ใหม่

```bash
helm create my-app
```

### โครงสร้างที่ได้

```
my-app/
├── Chart.yaml          # ข้อมูล metadata ของ chart
├── values.yaml         # Default values สำหรับ templates
├── charts/             # Chart dependencies
├── templates/          # Template files
│   ├── NOTES.txt       # ข้อความแสดงหลัง install สำเร็จ
│   ├── _helpers.tpl    # Template helpers/partials
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   └── tests/
│       └── test-connection.yaml
└── .helmignore         # ไฟล์ที่ไม่ต้องการ package
```

### Chart.yaml

```yaml
# my-app/Chart.yaml
apiVersion: v2          # Helm v3 ใช้ v2 เสมอ
name: my-app            # ชื่อ chart
description: A Helm chart for my application
type: application       # application หรือ library

# Chart version (SemVer)
version: 0.1.0

# Application version ที่ chart นี้ deploy
appVersion: "1.16.0"

# ข้อมูลเพิ่มเติม (optional)
keywords:
  - web
  - nodejs
home: https://example.com
sources:
  - https://github.com/example/my-app
maintainers:
  - name: John Doe
    email: john@example.com
icon: https://example.com/icon.png

# Dependencies
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

### values.yaml

```yaml
# my-app/values.yaml

# จำนวน replicas
replicaCount: 1

# Image configuration
image:
  repository: nginx
  pullPolicy: IfNotPresent
  # ถ้าไม่กำหนด จะใช้ appVersion จาก Chart.yaml
  tag: ""

# Image pull secrets สำหรับ private registry
imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

# Service Account
serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

# Pod annotations และ labels
podAnnotations: {}
podLabels: {}

# Pod security context
podSecurityContext: {}

# Container security context
securityContext: {}

# Service configuration
service:
  type: ClusterIP
  port: 80

# Ingress configuration
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Resource limits
resources: {}
# resources:
#   limits:
#     cpu: 100m
#     memory: 128Mi
#   requests:
#     cpu: 100m
#     memory: 128Mi

# Liveness probe
livenessProbe:
  httpGet:
    path: /
    port: http

# Readiness probe
readinessProbe:
  httpGet:
    path: /
    port: http

# Horizontal Pod Autoscaler
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# Volume mounts
volumes: []
volumeMounts: []

# Node selector
nodeSelector: {}

# Tolerations
tolerations: []

# Affinity rules
affinity: {}

# Environment variables
env: []
# env:
#   - name: ENV_VAR
#     value: "value"

# ConfigMap data
config: {}

# Application-specific values
app:
  port: 8080
  logLevel: info
  database:
    host: localhost
    port: 5432
    name: mydb
```

---

## Templates และ values.yaml

### Deployment Template

```yaml
# my-app/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "my-app.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.app.port }}
              protocol: TCP
          env:
            - name: LOG_LEVEL
              value: {{ .Values.app.logLevel | quote }}
            - name: DB_HOST
              value: {{ .Values.app.database.host | quote }}
            - name: DB_PORT
              value: {{ .Values.app.database.port | quote }}
            - name: DB_NAME
              value: {{ .Values.app.database.name | quote }}
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Service Template

```yaml
# my-app/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
```

### Ingress Template

```yaml
# my-app/templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
{{- $fullName := include "my-app.fullname" . -}}
{{- $svcPort := .Values.service.port -}}
{{- if and .Values.ingress.className (not (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion)) }}
  {{- if not (hasKey .Values.ingress.annotations "kubernetes.io/ingress.class") }}
  {{- $_ := set .Values.ingress.annotations "kubernetes.io/ingress.class" .Values.ingress.className}}
  {{- end }}
{{- end }}
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1
{{- else if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
apiVersion: networking.k8s.io/v1beta1
{{- else -}}
apiVersion: extensions/v1beta1
{{- end }}
kind: Ingress
metadata:
  name: {{ $fullName }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion) }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- if and .pathType (semverCompare ">=1.18-0" $.Capabilities.KubeVersion.GitVersion) }}
            pathType: {{ .pathType }}
            {{- end }}
            backend:
              {{- if semverCompare ">=1.19-0" $.Capabilities.KubeVersion.GitVersion }}
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
              {{- else }}
              serviceName: {{ $fullName }}
              servicePort: {{ $svcPort }}
              {{- end }}
          {{- end }}
    {{- end }}
{{- end }}
```

### HPA Template

```yaml
# my-app/templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### NOTES.txt

```
# my-app/templates/NOTES.txt
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
  {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "my-app.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch its status by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "my-app.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "my-app.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "my-app.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```

---

## คำสั่งพื้นฐาน Helm

### helm install

```bash
# Install chart จาก local directory
helm install my-release ./my-app

# Install พร้อม custom values file
helm install my-release ./my-app -f custom-values.yaml

# Install พร้อม override values
helm install my-release ./my-app \
  --set replicaCount=3 \
  --set image.tag=v1.2.0

# Install ใน namespace เฉพาะ
helm install my-release ./my-app \
  --namespace production \
  --create-namespace

# Install จาก repository
helm install my-release bitnami/nginx

# Dry run (ไม่ deploy จริง แค่แสดง manifest)
helm install my-release ./my-app --dry-run

# Debug mode
helm install my-release ./my-app --dry-run --debug

# Install พร้อม wait จนกว่า pods จะ ready
helm install my-release ./my-app --wait --timeout 5m
```

### helm upgrade

```bash
# Upgrade release ที่มีอยู่
helm upgrade my-release ./my-app

# Upgrade พร้อม new values
helm upgrade my-release ./my-app \
  --set image.tag=v2.0.0

# Install ถ้ายังไม่มี, upgrade ถ้ามีอยู่แล้ว
helm upgrade --install my-release ./my-app

# Upgrade พร้อม reset values เป็น default
helm upgrade my-release ./my-app --reset-values

# Upgrade พร้อมเก็บ values เดิม + เพิ่ม values ใหม่
helm upgrade my-release ./my-app \
  --reuse-values \
  --set image.tag=v2.0.0
```

### helm rollback

```bash
# ดู history ก่อน rollback
helm history my-release

# REVISION  UPDATED                  STATUS    CHART         APP VERSION  DESCRIPTION
# 1         Mon Jan  1 10:00:00 2024 superseded my-app-0.1.0 1.0.0        Install complete
# 2         Mon Jan  1 11:00:00 2024 deployed   my-app-0.1.0 1.1.0        Upgrade complete

# Rollback ไปยัง revision ที่กำหนด
helm rollback my-release 1

# Rollback ไปยัง revision ก่อนหน้า
helm rollback my-release 0

# Rollback แล้ว wait
helm rollback my-release 1 --wait
```

### helm uninstall

```bash
# Uninstall release
helm uninstall my-release

# Uninstall แต่เก็บ history
helm uninstall my-release --keep-history

# Uninstall จาก namespace เฉพาะ
helm uninstall my-release --namespace production
```

### helm status และ การดูข้อมูล

```bash
# ดู status ของ release
helm status my-release

# แสดง manifest ที่ถูก deploy
helm get manifest my-release

# แสดง values ที่ใช้
helm get values my-release

# แสดง all values (รวม default)
helm get values my-release --all

# แสดงข้อมูลทั้งหมด
helm get all my-release

# List releases ทั้งหมด
helm list

# List ทุก namespace
helm list --all-namespaces

# List ที่ failed
helm list --failed
```

### helm template

```bash
# Render templates โดยไม่ deploy
helm template my-release ./my-app

# Render พร้อม custom values
helm template my-release ./my-app \
  -f production.yaml \
  --set image.tag=v1.0.0

# Render เฉพาะบาง template
helm template my-release ./my-app \
  --show-only templates/deployment.yaml
```

### helm lint

```bash
# ตรวจสอบ chart syntax
helm lint ./my-app

# Lint พร้อม values
helm lint ./my-app -f values-prod.yaml

# Strict mode
helm lint ./my-app --strict
```

### helm test

```bash
# รัน chart tests
helm test my-release

# รัน test พร้อม log output
helm test my-release --logs
```

---

## Built-in Objects

Helm มี built-in objects ที่สามารถใช้ใน templates ได้:

### Release Object

```yaml
# .Release.Name - ชื่อ release
name: {{ .Release.Name }}

# .Release.Namespace - namespace ที่ deploy
namespace: {{ .Release.Namespace }}

# .Release.IsUpgrade - true ถ้า operation นี้คือ upgrade
# .Release.IsInstall - true ถ้า operation นี้คือ install

# .Release.Revision - revision number ปัจจุบัน
revision: {{ .Release.Revision }}

# .Release.Service - service ที่ render template (ปกติคือ "Helm")
service: {{ .Release.Service }}
```

### Chart Object

```yaml
# .Chart.Name - ชื่อ chart
name: {{ .Chart.Name }}

# .Chart.Version - version ของ chart
version: {{ .Chart.Version }}

# .Chart.AppVersion - app version
appVersion: {{ .Chart.AppVersion }}

# .Chart.Description - description ของ chart
description: {{ .Chart.Description }}
```

### Values Object

```yaml
# .Values - ค่าจาก values.yaml และ -f flags
replicaCount: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

### Files Object

```yaml
# .Files - ใช้ access ไฟล์ใน chart package
{{- $config := .Files.Get "config/app.conf" }}
data:
  app.conf: |
    {{ $config | nindent 4 }}

# อ่านหลายไฟล์
{{- range $path, $bytes := .Files.Glob "config/*.conf" }}
  {{ $path }}: |
    {{ $.Files.Get $path | nindent 4 }}
{{- end }}

# Encode เป็น base64
data:
  secret.key: {{ .Files.Get "secrets/app.key" | b64enc }}
```

### Capabilities Object

```yaml
# .Capabilities.KubeVersion - Kubernetes version
{{- if semverCompare ">=1.19-0" .Capabilities.KubeVersion.GitVersion }}
apiVersion: networking.k8s.io/v1
{{- else }}
apiVersion: networking.k8s.io/v1beta1
{{- end }}

# .Capabilities.APIVersions - API versions ที่ cluster support
{{- if .Capabilities.APIVersions.Has "batch/v1/CronJob" }}
# สามารถใช้ CronJob ได้
{{- end }}

# .Capabilities.HelmVersion - Helm version
helmVersion: {{ .Capabilities.HelmVersion.Version }}
```

### Template Object

```yaml
# .Template.Name - ชื่อ template file ปัจจุบัน
# .Template.BasePath - base path ของ templates directory
```

---

## Template Functions

Helm ใช้ Go template engine และเพิ่ม Sprig library functions:

### String Functions

```yaml
# upper - แปลงเป็นตัวพิมพ์ใหญ่
name: {{ "hello" | upper }}          # HELLO

# lower - แปลงเป็นตัวพิมพ์เล็ก
name: {{ "HELLO" | lower }}          # hello

# title - Title Case
name: {{ "hello world" | title }}    # Hello World

# trim - ตัด whitespace
name: {{ "  hello  " | trim }}       # hello

# replace - แทนที่ string
name: {{ "hello world" | replace " " "-" }}  # hello-world

# contains - ตรวจสอบว่า string มี substring
{{- if contains "prod" .Values.environment }}
  # production config
{{- end }}

# hasPrefix / hasSuffix
{{- if hasPrefix "app-" .Release.Name }}
  # release ขึ้นต้นด้วย app-
{{- end }}

# quote - ห่อด้วย quotes
value: {{ .Values.name | quote }}

# indent / nindent
config: |
  {{ .Values.config | indent 2 }}

# trunc - ตัด string ให้สั้นลง
name: {{ .Release.Name | trunc 63 }}

# trimSuffix
name: {{ "hello-" | trimSuffix "-" }}  # hello

# splitList / join
{{- $parts := splitList "," "a,b,c" }}
{{- $parts | join "-" }}  # a-b-c
```

### Numeric Functions

```yaml
# add, sub, mul, div
replicas: {{ add .Values.replicaCount 1 }}
maxSurge: {{ mul .Values.replicaCount 2 }}

# max, min
resources:
  requests:
    cpu: {{ max .Values.cpu "100m" }}

# int / float64
port: {{ .Values.port | int }}
```

### Type Conversion

```yaml
# toString
value: {{ 42 | toString }}

# toJson / fromJson
config: {{ .Values.config | toJson }}

# toYaml / fromYaml
spec:
  {{- .Values.podSpec | toYaml | nindent 2 }}

# b64enc / b64dec (base64 encode/decode)
data:
  password: {{ .Values.password | b64enc }}

# sha256sum
checksum: {{ .Values.config | toYaml | sha256sum }}
```

### List Functions

```yaml
# len
count: {{ len .Values.hosts }}

# first / last
firstHost: {{ first .Values.hosts }}

# append
{{- $list := append .Values.hosts "new-host" }}

# has
{{- if has "production" .Values.environments }}
  # production environment
{{- end }}

# without
{{- $filtered := without .Values.tags "debug" }}

# uniq
{{- $unique := uniq .Values.tags }}

# concat
{{- $combined := concat .Values.list1 .Values.list2 }}
```

### Dict Functions

```yaml
# get
value: {{ get .Values.config "key" }}

# set
{{- $_ := set .Values "newKey" "newValue" }}

# hasKey
{{- if hasKey .Values.labels "env" }}
  env: {{ .Values.labels.env }}
{{- end }}

# merge / mergeOverwrite
{{- $merged := merge .Values.defaults .Values.overrides }}

# keys
{{- range keys .Values.config }}
  key: {{ . }}
{{- end }}

# omit / pick
{{- $filtered := omit .Values.labels "debug" "test" }}

# dict
{{- $map := dict "key1" "val1" "key2" "val2" }}
```

### Flow Control

```yaml
# if / else / end
{{- if .Values.ingress.enabled }}
# ingress config
{{- else }}
# no ingress
{{- end }}

# with (เปลี่ยน scope)
{{- with .Values.resources }}
resources:
  limits:
    cpu: {{ .limits.cpu }}
    memory: {{ .limits.memory }}
{{- end }}

# range (loop)
{{- range .Values.hosts }}
- host: {{ . }}
{{- end }}

# range กับ key-value
{{- range $key, $value := .Values.labels }}
{{ $key }}: {{ $value }}
{{- end }}

# range กับ index
{{- range $index, $host := .Values.hosts }}
host-{{ $index }}: {{ $host }}
{{- end }}

# define / template (reusable templates)
{{- define "my-app.configBlock" }}
config:
  key: value
{{- end }}

# ใช้ template ที่ define ไว้
{{- template "my-app.configBlock" . }}
# หรือใช้ include (แนะนำ เพราะ pipe ได้)
{{- include "my-app.configBlock" . | nindent 2 }}
```

---

## Helpers และ _helpers.tpl

```yaml
# my-app/templates/_helpers.tpl

{{/*
Expand the name of the chart.
*/}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this.
If release name contains chart name it will be used as a full name.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
สร้าง environment variables จาก secret
*/}}
{{- define "my-app.envFromSecret" -}}
{{- range $key, $value := .Values.secrets }}
- name: {{ $key }}
  valueFrom:
    secretKeyRef:
      name: {{ include "my-app.fullname" $ }}-secrets
      key: {{ $key }}
{{- end }}
{{- end }}

{{/*
สร้าง resource limits
*/}}
{{- define "my-app.resources" -}}
{{- if .Values.resources }}
resources:
  {{- toYaml .Values.resources | nindent 2 }}
{{- end }}
{{- end }}

{{/*
Image name with tag
*/}}
{{- define "my-app.image" -}}
{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}
{{- end }}

{{/*
Generate checksum annotation สำหรับ force pod restart เมื่อ config เปลี่ยน
*/}}
{{- define "my-app.configChecksum" -}}
checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
{{- end }}
```

---

## Chart Dependencies

### กำหนด Dependencies ใน Chart.yaml

```yaml
# my-app/Chart.yaml
dependencies:
  # PostgreSQL database
  - name: postgresql
    version: "12.5.6"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
    tags:
      - database

  # Redis cache
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
    tags:
      - cache

  # แบบ alias (เมื่อต้องการ postgresql สองตัว)
  - name: postgresql
    version: "12.5.6"
    repository: "https://charts.bitnami.com/bitnami"
    alias: postgresql-secondary
    condition: postgresql-secondary.enabled
```

### values.yaml สำหรับ Dependencies

```yaml
# my-app/values.yaml
postgresql:
  enabled: true
  auth:
    username: myapp
    password: "secretpassword"
    database: myappdb
  primary:
    persistence:
      enabled: true
      size: 10Gi

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      enabled: false

postgresql-secondary:
  enabled: false
```

### ดาวน์โหลด Dependencies

```bash
# ดาวน์โหลด dependencies ลง charts/ directory
helm dependency update ./my-app

# หรือ
helm dep update ./my-app

# ตรวจสอบ dependencies
helm dependency list ./my-app

# Build dependencies (สำหรับ offline)
helm dependency build ./my-app
```

### Local Chart Dependencies

```yaml
# my-app/Chart.yaml
dependencies:
  - name: common-library
    version: "0.1.0"
    repository: "file://../common-library"
```

```
my-project/
├── my-app/          # main chart
│   ├── Chart.yaml
│   └── templates/
└── common-library/  # local dependency
    ├── Chart.yaml
    └── templates/
```

---

## Helm Repositories และ Artifact Hub

### เพิ่ม Repository

```bash
# เพิ่ม Bitnami repository (ยอดนิยม)
helm repo add bitnami https://charts.bitnami.com/bitnami

# เพิ่ม Ingress NGINX
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# เพิ่ม cert-manager
helm repo add cert-manager https://charts.jetstack.io

# เพิ่ม Prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# เพิ่ม Grafana
helm repo add grafana https://grafana.github.io/helm-charts
```

### จัดการ Repository

```bash
# List repositories ที่เพิ่มแล้ว
helm repo list

# Update repositories (ดึง metadata ใหม่)
helm repo update

# ค้นหา chart
helm search repo nginx

# ค้นหาใน Artifact Hub (hub.helm.sh)
helm search hub nginx

# ดูทุก version
helm search repo bitnami/nginx --versions

# ลบ repository
helm repo remove bitnami
```

### ค้นหา Chart ใน Artifact Hub

```bash
# ค้นหา
helm search hub postgresql

# ดูข้อมูล chart
helm show chart bitnami/postgresql
helm show values bitnami/postgresql
helm show readme bitnami/postgresql
helm show all bitnami/postgresql

# ดู available versions
helm search repo bitnami/postgresql --versions | head -10
```

### ติดตั้ง Chart จาก Repository

```bash
# Install nginx ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2

# Install cert-manager
helm install cert-manager cert-manager/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml
```

---

## Packaging และ Publishing Charts

### Package Chart

```bash
# Package chart เป็น .tgz file
helm package ./my-app

# Output: my-app-0.1.0.tgz

# Package พร้อม update dependencies
helm package ./my-app --dependency-update

# Package ไปยัง directory เฉพาะ
helm package ./my-app --destination ./packages
```

### สร้าง Helm Repository ใน GitHub Pages

```bash
# สร้าง repository structure
mkdir helm-charts
cd helm-charts
git init
git checkout -b gh-pages

# Package chart
helm package ../my-app -d .

# สร้าง index.yaml
helm repo index . --url https://username.github.io/helm-charts

# Commit และ push
git add .
git commit -m "Add my-app chart"
git push origin gh-pages
```

### GitHub Actions สำหรับ Publish Chart

```yaml
# .github/workflows/release-charts.yaml
name: Release Charts

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Install Helm
        uses: azure/setup-helm@v3

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.6.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

### Publish ไปยัง OCI Registry

```bash
# Login ไปยัง registry
helm registry login registry.example.com \
  --username myuser \
  --password mypassword

# Push chart ไปยัง OCI registry
helm push my-app-0.1.0.tgz oci://registry.example.com/charts

# Pull chart
helm pull oci://registry.example.com/charts/my-app --version 0.1.0

# Install จาก OCI registry
helm install my-release oci://registry.example.com/charts/my-app --version 0.1.0

# Logout
helm registry logout registry.example.com
```

### ใช้ GitHub Container Registry (GHCR) เป็น Helm Repository

```bash
# Login ไปยัง GHCR
echo $GITHUB_TOKEN | helm registry login ghcr.io -u USERNAME --password-stdin

# Push chart
helm push my-app-0.1.0.tgz oci://ghcr.io/USERNAME/helm-charts

# Install
helm install my-release oci://ghcr.io/USERNAME/helm-charts/my-app --version 0.1.0
```

---

## Helmfile

Helmfile เป็น declarative spec สำหรับ deploy Helm charts หลายตัวพร้อมกัน

### ติดตั้ง Helmfile

```bash
# Linux
wget https://github.com/helmfile/helmfile/releases/download/v0.162.0/helmfile_linux_amd64.tar.gz
tar -xf helmfile_linux_amd64.tar.gz
sudo mv helmfile /usr/local/bin/

# macOS
brew install helmfile

# ติดตั้ง helm-diff plugin (ใช้กับ helmfile diff)
helm plugin install https://github.com/databus23/helm-diff
```

### helmfile.yaml พื้นฐาน

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
  - name: cert-manager
    url: https://charts.jetstack.io
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts

releases:
  # Ingress NGINX
  - name: ingress-nginx
    namespace: ingress-nginx
    createNamespace: true
    chart: ingress-nginx/ingress-nginx
    version: "4.9.0"
    values:
      - controller:
          replicaCount: 2
          service:
            type: LoadBalancer

  # cert-manager
  - name: cert-manager
    namespace: cert-manager
    createNamespace: true
    chart: cert-manager/cert-manager
    version: "1.14.0"
    set:
      - name: installCRDs
        value: true

  # PostgreSQL
  - name: postgresql
    namespace: database
    createNamespace: true
    chart: bitnami/postgresql
    version: "12.x.x"
    values:
      - ./values/postgresql.yaml
    secrets:
      - ./secrets/postgresql-secrets.yaml

  # My Application
  - name: my-app
    namespace: production
    createNamespace: true
    chart: ./charts/my-app
    version: "0.1.0"
    values:
      - ./values/my-app-prod.yaml
    needs:
      - database/postgresql
      - ingress-nginx/ingress-nginx
```

### helmfile กับ Multiple Environments

```yaml
# helmfile.yaml
environments:
  development:
    values:
      - env/development.yaml
  staging:
    values:
      - env/staging.yaml
  production:
    values:
      - env/production.yaml
    secrets:
      - env/production-secrets.yaml

---
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami

releases:
  - name: my-app
    chart: ./charts/my-app
    namespace: {{ .Environment.Name }}
    values:
      - replicaCount: {{ .Values.replicaCount }}
      - image:
          tag: {{ .Values.imageTag }}
      - ingress:
          enabled: {{ .Values.ingressEnabled }}
          hosts:
            - host: {{ .Values.appDomain }}
              paths:
                - path: /
```

```yaml
# env/development.yaml
replicaCount: 1
imageTag: "latest"
ingressEnabled: true
appDomain: "dev.example.com"

# env/production.yaml
replicaCount: 3
imageTag: "v1.2.0"
ingressEnabled: true
appDomain: "example.com"
```

### คำสั่ง Helmfile

```bash
# Deploy ทุก releases
helmfile apply

# Deploy เฉพาะ environment
helmfile apply --environment production

# ดู diff ก่อน deploy
helmfile diff

# Sync (เหมือน apply แต่ force)
helmfile sync

# Destroy ทุก releases
helmfile destroy

# List releases
helmfile list

# Template (render manifests)
helmfile template

# Deploy เฉพาะ release
helmfile apply --selector name=my-app

# Deploy เฉพาะ namespace
helmfile apply --selector namespace=production
```

---

## CI/CD Integration ด้วย GitHub Actions

### Workflow สำหรับ Deploy ด้วย Helm

```yaml
# .github/workflows/deploy.yaml
name: Deploy to Kubernetes

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    permissions:
      contents: read
      packages: write
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy with Helm
    runs-on: ubuntu-latest
    needs: build-and-push
    environment: ${{ github.event.inputs.environment || 'staging' }}
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.14.0'

      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Verify cluster connection
        run: kubectl cluster-info

      - name: Add Helm repositories
        run: |
          helm repo add bitnami https://charts.bitnami.com/bitnami
          helm repo update

      - name: Deploy with Helm
        run: |
          ENVIRONMENT=${{ github.event.inputs.environment || 'staging' }}
          IMAGE_TAG=${{ needs.build-and-push.outputs.image-tag }}
          
          helm upgrade --install my-app ./charts/my-app \
            --namespace ${ENVIRONMENT} \
            --create-namespace \
            -f ./charts/my-app/values-${ENVIRONMENT}.yaml \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${IMAGE_TAG} \
            --set deployment.revision=${{ github.sha }} \
            --wait \
            --timeout 10m \
            --atomic

      - name: Verify deployment
        run: |
          ENVIRONMENT=${{ github.event.inputs.environment || 'staging' }}
          kubectl rollout status deployment/my-app -n ${ENVIRONMENT}
          kubectl get pods -n ${ENVIRONMENT}

      - name: Run smoke tests
        run: |
          ENVIRONMENT=${{ github.event.inputs.environment || 'staging' }}
          helm test my-app -n ${ENVIRONMENT} --logs

  rollback:
    name: Rollback on Failure
    runs-on: ubuntu-latest
    needs: deploy
    if: failure()
    
    steps:
      - name: Configure kubectl
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Set up Helm
        uses: azure/setup-helm@v3

      - name: Rollback deployment
        run: |
          ENVIRONMENT=${{ github.event.inputs.environment || 'staging' }}
          helm rollback my-app -n ${ENVIRONMENT}
          echo "Rolled back to previous version"
```

### Workflow สำหรับ Lint และ Test Chart

```yaml
# .github/workflows/lint-test.yaml
name: Lint and Test Charts

on:
  pull_request:
    paths:
      - 'charts/**'

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.14.0

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Set up chart-testing
        uses: helm/chart-testing-action@v2.6.1

      - name: Run chart-testing (list-changed)
        id: list-changed
        run: |
          changed=$(ct list-changed --target-branch ${{ github.event.repository.default_branch }})
          if [[ -n "$changed" ]]; then
            echo "changed=true" >> "$GITHUB_OUTPUT"
          fi

      - name: Run chart-testing (lint)
        if: steps.list-changed.outputs.changed == 'true'
        run: ct lint --target-branch ${{ github.event.repository.default_branch }}

      - name: Create kind cluster
        if: steps.list-changed.outputs.changed == 'true'
        uses: helm/kind-action@v1.9.0

      - name: Run chart-testing (install)
        if: steps.list-changed.outputs.changed == 'true'
        run: ct install --target-branch ${{ github.event.repository.default_branch }}
```

### values files สำหรับ Environment ต่างๆ

```yaml
# charts/my-app/values-staging.yaml
replicaCount: 1

image:
  repository: ghcr.io/myorg/my-app
  pullPolicy: Always

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
  hosts:
    - host: staging.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: staging-tls
      hosts:
        - staging.example.com

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

app:
  logLevel: debug
  database:
    host: postgresql.database.svc.cluster.local
    port: 5432
    name: myapp_staging
```

```yaml
# charts/my-app/values-production.yaml
replicaCount: 3

image:
  repository: ghcr.io/myorg/my-app
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: production-tls
      hosts:
        - example.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

app:
  logLevel: info
  database:
    host: postgresql-primary.database.svc.cluster.local
    port: 5432
    name: myapp_production
```

---

## Best Practices

### 1. Chart Versioning

```yaml
# Chart.yaml
# ใช้ SemVer
# - MAJOR: breaking changes
# - MINOR: new features, backward compatible
# - PATCH: bug fixes

version: 1.2.3      # chart version
appVersion: "2.0.0" # application version (ใส่ quotes เพราะเป็น string)
```

### 2. Values ที่ดี

```yaml
# ✓ ดี - มี type annotations ใน comments
# -- จำนวน replicas
# @default 1
replicaCount: 1

# -- Container image configuration
# @type object
image:
  # -- Image repository
  repository: nginx
  # -- Image tag ถ้าไม่กำหนดจะใช้ appVersion
  # @default .Chart.AppVersion
  tag: ""
```

### 3. ใช้ _helpers.tpl สำหรับ Common Logic

```yaml
# ✓ ดี - reuse ได้
{{- define "my-app.commonEnv" -}}
- name: APP_ENV
  value: {{ .Values.environment }}
- name: LOG_LEVEL
  value: {{ .Values.logLevel }}
{{- end }}

# ใน deployment.yaml
env:
  {{- include "my-app.commonEnv" . | nindent 12 }}
```

### 4. Validation ด้วย required

```yaml
# บังคับให้กำหนด values ที่จำเป็น
image:
  repository: {{ required "image.repository is required" .Values.image.repository }}
  tag: {{ required "image.tag is required" .Values.image.tag }}
```

### 5. ใช้ fail สำหรับ validation ที่ซับซ้อน

```yaml
{{- if and .Values.autoscaling.enabled (lt (int .Values.replicaCount) (int .Values.autoscaling.minReplicas)) }}
  {{- fail "replicaCount must be >= autoscaling.minReplicas when autoscaling is enabled" }}
{{- end }}
```

### 6. Checksum Annotations สำหรับ Auto-restart

```yaml
# deployment.yaml
spec:
  template:
    metadata:
      annotations:
        # restart pods เมื่อ configmap เปลี่ยน
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        # restart pods เมื่อ secret เปลี่ยน
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
```

### 7. Schema Validation

```json
// values.schema.json
{
  "$schema": "http://json-schema.org/draft-07/schema",
  "type": "object",
  "required": ["image"],
  "properties": {
    "replicaCount": {
      "type": "integer",
      "minimum": 1,
      "description": "Number of replicas"
    },
    "image": {
      "type": "object",
      "required": ["repository"],
      "properties": {
        "repository": {
          "type": "string",
          "description": "Image repository"
        },
        "tag": {
          "type": "string",
          "description": "Image tag"
        },
        "pullPolicy": {
          "type": "string",
          "enum": ["Always", "Never", "IfNotPresent"]
        }
      }
    },
    "service": {
      "type": "object",
      "properties": {
        "type": {
          "type": "string",
          "enum": ["ClusterIP", "NodePort", "LoadBalancer"]
        },
        "port": {
          "type": "integer",
          "minimum": 1,
          "maximum": 65535
        }
      }
    }
  }
}
```

---

## แบบฝึกหัด

### Exercise 1: สร้าง Helm Chart จาก Scratch

สร้าง Helm chart สำหรับ Node.js web application:

**Requirements:**
- Deployment มี health checks (liveness/readiness)
- Service ชนิด ClusterIP
- Ingress พร้อม TLS
- HPA สำหรับ auto scaling
- ConfigMap สำหรับ environment config
- Secret สำหรับ database credentials

**ขั้นตอน:**

1. สร้าง chart structure:
```bash
helm create nodejs-app
cd nodejs-app
```

2. แก้ไข `Chart.yaml`:
```yaml
apiVersion: v2
name: nodejs-app
description: A Node.js web application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

3. กำหนด values ใน `values.yaml`:
```yaml
replicaCount: 2
image:
  repository: my-nodejs-app
  tag: ""
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 3000
ingress:
  enabled: true
  host: myapp.example.com
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
database:
  host: postgresql
  port: 5432
  name: nodejs_db
  secretName: db-credentials
config:
  NODE_ENV: production
  LOG_LEVEL: info
```

4. สร้าง templates และ deploy:
```bash
# Lint ก่อน
helm lint ./nodejs-app

# Render templates
helm template my-nodejs-app ./nodejs-app

# Deploy ไปยัง cluster
helm install my-nodejs-app ./nodejs-app --dry-run --debug
helm install my-nodejs-app ./nodejs-app

# ตรวจสอบ
helm status my-nodejs-app
helm get manifest my-nodejs-app
```

### Exercise 2: ใช้ Helm กับ External Chart

Deploy WordPress โดยใช้ Bitnami chart:

```bash
# เพิ่ม repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# ดู default values
helm show values bitnami/wordpress > wordpress-values.yaml

# สร้าง custom values
cat > my-wordpress-values.yaml << 'EOF'
wordpressUsername: admin
wordpressPassword: "secretpassword"
wordpressEmail: admin@example.com
wordpressBlogName: "My Blog"

mariadb:
  auth:
    rootPassword: "rootpassword"
    database: wordpress
    username: wordpress
    password: "dbpassword"

persistence:
  enabled: true
  size: 10Gi

service:
  type: LoadBalancer

ingress:
  enabled: true
  hostname: blog.example.com
EOF

# Install
helm install my-wordpress bitnami/wordpress \
  -f my-wordpress-values.yaml \
  --namespace wordpress \
  --create-namespace

# ตรวจสอบ
helm status my-wordpress -n wordpress
kubectl get pods -n wordpress
kubectl get svc -n wordpress
```

### Exercise 3: Helmfile สำหรับ Full Stack

สร้าง helmfile.yaml สำหรับ deploy full stack application:

```yaml
# helmfile.yaml
repositories:
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx

releases:
  - name: ingress-nginx
    namespace: ingress-nginx
    createNamespace: true
    chart: ingress-nginx/ingress-nginx
    version: "4.9.0"

  - name: postgresql
    namespace: database
    createNamespace: true
    chart: bitnami/postgresql
    version: "12.x.x"
    values:
      - postgresql:
          auth:
            password: "{{ requiredEnv \"DB_PASSWORD\" }}"
            database: myapp

  - name: redis
    namespace: cache
    createNamespace: true
    chart: bitnami/redis
    version: "18.x.x"
    values:
      - auth:
          enabled: false

  - name: backend
    namespace: production
    createNamespace: true
    chart: ./charts/backend
    needs:
      - database/postgresql
      - cache/redis
    values:
      - ./values/backend-prod.yaml

  - name: frontend
    namespace: production
    chart: ./charts/frontend
    needs:
      - production/backend
    values:
      - ./values/frontend-prod.yaml
```

```bash
# Set environment variables
export DB_PASSWORD=secretpassword

# Deploy ทั้งหมด
helmfile apply

# ดู diff ก่อน apply
helmfile diff

# Destroy ทั้งหมด
helmfile destroy
```

### Exercise 4: CI/CD Pipeline

สร้าง GitHub Actions workflow ที่:
1. Build Docker image เมื่อมี push ไปยัง main
2. Push image ไปยัง GitHub Container Registry
3. Lint Helm chart
4. Deploy ไปยัง staging อัตโนมัติ
5. รอ approval ก่อน deploy ไปยัง production
6. Rollback อัตโนมัติเมื่อ deployment ล้มเหลว

ดูตัวอย่างใน [CI/CD Integration section](#cicd-integration-ด้วย-github-actions) และปรับแต่งเพิ่มเติม

---

## สรุป

Helm เป็นเครื่องมือที่จำเป็นสำหรับการจัดการ Kubernetes deployments ในระดับ production สิ่งสำคัญที่ควรจำ:

1. **Chart Structure**: รู้จักทุกไฟล์ในโครงสร้าง chart
2. **Templates**: ใช้ Go templates และ Sprig functions อย่างมีประสิทธิภาพ
3. **Values**: แยก values สำหรับแต่ละ environment
4. **Dependencies**: ใช้ Chart dependencies สำหรับ third-party services
5. **Repositories**: รู้จัก Artifact Hub และ repository management
6. **Helmfile**: ใช้สำหรับ manage หลาย charts พร้อมกัน
7. **CI/CD**: Integrate Helm เข้ากับ GitHub Actions
8. **Best Practices**: Validation, checksums, schema validation

ในบทต่อไปเราจะเรียนรู้เกี่ยวกับ AWS CodePipeline ซึ่งเป็น CI/CD service ของ Amazon Web Services

# Part 24: Kubernetes Fundamentals

## บทนำ

Kubernetes (K8s) คือ open-source container orchestration platform ที่พัฒนาโดย Google และปัจจุบันดูแลโดย Cloud Native Computing Foundation (CNCF) Kubernetes ช่วยจัดการ containers ในระดับ production โดย automate การ deploy, scale, และ manage containerized applications

**ทำไมต้องใช้ Kubernetes?**
- **Auto-scaling**: Scale แอปพลิเคชันตาม load อัตโนมัติ
- **Self-healing**: Restart failed containers โดยอัตโนมัติ
- **Load balancing**: กระจาย traffic ให้ containers
- **Rolling updates**: Deploy version ใหม่โดยไม่มี downtime
- **Service discovery**: Containers ค้นหากันเองได้
- **Storage orchestration**: จัดการ persistent storage

---

## สารบัญ

1. Kubernetes คืออะไร
2. สถาปัตยกรรม Kubernetes
3. Setup kubectl
4. Pods
5. ReplicaSets
6. Deployments
7. Services
8. ConfigMaps และ Secrets
9. Namespaces
10. Resource Limits
11. Labels และ Selectors
12. minikube/kind Setup
13. kubectl Commands
14. YAML Manifests
15. แบบฝึกหัด

---

## 1. Kubernetes คืออะไร?

### Container Orchestration

ก่อน Kubernetes เราต้องจัดการ containers เองด้วยมือ:

```
ปัญหา:                          Kubernetes แก้:
- ต้องรู้ว่า container อยู่ที่ไหน  -> Auto-scheduling
- Scale ด้วยมือ                  -> Auto-scaling
- Restart ด้วยมือ                -> Self-healing
- Load balance ด้วยมือ           -> Built-in load balancing
- Update แต่ละ container         -> Rolling updates
- Configure network               -> Container networking
```

### Kubernetes Glossary

- **Node**: เครื่อง server (physical หรือ virtual)
- **Cluster**: กลุ่มของ nodes ที่ทำงานร่วมกัน
- **Pod**: unit ที่เล็กที่สุดที่ deploy ได้ (1+ containers)
- **Deployment**: จัดการ Pod replicas
- **Service**: expose Pods เป็น network service
- **Namespace**: virtual cluster เพื่อ isolate resources
- **ConfigMap**: เก็บ configuration data
- **Secret**: เก็บ sensitive data
- **Volume**: persistent storage สำหรับ Pods
- **Ingress**: HTTP/HTTPS routing

---

## 2. สถาปัตยกรรม Kubernetes

```
+--------------------------------------------------+
|                  Kubernetes Cluster               |
|                                                   |
|  +------------------+   +--------------------+   |
|  |   Control Plane  |   |    Worker Node 1   |   |
|  |                  |   |                    |   |
|  | +--------------+ |   | +--------------+   |   |
|  | | API Server   | |   | | kubelet      |   |   |
|  | +--------------+ |   | +--------------+   |   |
|  |                  |   | | kube-proxy   |   |   |
|  | +--------------+ |   | +--------------+   |   |
|  | | etcd         | |   | | Container    |   |   |
|  | +--------------+ |   | | Runtime      |   |   |
|  |                  |   | +--------------+   |   |
|  | +--------------+ |   |                    |   |
|  | | Scheduler    | |   | +----+ +----+      |   |
|  | +--------------+ |   | |Pod | |Pod |      |   |
|  |                  |   | +----+ +----+      |   |
|  | +--------------+ |   +--------------------+   |
|  | | Controller   | |                            |
|  | | Manager      | |   +--------------------+   |
|  | +--------------+ |   |    Worker Node 2   |   |
|  +------------------+   |    ...             |   |
|                          +--------------------+   |
+--------------------------------------------------+
```

### Control Plane Components

#### API Server (kube-apiserver)
- จุดเชื่อมต่อหลักกับ cluster
- รองรับ RESTful API
- ตรวจสอบ authentication และ authorization
- kubectl ส่ง requests ไปยัง API Server

#### etcd
- Distributed key-value store
- เก็บ cluster state ทั้งหมด
- ต้อง backup เสมอ!
- ใช้ Raft consensus algorithm

#### Scheduler (kube-scheduler)
- กำหนดว่า Pod ควรรันบน Node ไหน
- พิจารณา: resource requirements, affinity/anti-affinity, taints/tolerations
- ไม่ได้รัน Pod เอง แค่เลือก Node

#### Controller Manager (kube-controller-manager)
- รัน controller loops
- Deployment Controller: จัดการ ReplicaSets
- ReplicaSet Controller: จัดการ Pods
- Node Controller: ตรวจสอบ node health
- Service Account Controller: สร้าง service accounts

### Worker Node Components

#### kubelet
- Agent ที่รันบนทุก node
- รับ Pod specification จาก API Server
- สั่ง container runtime รัน containers
- ตรวจสอบ container health

#### kube-proxy
- รัน network proxy บน node
- จัดการ network rules
- Implement Service concept

#### Container Runtime
- Software ที่รัน containers
- Options: containerd (default), CRI-O, Docker Engine
- ต้องรองรับ Container Runtime Interface (CRI)

---

## 3. Setup kubectl

### ติดตั้ง kubectl

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# macOS
brew install kubectl
# หรือ
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Windows (Chocolatey)
choco install kubernetes-cli
```

### kubeconfig

```yaml
# ~/.kube/config
apiVersion: v1
kind: Config
preferences: {}

clusters:
- name: production-cluster
  cluster:
    server: https://api.production.example.com
    certificate-authority-data: <base64-encoded-ca>

- name: staging-cluster
  cluster:
    server: https://api.staging.example.com
    certificate-authority-data: <base64-encoded-ca>

users:
- name: admin-production
  user:
    client-certificate-data: <base64-encoded-cert>
    client-key-data: <base64-encoded-key>

- name: admin-staging
  user:
    token: my-service-account-token

contexts:
- name: production
  context:
    cluster: production-cluster
    user: admin-production
    namespace: default

- name: staging
  context:
    cluster: staging-cluster
    user: admin-staging
    namespace: staging

current-context: staging
```

```bash
# ดู contexts
kubectl config get-contexts

# Switch context
kubectl config use-context production

# ดู current context
kubectl config current-context

# Merge kubeconfig files
KUBECONFIG=~/.kube/config:~/.kube/other-config kubectl config view --flatten > ~/.kube/merged-config

# Set namespace สำหรับ context
kubectl config set-context --current --namespace=production

# ใช้ kubectx และ kubens (helper tools)
brew install kubectx
kubectx production    # switch context
kubens staging        # switch namespace
```

### kubectl Autocomplete

```bash
# Bash
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# Zsh
source <(kubectl completion zsh)
echo "[[ $commands[kubectl] ]] && source <(kubectl completion zsh)" >> ~/.zshrc

# Alias
alias k=kubectl
complete -F __start_kubectl k
```

---

## 4. Pods

Pod คือ unit ที่เล็กที่สุดใน Kubernetes ประกอบด้วย containers หนึ่งตัวหรือมากกว่า ที่แชร์ network และ storage

### Pod YAML

```yaml
# simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
  namespace: default
  labels:
    app: my-app
    version: "1.0"
    tier: frontend
  annotations:
    description: "Frontend web application"
    contact: "team@example.com"
spec:
  containers:
  - name: my-app
    image: nginx:1.24-alpine
    ports:
    - containerPort: 80
      name: http
    - containerPort: 443
      name: https
    
    # Resource limits
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    
    # Environment variables
    env:
    - name: APP_ENV
      value: "production"
    - name: APP_PORT
      value: "8080"
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db_host
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: db_password
    
    # Health probes
    livenessProbe:
      httpGet:
        path: /health/live
        port: 80
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
      timeoutSeconds: 5
    
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
    
    startupProbe:
      httpGet:
        path: /health/startup
        port: 80
      failureThreshold: 30
      periodSeconds: 10
    
    # Volume mounts
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app/config
      readOnly: true
    - name: data-volume
      mountPath: /var/app/data
  
  # Init containers
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      until nc -z $DB_HOST $DB_PORT; do
        echo "Waiting for database..."
        sleep 2
      done
      echo "Database is ready!"
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: db_host
    - name: DB_PORT
      value: "5432"
  
  - name: run-migrations
    image: my-app:migrate
    command: ["./migrate.sh"]
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
  
  # Volumes
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.yaml
        path: app.yaml
  
  - name: data-volume
    persistentVolumeClaim:
      claimName: app-data-pvc
  
  # Pod settings
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
  
  # Security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  
  # Node scheduling
  nodeSelector:
    kubernetes.io/os: linux
    node-type: application
  
  # Affinity
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-type
            operator: In
            values:
            - application
            - general
    
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - my-app
          topologyKey: kubernetes.io/hostname
  
  # Tolerations
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "app"
    effect: "NoSchedule"
  
  # DNS settings
  dnsPolicy: ClusterFirst
  dnsConfig:
    searches:
    - svc.cluster.local
    - cluster.local
  
  # Image pull settings
  imagePullPolicy: Always
  imagePullSecrets:
  - name: registry-credentials
```

### Multi-Container Pods

```yaml
# sidecar-pattern.yaml
# Sidecar Pattern: รัน log shipper ข้างๆ app
apiVersion: v1
kind: Pod
metadata:
  name: app-with-sidecar
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
  
  # Sidecar: Fluent Bit log shipper
  - name: log-shipper
    image: fluent/fluent-bit:2.1
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/app
      readOnly: true
    - name: fluent-bit-config
      mountPath: /fluent-bit/etc
  
  # Ambassador Pattern: กลาง proxy
  - name: envoy-proxy
    image: envoyproxy/envoy:v1.28
    ports:
    - containerPort: 9901  # admin
    - containerPort: 10000 # proxy
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
  
  volumes:
  - name: shared-logs
    emptyDir: {}
  - name: fluent-bit-config
    configMap:
      name: fluent-bit-config
  - name: envoy-config
    configMap:
      name: envoy-config
```

---

## 5. ReplicaSets

ReplicaSet ดูแลให้มี Pod จำนวนที่กำหนดอยู่เสมอ

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-app-rs
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: "1.0"
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0"
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:1.0
        ports:
        - containerPort: 8080
```

```bash
# ดู ReplicaSets
kubectl get rs -n production

# Scale
kubectl scale rs my-app-rs --replicas=5 -n production

# ดู Pods ที่ RS ควบคุม
kubectl get pods -n production -l app=my-app
```

> **หมายเหตุ**: ในทางปฏิบัติ เราไม่ค่อยสร้าง ReplicaSet โดยตรง แต่ใช้ Deployment แทน เพราะ Deployment จัดการ ReplicaSet ให้

---

## 6. Deployments

Deployment เป็น higher-level abstraction ที่จัดการ ReplicaSets และ Pods พร้อม rolling update

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    team: backend
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubernetes.io/change-cause: "Initial deployment v1.0"
spec:
  replicas: 3
  
  selector:
    matchLabels:
      app: my-app
  
  # Update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1       # Pods เพิ่มขึ้นได้สูงสุด
      maxUnavailable: 0  # Pods ที่ unavailable ได้
  
  # Revision history (สำหรับ rollback)
  revisionHistoryLimit: 5
  
  # Progress deadline
  progressDeadlineSeconds: 600
  
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:1.0
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        
        env:
        - name: APP_ENV
          value: "production"
        - name: LOG_LEVEL
          value: "info"
        
        envFrom:
        - configMapRef:
            name: my-app-config
        - secretRef:
            name: my-app-secrets
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: tmp
          mountPath: /tmp
      
      volumes:
      - name: config
        configMap:
          name: my-app-config
      - name: tmp
        emptyDir: {}
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
      
      # Security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      # Spread Pods across nodes
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: my-app
      
      # Service account
      serviceAccountName: my-app-sa
      
      automountServiceAccountToken: false
```

### Deployment Commands

```bash
# Create deployment
kubectl apply -f deployment.yaml

# ดู deployment
kubectl get deployment my-app -n production

# ดูรายละเอียด
kubectl describe deployment my-app -n production

# ดู ReplicaSets
kubectl get rs -n production -l app=my-app

# ดู Pods
kubectl get pods -n production -l app=my-app

# Watch pods
kubectl get pods -n production -w -l app=my-app

# Scale deployment
kubectl scale deployment my-app --replicas=5 -n production

# Update image
kubectl set image deployment/my-app my-app=my-registry/my-app:2.0 -n production

# ดู rollout status
kubectl rollout status deployment/my-app -n production

# ดู rollout history
kubectl rollout history deployment/my-app -n production

# Rollback ไปยัง version ก่อนหน้า
kubectl rollout undo deployment/my-app -n production

# Rollback ไปยัง revision เฉพาะ
kubectl rollout undo deployment/my-app --to-revision=2 -n production

# Pause rollout (หยุดชั่วคราว)
kubectl rollout pause deployment/my-app -n production

# Resume rollout
kubectl rollout resume deployment/my-app -n production

# Annotate deployment (สำหรับ change history)
kubectl annotate deployment/my-app kubernetes.io/change-cause="Update to v2.0 - new features" -n production
```

---

## 7. Services

Service ทำให้ Pods เข้าถึงได้ผ่าน network ที่เสถียร

### Service Types

```yaml
# ClusterIP: เข้าถึงได้เฉพาะภายใน cluster
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  namespace: production
spec:
  type: ClusterIP  # default
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80          # port ของ Service
    targetPort: 8080  # port ของ Pod
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
---
# NodePort: เข้าถึงได้ผ่าน Node IP:NodePort
apiVersion: v1
kind: Service
metadata:
  name: my-app-nodeport
  namespace: production
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 30000-32767 (ถ้าไม่ระบุ จะ assign อัตโนมัติ)
---
# LoadBalancer: สร้าง cloud load balancer
apiVersion: v1
kind: Service
metadata:
  name: my-app-lb
  namespace: production
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  # กำหนด source IP preservation
  externalTrafficPolicy: Local
---
# ExternalName: ชี้ไปยัง external DNS
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: production
spec:
  type: ExternalName
  externalName: mydb.external.example.com
---
# Headless Service: ไม่มี ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: my-app-headless
  namespace: production
spec:
  clusterIP: None  # Headless
  selector:
    app: my-app
  ports:
  - port: 8080
    targetPort: 8080
```

### Service Discovery

```yaml
# Pods สามารถ connect ด้วย:
# 1. Service name (ภายใน namespace เดียวกัน)
#    http://my-app-svc:80
#
# 2. Full DNS name
#    http://my-app-svc.production.svc.cluster.local:80
#
# 3. ClusterIP
#    http://10.96.0.1:80

# ตัวอย่าง
- name: APP_DB_URL
  value: "postgresql://my-db-svc.production.svc.cluster.local:5432/mydb"

- name: REDIS_URL
  value: "redis://redis-svc:6379"
```

---

## 8. ConfigMaps และ Secrets

### ConfigMaps

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: production
data:
  # Key-value pairs
  APP_ENV: "production"
  LOG_LEVEL: "info"
  DB_HOST: "postgres-svc"
  DB_PORT: "5432"
  DB_NAME: "myapp"
  REDIS_HOST: "redis-svc"
  REDIS_PORT: "6379"
  
  # Config file content
  app.yaml: |
    server:
      port: 8080
      timeout: 30s
    
    database:
      host: postgres-svc
      port: 5432
      name: myapp
      max_connections: 20
    
    logging:
      level: info
      format: json
  
  nginx.conf: |
    server {
        listen 80;
        location / {
            proxy_pass http://localhost:8080;
        }
    }
```

```bash
# สร้าง ConfigMap จาก command line
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info

# สร้างจาก file
kubectl create configmap nginx-config \
  --from-file=nginx.conf=./nginx.conf

# สร้างจาก directory
kubectl create configmap app-configs \
  --from-file=./config/

# ดู ConfigMap
kubectl get configmap my-app-config -n production -o yaml
kubectl describe configmap my-app-config -n production
```

### Secrets

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
  namespace: production
  annotations:
    # ใช้ External Secrets Operator ใน production
    kubernetes.io/secret-type: "generic"
type: Opaque
data:
  # ต้อง base64 encode
  # echo -n "my-password" | base64
  DB_PASSWORD: bXktcGFzc3dvcmQ=
  API_KEY: bXktYXBpLWtleQ==
  JWT_SECRET: bXktand0LXNlY3JldA==
  
  # หรือใช้ stringData (Kubernetes encode ให้อัตโนมัติ)
stringData:
  SMTP_PASSWORD: "smtp-secret-password"
  
---
# Docker Registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: |
    <base64 encoded docker config>

---
# TLS Secret
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64 encoded certificate>
  tls.key: <base64 encoded private key>
```

```bash
# สร้าง Generic Secret
kubectl create secret generic my-app-secrets \
  --from-literal=DB_PASSWORD=mypassword \
  --from-literal=API_KEY=my-api-key \
  -n production

# สร้างจาก file
kubectl create secret generic ssl-certs \
  --from-file=ssl.cert=./ssl.cert \
  --from-file=ssl.key=./ssl.key \
  -n production

# สร้าง Docker registry secret
kubectl create secret docker-registry registry-credentials \
  --docker-server=ghcr.io \
  --docker-username=myusername \
  --docker-password=mytoken \
  --docker-email=me@example.com \
  -n production

# ดู secrets (values จะถูก mask)
kubectl get secrets -n production
kubectl describe secret my-app-secrets -n production

# Decode secret value
kubectl get secret my-app-secrets -n production \
  -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

### External Secrets Operator

ใน production ควรใช้ External Secrets Operator แทน manual secrets:

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-secrets
  namespace: production
spec:
  refreshInterval: 1h
  
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  
  target:
    name: my-app-secrets
    creationPolicy: Owner
  
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: production/myapp/database
      property: password
  
  - secretKey: API_KEY
    remoteRef:
      key: production/myapp/api
      property: key
  
  dataFrom:
  - extract:
      key: production/myapp/all-secrets
```

---

## 9. Namespaces

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
  annotations:
    contact: "platform-team@example.com"
```

```bash
# ดู namespaces
kubectl get namespaces

# สร้าง namespace
kubectl create namespace staging

# รัน resource ใน namespace
kubectl apply -f deployment.yaml -n production

# ดู resources ใน namespace
kubectl get all -n production

# ดู resources ทุก namespaces
kubectl get pods --all-namespaces
# หรือ
kubectl get pods -A

# ลบ namespace (ลบทุก resource ใน namespace ด้วย!)
kubectl delete namespace test-env

# Set default namespace
kubectl config set-context --current --namespace=production
```

### Resource Quota

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute resources
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    
    # Object counts
    pods: "50"
    services: "20"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "20"
    
    # Storage
    requests.storage: 100Gi
    persistentvolumeclaims: "10"
    "standard.storageclass.storage.k8s.io/requests.storage": 50Gi
```

### LimitRange

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "1Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
  
  - type: Pod
    max:
      cpu: "4"
      memory: "2Gi"
  
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

---

## 10. Resource Limits

```yaml
# resource-limits-example.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:1.0
        
        resources:
          # requests: Kubernetes ใช้ schedule Pod
          # node ต้องมีทรัพยากรเหลือเท่า requests อย่างน้อย
          requests:
            cpu: "100m"       # 0.1 CPU core
            memory: "128Mi"   # 128 MB
            ephemeral-storage: "1Gi"
          
          # limits: ขีดจำกัดสูงสุดที่ container ใช้ได้
          # ถ้าใช้ CPU เกิน limit: throttled (ช้าลง)
          # ถ้าใช้ memory เกิน limit: OOMKilled (ถูก kill)
          limits:
            cpu: "500m"       # 0.5 CPU core
            memory: "256Mi"   # 256 MB
            ephemeral-storage: "2Gi"
```

### CPU Units

```
1 CPU = 1000m (millicores)
0.5 CPU = 500m
0.1 CPU = 100m

Examples:
- 100m = 0.1 CPU core = 10% ของ 1 core
- 500m = 0.5 CPU core = half of 1 core
- 1000m = 1 = 1 CPU core
- 2000m = 2 = 2 CPU cores
```

### QoS Classes

```
Guaranteed: requests == limits
Burstable:  requests < limits (หรือ requests ไม่ระบุ)
BestEffort: ไม่ระบุ requests และ limits
```

```yaml
# Guaranteed QoS
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# Burstable QoS
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"

# BestEffort QoS (ไม่แนะนำใน production)
# ไม่ระบุ resources เลย
```

---

## 11. Labels และ Selectors

```yaml
# labels-example.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    # Labels ที่แนะนำ (Kubernetes Well-Known Labels)
    app.kubernetes.io/name: my-app
    app.kubernetes.io/version: "1.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: my-platform
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/instance: my-app-production
    
    # Custom labels
    team: platform
    environment: production
    tier: frontend
    cost-center: engineering
spec:
  selector:
    matchLabels:
      app: my-app
      version: "1.0"
```

```bash
# Filter ด้วย labels
kubectl get pods -l app=my-app
kubectl get pods -l app=my-app,version=1.0
kubectl get pods -l "environment in (production, staging)"
kubectl get pods -l "tier notin (backend)"
kubectl get pods -l app=my-app --show-labels

# Add label
kubectl label pod my-app-pod-xxx version=1.0

# Remove label
kubectl label pod my-app-pod-xxx version-

# Annotate
kubectl annotate deployment my-app \
  kubernetes.io/change-cause="Deploy v2.0 - performance improvements"
```

---

## 12. minikube/kind Setup

### minikube

```bash
# ติดตั้ง minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start minikube
minikube start \
  --cpus=4 \
  --memory=8192 \
  --disk-size=20g \
  --driver=docker \
  --kubernetes-version=v1.28.0

# ดู status
minikube status

# ดู dashboard
minikube dashboard

# Stop/Delete
minikube stop
minikube delete

# Add-ons
minikube addons list
minikube addons enable ingress
minikube addons enable metrics-server

# Load local image เข้า minikube
minikube image load my-app:local

# Tunnel (สำหรับ LoadBalancer services)
minikube tunnel

# Service URL
minikube service my-app-svc --url
```

### kind (Kubernetes in Docker)

```bash
# ติดตั้ง kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# สร้าง cluster ธรรมดา
kind create cluster

# สร้าง cluster พร้อม config
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
- role: worker
EOF

kind create cluster --config kind-config.yaml --name dev-cluster

# Load image เข้า kind
kind load docker-image my-app:local --name dev-cluster

# ดู clusters
kind get clusters

# Delete cluster
kind delete cluster --name dev-cluster
```

---

## 13. kubectl Commands

### การดู Resources

```bash
# Get resources
kubectl get pods
kubectl get pods -o wide                    # ดูรายละเอียดเพิ่มเติม
kubectl get pods -o yaml                    # ดู full YAML
kubectl get pods -o json                    # JSON format
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Watch
kubectl get pods -w
kubectl get pods --watch

# Custom columns
kubectl get pods \
  -o custom-columns=\
"NAME:.metadata.name,\
STATUS:.status.phase,\
NODE:.spec.nodeName,\
IP:.status.podIP"

# ดูทุก resource types
kubectl get all
kubectl get all -n production

# ดู resource types ที่มี
kubectl api-resources

# ดู API versions
kubectl api-versions

# Describe resource
kubectl describe pod my-pod
kubectl describe deployment my-app
kubectl describe service my-svc
kubectl describe node my-node
```

### Create/Update/Delete

```bash
# Apply (สร้างหรืออัพเดต)
kubectl apply -f manifest.yaml
kubectl apply -f ./manifests/          # apply ทุก files ใน directory
kubectl apply -f https://url/to/manifest.yaml
kubectl apply -k ./kustomize/           # Kustomize

# Create (สร้างอย่างเดียว, error ถ้ามีอยู่แล้ว)
kubectl create -f manifest.yaml

# Delete
kubectl delete -f manifest.yaml
kubectl delete pod my-pod
kubectl delete deployment my-app
kubectl delete pod -l app=my-app

# Edit (เปิด editor)
kubectl edit deployment my-app

# Patch
kubectl patch deployment my-app \
  -p '{"spec":{"replicas":5}}'

kubectl patch deployment my-app \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/replicas", "value": 5}]'
```

### Debugging

```bash
# Logs
kubectl logs my-pod
kubectl logs my-pod -c my-container     # specific container
kubectl logs my-pod -f                  # follow
kubectl logs my-pod --tail=100          # สุดท้าย 100 lines
kubectl logs my-pod --since=1h          # ชั่วโมงที่ผ่านมา
kubectl logs my-pod --previous          # ของ container ก่อนหน้า

# Exec into pod
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -c my-container -- /bin/sh
kubectl exec my-pod -- ls /app

# Port forward
kubectl port-forward my-pod 8080:8080
kubectl port-forward service/my-svc 8080:80
kubectl port-forward deployment/my-app 8080:8080

# Copy files
kubectl cp my-pod:/app/logs/app.log ./app.log
kubectl cp ./config.yaml my-pod:/app/config/

# Top (resource usage)
kubectl top pods
kubectl top pods -n production
kubectl top nodes

# Events
kubectl get events -n production
kubectl get events --sort-by='.metadata.creationTimestamp'
kubectl get events --field-selector type=Warning
```

---

## 14. YAML Manifests

### Complete Application Example

```yaml
# complete-app.yaml
---
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    environment: production

---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: my-app

---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
  namespace: my-app
data:
  APP_ENV: production
  LOG_LEVEL: info
  app.yaml: |
    server:
      port: 8080
    database:
      host: postgres-svc
      port: 5432
      name: myapp

---
# Secret (ใน production ใช้ External Secrets)
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
  namespace: my-app
type: Opaque
stringData:
  DB_PASSWORD: "change-me-in-production"
  JWT_SECRET: "change-me-in-production"

---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0"
    spec:
      serviceAccountName: my-app
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      containers:
      - name: my-app
        image: my-registry/my-app:1.0
        ports:
        - containerPort: 8080
          name: http
        
        envFrom:
        - configMapRef:
            name: my-app-config
        - secretRef:
            name: my-app-secrets
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
      
      volumes:
      - name: config
        configMap:
          name: my-app-config
      
      terminationGracePeriodSeconds: 60
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: kubernetes.io/hostname

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: my-app-svc
  namespace: my-app
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 80
    targetPort: 8080

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: my-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## 15. แบบฝึกหัด

### แบบฝึกหัดที่ 1: First Pod

```bash
# 1. สร้าง minikube cluster
minikube start --cpus=2 --memory=4096

# 2. สร้าง namespace
kubectl create namespace exercise-1

# 3. สร้าง Pod จาก YAML
cat > ex1-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  namespace: exercise-1
  labels:
    app: hello
spec:
  containers:
  - name: hello
    image: hashicorp/http-echo:latest
    args:
    - "-text=Hello from Kubernetes!"
    ports:
    - containerPort: 5678
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 64Mi
EOF

kubectl apply -f ex1-pod.yaml

# 4. ตรวจสอบ Pod
kubectl get pod hello-pod -n exercise-1
kubectl describe pod hello-pod -n exercise-1

# 5. Port forward และทดสอบ
kubectl port-forward pod/hello-pod 8080:5678 -n exercise-1 &
curl http://localhost:8080

# 6. ดู logs
kubectl logs hello-pod -n exercise-1
```

### แบบฝึกหัดที่ 2: Deployment + Service

```yaml
# exercise-2.yaml
# สร้าง Deployment ที่มี:
# - 3 replicas
# - Resource limits
# - Rolling update strategy
# - Health probes
# - ConfigMap สำหรับ environment variables
# - Service ประเภท NodePort

# TODO: เขียน complete YAML

# 1. สร้าง ConfigMap
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: ex2-config
  namespace: exercise-2
# TODO: เพิ่ม data

# 2. สร้าง Deployment
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ex2-app
  namespace: exercise-2
# TODO: spec

# 3. สร้าง Service
---
apiVersion: v1
kind: Service
metadata:
  name: ex2-svc
  namespace: exercise-2
# TODO: spec

# หลังสร้างแล้ว:
# kubectl apply -f exercise-2.yaml
# kubectl rollout status deployment/ex2-app -n exercise-2
# minikube service ex2-svc -n exercise-2 --url
```

### แบบฝึกหัดที่ 3: Namespaces และ RBAC

```bash
# สร้าง namespaces
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Resource Quota สำหรับ development
cat > dev-quota.yaml << 'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: development
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
EOF

kubectl apply -f dev-quota.yaml

# LimitRange
cat > dev-limits.yaml << 'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: development
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 128Mi
    defaultRequest:
      cpu: 100m
      memory: 64Mi
EOF

kubectl apply -f dev-limits.yaml

# ดู quotas
kubectl describe resourcequota -n development
kubectl describe limitrange -n development
```

### แบบฝึกหัดที่ 4: Secrets Management

```bash
# 1. สร้าง generic secret
kubectl create secret generic db-credentials \
  --from-literal=username=myapp \
  --from-literal=password=secretpassword \
  -n exercise-4

# 2. สร้าง Docker registry secret
kubectl create secret docker-registry registry-creds \
  --docker-server=ghcr.io \
  --docker-username=myusername \
  --docker-password=mytoken \
  -n exercise-4

# 3. สร้าง TLS secret (ด้วย self-signed cert)
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=myapp.example.com"

kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n exercise-4

# 4. สร้าง deployment ที่ใช้ secrets
cat > ex4-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ex4-app
  namespace: exercise-4
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ex4-app
  template:
    metadata:
      labels:
        app: ex4-app
    spec:
      imagePullSecrets:
      - name: registry-creds
      containers:
      - name: app
        image: nginx:alpine
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: tls
          mountPath: /etc/nginx/ssl
          readOnly: true
      volumes:
      - name: tls
        secret:
          secretName: tls-secret
EOF

kubectl apply -f ex4-deployment.yaml
```

### แบบฝึกหัดที่ 5: Debugging

```bash
# ลองสร้าง broken deployment และ debug

cat > broken-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app
    spec:
      containers:
      - name: app
        image: nginx:nonexistent-tag   # image ไม่มีอยู่จริง
        resources:
          requests:
            memory: "10Gi"   # ขอ memory มากเกินไป
EOF

kubectl apply -f broken-app.yaml

# Debug tasks:
# 1. ดู pod status และ events
kubectl get pods -w
kubectl describe pod <pod-name>
kubectl get events --sort-by='.lastTimestamp'

# 2. แก้ไขปัญหา image
kubectl set image deployment/broken-app app=nginx:latest

# 3. แก้ไขปัญหา resource
kubectl edit deployment broken-app
# เปลี่ยน memory request เป็น 128Mi

# 4. ตรวจสอบว่าแก้ไขได้แล้ว
kubectl rollout status deployment/broken-app
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **Kubernetes Architecture**: Control Plane + Worker Nodes
2. **Pods**: unit ที่เล็กที่สุด ประกอบด้วย containers
3. **ReplicaSets**: รักษาจำนวน Pod replicas
4. **Deployments**: จัดการ rolling updates และ rollbacks
5. **Services**: ClusterIP, NodePort, LoadBalancer
6. **ConfigMaps/Secrets**: จัดการ configuration และ sensitive data
7. **Namespaces**: isolation และ resource management
8. **Resource Limits**: requests และ limits สำหรับ containers

**Key Takeaways:**
- ใช้ Deployment ไม่ใช่ Pod โดยตรงใน production
- ตั้ง resource requests/limits เสมอ
- ใช้ readinessProbe เพื่อให้แน่ใจว่า Pod พร้อมรับ traffic
- เก็บ secrets ใน Secret ไม่ใช่ ConfigMap หรือ environment variables ใน code
- ใช้ namespaces เพื่อ isolate environments

**ใน Part 25** เราจะเรียนรู้เรื่อง Advanced Kubernetes Deployments

---

## แหล่งอ้างอิง

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Kubernetes Patterns](https://www.oreilly.com/library/view/kubernetes-patterns/9781492050278/)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)

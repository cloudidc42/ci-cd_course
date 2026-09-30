# Part 25: Kubernetes Deployment & Services (Advanced)

## บทนำ

ในบทนี้เราจะเรียนรู้ advanced topics ของ Kubernetes ที่จำเป็นสำหรับ production deployment ได้แก่ StatefulSets, DaemonSets, Jobs, Ingress Controllers, Horizontal Pod Autoscaler, Persistent Volumes และ deployment strategies ใน Kubernetes

---

## สารบัญ

1. Deployment YAML ครบถ้วน
2. Rolling Updates และ Rollback
3. StatefulSets
4. DaemonSets
5. Jobs และ CronJobs
6. Ingress Controllers
7. Horizontal Pod Autoscaler (HPA)
8. PersistentVolumes และ PVCs
9. Resource Requests/Limits
10. Liveness/Readiness Probes
11. Deployment Strategies ใน K8s
12. แบบฝึกหัด

---

## 1. Deployment YAML ครบถ้วน

```yaml
# production-deployment.yaml
# Complete production-grade deployment with all best practices

---
# 1. Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce
  labels:
    name: ecommerce
    environment: production
    istio-injection: enabled

---
# 2. ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app
  namespace: ecommerce
  annotations:
    # AWS IAM Role for Service Account (IRSA)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/web-app-role

---
# 3. ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  namespace: ecommerce
data:
  APP_ENV: production
  LOG_LEVEL: info
  LOG_FORMAT: json
  SERVER_PORT: "8080"
  METRICS_PORT: "9090"
  DB_HOST: postgres-svc.ecommerce.svc.cluster.local
  DB_PORT: "5432"
  DB_NAME: ecommerce
  REDIS_HOST: redis-svc.ecommerce.svc.cluster.local
  REDIS_PORT: "6379"
  S3_BUCKET: ecommerce-assets
  CDN_URL: https://cdn.example.com

  # Config file
  app-config.yaml: |
    server:
      port: 8080
      timeout: 30s
      graceful_shutdown: 30s
    
    database:
      max_connections: 20
      min_connections: 5
      connection_timeout: 10s
      idle_timeout: 600s
    
    cache:
      ttl: 300s
      max_items: 10000
    
    rate_limit:
      enabled: true
      requests_per_second: 100
      burst: 200

---
# 4. Main Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: ecommerce
  labels:
    app.kubernetes.io/name: web-app
    app.kubernetes.io/version: "2.0"
    app.kubernetes.io/component: frontend
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/managed-by: kubectl
    team: frontend
    environment: production
  annotations:
    kubernetes.io/change-cause: "Deploy v2.0 - New checkout flow"
spec:
  # จำนวน replicas
  replicas: 3
  
  # Label selector (immutable หลังสร้าง)
  selector:
    matchLabels:
      app: web-app
  
  # Update strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # เพิ่ม Pod ได้สูงสุด 1 ตัว
      maxUnavailable: 0     # ห้ามมี Pod ที่ unavailable
  
  # เก็บ history กี่ revision
  revisionHistoryLimit: 10
  
  # หมดเวลา deploy เมื่อไร
  progressDeadlineSeconds: 600
  
  # Pod template
  template:
    metadata:
      labels:
        app: web-app
        version: "2.0"
        tier: frontend
      annotations:
        # Prometheus scraping
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
        
        # Vault injection (ถ้าใช้ Vault Agent Injector)
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "web-app"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/ecommerce/web-app"
    
    spec:
      # Service Account
      serviceAccountName: web-app
      automountServiceAccountToken: false
      
      # Pod-level security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
        sysctls:
        - name: net.core.somaxconn
          value: "65535"
      
      # Image pull secrets
      imagePullSecrets:
      - name: registry-credentials
      
      # Init containers
      initContainers:
      - name: wait-for-db
        image: busybox:1.36
        command:
        - sh
        - -c
        - |
          echo "Waiting for database..."
          until nc -z $DB_HOST $DB_PORT; do
            echo "Database not ready, waiting..."
            sleep 2
          done
          echo "Database is ready!"
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: DB_PORT
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 65534
          capabilities:
            drop:
            - ALL
      
      - name: run-migrations
        image: my-registry/web-app:2.0
        command: ["./migrate"]
        args: ["--direction=up"]
        envFrom:
        - configMapRef:
            name: web-app-config
        - secretRef:
            name: web-app-secrets
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: false
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL
      
      # Main containers
      containers:
      - name: web-app
        image: my-registry/web-app:2.0
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
        - name: health
          containerPort: 8081
          protocol: TCP
        
        # Environment variables
        envFrom:
        - configMapRef:
            name: web-app-config
        - secretRef:
            name: web-app-secrets
        
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        
        # Resource management
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
            ephemeral-storage: "1Gi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
            ephemeral-storage: "2Gi"
        
        # Health probes
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8081
            scheme: HTTP
          failureThreshold: 30
          periodSeconds: 10
          timeoutSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 5
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 3
        
        # Container security
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        
        # Volume mounts
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /app/cache
        
        # Lifecycle hooks
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 5  # รอให้ load balancer update ก่อน shutdown
      
      # Sidecar: Log shipper
      - name: log-shipper
        image: fluent/fluent-bit:2.2
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        volumeMounts:
        - name: app-logs
          mountPath: /var/log/app
          readOnly: true
        - name: fluentbit-config
          mountPath: /fluent-bit/etc
          readOnly: true
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
      
      # Volumes
      volumes:
      - name: config
        configMap:
          name: web-app-config
          items:
          - key: app-config.yaml
            path: app-config.yaml
      - name: tmp
        emptyDir:
          medium: Memory
          sizeLimit: 256Mi
      - name: cache
        emptyDir:
          sizeLimit: 1Gi
      - name: app-logs
        emptyDir:
          sizeLimit: 1Gi
      - name: fluentbit-config
        configMap:
          name: fluentbit-config
      
      # Scheduling
      nodeSelector:
        kubernetes.io/os: linux
        workload-type: application
      
      # Affinity rules
      affinity:
        # Pod anti-affinity: กระจาย pods ไปต่างๆ nodes
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web-app
            topologyKey: kubernetes.io/hostname
        
        # Node affinity: ต้องการ node ที่มี label
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload-type
                operator: In
                values:
                - application
                - general
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            preference:
              matchExpressions:
              - key: node-type
                operator: In
                values:
                - on-demand
      
      # Tolerations
      tolerations:
      - key: "workload-type"
        operator: "Equal"
        value: "application"
        effect: "NoSchedule"
      
      # Topology spread
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web-app
      
      terminationGracePeriodSeconds: 60
      dnsPolicy: ClusterFirst

---
# 5. Service
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
  namespace: ecommerce
  labels:
    app: web-app
spec:
  selector:
    app: web-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
    protocol: TCP
  type: ClusterIP
  sessionAffinity: None

---
# 6. PodDisruptionBudget (สำหรับ High Availability)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
  namespace: ecommerce
spec:
  minAvailable: 2  # หรือใช้ maxUnavailable: 1
  selector:
    matchLabels:
      app: web-app
```

---

## 2. Rolling Updates และ Rollback

```bash
# Update image
kubectl set image deployment/web-app web-app=my-registry/web-app:3.0 \
  -n ecommerce \
  --record  # deprecated, ใช้ annotation แทน

# ดู rollout status
kubectl rollout status deployment/web-app -n ecommerce

# Watch pods
kubectl get pods -n ecommerce -l app=web-app -w

# ดู rollout history
kubectl rollout history deployment/web-app -n ecommerce

# ดูรายละเอียด revision เฉพาะ
kubectl rollout history deployment/web-app -n ecommerce --revision=3

# Rollback ไปยัง version ก่อนหน้า
kubectl rollout undo deployment/web-app -n ecommerce

# Rollback ไปยัง revision เฉพาะ
kubectl rollout undo deployment/web-app -n ecommerce --to-revision=2

# Pause update (เพื่อตรวจสอบ)
kubectl rollout pause deployment/web-app -n ecommerce

# Resume
kubectl rollout resume deployment/web-app -n ecommerce

# Annotate สำหรับ change history
kubectl annotate deployment/web-app \
  kubernetes.io/change-cause="v3.0: Add new payment gateway integration" \
  -n ecommerce
```

### Rolling Update Script

```bash
#!/bin/bash
# rolling-update.sh

set -euo pipefail

DEPLOYMENT="web-app"
NAMESPACE="ecommerce"
NEW_IMAGE="my-registry/web-app:${1:-latest}"
TIMEOUT="10m"

echo "Starting rolling update..."
echo "Deployment: ${DEPLOYMENT}"
echo "New Image: ${NEW_IMAGE}"
echo "Namespace: ${NAMESPACE}"

# บันทึก current image สำหรับ rollback
CURRENT_IMAGE=$(kubectl get deployment ${DEPLOYMENT} -n ${NAMESPACE} \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "Current image: ${CURRENT_IMAGE}"

# Annotate change reason
VERSION=$(echo ${NEW_IMAGE} | cut -d: -f2)
kubectl annotate deployment/${DEPLOYMENT} \
  kubernetes.io/change-cause="Deploy ${VERSION} by ${USER}" \
  -n ${NAMESPACE} \
  --overwrite

# Update image
kubectl set image deployment/${DEPLOYMENT} \
  ${DEPLOYMENT}=${NEW_IMAGE} \
  -n ${NAMESPACE}

# Monitor rollout
if kubectl rollout status deployment/${DEPLOYMENT} \
  -n ${NAMESPACE} --timeout=${TIMEOUT}; then
  
  echo "Rolling update completed successfully!"
  
  # Run smoke tests
  echo "Running smoke tests..."
  POD=$(kubectl get pods -n ${NAMESPACE} -l app=${DEPLOYMENT} \
    --field-selector=status.phase=Running \
    -o jsonpath='{.items[0].metadata.name}')
  
  # Health check
  kubectl exec ${POD} -n ${NAMESPACE} -- \
    curl -sf http://localhost:8081/health/ready
  
  echo "Smoke tests passed!"
  echo "Deployed: ${NEW_IMAGE}"
  
else
  echo "Rolling update FAILED! Rolling back..."
  kubectl rollout undo deployment/${DEPLOYMENT} -n ${NAMESPACE}
  kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE}
  echo "Rolled back to: ${CURRENT_IMAGE}"
  exit 1
fi
```

---

## 3. StatefulSets

StatefulSet ใช้สำหรับ stateful applications เช่น databases ที่ต้องการ:
- Stable network identity
- Stable persistent storage
- Ordered deployment และ scaling

```yaml
# statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: ecommerce
spec:
  serviceName: postgres-headless  # ต้องมี Headless Service
  replicas: 3
  
  selector:
    matchLabels:
      app: postgres
  
  # Update strategy
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # อัพเดตทุก pods (0 = all)
  
  # Pod management
  podManagementPolicy: OrderedReady  # หรือ Parallel
  
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        
        ports:
        - containerPort: 5432
          name: postgres
        
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: password
        - name: POSTGRES_DB
          value: ecommerce
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        
        # PostgreSQL replication config
        - name: POSTGRES_REPLICATION_USER
          value: replicator
        - name: POSTGRES_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secrets
              key: replication_password
        
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
        
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 5
        
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - postgres
          initialDelaySeconds: 30
          periodSeconds: 15
          failureThreshold: 3
        
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        - name: config
          mountPath: /etc/postgresql
          readOnly: true
      
      volumes:
      - name: config
        configMap:
          name: postgres-config
  
  # Volume claim templates (แต่ละ Pod ได้ PVC ของตัวเอง)
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi

---
# Headless Service (จำเป็นสำหรับ StatefulSet)
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: ecommerce
spec:
  clusterIP: None  # Headless
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
    name: postgres

---
# Regular Service สำหรับ read
apiVersion: v1
kind: Service
metadata:
  name: postgres-svc
  namespace: ecommerce
spec:
  selector:
    app: postgres
    role: primary  # หรือใช้ pattern อื่น
  ports:
  - port: 5432
    targetPort: 5432
```

### StatefulSet DNS

```
# แต่ละ Pod ใน StatefulSet มี stable DNS:
# <pod-name>.<service-name>.<namespace>.svc.cluster.local
# 
# Example:
# postgres-0.postgres-headless.ecommerce.svc.cluster.local
# postgres-1.postgres-headless.ecommerce.svc.cluster.local
# postgres-2.postgres-headless.ecommerce.svc.cluster.local
```

```bash
# StatefulSet commands
kubectl get statefulsets -n ecommerce
kubectl describe statefulset postgres -n ecommerce

# Scale
kubectl scale statefulset postgres --replicas=5 -n ecommerce

# ดู pods (จะมีชื่อ: postgres-0, postgres-1, postgres-2)
kubectl get pods -n ecommerce -l app=postgres

# ดู PVCs
kubectl get pvc -n ecommerce -l app=postgres

# Delete pod (StatefulSet จะสร้างใหม่ด้วยชื่อเดิม)
kubectl delete pod postgres-0 -n ecommerce
```

---

## 4. DaemonSets

DaemonSet รัน Pod หนึ่งตัวบนทุก node (หรือบน node ที่ระบุ)

```yaml
# daemonset.yaml
# Use cases: log collectors, monitoring agents, network plugins

---
# Node Exporter (Prometheus metrics สำหรับ nodes)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # อัพเดต 1 node ต่อครั้ง
  
  template:
    metadata:
      labels:
        app: node-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
    spec:
      # รัน DaemonSet บน Master nodes ด้วย
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.7.0
        
        args:
        - '--path.procfs=/host/proc'
        - '--path.rootfs=/rootfs'
        - '--path.sysfs=/host/sys'
        - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
        
        ports:
        - containerPort: 9100
          name: metrics
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
        - name: root
          mountPath: /rootfs
          readOnly: true
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
      
      hostNetwork: true
      hostPID: true
      
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      - name: root
        hostPath:
          path: /

---
# Log Collector DaemonSet
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      
      serviceAccountName: fluent-bit
      
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: dockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc/
      
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: dockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: fluent-bit-config
```

---

## 5. Jobs และ CronJobs

### Job

Job รัน task จนสำเร็จแล้วหยุด

```yaml
# job.yaml

# 1. Simple Job
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: ecommerce
spec:
  # จำนวนครั้งที่ต้องรันสำเร็จ
  completions: 1
  
  # รัน parallel ได้กี่ตัว
  parallelism: 1
  
  # ลองใหม่กี่ครั้งถ้า fail
  backoffLimit: 3
  
  # ลบ Job หลังจาก X วินาทีหลังสำเร็จ
  ttlSecondsAfterFinished: 3600
  
  # ใช้เวลาสูงสุดเท่าไร
  activeDeadlineSeconds: 1800
  
  template:
    spec:
      restartPolicy: OnFailure  # Job ต้องการ OnFailure หรือ Never
      
      containers:
      - name: migration
        image: my-registry/web-app:2.0
        command: ["./migrate"]
        args: ["--direction=up", "--verbose"]
        
        envFrom:
        - configMapRef:
            name: web-app-config
        - secretRef:
            name: web-app-secrets
        
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi

---
# 2. Parallel Job (สร้าง thumbnails)
apiVersion: batch/v1
kind: Job
metadata:
  name: generate-thumbnails
  namespace: ecommerce
spec:
  completions: 100     # ต้องทำให้ครบ 100 images
  parallelism: 10      # รัน 10 pods พร้อมกัน
  backoffLimit: 5
  
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: thumbnail-generator
        image: my-registry/thumbnail-gen:1.0
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: 2000m
            memory: 2Gi
```

### CronJob

```yaml
# cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
  namespace: ecommerce
spec:
  # Cron expression: นาที ชั่วโมง วัน เดือน วันในสัปดาห์
  schedule: "0 2 * * *"     # ทุกวัน 2:00 AM
  # schedule: "*/5 * * * *"  # ทุก 5 นาที
  # schedule: "0 0 * * 0"    # ทุกวันอาทิตย์ midnight
  # schedule: "0 9 * * 1-5"  # ทุกวันธรรมดา 9:00 AM
  
  # Timezone (Kubernetes 1.27+)
  timeZone: "Asia/Bangkok"
  
  # ถ้า schedule ซ้อนกัน ให้ทำอย่างไร
  concurrencyPolicy: Forbid  # Allow | Forbid | Replace
  
  # รักษา jobs ที่ success/fail ไว้กี่ตัว
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  
  # ถ้า missed เกิน X times ให้ยกเลิก
  startingDeadlineSeconds: 3600
  
  # ไม่รัน CronJob (suspend)
  suspend: false
  
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 86400  # ลบหลัง 1 วัน
      
      template:
        spec:
          restartPolicy: OnFailure
          
          containers:
          - name: report-generator
            image: my-registry/report-gen:1.0
            command: ["./generate-report.sh"]
            
            env:
            - name: REPORT_DATE
              value: "yesterday"
            - name: OUTPUT_FORMAT
              value: "pdf"
            - name: RECIPIENTS
              value: "ceo@example.com,cfo@example.com"
            
            envFrom:
            - configMapRef:
                name: web-app-config
            - secretRef:
                name: web-app-secrets
            
            resources:
              requests:
                cpu: 200m
                memory: 256Mi
              limits:
                cpu: 1000m
                memory: 1Gi
            
            volumeMounts:
            - name: reports
              mountPath: /reports
          
          volumes:
          - name: reports
            persistentVolumeClaim:
              claimName: reports-pvc

---
# Database Backup CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: ecommerce
spec:
  schedule: "0 3 * * *"  # 3:00 AM daily
  timeZone: "Asia/Bangkok"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  
  jobTemplate:
    spec:
      backoffLimit: 1
      
      template:
        spec:
          restartPolicy: OnFailure
          
          containers:
          - name: pg-backup
            image: postgres:15-alpine
            command:
            - /bin/sh
            - -c
            - |
              DATE=$(date +%Y%m%d_%H%M%S)
              BACKUP_FILE="/backups/ecommerce_${DATE}.sql.gz"
              
              pg_dump \
                -h ${PGHOST} \
                -U ${PGUSER} \
                -d ${PGDATABASE} \
                | gzip > ${BACKUP_FILE}
              
              echo "Backup created: ${BACKUP_FILE}"
              
              # Upload to S3
              aws s3 cp ${BACKUP_FILE} \
                s3://${S3_BACKUP_BUCKET}/postgres/$(basename ${BACKUP_FILE})
              
              # Delete local file
              rm ${BACKUP_FILE}
              
              # Keep only last 7 days in S3
              aws s3 ls s3://${S3_BACKUP_BUCKET}/postgres/ \
                | sort \
                | head -n -7 \
                | awk '{print $4}' \
                | xargs -I {} aws s3 rm s3://${S3_BACKUP_BUCKET}/postgres/{}
              
              echo "Backup completed successfully!"
            
            env:
            - name: PGHOST
              valueFrom:
                configMapKeyRef:
                  name: web-app-config
                  key: DB_HOST
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: web-app-secrets
                  key: DB_USERNAME
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: web-app-secrets
                  key: DB_PASSWORD
            - name: PGDATABASE
              value: ecommerce
            - name: S3_BACKUP_BUCKET
              value: my-backups
            
            resources:
              requests:
                cpu: 200m
                memory: 256Mi
              limits:
                cpu: 1000m
                memory: 1Gi
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backups
          
          volumes:
          - name: backup-storage
            emptyDir:
              sizeLimit: 5Gi
```

---

## 6. Ingress Controllers

Ingress จัดการ HTTP/HTTPS traffic เข้ามายัง cluster

```bash
# ติดตั้ง NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/aws/deploy.yaml

# สำหรับ minikube
minikube addons enable ingress

# ดู ingress controller
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx
```

### Ingress Resources

```yaml
# ingress.yaml

# 1. Basic Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: ecommerce
  annotations:
    # NGINX specific annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "20"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    
    # TLS redirect
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Certificate manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  
  tls:
  - hosts:
    - example.com
    - www.example.com
    - api.example.com
    secretName: example-com-tls
  
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-svc
            port:
              number: 80
      
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-svc
            port:
              number: 80

---
# 2. Ingress with Authentication
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: admin-ingress
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Admin Area"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - admin.example.com
    secretName: admin-tls
  rules:
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-svc
            port:
              number: 80

---
# 3. Canary Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-canary
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% traffic
    # หรือ canary by header
    # nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    # nginx.ingress.kubernetes.io/canary-by-header-value: "always"
    # หรือ canary by cookie
    # nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-v2-svc  # new version
            port:
              number: 80
```

### Cert-Manager (SSL Certificates)

```bash
# ติดตั้ง cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# ดู status
kubectl get pods -n cert-manager
```

```yaml
# clusterissuer.yaml
# Let's Encrypt Issuer
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
    - dns01:
        route53:
          region: ap-southeast-1
          hostedZoneID: Z1234567890
          accessKeyIDSecretRef:
            name: route53-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: route53-credentials
            key: secret-access-key

---
# Staging issuer (สำหรับทดสอบ)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
```

---

## 7. Horizontal Pod Autoscaler (HPA)

```yaml
# hpa.yaml

# 1. CPU-based HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: ecommerce
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
  # CPU metric
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale เมื่อ CPU > 70%
  
  # Memory metric
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric (Prometheus)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  
  # External metric
  - type: External
    external:
      metric:
        name: sqs_queue_length
        selector:
          matchLabels:
            queue: web-app-tasks
      target:
        type: Value
        value: "30"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # รอ 60s ก่อน scale up
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
      - type: Pods
        value: 4
        periodSeconds: 60
      selectPolicy: Max  # ใช้ policy ที่ scale มากกว่า
    
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาที ก่อน scale down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min  # ใช้ policy ที่ scale น้อยกว่า (conservative)
```

```bash
# ดู HPA
kubectl get hpa -n ecommerce
kubectl describe hpa web-app-hpa -n ecommerce

# Watch HPA
kubectl get hpa -n ecommerce -w

# Manual scale (override HPA ชั่วคราว)
kubectl scale deployment web-app --replicas=10 -n ecommerce

# Stress test เพื่อทดสอบ HPA
kubectl run stress-test --image=busybox -n ecommerce -- \
  sh -c "while true; do wget -q -O- http://web-app-svc/api/test; done"
```

### Metrics Server Installation

```bash
# ติดตั้ง Metrics Server (จำเป็นสำหรับ HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# สำหรับ minikube
minikube addons enable metrics-server

# ตรวจสอบ
kubectl top pods -n ecommerce
kubectl top nodes
```

---

## 8. PersistentVolumes และ PVCs

### Storage Classes

```yaml
# storageclass.yaml

# 1. AWS EBS StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  fsType: ext4
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:ap-southeast-1:123456789:key/xxx"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - ap-southeast-1a
    - ap-southeast-1b
    - ap-southeast-1c

---
# 2. GCP Persistent Disk StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-rwo
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-balanced
  replication-type: regional-pd
  disk-encryption-kms-key: projects/my-project/locations/global/keyRings/my-ring/cryptoKeys/my-key
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# 3. NFS StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.example.com
  share: /exports/k8s
  readOnly: "false"
mountOptions:
  - vers=4
  - minorversion=1
reclaimPolicy: Retain
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

### PersistentVolumes และ PVCs

```yaml
# pv-pvc.yaml

# 1. Static PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-data-pv
  labels:
    type: local
    app: postgres
spec:
  storageClassName: manual
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  reclaimPolicy: Retain
  
  # NFS
  nfs:
    server: nfs.example.com
    path: /exports/postgres
  
  # หรือ Local path
  # local:
  #   path: /data/postgres
  # nodeAffinity:
  #   required:
  #     nodeSelectorTerms:
  #     - matchExpressions:
  #       - key: kubernetes.io/hostname
  #         operator: In
  #         values:
  #         - node1

---
# 2. PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-pvc
  namespace: ecommerce
spec:
  storageClassName: fast-ssd  # ใช้ StorageClass ที่กำหนด
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  # ระบุ labels ของ PV ที่ต้องการ (สำหรับ static binding)
  # selector:
  #   matchLabels:
  #     app: postgres

---
# 3. Shared Storage PVC (ReadWriteMany)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-assets-pvc
  namespace: ecommerce
spec:
  storageClassName: nfs-storage
  accessModes:
    - ReadWriteMany  # หลาย pods สามารถ mount ได้พร้อมกัน
  resources:
    requests:
      storage: 50Gi
```

### Access Modes

```
ReadWriteOnce (RWO):  mount ได้ 1 node (อ่านและเขียน)
ReadOnlyMany (ROX):   mount ได้หลาย nodes (อ่านอย่างเดียว)
ReadWriteMany (RWX):  mount ได้หลาย nodes (อ่านและเขียน) - ต้องการ NFS/EFS/GlusterFS
ReadWriteOncePod (RWOP): mount ได้ 1 pod เท่านั้น (Kubernetes 1.22+)
```

---

## 9. Resource Requests/Limits (Detailed)

```yaml
# resource-management.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
      - name: app
        image: my-app:1.0
        
        resources:
          requests:
            # CPU: ขั้นต่ำที่ต้องการ (scheduler ใช้จัดสรร)
            cpu: "200m"      # 0.2 CPU = 200 millicores
            
            # Memory: ขั้นต่ำที่ต้องการ
            memory: "256Mi"  # 256 Megabytes
            
            # Ephemeral storage
            ephemeral-storage: "1Gi"
            
            # GPU (ถ้า node รองรับ)
            # nvidia.com/gpu: "1"
          
          limits:
            # CPU: ถ้าใช้เกินนี้ จะถูก throttle
            cpu: "1000m"     # 1 CPU core
            
            # Memory: ถ้าใช้เกินนี้ OOMKilled!
            memory: "512Mi"
            
            # Ephemeral storage limit
            ephemeral-storage: "2Gi"
      
      # Init containers ก็ต้องกำหนด resources
      initContainers:
      - name: init
        image: busybox
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
```

### Vertical Pod Autoscaler (VPA)

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
  namespace: ecommerce
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  
  updatePolicy:
    updateMode: "Auto"  # Off | Initial | Recreation | Auto
  
  resourcePolicy:
    containerPolicies:
    - containerName: web-app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2000m
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

---

## 10. Liveness/Readiness Probes (Advanced)

```yaml
# probes-advanced.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:1.0
        
        # =========================================
        # Startup Probe
        # =========================================
        # ตรวจสอบว่า app เริ่มต้นสำเร็จ
        # Liveness/Readiness จะไม่ทำงานจนกว่า Startup จะ pass
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
            httpHeaders:
            - name: X-Health-Check
              value: startup
          
          # รอสูงสุด 30 * 10 = 300 seconds
          failureThreshold: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
        
        # =========================================
        # Liveness Probe
        # =========================================
        # ถ้า fail: restart container
        livenessProbe:
          # HTTP check
          httpGet:
            path: /health/live
            port: 8080
            scheme: HTTP  # หรือ HTTPS
          
          initialDelaySeconds: 30    # รอหลัง container start
          periodSeconds: 10         # ตรวจสอบทุก 10 วินาที
          timeoutSeconds: 5         # timeout แต่ละครั้ง
          failureThreshold: 3       # restart หลัง fail 3 ครั้ง
          successThreshold: 1       # success เมื่อ pass 1 ครั้ง
        
        # =========================================
        # Readiness Probe
        # =========================================
        # ถ้า fail: ไม่รับ traffic (ไม่ restart)
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
          successThreshold: 2  # ต้อง pass 2 ครั้งติดต่อกัน
        
        # =========================================
        # Alternative Probe Types
        # =========================================
        
        # TCP Probe (เช็คว่า port เปิดอยู่)
        # livenessProbe:
        #   tcpSocket:
        #     port: 8080
        #   initialDelaySeconds: 15
        
        # Exec Probe (รัน command)
        # livenessProbe:
        #   exec:
        #     command:
        #     - /bin/sh
        #     - -c
        #     - "ps aux | grep -q '[m]y-app'"
        #   initialDelaySeconds: 15
        
        # gRPC Probe (Kubernetes 1.24+)
        # livenessProbe:
        #   grpc:
        #     port: 50051
        #     service: liveness
        #   initialDelaySeconds: 30
```

---

## 11. Deployment Strategies ใน K8s

### Recreate Strategy

```yaml
# recreate-strategy.yaml
spec:
  strategy:
    type: Recreate  # Kill pods ทั้งหมดก่อน สร้างใหม่
  # ข้อดี: ไม่มี version ปนกัน
  # ข้อเสีย: มี downtime
```

### Rolling Update

```yaml
# rolling-update-strategy.yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%      # เพิ่มได้ 25% ของ replicas
      maxUnavailable: 0  # ห้ามมี unavailable
```

### Blue/Green ด้วย Kubernetes Services

```yaml
# blue-green-k8s.yaml
---
# Blue Deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-blue
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      slot: blue
  template:
    metadata:
      labels:
        app: web-app
        slot: blue
        version: "1.0"
    spec:
      containers:
      - name: web-app
        image: my-registry/web-app:1.0

---
# Green Deployment (new)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-green
  namespace: ecommerce
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
      slot: green
  template:
    metadata:
      labels:
        app: web-app
        slot: green
        version: "2.0"
    spec:
      containers:
      - name: web-app
        image: my-registry/web-app:2.0

---
# Service pointing to blue (default)
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
  namespace: ecommerce
spec:
  selector:
    app: web-app
    slot: blue  # เปลี่ยนเป็น green เมื่อ switch
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Blue/Green switch script
#!/bin/bash
NAMESPACE="ecommerce"
SERVICE="web-app-svc"
CURRENT_SLOT=$(kubectl get service ${SERVICE} -n ${NAMESPACE} \
  -o jsonpath='{.spec.selector.slot}')

if [ "$CURRENT_SLOT" == "blue" ]; then
  NEW_SLOT="green"
else
  NEW_SLOT="blue"
fi

echo "Switching from $CURRENT_SLOT to $NEW_SLOT"

# รอให้ new deployment ready
kubectl rollout status deployment/web-app-${NEW_SLOT} -n ${NAMESPACE}

# Switch traffic
kubectl patch service ${SERVICE} -n ${NAMESPACE} \
  -p "{\"spec\":{\"selector\":{\"slot\":\"${NEW_SLOT}\"}}}"

echo "Traffic switched to ${NEW_SLOT}"
```

### Canary ด้วย Nginx Ingress

```yaml
# canary-ingress.yaml
---
# Stable version
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-stable
  namespace: ecommerce
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-v1-svc
            port:
              number: 80

---
# Canary version (10% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-canary
  namespace: ecommerce
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-v2-svc  # new version
            port:
              number: 80
```

---

## 12. แบบฝึกหัด

### แบบฝึกหัดที่ 1: StatefulSet Database

```yaml
# exercise-1-statefulset.yaml
# สร้าง Redis StatefulSet ที่มี:
# - 3 replicas
# - Persistent storage
# - Headless service สำหรับ DNS resolution
# - ConfigMap สำหรับ redis.conf
# - Password authentication

# 1. Redis Config
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-config
  namespace: exercise-1
data:
  redis.conf: |
    # TODO: เขียน redis.conf ที่ต้องการ
    # - enable persistence (appendonly yes)
    # - set maxmemory 256mb
    # - set maxmemory-policy allkeys-lru

# 2. Redis Secret
# TODO: สร้าง secret สำหรับ redis password

# 3. Headless Service
# TODO: สร้าง headless service

# 4. StatefulSet
# TODO: สร้าง statefulset ที่ใช้:
# - image: redis:7-alpine
# - volumeClaimTemplates: 1Gi
# - livenessProbe ด้วย redis-cli ping
# - readinessProbe
```

### แบบฝึกหัดที่ 2: CronJob Backup

```yaml
# exercise-2-cronjob.yaml
# สร้าง CronJob ที่ทำ daily backup:
# 1. รันทุกวัน 2:00 AM
# 2. Backup MySQL database
# 3. Upload ไปยัง S3
# 4. ส่ง notification ผ่าน Slack webhook

apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
  namespace: exercise-2
spec:
  # TODO: กำหนด schedule
  # TODO: กำหนด timezone
  # TODO: กำหนด concurrencyPolicy
  
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: mysql:8.0
            command:
            - /bin/sh
            - -c
            - |
              # TODO: เขียน backup script
              # 1. mysqldump
              # 2. compress
              # 3. upload to S3
              # 4. notify slack
              echo "TODO: implement backup"
```

### แบบฝึกหัดที่ 3: Ingress + TLS

```bash
# exercise-3: Setup HTTPS ingress

# 1. ติดตั้ง cert-manager บน minikube
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.yaml

# 2. สร้าง self-signed ClusterIssuer สำหรับ development
cat > self-signed-issuer.yaml << 'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF
kubectl apply -f self-signed-issuer.yaml

# 3. สร้าง deployment และ service
# TODO: สร้าง nginx deployment และ service

# 4. สร้าง Ingress พร้อม TLS
# TODO: สร้าง ingress ที่:
# - ใช้ cert-manager annotation
# - Enable TLS
# - Redirect HTTP to HTTPS

# 5. ทดสอบ
minikube tunnel &
curl -k https://myapp.example.com
```

### แบบฝึกหัดที่ 4: HPA Load Test

```bash
# exercise-4: Test HPA

# 1. สร้าง deployment
cat > hpa-demo.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hpa-demo-svc
spec:
  selector:
    app: hpa-demo
  ports:
  - port: 80
EOF

kubectl apply -f hpa-demo.yaml

# 2. สร้าง HPA
cat > hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

kubectl apply -f hpa.yaml

# 3. Load test
kubectl run load-gen \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hpa-demo-svc; done" &

# 4. Watch scaling
kubectl get hpa -w

# 5. Stop load test
kubectl delete pod load-gen

# 6. Watch scale down
kubectl get pods -w
```

### แบบฝึกหัดที่ 5: Complete Deployment Pipeline

```yaml
# exercise-5: Full deployment with all features
# สร้าง complete setup สำหรับ Node.js application ที่มี:
# - Deployment (3 replicas, rolling update)
# - ConfigMap (app config)
# - Secret (database credentials)
# - Service (ClusterIP)
# - Ingress (with TLS)
# - HPA (min 3, max 10)
# - PDB (minAvailable: 2)
# - CronJob (daily health report)

# สร้าง namespace
kubectl create namespace exercise-5

# TODO: สร้าง configmap.yaml
# - database url
# - redis url
# - log level
# - app port

# TODO: สร้าง secret.yaml
# - db password
# - jwt secret
# - api key

# TODO: สร้าง deployment.yaml
# - image: node:20-alpine
# - command: ["node", "server.js"]
# - readinessProbe (http /health)
# - livenessProbe (http /health)
# - resources (requests: 100m/128Mi, limits: 500m/256Mi)

# TODO: สร้าง service.yaml
# - ClusterIP
# - port 80 -> 3000

# TODO: สร้าง ingress.yaml
# - host: myapp.local
# - TLS

# TODO: สร้าง hpa.yaml
# - min: 3, max: 10
# - CPU 70%

# TODO: สร้าง pdb.yaml
# - minAvailable: 2

# TODO: สร้าง cronjob.yaml
# - รันทุก 1 ชั่วโมง
# - ส่ง HTTP request ไปยัง /health และ log ผลลัพธ์

# Apply ทั้งหมด
# kubectl apply -f exercise-5/ -n exercise-5

# ตรวจสอบ
# kubectl get all -n exercise-5
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้ advanced Kubernetes topics:

1. **Complete Deployment YAML**: Best practices สำหรับ production
2. **Rolling Updates**: อัพเดต pods ทีละตัวโดยไม่มี downtime
3. **StatefulSets**: สำหรับ stateful workloads เช่น databases
4. **DaemonSets**: รัน pod บนทุก node
5. **Jobs/CronJobs**: สำหรับ batch workloads
6. **Ingress**: HTTP routing พร้อม TLS
7. **HPA**: Auto-scaling ตาม metrics
8. **PersistentVolumes**: Persistent storage สำหรับ applications
9. **Deployment Strategies**: Rolling, Blue/Green, Canary ใน K8s

**Production Checklist:**
- กำหนด resource requests/limits เสมอ
- ใช้ readinessProbe ป้องกัน traffic ไปยัง pod ที่ยังไม่พร้อม
- ใช้ livenessProbe เพื่อ restart pods ที่ stuck
- ตั้ง PDB เพื่อป้องกัน downtime ระหว่าง maintenance
- ใช้ anti-affinity กระจาย pods ไปต่าง nodes
- ไม่รัน containers เป็น root
- ใช้ readOnlyRootFilesystem: true เมื่อเป็นไปได้
- ตั้ง terminationGracePeriodSeconds ให้เพียงพอ

---

## แหล่งอ้างอิง

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [Cert-Manager](https://cert-manager.io/docs/)
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [Kubernetes Patterns](https://k8spatterns.io/)
- [Production Best Practices](https://learnk8s.io/production-best-practices)

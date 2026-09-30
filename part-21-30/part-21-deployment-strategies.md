# Part 21: Deployment Strategies

## บทนำ

ในโลกของ DevOps และ CI/CD การ deploy แอปพลิเคชันไปยัง production เป็นขั้นตอนที่สำคัญและต้องการความระมัดระวัง กลยุทธ์การ deploy ที่ดีจะช่วยให้เราสามารถ:

- ลด downtime ให้น้อยที่สุด หรือเป็นศูนย์ (zero downtime)
- ลดความเสี่ยงจากการ deploy version ใหม่
- สามารถ rollback ได้อย่างรวดเร็วหากเกิดปัญหา
- ทดสอบกับ traffic จริงในสัดส่วนที่ควบคุมได้

ในบทนี้เราจะเรียนรู้กลยุทธ์การ deploy หลัก ๆ 5 รูปแบบ พร้อม implementation ด้วย Kubernetes, AWS, GCP และ Azure

---

## สารบัญ

1. Rolling Deployment
2. Blue/Green Deployment
3. Canary Release
4. A/B Testing
5. Shadow Deployment
6. Feature Flags Integration
7. Health Checks และ Rollback Procedures
8. Smoke Tests
9. Deployment Pipeline Examples
10. แบบฝึกหัด

---

## 1. Rolling Deployment

### Rolling Deployment คืออะไร?

Rolling Deployment เป็นกลยุทธ์ที่ค่อย ๆ อัพเดต instance ของแอปพลิเคชันทีละส่วน โดยไม่ต้อง shut down ทั้งหมดพร้อมกัน

```
Before:  [v1][v1][v1][v1]
Step 1:  [v2][v1][v1][v1]
Step 2:  [v2][v2][v1][v1]
Step 3:  [v2][v2][v2][v1]
After:   [v2][v2][v2][v2]
```

### ข้อดีของ Rolling Deployment

- **Zero downtime**: แอปยังทำงานได้ตลอดเวลา
- **ใช้ทรัพยากรน้อย**: ไม่ต้องมี infrastructure เพิ่มเติม
- **ง่ายต่อการ implement**: Kubernetes รองรับโดย default
- **ค่อยๆ roll out**: สามารถตรวจสอบปัญหาได้ระหว่างกระบวนการ

### ข้อเสียของ Rolling Deployment

- **ช่วง transition**: มี v1 และ v2 ทำงานพร้อมกัน อาจเกิดปัญหา compatibility
- **Rollback ช้า**: ต้อง rollback ทีละ instance
- **ตรวจจับปัญหายาก**: ปัญหาอาจเจอเมื่อ deploy ไปหลาย instance แล้ว

### Implementation ด้วย Kubernetes

```yaml
# rolling-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    version: "2.0"
spec:
  replicas: 4
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # จำนวน pod เพิ่มเติมที่อนุญาตระหว่าง update
      maxUnavailable: 0    # จำนวน pod ที่ไม่พร้อมใช้งานได้
  template:
    metadata:
      labels:
        app: my-app
        version: "2.0"
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
      terminationGracePeriodSeconds: 30
```

### การ Monitor Rolling Deployment

```bash
# ดู status ของ deployment
kubectl rollout status deployment/my-app -n production

# ดู history ของ deployment
kubectl rollout history deployment/my-app -n production

# ดู events
kubectl describe deployment my-app -n production

# ดู pods ที่กำลัง update
kubectl get pods -n production -w -l app=my-app
```

### Rolling Deployment บน AWS (ECS)

```json
{
  "deploymentConfiguration": {
    "deploymentCircuitBreaker": {
      "enable": true,
      "rollback": true
    },
    "maximumPercent": 125,
    "minimumHealthyPercent": 75
  },
  "deploymentController": {
    "type": "ECS"
  }
}
```

```bash
# AWS CLI สำหรับ ECS Rolling Update
aws ecs update-service \
  --cluster production-cluster \
  --service my-app-service \
  --task-definition my-app:2 \
  --deployment-configuration "maximumPercent=125,minimumHealthyPercent=75" \
  --region ap-southeast-1

# ดู status
aws ecs describe-services \
  --cluster production-cluster \
  --services my-app-service \
  --query 'services[0].deployments'
```

---

## 2. Blue/Green Deployment

### Blue/Green Deployment คืออะไร?

Blue/Green Deployment เป็นกลยุทธ์ที่มี environment สองชุด (Blue = current, Green = new) โดย deploy version ใหม่ไปยัง Green environment ก่อน แล้วค่อย switch traffic ทั้งหมด

```
                    Load Balancer
                         |
              +----------+----------+
              |                     |
         [Blue: v1]            [Green: v2]
         (production)          (staging/new)
              |                     |
         100% traffic           0% traffic
              
    After switch:
         [Blue: v1]            [Green: v2]
         (old/standby)         (production)
              |                     |
          0% traffic           100% traffic
```

### ข้อดีของ Blue/Green Deployment

- **Instant rollback**: เพียง switch traffic กลับไป Blue
- **Zero downtime**: switch เกิดขึ้นทันที
- **Testing ก่อน production**: ทดสอบ Green environment ก่อน switch
- **Separation ชัดเจน**: ไม่มี version ผสมกัน

### ข้อเสียของ Blue/Green Deployment

- **ค่าใช้จ่ายสูง**: ต้องมี infrastructure สองชุด
- **Database migration ซับซ้อน**: ต้องจัดการ database schema ที่รองรับทั้งสอง version
- **State management**: Session, cache อาจมีปัญหาเมื่อ switch

### Implementation ด้วย Kubernetes

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
  namespace: production
  labels:
    app: my-app
    version: blue
    release: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
        release: "1.0"
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:1.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
  namespace: production
  labels:
    app: my-app
    version: green
    release: "2.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
        release: "2.0"
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:2.0
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# service.yaml - initially points to blue
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: production
spec:
  selector:
    app: my-app
    version: blue  # เปลี่ยนเป็น green เมื่อต้องการ switch
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
# สคริปต์ Blue/Green Switch
#!/bin/bash

NAMESPACE="production"
SERVICE="my-app-service"
NEW_VERSION="green"

echo "Switching traffic to ${NEW_VERSION}..."

# ตรวจสอบว่า Green pods พร้อมทั้งหมด
READY_PODS=$(kubectl get pods -n ${NAMESPACE} \
  -l app=my-app,version=${NEW_VERSION} \
  --field-selector=status.phase=Running \
  -o json | jq '.items | length')

TOTAL_PODS=$(kubectl get deployment my-app-${NEW_VERSION} \
  -n ${NAMESPACE} \
  -o jsonpath='{.spec.replicas}')

if [ "${READY_PODS}" != "${TOTAL_PODS}" ]; then
  echo "ERROR: Not all ${NEW_VERSION} pods are ready (${READY_PODS}/${TOTAL_PODS})"
  exit 1
fi

# Switch traffic
kubectl patch service ${SERVICE} -n ${NAMESPACE} \
  -p '{"spec":{"selector":{"version":"'${NEW_VERSION}'"}}}'

echo "Traffic switched to ${NEW_VERSION} successfully!"

# ตรวจสอบ
kubectl describe service ${SERVICE} -n ${NAMESPACE} | grep Selector
```

### Blue/Green บน AWS Route 53 + ECS

```python
# blue_green_switch.py
import boto3
import time
import sys

def blue_green_switch(cluster_name, old_service, new_service, hosted_zone_id, record_name):
    ecs = boto3.client('ecs', region_name='ap-southeast-1')
    route53 = boto3.client('route53')
    elb = boto3.client('elbv2', region_name='ap-southeast-1')
    
    print(f"Starting Blue/Green deployment switch...")
    
    # รอให้ new service healthy
    print(f"Waiting for {new_service} to be stable...")
    waiter = ecs.get_waiter('services_stable')
    waiter.wait(
        cluster=cluster_name,
        services=[new_service],
        WaiterConfig={'Delay': 15, 'MaxAttempts': 40}
    )
    
    # ดึง ALB target group ของ new service
    response = ecs.describe_services(
        cluster=cluster_name,
        services=[new_service]
    )
    new_lb = response['services'][0]['loadBalancers'][0]
    new_target_group_arn = new_lb['targetGroupArn']
    
    # ดึง ALB ARN
    tg_response = elb.describe_target_groups(
        TargetGroupArns=[new_target_group_arn]
    )
    load_balancer_arns = tg_response['TargetGroups'][0]['LoadBalancerArns']
    
    lb_response = elb.describe_load_balancers(
        LoadBalancerArns=load_balancer_arns
    )
    lb_dns = lb_response['LoadBalancers'][0]['DNSName']
    
    # Update Route 53
    print(f"Updating Route 53 to point to {lb_dns}...")
    route53.change_resource_record_sets(
        HostedZoneId=hosted_zone_id,
        ChangeBatch={
            'Changes': [{
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': record_name,
                    'Type': 'CNAME',
                    'TTL': 60,
                    'ResourceRecords': [{'Value': lb_dns}]
                }
            }]
        }
    )
    
    print("Blue/Green switch completed successfully!")
    print(f"Traffic now routing to: {lb_dns}")
    
    return True

if __name__ == '__main__':
    blue_green_switch(
        cluster_name='production-cluster',
        old_service='my-app-blue',
        new_service='my-app-green',
        hosted_zone_id='Z1234567890ABC',
        record_name='api.example.com'
    )
```

### Blue/Green บน GCP (Cloud Run)

```bash
# Deploy Green version
gcloud run deploy my-app-green \
  --image gcr.io/my-project/my-app:2.0 \
  --region asia-southeast1 \
  --no-traffic \
  --tag green

# ทดสอบ Green version
curl https://green---my-app-green-xxx-as.a.run.app/health

# Switch traffic (gradual)
gcloud run services update-traffic my-app \
  --to-tags green=100 \
  --region asia-southeast1

# หรือ split traffic
gcloud run services update-traffic my-app \
  --to-tags green=50,blue=50 \
  --region asia-southeast1
```

---

## 3. Canary Release

### Canary Release คืออะไร?

Canary Release เป็นกลยุทธ์ที่ค่อยๆ เพิ่ม traffic ไปยัง version ใหม่ทีละน้อย โดยเริ่มจากสัดส่วนเล็กน้อย (เช่น 5%) แล้วค่อยๆ เพิ่มขึ้นตาม metrics

```
Phase 1:  [v2: 5%]  [v1: 95%]  -> Monitor
Phase 2:  [v2: 25%] [v1: 75%]  -> Monitor
Phase 3:  [v2: 50%] [v1: 50%]  -> Monitor
Phase 4:  [v2: 100%][v1: 0%]   -> Complete
```

ชื่อมาจาก "canary in a coal mine" - นกขมิ้นที่นำลงไปในเหมืองถ่านหิน เพื่อตรวจจับก๊าซพิษก่อนที่จะถึงคนงาน

### ข้อดีของ Canary Release

- **ความเสี่ยงต่ำ**: เริ่มต้นด้วย traffic น้อย
- **Early detection**: ตรวจจับปัญหาก่อนที่จะกระทบผู้ใช้ทั้งหมด
- **Real user testing**: ทดสอบกับ traffic จริงในสภาพแวดล้อม production
- **Gradual rollout**: ค่อยๆ เพิ่มความมั่นใจ

### ข้อเสียของ Canary Release

- **ซับซ้อนกว่า Rolling**: ต้องการ traffic splitting
- **Monitoring ต้องดี**: ต้องมี metrics ที่ชัดเจน
- **ระยะเวลานาน**: ใช้เวลามากกว่า full deployment

### Implementation ด้วย Kubernetes + Argo Rollouts

```bash
# ติดตั้ง Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# ติดตั้ง kubectl plugin
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

```yaml
# canary-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app-rollout
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:2.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
  strategy:
    canary:
      canaryService: my-app-canary-svc
      stableService: my-app-stable-svc
      trafficRouting:
        nginx:
          stableIngress: my-app-ingress
      steps:
      - setWeight: 5        # 5% traffic ไปยัง canary
      - pause: {duration: 2m}  # รอ 2 นาที
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: my-app-canary-svc
      - setWeight: 25       # 25% traffic
      - pause: {duration: 5m}
      - setWeight: 50       # 50% traffic
      - pause: {duration: 5m}
      - setWeight: 75       # 75% traffic
      - pause: {duration: 5m}
      # 100% เมื่อ steps ทั้งหมดผ่าน
---
# canary-services.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-stable-svc
  namespace: production
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-canary-svc
  namespace: production
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
---
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus-server.monitoring:80
        query: |
          sum(irate(http_requests_total{
            service="{{args.service-name}}",
            status=~"2.."
          }[5m])) /
          sum(irate(http_requests_total{
            service="{{args.service-name}}"
          }[5m]))
```

```bash
# Canary Commands
# ดู status
kubectl argo rollouts get rollout my-app-rollout -n production --watch

# Promote canary (เพิ่ม weight ขั้นต่อไป)
kubectl argo rollouts promote my-app-rollout -n production

# Abort canary (rollback)
kubectl argo rollouts abort my-app-rollout -n production

# Retry หลัง abort
kubectl argo rollouts retry rollout my-app-rollout -n production
```

### Canary บน Istio Service Mesh

```yaml
# istio-virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-vs
  namespace: production
spec:
  hosts:
  - my-app-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: my-app-canary
        port:
          number: 80
  - route:
    - destination:
        host: my-app-stable
        port:
          number: 80
      weight: 95
    - destination:
        host: my-app-canary
        port:
          number: 80
      weight: 5
```

---

## 4. A/B Testing

### A/B Testing คืออะไร?

A/B Testing คล้ายกับ Canary แต่มีเป้าหมายต่างกัน - เราส่ง traffic กลุ่มต่างๆ ไปยัง version ที่ต่างกัน เพื่อวัดผลทางธุรกิจ (conversion rate, user engagement, etc.)

```
Group A (50%): ผู้ใช้เห็น Feature ใหม่ (สีปุ่ม, layout, ฟีเจอร์)
Group B (50%): ผู้ใช้เห็น version เดิม

วัดผล: Conversion rate, Click-through rate, Session duration
```

### ความแตกต่างระหว่าง Canary และ A/B Testing

| | Canary | A/B Testing |
|---|---|---|
| เป้าหมาย | ลดความเสี่ยง deployment | วัดผลทางธุรกิจ |
| Duration | ชั่วคราว จนกว่าจะ full deploy | ระยะยาวจนกว่าจะมีข้อมูลเพียงพอ |
| Routing | ตาม % traffic | ตาม user segment |
| Metric | Error rate, latency | Business KPIs |

### Implementation ด้วย Kubernetes + Nginx Ingress

```yaml
# ab-testing-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress-b
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-cookie: "experiment_group"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Experiment-Group"
    nginx.ingress.kubernetes.io/canary-by-header-value: "B"
    nginx.ingress.kubernetes.io/canary-weight: "50"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-b
            port:
              number: 80
---
# app-a-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-a
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      variant: a
  template:
    metadata:
      labels:
        app: my-app
        variant: a
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:feature-a
        env:
        - name: VARIANT
          value: "A"
        - name: FEATURE_NEW_UI
          value: "false"
---
# app-b-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-b
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      variant: b
  template:
    metadata:
      labels:
        app: my-app
        variant: b
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:feature-b
        env:
        - name: VARIANT
          value: "B"
        - name: FEATURE_NEW_UI
          value: "true"
```

### A/B Testing ด้วย Feature Flags (LaunchDarkly-style)

```python
# ab_testing_service.py
import hashlib
import json
from typing import Optional

class ABTestingService:
    def __init__(self, experiments: dict):
        self.experiments = experiments
    
    def get_variant(self, experiment_name: str, user_id: str) -> str:
        """
        คืนค่า variant สำหรับ user โดยใช้ consistent hashing
        ทำให้ user เดิมได้ variant เดิมเสมอ
        """
        if experiment_name not in self.experiments:
            return "control"
        
        experiment = self.experiments[experiment_name]
        
        # Hash user_id เพื่อให้ consistent
        hash_value = int(hashlib.md5(
            f"{experiment_name}:{user_id}".encode()
        ).hexdigest(), 16)
        
        percentage = hash_value % 100
        
        # กำหนด variant ตาม percentage
        cumulative = 0
        for variant, weight in experiment['variants'].items():
            cumulative += weight
            if percentage < cumulative:
                return variant
        
        return "control"
    
    def track_event(self, experiment_name: str, user_id: str, 
                    variant: str, event: str, value: float = 1.0):
        """บันทึก event สำหรับการวิเคราะห์"""
        tracking_data = {
            "experiment": experiment_name,
            "user_id": user_id,
            "variant": variant,
            "event": event,
            "value": value,
            "timestamp": "2024-01-01T00:00:00Z"
        }
        # บันทึกไปยัง analytics system (Mixpanel, Amplitude, etc.)
        print(f"Tracking: {json.dumps(tracking_data)}")


# ตัวอย่างการใช้งาน
experiments = {
    "checkout_button_color": {
        "variants": {
            "control": 50,  # ปุ่มสีเดิม (50%)
            "blue": 25,     # ปุ่มสีน้ำเงิน (25%)
            "orange": 25    # ปุ่มสีส้ม (25%)
        }
    },
    "new_checkout_flow": {
        "variants": {
            "control": 80,  # flow เดิม (80%)
            "new": 20       # flow ใหม่ (20%)
        }
    }
}

ab_service = ABTestingService(experiments)

# สำหรับ user แต่ละคน
user_id = "user-12345"
variant = ab_service.get_variant("checkout_button_color", user_id)
print(f"User {user_id} sees variant: {variant}")

# บันทึก conversion
ab_service.track_event(
    "checkout_button_color", 
    user_id, 
    variant, 
    "purchase_completed", 
    value=99.99
)
```

---

## 5. Shadow Deployment

### Shadow Deployment คืออะไร?

Shadow Deployment คือการส่ง request จริงไปยัง version ใหม่พร้อมกับ version เดิม แต่ user จะเห็นเฉพาะ response จาก version เดิม เราใช้ version ใหม่เพื่อทดสอบประสิทธิภาพและความถูกต้อง

```
User Request
     |
     v
 Load Balancer
   /        \
  /          \
v1 (live)  v2 (shadow)
  |          |
response   response
(sent to   (discarded/
  user)      analyzed)
```

### ข้อดีของ Shadow Deployment

- **ความปลอดภัย 100%**: User ไม่เคยเห็น response จาก version ใหม่
- **ทดสอบด้วย traffic จริง**: ได้รับ load จริง request จริง
- **เหมาะสำหรับ ML Models**: ทดสอบ model ใหม่กับ production data

### ข้อเสียของ Shadow Deployment

- **Side effects**: ถ้า v2 มี write operations จะเป็นปัญหา
- **ค่าใช้จ่าย**: ต้องรัน infrastructure สองชุด
- **ซับซ้อน**: ต้องจัดการ response comparison

### Implementation ด้วย Istio

```yaml
# shadow-deployment.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app-shadow-vs
  namespace: production
spec:
  hosts:
  - my-app
  http:
  - route:
    - destination:
        host: my-app-v1
        port:
          number: 80
      weight: 100
    mirror:
      host: my-app-v2
      port:
        number: 80
    mirrorPercentage:
      value: 100.0  # Shadow 100% ของ traffic
```

### Shadow Deployment สำหรับ ML Model

```python
# shadow_ml_service.py
import asyncio
import aiohttp
import time
import logging
from dataclasses import dataclass

@dataclass
class ShadowResult:
    production_response: dict
    shadow_response: dict
    production_latency: float
    shadow_latency: float
    responses_match: bool

class ShadowMLService:
    def __init__(self, production_url: str, shadow_url: str):
        self.production_url = production_url
        self.shadow_url = shadow_url
        self.logger = logging.getLogger(__name__)
    
    async def predict(self, features: dict) -> dict:
        """
        ส่ง request ไปยัง production และ shadow พร้อมกัน
        คืนค่าเฉพาะ production response
        """
        start_time = time.time()
        
        async with aiohttp.ClientSession() as session:
            # ส่งทั้งสอง request พร้อมกัน
            prod_task = asyncio.create_task(
                self._call_model(session, self.production_url, features)
            )
            shadow_task = asyncio.create_task(
                self._call_model(session, self.shadow_url, features)
            )
            
            # รอผลจากทั้งสอง (ไม่รอ shadow ถ้า timeout)
            prod_response, prod_latency = await prod_task
            
            try:
                shadow_response, shadow_latency = await asyncio.wait_for(
                    shadow_task, timeout=5.0
                )
                
                # เปรียบเทียบผลลัพธ์
                match = self._compare_responses(prod_response, shadow_response)
                
                result = ShadowResult(
                    production_response=prod_response,
                    shadow_response=shadow_response,
                    production_latency=prod_latency,
                    shadow_latency=shadow_latency,
                    responses_match=match
                )
                
                # บันทึกสำหรับการวิเคราะห์
                await self._log_shadow_result(result)
                
            except asyncio.TimeoutError:
                self.logger.warning("Shadow model timed out")
        
        return prod_response
    
    async def _call_model(self, session, url: str, features: dict):
        start = time.time()
        async with session.post(
            f"{url}/predict",
            json=features,
            timeout=aiohttp.ClientTimeout(total=10)
        ) as response:
            result = await response.json()
            latency = time.time() - start
            return result, latency
    
    def _compare_responses(self, prod: dict, shadow: dict) -> bool:
        """เปรียบเทียบ prediction output"""
        prod_label = prod.get('prediction')
        shadow_label = shadow.get('prediction')
        return prod_label == shadow_label
    
    async def _log_shadow_result(self, result: ShadowResult):
        """บันทึก metrics ไปยัง monitoring system"""
        self.logger.info(
            f"Shadow comparison - "
            f"match: {result.responses_match}, "
            f"prod_latency: {result.production_latency:.3f}s, "
            f"shadow_latency: {result.shadow_latency:.3f}s"
        )
```

---

## 6. Feature Flags Integration

### Feature Flags คืออะไร?

Feature Flags (หรือ Feature Toggles) คือ mechanism ที่ช่วยให้เราสามารถ enable/disable features ได้โดยไม่ต้อง deploy code ใหม่

### ประเภทของ Feature Flags

1. **Release Toggles**: Enable feature ใหม่เมื่อพร้อม
2. **Experiment Toggles**: A/B Testing
3. **Ops Toggles**: Kill switch สำหรับ feature ที่มีปัญหา
4. **Permission Toggles**: เปิดใช้เฉพาะ user บางกลุ่ม

### Implementation ใน Python

```python
# feature_flags.py
import os
import json
from enum import Enum
from typing import Optional, Any
from dataclasses import dataclass

class FlagEvaluationStrategy(Enum):
    BOOLEAN = "boolean"
    PERCENTAGE = "percentage"
    USER_LIST = "user_list"
    USER_ATTRIBUTE = "user_attribute"

@dataclass
class FeatureFlag:
    name: str
    enabled: bool
    strategy: FlagEvaluationStrategy
    config: dict

class FeatureFlagService:
    def __init__(self, flags_config: dict):
        self.flags = {
            name: FeatureFlag(
                name=name,
                enabled=config.get('enabled', False),
                strategy=FlagEvaluationStrategy(
                    config.get('strategy', 'boolean')
                ),
                config=config
            )
            for name, config in flags_config.items()
        }
    
    def is_enabled(self, flag_name: str, 
                   user_id: Optional[str] = None,
                   user_attributes: Optional[dict] = None) -> bool:
        """ตรวจสอบว่า feature flag เปิดอยู่หรือไม่"""
        if flag_name not in self.flags:
            return False
        
        flag = self.flags[flag_name]
        
        if not flag.enabled:
            return False
        
        strategy = flag.strategy
        
        if strategy == FlagEvaluationStrategy.BOOLEAN:
            return flag.enabled
        
        elif strategy == FlagEvaluationStrategy.PERCENTAGE:
            if not user_id:
                return False
            percentage = flag.config.get('percentage', 0)
            import hashlib
            hash_val = int(hashlib.md5(
                f"{flag_name}:{user_id}".encode()
            ).hexdigest(), 16)
            return (hash_val % 100) < percentage
        
        elif strategy == FlagEvaluationStrategy.USER_LIST:
            if not user_id:
                return False
            allowed_users = flag.config.get('allowed_users', [])
            return user_id in allowed_users
        
        elif strategy == FlagEvaluationStrategy.USER_ATTRIBUTE:
            if not user_attributes:
                return False
            required_attrs = flag.config.get('required_attributes', {})
            for attr, value in required_attrs.items():
                if user_attributes.get(attr) != value:
                    return False
            return True
        
        return False
    
    def get_variant(self, flag_name: str, user_id: str) -> str:
        """ดึง variant สำหรับ multivariate flag"""
        if flag_name not in self.flags:
            return "control"
        
        flag = self.flags[flag_name]
        variants = flag.config.get('variants', {})
        
        if not variants:
            return "treatment" if self.is_enabled(flag_name, user_id) else "control"
        
        import hashlib
        hash_val = int(hashlib.md5(
            f"{flag_name}:{user_id}".encode()
        ).hexdigest(), 16)
        percentage = hash_val % 100
        
        cumulative = 0
        for variant, weight in variants.items():
            cumulative += weight
            if percentage < cumulative:
                return variant
        
        return "control"


# ตัวอย่าง config
flags_config = {
    "new_checkout_ui": {
        "enabled": True,
        "strategy": "percentage",
        "percentage": 20  # เปิดสำหรับ 20% ของ users
    },
    "dark_mode": {
        "enabled": True,
        "strategy": "user_attribute",
        "required_attributes": {
            "subscription": "premium",
            "beta_tester": True
        }
    },
    "emergency_maintenance": {
        "enabled": False,  # Kill switch
        "strategy": "boolean"
    },
    "new_recommendation_engine": {
        "enabled": True,
        "strategy": "user_list",
        "allowed_users": ["admin-1", "tester-1", "tester-2"]
    }
}

ff_service = FeatureFlagService(flags_config)

# ใช้งานใน application
user_id = "user-12345"
user_attrs = {"subscription": "premium", "beta_tester": True}

if ff_service.is_enabled("new_checkout_ui", user_id=user_id):
    # แสดง UI ใหม่
    print("Showing new checkout UI")
else:
    # แสดง UI เดิม
    print("Showing old checkout UI")

if ff_service.is_enabled("dark_mode", user_attributes=user_attrs):
    print("Dark mode enabled")
```

### Feature Flags ใน Kubernetes ConfigMap

```yaml
# feature-flags-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
  namespace: production
data:
  flags.json: |
    {
      "new_checkout_ui": {
        "enabled": true,
        "strategy": "percentage",
        "percentage": 20
      },
      "new_payment_processor": {
        "enabled": false,
        "strategy": "boolean"
      },
      "dark_mode": {
        "enabled": true,
        "strategy": "user_attribute",
        "required_attributes": {
          "subscription": "premium"
        }
      }
    }
---
# ใช้ใน pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: my-registry/my-app:2.0
        volumeMounts:
        - name: feature-flags
          mountPath: /app/config/flags.json
          subPath: flags.json
      volumes:
      - name: feature-flags
        configMap:
          name: feature-flags
```

---

## 7. Health Checks และ Rollback Procedures

### Health Check Types

```yaml
# health-checks.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:2.0
        
        # Startup Probe: ตรวจสอบว่า container เริ่มต้นสำเร็จ
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30    # รอ 30 * 10s = 5 นาที
          periodSeconds: 10
        
        # Liveness Probe: ตรวจสอบว่า container ยังทำงานอยู่
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30  # รอหลังจาก startup
          periodSeconds: 10
          failureThreshold: 3      # restart หลังจาก fail 3 ครั้ง
          timeoutSeconds: 5
        
        # Readiness Probe: ตรวจสอบว่าพร้อมรับ traffic
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
          timeoutSeconds: 3
```

### Health Check Endpoint Implementation

```python
# health_check.py (FastAPI example)
from fastapi import FastAPI, Response
import psycopg2
import redis
import httpx
from typing import Dict, Any

app = FastAPI()

async def check_database() -> Dict[str, Any]:
    try:
        conn = psycopg2.connect(
            host="db-host",
            database="mydb",
            user="user",
            password="password",
            connect_timeout=3
        )
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        conn.close()
        return {"status": "healthy"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

async def check_redis() -> Dict[str, Any]:
    try:
        r = redis.Redis(host='redis-host', port=6379, socket_timeout=3)
        r.ping()
        return {"status": "healthy"}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

async def check_external_api() -> Dict[str, Any]:
    try:
        async with httpx.AsyncClient(timeout=3) as client:
            response = await client.get("https://api.external.com/health")
            if response.status_code == 200:
                return {"status": "healthy"}
            return {"status": "degraded", "code": response.status_code}
    except Exception as e:
        return {"status": "unhealthy", "error": str(e)}

@app.get("/health/startup")
async def startup_health():
    """ตรวจสอบว่า app เริ่มต้นสำเร็จ"""
    return {"status": "started"}

@app.get("/health/live")
async def liveness():
    """ตรวจสอบว่า process ยังทำงานอยู่"""
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness(response: Response):
    """ตรวจสอบว่าพร้อมรับ request"""
    checks = {
        "database": await check_database(),
        "redis": await check_redis(),
    }
    
    # กำหนด status code ตามผลการตรวจสอบ
    if all(c["status"] == "healthy" for c in checks.values()):
        return {"status": "ready", "checks": checks}
    else:
        response.status_code = 503
        return {"status": "not_ready", "checks": checks}

@app.get("/health/full")
async def full_health_check(response: Response):
    """Full health check รวม external dependencies"""
    checks = {
        "database": await check_database(),
        "redis": await check_redis(),
        "external_api": await check_external_api(),
    }
    
    unhealthy = [k for k, v in checks.items() if v["status"] == "unhealthy"]
    
    if unhealthy:
        response.status_code = 503
        return {
            "status": "unhealthy",
            "unhealthy_components": unhealthy,
            "checks": checks
        }
    
    return {"status": "healthy", "checks": checks}
```

### Rollback Procedures

```bash
#!/bin/bash
# rollback.sh - สคริปต์ rollback อัตโนมัติ

set -euo pipefail

DEPLOYMENT="$1"
NAMESPACE="${2:-production}"

echo "=== Starting Rollback ==="
echo "Deployment: ${DEPLOYMENT}"
echo "Namespace: ${NAMESPACE}"

# ดู history
echo "=== Deployment History ==="
kubectl rollout history deployment/${DEPLOYMENT} -n ${NAMESPACE}

# ดู current version
CURRENT_VERSION=$(kubectl get deployment ${DEPLOYMENT} -n ${NAMESPACE} \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "Current version: ${CURRENT_VERSION}"

# Rollback ไปยัง version ก่อนหน้า
echo "Rolling back..."
kubectl rollout undo deployment/${DEPLOYMENT} -n ${NAMESPACE}

# รอให้ rollback เสร็จสมบูรณ์
echo "Waiting for rollback to complete..."
kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE} --timeout=5m

# ตรวจสอบ
NEW_VERSION=$(kubectl get deployment ${DEPLOYMENT} -n ${NAMESPACE} \
  -o jsonpath='{.spec.template.spec.containers[0].image}')
echo "Rolled back to: ${NEW_VERSION}"

# ตรวจสอบ pods
echo "=== Pod Status ==="
kubectl get pods -n ${NAMESPACE} -l app=${DEPLOYMENT}

# Run smoke tests
echo "=== Running Smoke Tests ==="
./smoke_test.sh ${DEPLOYMENT} ${NAMESPACE}

if [ $? -eq 0 ]; then
  echo "=== Rollback SUCCESSFUL ==="
  
  # ส่ง notification
  curl -X POST "${SLACK_WEBHOOK_URL}" \
    -H 'Content-type: application/json' \
    --data "{
      \"text\": \"✅ Rollback completed for ${DEPLOYMENT}. Rolled back from ${CURRENT_VERSION} to ${NEW_VERSION}\"
    }"
else
  echo "=== Smoke tests FAILED after rollback ==="
  exit 1
fi
```

### Automated Rollback ด้วย Prometheus Metrics

```python
# auto_rollback.py
import time
import subprocess
import requests
from dataclasses import dataclass
from typing import Optional

@dataclass
class DeploymentMetrics:
    error_rate: float
    p99_latency: float
    availability: float

class AutoRollback:
    def __init__(self, 
                 deployment_name: str,
                 namespace: str,
                 prometheus_url: str,
                 thresholds: dict):
        self.deployment = deployment_name
        self.namespace = namespace
        self.prometheus_url = prometheus_url
        self.thresholds = thresholds
    
    def get_metrics(self) -> DeploymentMetrics:
        """ดึง metrics จาก Prometheus"""
        
        # Error rate
        error_rate_query = f"""
            sum(rate(http_requests_total{{
                deployment="{self.deployment}",
                status=~"5.."
            }}[5m])) /
            sum(rate(http_requests_total{{
                deployment="{self.deployment}"
            }}[5m]))
        """
        
        # P99 Latency
        p99_query = f"""
            histogram_quantile(0.99, 
                rate(http_request_duration_seconds_bucket{{
                    deployment="{self.deployment}"
                }}[5m])
            )
        """
        
        error_rate = self._query_prometheus(error_rate_query) or 0
        p99_latency = self._query_prometheus(p99_query) or 0
        availability = 1 - error_rate
        
        return DeploymentMetrics(
            error_rate=error_rate,
            p99_latency=p99_latency,
            availability=availability
        )
    
    def _query_prometheus(self, query: str) -> Optional[float]:
        try:
            response = requests.get(
                f"{self.prometheus_url}/api/v1/query",
                params={"query": query},
                timeout=10
            )
            result = response.json()
            if result['data']['result']:
                return float(result['data']['result'][0]['value'][1])
        except Exception:
            pass
        return None
    
    def should_rollback(self, metrics: DeploymentMetrics) -> tuple[bool, str]:
        """ตรวจสอบว่าควร rollback หรือไม่"""
        
        if metrics.error_rate > self.thresholds['max_error_rate']:
            return True, f"Error rate {metrics.error_rate:.2%} exceeds threshold {self.thresholds['max_error_rate']:.2%}"
        
        if metrics.p99_latency > self.thresholds['max_p99_latency']:
            return True, f"P99 latency {metrics.p99_latency:.3f}s exceeds threshold {self.thresholds['max_p99_latency']:.3f}s"
        
        if metrics.availability < self.thresholds['min_availability']:
            return True, f"Availability {metrics.availability:.2%} below threshold {self.thresholds['min_availability']:.2%}"
        
        return False, "All metrics within thresholds"
    
    def execute_rollback(self) -> bool:
        """Execute rollback command"""
        try:
            result = subprocess.run([
                "kubectl", "rollout", "undo",
                f"deployment/{self.deployment}",
                "-n", self.namespace
            ], capture_output=True, text=True, timeout=300)
            
            if result.returncode == 0:
                # รอ rollback เสร็จ
                subprocess.run([
                    "kubectl", "rollout", "status",
                    f"deployment/{self.deployment}",
                    "-n", self.namespace,
                    "--timeout=5m"
                ], check=True)
                return True
            else:
                print(f"Rollback failed: {result.stderr}")
                return False
        except Exception as e:
            print(f"Error during rollback: {e}")
            return False
    
    def monitor_and_rollback(self, 
                              check_interval: int = 30,
                              monitoring_window: int = 300):
        """
        Monitor deployment และ rollback อัตโนมัติถ้า metrics เกิน threshold
        """
        print(f"Monitoring deployment: {self.deployment}")
        print(f"Check interval: {check_interval}s")
        print(f"Monitoring window: {monitoring_window}s")
        
        start_time = time.time()
        
        while time.time() - start_time < monitoring_window:
            metrics = self.get_metrics()
            
            print(f"Metrics: error_rate={metrics.error_rate:.2%}, "
                  f"p99={metrics.p99_latency:.3f}s, "
                  f"availability={metrics.availability:.2%}")
            
            should_roll, reason = self.should_rollback(metrics)
            
            if should_roll:
                print(f"ROLLBACK TRIGGERED: {reason}")
                success = self.execute_rollback()
                
                if success:
                    print("Rollback completed successfully")
                    return True
                else:
                    print("Rollback FAILED!")
                    return False
            
            time.sleep(check_interval)
        
        print("Monitoring complete - no rollback needed")
        return False


# ใช้งาน
rollback = AutoRollback(
    deployment_name="my-app",
    namespace="production",
    prometheus_url="http://prometheus:9090",
    thresholds={
        "max_error_rate": 0.05,       # 5% error rate
        "max_p99_latency": 2.0,       # 2 วินาที
        "min_availability": 0.99      # 99% availability
    }
)

rollback.monitor_and_rollback(
    check_interval=30,
    monitoring_window=600  # monitor 10 นาที
)
```

---

## 8. Smoke Tests

### Smoke Tests คืออะไร?

Smoke Tests คือการทดสอบ basic functionality หลังจาก deploy เพื่อยืนยันว่าระบบทำงานได้ก่อนที่จะ open traffic เต็ม

```bash
#!/bin/bash
# smoke_test.sh

set -euo pipefail

BASE_URL="${1:-http://localhost:8080}"
DEPLOYMENT="${2:-my-app}"
MAX_RETRIES=5
RETRY_INTERVAL=10

echo "Running smoke tests for ${DEPLOYMENT} at ${BASE_URL}"

# Function สำหรับ retry
retry() {
  local n=0
  until [ $n -ge $MAX_RETRIES ]; do
    "$@" && return 0
    n=$((n+1))
    echo "Attempt $n failed. Retrying in ${RETRY_INTERVAL}s..."
    sleep $RETRY_INTERVAL
  done
  echo "All $MAX_RETRIES attempts failed!"
  return 1
}

# Test 1: Health endpoint
echo "Test 1: Health check..."
retry curl -f -s "${BASE_URL}/health/ready" > /dev/null
echo "✓ Health check passed"

# Test 2: API version endpoint
echo "Test 2: API version..."
VERSION=$(curl -s "${BASE_URL}/api/version" | python3 -c "import sys,json; print(json.load(sys.stdin)['version'])")
echo "✓ API version: ${VERSION}"

# Test 3: Authentication
echo "Test 3: Authentication..."
TOKEN=$(curl -s -X POST "${BASE_URL}/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"smoke_test","password":"smoke_test_pass"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

if [ -z "$TOKEN" ]; then
  echo "✗ Authentication failed"
  exit 1
fi
echo "✓ Authentication passed"

# Test 4: Main business logic
echo "Test 4: Main API endpoint..."
STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer ${TOKEN}" \
  "${BASE_URL}/api/products?limit=1")

if [ "$STATUS" != "200" ]; then
  echo "✗ Main API failed with status ${STATUS}"
  exit 1
fi
echo "✓ Main API endpoint passed"

# Test 5: Database connectivity
echo "Test 5: Database check..."
DB_STATUS=$(curl -s "${BASE_URL}/health/full" | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print(d['checks']['database']['status'])")

if [ "$DB_STATUS" != "healthy" ]; then
  echo "✗ Database check failed: ${DB_STATUS}"
  exit 1
fi
echo "✓ Database check passed"

# Test 6: Cache connectivity
echo "Test 6: Cache check..."
CACHE_STATUS=$(curl -s "${BASE_URL}/health/full" | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print(d['checks']['redis']['status'])")

if [ "$CACHE_STATUS" != "healthy" ]; then
  echo "✗ Cache check failed: ${CACHE_STATUS}"
  exit 1
fi
echo "✓ Cache check passed"

# Test 7: Performance test (basic)
echo "Test 7: Performance check..."
RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" "${BASE_URL}/api/products?limit=10")
MAX_RESPONSE_TIME="2.0"

if (( $(echo "$RESPONSE_TIME > $MAX_RESPONSE_TIME" | bc -l) )); then
  echo "✗ Performance check failed: ${RESPONSE_TIME}s (max: ${MAX_RESPONSE_TIME}s)"
  exit 1
fi
echo "✓ Performance check passed: ${RESPONSE_TIME}s"

echo ""
echo "=== All smoke tests PASSED ==="
echo "Deployment ${DEPLOYMENT} is ready for production traffic"
```

### Smoke Test ด้วย Python (pytest)

```python
# test_smoke.py
import pytest
import requests
import time
import os

BASE_URL = os.getenv("SMOKE_TEST_URL", "http://localhost:8080")
API_VERSION = os.getenv("EXPECTED_API_VERSION", "2.0")

@pytest.fixture(scope="session")
def auth_token():
    """Login และรับ token"""
    response = requests.post(
        f"{BASE_URL}/api/auth/login",
        json={"username": "smoke_test", "password": "smoke_test_pass"},
        timeout=10
    )
    assert response.status_code == 200, f"Login failed: {response.text}"
    return response.json()["token"]

@pytest.fixture(scope="session")
def headers(auth_token):
    return {"Authorization": f"Bearer {auth_token}"}

class TestHealthChecks:
    def test_liveness(self):
        response = requests.get(f"{BASE_URL}/health/live", timeout=10)
        assert response.status_code == 200
        assert response.json()["status"] == "alive"
    
    def test_readiness(self):
        response = requests.get(f"{BASE_URL}/health/ready", timeout=10)
        assert response.status_code == 200
        assert response.json()["status"] == "ready"
    
    def test_database_health(self):
        response = requests.get(f"{BASE_URL}/health/full", timeout=10)
        assert response.status_code == 200
        data = response.json()
        assert data["checks"]["database"]["status"] == "healthy"

class TestAPIVersion:
    def test_api_version(self):
        response = requests.get(f"{BASE_URL}/api/version", timeout=10)
        assert response.status_code == 200
        data = response.json()
        assert data["version"] == API_VERSION
    
    def test_api_response_structure(self):
        response = requests.get(f"{BASE_URL}/api/version", timeout=10)
        data = response.json()
        assert "version" in data
        assert "build" in data
        assert "timestamp" in data

class TestCriticalAPIs:
    def test_list_products(self, headers):
        response = requests.get(
            f"{BASE_URL}/api/products",
            headers=headers,
            params={"limit": 1},
            timeout=10
        )
        assert response.status_code == 200
        data = response.json()
        assert "items" in data
        assert "total" in data
    
    def test_create_and_delete_item(self, headers):
        # Create
        create_response = requests.post(
            f"{BASE_URL}/api/items",
            headers=headers,
            json={"name": "smoke_test_item", "test": True},
            timeout=10
        )
        assert create_response.status_code == 201
        item_id = create_response.json()["id"]
        
        # Read
        read_response = requests.get(
            f"{BASE_URL}/api/items/{item_id}",
            headers=headers,
            timeout=10
        )
        assert read_response.status_code == 200
        
        # Delete
        delete_response = requests.delete(
            f"{BASE_URL}/api/items/{item_id}",
            headers=headers,
            timeout=10
        )
        assert delete_response.status_code == 204

class TestPerformance:
    def test_response_time(self):
        start = time.time()
        response = requests.get(f"{BASE_URL}/api/products?limit=10", timeout=10)
        duration = time.time() - start
        
        assert response.status_code == 200
        assert duration < 2.0, f"Response time {duration:.3f}s exceeds 2s threshold"
    
    def test_concurrent_requests(self):
        import concurrent.futures
        
        def make_request():
            response = requests.get(
                f"{BASE_URL}/health/ready",
                timeout=10
            )
            return response.status_code
        
        with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
            futures = [executor.submit(make_request) for _ in range(10)]
            results = [f.result() for f in concurrent.futures.as_completed(futures)]
        
        assert all(r == 200 for r in results), "Some concurrent requests failed"


if __name__ == "__main__":
    pytest.main([__file__, "-v", "--tb=short"])
```

---

## 9. Deployment Pipeline Examples

### GitHub Actions - Complete Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      strategy:
        description: 'Deployment strategy'
        required: true
        default: 'canary'
        type: choice
        options:
        - rolling
        - blue-green
        - canary

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  DEPLOYMENT_NAME: my-app
  NAMESPACE: production

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
    - uses: actions/checkout@v4
    
    - name: Run unit tests
      run: |
        npm install
        npm test
    
    - name: Run integration tests
      run: npm run test:integration
    
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
          type=sha,prefix=
          type=ref,event=branch
          type=semver,pattern={{version}}
    
    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > kubeconfig
        echo "KUBECONFIG=$PWD/kubeconfig" >> $GITHUB_ENV
    
    - name: Deploy to staging
      run: |
        IMAGE_TAG=$(echo "${{ needs.build-and-test.outputs.image-tag }}" | head -1)
        kubectl set image deployment/${DEPLOYMENT_NAME} \
          ${DEPLOYMENT_NAME}=${IMAGE_TAG} \
          -n staging
        kubectl rollout status deployment/${DEPLOYMENT_NAME} -n staging --timeout=10m
    
    - name: Run smoke tests on staging
      run: |
        STAGING_URL=$(kubectl get service ${DEPLOYMENT_NAME} \
          -n staging -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
        SMOKE_TEST_URL="http://${STAGING_URL}" \
          pytest tests/smoke/ -v --timeout=60

  deploy-production:
    needs: [build-and-test, deploy-staging]
    runs-on: ubuntu-latest
    environment: production
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBECONFIG_PRODUCTION }}" | base64 -d > kubeconfig
        echo "KUBECONFIG=$PWD/kubeconfig" >> $GITHUB_ENV
    
    - name: Deploy with selected strategy
      run: |
        IMAGE_TAG=$(echo "${{ needs.build-and-test.outputs.image-tag }}" | head -1)
        STRATEGY="${{ github.event.inputs.strategy || 'canary' }}"
        
        echo "Deploying with strategy: ${STRATEGY}"
        
        case "$STRATEGY" in
          rolling)
            kubectl set image deployment/${DEPLOYMENT_NAME} \
              ${DEPLOYMENT_NAME}=${IMAGE_TAG} \
              -n ${NAMESPACE}
            kubectl rollout status deployment/${DEPLOYMENT_NAME} \
              -n ${NAMESPACE} --timeout=15m
            ;;
          blue-green)
            ./scripts/blue-green-deploy.sh ${IMAGE_TAG} ${NAMESPACE}
            ;;
          canary)
            kubectl argo rollouts set image rollout/${DEPLOYMENT_NAME} \
              ${DEPLOYMENT_NAME}=${IMAGE_TAG} \
              -n ${NAMESPACE}
            kubectl argo rollouts status rollout/${DEPLOYMENT_NAME} \
              -n ${NAMESPACE} --timeout=30m
            ;;
        esac
    
    - name: Run production smoke tests
      run: |
        PROD_URL=$(kubectl get service ${DEPLOYMENT_NAME} \
          -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
        SMOKE_TEST_URL="http://${PROD_URL}" \
        EXPECTED_API_VERSION="${{ needs.build-and-test.outputs.image-tag }}" \
          pytest tests/smoke/ -v --timeout=60
    
    - name: Monitor deployment (auto-rollback)
      run: |
        python3 scripts/auto_rollback.py \
          --deployment ${DEPLOYMENT_NAME} \
          --namespace ${NAMESPACE} \
          --prometheus http://prometheus:9090 \
          --monitor-window 600 \
          --error-threshold 0.05 \
          --latency-threshold 2.0
    
    - name: Notify on success
      if: success()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "✅ Deployment successful!\nService: ${{ env.DEPLOYMENT_NAME }}\nStrategy: ${{ github.event.inputs.strategy }}\nCommit: ${{ github.sha }}"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    
    - name: Notify on failure
      if: failure()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "❌ Deployment FAILED!\nService: ${{ env.DEPLOYMENT_NAME }}\nPlease check: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    
    - name: Rollback on failure
      if: failure()
      run: |
        echo "Deployment failed, initiating rollback..."
        kubectl rollout undo deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
        kubectl rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
```

---

## 10. การเปรียบเทียบ Deployment Strategies

| กลยุทธ์ | Zero Downtime | Rollback Speed | Cost | Complexity | ใช้เมื่อ |
|---------|--------------|----------------|------|------------|---------|
| Rolling | ✅ | ช้า | ต่ำ | ต่ำ | Application ทั่วไป |
| Blue/Green | ✅ | เร็วมาก | สูง | ปานกลาง | Mission-critical systems |
| Canary | ✅ | เร็ว | ปานกลาง | สูง | ต้องการ gradual rollout |
| A/B Testing | ✅ | เร็ว | ปานกลาง | สูง | Business experiments |
| Shadow | ✅ | N/A | สูง | สูงมาก | ML models, critical migrations |

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Rolling Deployment

**เป้าหมาย**: Implement Rolling Deployment บน Kubernetes

```bash
# 1. สร้าง deployment
cat > exercise-1-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exercise-app
  namespace: default
spec:
  replicas: 5
  selector:
    matchLabels:
      app: exercise-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: exercise-app
    spec:
      containers:
      - name: exercise-app
        image: nginx:1.24
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
EOF

kubectl apply -f exercise-1-deployment.yaml

# 2. อัพเดต image และ watch rolling update
kubectl set image deployment/exercise-app exercise-app=nginx:1.25 &
kubectl rollout status deployment/exercise-app --watch

# 3. ตรวจสอบ history
kubectl rollout history deployment/exercise-app

# 4. Rollback
kubectl rollout undo deployment/exercise-app

# 5. ตรวจสอบ version หลัง rollback
kubectl get deployment exercise-app -o jsonpath='{.spec.template.spec.containers[0].image}'
```

### แบบฝึกหัดที่ 2: Blue/Green Deployment

**เป้าหมาย**: Implement Blue/Green Deployment และ switch traffic

```bash
# สร้าง script สำหรับ Blue/Green deployment
cat > exercise-2-bluegreen.sh << 'SCRIPT'
#!/bin/bash

# Deploy Blue (v1)
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: blue
  template:
    metadata:
      labels:
        app: myapp
        slot: blue
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:latest
        args: ["-text=I am version 1 (Blue)"]
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
    slot: blue
  ports:
  - port: 80
    targetPort: 5678
EOF

echo "Blue deployed. Testing..."
sleep 5
kubectl port-forward service/myapp-service 8080:80 &
PF_PID=$!
sleep 2
curl http://localhost:8080
kill $PF_PID

# Deploy Green (v2) ไม่รับ traffic ก่อน
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      slot: green
  template:
    metadata:
      labels:
        app: myapp
        slot: green
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:latest
        args: ["-text=I am version 2 (Green)"]
        ports:
        - containerPort: 5678
EOF

echo "Green deployed. Waiting for pods to be ready..."
kubectl rollout status deployment/app-green

# Switch traffic to Green
kubectl patch service myapp-service -p '{"spec":{"selector":{"slot":"green"}}}'

echo "Traffic switched to Green. Testing..."
kubectl port-forward service/myapp-service 8080:80 &
PF_PID=$!
sleep 2
curl http://localhost:8080
kill $PF_PID

echo "Blue/Green deployment complete!"
SCRIPT

chmod +x exercise-2-bluegreen.sh
```

### แบบฝึกหัดที่ 3: Feature Flags

**เป้าหมาย**: Implement Feature Flag system และทดสอบ

```python
# exercise-3-feature-flags.py
"""
แบบฝึกหัด: สร้าง Feature Flag system ที่รองรับ:
1. Boolean flags
2. Percentage rollout
3. User segment targeting
4. Environment-based flags
"""

import json
import hashlib
from typing import Optional

class FeatureFlagExercise:
    def __init__(self):
        self.flags = {}
    
    def add_flag(self, name: str, config: dict):
        """TODO: implement this method"""
        pass
    
    def is_enabled(self, flag_name: str, user_id: Optional[str] = None, 
                   environment: str = "production") -> bool:
        """TODO: implement this method"""
        pass
    
    def get_flag_info(self, flag_name: str) -> dict:
        """TODO: implement this method"""
        pass


# Test cases
def test_feature_flags():
    ff = FeatureFlagExercise()
    
    # Test 1: Boolean flag
    ff.add_flag("simple_flag", {"type": "boolean", "enabled": True})
    assert ff.is_enabled("simple_flag") == True
    
    # Test 2: Disabled flag
    ff.add_flag("disabled_flag", {"type": "boolean", "enabled": False})
    assert ff.is_enabled("disabled_flag") == False
    
    # Test 3: Percentage rollout (50%)
    ff.add_flag("partial_rollout", {"type": "percentage", "percentage": 50})
    # user-1 ควรได้รับ consistent result
    result1 = ff.is_enabled("partial_rollout", user_id="user-1")
    result2 = ff.is_enabled("partial_rollout", user_id="user-1")
    assert result1 == result2, "Same user should get consistent result"
    
    # Test 4: Environment-based flag
    ff.add_flag("prod_only", {
        "type": "environment",
        "environments": ["production"]
    })
    assert ff.is_enabled("prod_only", environment="production") == True
    assert ff.is_enabled("prod_only", environment="staging") == False
    
    print("All tests passed! Great work!")


if __name__ == "__main__":
    test_feature_flags()
```

### แบบฝึกหัดที่ 4: Deployment Pipeline

**เป้าหมาย**: สร้าง GitHub Actions workflow สำหรับ Blue/Green deployment

```yaml
# TODO: สร้างไฟล์ .github/workflows/bluegreen-deploy.yml
# ที่มี steps ต่อไปนี้:
# 1. Build Docker image
# 2. Push ไปยัง registry
# 3. Deploy Green environment
# 4. Run smoke tests บน Green
# 5. Switch traffic ถ้า tests ผ่าน
# 6. Notify team
# 7. Rollback ถ้า tests ล้มเหลว

# Hint: ใช้ structure ต่อไปนี้
name: Blue/Green Deployment

on:
  push:
    branches: [main]

jobs:
  build:
    # TODO: เพิ่ม steps สำหรับ build

  deploy-green:
    needs: build
    # TODO: เพิ่ม steps สำหรับ deploy green

  smoke-test:
    needs: deploy-green
    # TODO: เพิ่ม steps สำหรับ smoke test

  switch-traffic:
    needs: smoke-test
    # TODO: เพิ่ม steps สำหรับ switch traffic

  notify:
    needs: [switch-traffic]
    if: always()
    # TODO: เพิ่ม steps สำหรับ notification
```

### แบบฝึกหัดที่ 5: Auto Rollback System

**เป้าหมาย**: Implement automated rollback based on metrics

```python
# exercise-5-auto-rollback.py
"""
สร้างระบบ Auto Rollback ที่:
1. ตรวจสอบ error rate จาก mock metrics
2. ตรวจสอบ response time
3. Trigger rollback เมื่อ threshold เกิน
4. บันทึก rollback event
5. ส่ง notification (mock)
"""

import random
import time
from typing import Callable

def generate_mock_metrics(error_rate_increase: bool = False) -> dict:
    """
    Generate mock metrics สำหรับทดสอบ
    """
    base_error_rate = 0.02  # 2% error rate ปกติ
    
    if error_rate_increase:
        # Simulate degradation
        error_rate = base_error_rate + random.uniform(0.05, 0.10)
    else:
        error_rate = base_error_rate + random.uniform(-0.01, 0.01)
    
    return {
        "error_rate": max(0, error_rate),
        "p99_latency": random.uniform(0.5, 1.5) if not error_rate_increase else random.uniform(2.0, 4.0),
        "requests_per_second": random.uniform(100, 1000),
        "cpu_usage": random.uniform(20, 80)
    }


class AutoRollbackExercise:
    def __init__(self, thresholds: dict, rollback_fn: Callable):
        self.thresholds = thresholds
        self.rollback_fn = rollback_fn
        self.rollback_events = []
    
    def check_and_rollback(self, metrics: dict) -> bool:
        """
        TODO: ตรวจสอบ metrics และ rollback ถ้าจำเป็น
        Return True ถ้า rollback ถูก trigger
        """
        pass
    
    def monitor(self, 
                metrics_fn: Callable,
                duration: int = 60,
                interval: int = 5) -> list:
        """
        TODO: Monitor และ return list ของ rollback events
        """
        pass


# ทดสอบ
def test_auto_rollback():
    rollback_called = []
    
    def mock_rollback():
        rollback_called.append(time.time())
        print("ROLLBACK EXECUTED!")
    
    rb = AutoRollbackExercise(
        thresholds={
            "max_error_rate": 0.05,
            "max_p99_latency": 2.0
        },
        rollback_fn=mock_rollback
    )
    
    # Test 1: ไม่ควร rollback เมื่อ metrics ปกติ
    normal_metrics = generate_mock_metrics(error_rate_increase=False)
    result = rb.check_and_rollback(normal_metrics)
    assert result == False, "Should not rollback with normal metrics"
    
    # Test 2: ควร rollback เมื่อ error rate สูง
    bad_metrics = generate_mock_metrics(error_rate_increase=True)
    result = rb.check_and_rollback(bad_metrics)
    # TODO: เพิ่ม assertion
    
    print("Exercise 5 complete!")


if __name__ == "__main__":
    test_auto_rollback()
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้ deployment strategies หลัก 5 รูปแบบ:

1. **Rolling Deployment**: เหมาะสำหรับการ deploy ทั่วไป ใช้ทรัพยากรน้อย
2. **Blue/Green Deployment**: เหมาะสำหรับระบบที่ต้องการ instant rollback
3. **Canary Release**: เหมาะสำหรับการ deploy ที่ต้องการลดความเสี่ยง
4. **A/B Testing**: เหมาะสำหรับการทดสอบ features ทางธุรกิจ
5. **Shadow Deployment**: เหมาะสำหรับทดสอบ version ใหม่โดยไม่กระทบ user

กลยุทธ์ที่เลือกขึ้นอยู่กับ:
- ความสำคัญของ service
- ความซับซ้อนของ infrastructure
- ค่าใช้จ่ายที่ยอมรับได้
- ความต้องการในการ rollback

**ใน Part 22** เราจะเรียนรู้เรื่อง Infrastructure as Code ด้วย Terraform

---

## แหล่งอ้างอิง

- [Kubernetes Deployment Documentation](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
- [Istio Traffic Management](https://istio.io/latest/docs/concepts/traffic-management/)
- [Martin Fowler: BlueGreenDeployment](https://martinfowler.com/bliki/BlueGreenDeployment.html)
- [Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)

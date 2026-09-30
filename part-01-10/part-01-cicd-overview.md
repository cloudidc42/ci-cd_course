# Part 01: CI/CD คืออะไร? ภาพรวมและแนวคิดพื้นฐาน

> **ระดับ:** เริ่มต้น (Beginner)
> **เวลาที่ใช้:** 3-4 ชั่วโมง
> **ข้อกำหนดเบื้องต้น:** ความเข้าใจพื้นฐานด้านการพัฒนาซอฟต์แวร์

---

## สารบัญ

1. [ปัญหาก่อนมี CI/CD](#1-ปัญหาก่อนมี-cicd)
2. [ประวัติศาสตร์และวิวัฒนาการ](#2-ประวัติศาสตร์และวิวัฒนาการ)
3. [CI/CD คืออะไร?](#3-cicd-คืออะไร)
4. [ส่วนประกอบหลักของ CI/CD](#4-ส่วนประกอบหลักของ-cicd)
5. [DevOps vs CI/CD](#5-devops-vs-cicd)
6. [ประโยชน์ของ CI/CD](#6-ประโยชน์ของ-cicd)
7. [เครื่องมือ CI/CD ที่นิยมใช้](#7-เครื่องมือ-cicd-ที่นิยมใช้)
8. [DORA Metrics](#8-dora-metrics)
9. [Manual vs Automated Deployment](#9-manual-vs-automated-deployment)
10. [Real-World Workflow ตัวอย่าง](#10-real-world-workflow-ตัวอย่าง)
11. [แบบฝึกหัด](#11-แบบฝึกหัด)

---

## 1. ปัญหาก่อนมี CI/CD

### 1.1 "Works on My Machine" Syndrome

ก่อนที่จะมี CI/CD ทีมพัฒนาซอฟต์แวร์มักเจอปัญหาคลาสสิกที่เรียกว่า **"It works on my machine!"**

```
นักพัฒนา A: "โค้ดผมรันได้ปกติในเครื่องผมนะ"
นักพัฒนา B: "แต่พอ deploy ขึ้น production มัน error เลย"
นักพัฒนา A: "แปลกจัง... ของผมมัน work นะ 🤷"
```

**สาเหตุของปัญหา:**
- เครื่องของแต่ละคนมี environment ต่างกัน (OS, library versions, config)
- ไม่มีมาตรฐานกลาง
- ไม่มีการทดสอบอัตโนมัติ

### 1.2 Integration Hell

**สถานการณ์จริง:** ทีม 10 คน ต่างคนต่าง code ใน branch ของตัวเองนาน 2-3 สัปดาห์

```
สัปดาห์ที่ 1-3:
├── นักพัฒนา A แก้ไข user authentication module
├── นักพัฒนา B แก้ไข payment module  
├── นักพัฒนา C แก้ไข database schema
├── นักพัฒนา D แก้ไข API endpoints
└── ... (ทุกคน merge ไป main ในวันเดียวกัน)

สัปดาห์ที่ 4 (Merge Day = "Hell Day"):
├── ❌ Conflict ทุกที่
├── ❌ Tests fail 60%
├── ❌ Build ไม่ผ่าน
└── ❌ ทีมทำงาน overtime 3 วัน เพื่อแก้ปัญหา
```

### 1.3 Manual Deployment ที่เสี่ยงต่อ Human Error

ตัวอย่าง deployment script แบบ manual ที่ใช้กันเมื่อก่อน:

```bash
# ❌ วิธีเก่า - Manual Deployment (อันตรายมาก)
# ขั้นตอนนี้ต้องทำมือทั้งหมด และง่ายต่อการพลาด

# 1. SSH เข้า production server
ssh deploy@production-server.company.com

# 2. backup ก่อน (บางทีลืม)
cp -r /var/www/html /var/www/html_backup_$(date +%Y%m%d)

# 3. pull code ล่าสุด
cd /var/www/html
git pull origin main

# 4. install dependencies (บางทีลืมรัน)
npm install

# 5. build (บางทีลืมรัน)  
npm run build

# 6. restart service (บางทีทำผิด service)
sudo systemctl restart nginx
sudo systemctl restart node-app

# 7. ตรวจสอบว่า work ไหม (บางทีข้ามขั้นตอนนี้)
curl http://localhost:3000/health
```

**ปัญหาที่เกิดจาก Manual Deployment:**
- ลืมขั้นตอน
- รัน command ผิด
- Deploy ผิด environment (deploy dev code ขึ้น production!)
- ไม่มี rollback plan ที่ชัดเจน
- ไม่มีการทดสอบก่อน deploy
- ทำได้แค่คนเดียวในแต่ละครั้ง

### 1.4 Slow Release Cycles

```
แนวทางเก่า (Waterfall + Manual):
├── Development:     2-3 เดือน
├── Testing:         1-2 เดือน  
├── UAT:             2-4 สัปดาห์
├── Deployment prep: 1-2 สัปดาห์
└── Go-Live:         1 วัน (วันศุกร์ตอนเย็น 😰)

รวม: 4-6 เดือนต่อ release หนึ่ง!

ปัญหา: ถ้า release นี้มี bug ใหญ่
└── ต้องรออีก 4-6 เดือนเพื่อ fix 😱
```

---

## 2. ประวัติศาสตร์และวิวัฒนาการ

### 2.1 Timeline ของ CI/CD

```
1990s - Waterfall Era
│
├── ซอฟต์แวร์พัฒนาแบบ Waterfall
├── Release cycle: 6-18 เดือน
└── Testing: Manual ทั้งหมด

2000 - Agile Manifesto
│
├── แนวคิด iterative development
├── Shorter release cycles
└── ความต้องการ automated testing เพิ่มขึ้น

2001 - XP (Extreme Programming) และ CI
│
├── Martin Fowler เขียนบทความ "Continuous Integration" 
├── Kent Beck แนะนำ CI ใน XP
└── แนวคิด: integrate often, test automatically

2008 - CruiseControl, Bamboo, Hudson
│
├── เครื่องมือ CI ตัวแรกๆ ออกสู่ตลาด
├── Jenkins (fork จาก Hudson) เกิดขึ้น 2011
└── ทีมใหญ่เริ่มใช้ automated build

2011-2014 - Cloud Era
│
├── Travis CI เปิดตัวสำหรับ GitHub projects
├── CircleCI, TeamCity ออกสู่ตลาด
└── CD (Continuous Delivery) เริ่มเป็นที่รู้จัก

2014 - "Continuous Delivery" Book
│
├── Jez Humble & David Farley เขียนหนังสือ
└── CD กลายเป็น mainstream concept

2015-2020 - Docker & Kubernetes Era  
│
├── Container technology เปลี่ยนเกม
├── GitLab CI/CD, GitHub Actions ออกสู่ตลาด
└── Infrastructure as Code เกิดขึ้น

2020-ปัจจุบัน - GitOps Era
│
├── GitHub Actions กลายเป็น dominant
├── ArgoCD, Flux สำหรับ Kubernetes deployments
└── Everything as Code
```

### 2.2 วิวัฒนาการของ Software Delivery

| ยุค | วิธีการ | Release Frequency | ความเสี่ยง |
|-----|---------|-------------------|-----------|
| 1990s | Waterfall | ทุก 6-18 เดือน | สูงมาก |
| 2000s | Agile | ทุก 1-3 เดือน | สูง |
| 2010s | CI/CD เบื้องต้น | ทุก 1-4 สัปดาห์ | ปานกลาง |
| 2015s | CI/CD + Docker | ทุก 1-7 วัน | ต่ำ |
| ปัจจุบัน | Full CI/CD + GitOps | หลายครั้งต่อวัน | ต่ำมาก |

---

## 3. CI/CD คืออะไร?

### 3.1 นิยามของ CI/CD

**CI/CD** ย่อมาจาก:
- **CI** = Continuous Integration (การรวมโค้ดอย่างต่อเนื่อง)
- **CD** = Continuous Delivery หรือ Continuous Deployment

### 3.2 Continuous Integration (CI)

**CI คือ:** การที่นักพัฒนา merge โค้ดของตัวเองเข้า main branch บ่อยๆ (อย่างน้อยวันละครั้ง) และทุกครั้งที่ merge จะมีการ build และ test อัตโนมัติ

```
ขั้นตอน CI:
1. นักพัฒนา commit และ push code
        ↓
2. CI server ตรวจพบ push ใหม่
        ↓
3. ดาวน์โหลด code ล่าสุด
        ↓
4. Build application
        ↓
5. รัน automated tests
        ↓
6. แจ้งผล: PASS ✅ หรือ FAIL ❌
        ↓
7. ถ้า FAIL → แจ้ง developer ทันที
```

**หลักการสำคัญของ CI:**
- Commit เล็กๆ บ่อยๆ ดีกว่า commit ใหญ่นานๆ ครั้ง
- Build ต้องเร็ว (ไม่เกิน 10-15 นาที)
- ทุกคนในทีมต้อง fix broken build ทันที
- อย่า push code ที่ทำให้ build fail

### 3.3 Continuous Delivery (CD)

**Continuous Delivery คือ:** การที่ซอฟต์แวร์อยู่ในสภาพพร้อม deploy ตลอดเวลา แต่การ deploy จริงยังต้องการ manual approval

```
Continuous Delivery Flow:

Code Push
    ↓
CI Pipeline (Build + Test)
    ↓
Deploy to Staging (อัตโนมัติ)
    ↓
Automated Testing บน Staging
    ↓
⏸️ รอ Manual Approval
    ↓
[คนกด "Approve Deploy"]
    ↓
Deploy to Production
```

**เหมาะกับ:** องค์กรที่ต้องการ human oversight ก่อน production deploy เช่น ธนาคาร, โรงพยาบาล, ระบบ critical

### 3.4 Continuous Deployment (CD)

**Continuous Deployment คือ:** ทุกครั้งที่ code ผ่าน tests ทั้งหมด จะถูก deploy ไป production โดยอัตโนมัติ ไม่ต้องรอ manual approval

```
Continuous Deployment Flow:

Code Push
    ↓
CI Pipeline (Build + Test)
    ↓
Deploy to Staging (อัตโนมัติ)
    ↓
Automated Testing บน Staging
    ↓
Deploy to Production (อัตโนมัติ! 🚀)
    ↓
Monitor & Alert
```

**เหมาะกับ:** บริษัทที่ต้องการ velocity สูง เช่น startup, Netflix, Amazon

### 3.5 ความแตกต่างระหว่าง Delivery vs Deployment

```
Continuous DELIVERY:
├── ✅ Build อัตโนมัติ
├── ✅ Test อัตโนมัติ
├── ✅ Deploy ถึง Staging อัตโนมัติ
├── ⏸️  Manual approval ก่อน Production
└── ✅ Deploy to Production (หลัง approve)

Continuous DEPLOYMENT:
├── ✅ Build อัตโนมัติ
├── ✅ Test อัตโนมัติ
├── ✅ Deploy ถึง Staging อัตโนมัติ
├── ✅ Deploy to Production อัตโนมัติ (ไม่ต้อง approve)
└── ✅ Monitor อัตโนมัติ
```

---

## 4. ส่วนประกอบหลักของ CI/CD

### 4.1 Source Control Management (SCM)

จุดเริ่มต้นของทุก pipeline คือ Version Control System

```
Source Code Repository
├── Git (GitHub, GitLab, Bitbucket, Azure Repos)
├── Branch strategy (Git Flow, GitHub Flow, Trunk-based)
└── Code review process (Pull Requests / Merge Requests)
```

### 4.2 Build System

แปลง source code เป็น artifact ที่พร้อม deploy

```bash
# ตัวอย่าง Build สำหรับภาษาต่างๆ

# Node.js
npm install
npm run build

# Java/Maven
mvn clean package

# Python
pip install -r requirements.txt
python setup.py build

# Go
go build -o myapp ./...

# Docker
docker build -t myapp:v1.0 .
```

### 4.3 Automated Testing Pyramid

```
                    /\
                   /  \
                  / E2E \          ← น้อย, ช้า, แต่ครอบคลุม
                 /  Tests \
                /          \
               /────────────\
              /  Integration  \    ← ปานกลาง
             /    Tests        \
            /                  \
           /────────────────────\
          /    Unit Tests        \  ← เยอะ, เร็ว, ราคาถูก
         /                        \
        /──────────────────────────\
```

**Unit Tests:**
```python
# ตัวอย่าง Unit Test (Python/pytest)
def calculate_discount(price, discount_percent):
    if discount_percent < 0 or discount_percent > 100:
        raise ValueError("Discount must be between 0 and 100")
    return price * (1 - discount_percent / 100)

def test_calculate_discount_normal():
    assert calculate_discount(100, 10) == 90.0

def test_calculate_discount_zero():
    assert calculate_discount(100, 0) == 100.0

def test_calculate_discount_invalid():
    with pytest.raises(ValueError):
        calculate_discount(100, -10)
```

**Integration Tests:**
```javascript
// ตัวอย่าง Integration Test (Jest + Supertest)
describe('User API', () => {
  it('should create a new user', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'John', email: 'john@example.com' })
      .expect(201);
    
    expect(response.body.id).toBeDefined();
    expect(response.body.name).toBe('John');
  });
});
```

**E2E Tests:**
```javascript
// ตัวอย่าง E2E Test (Playwright)
test('user can login and see dashboard', async ({ page }) => {
  await page.goto('https://myapp.com');
  await page.fill('#email', 'user@example.com');
  await page.fill('#password', 'password123');
  await page.click('#login-btn');
  
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('Welcome');
});
```

### 4.4 Artifact Repository

ที่เก็บ build artifacts (compiled code, docker images, packages)

```
Artifact Types และ Repositories:
├── Docker Images → Docker Hub, AWS ECR, GCP Artifact Registry
├── npm packages → npm registry, JFrog Artifactory  
├── Maven artifacts → Maven Central, Nexus
├── Python packages → PyPI, JFrog Artifactory
└── Binary files → S3, Azure Blob Storage
```

### 4.5 Deployment Automation

เครื่องมือที่ใช้ deploy application

```yaml
# ตัวอย่าง deployment strategies

# Rolling Deployment
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1

# Blue-Green Deployment
# production-blue (current)
# production-green (new version)
# Switch traffic: blue → green

# Canary Deployment  
# 5% traffic → new version
# ถ้าปกติดี → เพิ่มเป็น 25% → 50% → 100%
```

### 4.6 Monitoring & Observability

หลัง deploy ต้องมีการ monitor

```
Monitoring Stack:
├── Metrics → Prometheus + Grafana
├── Logs → ELK Stack (Elasticsearch, Logstash, Kibana)
├── Tracing → Jaeger, Zipkin
├── Alerting → PagerDuty, OpsGenie
└── Uptime → Pingdom, StatusPage
```

---

## 5. DevOps vs CI/CD

### 5.1 DevOps คืออะไร?

**DevOps** คือ **วัฒนธรรม** และ **แนวทางปฏิบัติ** ที่รวม Development (Dev) และ Operations (Ops) เข้าด้วยกัน

```
Development Team:          Operations Team:
├── เขียน code            ├── ดูแล infrastructure
├── สร้าง features        ├── ดูแล servers
├── Fix bugs              ├── Monitor systems
└── ต้องการ deploy เร็ว  └── ต้องการ stability

ก่อน DevOps: สองทีมนี้ทำงานแยกกัน ขัดแย้งกันบ่อย
หลัง DevOps: ทำงานร่วมกัน มีเป้าหมายเดียวกัน
```

### 5.2 ความสัมพันธ์ระหว่าง DevOps และ CI/CD

```
DevOps = วัฒนธรรม + แนวทางปฏิบัติ (Culture + Practices)
    ↓
CI/CD = เครื่องมือ + กระบวนการ (Tools + Processes)
    ↓
CI/CD คือหนึ่งใน practices ของ DevOps
```

**DevOps Practices ที่สำคัญ:**
1. **CI/CD** - Automate build, test, deploy
2. **Infrastructure as Code (IaC)** - Terraform, Ansible
3. **Monitoring & Logging** - Observability
4. **Microservices** - Architecture approach
5. **Containerization** - Docker, Kubernetes
6. **Collaboration** - บทบาทและความรับผิดชอบร่วมกัน

### 5.3 DevOps Infinity Loop

```
         Plan ←────────────────────────────────┐
           │                                    │
           ▼                                    │
         Code                               Monitor
           │                                    │
           ▼                                    │
         Build ──→ CI/CD Pipeline ──────→ Operate
           │                                    │
           ▼                                    │
         Test ────────────────────────→ Deploy  │
           │                                    │
           └────────────────────────────────────┘
```

---

## 6. ประโยชน์ของ CI/CD

### 6.1 ประโยชน์ด้านความเร็ว

```
ก่อน CI/CD:
- Time to Market: 3-6 เดือน
- Deploy frequency: 2-4 ครั้งต่อปี
- Lead time for changes: หลายสัปดาห์

หลัง CI/CD:
- Time to Market: 1-2 สัปดาห์
- Deploy frequency: หลายครั้งต่อวัน
- Lead time for changes: ชั่วโมง - วัน
```

**ตัวอย่างจริงจาก Amazon:**
- ปี 2011: deploy ทุก 11.6 วัน
- ปี 2013: deploy ทุก 11.6 วินาที (เพิ่มขึ้น 75x!)
- ปัจจุบัน: มากกว่า 50 million deployments ต่อปี

### 6.2 ประโยชน์ด้านคุณภาพ

```
ปัญหาที่ลดลงหลัง CI/CD:
├── ❌ Integration conflicts ลดลง 90%
├── ❌ Production bugs ลดลง 60-70%
├── ❌ Rollback time ลดจากชั่วโมง → นาที
└── ❌ "Works on my machine" หมดไป
```

### 6.3 ประโยชน์ด้าน Team

```
ก่อน CI/CD:
├── Developer หมดเวลาไปกับ manual tasks
├── Ops team เครียดทุกครั้งที่ deploy
└── Blame culture เมื่อมี production issues

หลัง CI/CD:
├── Developer โฟกัสที่ business logic
├── Deploy เป็นเรื่องปกติ ไม่น่ากลัว
└── Culture of collaboration
```

### 6.4 ประโยชน์ด้านธุรกิจ

| ตัวชี้วัด | ก่อน CI/CD | หลัง CI/CD |
|-----------|-----------|-----------|
| Time to market | 6 เดือน | 2 สัปดาห์ |
| Cost per deployment | สูงมาก | ต่ำมาก |
| Mean Time to Recovery | หลายชั่วโมง | < 1 ชั่วโมง |
| Change failure rate | 15-46% | < 5% |
| Developer satisfaction | ต่ำ | สูง |

---

## 7. เครื่องมือ CI/CD ที่นิยมใช้

### 7.1 ภาพรวมเครื่องมือ

```
CI/CD Tools Landscape:

Cloud-based (SaaS):
├── GitHub Actions     → รวมอยู่ใน GitHub
├── GitLab CI/CD       → รวมอยู่ใน GitLab
├── CircleCI           → cloud-native CI/CD
├── Travis CI          → popular สำหรับ open source
└── Bitbucket Pipelines→ รวมอยู่ใน Bitbucket

Self-hosted:
├── Jenkins            → open source, ยืดหยุ่นมาก
├── TeamCity           → JetBrains, enterprise
├── Bamboo             → Atlassian ecosystem
└── Drone CI           → container-native

Container/Kubernetes Focused:
├── ArgoCD             → GitOps สำหรับ K8s
├── Flux               → GitOps operator
└── Tekton             → Kubernetes-native pipelines
```

### 7.2 GitHub Actions (จะใช้ในคอร์สนี้)

**ทำไมถึงเลือก GitHub Actions:**
- ฟรีสำหรับ public repos
- รวมอยู่กับ GitHub แล้ว
- Community ใหญ่มาก (marketplace มี 10,000+ actions)
- ใช้งานง่าย ไม่ต้องตั้ง server เอง

```yaml
# ตัวอย่าง GitHub Actions workflow พื้นฐาน
name: Basic CI Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm install
      
      - name: Run tests
        run: npm test
      
      - name: Build application
        run: npm run build
```

### 7.3 Jenkins (เครื่องมือ classic)

```groovy
// Jenkinsfile ตัวอย่าง
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh staging'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Yes, deploy!"
            }
            steps {
                sh './deploy.sh production'
            }
        }
    }
    
    post {
        failure {
            mail to: 'team@company.com',
                 subject: "Build Failed: ${env.JOB_NAME}",
                 body: "Check console output at ${env.BUILD_URL}"
        }
    }
}
```

### 7.4 GitLab CI/CD

```yaml
# .gitlab-ci.yml ตัวอย่าง
stages:
  - build
  - test
  - staging
  - production

build:
  stage: build
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/

test:
  stage: test
  script:
    - npm test
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'

deploy_staging:
  stage: staging
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.myapp.com
  only:
    - main

deploy_production:
  stage: production
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://myapp.com
  when: manual
  only:
    - main
```

### 7.5 เปรียบเทียบเครื่องมือ

| เครื่องมือ | ค่าใช้จ่าย | Ease of Use | Flexibility | Best For |
|-----------|-----------|-------------|-------------|----------|
| GitHub Actions | ฟรี/Paid | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | GitHub repos |
| GitLab CI | ฟรี/Paid | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | GitLab repos |
| Jenkins | ฟรี (infrastructure cost) | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Enterprise/On-premise |
| CircleCI | ฟรี/Paid | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Cloud-native teams |
| Travis CI | ฟรี (open source) | ⭐⭐⭐⭐ | ⭐⭐⭐ | Open source projects |

---

## 8. DORA Metrics

### 8.1 DORA คืออะไร?

**DORA** (DevOps Research and Assessment) เป็นทีมวิจัยของ Google ที่ศึกษาว่าอะไรทำให้ software delivery ดีขึ้น

พวกเขาพบว่ามี **4 Key Metrics** ที่แยก "high performers" จาก "low performers" ได้ชัดเจน

### 8.2 The 4 DORA Metrics

#### Metric 1: Deployment Frequency (DF)

**ความถี่ในการ deploy to production**

```
Elite:  หลายครั้งต่อวัน (Netflix, Amazon, Google)
High:   ครั้งหนึ่งต่อวัน ถึง ครั้งหนึ่งต่อสัปดาห์
Medium: ครั้งหนึ่งต่อสัปดาห์ ถึง ครั้งหนึ่งต่อเดือน
Low:    น้อยกว่าครั้งหนึ่งต่อเดือน (ระวัง! นี่คือ danger zone)
```

#### Metric 2: Lead Time for Changes (LT)

**เวลาตั้งแต่ commit จนถึง production**

```
Elite:  < 1 ชั่วโมง
High:   1 วัน ถึง 1 สัปดาห์
Medium: 1 สัปดาห์ ถึง 1 เดือน
Low:    > 6 เดือน (!) 
```

#### Metric 3: Mean Time to Recovery (MTTR)

**เวลาเฉลี่ยที่ใช้แก้ปัญหาเมื่อ production พัง**

```
Elite:  < 1 ชั่วโมง
High:   < 1 วัน
Medium: 1 วัน ถึง 1 สัปดาห์
Low:    > 1 เดือน (ร้ายแรงมาก)
```

#### Metric 4: Change Failure Rate (CFR)

**% ของ deployments ที่ทำให้เกิดปัญหาใน production**

```
Elite:  0-15%
High:   16-30%
Medium: 16-30%
Low:    > 45%
```

### 8.3 ตัวอย่างการวัด DORA Metrics

```bash
# Script คำนวณ Deployment Frequency (simplified)
#!/bin/bash

# นับ deployments ใน 30 วันที่ผ่านมา
DAYS=30
DEPLOY_COUNT=$(git log --oneline --after="30 days ago" | grep "deploy" | wc -l)
FREQ=$(echo "scale=2; $DEPLOY_COUNT / $DAYS" | bc)

echo "Deployments ใน $DAYS วัน: $DEPLOY_COUNT"
echo "Deployment Frequency: $FREQ per day"

if (( $(echo "$FREQ >= 1" | bc -l) )); then
    echo "Performance Level: ELITE ⭐"
elif (( $(echo "$FREQ >= 0.14" | bc -l) )); then
    echo "Performance Level: HIGH 👍"
else
    echo "Performance Level: NEEDS IMPROVEMENT 📈"
fi
```

### 8.4 DORA Performance Levels สรุป

```
                    Deployment    Lead Time    MTTR        CFR
                    Frequency     for Changes
Elite Performers:   Multiple/day  < 1 hour     < 1 hour    0-15%
High Performers:    1/day-1/week  1d-1week     < 1 day     16-30%
Medium Performers:  1/week-1/month 1week-1mo  1d-1week     16-30%
Low Performers:     < 1/month     > 6 months  > 1 month    > 45%
```

---

## 9. Manual vs Automated Deployment

### 9.1 เปรียบเทียบแบบละเอียด

#### Manual Deployment Process

```
Manual Deployment (แบบเก่า):

1. Developer บอก Ops team: "ผมพร้อม deploy แล้ว"
   ⏱️  รอ: 1-3 วัน (scheduling, approvals)

2. Ops ขอ deployment package
   ⏱️  รอ: หลายชั่วโมง
   
3. Ops อ่าน deployment guide (ถ้ามี)
   ⏱️  เวลา: 30 นาที - 2 ชั่วโมง
   
4. Backup database และ files
   ⏱️  เวลา: 30 นาที - 2 ชั่วโมง

5. Stop services
   ⏱️  Downtime เริ่มต้น!
   
6. Deploy code manually
   ⏱️  เวลา: 30 นาที - 2 ชั่วโมง
   ⚠️  Human error เกิดได้ทุกขั้นตอน
   
7. Run database migrations (บางทีลืม!)
   ⚠️  ถ้าลืม: production พัง
   
8. Start services
9. Manual smoke testing
   ⏱️  เวลา: 30 นาที - 1 ชั่วโมง
   
10. Monitor ด้วยมือ
    ⏱️  เวลา: หลายชั่วโมง

รวม Total Time: 4-12 ชั่วโมง
Downtime: 30 นาที - 2 ชั่วโมง
Risk level: สูงมาก ❌
```

#### Automated Deployment Process

```
Automated Deployment (CI/CD):

1. Developer push code → Git
   ⏱️  เวลา: ทันที
   
2. CI/CD pipeline trigger อัตโนมัติ
   ⏱️  เวลา: ทันที
   
3. Build + Test อัตโนมัติ
   ⏱️  เวลา: 5-15 นาที
   
4. Security scan อัตโนมัติ
   ⏱️  เวลา: 2-5 นาที
   
5. Deploy to staging อัตโนมัติ
   ⏱️  เวลา: 2-5 นาที
   
6. Automated tests บน staging
   ⏱️  เวลา: 5-20 นาที
   
7. (Optional) Manual approval
   ⏱️  เวลา: ขึ้นอยู่กับ team
   
8. Deploy to production (อาจอัตโนมัติ)
   ⏱️  เวลา: 2-5 นาที (zero downtime!)
   
9. Automated smoke tests
   ⏱️  เวลา: 1-2 นาที
   
10. Automated monitoring + alerts

รวม Total Time: 20-60 นาที
Downtime: 0 (zero downtime deployment)
Risk level: ต่ำมาก ✅
```

### 9.2 ตัวอย่าง Zero Downtime Deployment

```yaml
# Kubernetes Rolling Update (Zero Downtime)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # ลด 1 pod ก่อน
      maxSurge: 1          # เพิ่ม 1 pod ใหม่ขณะ update
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
        image: my-app:new-version
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
```

```
Rolling Update Process:
เริ่มต้น: [v1][v1][v1][v1]  ← 4 instances ทำงานอยู่

Step 1:  [  ][v1][v1][v1]  ← Stop 1 instance
         [v2][v1][v1][v1]  ← Start v2
         
Step 2:  [v2][  ][v1][v1]  ← Stop 1 instance
         [v2][v2][v1][v1]  ← Start v2
         
Step 3:  [v2][v2][  ][v1]  ← Stop 1 instance
         [v2][v2][v2][v1]  ← Start v2
         
Step 4:  [v2][v2][v2][  ]  ← Stop last instance
         [v2][v2][v2][v2]  ← Start v2

ตลอดกระบวนการ: traffic ไม่หยุด! 🎉
```

---

## 10. Real-World Workflow ตัวอย่าง

### 10.1 E-Commerce Company Workflow

สมมติบริษัท e-commerce ที่ใช้ CI/CD เต็มรูปแบบ

```
Feature: เพิ่มฟีเจอร์ "ชำระเงินด้วย QR Code"

ขั้นตอน 1: Planning
├── Product Manager สร้าง ticket ใน Jira/Linear
├── Engineer estimate งาน
└── Create feature branch: feature/qr-payment

ขั้นตอน 2: Development (นักพัฒนา A)
├── git checkout -b feature/qr-payment
├── เขียน code
├── เขียน unit tests
├── commit และ push
└── สร้าง Pull Request

ขั้นตอน 3: CI Pipeline (อัตโนมัติ ~10 นาที)
├── ✅ Lint check (code style)
├── ✅ Unit tests (500 tests, all pass)
├── ✅ Integration tests
├── ✅ Security scan (SAST)
├── ✅ Build Docker image
└── ✅ Deploy to dev environment

ขั้นตอน 4: Code Review
├── Reviewer ดู code ใน PR
├── Request changes หรือ Approve
└── Merge to main branch

ขั้นตอน 5: Staging Deploy (อัตโนมัติ)
├── CI pipeline รัน tests อีกครั้ง
├── Deploy to staging
├── Automated E2E tests
├── Performance tests
└── QA team ทดสอบ manual บน staging

ขั้นตอน 6: Production Deploy
├── (Continuous Delivery) Manual approval → Deploy
├── หรือ (Continuous Deployment) อัตโนมัติ
├── Zero downtime rolling update
├── Automated smoke tests
└── Monitor metrics 30 นาที

Total Time: 2-3 วัน (แทนที่จะเป็น 2-3 สัปดาห์)
```

### 10.2 GitHub Actions Pipeline ตัวอย่างเต็ม

```yaml
# .github/workflows/main.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ===== JOB 1: Code Quality =====
  lint-and-format:
    name: Lint & Format Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Check code formatting
        run: npm run format:check

  # ===== JOB 2: Tests =====
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: lint-and-format
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit
      
      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://postgres:testpassword@localhost:5432/testdb
        run: npm run test:integration
      
      - name: Upload test coverage
        uses: codecov/codecov-action@v3

  # ===== JOB 3: Security Scan =====
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: lint-and-format
    steps:
      - uses: actions/checkout@v4
      
      - name: Run dependency audit
        run: npm audit --audit-level=high
      
      - name: Run SAST scan
        uses: github/codeql-action/analyze@v2
        with:
          languages: javascript

  # ===== JOB 4: Build =====
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch
            type=semver,pattern={{version}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  # ===== JOB 5: Deploy to Staging =====
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        run: |
          echo "Deploying to staging..."
          # kubectl set image deployment/myapp myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
      
      - name: Run E2E tests on staging
        run: |
          echo "Running E2E tests..."
          # npx playwright test --base-url https://staging.myapp.com

  # ===== JOB 6: Deploy to Production =====
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production  # ต้องการ manual approval
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          echo "Deploying to production..."
          # kubectl set image deployment/myapp myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
      
      - name: Verify deployment
        run: |
          echo "Verifying deployment..."
          # curl https://myapp.com/health
      
      - name: Notify team
        if: success()
        run: |
          echo "Sending Slack notification..."
          # curl -X POST ${{ secrets.SLACK_WEBHOOK }} -d '{"text":"✅ Production deployment successful!"}'
```

### 10.3 Real-World ตัวอย่างจากบริษัทชั้นนำ

**Netflix:**
```
- Deploy มากกว่า 100 ครั้งต่อวัน
- ใช้ Spinnaker สำหรับ deployment
- Chaos Engineering (Netflix Monkey) ทดสอบ resilience
- Lead time: นาที ไม่ใช่ สัปดาห์
```

**Etsy:**
```
- Deploy มากกว่า 50 ครั้งต่อวัน
- ทุก engineer มีสิทธิ์ deploy
- Feature flags ควบคุม rollout
- "If it hurts, do it more often" - หลักการหลัก
```

**Amazon:**
```
- Deploy ทุก 11.6 วินาที (ปี 2013)
- ทุก team เป็น "you build it, you run it"
- Two-pizza team rule
- Rollback ภายใน < 5 นาที
```

---

## 11. แบบฝึกหัด

### แบบฝึกหัดที่ 1: วิเคราะห์ปัญหา (ความเข้าใจพื้นฐาน)

**โจทย์:** อ่านสถานการณ์ต่อไปนี้และตอบคำถาม

```
บริษัท ABC มีทีมพัฒนา 8 คน พวกเขาทำงานในแอปพลิเคชัน e-commerce
ทุก 2 เดือน พวกเขาจะทำ "Big Bang Release" โดย:
1. ทุกคน merge code ใน sprint สุดท้ายของ release
2. QA ทดสอบด้วยมือนาน 3 สัปดาห์
3. Ops team deploy ในวันศุกร์ตอนดึก
4. บ่อยครั้งที่ deployment มีปัญหาและต้อง rollback
5. ทีมทำงานตลอดคืนวันศุกร์เพื่อแก้ปัญหา
```

**คำถาม:**
1. ปัญหาหลักที่บริษัท ABC เจออยู่คืออะไร? (ระบุอย่างน้อย 5 ข้อ)
2. DORA metrics ของบริษัทนี้อยู่ในระดับไหน? (Elite/High/Medium/Low)
3. ถ้าคุณเป็น consultant ที่จะช่วยแก้ปัญหา คุณจะแนะนำอะไรเป็นอย่างแรก?

**เฉลย:**
1. ปัญหาหลัก:
   - Integration Hell (merge พร้อมกัน)
   - Long feedback loop (รู้ปัญหาช้า)
   - Risky deployments (deploy ดึกวันศุกร์)
   - Manual testing bottleneck
   - Poor work-life balance
   - No automated rollback plan

2. DORA Level: **Low** (deploy ทุก 2 เดือน)

3. แนะนำก่อน: เริ่ม CI - ตั้ง automated tests + build pipeline ก่อน จากนั้นค่อยเพิ่ม CD

---

### แบบฝึกหัดที่ 2: วาด CI/CD Flow

**โจทย์:** วาด (หรือเขียน) CI/CD pipeline สำหรับ Node.js web application ที่มี:
- Unit tests ด้วย Jest
- Integration tests กับ PostgreSQL database
- Build Docker image
- Deploy ไปยัง staging และ production

```
เฉลย - Pipeline Flow:

[Push to GitHub]
      ↓
[Trigger CI Pipeline]
      ↓
┌─────────────────────┐
│  Parallel Jobs:     │
│  ├── Lint/Format    │
│  ├── Unit Tests     │
│  └── Security Scan  │
└─────────────────────┘
      ↓ (ถ้าทุกอย่าง pass)
[Integration Tests]
  (กับ PostgreSQL)
      ↓
[Build Docker Image]
      ↓
[Push to Registry]
      ↓
[Deploy to Staging] ← อัตโนมัติ
      ↓
[E2E Tests on Staging]
      ↓
[Manual Approval] ← รอคนกด
      ↓
[Deploy to Production]
      ↓
[Smoke Tests]
      ↓
[Monitor & Alert]
```

---

### แบบฝึกหัดที่ 3: เลือกเครื่องมือที่เหมาะสม

**โจทย์:** แต่ละ scenario ควรใช้เครื่องมืออะไร?

**Scenario A:** Startup 5 คน ใช้ GitHub เก็บ code ต้องการ CI/CD ฟรี

**Scenario B:** บริษัทใหญ่ 500 คน มี data center ของตัวเอง ต้องการ control เต็มที่

**Scenario C:** Team ที่ใช้ GitLab เป็น source control อยู่แล้ว

**Scenario D:** Kubernetes-heavy environment ต้องการ GitOps

**เฉลย:**
- A: **GitHub Actions** (ฟรี, รวมกับ GitHub)
- B: **Jenkins** (self-hosted, full control, ฟรี)
- C: **GitLab CI/CD** (built-in, seamless integration)
- D: **ArgoCD หรือ Flux** (GitOps operators)

---

### แบบฝึกหัดที่ 4: คำนวณ DORA Metrics

**โจทย์:** บริษัท XYZ มีข้อมูลต่อไปนี้ใน 30 วันที่ผ่านมา:
- จำนวน deployments: 12 ครั้ง
- เวลาเฉลี่ย commit → production: 4 วัน
- เวลาเฉลี่ยที่ใช้แก้ production incident: 6 ชั่วโมง
- จำนวน deployments ที่มีปัญหา: 3 จาก 12

**คำนวณ:**
1. Deployment Frequency = ?
2. Lead Time for Changes = ?
3. MTTR = ?
4. Change Failure Rate = ?
5. บริษัทนี้อยู่ใน performance level ไหน?

```
เฉลย:
1. DF = 12/30 = 0.4 deploys/day → High Performer
2. LT = 4 วัน → High Performer
3. MTTR = 6 ชั่วโมง → High Performer
4. CFR = 3/12 = 25% → High Performer

Performance Level: HIGH PERFORMER 🎉
```

---

### แบบฝึกหัดที่ 5: แยกแยะ CI vs CD

**โจทย์:** บอกว่าแต่ละ activity นี้เป็นส่วนหนึ่งของ CI หรือ CD (Delivery/Deployment)?

```
1. รัน unit tests ทุกครั้งที่ push code
2. Deploy อัตโนมัติไป staging เมื่อ tests pass
3. Lint check source code
4. รอ manager approve ก่อน deploy production
5. Deploy อัตโนมัติไป production ทันทีที่ tests pass
6. Build Docker image
7. Run E2E tests บน staging
8. Monitor production metrics หลัง deploy
9. Send Slack notification เมื่อ build fail
10. คนกด "deploy to production" button
```

```
เฉลย:
1. CI (automated testing on code change)
2. CD Delivery/Deployment (auto deploy to staging)
3. CI (code quality check)
4. CD Delivery (manual gate)
5. CD Deployment (full automation)
6. CI (build artifact)
7. CD Delivery (validate before production)
8. CD Deployment (post-deploy monitoring)
9. CI (build feedback)
10. CD Delivery (manual trigger)
```

---

### แบบฝึกหัดที่ 6: ออกแบบ Pipeline

**โจทย์:** คุณเป็น DevOps Engineer ที่ได้รับมอบหมายให้ออกแบบ CI/CD pipeline สำหรับ:
- Python Flask API
- PostgreSQL database
- Redis cache
- Docker deployment บน AWS ECS

**สิ่งที่ต้องออกแบบ:**
1. Pipeline stages (อย่างน้อย 5 stages)
2. Tests แต่ละ stage ต้องรันอะไร?
3. ใช้เครื่องมืออะไรบ้าง?
4. เมื่อ fail แต่ละ stage จะทำอะไร?

```
ตัวอย่างเฉลย:

Pipeline Design:

Stage 1: Code Quality
├── ruff (Python linter)
├── black (formatter check)
├── mypy (type checking)
└── Fail → แจ้ง PR comment

Stage 2: Security
├── bandit (Python security scanner)
├── pip-audit (dependency vulnerabilities)  
└── Fail → block merge, แจ้ง security team

Stage 3: Unit Tests
├── pytest unit tests
├── coverage > 80%
└── Fail → แจ้ง developer, block merge

Stage 4: Integration Tests
├── Start PostgreSQL + Redis containers
├── pytest integration tests
└── Fail → แจ้ง developer, block merge

Stage 5: Build
├── Build Docker image
├── Push to AWS ECR
└── Fail → แจ้ง team

Stage 6: Deploy to Staging
├── Update ECS task definition
├── Rolling deployment
├── Run smoke tests
└── Fail → Auto rollback, แจ้ง team

Stage 7: E2E Tests
├── Run Playwright tests บน staging
└── Fail → แจ้ง QA team

Stage 8: Deploy to Production
├── Manual approval (product owner)
├── Update ECS task definition
├── Canary deployment (10% → 100%)
├── Run smoke tests
└── Fail → Auto rollback, แจ้ง team, page on-call engineer
```

---

### แบบฝึกหัดที่ 7: Research

**โจทย์:** ค้นหาข้อมูลและตอบคำถาม:

1. เข้าไปที่ [DORA State of DevOps Report](https://dora.dev/research/) และหาว่า Elite performers มีลักษณะอะไรที่แตกต่างจาก Low performers นอกจาก 4 metrics หลัก

2. ค้นหา blog post หรือ case study จากบริษัทที่คุณสนใจ (Netflix, Spotify, Etsy, etc.) เกี่ยวกับ CI/CD journey ของพวกเขา และสรุป 5 ข้อที่ได้เรียนรู้

---

## สรุปบทที่ 1

ในบทนี้เราได้เรียนรู้:

```
✅ ปัญหาก่อนมี CI/CD:
   - Integration Hell
   - Manual deployment errors
   - Slow release cycles
   - "Works on my machine"

✅ CI/CD คืออะไร:
   - CI: รวม code บ่อย + test อัตโนมัติ
   - CD Delivery: พร้อม deploy ตลอด + manual gate
   - CD Deployment: deploy อัตโนมัติเต็มรูปแบบ

✅ ส่วนประกอบหลัก:
   - SCM (Git)
   - Build System
   - Test Automation
   - Artifact Repository
   - Deployment Automation
   - Monitoring

✅ DevOps vs CI/CD:
   - DevOps = วัฒนธรรม
   - CI/CD = เครื่องมือ/กระบวนการ

✅ DORA Metrics:
   - Deployment Frequency
   - Lead Time for Changes
   - MTTR
   - Change Failure Rate

✅ เครื่องมือยอดนิยม:
   - GitHub Actions, GitLab CI/CD, Jenkins
```

---

## บทต่อไป

**Part 02: Version Control ด้วย Git** - เราจะเริ่มเรียนรู้ Git ซึ่งเป็นรากฐานของ CI/CD ทั้งหมด

---

## แหล่งเรียนรู้เพิ่มเติม

```
เอกสาร/หนังสือ:
- "Continuous Delivery" - Jez Humble & David Farley
- "The DevOps Handbook" - Gene Kim et al.
- "Accelerate" - Nicole Forsgren et al.
- DORA Research: https://dora.dev

เครื่องมือที่ใช้ในคอร์สนี้:
- GitHub Actions Docs: https://docs.github.com/en/actions
- Docker Docs: https://docs.docker.com
- Kubernetes Docs: https://kubernetes.io/docs
```

---

*ยินดีต้อนรับสู่โลกของ CI/CD! ในบทต่อไปเราจะเริ่ม practice จริงด้วย Git* 🚀

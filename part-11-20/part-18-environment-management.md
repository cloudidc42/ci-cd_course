# Part 18: Environment Management - การจัดการ Environment ใน CI/CD

## สารบัญ

1. [บทนำ - ทำไม Environment Management ถึงสำคัญ](#บทนำ)
2. [Environment Strategy ที่ดี](#environment-strategy)
3. [ความแตกต่างระหว่าง Dev, Staging, และ Production](#ความแตกต่าง-dev-staging-production)
4. [Environment Variables Management](#environment-variables-management)
5. [Dotenv Patterns](#dotenv-patterns)
6. [12-Factor App Methodology](#12-factor-app)
7. [Config vs Secrets](#config-vs-secrets)
8. [Environment-Specific Configurations](#environment-specific-configs)
9. [Infrastructure Per Environment](#infrastructure-per-environment)
10. [Promoting Artifacts Between Environments](#promoting-artifacts)
11. [Environment Cleanup](#environment-cleanup)
12. [Ephemeral Environments สำหรับ PR Reviews](#ephemeral-environments)
13. [GitHub Environments Feature](#github-environments)
14. [แบบฝึกหัด (Exercises)](#exercises)

---

## บทนำ - ทำไม Environment Management ถึงสำคัญ {#บทนำ}

ในการพัฒนา Software สมัยใหม่ การมี Environment ที่แยกจากกันอย่างชัดเจนถือเป็นหัวใจสำคัญของกระบวนการ CI/CD ที่มีประสิทธิภาพ หลายทีมที่เพิ่งเริ่มต้นมักทำผิดพลาดด้วยการ deploy code โดยตรงไปยัง Production โดยไม่ผ่านขั้นตอนการทดสอบที่เพียงพอ ผลลัพธ์คือ outage, data corruption, หรือ security breach ที่อาจส่งผลกระทบต่อธุรกิจอย่างร้ายแรง

### ปัญหาที่เกิดขึ้นเมื่อไม่มี Environment Management ที่ดี

```
ตัวอย่างสถานการณ์จริง:

นักพัฒนา A กำลัง test feature ใหม่บน production database
นักพัฒนา B กำลัง debug โดยการ print sensitive data ใน log
นักพัฒนา C เพิ่งลบ table ที่สำคัญโดยคิดว่าเป็น dev database

ผลลัพธ์: System ล่ม, ข้อมูล customer หาย, ทีมทำงานทั้งคืน
```

### ประโยชน์ของ Environment Management ที่ดี

1. **ลดความเสี่ยง** - ปัญหาถูกค้นพบก่อนที่จะถึง production
2. **เพิ่มความมั่นใจ** - ทีมสามารถ deploy บ่อยขึ้นด้วยความมั่นใจ
3. **Audit Trail** - รู้ว่า version ไหนอยู่ที่ไหน และเปลี่ยนเมื่อไหร่
4. **Rollback ง่ายขึ้น** - มีจุดอ้างอิงที่ชัดเจน
5. **การทำงานร่วมกัน** - ทีมหลายคนทำงานพร้อมกันโดยไม่รบกวนกัน

---

## Environment Strategy ที่ดี {#environment-strategy}

### โมเดล Multi-Environment พื้นฐาน

```
┌─────────────────────────────────────────────────────────────┐
│                    Environment Pipeline                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Developer → [Local] → [Dev] → [Staging] → [Production]   │
│                                                             │
│  ความเสี่ยง:  สูงมาก    สูง      กลาง         ต่ำ           │
│  ความเร็ว:   เร็วมาก   เร็ว    ปานกลาง      ช้า            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Environment ที่ควรมีในโปรเจกต์ขนาดต่างๆ

#### โปรเจกต์ขนาดเล็ก (1-5 คน)
```
Local → Staging → Production
```

#### โปรเจกต์ขนาดกลาง (5-20 คน)
```
Local → Dev → Staging → Production
```

#### โปรเจกต์ขนาดใหญ่ (20+ คน)
```
Local → Dev → Integration → Staging → Pre-Production → Production
          ↑
      Feature Branches (Ephemeral)
```

### หลักการ Promotion

```
Code เคลื่อนที่จาก lower environment ไปยัง higher environment เสมอ
ไม่มีการ skip environment ยกเว้นในกรณีฉุกเฉิน

Artifact เดิม (Binary/Container Image) ที่ผ่านการ test แล้ว
จะถูก promote ไปยัง environment ถัดไป
ไม่ใช่การ build ใหม่ในแต่ละ environment
```

---

## ความแตกต่างระหว่าง Dev, Staging, และ Production {#ความแตกต่าง-dev-staging-production}

### Development Environment

**วัตถุประสงค์:** การพัฒนาและทดสอบ feature ใหม่

```yaml
# dev-environment-spec.yaml
environment: development
characteristics:
  - infrastructure: minimal (เล็กที่สุด, ประหยัดค่าใช้จ่าย)
  - data: synthetic/mock data
  - debugging: enabled (verbose logging, debug tools)
  - availability: 9x5 (business hours only)
  - deployment: automatic (ทุก commit บน develop branch)
  - access: developers only
  - external_services: mocked/stubbed
  
resource_limits:
  cpu: "0.5 cores"
  memory: "512MB"
  storage: "10GB"
  
database:
  type: shared (นักพัฒนาหลายคนใช้ร่วมกัน)
  data_refresh: weekly
  migrations: auto-applied
```

**ตัวอย่าง Dev Docker Compose:**

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app                    # Hot reload
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=true
      - LOG_LEVEL=debug
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=myapp_dev
      - REDIS_HOST=redis
    ports:
      - "3000:3000"
      - "9229:9229"               # Node.js debugger port
    command: npm run dev
    
  postgres:
    image: postgres:14
    environment:
      - POSTGRES_DB=myapp_dev
      - POSTGRES_USER=devuser
      - POSTGRES_PASSWORD=devpass  # ไม่ต้องซับซ้อนใน dev
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
      - ./scripts/seed-dev.sql:/docker-entrypoint-initdb.d/seed.sql
    ports:
      - "5432:5432"               # Expose สำหรับ developer tools
      
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"               # Expose สำหรับ debugging
      
  mailhog:                        # Email testing tool
    image: mailhog/mailhog
    ports:
      - "1025:1025"               # SMTP
      - "8025:8025"               # Web UI

volumes:
  postgres_dev_data:
```

### Staging Environment

**วัตถุประสงค์:** การทดสอบ Integration และ UAT (User Acceptance Testing)

```yaml
# staging-environment-spec.yaml
environment: staging
characteristics:
  - infrastructure: production-like (configuration เหมือน prod แต่ขนาดเล็กกว่า)
  - data: anonymized production data หรือ realistic test data
  - debugging: limited (log errors and warnings only)
  - availability: 24x7
  - deployment: manual approval required
  - access: developers + QA + product team
  - external_services: sandbox/test accounts
  
resource_limits:
  cpu: "2 cores"                  # น้อยกว่า prod แต่ realistic
  memory: "2GB"
  storage: "50GB"
  
database:
  type: dedicated
  data_refresh: biweekly (from anonymized prod snapshot)
  migrations: semi-auto (reviewed before apply)
  
ssl: enabled (ใช้ Let's Encrypt หรือ test certificates)
monitoring: enabled
alerting: developers only (ไม่ notify on-call)
```

**Staging Deploy Script:**

```bash
#!/bin/bash
# scripts/deploy-staging.sh

set -euo pipefail

ENVIRONMENT="staging"
CLUSTER="k8s-staging"
NAMESPACE="myapp-staging"
IMAGE_TAG="${1:-latest}"

echo "=== Deploying to ${ENVIRONMENT} ==="
echo "Image Tag: ${IMAGE_TAG}"

# 1. Validate image exists
echo "Checking image availability..."
docker pull "registry.example.com/myapp:${IMAGE_TAG}" || {
    echo "ERROR: Image not found: ${IMAGE_TAG}"
    exit 1
}

# 2. Run pre-deploy checks
echo "Running pre-deploy checks..."
kubectl --context="${CLUSTER}" get nodes
kubectl --context="${CLUSTER}" -n "${NAMESPACE}" get pods

# 3. Apply database migrations (staging ดำเนินการก่อน)
echo "Running database migrations..."
kubectl --context="${CLUSTER}" -n "${NAMESPACE}" run migration-runner \
    --image="registry.example.com/myapp:${IMAGE_TAG}" \
    --restart=Never \
    --rm -it \
    -- npm run migrate

# 4. Deploy application
echo "Deploying application..."
helm upgrade --install myapp ./charts/myapp \
    --kube-context="${CLUSTER}" \
    --namespace="${NAMESPACE}" \
    --values="./charts/myapp/values-staging.yaml" \
    --set="image.tag=${IMAGE_TAG}" \
    --wait \
    --timeout=5m

# 5. Run smoke tests
echo "Running smoke tests..."
sleep 30  # Wait for pods to be ready
./scripts/smoke-test.sh "${ENVIRONMENT}"

echo "=== Deployment to ${ENVIRONMENT} completed successfully ==="
```

### Production Environment

**วัตถุประสงค์:** ให้บริการ users จริง

```yaml
# production-environment-spec.yaml
environment: production
characteristics:
  - infrastructure: full scale, high availability
  - data: live production data
  - debugging: disabled (security risk)
  - availability: 99.9%+ SLA
  - deployment: manual approval + change window
  - access: ops team only (developers read-only)
  - external_services: live accounts
  
resource_limits:
  cpu: "16 cores"                 # Auto-scaling
  memory: "32GB"
  storage: "500GB" + backups
  
database:
  type: RDS Multi-AZ / Cloud SQL HA
  backups: daily automated + point-in-time recovery
  migrations: manual review + maintenance window
  
ssl: valid commercial certificates
monitoring: full stack (APM, logs, metrics, traces)
alerting: PagerDuty on-call rotation
change_management: required approval from 2+ senior engineers
deployment_window: Tuesday/Thursday 10:00-12:00 (business hours, low traffic)
```

### เปรียบเทียบ Environment แบบ Side-by-Side

| Feature | Development | Staging | Production |
|---------|-------------|---------|------------|
| Data | Fake/Seed | Anonymized Prod | Real |
| SSL | Optional | Yes | Yes (Required) |
| Debug Mode | Enabled | Disabled | Disabled |
| Log Level | DEBUG | WARN | ERROR |
| Auto Deploy | Yes | No | No |
| Approval Required | No | Sometimes | Always |
| Monitoring | Basic | Full | Full + Alerting |
| Backup | No | Weekly | Daily + PITR |
| Uptime SLA | Best Effort | 95% | 99.9%+ |
| Database Migrations | Auto | Semi-auto | Manual |
| External Services | Mocked | Sandbox | Live |

---

## Environment Variables Management {#environment-variables-management}

### ปัญหากับการ hardcode configuration

```python
# ❌ แบบผิด - อย่าทำแบบนี้!
DATABASE_URL = "postgresql://admin:password123@prod-db.example.com/myapp"
SECRET_KEY = "my-super-secret-key-12345"
API_KEY = "sk-live-abcdefghijklmnop"

class Config:
    DEBUG = False  # hardcoded!
    DATABASE_URL = DATABASE_URL
    SECRET_KEY = SECRET_KEY
```

```python
# ✅ แบบถูกต้อง - ใช้ Environment Variables
import os
from typing import Optional

class Config:
    DEBUG: bool = os.getenv("DEBUG", "false").lower() == "true"
    DATABASE_URL: str = os.environ["DATABASE_URL"]  # Required - จะ raise error ถ้าไม่มี
    SECRET_KEY: str = os.environ["SECRET_KEY"]
    API_KEY: Optional[str] = os.getenv("EXTERNAL_API_KEY")  # Optional
    
    # With default values
    MAX_CONNECTIONS: int = int(os.getenv("MAX_CONNECTIONS", "10"))
    TIMEOUT_SECONDS: int = int(os.getenv("TIMEOUT_SECONDS", "30"))
    LOG_LEVEL: str = os.getenv("LOG_LEVEL", "INFO")
```

### Node.js Configuration Pattern

```javascript
// config/index.js
const Joi = require('joi');

// Define schema with validation
const envSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'staging', 'production', 'test')
    .required(),
  
  PORT: Joi.number()
    .default(3000),
  
  DATABASE_URL: Joi.string()
    .uri()
    .required(),
  
  REDIS_URL: Joi.string()
    .uri()
    .default('redis://localhost:6379'),
  
  JWT_SECRET: Joi.string()
    .min(32)
    .required(),
  
  LOG_LEVEL: Joi.string()
    .valid('error', 'warn', 'info', 'debug')
    .default('info'),
  
  RATE_LIMIT_MAX: Joi.number()
    .default(100),
  
  // Optional with conditional validation
  SMTP_HOST: Joi.when('NODE_ENV', {
    is: 'production',
    then: Joi.string().required(),
    otherwise: Joi.string().default('localhost')
  }),
}).unknown(); // Allow additional env vars

// Validate and extract
const { error, value: envVars } = envSchema.validate(process.env);

if (error) {
  throw new Error(`Config validation error: ${error.message}`);
}

module.exports = {
  env: envVars.NODE_ENV,
  port: envVars.PORT,
  isProduction: envVars.NODE_ENV === 'production',
  isDevelopment: envVars.NODE_ENV === 'development',
  
  database: {
    url: envVars.DATABASE_URL,
    pool: {
      min: 2,
      max: envVars.NODE_ENV === 'production' ? 20 : 5,
    },
  },
  
  redis: {
    url: envVars.REDIS_URL,
  },
  
  auth: {
    jwtSecret: envVars.JWT_SECRET,
    jwtExpiry: '7d',
  },
  
  logging: {
    level: envVars.LOG_LEVEL,
  },
};
```

### Go Configuration Pattern

```go
// config/config.go
package config

import (
    "fmt"
    "os"
    "strconv"
    "time"
)

type Config struct {
    App      AppConfig
    Database DatabaseConfig
    Redis    RedisConfig
    Auth     AuthConfig
}

type AppConfig struct {
    Environment string
    Port        int
    Debug       bool
    LogLevel    string
}

type DatabaseConfig struct {
    URL             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime time.Duration
}

type RedisConfig struct {
    URL      string
    Password string
    DB       int
}

type AuthConfig struct {
    JWTSecret  string
    JWTExpiry  time.Duration
}

func Load() (*Config, error) {
    env := getEnvOrDefault("APP_ENV", "development")
    
    dbURL := os.Getenv("DATABASE_URL")
    if dbURL == "" {
        return nil, fmt.Errorf("DATABASE_URL is required")
    }
    
    jwtSecret := os.Getenv("JWT_SECRET")
    if jwtSecret == "" {
        return nil, fmt.Errorf("JWT_SECRET is required")
    }
    if len(jwtSecret) < 32 {
        return nil, fmt.Errorf("JWT_SECRET must be at least 32 characters")
    }
    
    port, err := strconv.Atoi(getEnvOrDefault("PORT", "8080"))
    if err != nil {
        return nil, fmt.Errorf("invalid PORT value: %w", err)
    }
    
    maxOpenConns := 10
    if env == "production" {
        maxOpenConns = 25
    }
    
    return &Config{
        App: AppConfig{
            Environment: env,
            Port:        port,
            Debug:       getEnvOrDefault("DEBUG", "false") == "true",
            LogLevel:    getEnvOrDefault("LOG_LEVEL", "info"),
        },
        Database: DatabaseConfig{
            URL:             dbURL,
            MaxOpenConns:    maxOpenConns,
            MaxIdleConns:    5,
            ConnMaxLifetime: 5 * time.Minute,
        },
        Redis: RedisConfig{
            URL:      getEnvOrDefault("REDIS_URL", "redis://localhost:6379"),
            Password: os.Getenv("REDIS_PASSWORD"),
            DB:       0,
        },
        Auth: AuthConfig{
            JWTSecret: jwtSecret,
            JWTExpiry: 7 * 24 * time.Hour,
        },
    }, nil
}

func getEnvOrDefault(key, defaultVal string) string {
    if val := os.Getenv(key); val != "" {
        return val
    }
    return defaultVal
}
```

---

## Dotenv Patterns {#dotenv-patterns}

### โครงสร้าง .env Files

```
project/
├── .env                    # Default values (commit ได้ - ไม่มี secrets)
├── .env.local              # Local overrides (ไม่ commit - .gitignore)
├── .env.development        # Dev-specific defaults
├── .env.staging            # Staging defaults
├── .env.production         # Production defaults (ค่า non-secret เท่านั้น)
├── .env.test               # Test environment
├── .env.example            # Template ให้ developer ดู (commit ได้)
└── .gitignore
```

### .env.example - Template ที่ Commit ได้

```bash
# .env.example
# Copy this file to .env.local and fill in values
# NEVER commit .env.local or any file with real secrets

# Application
NODE_ENV=development
PORT=3000
APP_URL=http://localhost:3000

# Database (เปลี่ยนเป็น connection string จริงของคุณ)
DATABASE_URL=postgresql://username:password@localhost:5432/myapp_dev

# Redis
REDIS_URL=redis://localhost:6379

# Authentication
JWT_SECRET=your-secret-key-minimum-32-characters-long

# External APIs (ใส่ sandbox/test keys)
STRIPE_PUBLIC_KEY=pk_test_your_key_here
STRIPE_SECRET_KEY=sk_test_your_key_here

# Email
SMTP_HOST=localhost
SMTP_PORT=1025
SMTP_USER=
SMTP_PASS=

# Feature Flags
FEATURE_NEW_DASHBOARD=false
FEATURE_BETA_API=false

# Monitoring (optional in dev)
SENTRY_DSN=
```

### .gitignore สำหรับ .env files

```gitignore
# .gitignore

# Environment files with real values
.env
.env.local
.env.*.local
.env.development.local
.env.test.local
.env.production.local

# ให้ commit ได้ (ไม่มี real secrets)
!.env.example
!.env.test         # ถ้าใช้ mock values เท่านั้น
```

### Python dotenv Pattern

```python
# config/settings.py
import os
from pathlib import Path
from dotenv import load_dotenv

# Load .env files ตาม priority
env = os.getenv("APP_ENV", "development")

# Load ตาม order (ทีหลังทับก่อน)
load_dotenv(".env")                    # Default
load_dotenv(f".env.{env}")             # Environment-specific  
load_dotenv(".env.local", override=True)  # Local override (highest priority)

# Validate required variables
REQUIRED_VARS = [
    "DATABASE_URL",
    "SECRET_KEY",
    "ALLOWED_HOSTS",
]

missing = [var for var in REQUIRED_VARS if not os.getenv(var)]
if missing:
    raise EnvironmentError(
        f"Missing required environment variables: {', '.join(missing)}\n"
        f"Please check your .env file or environment configuration."
    )

# Settings
DATABASE_URL = os.environ["DATABASE_URL"]
SECRET_KEY = os.environ["SECRET_KEY"]
DEBUG = os.getenv("DEBUG", "false").lower() == "true"
ALLOWED_HOSTS = os.environ["ALLOWED_HOSTS"].split(",")

# Environment-specific settings
if env == "production":
    SESSION_COOKIE_SECURE = True
    CSRF_COOKIE_SECURE = True
    SECURE_HSTS_SECONDS = 31536000
    SECURE_CONTENT_TYPE_NOSNIFF = True
```

### Node.js dotenv-flow Pattern

```javascript
// ecosystem.config.js (PM2)
module.exports = {
  apps: [{
    name: 'myapp',
    script: 'src/server.js',
    env: {
      NODE_ENV: 'development',
    },
    env_staging: {
      NODE_ENV: 'staging',
    },
    env_production: {
      NODE_ENV: 'production',
    }
  }]
};

// src/config/env.js
require('dotenv-flow').config({
  // จะ load ตาม order:
  // .env
  // .env.local
  // .env.${NODE_ENV}
  // .env.${NODE_ENV}.local
  silent: true,           // ไม่ error ถ้าไม่มีไฟล์
  path: process.cwd(),
});
```

---

## 12-Factor App Methodology {#12-factor-app}

12-Factor App เป็น methodology ที่ถูกพัฒนาโดย Heroku เพื่อสร้าง web applications ที่ scalable และ maintainable

### Factor III: Config - Store config in the environment

```
หลักการ: Configuration ที่แตกต่างกันระหว่าง deployments (dev/staging/prod)
ต้องถูกเก็บไว้ใน environment ไม่ใช่ใน code

Test: สามารถ open source code ได้โดยไม่มีความเสี่ยง
      (ไม่มี credentials, private URLs, หรือ secrets ใน code)
```

### การ Implement 12-Factor Config

```python
# ❌ Factor Violation - config ใน code
class Config:
    DATABASE_HOST = "db.internal.example.com"
    DATABASE_PORT = 5432
    API_BASE_URL = "https://api.partner.com/v2"
    
# ✅ Factor Compliant - config ใน environment
import os

class Config:
    DATABASE_HOST = os.environ["DATABASE_HOST"]
    DATABASE_PORT = int(os.getenv("DATABASE_PORT", "5432"))
    API_BASE_URL = os.environ["API_BASE_URL"]
```

### การแยก Environment-Specific จาก App Code

```yaml
# kubernetes/configmap-dev.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp-dev
data:
  NODE_ENV: "development"
  LOG_LEVEL: "debug"
  DB_POOL_MIN: "2"
  DB_POOL_MAX: "5"
  CACHE_TTL: "60"
  RATE_LIMIT_MAX: "1000"    # Generous limits in dev
  FEATURE_NEW_UI: "true"    # Can test features in dev
  
---
# kubernetes/configmap-production.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp-prod
data:
  NODE_ENV: "production"
  LOG_LEVEL: "error"
  DB_POOL_MIN: "5"
  DB_POOL_MAX: "25"
  CACHE_TTL: "3600"
  RATE_LIMIT_MAX: "100"     # Strict limits in prod
  FEATURE_NEW_UI: "false"   # Feature flags controlled per env
```

---

## Config vs Secrets {#config-vs-secrets}

### ความแตกต่างระหว่าง Config และ Secrets

```
CONFIG (สามารถ commit ได้, ไม่ sensitive):
├── Application settings (port, timeout, feature flags)
├── Infrastructure endpoints (database host, redis host)
├── Log levels
├── Feature flags
└── Rate limits

SECRETS (ห้าม commit เด็ดขาด):
├── Passwords & credentials
├── API keys & tokens
├── Private certificates & keys
├── Encryption keys
├── Connection strings ที่มี credentials
└── Session secrets
```

### Kubernetes: แยก ConfigMap จาก Secret

```yaml
# ✅ ConfigMap - สำหรับค่าที่ไม่ sensitive
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  APP_PORT: "3000"
  DB_HOST: "postgres.myapp-prod.svc.cluster.local"
  DB_PORT: "5432"
  DB_NAME: "myapp_production"
  REDIS_HOST: "redis.myapp-prod.svc.cluster.local"
  LOG_LEVEL: "warn"
  
---
# ✅ Secret - สำหรับค่า sensitive (encoded แต่ไม่ encrypted by default)
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:                           # stringData จะ auto-encode เป็น base64
  DB_PASSWORD: "super-secret-password"
  JWT_SECRET: "minimum-32-chars-secret-key-here"
  STRIPE_SECRET_KEY: "sk_live_..."
  
---
# Deployment ที่ใช้ทั้ง ConfigMap และ Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          envFrom:
            - configMapRef:
                name: myapp-config     # Load ทุก key จาก ConfigMap
            - secretRef:
                name: myapp-secrets    # Load ทุก key จาก Secret
          env:
            # Override specific values
            - name: DB_URL
              value: "postgresql://$(DB_USER):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)"
```

---

## Environment-Specific Configurations {#environment-specific-configs}

### Helm Values Files Per Environment

```
charts/myapp/
├── Chart.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
├── values.yaml           # Default values
├── values-dev.yaml       # Dev overrides
├── values-staging.yaml   # Staging overrides
└── values-production.yaml # Production overrides
```

```yaml
# charts/myapp/values.yaml (defaults)
replicaCount: 1

image:
  repository: registry.example.com/myapp
  tag: latest
  pullPolicy: Always

service:
  type: ClusterIP
  port: 3000

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 3

ingress:
  enabled: false
  
database:
  host: postgres
  port: 5432
  
redis:
  host: redis
  port: 6379

---
# charts/myapp/values-production.yaml
replicaCount: 3              # HA setup

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: true
  hostname: api.example.com
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
  tls:
    - secretName: myapp-tls
      hosts:
        - api.example.com

database:
  host: postgres.myapp-prod.svc.cluster.local
  
redis:
  host: redis.myapp-prod.svc.cluster.local

podDisruptionBudget:
  enabled: true
  minAvailable: 2

affinity:                    # Spread pods across nodes
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
```

### GitHub Actions per-environment variables

```yaml
# .github/workflows/deploy.yml
name: Deploy Application

on:
  push:
    branches:
      - develop
      - staging
      - main

jobs:
  deploy:
    name: Deploy to ${{ matrix.environment }}
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        include:
          - branch: develop
            environment: development
            cluster: dev-cluster
            namespace: myapp-dev
          - branch: staging
            environment: staging
            cluster: staging-cluster
            namespace: myapp-staging
          - branch: main
            environment: production
            cluster: prod-cluster
            namespace: myapp-prod
    
    # Only run for matching branch
    if: github.ref == format('refs/heads/{0}', matrix.branch)
    
    environment: ${{ matrix.environment }}  # GitHub Environment
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBECONFIG }}
          context: ${{ matrix.cluster }}
      
      - name: Deploy with Helm
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}        # จาก GitHub Environment secrets
          JWT_SECRET: ${{ secrets.JWT_SECRET }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace ${{ matrix.namespace }} \
            --values ./charts/myapp/values-${{ matrix.environment }}.yaml \
            --set image.tag=${IMAGE_TAG} \
            --set secrets.dbPassword="${DB_PASSWORD}" \
            --set secrets.jwtSecret="${JWT_SECRET}" \
            --wait --timeout=5m
```

---

## Infrastructure Per Environment {#infrastructure-per-environment}

### Terraform ที่รองรับ Multi-Environment

```
infrastructure/
├── modules/
│   ├── networking/
│   ├── kubernetes/
│   ├── database/
│   └── monitoring/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
└── shared/
    └── backend.tf
```

```hcl
# infrastructure/environments/production/main.tf
terraform {
  required_version = ">= 1.0"
  
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "environments/production/terraform.tfstate"
    region = "ap-southeast-1"
    
    # State locking
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

module "networking" {
  source = "../../modules/networking"
  
  environment     = "production"
  vpc_cidr        = "10.0.0.0/16"
  azs             = ["ap-southeast-1a", "ap-southeast-1b", "ap-southeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = false      # HA: NAT gateway ทุก AZ
  
  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    CostCenter  = "platform"
  }
}

module "eks" {
  source = "../../modules/kubernetes"
  
  cluster_name    = "myapp-prod"
  cluster_version = "1.28"
  vpc_id          = module.networking.vpc_id
  subnet_ids      = module.networking.private_subnets
  
  node_groups = {
    general = {
      instance_types = ["m5.xlarge"]
      min_size       = 3
      max_size       = 20
      desired_size   = 5
      
      taints = []
      labels = {
        role = "general"
      }
    }
    
    memory_optimized = {
      instance_types = ["r5.2xlarge"]
      min_size       = 1
      max_size       = 5
      desired_size   = 2
      
      labels = {
        role = "memory-intensive"
      }
    }
  }
}

module "rds" {
  source = "../../modules/database"
  
  identifier     = "myapp-prod"
  engine         = "postgres"
  engine_version = "14.9"
  instance_class = "db.r5.2xlarge"
  
  # Multi-AZ สำหรับ HA
  multi_az              = true
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "Mon:04:00-Mon:05:00"
  
  # Storage
  allocated_storage     = 100
  max_allocated_storage = 500    # Auto-scaling storage
  storage_encrypted     = true
  
  # Networking
  vpc_security_group_ids = [module.networking.rds_security_group_id]
  db_subnet_group_name   = module.networking.database_subnet_group_name
  
  # Deletion protection
  deletion_protection = true
  
  tags = {
    Environment = "production"
  }
}
```

```hcl
# infrastructure/environments/dev/main.tf
# ขนาดเล็กกว่า - ประหยัดค่าใช้จ่าย
module "eks" {
  source = "../../modules/kubernetes"
  
  cluster_name    = "myapp-dev"
  cluster_version = "1.28"
  
  node_groups = {
    general = {
      instance_types = ["t3.medium"]   # ราคาถูกกว่า
      min_size       = 1
      max_size       = 3
      desired_size   = 2
    }
  }
}

module "rds" {
  source = "../../modules/database"
  
  identifier     = "myapp-dev"
  instance_class = "db.t3.medium"    # ราคาถูกกว่า
  
  multi_az                  = false  # ไม่ต้อง HA ใน dev
  backup_retention_period   = 1
  deletion_protection       = false  # ลบได้ใน dev
  
  # Shared instance options
  performance_insights_enabled = false
}
```

---

## Promoting Artifacts Between Environments {#promoting-artifacts}

### หลักการ Artifact Promotion

```
Build Once → Test → Promote → Test → Promote → Deploy

Container Image ที่ build ครั้งเดียวจาก source code
จะถูก promote (tag ใหม่) เพื่อย้ายไปยัง environment ถัดไป
ไม่ใช่การ build ใหม่ ซึ่งอาจได้ binary ที่ต่างกัน

Image Tags ตัวอย่าง:
myapp:abc123def    ← SHA ของ commit (immutable)
myapp:dev          ← ชี้ไปยัง image ที่ผ่าน dev
myapp:staging      ← ชี้ไปยัง image ที่ผ่าน staging
myapp:production   ← ชี้ไปยัง image ที่ถูก deploy ไป prod
myapp:v2.1.0       ← Semantic version tag
```

### GitHub Actions: Artifact Promotion Pipeline

```yaml
# .github/workflows/promote.yml
name: Promote Artifact to Environment

on:
  workflow_dispatch:
    inputs:
      source_environment:
        description: 'Source environment'
        required: true
        type: choice
        options:
          - staging
          - pre-production
      target_environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - pre-production
          - production
      image_tag:
        description: 'Image tag to promote (e.g., abc123def)'
        required: true

jobs:
  validate:
    name: Validate Promotion
    runs-on: ubuntu-latest
    outputs:
      validated: ${{ steps.check.outputs.validated }}
    
    steps:
      - name: Check tag exists in source
        id: check
        run: |
          # Verify image exists in registry
          IMAGE="registry.example.com/myapp:${{ inputs.image_tag }}"
          
          if docker manifest inspect "${IMAGE}" > /dev/null 2>&1; then
            echo "✅ Image found: ${IMAGE}"
            echo "validated=true" >> $GITHUB_OUTPUT
          else
            echo "❌ Image not found: ${IMAGE}"
            echo "validated=false" >> $GITHUB_OUTPUT
            exit 1
          fi
      
      - name: Verify passed all tests in source env
        run: |
          # Check deployment history
          echo "Checking test results for ${{ inputs.image_tag }} in ${{ inputs.source_environment }}..."
          # คุณอาจ query deployment database หรือ check previous job results

  promote:
    name: Promote to ${{ inputs.target_environment }}
    needs: validate
    runs-on: ubuntu-latest
    environment: ${{ inputs.target_environment }}
    
    steps:
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: registry.example.com
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      
      - name: Promote image tag
        run: |
          SOURCE_IMAGE="registry.example.com/myapp:${{ inputs.image_tag }}"
          TARGET_TAG="${{ inputs.target_environment }}"
          TARGET_IMAGE="registry.example.com/myapp:${TARGET_TAG}"
          
          echo "Promoting: ${SOURCE_IMAGE} → ${TARGET_IMAGE}"
          
          # Pull source
          docker pull "${SOURCE_IMAGE}"
          
          # Tag for target environment
          docker tag "${SOURCE_IMAGE}" "${TARGET_IMAGE}"
          
          # Also create version-specific tag
          TIMESTAMP=$(date +%Y%m%d-%H%M%S)
          VERSION_TAG="registry.example.com/myapp:${TARGET_TAG}-${TIMESTAMP}"
          docker tag "${SOURCE_IMAGE}" "${VERSION_TAG}"
          
          # Push both tags
          docker push "${TARGET_IMAGE}"
          docker push "${VERSION_TAG}"
          
          echo "✅ Promoted to ${TARGET_TAG}"
          echo "IMAGE_TAG=${{ inputs.image_tag }}" >> $GITHUB_ENV
      
      - name: Deploy to environment
        run: |
          ./scripts/deploy.sh \
            --environment "${{ inputs.target_environment }}" \
            --image-tag "${IMAGE_TAG}"
      
      - name: Run post-deploy tests
        run: |
          ./scripts/integration-tests.sh "${{ inputs.target_environment }}"
      
      - name: Create deployment record
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.repos.createDeployment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              ref: '${{ inputs.image_tag }}',
              environment: '${{ inputs.target_environment }}',
              description: 'Promoted from ${{ inputs.source_environment }}',
              auto_merge: false,
              required_contexts: [],
            });
```

---

## Environment Cleanup {#environment-cleanup}

### Automated Cleanup Strategy

```yaml
# .github/workflows/cleanup.yml
name: Environment Cleanup

on:
  schedule:
    - cron: '0 2 * * *'    # ทุกวันตี 2
  workflow_dispatch:
    inputs:
      dry_run:
        description: 'Dry run (preview only)'
        type: boolean
        default: true

jobs:
  cleanup-pr-environments:
    name: Cleanup Closed PR Environments
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Find and remove closed PR namespaces
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DRY_RUN: ${{ inputs.dry_run || 'false' }}
        run: |
          #!/bin/bash
          set -euo pipefail
          
          echo "Finding stale PR environments..."
          
          # Get all PR namespaces in cluster
          PR_NAMESPACES=$(kubectl get namespaces -l "type=pr-preview" -o jsonpath='{.items[*].metadata.name}')
          
          for NS in $PR_NAMESPACES; do
            # Extract PR number from namespace name (e.g., "pr-preview-123")
            PR_NUMBER=$(echo "${NS}" | grep -oP 'pr-preview-\K[0-9]+')
            
            # Check if PR is still open
            PR_STATE=$(gh api repos/${{ github.repository }}/pulls/${PR_NUMBER} --jq '.state' 2>/dev/null || echo "not_found")
            
            if [[ "${PR_STATE}" != "open" ]]; then
              echo "PR #${PR_NUMBER} is ${PR_STATE}, cleaning up namespace: ${NS}"
              
              if [[ "${DRY_RUN}" == "true" ]]; then
                echo "[DRY RUN] Would delete namespace: ${NS}"
              else
                kubectl delete namespace "${NS}" --wait=false
                echo "✅ Deleted namespace: ${NS}"
              fi
            else
              echo "PR #${PR_NUMBER} is still open, keeping namespace: ${NS}"
            fi
          done

  cleanup-old-images:
    name: Cleanup Old Container Images
    runs-on: ubuntu-latest
    
    steps:
      - name: Remove dev images older than 7 days
        run: |
          # ลบ images ที่ไม่ได้ใช้และเก่ากว่า 7 วัน
          CUTOFF_DATE=$(date -d '7 days ago' +%s)
          
          # List images and delete old ones
          # ตัวอย่างสำหรับ GitHub Container Registry
          gh api \
            -H "Accept: application/vnd.github+json" \
            "/user/packages/container/myapp/versions" \
            --jq '.[] | select(.metadata.container.tags[] | startswith("dev-"))' | \
          while read -r version; do
            CREATED=$(echo "${version}" | jq -r '.created_at')
            CREATED_TS=$(date -d "${CREATED}" +%s)
            
            if [[ ${CREATED_TS} -lt ${CUTOFF_DATE} ]]; then
              VERSION_ID=$(echo "${version}" | jq -r '.id')
              echo "Deleting old dev image version: ${VERSION_ID}"
              gh api -X DELETE "/user/packages/container/myapp/versions/${VERSION_ID}"
            fi
          done
```

---

## Ephemeral Environments สำหรับ PR Reviews {#ephemeral-environments}

### Preview Environments คืออะไร

```
เมื่อนักพัฒนาเปิด Pull Request
→ CI/CD สร้าง Environment ชั่วคราว
→ Deploy code จาก PR branch ไปยัง environment นั้น
→ สร้าง URL สำหรับ review (เช่น: pr-123.preview.example.com)
→ QA/Product Manager เข้ามา review ได้ทันที
→ เมื่อ PR merged หรือ closed → environment ถูกลบอัตโนมัติ
```

### GitHub Actions: Ephemeral PR Environment

```yaml
# .github/workflows/pr-preview.yml
name: PR Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
    branches:
      - main
      - develop

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  deploy-preview:
    name: Deploy Preview Environment
    runs-on: ubuntu-latest
    
    # ไม่ทำงานเมื่อ PR ถูกปิด (จัดการโดย cleanup job)
    if: github.event.action != 'closed'
    
    permissions:
      contents: read
      packages: write
      deployments: write
      pull-requests: write
    
    environment:
      name: pr-preview-${{ github.event.number }}
      url: https://pr-${{ github.event.number }}.preview.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push PR image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:pr-${{ github.event.number }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
            pr.number=${{ github.event.number }}
      
      - name: Create Kubernetes namespace for PR
        run: |
          PR_NUMBER="${{ github.event.number }}"
          NAMESPACE="pr-preview-${PR_NUMBER}"
          
          kubectl apply -f - <<EOF
          apiVersion: v1
          kind: Namespace
          metadata:
            name: ${NAMESPACE}
            labels:
              type: pr-preview
              pr-number: "${PR_NUMBER}"
              managed-by: github-actions
            annotations:
              github.com/pr-url: "${{ github.event.pull_request.html_url }}"
              created-at: "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
          EOF
      
      - name: Deploy PR application
        run: |
          PR_NUMBER="${{ github.event.number }}"
          NAMESPACE="pr-preview-${PR_NUMBER}"
          IMAGE_TAG="${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:pr-${PR_NUMBER}"
          
          helm upgrade --install "myapp-pr-${PR_NUMBER}" ./charts/myapp \
            --namespace "${NAMESPACE}" \
            --create-namespace \
            --values ./charts/myapp/values-preview.yaml \
            --set "image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}" \
            --set "image.tag=pr-${PR_NUMBER}" \
            --set "ingress.hostname=pr-${PR_NUMBER}.preview.example.com" \
            --set "env.DATABASE_URL=${{ secrets.PREVIEW_DB_URL }}" \
            --set "env.JWT_SECRET=${{ secrets.PREVIEW_JWT_SECRET }}" \
            --wait --timeout=5m
      
      - name: Comment preview URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            const prNumber = ${{ github.event.number }};
            const previewUrl = `https://pr-${prNumber}.preview.example.com`;
            const sha = '${{ github.sha }}'.substring(0, 7);
            
            // ลบ comment เก่าก่อน (ถ้ามี)
            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: prNumber,
            });
            
            const botComment = comments.data.find(c => 
              c.user.type === 'Bot' && c.body.includes('🚀 Preview Environment')
            );
            
            if (botComment) {
              await github.rest.issues.deleteComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
              });
            }
            
            // สร้าง comment ใหม่
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: prNumber,
              body: `## 🚀 Preview Environment Ready!
              
              | ข้อมูล | ค่า |
              |--------|-----|
              | 🌐 URL | [pr-${prNumber}.preview.example.com](${previewUrl}) |
              | 📝 Commit | \`${sha}\` |
              | ⏰ Updated | ${new Date().toUTCString()} |
              
              > Preview environment จะถูกลบโดยอัตโนมัติเมื่อ PR ถูกปิดหรือ merge
              `,
            });
  
  cleanup-preview:
    name: Cleanup Preview Environment
    runs-on: ubuntu-latest
    
    if: github.event.action == 'closed'
    
    steps:
      - name: Delete PR namespace
        run: |
          PR_NUMBER="${{ github.event.number }}"
          NAMESPACE="pr-preview-${PR_NUMBER}"
          
          echo "Cleaning up preview environment: ${NAMESPACE}"
          
          # Delete Helm release first
          helm uninstall "myapp-pr-${PR_NUMBER}" \
            --namespace "${NAMESPACE}" \
            --wait || true
          
          # Delete namespace
          kubectl delete namespace "${NAMESPACE}" --wait=false || true
          
          echo "✅ Preview environment cleaned up"
      
      - name: Delete PR container image
        run: |
          PR_NUMBER="${{ github.event.number }}"
          
          # Delete PR-specific image
          gh api -X DELETE \
            "/users/${{ github.repository_owner }}/packages/container/myapp/versions" \
            --field "package_version_id=pr-${PR_NUMBER}" || true
      
      - name: Update PR comment
        uses: actions/github-script@v7
        with:
          script: |
            const prNumber = ${{ github.event.number }};
            
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: prNumber,
              body: `## 🗑️ Preview Environment Removed
              
              Preview environment สำหรับ PR #${prNumber} ถูกลบแล้ว
              `,
            });
```

---

## GitHub Environments Feature {#github-environments}

### การตั้งค่า GitHub Environments

GitHub Environments ให้คุณกำหนด:
- **Protection rules** - ต้องได้รับ approval ก่อน deploy
- **Required reviewers** - ระบุ person หรือ team ที่ต้อง approve
- **Wait timer** - รอก่อน deploy
- **Deployment branches** - กำหนดว่า branch ไหน deploy ได้
- **Environment secrets** - secrets เฉพาะของ environment นั้น

### การใช้งาน GitHub Environments ใน Workflow

```yaml
# .github/workflows/deploy-with-environments.yml
name: Deploy with GitHub Environments

on:
  push:
    branches:
      - main

jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      
    steps:
      - uses: actions/checkout@v4
      
      - name: Docker meta
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=commit-
            type=raw,value=latest
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
  
  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    
    # การใช้ GitHub Environment
    environment:
      name: staging                        # ชื่อ environment ใน GitHub
      url: https://staging.example.com     # URL จะแสดงใน GitHub UI
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        env:
          # secrets ที่กำหนดใน GitHub Environment "staging"
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          echo "Deploying to staging..."
          ./scripts/deploy.sh staging ${{ github.sha }}
      
      - name: Run smoke tests
        run: |
          ./scripts/smoke-test.sh https://staging.example.com
  
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    
    # Production environment มี protection rules:
    # - Required reviewers: 2 senior engineers
    # - Wait timer: 5 minutes
    # - Only main branch allowed
    environment:
      name: production
      url: https://example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Notify deployment starting
        uses: slackapi/slack-github-action@v1.26.0
        with:
          channel-id: 'deployments'
          payload: |
            {
              "text": "🚀 Production deployment starting",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment* started by ${{ github.actor }}\n*Commit:* ${{ github.sha }}\n*Branch:* ${{ github.ref }}"
                  }
                }
              ]
            }
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      
      - name: Deploy to production
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}         # Production secrets
          API_KEY: ${{ secrets.API_KEY }}
          MONITORING_KEY: ${{ secrets.MONITORING_KEY }}
        run: |
          ./scripts/deploy.sh production ${{ github.sha }}
      
      - name: Verify deployment
        run: |
          ./scripts/verify-deployment.sh https://example.com
          
      - name: Create GitHub Release
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.repos.createRelease({
              owner: context.repo.owner,
              repo: context.repo.repo,
              tag_name: `v${new Date().toISOString().split('T')[0]}-${context.sha.substring(0, 7)}`,
              name: `Production Release ${new Date().toISOString().split('T')[0]}`,
              body: `Deployed commit ${context.sha} to production`,
              draft: false,
              prerelease: false,
            });
```

### การกำหนด Environment via GitHub API / Terraform

```hcl
# terraform/github-environments.tf
resource "github_repository_environment" "staging" {
  repository  = github_repository.myapp.name
  environment = "staging"
  
  reviewers {
    teams = [data.github_team.qa.id]
  }
  
  deployment_branch_policy {
    protected_branches     = false
    custom_branch_policies = true
  }
}

resource "github_repository_environment" "production" {
  repository  = github_repository.myapp.name
  environment = "production"
  
  # ต้องมี approval จาก senior engineers
  reviewers {
    users = [
      data.github_user.senior_engineer_1.id,
      data.github_user.senior_engineer_2.id,
    ]
    teams = [data.github_team.platform.id]
  }
  
  # รอ 5 นาทีก่อน deploy
  wait_timer = 5
  
  # เฉพาะ main branch เท่านั้น
  deployment_branch_policy {
    protected_branches     = true
    custom_branch_policies = false
  }
}

# Environment Secrets
resource "github_actions_environment_secret" "prod_db_password" {
  repository      = github_repository.myapp.name
  environment     = github_repository_environment.production.environment
  secret_name     = "DB_PASSWORD"
  plaintext_value = var.prod_db_password
}
```

---

## แบบฝึกหัด (Exercises) {#exercises}

### Exercise 1: สร้าง Multi-Environment Docker Compose Setup

สร้างไฟล์ docker-compose สำหรับ 3 environments โดยแต่ละ environment มีความแตกต่างกัน:

```
งาน:
1. สร้าง docker-compose.base.yml ที่มี common services
2. สร้าง docker-compose.dev.yml ที่ extends base และ add debug tools
3. สร้าง docker-compose.staging.yml ที่เหมือน production แต่ขนาดเล็กกว่า
4. สร้าง .env.example ที่มี placeholders สำหรับทุก environment
5. เขียน script ที่ start ได้ด้วย: ./scripts/start.sh dev|staging
```

**Solution:**

```yaml
# docker-compose.base.yml
version: '3.8'

services:
  app:
    image: ${APP_IMAGE:-myapp:latest}
    environment:
      - NODE_ENV=${NODE_ENV}
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis
    restart: unless-stopped
    
  postgres:
    image: postgres:14-alpine
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
    
  redis:
    image: redis:7-alpine
    restart: unless-stopped

volumes:
  postgres_data:
```

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - DEBUG=true
      - LOG_LEVEL=debug
    ports:
      - "3000:3000"
      - "9229:9229"     # debugger
    command: npm run dev
    
  postgres:
    ports:
      - "5432:5432"     # expose สำหรับ DB tools
      
  redis:
    ports:
      - "6379:6379"     # expose สำหรับ Redis tools
      
  # Dev-only services
  adminer:              # Database UI
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      
  mailhog:              # Email testing
    image: mailhog/mailhog
    ports:
      - "1025:1025"
      - "8025:8025"
```

```bash
#!/bin/bash
# scripts/start.sh

ENV=${1:-dev}

case "${ENV}" in
  dev)
    echo "Starting development environment..."
    docker compose -f docker-compose.base.yml \
                   -f docker-compose.dev.yml \
                   --env-file .env.development \
                   up --build
    ;;
  staging)
    echo "Starting staging environment..."
    docker compose -f docker-compose.base.yml \
                   --env-file .env.staging \
                   up -d
    ;;
  *)
    echo "Usage: $0 [dev|staging]"
    exit 1
    ;;
esac
```

### Exercise 2: GitHub Actions Multi-Environment Pipeline

สร้าง GitHub Actions workflow ที่:
1. Build Docker image เมื่อ push to main
2. Deploy to staging อัตโนมัติ
3. รอ manual approval ก่อน deploy to production
4. Run smoke tests หลัง deploy แต่ละ environment

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
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
        run: npm test
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          NODE_ENV: test
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
  
  build:
    name: Build and Push Image
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to GHCR
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
            type=sha,prefix=
            type=raw,value=latest
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        env:
          IMAGE_TAG: ${{ needs.build.outputs.image_tag }}
          KUBECONFIG_DATA: ${{ secrets.STAGING_KUBECONFIG }}
          DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
        run: |
          echo "${KUBECONFIG_DATA}" | base64 -d > /tmp/kubeconfig
          export KUBECONFIG=/tmp/kubeconfig
          
          helm upgrade --install myapp ./charts/myapp \
            --namespace myapp-staging \
            --values ./charts/myapp/values-staging.yaml \
            --set image.tag=${IMAGE_TAG} \
            --set secrets.dbPassword="${DB_PASSWORD}" \
            --wait --timeout=5m
      
      - name: Smoke test staging
        run: |
          sleep 15
          curl -f https://staging.myapp.example.com/health || exit 1
          echo "✅ Staging smoke test passed"
  
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        env:
          IMAGE_TAG: ${{ needs.build.outputs.image_tag }}
          KUBECONFIG_DATA: ${{ secrets.PROD_KUBECONFIG }}
          DB_PASSWORD: ${{ secrets.PROD_DB_PASSWORD }}
        run: |
          echo "${KUBECONFIG_DATA}" | base64 -d > /tmp/kubeconfig
          export KUBECONFIG=/tmp/kubeconfig
          
          helm upgrade --install myapp ./charts/myapp \
            --namespace myapp-production \
            --values ./charts/myapp/values-production.yaml \
            --set image.tag=${IMAGE_TAG} \
            --set secrets.dbPassword="${DB_PASSWORD}" \
            --wait --timeout=10m
      
      - name: Smoke test production
        run: |
          sleep 30
          curl -f https://myapp.example.com/health || exit 1
          echo "✅ Production smoke test passed"
```

### Exercise 3: Ephemeral Environment Setup

ตั้งค่า ephemeral environments สำหรับ PR reviews โดย:
1. Deploy เมื่อ PR เปิด
2. Update เมื่อ push to PR branch
3. Comment URL บน PR
4. Cleanup เมื่อ PR ปิด

ลองทำ Exercise นี้โดยใช้ namespace ชื่อ `pr-preview-{PR_NUMBER}` บน Kubernetes

### Exercise 4: Environment Variable Validation

เขียน script หรือ code ที่:
1. ตรวจสอบว่า environment variables ที่จำเป็นทั้งหมดมีอยู่
2. Validate ค่าว่ามี format ถูกต้อง (URL, port range, etc.)
3. แสดง error message ที่ชัดเจนเมื่อมีค่าผิดพลาด

```python
# utils/config_validator.py
import os
import re
from urllib.parse import urlparse
from dataclasses import dataclass
from typing import Optional, List, Callable

@dataclass
class ConfigVar:
    name: str
    required: bool = True
    validator: Optional[Callable] = None
    description: str = ""
    default: Optional[str] = None

def validate_url(value: str) -> bool:
    try:
        result = urlparse(value)
        return all([result.scheme, result.netloc])
    except:
        return False

def validate_port(value: str) -> bool:
    try:
        port = int(value)
        return 1 <= port <= 65535
    except:
        return False

def validate_min_length(min_len: int):
    return lambda v: len(v) >= min_len

REQUIRED_CONFIG = [
    ConfigVar("DATABASE_URL", validator=validate_url, description="PostgreSQL connection URL"),
    ConfigVar("REDIS_URL", validator=validate_url, description="Redis connection URL"),
    ConfigVar("SECRET_KEY", validator=validate_min_length(32), description="App secret key (min 32 chars)"),
    ConfigVar("PORT", required=False, validator=validate_port, default="3000", description="HTTP port"),
]

def validate_config() -> List[str]:
    errors = []
    
    for var in REQUIRED_CONFIG:
        value = os.getenv(var.name, var.default)
        
        if var.required and not value:
            errors.append(f"Missing required: {var.name} - {var.description}")
            continue
        
        if value and var.validator:
            if not var.validator(value):
                errors.append(f"Invalid value for {var.name}: '{value}' - {var.description}")
    
    return errors

if __name__ == "__main__":
    errors = validate_config()
    
    if errors:
        print("❌ Configuration errors found:")
        for error in errors:
            print(f"  • {error}")
        print("\nPlease check your environment variables.")
        exit(1)
    else:
        print("✅ Configuration validated successfully")
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **Environment Strategy** - วิธีออกแบบ environments ที่เหมาะสมกับขนาดทีม
2. **ความแตกต่างของ Environments** - Dev ต้องเร็วและ debug ได้, Staging เหมือน Prod, Production ต้อง stable
3. **Environment Variables** - การจัดการ config อย่างถูกต้องตาม 12-Factor principles
4. **Dotenv Patterns** - การใช้ .env files อย่างปลอดภัย
5. **Config vs Secrets** - อะไร commit ได้, อะไรห้าม
6. **Helm Values** - การจัดการ config per environment
7. **Artifact Promotion** - Build once, promote ไม่ใช่ build ใหม่
8. **Ephemeral Environments** - PR Preview environments
9. **GitHub Environments** - Protection rules และ approvals

### Key Takeaways

- **ห้าม hardcode** configuration values ใน code
- **Commit ได้** เฉพาะ .env.example และ config templates ที่ไม่มี secrets
- **Separate concerns** - config แยกจาก secrets แยกจาก application code
- **Promote, don't rebuild** - artifact เดิมไปยัง environment ถัดไป
- **Automate cleanup** - ephemeral environments ต้องมี lifecycle management

---

*บทต่อไป: Part 19 - Secrets Management & Security*

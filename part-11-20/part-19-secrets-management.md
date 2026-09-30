# Part 19: Secrets Management & Security - การจัดการ Secrets และความปลอดภัยใน CI/CD

## สารบัญ

1. [บทนำ - ทำไม Secrets ต้องระวัง](#บทนำ)
2. [ประเภทของ Secrets](#ประเภทของ-secrets)
3. [GitHub Secrets](#github-secrets)
4. [GitLab Variables](#gitlab-variables)
5. [HashiCorp Vault พื้นฐาน](#hashicorp-vault)
6. [AWS Secrets Manager](#aws-secrets-manager)
7. [Azure Key Vault](#azure-key-vault)
8. [Secret Rotation](#secret-rotation)
9. [OIDC และ Keyless Authentication](#oidc-keyless-auth)
10. [Secrets ใน Docker](#secrets-ใน-docker)
11. [การ Scan หา Leaked Secrets](#scanning-leaked-secrets)
12. [Best Practices](#best-practices)
13. [ห้าม Log Secrets เด็ดขาด](#ห้าม-log-secrets)
14. [แบบฝึกหัด (Exercises)](#exercises)

---

## บทนำ - ทำไม Secrets ต้องระวัง {#บทนำ}

### เหตุการณ์จริงที่เกิดขึ้นจาก Secret Exposure

```
กรณีศึกษา 1: Uber Data Breach (2022)
- นักพัฒนา commit AWS credentials ไป GitHub repo
- Bot scan GitHub ตลอด 24 ชั่วโมง
- ภายใน 30 วินาที credentials ถูก harvest
- Attacker เข้าถึง customer data ของ Uber หลายล้านคน

กรณีศึกษา 2: Toyota GitHub Leak (2022)
- Access key สำหรับ supplier portal ถูก expose ใน GitHub
- ข้อมูล customer ในญี่ปุ่น 296,019 ราย ถูก expose
- ข้อมูลรั่วไหลมานานกว่า 5 ปีก่อนถูกค้นพบ

กรณีศึกษา 3: Twitch Breach (2021)
- Source code ทั้งหมด + internal tools ถูก leak
- พบ hardcoded credentials ใน legacy code
- ต้องใช้เวลา rotate credentials ทั้งระบบ
```

### ต้นทุนของการ Leak Secrets

```
Direct Costs:
- Cloud bill พุ่งสูง (crypto mining บน AWS/GCP)
- ค่าใช้จ่ายในการ investigate และ remediate
- Legal fees (GDPR, PDPA fines)

Indirect Costs:
- ความเชื่อใจของ customer ลดลง
- Reputation damage
- Engineering time ที่เสียไป
- Regulatory penalties
- Class action lawsuits
```

### Secret ถูก Leak ได้อย่างไร

```
วิธีที่ Secret รั่วออกไปบ่อยที่สุด:

1. Committed to Git (40%)
   - Accidental commit ไฟล์ .env
   - Hardcoded ใน source code
   - ใน config files
   - ใน test fixtures

2. Exposed in Logs (25%)
   - Console.log(config) ที่มี secrets
   - Error messages ที่แสดง connection string
   - Debug mode ที่ log headers

3. Leaked via CI/CD (20%)
   - Print environment variables ใน logs
   - Artifact ที่มี secrets baked in
   - Misconfigured pipeline permissions

4. Insecure Storage (15%)
   - Plaintext ในไฟล์ config บน server
   - ใน unencrypted S3 buckets
   - ใน chat/email messages
```

---

## ประเภทของ Secrets {#ประเภทของ-secrets}

### 1. API Keys & Tokens

```python
# ตัวอย่าง API Keys ประเภทต่างๆ
secrets_examples = {
    # Payment APIs
    "stripe_secret_key": "<YOUR_STRIPE_SECRET_KEY>",    # format: sk_live_... หรือ sk_test_...
    "stripe_webhook_secret": "<YOUR_STRIPE_WEBHOOK_SECRET>",  # format: whsec_...
    
    # Cloud Providers
    "aws_access_key": "<YOUR_AWS_ACCESS_KEY_ID>",       # format: AKIA...
    "aws_secret_key": "<YOUR_AWS_SECRET_ACCESS_KEY>",  # 40-char secret
    "gcp_service_account": "{ json credentials }",
    
    # Communication
    "sendgrid_api_key": "<YOUR_SENDGRID_API_KEY>",  # format: SG.xxxxx
    "twilio_auth_token": "abcdefghijklmnopqrstuvwx",
    
    # Version Control
    "github_token": "<YOUR_GITHUB_PAT>",  # format: ghp_xxxxx (Personal Access Token)
    
    # Monitoring
    "datadog_api_key": "abcdefghijklmnopqrstuvwx",
    "sentry_dsn": "https://key@sentry.io/project",
}

# Risk levels
HIGH_RISK = ["payment", "cloud_provider", "database"]
MEDIUM_RISK = ["third_party_api", "monitoring"]
LOW_RISK = ["public_api_key", "rate_limited_api"]
```

### 2. Database Credentials

```bash
# ประเภทต่างๆ ของ Database Credentials

# Connection Strings (มีทุกอย่างรวมกัน - risk สูงมาก)
postgresql://username:password@hostname:5432/dbname
mysql://root:password@localhost:3306/mydb
mongodb://admin:password@mongo:27017/mydb?authSource=admin

# Individual credentials
DB_HOST=prod-db.example.com
DB_PORT=5432
DB_NAME=myapp_prod
DB_USERNAME=app_user
DB_PASSWORD=super-secret-password    # ← This is the secret

# Certificate-based authentication (more secure)
PGSSLCERT=/path/to/client-cert.pem
PGSSLKEY=/path/to/client-key.pem
PGSSLROOTCERT=/path/to/server-ca.pem
```

### 3. Certificates & Private Keys

```
ประเภท Certificate ที่ต้องจัดการ:

TLS/SSL Certificates:
- server.crt (public - สามารถ share ได้)
- server.key (private - SECRET, ห้าม commit)
- ca.crt (Certificate Authority - สามารถ share ได้)

Code Signing Certificates:
- app-signing.p12 (private keystore - SECRET)

SSH Keys:
- id_rsa / id_ed25519 (private - SECRET)
- id_rsa.pub / id_ed25519.pub (public - ok to share)

JWT Signing Keys:
- jwt-private.pem (SECRET)
- jwt-public.pem (ok to share)

Encryption Keys:
- encryption.key (SECRET)
- คีย์สำหรับ encrypt data at rest
```

### 4. OAuth & Service Account Credentials

```json
{
  "type": "service_account",
  "project_id": "my-project-123",
  "private_key_id": "key-id-here",
  "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----\n",
  "client_email": "myapp@my-project.iam.gserviceaccount.com",
  "client_id": "123456789",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token"
}
```

---

## GitHub Secrets {#github-secrets}

### ประเภทของ GitHub Secrets

```
1. Repository Secrets
   - เข้าถึงได้จาก workflows ใน repository นั้นเท่านั้น
   - เหมาะสำหรับ: API keys สำหรับ deployment

2. Environment Secrets
   - เข้าถึงได้เฉพาะเมื่อ workflow ใช้ environment นั้น
   - เหมาะสำหรับ: credentials สำหรับ specific environment

3. Organization Secrets
   - Share ระหว่าง repositories ใน organization
   - สามารถกำหนดว่า repositories ไหนเข้าถึงได้
   - เหมาะสำหรับ: shared infrastructure credentials
```

### การตั้งค่า GitHub Secrets ผ่าน CLI

```bash
# ติดตั้ง GitHub CLI
brew install gh   # macOS
# หรือ
sudo apt install gh  # Ubuntu

# Login
gh auth login

# เพิ่ม secret ไปยัง repository
gh secret set SECRET_NAME --body "secret-value"

# เพิ่มจากไฟล์
gh secret set PRIVATE_KEY < private-key.pem

# เพิ่ม environment secret
gh secret set DB_PASSWORD \
  --env production \
  --body "super-secret-password"

# List secrets (จะเห็นชื่อแต่ไม่เห็นค่า)
gh secret list

# ลบ secret
gh secret delete SECRET_NAME

# Organization secrets
gh secret set ORG_SHARED_KEY \
  --org myorganization \
  --body "shared-key-value" \
  --visibility selected \
  --repos "repo1,repo2,repo3"
```

### การใช้ Secrets ใน GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy Application

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    # ใช้ environment เพื่อเข้าถึง environment-specific secrets
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      # การใช้ secrets พื้นฐาน
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      # Secret ใน environment variable
      - name: Build with secrets
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.EXTERNAL_API_KEY }}
        run: |
          # ใน script นี้ $DATABASE_URL และ $API_KEY ใช้ได้
          npm run build
      
      # Secret ใน file (สำหรับ certificates, keys)
      - name: Setup SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.DEPLOY_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan deploy.example.com >> ~/.ssh/known_hosts
      
      # ❌ อย่าทำแบบนี้ - echo secret ออกมา
      # - run: echo "Secret is: ${{ secrets.MY_SECRET }}"
      
      # ✅ GitHub Actions จะ mask secrets โดยอัตโนมัติ
      # แต่ยังควรหลีกเลี่ยง expose ไปยัง stdout
      
      - name: Deploy
        run: |
          ./scripts/deploy.sh
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

### Secret Masking และข้อจำกัด

```yaml
# GitHub Actions masking ทำงานอย่างไร

# สมมติ secret = "abc123xyz"
# GitHub จะแทนที่ด้วย *** ใน logs โดยอัตโนมัติ

- name: Test masking
  env:
    MY_SECRET: ${{ secrets.MY_SECRET }}
  run: |
    echo "Secret: ${MY_SECRET}"        # Output: Secret: ***
    echo "$MY_SECRET" | base64         # ⚠️ อาจไม่ถูก mask (encoded form)
    echo "${MY_SECRET:0:3}"            # ⚠️ อาจไม่ถูก mask (partial)

# ข้อจำกัดของ masking:
# - ถ้า secret สั้นกว่า 3 ตัวอักษร จะไม่ถูก mask
# - encoded forms (base64, hex) อาจไม่ถูก mask
# - ถ้า secret มีค่าเหมือน common words อาจมีปัญหา
```

### Reusable Workflow สำหรับ Secret Handling

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string
    secrets:
      KUBECONFIG:
        required: true
      DB_PASSWORD:
        required: true
      JWT_SECRET:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > /tmp/kubeconfig
          chmod 600 /tmp/kubeconfig
        env:
          KUBECONFIG: /tmp/kubeconfig
      
      - name: Create/Update Kubernetes Secret
        run: |
          kubectl create secret generic myapp-secrets \
            --from-literal=db-password="${{ secrets.DB_PASSWORD }}" \
            --from-literal=jwt-secret="${{ secrets.JWT_SECRET }}" \
            --namespace myapp-${{ inputs.environment }} \
            --dry-run=client -o yaml | kubectl apply -f -
      
      - name: Deploy
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace myapp-${{ inputs.environment }} \
            --set image.tag=${{ inputs.image_tag }} \
            --wait
```

---

## GitLab Variables {#gitlab-variables}

### ประเภทของ GitLab CI/CD Variables

```
Variable Types:
1. Variable  - plain text (default)
2. File      - บันทึก value เป็นไฟล์, inject path เป็น variable

Scopes:
- All environments
- Specific environment (production, staging, etc.)
- Wildcards (review/*, staging/*)

Protection:
- Protected: ใช้ได้เฉพาะใน protected branches/tags
- Unprotected: ใช้ได้ในทุก branch

Masking:
- Masked: ค่าจะถูกซ่อนใน job logs
- Unmasked: ค่าจะแสดงใน logs (ใช้สำหรับ non-sensitive config)
```

### GitLab CI/CD Configuration

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  # Non-sensitive config (ok to define here)
  APP_PORT: "3000"
  NODE_ENV: "production"
  # SECRET vars ต้องกำหนดใน GitLab UI/API ไม่ใช่ที่นี่

test:
  stage: test
  image: node:20
  script:
    - npm ci
    - npm test
  variables:
    # Override สำหรับ test environment
    NODE_ENV: test
    # DATABASE_URL จะมาจาก GitLab Variable ที่ตั้งไว้
    DATABASE_URL: $DATABASE_URL_TEST

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
    - docker build -t "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA" .
    - docker push "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"
  only:
    - main

deploy-staging:
  stage: deploy
  image: alpine/helm:3.12
  script:
    # SECRET_KEY, DB_PASSWORD มาจาก GitLab Variables (masked + protected)
    - helm upgrade --install myapp ./charts/myapp
        --set image.tag=$CI_COMMIT_SHA
        --set secrets.dbPassword="$STAGING_DB_PASSWORD"
        --set secrets.secretKey="$STAGING_SECRET_KEY"
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main

deploy-production:
  stage: deploy
  image: alpine/helm:3.12
  script:
    - helm upgrade --install myapp ./charts/myapp
        --set image.tag=$CI_COMMIT_SHA
        --set secrets.dbPassword="$PROD_DB_PASSWORD"
        --set secrets.secretKey="$PROD_SECRET_KEY"
  environment:
    name: production
    url: https://example.com
  when: manual                           # ต้อง trigger manual
  only:
    - main
  # เฉพาะ protected branch เท่านั้น (ตั้งค่า protected variables)
```

### GitLab Variable API

```bash
# ตั้งค่า variable ผ่าน API
curl --request POST \
  --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --form "key=PRODUCTION_DB_PASSWORD" \
  --form "value=super-secret-password" \
  --form "protected=true" \
  --form "masked=true" \
  --form "environment_scope=production" \
  "https://gitlab.com/api/v4/projects/${PROJECT_ID}/variables"

# List variables
curl --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  "https://gitlab.com/api/v4/projects/${PROJECT_ID}/variables"

# Group-level variables (shared ทุก projects ในกลุ่ม)
curl --request POST \
  --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --form "key=SHARED_API_KEY" \
  --form "value=shared-key-value" \
  --form "masked=true" \
  "https://gitlab.com/api/v4/groups/${GROUP_ID}/variables"
```

---

## HashiCorp Vault พื้นฐาน {#hashicorp-vault}

### Vault คืออะไร และทำไมต้องใช้

```
HashiCorp Vault เป็น secrets management platform ที่ให้:

1. Centralized secret storage (เก็บ secrets ทุกอย่างที่เดียว)
2. Dynamic secrets (สร้าง credentials ใหม่ทุกครั้งที่ request)
3. Secret leasing (credentials หมดอายุอัตโนมัติ)
4. Audit logging (รู้ว่าใครเข้าถึง secret อะไร เมื่อไหร่)
5. Access control (fine-grained policies)
6. Encryption as a Service
```

### ติดตั้ง Vault สำหรับ Development

```bash
# Docker Compose สำหรับ Vault dev setup
cat > docker-compose.vault.yml << 'EOF'
version: '3.8'

services:
  vault:
    image: hashicorp/vault:1.15
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "dev-root-token"
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    cap_add:
      - IPC_LOCK
    command: server -dev
    
  vault-init:
    image: hashicorp/vault:1.15
    depends_on:
      - vault
    environment:
      VAULT_ADDR: "http://vault:8200"
      VAULT_TOKEN: "dev-root-token"
    command: |
      sh -c "
        sleep 5
        # Enable KV secrets engine
        vault secrets enable -path=secret kv-v2
        
        # Store some example secrets
        vault kv put secret/myapp/production \
          db_password='super-secret-prod-password' \
          jwt_secret='production-jwt-secret-min-32-chars!!'
          
        vault kv put secret/myapp/staging \
          db_password='staging-password' \
          jwt_secret='staging-jwt-secret-min-32-chars!!!'
          
        echo 'Vault initialized with example secrets'
      "

EOF

docker compose -f docker-compose.vault.yml up -d
```

### Vault Basic Operations

```bash
# ตั้งค่า Vault CLI
export VAULT_ADDR="https://vault.example.com"
export VAULT_TOKEN="your-vault-token"

# หรือ login ด้วย method ต่างๆ
vault login -method=github token="your-github-token"
vault login -method=ldap username="myuser"
vault login -method=aws

# KV Secrets Engine (Version 2)
# เขียน secret
vault kv put secret/myapp/production \
  db_password="super-secret-password" \
  api_key="my-api-key"

# อ่าน secret
vault kv get secret/myapp/production

# อ่านเฉพาะ field
vault kv get -field=db_password secret/myapp/production

# อ่านเป็น JSON
vault kv get -format=json secret/myapp/production | jq '.data.data'

# ดู versions
vault kv metadata get secret/myapp/production

# Rollback ไปยัง version ก่อนหน้า
vault kv rollback -version=1 secret/myapp/production

# ลบ (soft delete)
vault kv delete secret/myapp/production

# ลบถาวร
vault kv destroy -versions=1,2 secret/myapp/production
```

### Dynamic Database Credentials

```bash
# เปิดใช้งาน Database Secrets Engine
vault secrets enable database

# Configure database connection
vault write database/config/myapp-postgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/myapp?sslmode=disable" \
  username="vault" \
  password="vault-admin-password"

# Create role สำหรับ dynamic credentials
vault write database/roles/myapp-role \
  db_name=myapp-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Request dynamic credentials (ทุกครั้งจะได้ username/password ใหม่)
vault read database/creds/myapp-role

# Output:
# Key                Value
# ---                -----
# lease_id           database/creds/myapp-role/abc123
# lease_duration     1h
# lease_renewable    true
# password           A1a-xyz987
# username           v-root-myapp-role-abc123
```

### Vault Policies

```hcl
# policies/myapp-production.hcl
# อนุญาต read เฉพาะ production secrets
path "secret/data/myapp/production" {
  capabilities = ["read"]
}

# ห้าม delete
path "secret/metadata/myapp/production" {
  capabilities = ["read", "list"]
}

# Dynamic database credentials
path "database/creds/myapp-prod-role" {
  capabilities = ["read"]
}

# ห้ามเข้าถึง admin secrets
path "secret/data/admin/*" {
  capabilities = ["deny"]
}
```

```bash
# Upload policy
vault policy write myapp-production policies/myapp-production.hcl

# Test policy
vault token create \
  -policy=myapp-production \
  -period=1h \
  -display-name="myapp-service"
```

### Integration กับ GitHub Actions

```yaml
# .github/workflows/deploy-with-vault.yml
name: Deploy with Vault Secrets

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    # ต้องตั้งค่า VAULT_ADDR และ Vault Auth สำหรับ GitHub
    permissions:
      id-token: write   # สำหรับ OIDC authentication
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get Vault secrets
        uses: hashicorp/vault-action@v2
        id: vault
        with:
          url: ${{ secrets.VAULT_ADDR }}
          # ใช้ OIDC auth แทน long-lived token
          method: jwt
          role: github-actions-production
          jwtGithubAudience: https://vault.example.com
          secrets: |
            secret/data/myapp/production db_password | DB_PASSWORD ;
            secret/data/myapp/production jwt_secret | JWT_SECRET ;
            secret/data/myapp/production api_key | API_KEY
      
      - name: Deploy application
        env:
          DB_PASSWORD: ${{ steps.vault.outputs.DB_PASSWORD }}
          JWT_SECRET: ${{ steps.vault.outputs.JWT_SECRET }}
          API_KEY: ${{ steps.vault.outputs.API_KEY }}
        run: |
          ./scripts/deploy.sh
```

### Kubernetes + Vault Integration (Vault Agent Injector)

```yaml
# kubernetes/deployment-with-vault.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp-prod
spec:
  template:
    metadata:
      annotations:
        # Vault Agent Injector annotations
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-production"
        vault.hashicorp.com/agent-inject-secret-config.env: "secret/data/myapp/production"
        vault.hashicorp.com/agent-inject-template-config.env: |
          {{- with secret "secret/data/myapp/production" -}}
          export DB_PASSWORD="{{ .Data.data.db_password }}"
          export JWT_SECRET="{{ .Data.data.jwt_secret }}"
          {{- end -}}
    spec:
      serviceAccountName: myapp-sa    # ต้องมี Vault policy ที่ allow
      containers:
        - name: myapp
          image: myapp:latest
          command:
            - sh
            - -c
            - "source /vault/secrets/config.env && node server.js"
```

---

## AWS Secrets Manager {#aws-secrets-manager}

### การตั้งค่าและใช้งาน

```python
# Python: อ่าน secret จาก AWS Secrets Manager
import boto3
import json
from botocore.exceptions import ClientError

def get_secret(secret_name: str, region_name: str = "ap-southeast-1") -> dict:
    """
    ดึง secret จาก AWS Secrets Manager
    Returns: dict with secret values
    """
    client = boto3.client(
        service_name='secretsmanager',
        region_name=region_name
    )
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
    except ClientError as e:
        error_code = e.response['Error']['Code']
        
        if error_code == 'DecryptionFailureException':
            raise Exception("Cannot decrypt secret - check KMS permissions")
        elif error_code == 'InternalServiceErrorException':
            raise Exception("AWS internal service error")
        elif error_code == 'InvalidParameterException':
            raise Exception("Invalid parameter")
        elif error_code == 'InvalidRequestException':
            raise Exception("Invalid request")
        elif error_code == 'ResourceNotFoundException':
            raise Exception(f"Secret '{secret_name}' not found")
        else:
            raise
    
    # Parse secret value
    if 'SecretString' in response:
        return json.loads(response['SecretString'])
    else:
        # Binary secret
        import base64
        return {"binary": base64.b64decode(response['SecretBinary'])}


# การใช้งานใน application startup
def load_config():
    # Load from AWS Secrets Manager
    db_secrets = get_secret("myapp/production/database")
    api_secrets = get_secret("myapp/production/api-keys")
    
    return {
        "DATABASE_URL": f"postgresql://{db_secrets['username']}:{db_secrets['password']}@{db_secrets['host']}/{db_secrets['dbname']}",
        "STRIPE_SECRET_KEY": api_secrets["stripe_secret_key"],
        "SENDGRID_API_KEY": api_secrets["sendgrid_api_key"],
    }


# Caching สำหรับ performance
from functools import lru_cache
import time

class SecretsCache:
    def __init__(self, ttl_seconds: int = 300):
        self._cache = {}
        self._timestamps = {}
        self._ttl = ttl_seconds
    
    def get(self, secret_name: str) -> dict:
        now = time.time()
        
        if (secret_name in self._cache and 
            now - self._timestamps[secret_name] < self._ttl):
            return self._cache[secret_name]
        
        # Cache miss - fetch from AWS
        secret = get_secret(secret_name)
        self._cache[secret_name] = secret
        self._timestamps[secret_name] = now
        
        return secret
    
    def invalidate(self, secret_name: str = None):
        if secret_name:
            self._cache.pop(secret_name, None)
            self._timestamps.pop(secret_name, None)
        else:
            self._cache.clear()
            self._timestamps.clear()

# Global cache instance
secrets_cache = SecretsCache(ttl_seconds=300)  # 5 minutes cache
```

### Terraform สำหรับ AWS Secrets Manager

```hcl
# terraform/secrets.tf

# สร้าง secret
resource "aws_secretsmanager_secret" "database" {
  name                    = "myapp/production/database"
  description             = "Database credentials for production"
  recovery_window_in_days = 7    # 7 days ก่อนลบจริง
  
  tags = {
    Environment = "production"
    Application = "myapp"
    ManagedBy   = "terraform"
  }
}

# ตั้งค่า secret value (ไม่ควร hardcode - ใช้ variables หรือ data source)
resource "aws_secretsmanager_secret_version" "database" {
  secret_id = aws_secretsmanager_secret.database.id
  
  secret_string = jsonencode({
    username = var.db_username
    password = var.db_password     # ← มาจาก Terraform variable (sensitive)
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    dbname   = aws_db_instance.main.db_name
  })
}

# IAM Policy สำหรับ application
resource "aws_iam_policy" "secrets_read" {
  name        = "myapp-production-secrets-read"
  description = "Allow reading myapp production secrets"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ]
        Resource = [
          aws_secretsmanager_secret.database.arn,
          "arn:aws:secretsmanager:${var.region}:${var.account_id}:secret:myapp/production/*"
        ]
      },
      {
        # Allow decryption with KMS
        Effect = "Allow"
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey"
        ]
        Resource = aws_kms_key.secrets.arn
      }
    ]
  })
}

# Rotation Lambda
resource "aws_secretsmanager_secret_rotation" "database" {
  secret_id           = aws_secretsmanager_secret.database.id
  rotation_lambda_arn = aws_lambda_function.rotate_secret.arn
  
  rotation_rules {
    automatically_after_days = 30    # Rotate ทุก 30 วัน
  }
}
```

---

## Azure Key Vault {#azure-key-vault}

### การใช้งาน Azure Key Vault

```python
# Python: Azure Key Vault integration
from azure.keyvault.secrets import SecretClient
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential

def get_azure_client(vault_url: str) -> SecretClient:
    """
    สร้าง Key Vault client
    ใช้ DefaultAzureCredential ซึ่งรองรับหลาย auth methods:
    - Managed Identity (บน Azure resources)
    - Service Principal
    - Azure CLI
    - Visual Studio
    """
    credential = DefaultAzureCredential()
    return SecretClient(vault_url=vault_url, credential=credential)


def get_secret(secret_name: str, vault_url: str = None) -> str:
    """ดึง secret value จาก Azure Key Vault"""
    vault_url = vault_url or os.environ["AZURE_KEYVAULT_URL"]
    client = get_azure_client(vault_url)
    
    secret = client.get_secret(secret_name)
    return secret.value


# ใช้งานใน application
import os

VAULT_URL = os.environ["AZURE_KEYVAULT_URL"]  # เช่น: https://myapp.vault.azure.net/

# ดึง secrets
db_password = get_secret("database-password")
api_key = get_secret("external-api-key")
jwt_secret = get_secret("jwt-signing-secret")
```

### Azure Key Vault ใน GitHub Actions

```yaml
# .github/workflows/deploy-azure.yml
name: Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    permissions:
      id-token: write    # สำหรับ OIDC
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      # Login ไปยัง Azure ด้วย OIDC (keyless!)
      - name: Login to Azure
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      # ดึง secrets จาก Key Vault
      - name: Get secrets from Key Vault
        uses: azure/get-keyvault-secrets@v1
        with:
          keyvault: "myapp-prod-kv"
          secrets: |
            database-password,
            jwt-signing-secret,
            external-api-key
        id: keyvault
      
      # ใช้ secrets ใน deployment
      - name: Deploy
        env:
          DB_PASSWORD: ${{ steps.keyvault.outputs.database-password }}
          JWT_SECRET: ${{ steps.keyvault.outputs.jwt-signing-secret }}
          API_KEY: ${{ steps.keyvault.outputs.external-api-key }}
        run: |
          ./scripts/deploy-azure.sh
```

---

## Secret Rotation {#secret-rotation}

### ทำไมต้อง Rotate Secrets

```
เหตุผลที่ต้อง Rotate:
1. ลดความเสียหายถ้า secret ถูก compromise
2. Compliance requirements (PCI-DSS, SOC2, ISO27001)
3. Employee turnover - revoke access เมื่อคนออก
4. Scheduled rotation ตาม security policy
5. Emergency rotation เมื่อสงสัย compromise

Rotation Frequency:
- High-value secrets (payment, admin): ทุก 30 วัน หรือน้อยกว่า
- Regular API keys: ทุก 90 วัน
- Certificates: ก่อน expiry (ควร automate)
- Database passwords: ทุก 90 วัน
- SSH keys: ทุก 1 ปี หรือเมื่อ staff เปลี่ยน
```

### Automated Secret Rotation Script

```python
#!/usr/bin/env python3
"""
Secret Rotation Automation
สำหรับ rotate database passwords โดยอัตโนมัติ
"""

import boto3
import json
import secrets
import string
import logging
from datetime import datetime

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def generate_password(length: int = 32) -> str:
    """สร้าง secure password"""
    alphabet = string.ascii_letters + string.digits + "!@#$%^&*()"
    
    # ต้องมีอย่างน้อย 1 ของแต่ละประเภท
    password = [
        secrets.choice(string.ascii_lowercase),
        secrets.choice(string.ascii_uppercase),
        secrets.choice(string.digits),
        secrets.choice("!@#$%^"),
    ]
    
    # เติมที่เหลือ
    password += [secrets.choice(alphabet) for _ in range(length - 4)]
    
    # Shuffle เพื่อไม่ให้ predictable
    secrets.SystemRandom().shuffle(password)
    
    return ''.join(password)


def lambda_handler(event, context):
    """
    AWS Lambda handler สำหรับ secret rotation
    
    Steps:
    1. createSecret - สร้าง new credentials
    2. setSecret - ตั้งค่าใน database
    3. testSecret - ทดสอบว่า credentials ใหม่ใช้งานได้
    4. finishSecret - mark version ใหม่เป็น AWSCURRENT
    """
    
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']
    
    client = boto3.client('secretsmanager')
    
    # Get current secret
    try:
        current_secret = json.loads(
            client.get_secret_value(
                SecretId=arn,
                VersionStage='AWSCURRENT'
            )['SecretString']
        )
    except Exception as e:
        logger.error(f"Error getting current secret: {e}")
        raise
    
    if step == "createSecret":
        logger.info(f"Creating new version for {arn}")
        
        # สร้าง new password
        new_password = generate_password()
        
        # Prepare new secret value
        new_secret = current_secret.copy()
        new_secret['password'] = new_password
        new_secret['lastRotated'] = datetime.utcnow().isoformat()
        
        # Store new version
        client.put_secret_value(
            SecretId=arn,
            ClientRequestToken=token,
            SecretString=json.dumps(new_secret),
            VersionStages=['AWSPENDING']
        )
        
    elif step == "setSecret":
        logger.info(f"Setting new credentials in database for {arn}")
        
        # Get pending version
        pending_secret = json.loads(
            client.get_secret_value(
                SecretId=arn,
                VersionStage='AWSPENDING'
            )['SecretString']
        )
        
        # Update database password
        import psycopg2
        
        conn = psycopg2.connect(
            host=current_secret['host'],
            port=current_secret['port'],
            dbname='postgres',
            user=current_secret['username'],
            password=current_secret['password']
        )
        
        try:
            conn.autocommit = True
            with conn.cursor() as cur:
                cur.execute(
                    "ALTER USER %s WITH PASSWORD %s",
                    (pending_secret['username'], pending_secret['password'])
                )
            logger.info("Database password updated successfully")
        finally:
            conn.close()
    
    elif step == "testSecret":
        logger.info(f"Testing new credentials for {arn}")
        
        pending_secret = json.loads(
            client.get_secret_value(
                SecretId=arn,
                VersionStage='AWSPENDING'
            )['SecretString']
        )
        
        # Test connection with new credentials
        import psycopg2
        
        try:
            conn = psycopg2.connect(
                host=pending_secret['host'],
                port=pending_secret['port'],
                dbname=pending_secret['dbname'],
                user=pending_secret['username'],
                password=pending_secret['password']
            )
            conn.close()
            logger.info("New credentials test successful")
        except Exception as e:
            logger.error(f"New credentials test failed: {e}")
            raise
    
    elif step == "finishSecret":
        logger.info(f"Finalizing rotation for {arn}")
        
        # Get current version ID
        metadata = client.describe_secret(SecretId=arn)
        current_version = None
        for version, stages in metadata['VersionIdsToStages'].items():
            if 'AWSCURRENT' in stages:
                if version != token:
                    current_version = version
                break
        
        # Mark new version as current
        client.update_secret_version_stage(
            SecretId=arn,
            VersionStage='AWSCURRENT',
            MoveToVersionId=token,
            RemoveFromVersionId=current_version
        )
        
        logger.info(f"Rotation completed for {arn}")
    
    return {'status': 'success', 'step': step}
```

---

## OIDC และ Keyless Authentication {#oidc-keyless-auth}

### ทำไม OIDC ถึงดีกว่า Long-lived Tokens

```
Traditional Way (ไม่แนะนำ):
- เก็บ AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY ใน GitHub Secrets
- Keys ไม่หมดอายุ
- ถ้าถูก compromise → ต้อง revoke และ rotate ทันที
- ต้อง manage keys ทุก repository/pipeline

OIDC Way (แนะนำ):
- GitHub Actions ขอ short-lived token จาก AWS/GCP/Azure
- Token อายุแค่ 1 ชั่วโมง (หมดอายุเอง)
- ไม่ต้องเก็บ long-lived credentials
- Trust based on identity (which repo, which branch, which workflow)
```

### GitHub Actions + AWS OIDC

```yaml
# .github/workflows/deploy-oidc.yml
name: Deploy with OIDC

on:
  push:
    branches: [main]

permissions:
  id-token: write    # จำเป็นสำหรับ OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          role-session-name: github-actions-deploy
          aws-region: ap-southeast-1
          # ไม่ต้องใส่ access-key-id / secret-access-key เลย!
      
      - name: Deploy to ECS
        run: |
          # ใช้ AWS CLI ได้เลย - credentials ถูกตั้งค่าอัตโนมัติ
          aws ecs update-service \
            --cluster production \
            --service myapp \
            --force-new-deployment
```

### ตั้งค่า AWS IAM Role สำหรับ GitHub OIDC

```hcl
# terraform/github-oidc.tf

# สร้าง OIDC Provider
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"
  
  client_id_list = ["sts.amazonaws.com"]
  
  thumbprint_list = [
    "6938fd4d98bab03faadb97b34396831e3780aea1",
    "1c58a3a8518e8759bf075b76b750d4f2df264fcd"
  ]
}

# IAM Role ที่ GitHub Actions จะ assume
resource "aws_iam_role" "github_actions" {
  name = "GitHubActionsRole"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            # เฉพาะ repo นี้เท่านั้น และ branch main
            "token.actions.githubusercontent.com:sub" = [
              "repo:myorg/myrepo:ref:refs/heads/main",
              "repo:myorg/myrepo:environment:production"
            ]
          }
        }
      }
    ]
  })
}

# Policy สำหรับ deployment
resource "aws_iam_role_policy" "github_actions_deploy" {
  name = "DeployPolicy"
  role = aws_iam_role.github_actions.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ecs:UpdateService",
          "ecs:DescribeServices",
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:PutImage"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### GCP OIDC Workload Identity Federation

```yaml
# GitHub Actions + GCP OIDC
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: 'projects/123/locations/global/workloadIdentityPools/github/providers/github-actions'
    service_account: 'github-actions@myproject.iam.gserviceaccount.com'
    # ไม่ต้องใส่ credentials JSON เลย!

- name: Deploy to Cloud Run
  uses: google-github-actions/deploy-cloudrun@v2
  with:
    service: myapp
    image: asia-southeast1-docker.pkg.dev/myproject/myapp/api:${{ github.sha }}
    region: asia-southeast1
```

---

## Secrets ใน Docker {#secrets-ใน-docker}

### Docker Build Secrets (BuildKit)

```dockerfile
# Dockerfile
# ❌ อย่าทำแบบนี้ - secret จะอยู่ใน image layer ตลอดไป
FROM node:20
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > ~/.npmrc
RUN npm install
RUN rm ~/.npmrc  # ลบแล้วก็ยังอยู่ใน layer ก่อนหน้า!
```

```dockerfile
# ✅ ใช้ BuildKit secrets แทน
# syntax=docker/dockerfile:1
FROM node:20

# Mount secret เฉพาะตอน build ไม่ถูก baked in image
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install

# ตอน production stage ไม่มี secret
FROM node:20-alpine AS production
COPY --from=build /app/node_modules ./node_modules
```

```bash
# Build ด้วย BuildKit secrets
docker buildx build \
  --secret id=npmrc,src=.npmrc \
  --tag myapp:latest \
  .

# ใน GitHub Actions
- name: Build Docker image
  uses: docker/build-push-action@v5
  with:
    context: .
    secrets: |
      npmrc=${{ secrets.NPM_TOKEN }}
    tags: myapp:latest
```

### Docker Runtime Secrets (Swarm)

```yaml
# docker-compose.yml ที่ใช้ Docker Secrets
version: '3.8'

services:
  app:
    image: myapp:latest
    secrets:
      - db_password
      - jwt_secret
    environment:
      # Secret files จะอยู่ที่ /run/secrets/
      DB_PASSWORD_FILE: /run/secrets/db_password
      JWT_SECRET_FILE: /run/secrets/jwt_secret

secrets:
  db_password:
    external: true          # สร้างด้วย docker secret create
  jwt_secret:
    external: true
```

```bash
# สร้าง Docker secrets
echo "super-secret-password" | docker secret create db_password -
echo "jwt-secret-min-32-chars-here" | docker secret create jwt_secret -

# List secrets
docker secret ls

# Application ต้องอ่านจาก file
# Node.js example:
```

```javascript
// config/secrets.js
const fs = require('fs');

function readSecretFile(path) {
  try {
    return fs.readFileSync(path, 'utf8').trim();
  } catch (err) {
    return null;
  }
}

// อ่าน secret จาก file หรือ environment variable
const dbPassword = 
  readSecretFile(process.env.DB_PASSWORD_FILE) || 
  process.env.DB_PASSWORD;

if (!dbPassword) {
  throw new Error('DB_PASSWORD not configured (set DB_PASSWORD or DB_PASSWORD_FILE)');
}
```

### Kubernetes Secrets Best Practices

```yaml
# ❌ ไม่ดี - Secret ไม่ถูก encrypt at rest
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
data:
  # base64 ไม่ใช่ encryption! ทุกคนที่มีสิทธิ์เข้าถึง etcd อ่านได้
  db_password: c3VwZXItc2VjcmV0LXBhc3N3b3Jk

---
# ✅ ดีกว่า - ใช้ External Secrets Operator กับ Vault/AWS
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
spec:
  refreshInterval: 15m
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: myapp-secrets    # ชื่อ Kubernetes Secret ที่จะสร้าง
  data:
    - secretKey: db_password
      remoteRef:
        key: myapp/production
        property: db_password
    - secretKey: jwt_secret
      remoteRef:
        key: myapp/production
        property: jwt_secret
```

---

## การ Scan หา Leaked Secrets {#scanning-leaked-secrets}

### 1. git-secrets

```bash
# ติดตั้ง git-secrets
brew install git-secrets    # macOS
# หรือ
git clone https://github.com/awslabs/git-secrets.git
cd git-secrets && make install

# ตั้งค่า patterns
git secrets --install            # ติดตั้ง hooks ใน current repo
git secrets --register-aws       # เพิ่ม AWS patterns
git secrets --add 'password\s*=\s*.+'  # Custom pattern

# Scan existing repo
git secrets --scan               # scan staged files
git secrets --scan-history       # scan entire git history

# ตัวอย่าง output เมื่อพบ secret
# .env:3:AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# [ERROR] Matched one or more prohibited patterns
```

### 2. detect-secrets

```bash
# ติดตั้ง
pip install detect-secrets

# สร้าง baseline (whitelist known safe occurrences)
detect-secrets scan > .secrets.baseline

# Scan สำหรับ secrets ใหม่
detect-secrets scan --baseline .secrets.baseline

# Audit baseline (review ว่า occurrences เหล่านั้น safe จริงไหม)
detect-secrets audit .secrets.baseline

# Pre-commit hook
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
        exclude: package.lock.json
EOF

pre-commit install
```

### 3. Gitleaks (แนะนำมากที่สุด)

```bash
# ติดตั้ง
brew install gitleaks    # macOS
# หรือ download binary จาก GitHub releases

# Scan current repo
gitleaks detect --source . --verbose

# Scan git history ทั้งหมด
gitleaks detect --source . --log-opts="--all" --verbose

# Scan เฉพาะ staged files
gitleaks protect --staged --verbose

# Output format
gitleaks detect --source . --report-format json --report-path report.json
```

```yaml
# .gitleaks.toml - Custom configuration
[extend]
useDefault = true    # ใช้ default rules ของ gitleaks

# เพิ่ม custom rules
[[rules]]
id = "thai-api-key"
description = "Generic API key pattern"
regex = '''(?i)(api[_-]?key|apikey)\s*[=:]\s*['"]?([A-Za-z0-9_\-]{20,})['"]?'''
entropy = 3.5
secretGroup = 2

[[rules]]
id = "jwt-secret"
description = "JWT secret key"
regex = '''(?i)(jwt[_-]?secret|signing[_-]?key)\s*[=:]\s*['"]?([A-Za-z0-9_\-!@#$%]{32,})['"]?'''
secretGroup = 2

# Allowlist (false positives ที่รู้แล้วว่า safe)
[[allowlist]]
description = "Test fixtures"
paths = ['''(^|/)test/fixtures/''', '''\.test\.js$''']
commits = ["abc123def456"]    # Specific commits to ignore
regexes = ['''EXAMPLE_KEY''', '''test_password''']
```

### GitHub Actions: Secret Scanning ใน CI

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  secret-scan:
    name: Scan for Secrets
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # ต้องการ full history สำหรับ gitleaks
      
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}   # ถ้าใช้ commercial
      
      - name: Run Trivy secret scanning
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          scanners: 'secret'
          severity: 'HIGH,CRITICAL'
          exit-code: '1'
      
      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  dependency-check:
    name: Check Dependencies for Vulnerabilities
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run npm audit
        run: npm audit --audit-level=high
      
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          command: test
          args: --severity-threshold=high
```

### Pre-commit Hooks สำหรับ Secret Detection

```yaml
# .pre-commit-config.yaml
repos:
  # Gitleaks สำหรับ secret detection
  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
  
  # detect-secrets สำหรับ additional patterns
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
  
  # Prevent committing .env files
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: check-added-large-files
      - id: check-merge-conflict
      - id: detect-private-key
      - id: no-commit-to-branch
        args: ['--branch', 'main', '--branch', 'master']
  
  # Custom hook สำหรับ check .env files
  - repo: local
    hooks:
      - id: check-env-files
        name: Check for .env files
        language: script
        entry: scripts/check-env-files.sh
        types: [file]
```

```bash
#!/bin/bash
# scripts/check-env-files.sh
# ตรวจสอบว่าไม่มี .env files ถูก stage

for file in "$@"; do
  if [[ "${file}" =~ \.env($|\.) ]]; then
    echo "ERROR: Attempting to commit .env file: ${file}"
    echo "       Remove it from staging: git reset HEAD ${file}"
    echo "       Add to .gitignore if needed"
    exit 1
  fi
done

exit 0
```

---

## Best Practices {#best-practices}

### Secret Hygiene Checklist

```
✅ DO (ควรทำ):
□ ใช้ secrets manager (Vault, AWS SM, Azure KV)
□ Rotate secrets เป็น routine
□ ใช้ OIDC/Workload Identity แทน long-lived credentials
□ Audit access logs สม่ำเสมอ
□ Principle of least privilege - ให้สิทธิ์น้อยที่สุดที่จำเป็น
□ มี separate secrets per environment
□ ใช้ pre-commit hooks สำหรับ detect secrets
□ Scan ด้วย gitleaks ใน CI pipeline
□ Encrypt secrets at rest
□ Set secret expiration dates

❌ DON'T (ห้ามทำ):
□ ห้าม hardcode secrets ใน code
□ ห้าม commit .env files
□ ห้าม log secrets (แม้ใน debug)
□ ห้าม share secrets ผ่าน chat/email
□ ห้าม reuse secrets ระหว่าง environments
□ ห้าม store secrets ใน environment variables ที่ไม่ secure
□ ห้าม ignore secret scanning alerts
□ ห้าม use weak/short secrets
□ ห้าม grant excessive permissions
□ ห้าม forget to rotate after employee departure
```

### Secrets Audit Script

```bash
#!/bin/bash
# scripts/audit-secrets.sh
# ตรวจสอบ health ของ secrets management

echo "=== Secrets Audit Report ==="
echo "Date: $(date)"
echo ""

# 1. ตรวจสอบ .env files ที่อาจ commit แล้ว
echo "--- Checking for committed .env files ---"
git log --all --full-history -- "**/.env" "**/.env.*" 2>/dev/null | \
  grep "commit " | \
  head -20

# 2. ตรวจสอบ patterns ที่ดูเหมือน hardcoded secrets
echo ""
echo "--- Checking for potential hardcoded secrets ---"
grep -rn \
  -e "password\s*=\s*['\"][^'\"]\{8,\}" \
  -e "secret\s*=\s*['\"][^'\"]\{8,\}" \
  -e "api_key\s*=\s*['\"][^'\"]\{8,\}" \
  --include="*.py" \
  --include="*.js" \
  --include="*.ts" \
  --include="*.go" \
  --exclude-dir=".git" \
  --exclude-dir="node_modules" \
  --exclude-dir="vendor" \
  . 2>/dev/null | \
  grep -v "_test\." | \
  grep -v "example\|placeholder\|changeme\|your-" | \
  head -20

# 3. ตรวจสอบ .gitignore มี .env หรือไม่
echo ""
echo "--- Checking .gitignore ---"
if grep -q "\.env" .gitignore 2>/dev/null; then
  echo "✅ .gitignore contains .env patterns"
else
  echo "⚠️  WARNING: .gitignore may not exclude .env files"
fi

echo ""
echo "=== Audit Complete ==="
```

### Zero-Trust Secrets Architecture

```yaml
# หลักการ Zero-Trust สำหรับ Secrets:
#
# 1. Never Trust, Always Verify
#    - Authenticate ทุก request สำหรับ secrets
#    - ไม่มี "trusted network" ที่ skip authentication
#
# 2. Least Privilege Access
#    - Application อ่านได้เฉพาะ secrets ที่ต้องการ
#    - ไม่มี wildcard permissions
#
# 3. Assume Breach
#    - Rotate secrets บ่อยๆ
#    - Log ทุก access
#    - Alert บน unusual access patterns
#
# 4. Verify Explicitly
#    - ใช้ mutual TLS ที่เป็นไปได้
#    - Verify identity ของ requestor

# Example: Vault Policy ตาม Zero-Trust
path "secret/data/myapp/production/database" {
  capabilities = ["read"]
  
  # Allow only during business hours
  required_parameters = ["timestamp"]
  
  # Additional conditions
  control_group {
    factor "authorizer" {
      identity {
        group_names = ["database-admins"]
        approvals   = 1
      }
    }
  }
}
```

---

## ห้าม Log Secrets เด็ดขาด {#ห้าม-log-secrets}

### ตัวอย่างการ Log ที่อันตราย

```python
# ❌ อันตรายมาก - logging secrets
import logging

logger = logging.getLogger(__name__)

def connect_to_database(config):
    # DON'T DO THIS!
    logger.debug(f"Connecting with config: {config}")    # Config มี password!
    logger.info(f"DB URL: {config['DATABASE_URL']}")     # URL มี credentials!
    
    try:
        db = Database(config['DATABASE_URL'])
    except Exception as e:
        # DON'T DO THIS!
        logger.error(f"Failed to connect: {e}")          # Error อาจมี credentials!
        raise

def make_api_request(api_key: str, data: dict):
    # DON'T DO THIS!
    logger.debug(f"API key: {api_key}")                  # Logging API key!
    logger.debug(f"Headers: {{'Authorization': f'Bearer {api_key}'}}")
```

```python
# ✅ วิธีที่ถูกต้อง
import logging
import re

logger = logging.getLogger(__name__)

def mask_sensitive(text: str) -> str:
    """Mask sensitive values in text"""
    patterns = [
        (r'password["\s]*[:=]["\s]*\S+', 'password=***'),
        (r'api_key["\s]*[:=]["\s]*\S+', 'api_key=***'),
        (r'sk_[a-z]+_[A-Za-z0-9]+', 'sk_***'),
        (r'[A-Z0-9]{20}', '***AWS_KEY***'),             # AWS access keys
        (r'postgresql://[^@]+@', 'postgresql://***:***@'),
    ]
    
    result = text
    for pattern, replacement in patterns:
        result = re.sub(pattern, replacement, result, flags=re.IGNORECASE)
    
    return result

def connect_to_database(config: dict):
    # ✅ Log without sensitive data
    logger.info(f"Connecting to database at {config.get('DB_HOST')}:{config.get('DB_PORT')}")
    
    try:
        db = Database(config['DATABASE_URL'])
        logger.info("Database connection established")
        return db
    except Exception as e:
        # ✅ Log error message only, not the full error that might contain credentials
        logger.error(f"Failed to connect to database: {type(e).__name__}")
        # ❌ อย่าทำ: logger.error(f"Error: {e}")  # อาจมี URL + credentials ใน error
        raise

class SensitiveDataFilter(logging.Filter):
    """Filter ที่ mask sensitive data ใน log records"""
    
    PATTERNS = [
        re.compile(r'(?i)(password|passwd|pwd)\s*[=:]\s*\S+'),
        re.compile(r'(?i)(api[_-]?key|apikey)\s*[=:]\s*\S+'),
        re.compile(r'(?i)(secret|token)\s*[=:]\s*\S+'),
        re.compile(r'postgresql://[^@]*:[^@]*@'),
        re.compile(r'mysql://[^@]*:[^@]*@'),
    ]
    
    def filter(self, record):
        record.msg = self._mask(str(record.msg))
        if record.args:
            if isinstance(record.args, tuple):
                record.args = tuple(self._mask(str(a)) for a in record.args)
            elif isinstance(record.args, dict):
                record.args = {k: self._mask(str(v)) for k, v in record.args.items()}
        return True
    
    def _mask(self, text: str) -> str:
        for pattern in self.PATTERNS:
            text = pattern.sub('[REDACTED]', text)
        return text

# เพิ่ม filter ไปยัง logger
logging.getLogger().addFilter(SensitiveDataFilter())
```

### HTTP Request Logging ที่ปลอดภัย

```python
# ❌ อันตราย - log headers ทั้งหมด
def log_request(request):
    logger.debug(f"Request headers: {dict(request.headers)}")
    # Authorization header จะถูก log!

# ✅ ปลอดภัย - filter headers ที่ sensitive
SENSITIVE_HEADERS = {
    'authorization',
    'x-api-key',
    'x-auth-token',
    'cookie',
    'set-cookie',
    'x-csrf-token',
}

def log_request_safely(request):
    safe_headers = {
        key: '***' if key.lower() in SENSITIVE_HEADERS else value
        for key, value in request.headers.items()
    }
    logger.debug(f"Request: {request.method} {request.path}")
    logger.debug(f"Headers: {safe_headers}")
```

---

## แบบฝึกหัด (Exercises) {#exercises}

### Exercise 1: ตั้งค่า GitHub Secrets สำหรับ Multi-Environment

**งาน:**
1. สร้าง GitHub Environments: `staging` และ `production`
2. เพิ่ม secrets ที่แตกต่างกันในแต่ละ environment
3. เขียน workflow ที่ใช้ environment-specific secrets
4. ตั้งค่า protection rules: production ต้องมี manual approval

```yaml
# solution/workflows/multi-env-deploy.yml
name: Multi-Environment Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to staging
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}    # Staging DB
          API_KEY: ${{ secrets.API_KEY }}              # Staging API key
        run: |
          echo "Deploying to staging with staging-specific secrets"
          # Secrets จะต่างกันระหว่าง staging และ production
  
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}    # Production DB (different!)
          API_KEY: ${{ secrets.API_KEY }}              # Production API key (different!)
        run: |
          echo "Deploying to production with production-specific secrets"
```

### Exercise 2: Setup Gitleaks ใน Pipeline

**งาน:**
1. เพิ่ม gitleaks scan ใน GitHub Actions workflow
2. สร้าง custom .gitleaks.toml configuration
3. เพิ่ม pre-commit hook ที่ run gitleaks locally
4. จงจงสร้าง test case ที่ intentionally fail เพื่อ verify ว่า scan ทำงาน

```yaml
# solution: .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Scan for secrets
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        
      - name: Check for test patterns
        run: |
          # Verify our test file with fake credentials is properly caught
          cat test-secrets.txt | gitleaks detect --pipe --verbose || true
```

### Exercise 3: Implement Secret Rotation

**งาน:**
1. สร้าง script ที่ rotate database password
2. Update secret ใน Kubernetes/environment
3. Verify application ยังทำงานได้หลัง rotation
4. Setup automated rotation ทุก 30 วัน

```bash
#!/bin/bash
# solution/scripts/rotate-db-password.sh

set -euo pipefail

DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-myapp}"
DB_USER="${DB_USER:-app_user}"
NAMESPACE="${NAMESPACE:-myapp-production}"
SECRET_NAME="${SECRET_NAME:-myapp-database}"

echo "=== Rotating Database Password ==="

# สร้าง new password
NEW_PASSWORD=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9!@#$%' | head -c 32)

echo "Changing database password..."

# เปลี่ยน password ใน database
PGPASSWORD="${DB_ADMIN_PASSWORD}" psql \
  -h "${DB_HOST}" \
  -p "${DB_PORT}" \
  -U "postgres" \
  -d "${DB_NAME}" \
  -c "ALTER USER ${DB_USER} WITH PASSWORD '${NEW_PASSWORD}';"

echo "Updating Kubernetes secret..."

# Update Kubernetes secret
kubectl create secret generic "${SECRET_NAME}" \
  --from-literal=DB_PASSWORD="${NEW_PASSWORD}" \
  --namespace "${NAMESPACE}" \
  --dry-run=client -o yaml | kubectl apply -f -

echo "Triggering rolling restart..."

# Restart pods เพื่อ pick up new secret
kubectl rollout restart deployment/myapp \
  --namespace "${NAMESPACE}"

# รอ deployment เสร็จ
kubectl rollout status deployment/myapp \
  --namespace "${NAMESPACE}" \
  --timeout=5m

echo "Testing connection with new password..."

# Test new password
PGPASSWORD="${NEW_PASSWORD}" psql \
  -h "${DB_HOST}" \
  -p "${DB_PORT}" \
  -U "${DB_USER}" \
  -d "${DB_NAME}" \
  -c "SELECT current_user, now();"

echo "✅ Password rotation completed successfully!"

# Log rotation event (ไม่ log password!)
echo "Rotation completed at $(date) for user ${DB_USER}" >> /var/log/secret-rotations.log
```

### Exercise 4: Secret Scanning Pipeline

สร้าง complete security scanning pipeline ที่:
1. Scan สำหรับ hardcoded secrets
2. Check dependencies สำหรับ known vulnerabilities
3. Verify ไม่มี .env files ใน repository
4. Generate security report

```yaml
# solution: .github/workflows/security-audit.yml
name: Security Audit

on:
  schedule:
    - cron: '0 9 * * 1'    # ทุกวันจันทร์ 9 โมง
  workflow_dispatch:

jobs:
  full-security-scan:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # Full history สำหรับ gitleaks
      
      - name: Scan secrets in code
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Check for .env files
        run: |
          if git log --all --name-only --format="" | grep -E "^\.env($|\.)"; then
            echo "⚠️  Found .env files in git history!"
            exit 1
          fi
          echo "✅ No .env files found in history"
      
      - name: Node.js dependency audit
        run: npm audit --audit-level=moderate
      
      - name: Generate security report
        run: |
          echo "# Security Audit Report" > security-report.md
          echo "Date: $(date)" >> security-report.md
          echo "Repository: ${{ github.repository }}" >> security-report.md
          echo "" >> security-report.md
          echo "## Results" >> security-report.md
          echo "- Secret scan: PASSED" >> security-report.md
          echo "- .env check: PASSED" >> security-report.md
          echo "- Dependency audit: PASSED" >> security-report.md
      
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: security-report
          path: security-report.md
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **ทำไม Secrets ต้องระวัง** - ตัวอย่างจริงของ data breaches ที่เกิดจาก secret exposure
2. **ประเภทของ Secrets** - API keys, database credentials, certificates, OAuth tokens
3. **GitHub/GitLab Secrets** - การจัดการ secrets บน platform ที่ใช้อยู่
4. **HashiCorp Vault** - Centralized secrets management พร้อม dynamic credentials
5. **AWS Secrets Manager/Azure Key Vault** - Cloud-native secrets solutions
6. **Secret Rotation** - การ rotate secrets อัตโนมัติเพื่อลดความเสี่ยง
7. **OIDC Keyless Auth** - วิธีที่ดีที่สุดในการ authenticate CI/CD pipelines
8. **Docker Secrets** - BuildKit secrets และ runtime secrets
9. **Secret Scanning** - gitleaks, detect-secrets, git-secrets
10. **Best Practices** - ทำและไม่ทำ, Zero-trust architecture
11. **ห้าม Log Secrets** - วิธีป้องกัน accidental logging

### Key Takeaways

- **Secrets ใน code = time bomb** - scan ทุก commit ก่อน push
- **Rotate regularly** - ถึงแม้ยังไม่ compromise
- **Use OIDC** - ดีกว่า long-lived tokens เสมอ
- **Least privilege** - ให้สิทธิ์น้อยที่สุดที่จำเป็น
- **Audit logs** - รู้ว่าใครเข้าถึง secret อะไร เมื่อไหร่
- **Never log secrets** - แม้ใน debug mode

---

*บทต่อไป: Part 20 - Artifact Management & Storage*

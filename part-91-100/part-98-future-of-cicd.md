# Part 98: Future of CI/CD

## บทนำ

CI/CD กำลังเปลี่ยนแปลงอย่างรวดเร็วด้วยพลังของ AI, Platform Engineering, และ emerging technologies บทนี้สำรวจอนาคตของ CI/CD:

- AI-Native Pipelines
- Platform Engineering Evolution
- WebAssembly ใน CI/CD
- Edge Deployment
- Quantum-Safe Cryptography
- Supply Chain Security
- Emerging Standards
- แนวโน้มที่ควรติดตาม

---

## 98.1 AI-Native CI/CD Pipelines

### AI-Assisted Pipeline Optimization

```python
# ai_pipeline_optimizer.py
# ตัวอย่าง AI-driven pipeline optimization

import anthropic
import json
from typing import List, Dict

class AIPipelineOptimizer:
    """
    ใช้ AI ช่วย optimize CI/CD pipelines
    - วิเคราะห์ pipeline performance
    - แนะนำ optimizations
    - Predict build failures
    - Auto-generate pipeline configs
    """
    
    def __init__(self):
        self.client = anthropic.Anthropic()
    
    def analyze_pipeline_bottleneck(self, pipeline_metrics: dict) -> dict:
        """วิเคราะห์ bottleneck ใน pipeline"""
        
        prompt = f"""
        วิเคราะห์ CI/CD pipeline metrics ต่อไปนี้และระบุ bottlenecks:
        
        Pipeline Metrics:
        {json.dumps(pipeline_metrics, indent=2, ensure_ascii=False)}
        
        กรุณาระบุ:
        1. Top 3 bottlenecks ที่ใช้เวลามากที่สุด
        2. สาเหตุที่เป็นไปได้
        3. Recommendations เฉพาะเจาะจงสำหรับแต่ละ bottleneck
        4. Expected time savings ถ้าแก้ไข
        
        ตอบเป็น JSON format
        """
        
        message = self.client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return json.loads(message.content[0].text)
    
    def generate_pipeline_config(self, requirements: dict) -> str:
        """Auto-generate pipeline configuration"""
        
        prompt = f"""
        สร้าง GitHub Actions CI/CD pipeline สำหรับ project ที่มี requirements ดังนี้:
        
        {json.dumps(requirements, indent=2, ensure_ascii=False)}
        
        Pipeline ต้องมี:
        1. Code quality checks
        2. Security scanning
        3. Automated tests
        4. Docker build
        5. Deployment (ตาม requirements)
        
        ตอบเป็น valid YAML สำหรับ GitHub Actions
        """
        
        message = self.client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2048,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return message.content[0].text
    
    def predict_build_failure(self, 
                               pr_diff: str, 
                               historical_failures: list) -> dict:
        """Predict ว่า build จะ fail หรือไม่"""
        
        prompt = f"""
        วิเคราะห์ Pull Request diff ต่อไปนี้และ predict ว่าจะมีปัญหาอะไรบ้าง:
        
        PR Diff (summary):
        {pr_diff[:2000]}  # Truncate for context limit
        
        Historical failure patterns:
        {json.dumps(historical_failures[:5], indent=2)}
        
        ให้ตอบ:
        1. Risk score (0-10, 10 = highest risk)
        2. Most likely failure points
        3. Specific files/changes ที่ควรระวัง
        4. Pre-check suggestions ก่อน push
        """
        
        message = self.client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        )
        
        return {'analysis': message.content[0].text}

# ตัวอย่าง pipeline metrics
pipeline_metrics = {
    "stages": {
        "checkout": {"avg_duration_seconds": 5},
        "install_deps": {"avg_duration_seconds": 180},
        "lint": {"avg_duration_seconds": 45},
        "unit_tests": {"avg_duration_seconds": 240},
        "integration_tests": {"avg_duration_seconds": 480},
        "build_docker": {"avg_duration_seconds": 300},
        "container_scan": {"avg_duration_seconds": 120},
        "deploy_staging": {"avg_duration_seconds": 60}
    },
    "total_avg_minutes": 24.3,
    "cache_hit_rate": 0.35,
    "parallel_jobs": 1
}

optimizer = AIPipelineOptimizer()
analysis = optimizer.analyze_pipeline_bottleneck(pipeline_metrics)
print(json.dumps(analysis, indent=2, ensure_ascii=False))
```

### AI Code Review ใน CI/CD

```yaml
# .github/workflows/ai-code-review.yml
# AI-powered code review ใน PR pipeline

name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    name: AI Code Review
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Get PR diff
        id: diff
        run: |
          git diff ${{ github.event.pull_request.base.sha }}...${{ github.sha }} > pr.diff
          echo "lines=$(wc -l < pr.diff)" >> $GITHUB_OUTPUT
      
      - name: AI Code Review
        if: steps.diff.outputs.lines < 1000  # Skip ถ้า diff ใหญ่เกิน
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          python scripts/ai_code_review.py \
            --diff-file pr.diff \
            --repo ${{ github.repository }} \
            --pr-number ${{ github.event.pull_request.number }} \
            --post-comment
```

```python
# scripts/ai_code_review.py

import anthropic
import subprocess
import os
import requests

def review_code_with_ai(diff_content: str) -> str:
    client = anthropic.Anthropic()
    
    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"""
            Review code changes และให้ feedback:
            
            ```diff
            {diff_content}
            ```
            
            มองหา:
            1. Security vulnerabilities (SQL injection, XSS, etc.)
            2. Performance issues
            3. Code smells
            4. Missing error handling
            5. Logic bugs
            
            ตอบเป็น Markdown format ที่ developer อ่านง่าย
            ถ้าไม่มีปัญหาสำคัญ บอก "LGTM" พร้อมคำอธิบาย
            """
        }]
    )
    
    return message.content[0].text

def post_pr_comment(repo: str, pr_number: int, comment: str, token: str):
    """Post comment ไปยัง GitHub PR"""
    headers = {
        'Authorization': f'Bearer {token}',
        'Accept': 'application/vnd.github.v3+json'
    }
    
    url = f"https://api.github.com/repos/{repo}/issues/{pr_number}/comments"
    
    response = requests.post(url, headers=headers, json={
        'body': f"## 🤖 AI Code Review\n\n{comment}"
    })
    
    response.raise_for_status()

if __name__ == '__main__':
    import argparse
    
    parser = argparse.ArgumentParser()
    parser.add_argument('--diff-file', required=True)
    parser.add_argument('--repo', required=True)
    parser.add_argument('--pr-number', type=int, required=True)
    parser.add_argument('--post-comment', action='store_true')
    args = parser.parse_args()
    
    with open(args.diff_file, 'r') as f:
        diff = f.read()
    
    review = review_code_with_ai(diff)
    
    if args.post_comment:
        post_pr_comment(
            args.repo, 
            args.pr_number, 
            review,
            os.environ['GITHUB_TOKEN']
        )
    else:
        print(review)
```

---

## 98.2 Platform Engineering Evolution

### Internal Developer Platform (IDP)

```
Internal Developer Platform Architecture
═════════════════════════════════════════

Developer Experience (Portals)
├── Backstage (Spotify)
│   ├── Service Catalog
│   ├── Tech Docs
│   ├── Software Templates
│   └── API Documentation
│
├── Self-Service Provisioning
│   ├── Create new service (scaffolding)
│   ├── Provision databases
│   ├── Create cloud resources
│   └── Configure monitoring
│
└── Observability Dashboard
    ├── Pipeline status
    ├── Deployment history
    └── Service health

Platform Layer
├── Shared Infrastructure
│   ├── Kubernetes clusters
│   ├── Service mesh (Istio)
│   └── Shared observability
│
├── Golden Paths
│   ├── Standard pipeline templates
│   ├── Approved base images
│   ├── Infrastructure templates
│   └── Security defaults
│
└── Self-Service APIs
    ├── Provision environments
    ├── Manage secrets
    └── Deploy applications
```

### Backstage Service Catalog

```yaml
# catalog-info.yaml
# Service definition ใน Backstage

apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: "Core payment processing service"
  annotations:
    github.com/project-slug: company/payment-service
    backstage.io/techdocs-ref: dir:.
    pagerduty.com/service-id: "PD1234"
    sonarqube.org/project-key: "payment-service"
    argocd/app-name: "payment-service-prod"
    
  tags:
    - payment
    - critical
    - pci-dss
    
  links:
    - url: https://grafana.company.com/d/payment
      title: Monitoring Dashboard
      icon: dashboard
    - url: https://runbooks.company.com/payment
      title: Runbook
      icon: docs
      
spec:
  type: service
  lifecycle: production
  owner: payments-team
  
  dependsOn:
    - component:default/postgres-payment
    - component:default/redis-cache
    - resource:default/aws-s3-receipts
    
  providesApis:
    - payment-api-v1
    - payment-api-v2
    
  consumesApis:
    - fraud-detection-api
    - notification-api
```

### Golden Path Templates

```typescript
// backstage-template.yaml
// Software Template สำหรับสร้าง microservice ใหม่

apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: node-microservice
  title: Node.js Microservice
  description: สร้าง Node.js microservice พร้อม CI/CD pipeline มาตรฐาน
  tags:
    - node
    - microservice
    - typescript

spec:
  owner: platform-team
  type: service
  
  parameters:
    - title: Service Information
      required: [name, description, owner]
      properties:
        name:
          title: Service Name
          type: string
          description: ชื่อ service (lowercase, kebab-case)
          pattern: "^[a-z][a-z0-9-]*[a-z0-9]$"
          
        description:
          title: Description
          type: string
          
        owner:
          title: Owner Team
          type: string
          ui:field: OwnerPicker
          
        database:
          title: Database
          type: string
          enum: [none, postgres, mysql, mongodb, redis]
          default: none
          
    - title: Pipeline Configuration
      properties:
        enableSast:
          title: Enable SAST (SonarQube)
          type: boolean
          default: true
          
        enablePerformanceTests:
          title: Enable Performance Tests (k6)
          type: boolean
          default: false
          
        environments:
          title: Deploy Environments
          type: array
          items:
            type: string
            enum: [dev, staging, production]
          default: [dev, staging, production]
  
  steps:
    - id: fetch-template
      name: Fetch Base Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          database: ${{ parameters.database }}
    
    - id: create-repo
      name: Create GitHub Repository
      action: github:repo:create
      input:
        repoUrl: github.com?repo=${{ parameters.name }}&owner=company
        description: ${{ parameters.description }}
        
    - id: publish
      name: Push to Repository
      action: publish:github
      input:
        sourcePath: .
        repoUrl: github.com?repo=${{ parameters.name }}&owner=company
    
    - id: create-argocd-app
      name: Create ArgoCD Application
      action: argocd:create-resources
      input:
        appName: ${{ parameters.name }}
        
    - id: register-catalog
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
```

---

## 98.3 WebAssembly ใน CI/CD

### Wasm สำหรับ Plugin Architecture

```
WebAssembly in CI/CD
════════════════════

ทำไม WASM ถึงน่าสนใจสำหรับ CI/CD:

1. SANDBOXED EXECUTION
   - Run untrusted plugins อย่างปลอดภัย
   - ไม่สามารถ escape sandbox
   - Better than Docker สำหรับ plugins (lighter weight)

2. LANGUAGE AGNOSTIC
   - เขียน CI/CD plugins ด้วยภาษาใดก็ได้
   - Compile to WASM, run anywhere

3. PERFORMANCE
   - Near-native performance
   - Fast startup (microseconds, not milliseconds)

4. PORTABILITY
   - Same binary runs บน Linux/Mac/Windows
   - No dependency issues

ตัวอย่างการใช้งาน:
- Dagger (CI/CD platform ด้วย WASM)
- wasmPlugin ใน Istio/Envoy
- Custom security scanners
```

### Dagger - CI/CD as Code

```go
// dagger/main.go
// CI/CD pipeline เขียนด้วย Go + Dagger (runs as WASM)

package main

import (
    "context"
    "fmt"
    
    "dagger.io/dagger"
)

func main() {
    ctx := context.Background()
    
    // เชื่อมต่อกับ Dagger
    client, err := dagger.Connect(ctx, dagger.WithLogOutput(os.Stderr))
    if err != nil {
        panic(err)
    }
    defer client.Close()
    
    // Source code
    src := client.Host().Directory(".")
    
    // Pipeline definition
    if err := pipeline(ctx, client, src); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}

func pipeline(ctx context.Context, client *dagger.Client, src *dagger.Directory) error {
    // ใช้ Node.js container
    node := client.Container().
        From("node:18-alpine").
        WithDirectory("/app", src).
        WithWorkdir("/app")
    
    // Install dependencies
    install := node.WithExec([]string{"npm", "ci"})
    
    // Run lint (parallel)
    lintFuture := install.WithExec([]string{"npm", "run", "lint"})
    
    // Run tests (parallel)
    testFuture := install.
        WithExec([]string{"npm", "test", "--", "--coverage"})
    
    // Wait for lint + test
    if _, err := lintFuture.Sync(ctx); err != nil {
        return fmt.Errorf("lint failed: %w", err)
    }
    
    testOutput, err := testFuture.Sync(ctx)
    if err != nil {
        return fmt.Errorf("tests failed: %w", err)
    }
    
    // Export test results
    if _, err := testOutput.Directory("coverage").Export(ctx, "./coverage"); err != nil {
        return fmt.Errorf("failed to export coverage: %w", err)
    }
    
    // Build Docker image
    imageRef, err := client.Container().
        Build(src).
        Publish(ctx, "registry.company.com/myapp:latest")
    
    if err != nil {
        return fmt.Errorf("build failed: %w", err)
    }
    
    fmt.Printf("Published: %s\n", imageRef)
    return nil
}
```

```python
# Python version ของ Dagger pipeline

import anyio
import dagger

async def main():
    async with dagger.Connection(dagger.Config(log_output=sys.stderr)) as client:
        src = await client.host().directory(".").id()
        
        # Node.js container
        node = (
            client.container()
            .from_("node:18-alpine")
            .with_mounted_directory("/app", client.host().directory("."))
            .with_workdir("/app")
        )
        
        # Install
        with_deps = await node.with_exec(["npm", "ci"])
        
        # Parallel: lint + test
        lint_result, test_result = await anyio.gather(
            with_deps.with_exec(["npm", "run", "lint"]).sync(),
            with_deps.with_exec(["npm", "test"]).sync(),
        )
        
        print("Pipeline succeeded!")

anyio.run(main)
```

---

## 98.4 Edge Deployment และ Multi-Region CI/CD

### Edge-First Deployment Strategy

```
Edge Deployment CI/CD Architecture
════════════════════════════════════

Traditional:
Developer → Build → Central DC → Users (latency)

Edge-First:
Developer → Build → Central DC → [Edge Nodes Worldwide] → Users (low latency)

Edge Platforms:
- Cloudflare Workers
- Vercel Edge Functions
- AWS Lambda@Edge
- Fastly Compute@Edge
- Fly.io

CI/CD สำหรับ Edge:
1. Build ครั้งเดียว
2. Deploy to edge nodes ทั่วโลก
3. A/B test per region
4. Rollback per region
```

```yaml
# .github/workflows/edge-deployment.yml
# Deploy to Cloudflare Workers Edge

name: Edge Deployment

on:
  push:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Wrangler
        run: npm install -g wrangler
      
      - name: Run Tests
        run: npm test
      
      - name: Build for Edge
        run: npm run build:edge
  
  deploy-staging:
    needs: build-and-test
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.company.workers.dev
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Staging Edge
        run: |
          wrangler deploy --env staging
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
  
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.company.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Production Edge
        run: |
          # Deploy with progressive rollout
          wrangler deploy \
            --env production \
            --compatibility-date 2024-01-01
          
          # Deploy to specific regions first
          wrangler deploy \
            --env production \
            --routes "th.api.company.com/*"  # Thailand first
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
      
      - name: Verify Edge Deployment
        run: |
          # Test each edge location
          for region in th sg jp au us; do
            curl -sf "https://${region}.api.company.com/health" || exit 1
            echo "${region}: OK"
          done
```

---

## 98.5 Supply Chain Security

### Software Supply Chain Attacks

```
Supply Chain Security Threats
═════════════════════════════

1. DEPENDENCY CONFUSION
   - Attacker อัปโหลด malicious package ชื่อเดียวกับ private package
   - Package manager ดึง public version แทน private
   
2. TYPOSQUATTING
   - "lodash" vs "lodahs" (malicious typo)
   - "colors" vs "colour" 
   
3. DEPENDENCY HIJACKING
   - Maintainer account ถูก compromise
   - Malicious code inject ใน popular package
   - ตัวอย่าง: event-stream (2018), ua-parser-js (2021)
   
4. BUILD SYSTEM COMPROMISE
   - CI/CD server ถูก compromise
   - Malicious code inject ขณะ build
   - ตัวอย่าง: SolarWinds (2020)
   
5. CONTAINER POISONING
   - Base image มี malware
   - Registry คือจุดอ่อน
```

### SLSA Framework

```
SLSA (Supply-chain Levels for Software Artifacts)
═══════════════════════════════════════════════════

Level 1: Build Process
- Build scripts documented
- Provenance generated
- Not tampered easily

Level 2: Build Service
- Uses hosted build service
- Provenance signed by build service
- ✅ GitHub Actions, GitLab CI qualify

Level 3: Hardened Build Service
- Isolated builds
- Two-party review for build changes
- More comprehensive attestation

Level 4: Two-Person Review (coming)
- All changes reviewed by 2 people
- Hermetic, reproducible builds

Current realistic target for most orgs: SLSA Level 2-3
```

```yaml
# SLSA Provenance ใน GitHub Actions

name: Build with SLSA Provenance

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  id-token: write  # สำหรับ OIDC token signing
  packages: write

jobs:
  build:
    uses: slsa-framework/slsa-github-generator/.github/workflows/builder_go_slsa3.yml@v1.9.0
    with:
      go-version: "1.21"
      upload-assets: true
```

```python
# verify_provenance.py
# ตรวจสอบ SLSA provenance ของ artifacts

import subprocess
import json

def verify_artifact_provenance(artifact_path: str, 
                                 expected_source: str,
                                 expected_builder: str) -> bool:
    """
    ตรวจสอบว่า artifact มาจาก trusted source
    และ built by trusted builder
    """
    
    # ใช้ slsa-verifier
    result = subprocess.run([
        'slsa-verifier', 'verify-artifact',
        artifact_path,
        '--provenance-path', f'{artifact_path}.intoto.jsonl',
        '--source-uri', expected_source,
        '--builder-id', expected_builder
    ], capture_output=True, text=True)
    
    if result.returncode == 0:
        print(f"✅ Provenance verified for {artifact_path}")
        return True
    else:
        print(f"❌ Provenance verification failed: {result.stderr}")
        return False

# Sigstore: Sign and verify ด้วย keyless signing
def sign_artifact_with_sigstore(artifact_path: str):
    """
    Sign artifact โดยใช้ Sigstore (keyless signing)
    - ไม่ต้องจัดการ private keys
    - Identity-based signing (GitHub Actions OIDC)
    - Transparency log (Rekor)
    """
    
    subprocess.run([
        'cosign', 'sign-blob',
        '--yes',
        '--output-signature', f'{artifact_path}.sig',
        '--output-certificate', f'{artifact_path}.cert',
        artifact_path
    ], check=True)
    
    print(f"✅ Artifact signed: {artifact_path}")

def verify_with_sigstore(artifact_path: str, cert_identity: str):
    """ตรวจสอบ signature ด้วย Sigstore"""
    
    result = subprocess.run([
        'cosign', 'verify-blob',
        '--certificate', f'{artifact_path}.cert',
        '--signature', f'{artifact_path}.sig',
        '--certificate-identity', cert_identity,
        '--certificate-oidc-issuer', 'https://token.actions.githubusercontent.com',
        artifact_path
    ], capture_output=True, text=True)
    
    return result.returncode == 0
```

---

## 98.6 Quantum-Safe Cryptography

```python
# quantum_safe_ci_cd.py
# เตรียม CI/CD สำหรับ post-quantum world

"""
ทำไมต้องสนใจ Quantum-Safe Cryptography ใน CI/CD?

1. HARVEST NOW, DECRYPT LATER
   - Adversaries อาจ collect encrypted data วันนี้
   - เพื่อ decrypt ในอนาคตเมื่อมี quantum computer
   - Secrets ที่อยู่ใน pipeline logs อาจถูก decrypt ในอนาคต

2. DIGITAL SIGNATURES
   - Code signing certificates
   - Artifact provenance signatures
   - Pipeline authentication tokens
   ⚠️ RSA/ECDSA จะ broken เมื่อ quantum computers มา

3. TIMELINE
   - NIST ได้ finalize post-quantum algorithms (2024)
   - Migration window: ~10 ปี
   - Start planning NOW สำหรับ long-lived secrets

NIST Post-Quantum Standards (2024):
- ML-KEM (Kyber): Key encapsulation
- ML-DSA (Dilithium): Digital signatures  
- SLH-DSA (SPHINCS+): Hash-based signatures
"""

# การเตรียมความพร้อมสำหรับ CI/CD

class QuantumSafeCICDChecklist:
    
    def audit_cryptographic_usage(self, codebase_path: str) -> dict:
        """
        ตรวจสอบการใช้ cryptography ใน codebase
        ระบุ algorithms ที่จะ vulnerable ต่อ quantum attacks
        """
        
        vulnerable_patterns = {
            'RSA': r'RSA|rsa_generate|generateRSAKeyPair',
            'ECDSA': r'ECDSA|createECDH|ec\.generateKeys',
            'DH': r'DiffieHellman|createDiffieHellman',
            'MD5': r'md5|createHash.*md5',
            'SHA1': r'sha1|createHash.*sha1'
        }
        
        # Scan codebase
        findings = {}
        for algo, pattern in vulnerable_patterns.items():
            # Use grep equivalent
            count = self.scan_pattern(codebase_path, pattern)
            if count > 0:
                findings[algo] = {
                    'count': count,
                    'quantum_safe': False,
                    'recommendation': self.get_recommendation(algo)
                }
        
        return findings
    
    def get_recommendation(self, algorithm: str) -> str:
        recommendations = {
            'RSA': 'Migrate to ML-KEM (CRYSTALS-Kyber) for encryption, ML-DSA for signatures',
            'ECDSA': 'Migrate to ML-DSA (CRYSTALS-Dilithium) or SLH-DSA (SPHINCS+)',
            'DH': 'Use ML-KEM for key agreement',
            'MD5': 'Already weak! Migrate to SHA-256 minimum',
            'SHA1': 'Migrate to SHA-256 or SHA-3'
        }
        return recommendations.get(algorithm, 'Review usage')
    
    def generate_migration_roadmap(self) -> str:
        return """
# Post-Quantum Migration Roadmap สำหรับ CI/CD

## Phase 1: Inventory (ทำได้เลย)
- [ ] Audit all cryptographic usage ใน codebase
- [ ] Identify long-lived secrets (> 10 ปี)
- [ ] Map certificate dependencies
- [ ] Assess third-party library quantum readiness

## Phase 2: Low-Hanging Fruit (2024-2025)  
- [ ] อัปเดต TLS to 1.3 (already better than TLS 1.2)
- [ ] Migrate จาก MD5/SHA1 ไป SHA-256+
- [ ] Enable hybrid modes (classical + post-quantum)
- [ ] Update code signing certificates timeline

## Phase 3: Core Migration (2025-2027)
- [ ] Migrate code signing to post-quantum algorithms
- [ ] Update CI/CD tool authentication mechanisms
- [ ] Migrate artifact signing (Sigstore → PQ-safe Sigstore)
- [ ] Update internal CAs

## Phase 4: Complete Migration (2027-2030)
- [ ] Remove classical-only cryptography
- [ ] Full post-quantum key infrastructure
- [ ] Audit and verify all components

## Tools ที่ต้องติดตาม
- OpenSSL 3.x: Post-quantum support
- AWS KMS: PQ algorithm support timeline
- HashiCorp Vault: PQ roadmap
- Let's Encrypt: PQ certificate timeline
"""
    
    def scan_pattern(self, path: str, pattern: str) -> int:
        """Mock implementation"""
        return 0
```

---

## 98.7 Emerging Trends

### Platform Engineering Trends

```markdown
# Top CI/CD Trends 2024-2026

## 1. AI-Assisted Development (Happening Now)
- GitHub Copilot, Cursor สำหรับ code generation
- AI code review ใน PR pipeline
- AI-powered test generation
- Intelligent pipeline optimization
- Predictive failure detection

## 2. Internal Developer Platforms (IDP) Growth
- Backstage adoption เพิ่มขึ้นมาก
- Self-service infrastructure provisioning
- Golden paths ลด cognitive load
- Platform as a Product mindset

## 3. eBPF-Based Observability
- Tetragon (Cilium): eBPF-based security
- Pixie: Auto-instrumentation ด้วย eBPF
- ไม่ต้อง modify application code
- More security + observability data

## 4. Supply Chain Security Maturity
- SBOM (Software Bill of Materials) mandatory
- SLSA Level 3+ สำหรับ critical software
- Policy-as-code สำหรับ deployment gates
- Keyless signing (Sigstore) mainstream

## 5. Platform Engineering Specialization
- Platform teams เป็น product teams
- "Paved roads" แทน "guardrails"
- Developer Experience (DX) metrics
- Cognitive load reduction

## 6. GitOps Maturity
- From GitOps tools → GitOps practices
- Multi-cluster management
- Policy enforcement ผ่าน GitOps
- Secret management ใน GitOps (External Secrets)

## 7. Shift-Left Security Evolution
- Security ใน IDE (real-time SAST)
- Policy-as-code (OPA, Kyverno)
- Supply chain security automation
- Compliance-as-code

## 8. Serverless และ FaaS CI/CD
- CI/CD สำหรับ Lambda/Cloud Functions
- Cold start optimization
- Canary deployments สำหรับ serverless
- Infrastructure-less testing

## 9. WebAssembly Pipeline
- Dagger WASM pipelines
- WASM plugin architecture
- Cross-language pipeline components

## 10. Sustainability (Green Software)
- Carbon-aware scheduling
- Efficiency metrics ใน CI/CD
- Resource optimization automation
- Sustainable infrastructure choices
```

### Policy as Code

```python
# policy_as_code.py
# Open Policy Agent (OPA) สำหรับ CI/CD gates

# deployment-policy.rego

package cicd.deployment

# Default: deny everything
default allow = false

# อนุญาต deployment ถ้า:
# 1. Image มาจาก approved registry
# 2. ผ่าน security scans
# 3. ได้รับ appropriate approvals

allow {
    input.image_source == "registry.company.com"
    input.security_scan_passed == true
    valid_approvals
    not_blocked_by_freeze
}

# ตรวจสอบ approvals
valid_approvals {
    input.environment == "development"
    # Dev: ไม่ต้องการ approval
}

valid_approvals {
    input.environment == "staging"
    input.approvers[_] == "tech-lead"  # ต้องการ tech lead approval
}

valid_approvals {
    input.environment == "production"
    count(input.approvers) >= 2  # ต้องการ 2 approvers
    input.approvers[_] == "engineering-manager"
    input.change_ticket != ""  # ต้องมี change ticket
}

# Deployment freeze check
not_blocked_by_freeze {
    not data.deployment_freeze.active
}

not_blocked_by_freeze {
    data.deployment_freeze.active
    input.change_type == "emergency"  # Emergency override
}
```

```bash
# ใช้ OPA ใน CI/CD pipeline

# ตรวจสอบ deployment policy
check_deployment_policy() {
    local input_json=$1
    
    # Query OPA
    RESULT=$(echo "$input_json" | \
        curl -s -X POST \
        -H "Content-Type: application/json" \
        -d @- \
        "https://opa.company.internal/v1/data/cicd/deployment/allow")
    
    ALLOWED=$(echo "$RESULT" | jq '.result')
    
    if [ "$ALLOWED" = "true" ]; then
        echo "✅ Deployment policy: ALLOWED"
        return 0
    else
        echo "❌ Deployment policy: DENIED"
        echo "Reason: $(echo "$RESULT" | jq -r '.result')"
        return 1
    fi
}

# Build input JSON
INPUT=$(cat << EOF
{
    "image_source": "registry.company.com",
    "image_tag": "${IMAGE_TAG}",
    "environment": "${ENVIRONMENT}",
    "security_scan_passed": true,
    "approvers": ["${APPROVER_1}", "${APPROVER_2}"],
    "change_ticket": "${CHANGE_TICKET}",
    "change_type": "normal"
}
EOF
)

check_deployment_policy "$INPUT" || exit 1
```

---

## 98.8 แบบฝึกหัด

### แบบฝึกหัดที่ 1: AI Pipeline Optimizer

**โจทย์:** สร้าง simple AI pipeline analyzer

```python
# TODO: สร้าง AI pipeline analyzer ที่:
# 1. รับ pipeline execution history
# 2. วิเคราะห์ stage duration trends
# 3. Identify stages ที่ duration เพิ่มขึ้น (อาจมีปัญหา)
# 4. แนะนำ optimizations

pipeline_history = [
    {
        "build_id": "1234",
        "date": "2024-01-01",
        "stages": {
            "unit-tests": 180,
            "integration-tests": 420,
            "docker-build": 120
        }
    },
    # เพิ่ม data points อีก...
]

class PipelineAnalyzer:
    def analyze_trends(self, history: list) -> dict:
        """TODO: วิเคราะห์ trends"""
        pass
    
    def recommend_optimizations(self, analysis: dict) -> list:
        """TODO: สร้าง recommendations"""
        pass
```

### แบบฝึกหัดที่ 2: Policy as Code

**โจทย์:** เขียน OPA policy สำหรับ container deployment

```rego
# TODO: เขียน OPA policy ที่:
# 1. ห้าม deploy containers ที่ run as root
# 2. ต้องมี resource limits กำหนด
# 3. ต้องมี liveness + readiness probes
# 4. ห้าม use :latest tag

package kubernetes.admission

# TODO: implement rules
```

### แบบฝึกหัดที่ 3: Supply Chain Security

**โจทย์:** Implement basic supply chain security checks

```bash
#!/bin/bash
# TODO: สร้าง supply chain security check script
# ที่ตรวจสอบ:
# 1. Generate SBOM สำหรับ container image
# 2. Check for known vulnerable packages ใน SBOM
# 3. Verify image signature
# 4. Output security report

SERVICE_NAME="myapp"
IMAGE_TAG="v1.2.3"

# TODO: implement
echo "TODO: Implement supply chain security checks"
```

---

## สรุป

อนาคตของ CI/CD จะ shaped โดย:

1. **AI Integration** — AI ช่วย optimize, predict failures, generate configs
2. **Platform Engineering** — IDP เป็น product, golden paths
3. **WebAssembly** — Lightweight, portable pipeline components
4. **Edge Computing** — Deploy closer to users
5. **Supply Chain Security** — SLSA, SBOM, Sigstore
6. **Post-Quantum Cryptography** — Start planning migration now
7. **Policy as Code** — OPA, Kyverno สำหรับ automated governance

สิ่งที่ engineer ควรเตรียมตัว:
- เรียนรู้ AI tools (GitHub Copilot, Claude API)
- Understand Platform Engineering concepts
- Learn eBPF observability
- Study supply chain security practices
- Follow NIST Post-Quantum standards

---

**ต่อไป:** Part 99 - Capstone Project

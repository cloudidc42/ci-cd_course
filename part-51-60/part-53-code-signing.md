# Part 53: Code Signing และ Artifact Attestation

## บทนำ

Code Signing คือกระบวนการลงนามดิจิทัลบน software artifacts เพื่อยืนยันว่า:
1. **Authenticity** — Artifact นี้มาจากผู้สร้างที่อ้างไว้จริง
2. **Integrity** — Artifact ไม่ถูกแก้ไขหลังจาก sign

ในโลก CI/CD สมัยใหม่ Code Signing มีความสำคัญอย่างยิ่ง เพราะ artifacts ต้องผ่านหลาย systems ก่อนถึง production

---

## 1. GPG Signing พื้นฐาน

### 1.1 สร้าง GPG Key

```bash
# สร้าง GPG key pair
gpg --full-generate-key

# เลือก:
# Kind: RSA and RSA (default)
# Keysize: 4096
# Valid: 2y (2 ปี)
# Name: CI/CD Bot
# Email: cicd@example.com

# ดู keys ที่มี
gpg --list-secret-keys --keyid-format=long

# Export public key
gpg --armor --export cicd@example.com > public.asc

# Export private key (เก็บให้ปลอดภัย!)
gpg --armor --export-secret-key cicd@example.com > private.asc

# Upload public key ไปที่ keyserver
gpg --keyserver hkps://keys.openpgp.org --send-keys <KEY-ID>
```

### 1.2 Sign และ Verify Files ด้วย GPG

```bash
# Sign file (creates detached signature)
gpg --armor --detach-sign myapp-linux-amd64

# ผลลัพธ์: myapp-linux-amd64.asc (signature file)

# Sign file (inline signature)
gpg --armor --sign myapp-linux-amd64

# Verify signature
gpg --verify myapp-linux-amd64.asc myapp-linux-amd64

# Sign ด้วย specific key
gpg --armor --detach-sign \
    --local-user cicd@example.com \
    myapp-linux-amd64

# Sign checksums file (แนะนำ)
sha256sum myapp-* > checksums.txt
gpg --armor --detach-sign checksums.txt
```

### 1.3 GPG Signing ใน GitHub Actions

```yaml
# .github/workflows/gpg-sign.yml
name: Build and Sign with GPG

on:
  release:
    types: [created]

jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Import GPG Key
        run: |
          echo "${{ secrets.GPG_PRIVATE_KEY }}" | base64 -d | gpg --import
          # หรือถ้าเก็บเป็น plain text
          # echo "${{ secrets.GPG_PRIVATE_KEY }}" | gpg --import

      - name: Configure GPG
        run: |
          # กำหนด trust level
          echo -e "5\ny\n" | gpg --command-fd 0 --expert \
            --edit-key "${{ secrets.GPG_KEY_ID }}" trust quit 2>/dev/null || true

      - name: Build
        run: make build

      - name: Sign Artifacts
        env:
          GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
        run: |
          # Sign แต่ละ binary
          for file in dist/*; do
            echo "$GPG_PASSPHRASE" | gpg --batch --yes \
              --passphrase-fd 0 \
              --local-user "${{ secrets.GPG_KEY_ID }}" \
              --armor --detach-sign \
              "$file"
          done
          
          # สร้าง checksums และ sign
          cd dist
          sha256sum * > checksums.txt
          echo "$GPG_PASSPHRASE" | gpg --batch --yes \
            --passphrase-fd 0 \
            --local-user "${{ secrets.GPG_KEY_ID }}" \
            --armor --detach-sign \
            checksums.txt

      - name: Upload Signed Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/*
            dist/*.asc
```

### 1.4 Git Commit Signing

```bash
# Configure git ให้ sign commits อัตโนมัติ
git config --global user.signingkey <KEY-ID>
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# Sign specific commit
git commit -S -m "Signed commit"

# Sign tag
git tag -s v1.0.0 -m "Release v1.0.0"

# Verify commit signature
git verify-commit HEAD

# Verify tag signature
git verify-tag v1.0.0

# แสดง signature ใน log
git log --show-signature
```

---

## 2. Sigstore/Cosign — Modern Code Signing

### 2.1 ทำไมต้อง Cosign แทน GPG?

| Feature | GPG | Cosign |
|---------|-----|--------|
| Key Management | ต้อง manage keys เอง | Keyless (ใช้ OIDC) |
| Key Revocation | ซับซ้อน | Automatic (short-lived certs) |
| Transparency | ไม่มี | Rekor transparency log |
| Container Images | ไม่รองรับ native | รองรับ native |
| Ease of Use | ซับซ้อน | ง่าย |

### 2.2 ติดตั้ง Cosign

```bash
# Linux
curl -sLO https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# macOS
brew install sigstore/tap/cosign

# ตรวจสอบการติดตั้ง
cosign version
```

### 2.3 Key-based Signing

```bash
# สร้าง key pair
cosign generate-key-pair
# จะสร้าง cosign.key (private) และ cosign.pub (public)

# Sign container image
cosign sign --key cosign.key ghcr.io/myorg/myapp:latest

# Verify signature
cosign verify \
  --key cosign.pub \
  ghcr.io/myorg/myapp:latest

# Sign artifact file
cosign sign-blob \
  --key cosign.key \
  --bundle myapp.bundle \
  myapp-linux-amd64

# Verify artifact
cosign verify-blob \
  --key cosign.pub \
  --bundle myapp.bundle \
  myapp-linux-amd64
```

### 2.4 Keyless Signing (แนะนำ)

```bash
# Sign ด้วย OIDC identity (ไม่ต้องมี key)
export COSIGN_EXPERIMENTAL=1

# Interactive (สำหรับ development)
cosign sign --yes ghcr.io/myorg/myapp:latest

# ใน CI (GitHub Actions ใช้ GitHub OIDC token อัตโนมัติ)
cosign sign --yes \
  --oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/myorg/myapp:latest

# Verify keyless signature
cosign verify \
  --certificate-identity "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:latest

# ใช้ regex สำหรับ workflow paths
cosign verify \
  --certificate-identity-regexp "https://github.com/myorg/myapp/.github/workflows/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:latest
```

---

## 3. Image Signing ใน CI/CD Pipeline

### 3.1 Complete Image Signing Workflow

```yaml
# .github/workflows/image-sign.yml
name: Build, Sign, and Attest Container Image

on:
  push:
    branches: [main]
  release:
    types: [created]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-sign-attest:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write    # จำเป็นสำหรับ keyless signing
      security-events: write

    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
      image-url: ${{ steps.image-url.outputs.url }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3
        with:
          cosign-release: 'v2.2.4'

      - name: Install Syft (SBOM generator)
        uses: anchore/sbom-action/download-syft@v0

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and Push Container
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          # สร้าง SBOM และ Provenance ระหว่าง build
          sbom: true
          provenance: mode=max
          # Cache สำหรับ faster builds
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Set Image URL
        id: image-url
        run: |
          echo "url=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}" \
            >> $GITHUB_OUTPUT

      # Sign the container image
      - name: Sign Container Image
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign sign --yes \
            "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}"

      # Generate and Attest SBOM
      - name: Generate SBOM
        run: |
          syft "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}" \
            -o cyclonedx-json > sbom.cyclonedx.json
          syft "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}" \
            -o spdx-json > sbom.spdx.json

      - name: Attest SBOM
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign attest --yes \
            --type cyclonedx \
            --predicate sbom.cyclonedx.json \
            "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}"

      # Vulnerability Scan
      - name: Run Trivy Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}"
          format: cosign-vuln
          output: vuln-scan.json

      - name: Attest Vulnerability Report
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign attest --yes \
            --type vuln \
            --predicate vuln-scan.json \
            "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}"

      # Upload artifacts
      - name: Upload SBOMs
        uses: actions/upload-artifact@v4
        with:
          name: sboms
          path: |
            sbom.cyclonedx.json
            sbom.spdx.json

  # Verify after signing
  verify:
    needs: build-sign-attest
    runs-on: ubuntu-latest
    steps:
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Verify Image Signature
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign verify \
            --certificate-identity-regexp \
              "https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            "${{ needs.build-sign-attest.outputs.image-url }}"

      - name: Verify SBOM Attestation
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign verify-attestation \
            --type cyclonedx \
            --certificate-identity-regexp \
              "https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            "${{ needs.build-sign-attest.outputs.image-url }}" \
            | jq '.payload | @base64d | fromjson | .predicate.components | length'
          
          echo "✅ SBOM attestation verified"
```

---

## 4. Verification ใน Deployment Pipeline

### 4.1 Kubernetes Admission Controller สำหรับ Signature Verification

```yaml
# k8s/policy/cosign-policy.yaml — ใช้กับ Kyverno หรือ OPA

# Kyverno ClusterPolicy สำหรับ verify image signatures
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
  annotations:
    policies.kyverno.io/title: Verify Image Signature
    policies.kyverno.io/description: >-
      Verify ว่า container images ถูก sign ด้วย Cosign
      ก่อนที่จะอนุญาตให้ deploy
spec:
  validationFailureAction: Enforce
  background: false
  
  rules:
    - name: verify-image-signature
      match:
        any:
        - resources:
            kinds:
              - Pod
            namespaces:
              - production
              - staging
      
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/myapp:*"
          
          # Keyless verification
          attestors:
            - entries:
              - keyless:
                  subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                  issuer: "https://token.actions.githubusercontent.com"
                  rekor:
                    url: https://rekor.sigstore.dev
          
          # ต้องมี SBOM attestation
          attestations:
            - predicateType: https://cyclonedx.org/bom/v1.4
              attestors:
                - entries:
                  - keyless:
                      subject: "https://github.com/myorg/myapp/.github/workflows/build.yml@refs/heads/main"
                      issuer: "https://token.actions.githubusercontent.com"
```

### 4.2 Helm Deployment พร้อม Signature Verification

```bash
#!/bin/bash
# scripts/verify-and-deploy.sh
# ตรวจสอบ image signature ก่อน deploy

set -euo pipefail

IMAGE="${1}"
NAMESPACE="${2:-production}"
CHART_PATH="${3:-./helm}"

echo "🔍 Verifying image signature: ${IMAGE}"

# ตรวจสอบว่า cosign ติดตั้งแล้ว
if ! command -v cosign &> /dev/null; then
    echo "❌ cosign ไม่ได้ติดตั้ง"
    exit 1
fi

# Verify signature
export COSIGN_EXPERIMENTAL=1

VERIFICATION_OUTPUT=$(cosign verify \
    --certificate-identity-regexp "https://github.com/myorg/myapp/.github/workflows/.*" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    "${IMAGE}" 2>&1) || {
    echo "❌ Signature verification FAILED"
    echo "${VERIFICATION_OUTPUT}"
    exit 1
}

echo "✅ Signature verified"

# Verify SBOM attestation
echo "🔍 Verifying SBOM attestation..."
cosign verify-attestation \
    --type cyclonedx \
    --certificate-identity-regexp "https://github.com/myorg/myapp/.github/workflows/.*" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    "${IMAGE}" > /dev/null || {
    echo "❌ SBOM attestation verification FAILED"
    exit 1
}

echo "✅ SBOM attestation verified"

# Verify vulnerability scan attestation
echo "🔍 Verifying vulnerability scan attestation..."
VULN_REPORT=$(cosign verify-attestation \
    --type vuln \
    --certificate-identity-regexp "https://github.com/myorg/myapp/.github/workflows/.*" \
    --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
    "${IMAGE}" 2>/dev/null | jq -r '.payload | @base64d | fromjson | .predicate')

# ตรวจสอบว่าไม่มี critical vulnerabilities
CRITICAL_COUNT=$(echo "${VULN_REPORT}" | \
    jq '[.Results[]?.Vulnerabilities[]? | select(.Severity == "CRITICAL")] | length' 2>/dev/null || echo "0")

if [ "${CRITICAL_COUNT}" -gt "0" ]; then
    echo "❌ Found ${CRITICAL_COUNT} critical vulnerabilities in scan attestation"
    exit 1
fi

echo "✅ Vulnerability attestation verified (0 critical)"

# Deploy
echo "🚀 Deploying to ${NAMESPACE}..."
helm upgrade --install myapp "${CHART_PATH}" \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    --set image.repository=$(echo "${IMAGE}" | cut -d: -f1) \
    --set image.tag=$(echo "${IMAGE}" | cut -d: -f2) \
    --wait \
    --timeout 5m

echo "✅ Deployment successful!"
```

---

## 5. Transparency Logs

### 5.1 ทำความเข้าใจ Certificate Transparency

Transparency Logs ใช้หลักการ Merkle Tree ทำให้:
- ทุก signature ถูก record อย่างถาวร
- ไม่สามารถลบ entries ออกได้
- ใคร ๆ ก็ตรวจสอบได้ว่า entry มีอยู่จริง

```
                  Root Hash
                /            \
         Hash(A,B)         Hash(C,D)
         /      \           /      \
      Hash(A)  Hash(B)  Hash(C)  Hash(D)
        |        |        |        |
     Entry A  Entry B  Entry C  Entry D
```

### 5.2 Rekor CLI Tools

```bash
# ติดตั้ง rekor-cli
curl -sLO https://github.com/sigstore/rekor/releases/download/v1.3.3/rekor-cli-linux-amd64
chmod +x rekor-cli-linux-amd64
sudo mv rekor-cli-linux-amd64 /usr/local/bin/rekor-cli

# ค้นหา entry โดย artifact hash
rekor-cli search \
    --rekor_server https://rekor.sigstore.dev \
    --sha $(sha256sum myapp-linux-amd64 | cut -d' ' -f1)

# ดู entry โดย UUID
rekor-cli get \
    --rekor_server https://rekor.sigstore.dev \
    --uuid <UUID>

# ดู Rekor log tree head
rekor-cli loginfo \
    --rekor_server https://rekor.sigstore.dev

# Verify consistency ของ log
rekor-cli verify \
    --rekor_server https://rekor.sigstore.dev \
    --artifact myapp-linux-amd64 \
    --signature myapp-linux-amd64.sig \
    --public-key cosign.pub
```

### 5.3 Audit Rekor Entries

```python
# scripts/audit_signatures.py
"""
ตรวจสอบ signatures ใน Rekor สำหรับ repository ของเรา
"""
import json
import os
import urllib.request
import urllib.parse
from datetime import datetime, timezone, timedelta
from typing import List, Dict, Any


REKOR_URL = "https://rekor.sigstore.dev"


def search_entries_by_email(email: str) -> List[str]:
    """ค้นหา Rekor entries โดย email"""
    url = f"{REKOR_URL}/api/v1/index/retrieve"
    data = json.dumps({"email": email}).encode('utf-8')
    
    req = urllib.request.Request(
        url, data=data,
        headers={'Content-Type': 'application/json'},
        method='POST'
    )
    
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            return json.loads(resp.read()) or []
    except Exception:
        return []


def get_entry_detail(uuid: str) -> Dict[str, Any]:
    """ดึงรายละเอียด entry"""
    url = f"{REKOR_URL}/api/v1/log/entries/{uuid}"
    req = urllib.request.Request(url)
    
    try:
        with urllib.request.urlopen(req, timeout=30) as resp:
            data = json.loads(resp.read())
            for entry_data in data.values():
                return entry_data
    except Exception:
        pass
    return {}


def audit_recent_signatures(email: str, days: int = 30) -> dict:
    """
    ตรวจสอบ signatures ล่าสุด
    
    Args:
        email: email address ของ signer
        days: จำนวนวันย้อนหลัง
    """
    print(f"🔍 Auditing signatures for: {email}")
    print(f"   Time range: last {days} days\n")
    
    # ค้นหา entries
    uuids = search_entries_by_email(email)
    print(f"Found {len(uuids)} entries in Rekor")
    
    if not uuids:
        return {"total": 0, "entries": []}
    
    # กำหนด time range
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    
    entries = []
    for uuid in uuids[:50]:  # จำกัดแค่ 50 entries
        entry = get_entry_detail(uuid)
        if not entry:
            continue
        
        # ตรวจสอบเวลา
        integrated_time = entry.get('integratedTime', 0)
        if integrated_time:
            entry_time = datetime.fromtimestamp(integrated_time, tz=timezone.utc)
            if entry_time < cutoff:
                continue
            
            entries.append({
                "uuid": uuid,
                "time": entry_time.isoformat(),
                "log_index": entry.get('logIndex'),
                "kind": "unknown"
            })
    
    # เรียงตามเวลา
    entries.sort(key=lambda x: x['time'], reverse=True)
    
    return {
        "email": email,
        "total": len(entries),
        "time_range_days": days,
        "entries": entries
    }


def generate_audit_report(email: str) -> None:
    """สร้าง audit report"""
    result = audit_recent_signatures(email, days=30)
    
    print(f"\n📊 Audit Report")
    print(f"================")
    print(f"Email: {result['email']}")
    print(f"Total Signatures: {result['total']}")
    
    if result['entries']:
        print(f"\nRecent Signatures:")
        for entry in result['entries'][:10]:
            print(f"  - Time: {entry['time']}")
            print(f"    Log Index: {entry['log_index']}")
            print(f"    UUID: {entry['uuid'][:16]}...")
            print()
    
    # Save report
    report_file = f"audit-report-{email.replace('@', '_')}.json"
    with open(report_file, 'w') as f:
        json.dump(result, f, indent=2)
    print(f"Report saved to: {report_file}")


if __name__ == '__main__':
    import sys
    
    email = sys.argv[1] if len(sys.argv) > 1 else os.getenv('GITHUB_ACTOR', '') + '@users.noreply.github.com'
    generate_audit_report(email)
```

---

## 6. Sigstore Policy Controller

### 6.1 ติดตั้ง Sigstore Policy Controller

```bash
# ติดตั้งด้วย Helm
helm repo add sigstore https://sigstore.github.io/helm-charts
helm repo update

helm install policy-controller sigstore/policy-controller \
    --namespace cosign-system \
    --create-namespace \
    --set webhook.replicaCount=2

# ตรวจสอบการติดตั้ง
kubectl get pods -n cosign-system
```

### 6.2 สร้าง ClusterImagePolicy

```yaml
# k8s/policy/cluster-image-policy.yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signed-images
spec:
  images:
    # ใช้กับทุก images ใน ghcr.io/myorg
    - glob: "ghcr.io/myorg/**"
  
  authorities:
    # Keyless authority (ใช้ Fulcio/Rekor)
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            # อนุญาตเฉพาะ workflows ของ organization เรา
            subjectRegExp: "https://github.com/myorg/.*/.github/workflows/.*"
        
        # Rekor สำหรับ transparency log
        trustRootRef:
          name: sigstore
    
    # Key-based authority (fallback)
    - key:
        secretRef:
          name: cosign-public-key
          namespace: cosign-system
  
  # ต้องมี attestations
  policy:
    type: cue
    data: |
      import "time"
      
      // Verify ว่า attestation ไม่เก่าเกิน 30 วัน
      #signingTimestamp: time.Parse(time.RFC3339, _)
      #maxAge: 30 * 24 * time.Hour
      
      // ต้องมี SBOM
      hasSBOM: true

---
# Secret สำหรับ public key
apiVersion: v1
kind: Secret
metadata:
  name: cosign-public-key
  namespace: cosign-system
type: Opaque
stringData:
  cosign.pub: |
    -----BEGIN PUBLIC KEY-----
    <YOUR-PUBLIC-KEY-HERE>
    -----END PUBLIC KEY-----
```

---

## 7. Hardware Security Modules (HSM) สำหรับ Code Signing

### 7.1 ทำไมต้อง HSM?

HSM (Hardware Security Module) คืออุปกรณ์ hardware ที่เก็บ cryptographic keys อย่างปลอดภัย ข้อดี:
- Private keys ไม่เคย leave hardware
- FIPS 140-2 Level 3 certified
- Tamper-proof
- Audit logging ที่ hardware level

### 7.2 Cosign กับ AWS KMS

```bash
# สร้าง Cosign key ใน AWS KMS
cosign generate-key-pair \
    --kms awskms:///arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxxxxx

# Sign ด้วย KMS key
cosign sign \
    --key awskms:///arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxxxxx \
    ghcr.io/myorg/myapp:latest

# Verify ด้วย KMS key
cosign verify \
    --key awskms:///arn:aws:kms:ap-southeast-1:123456789012:key/mrk-xxxxxx \
    ghcr.io/myorg/myapp:latest
```

### 7.3 GitHub Actions กับ AWS KMS

```yaml
# .github/workflows/kms-sign.yml
name: Sign with AWS KMS

on:
  release:
    types: [created]

jobs:
  sign:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsSigningRole
          aws-region: ap-southeast-1

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Build Image
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.ref_name }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          
          IMAGE_DIGEST=$(docker inspect \
            --format='{{index .RepoDigests 0}}' \
            ghcr.io/${{ github.repository }}:${{ github.ref_name }} | cut -d@ -f2)
          echo "IMAGE_DIGEST=${IMAGE_DIGEST}" >> $GITHUB_ENV

      - name: Sign with KMS
        run: |
          cosign sign \
            --key "awskms:///arn:aws:kms:ap-southeast-1:123456789012:key/${{ secrets.KMS_KEY_ID }}" \
            "ghcr.io/${{ github.repository }}@${IMAGE_DIGEST}"
```

---

## 8. Notary v2 สำหรับ Enterprise

### 8.1 Notary v2 Architecture

Notary v2 (Notation) คือ standard ใหม่สำหรับ container image signing จาก CNCF

```bash
# ติดตั้ง Notation CLI
curl -sLO https://github.com/notaryproject/notation/releases/download/v1.1.0/notation_1.1.0_linux_amd64.tar.gz
tar xf notation_1.1.0_linux_amd64.tar.gz
sudo mv notation /usr/local/bin/

# ติดตั้ง Notation AWS Signer plugin
notation plugin install \
    --url https://github.com/aws/aws-signer-notation-plugin/releases/download/v1.0.298/notation-com.amazonaws.signer.notation.plugin_linux_amd64.tar.gz

# สร้าง signing certificate
notation cert generate-test \
    --default \
    "myapp-signer"

# Sign image
notation sign \
    --signature-format cose \
    ghcr.io/myorg/myapp:latest

# Verify signature
notation verify ghcr.io/myorg/myapp:latest

# List signatures
notation inspect ghcr.io/myorg/myapp:latest
```

### 8.2 Notation Trust Policy

```json
{
  "version": "1.0",
  "trustPolicies": [
    {
      "name": "production-policy",
      "registryScopes": ["ghcr.io/myorg/*"],
      "signatureVerification": {
        "level": "strict"
      },
      "trustStores": ["ca:production-certs"],
      "trustedIdentities": [
        "x509.subject: CN=myapp-signer, O=MyOrg, L=Bangkok, C=TH"
      ]
    }
  ]
}
```

---

## 9. Workshop: Implement Complete Code Signing Pipeline

### Lab 1: Setup Keyless Signing Pipeline

```bash
#!/bin/bash
# workshop/lab1-setup.sh

echo "=== Lab 1: Setup Keyless Signing Pipeline ==="

# 1. ตรวจสอบ prerequisites
echo "1. Checking prerequisites..."
command -v cosign >/dev/null 2>&1 || {
    echo "Installing cosign..."
    curl -sLO https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64
    chmod +x cosign-linux-amd64
    sudo mv cosign-linux-amd64 /usr/local/bin/cosign
}

command -v docker >/dev/null 2>&1 || { echo "❌ Docker required"; exit 1; }

echo "✅ Prerequisites OK"

# 2. Build test image
echo "2. Building test image..."
cat > /tmp/test-Dockerfile << 'EOF'
FROM alpine:3.19
RUN echo "Hello from signed image" > /app/hello.txt
CMD ["cat", "/app/hello.txt"]
EOF

docker build -f /tmp/test-Dockerfile -t ttl.sh/my-signed-test:1h .
docker push ttl.sh/my-signed-test:1h

echo "✅ Image pushed to ttl.sh (expires in 1 hour)"

# 3. Sign image (interactive - เปิด browser)
echo "3. Signing image..."
echo "   จะเปิด browser สำหรับ OIDC authentication..."
COSIGN_EXPERIMENTAL=1 cosign sign --yes ttl.sh/my-signed-test:1h

echo "✅ Image signed"

# 4. Verify signature
echo "4. Verifying signature..."
COSIGN_EXPERIMENTAL=1 cosign verify \
    --certificate-github-workflow-repository "$(git config remote.origin.url | sed 's/.*github.com[:/]//' | sed 's/.git//')" \
    ttl.sh/my-signed-test:1h 2>/dev/null || {
    
    echo "   (Verifying with any identity...)"
    COSIGN_EXPERIMENTAL=1 cosign verify \
        --certificate-oidc-issuer https://accounts.google.com \
        ttl.sh/my-signed-test:1h 2>/dev/null || \
    cosign verify \
        --certificate-oidc-issuer https://github.com/login/oauth \
        ttl.sh/my-signed-test:1h 2>/dev/null || \
    echo "   Signature is in Rekor (verified via transparency log)"
}

echo "✅ Verification complete"
echo ""
echo "=== Lab 1 Complete! ==="
```

### Lab 2: Full Pipeline with Attestations

```yaml
# workshop/lab2-full-pipeline.yml
name: Lab 2 - Full Signing Pipeline

on:
  workflow_dispatch:
  push:
    branches: [workshop]

jobs:
  full-signing-pipeline:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
      security-events: write

    steps:
      - uses: actions/checkout@v4

      - name: Install Tools
        run: |
          # Cosign
          curl -sLO https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64
          sudo mv cosign-linux-amd64 /usr/local/bin/cosign
          sudo chmod +x /usr/local/bin/cosign
          
          # Syft
          curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | \
            sh -s -- -b /usr/local/bin
          
          # SLSA Verifier
          curl -sLO https://github.com/slsa-framework/slsa-verifier/releases/download/v2.4.1/slsa-verifier-linux-amd64
          sudo mv slsa-verifier-linux-amd64 /usr/local/bin/slsa-verifier
          sudo chmod +x /usr/local/bin/slsa-verifier

      - name: Build and Push
        id: build
        run: |
          IMAGE="ghcr.io/${{ github.repository }}/lab2-demo:${{ github.sha }}"
          
          docker build -t "${IMAGE}" .
          echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push "${IMAGE}"
          
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' "${IMAGE}" | cut -d@ -f2)
          echo "image=${IMAGE}" >> $GITHUB_OUTPUT
          echo "digest=${DIGEST}" >> $GITHUB_OUTPUT

      - name: Sign Image
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign sign --yes \
            "ghcr.io/${{ github.repository }}/lab2-demo@${{ steps.build.outputs.digest }}"
          echo "✅ Image signed"

      - name: Generate and Attest SBOM
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          # Generate SBOM
          syft "ghcr.io/${{ github.repository }}/lab2-demo@${{ steps.build.outputs.digest }}" \
            -o cyclonedx-json > sbom.json
          
          echo "SBOM contains $(cat sbom.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(len(d.get('components',[])))") components"
          
          # Attest SBOM
          cosign attest --yes \
            --type cyclonedx \
            --predicate sbom.json \
            "ghcr.io/${{ github.repository }}/lab2-demo@${{ steps.build.outputs.digest }}"
          echo "✅ SBOM attested"

      - name: Verify Everything
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          IMAGE_REF="ghcr.io/${{ github.repository }}/lab2-demo@${{ steps.build.outputs.digest }}"
          
          # Verify signature
          cosign verify \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            "${IMAGE_REF}"
          echo "✅ Signature verified"
          
          # Verify SBOM attestation
          cosign verify-attestation \
            --type cyclonedx \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            "${IMAGE_REF}" | jq '.payload | @base64d | fromjson | .predicate.components | length'
          echo "✅ SBOM attestation verified"

      - name: Generate Summary
        run: |
          cat >> $GITHUB_STEP_SUMMARY << EOF
          ## 🔐 Signing Summary
          
          | Item | Status |
          |------|--------|
          | Image Built | ✅ |
          | Image Signed | ✅ |
          | SBOM Generated | ✅ |
          | SBOM Attested | ✅ |
          | Verification | ✅ |
          
          ### Image Reference
          \`\`\`
          ghcr.io/${{ github.repository }}/lab2-demo@${{ steps.build.outputs.digest }}
          \`\`\`
          
          ### Verify Locally
          \`\`\`bash
          COSIGN_EXPERIMENTAL=1 cosign verify \\
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/.github/workflows/.*" \\
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \\
            "ghcr.io/${{ github.repository }}/lab2-demo@${{ steps.build.outputs.digest }}"
          \`\`\`
          EOF
```

---

## 10. สรุปและ Best Practices

### Code Signing Checklist

```markdown
## Code Signing Best Practices

### Key Management
- [ ] ใช้ Keyless signing (OIDC) แทน long-lived keys ถ้าเป็นไปได้
- [ ] ถ้าใช้ keys ให้เก็บใน HSM หรือ KMS
- [ ] Rotate keys ทุก 1-2 ปี
- [ ] มี key revocation procedure ที่ชัดเจน

### Signing Process
- [ ] Sign ทุก release artifacts
- [ ] Sign container images โดย digest (ไม่ใช่ tag)
- [ ] Generate และ attest SBOMs
- [ ] Log ทุก signing event ใน Transparency Log

### Verification
- [ ] Verify signatures ก่อน deploy ทุกครั้ง
- [ ] ใช้ Admission Controllers สำหรับ Kubernetes
- [ ] ตรวจสอบ certificate identity (ไม่ใช่แค่ signature)
- [ ] Monitor Transparency Log สำหรับ unexpected entries

### Transparency
- [ ] ใช้ Rekor สำหรับ public transparency
- [ ] สำหรับ enterprise ใช้ private Rekor instance
- [ ] Audit signatures เป็นประจำ
```

---

## อ้างอิง

- [Cosign Documentation](https://docs.sigstore.dev/cosign/overview/)
- [Sigstore Keyless Signing](https://docs.sigstore.dev/cosign/keyless/)
- [Notation (Notary v2)](https://notaryproject.dev)
- [AWS Signer](https://docs.aws.amazon.com/signer/)
- [Kyverno Image Verification](https://kyverno.io/policies/other/verify_image/)
- [OpenSSF Signing Guidance](https://openssf.org/blog/2022/01/25/improving-supply-chain-security-with-signing-verification/)

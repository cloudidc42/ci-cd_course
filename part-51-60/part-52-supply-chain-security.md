# Part 52: Supply Chain Security (SLSA)

## บทนำ

Supply Chain Security คือการปกป้องทุกขั้นตอนในกระบวนการพัฒนาซอฟต์แวร์ ตั้งแต่ source code ไปจนถึง production deployment การโจมตี SolarWinds ในปี 2020 และ Codecov ในปี 2021 ทำให้โลก IT ตระหนักถึงความสำคัญของการรักษาความปลอดภัยในทุกจุดของ supply chain

บทนี้จะครอบคลุม:
- SLSA Framework และ Levels
- Software Bill of Materials (SBOM)
- Sigstore และ Cosign
- in-toto Attestations
- SLSA GitHub Actions Generator
- Workshop ปฏิบัติจริง

---

## 1. SLSA Framework

### 1.1 SLSA คืออะไร?

**SLSA** (Supply chain Levels for Software Artifacts) อ่านว่า "salsa" เป็น framework ที่พัฒนาโดย Google เพื่อกำหนดมาตรฐานความปลอดภัยของ Software Supply Chain

```
┌─────────────────────────────────────────────────────────────┐
│                    SLSA Levels                              │
│                                                             │
│  Level 4 ──────── Two-party review, Hermetic builds        │
│     │                                                       │
│  Level 3 ──────── Hardened build environment               │
│     │                                                       │
│  Level 2 ──────── Hosted build, Source version control     │
│     │                                                       │
│  Level 1 ──────── Build process documented                 │
│     │                                                       │
│  Level 0 ──────── No guarantees                            │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 SLSA Requirements แต่ละ Level

#### Level 1 — Build Documented
- Build process เป็น script (ไม่ใช่ manual)
- มี Provenance (metadata เกี่ยวกับ build)
- **ป้องกัน**: Accidental build errors

#### Level 2 — Build Service
- ใช้ Version Control
- Build ทำโดย Hosted Build Service (เช่น GitHub Actions)
- Provenance ถูก generate โดย Build Service
- **ป้องกัน**: การแก้ไข source code หลัง build

#### Level 3 — Hardened Builds
- Build environment ถูก isolate
- Source code integrity verified
- Provenance ไม่สามารถ forge ได้
- **ป้องกัน**: Insider threats, Compromised build process

#### Level 4 — Two-Party Review (ยังพัฒนาอยู่)
- All changes require review
- Hermetic builds (fully reproducible)
- **ป้องกัน**: Insider attacks, Compromised maintainer account

### 1.3 SLSA Tracks

SLSA v1.0 แบ่งเป็น tracks:
- **Build Track** — ความปลอดภัยของ build process
- **Source Track** (Draft) — ความปลอดภัยของ source code
- **Package Track** (Planned) — ความปลอดภัยของ packages ที่ publish

---

## 2. SLSA Provenance

### 2.1 Provenance คืออะไร?

Provenance คือ metadata ที่บอกว่า artifact ถูกสร้างอย่างไร ประกอบด้วย:

```json
{
  "_type": "https://in-toto.io/Statement/v0.1",
  "predicateType": "https://slsa.dev/provenance/v1",
  "subject": [
    {
      "name": "myapp",
      "digest": {
        "sha256": "abc123def456..."
      }
    }
  ],
  "predicate": {
    "buildDefinition": {
      "buildType": "https://slsa-framework.github.io/github-actions-buildtypes/workflow/v1",
      "externalParameters": {
        "workflow": {
          "ref": "refs/heads/main",
          "repository": "https://github.com/myorg/myapp",
          "path": ".github/workflows/build.yml"
        }
      },
      "resolvedDependencies": [
        {
          "uri": "git+https://github.com/myorg/myapp@refs/heads/main",
          "digest": {
            "gitCommit": "abc123def456789..."
          }
        }
      ]
    },
    "runDetails": {
      "builder": {
        "id": "https://github.com/actions/runner"
      },
      "metadata": {
        "invocationId": "https://github.com/myorg/myapp/actions/runs/123456",
        "startedOn": "2024-01-15T10:00:00Z",
        "finishedOn": "2024-01-15T10:15:00Z"
      }
    }
  }
}
```

### 2.2 สร้าง Provenance ด้วย SLSA GitHub Actions Generator

```yaml
# .github/workflows/slsa-build.yml
name: Build with SLSA Provenance

on:
  push:
    tags:
      - 'v*'

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    outputs:
      # ส่ง digest ออกไปให้ provenance generator
      hashes: ${{ steps.hash.outputs.hashes }}

    steps:
      - uses: actions/checkout@v4

      - name: Build Binary
        run: |
          make build
          # สร้าง binary list
          ls -la dist/

      - name: Generate SHA256 Hashes
        id: hash
        run: |
          cd dist
          # สร้าง hash ของทุก binary
          HASHES=$(sha256sum * | base64 -w0)
          echo "hashes=$HASHES" >> $GITHUB_OUTPUT

      - name: Upload Binaries
        uses: actions/upload-artifact@v4
        with:
          name: binaries
          path: dist/

  # SLSA Provenance Generation
  provenance:
    needs: build
    permissions:
      actions: read
      id-token: write
      contents: write

    # ใช้ SLSA GitHub Actions Generator
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: "${{ needs.build.outputs.hashes }}"
      upload-assets: true  # อัพโหลดไปที่ GitHub Release
```

### 2.3 สร้าง Provenance สำหรับ Container Images

```yaml
# .github/workflows/slsa-container.yml
name: Build Container with SLSA

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write

    outputs:
      image: ${{ steps.image.outputs.image }}
      digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set image name
        id: image
        run: |
          echo "image=ghcr.io/${{ github.repository_owner }}/myapp" >> $GITHUB_OUTPUT

      - name: Build and Push Container
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ steps.image.outputs.image }}:latest
            ${{ steps.image.outputs.image }}:${{ github.sha }}
          # เก็บ provenance metadata
          provenance: true
          sbom: true

  # Container Provenance
  provenance:
    needs: build
    permissions:
      actions: read
      id-token: write
      packages: write

    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.0.0
    with:
      image: ${{ needs.build.outputs.image }}
      digest: ${{ needs.build.outputs.digest }}
      registry-username: ${{ github.actor }}
    secrets:
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

---

## 3. Software Bill of Materials (SBOM)

### 3.1 SBOM คืออะไร?

SBOM (Software Bill of Materials) คือรายการของส่วนประกอบทั้งหมดใน software เหมือน "ฉลากส่วนผสมอาหาร" ของซอฟต์แวร์

**รูปแบบ SBOM ที่นิยม**:
- **SPDX** (Software Package Data Exchange) — Standard จาก Linux Foundation
- **CycloneDX** — Format จาก OWASP
- **SWID** (Software Identification Tags) — Standard จาก ISO

### 3.2 สร้าง SBOM ด้วย Syft

```yaml
# .github/workflows/sbom.yml
name: Generate SBOM

on:
  push:
    branches: [main]
  release:
    types: [created]

jobs:
  generate-sbom:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Install Syft
        uses: anchore/sbom-action/download-syft@v0

      - name: Build Application
        run: |
          docker build -t myapp:${{ github.sha }} .

      # สร้าง SBOM สำหรับ Container Image
      - name: Generate Container SBOM (SPDX)
        uses: anchore/sbom-action@v0
        with:
          image: myapp:${{ github.sha }}
          format: spdx-json
          output-file: sbom-container.spdx.json

      # สร้าง SBOM สำหรับ source code
      - name: Generate Source SBOM (CycloneDX)
        uses: anchore/sbom-action@v0
        with:
          path: .
          format: cyclonedx-json
          output-file: sbom-source.cyclonedx.json

      # ตรวจสอบ vulnerabilities จาก SBOM
      - name: Scan SBOM for Vulnerabilities
        uses: anchore/scan-action@v3
        with:
          sbom: sbom-container.spdx.json
          fail-build: true
          severity-cutoff: high

      # Sign SBOM
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign SBOM
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cosign sign-blob --yes \
            --bundle sbom-container.bundle \
            sbom-container.spdx.json

      # Attach SBOM to OCI image
      - name: Attach SBOM to Image
        run: |
          cosign attach sbom \
            --sbom sbom-container.spdx.json \
            --type spdx \
            myapp:${{ github.sha }}

      - name: Upload SBOM Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: sbom-files
          path: |
            sbom-container.spdx.json
            sbom-container.bundle
            sbom-source.cyclonedx.json
```

### 3.3 Parse และ Analyze SBOM

```python
# scripts/analyze_sbom.py
"""
วิเคราะห์ SBOM เพื่อ compliance และ security reporting
"""
import json
import sys
from dataclasses import dataclass
from typing import List, Optional
from pathlib import Path


@dataclass
class Component:
    """ส่วนประกอบใน SBOM"""
    name: str
    version: str
    purl: Optional[str]
    license: Optional[str]
    supplier: Optional[str]


def parse_spdx_sbom(sbom_file: str) -> List[Component]:
    """Parse SPDX JSON SBOM"""
    with open(sbom_file) as f:
        sbom = json.load(f)
    
    components = []
    for package in sbom.get('packages', []):
        # ข้ามตัวเอง (root package)
        if package.get('SPDXID') == 'SPDXRef-DOCUMENT':
            continue
        
        # หา PURL
        purl = None
        for ref in package.get('externalRefs', []):
            if ref.get('referenceType') == 'purl':
                purl = ref.get('referenceLocator')
                break
        
        # หา License
        license_str = package.get('licenseConcluded', 'NOASSERTION')
        if license_str == 'NOASSERTION':
            license_str = package.get('licenseDeclared', 'Unknown')
        
        components.append(Component(
            name=package.get('name', 'Unknown'),
            version=package.get('versionInfo', 'Unknown'),
            purl=purl,
            license=license_str,
            supplier=package.get('supplier', 'Unknown')
        ))
    
    return components


def check_license_compliance(components: List[Component]) -> List[str]:
    """ตรวจสอบ license compliance"""
    # Licenses ที่ไม่อนุญาตใน commercial product
    prohibited_licenses = {
        'GPL-2.0', 'GPL-3.0', 'AGPL-3.0', 
        'LGPL-2.0', 'LGPL-2.1', 'LGPL-3.0'
    }
    
    # Licenses ที่ต้องระวัง
    caution_licenses = {
        'MPL-2.0', 'CDDL-1.0', 'EPL-1.0', 'EPL-2.0'
    }
    
    violations = []
    warnings = []
    
    for comp in components:
        if not comp.license or comp.license in ('Unknown', 'NOASSERTION'):
            warnings.append(f"⚠️  {comp.name}@{comp.version}: License ไม่ระบุ")
            continue
        
        # Check prohibited
        for lic in prohibited_licenses:
            if lic in comp.license:
                violations.append(
                    f"❌ {comp.name}@{comp.version}: {comp.license} (ห้ามใช้)"
                )
                break
        
        # Check caution
        for lic in caution_licenses:
            if lic in comp.license:
                warnings.append(
                    f"⚠️  {comp.name}@{comp.version}: {comp.license} (ต้องตรวจสอบ)"
                )
                break
    
    return violations + warnings


def generate_report(sbom_file: str) -> dict:
    """สร้าง SBOM Analysis Report"""
    components = parse_spdx_sbom(sbom_file)
    issues = check_license_compliance(components)
    
    # สถิติ
    license_counts = {}
    for comp in components:
        lic = comp.license or 'Unknown'
        license_counts[lic] = license_counts.get(lic, 0) + 1
    
    report = {
        "summary": {
            "total_components": len(components),
            "issues_found": len(issues),
            "top_licenses": sorted(license_counts.items(), key=lambda x: x[1], reverse=True)[:5]
        },
        "issues": issues,
        "components": [
            {
                "name": c.name,
                "version": c.version,
                "license": c.license,
                "purl": c.purl
            }
            for c in components
        ]
    }
    
    return report


if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python analyze_sbom.py <sbom.spdx.json>")
        sys.exit(1)
    
    report = generate_report(sys.argv[1])
    
    print(f"\n📊 SBOM Analysis Report")
    print(f"========================")
    print(f"Total Components: {report['summary']['total_components']}")
    print(f"Issues Found: {report['summary']['issues_found']}")
    print(f"\nTop Licenses:")
    for lic, count in report['summary']['top_licenses']:
        print(f"  {lic}: {count} packages")
    
    if report['issues']:
        print(f"\n⚠️  Issues:")
        for issue in report['issues']:
            print(f"  {issue}")
        
        # Exit ด้วย error ถ้ามี violations
        if any('❌' in issue for issue in report['issues']):
            print("\n❌ License compliance check FAILED")
            sys.exit(1)
    else:
        print("\n✅ License compliance check PASSED")
    
    # เขียน report ออกไป
    with open('sbom-report.json', 'w') as f:
        json.dump(report, f, indent=2)
    print("\nReport saved to sbom-report.json")
```

---

## 4. Sigstore และ Cosign

### 4.1 Sigstore คืออะไร?

Sigstore คือ open-source project สำหรับ software signing ที่ทำให้การ sign และ verify artifacts ง่ายขึ้น ประกอบด้วย:

- **Cosign** — เครื่องมือ sign container images และ artifacts
- **Fulcio** — Certificate Authority สำหรับ code signing
- **Rekor** — Transparency log สำหรับ audit trail

```
┌─────────────────────────────────────────────────────────────┐
│                    Sigstore Architecture                    │
│                                                             │
│  Developer ──sign──→ Cosign ──→ Fulcio (CA)                 │
│                         │          │                        │
│                         │          └──→ OIDC Provider       │
│                         │               (GitHub/Google)     │
│                         │                                   │
│                         └──────→ Rekor (Transparency Log)  │
│                                        │                    │
│  Verifier ──verify──→ Cosign ──→ Rekor │                    │
│                          └─────────────┘                    │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 Keyless Signing ด้วย Cosign

```bash
# ติดตั้ง Cosign
curl -sLO https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# Keyless signing (ใช้ OIDC identity)
# ต้องมี OIDC token จาก provider (GitHub Actions, Google Cloud, etc.)
export COSIGN_EXPERIMENTAL=1

# Sign container image
cosign sign --yes ghcr.io/myorg/myapp:latest

# Sign artifact file
cosign sign-blob --yes \
  --bundle myapp.bundle \
  myapp-linux-amd64

# Verify signature
cosign verify \
  --certificate-identity-regexp "https://github.com/myorg/myapp/.github/workflows/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/myapp:latest

# Verify signed blob
cosign verify-blob \
  --certificate-identity-regexp "https://github.com/myorg/myapp/.github/workflows/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  --bundle myapp.bundle \
  myapp-linux-amd64
```

### 4.3 Integration ใน GitHub Actions

```yaml
# .github/workflows/sign-and-verify.yml
name: Sign and Verify Artifacts

on:
  release:
    types: [created]

jobs:
  sign:
    runs-on: ubuntu-latest
    permissions:
      contents: write     # เพื่อ upload to release
      id-token: write     # จำเป็นสำหรับ keyless signing
      packages: write     # เพื่อ push to registry

    steps:
      - uses: actions/checkout@v4

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3
        with:
          cosign-release: 'v2.2.4'

      - name: Build Binaries
        run: |
          make build-all
          # สร้าง checksum file
          cd dist/
          sha256sum * > checksums.txt

      - name: Sign Binaries
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          for file in dist/*; do
            if [ -f "$file" ] && [ "${file##*.}" != "txt" ]; then
              echo "Signing $file..."
              cosign sign-blob --yes \
                --bundle "${file}.bundle" \
                "$file"
            fi
          done
          
          # Sign checksums file
          cosign sign-blob --yes \
            --bundle dist/checksums.txt.bundle \
            dist/checksums.txt

      - name: Build and Sign Container Image
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.ref_name }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          
          # Get image digest
          IMAGE_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' \
            ghcr.io/${{ github.repository }}:${{ github.ref_name }} | cut -d@ -f2)
          
          # Sign by digest (immutable)
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${IMAGE_DIGEST}

      - name: Upload Signed Artifacts
        uses: softprops/action-gh-release@v1
        with:
          files: |
            dist/*
            dist/*.bundle

  verify:
    runs-on: ubuntu-latest
    needs: sign
    steps:
      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Download Release Artifacts
        run: |
          # Download latest release artifacts
          gh release download ${{ github.ref_name }} \
            --dir ./download
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Verify Signatures
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          cd download
          for bundle in *.bundle; do
            artifact="${bundle%.bundle}"
            if [ -f "$artifact" ]; then
              echo "Verifying $artifact..."
              cosign verify-blob \
                --certificate-identity-regexp "https://github.com/${{ github.repository }}/.github/workflows/.*" \
                --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
                --bundle "$bundle" \
                "$artifact"
              echo "✅ $artifact verified successfully"
            fi
          done
```

---

## 5. in-toto Attestations

### 5.1 in-toto Framework

in-toto คือ framework ที่ ensure integrity ของ software supply chain โดยบันทึกและ verify ทุก step ใน pipeline

```
Source Code → Build → Test → Package → Sign → Deploy
     ↓           ↓       ↓       ↓        ↓       ↓
  Link        Link    Link    Link     Link    Link
  Metadata    Metadata        Metadata
     ↓                                         ↓
  Layout ──────────────────────────────→ Verify
```

### 5.2 สร้าง Attestations ด้วย cosign attest

```yaml
# .github/workflows/attestation.yml
name: Generate Attestations

on:
  push:
    branches: [main]

jobs:
  attest:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Install Tools
        run: |
          # Install cosign
          curl -sLO https://github.com/sigstore/cosign/releases/download/v2.2.4/cosign-linux-amd64
          chmod +x cosign-linux-amd64
          sudo mv cosign-linux-amd64 /usr/local/bin/cosign
          
          # Install syft
          curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

      - name: Build and Push Image
        id: build
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
          
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' \
            ghcr.io/${{ github.repository }}:${{ github.sha }} | cut -d@ -f2)
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT

      - name: Generate SBOM Attestation
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          # Generate SBOM
          syft ghcr.io/${{ github.repository }}:${{ github.sha }} \
            -o cyclonedx-json > sbom.json
          
          # Create attestation
          cosign attest --yes \
            --type cyclonedx \
            --predicate sbom.json \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

      - name: Generate Vulnerability Attestation
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          # Run vulnerability scan
          trivy image --format cosign-vuln \
            --output vuln-report.json \
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          
          # Attest vulnerability report
          cosign attest --yes \
            --type vuln \
            --predicate vuln-report.json \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

      - name: Create Custom Attestation
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          # สร้าง custom attestation สำหรับ deployment metadata
          cat > deployment-attestation.json << EOF
          {
            "buildDate": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "commitSha": "${{ github.sha }}",
            "buildTriggeredBy": "${{ github.actor }}",
            "repositoryUrl": "https://github.com/${{ github.repository }}",
            "workflowRunUrl": "https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}",
            "tests": {
              "unitTests": "passed",
              "integrationTests": "passed",
              "securityScan": "passed"
            }
          }
          EOF
          
          cosign attest --yes \
            --type https://example.com/DeploymentMetadata/v1 \
            --predicate deployment-attestation.json \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

      - name: Verify Attestations
        env:
          COSIGN_EXPERIMENTAL: "1"
        run: |
          # Verify SBOM attestation
          cosign verify-attestation \
            --type cyclonedx \
            --certificate-identity-regexp "https://github.com/${{ github.repository }}/.github/workflows/.*" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }} \
            | jq '.payload | @base64d | fromjson | .predicate.components | length'
```

---

## 6. Rekor — Transparency Log

### 6.1 ทำความเข้าใจ Rekor

Rekor คือ append-only transparency log สำหรับบันทึก software artifacts metadata คล้ายกับ Certificate Transparency สำหรับ TLS certificates

```bash
# ค้นหา entries ใน Rekor
rekor-cli search --email user@example.com
rekor-cli search --sha <artifact-sha256>

# ดู entry ที่ specific
rekor-cli get --uuid <uuid>

# ตรวจสอบ entry
rekor-cli verify \
  --artifact myapp-binary \
  --signature myapp-binary.sig \
  --public-key public.pem
```

### 6.2 Query Rekor API

```python
# scripts/query_rekor.py
"""
Query Rekor transparency log เพื่อตรวจสอบ artifact signatures
"""
import hashlib
import json
import base64
import urllib.request
import urllib.error
from typing import Optional, Dict, Any


REKOR_URL = "https://rekor.sigstore.dev"


def get_artifact_sha256(filepath: str) -> str:
    """คำนวณ SHA256 ของ artifact"""
    sha256_hash = hashlib.sha256()
    with open(filepath, "rb") as f:
        for byte_block in iter(lambda: f.read(4096), b""):
            sha256_hash.update(byte_block)
    return sha256_hash.hexdigest()


def search_rekor_by_hash(artifact_hash: str) -> list:
    """ค้นหา entries ใน Rekor โดย artifact hash"""
    url = f"{REKOR_URL}/api/v1/index/retrieve"
    data = json.dumps({"hash": f"sha256:{artifact_hash}"}).encode('utf-8')
    
    req = urllib.request.Request(
        url,
        data=data,
        headers={'Content-Type': 'application/json'},
        method='POST'
    )
    
    try:
        with urllib.request.urlopen(req, timeout=30) as response:
            return json.loads(response.read())
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return []
        raise


def get_rekor_entry(uuid: str) -> Optional[Dict[str, Any]]:
    """ดึง Rekor entry โดย UUID"""
    url = f"{REKOR_URL}/api/v1/log/entries/{uuid}"
    
    try:
        req = urllib.request.Request(url)
        with urllib.request.urlopen(req, timeout=30) as response:
            entries = json.loads(response.read())
            for entry_data in entries.values():
                return entry_data
    except urllib.error.HTTPError as e:
        if e.code == 404:
            return None
        raise


def verify_artifact_in_rekor(filepath: str) -> dict:
    """
    ตรวจสอบว่า artifact ถูก log ใน Rekor หรือไม่
    
    Returns:
        dict with verification result
    """
    artifact_hash = get_artifact_sha256(filepath)
    print(f"Artifact SHA256: {artifact_hash}")
    
    # ค้นหาใน Rekor
    uuids = search_rekor_by_hash(artifact_hash)
    
    if not uuids:
        return {
            "verified": False,
            "reason": "ไม่พบ artifact ใน Rekor transparency log",
            "artifact_hash": artifact_hash
        }
    
    # ดึงข้อมูล entry
    entries = []
    for uuid in uuids:
        entry = get_rekor_entry(uuid)
        if entry:
            # Decode body
            body_bytes = base64.b64decode(entry.get('body', ''))
            body = json.loads(body_bytes)
            
            # Extract signer information
            signer = "Unknown"
            spec = body.get('spec', {})
            
            if 'signature' in spec:
                sig = spec['signature']
                if 'publicKey' in sig:
                    signer = sig['publicKey'].get('content', 'Unknown')
            
            entries.append({
                "uuid": uuid,
                "logIndex": entry.get('logIndex'),
                "integratedTime": entry.get('integratedTime'),
                "kind": body.get('kind'),
                "apiVersion": body.get('apiVersion'),
            })
    
    return {
        "verified": True,
        "artifact_hash": artifact_hash,
        "rekor_entries": entries,
        "entry_count": len(entries)
    }


if __name__ == '__main__':
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python query_rekor.py <artifact-file>")
        sys.exit(1)
    
    result = verify_artifact_in_rekor(sys.argv[1])
    
    if result['verified']:
        print(f"\n✅ Artifact verified in Rekor!")
        print(f"Found {result['entry_count']} entries:")
        for entry in result['rekor_entries']:
            print(f"\n  UUID: {entry['uuid']}")
            print(f"  Log Index: {entry['logIndex']}")
            print(f"  Kind: {entry['kind']}")
    else:
        print(f"\n❌ {result['reason']}")
        sys.exit(1)
```

---

## 7. Dependency Track — SBOM Management Platform

### 7.1 ติดตั้ง Dependency Track

```yaml
# docker-compose-dependency-track.yml
version: '3.8'

services:
  dtrack-apiserver:
    image: dependencytrack/apiserver:latest
    environment:
      ALPINE_DATABASE_URL: "jdbc:postgresql://postgres:5432/dtrack"
      ALPINE_DATABASE_DRIVER: "org.postgresql.Driver"
      ALPINE_DATABASE_USERNAME: "dtrack"
      ALPINE_DATABASE_PASSWORD: "dtrack-password"
    ports:
      - "8081:8080"
    volumes:
      - dtrack-data:/data
    depends_on:
      - postgres
    restart: unless-stopped

  dtrack-frontend:
    image: dependencytrack/frontend:latest
    environment:
      API_BASE_URL: "http://localhost:8081"
    ports:
      - "8080:8080"
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: "dtrack"
      POSTGRES_PASSWORD: "dtrack-password"
      POSTGRES_DB: "dtrack"
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  dtrack-data:
  postgres-data:
```

### 7.2 Upload SBOM ไปที่ Dependency Track

```python
# scripts/upload_to_dependency_track.py
"""
Upload SBOM ไปที่ Dependency Track API
"""
import base64
import json
import os
import sys
import urllib.request
import urllib.error


class DependencyTrackClient:
    """Client สำหรับ Dependency Track API"""
    
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url.rstrip('/')
        self.api_key = api_key
    
    def _make_request(self, method: str, path: str, data=None) -> dict:
        """ทำ HTTP request ไปที่ DT API"""
        url = f"{self.base_url}/api/v1{path}"
        
        headers = {
            'X-Api-Key': self.api_key,
            'Content-Type': 'application/json'
        }
        
        body = json.dumps(data).encode('utf-8') if data else None
        
        req = urllib.request.Request(url, data=body, headers=headers, method=method)
        
        try:
            with urllib.request.urlopen(req, timeout=60) as response:
                if response.status == 200:
                    return json.loads(response.read())
                return {}
        except urllib.error.HTTPError as e:
            error_body = e.read().decode('utf-8')
            raise Exception(f"API Error {e.code}: {error_body}")
    
    def get_or_create_project(self, name: str, version: str) -> dict:
        """สร้างหรือดึง project"""
        # ลองค้นหา project ก่อน
        projects = self._make_request('GET', f'/project?name={name}')
        
        if isinstance(projects, list):
            for project in projects:
                if project.get('name') == name and project.get('version') == version:
                    return project
        
        # สร้าง project ใหม่
        return self._make_request('PUT', '/project', {
            'name': name,
            'version': version,
            'classifier': 'APPLICATION'
        })
    
    def upload_sbom(self, project_uuid: str, sbom_file: str, format: str = 'CycloneDX') -> dict:
        """Upload SBOM file ไปที่ project"""
        with open(sbom_file, 'rb') as f:
            sbom_base64 = base64.b64encode(f.read()).decode('utf-8')
        
        return self._make_request('PUT', '/bom', {
            'project': project_uuid,
            'bom': sbom_base64
        })
    
    def get_project_findings(self, project_uuid: str) -> list:
        """ดึง vulnerabilities ของ project"""
        return self._make_request('GET', f'/finding/project/{project_uuid}')
    
    def get_project_metrics(self, project_uuid: str) -> dict:
        """ดึง metrics ของ project"""
        return self._make_request('GET', f'/metrics/project/{project_uuid}/current')


def main():
    # Configuration จาก environment variables
    dt_url = os.getenv('DEPENDENCY_TRACK_URL', 'http://localhost:8080')
    dt_api_key = os.getenv('DEPENDENCY_TRACK_API_KEY', '')
    project_name = os.getenv('PROJECT_NAME', 'myapp')
    project_version = os.getenv('PROJECT_VERSION', '1.0.0')
    sbom_file = sys.argv[1] if len(sys.argv) > 1 else 'sbom.cyclonedx.json'
    
    if not dt_api_key:
        print("❌ DEPENDENCY_TRACK_API_KEY ไม่ได้ตั้งค่า")
        sys.exit(1)
    
    client = DependencyTrackClient(dt_url, dt_api_key)
    
    # สร้างหรือดึง project
    print(f"Getting/creating project: {project_name}@{project_version}")
    project = client.get_or_create_project(project_name, project_version)
    project_uuid = project['uuid']
    print(f"Project UUID: {project_uuid}")
    
    # Upload SBOM
    print(f"Uploading SBOM: {sbom_file}")
    result = client.upload_sbom(project_uuid, sbom_file)
    print(f"Upload result: {result}")
    
    # รอให้ process เสร็จ
    import time
    print("Waiting for analysis to complete...")
    time.sleep(30)
    
    # ดึง findings
    findings = client.get_project_findings(project_uuid)
    
    # สรุป
    critical = sum(1 for f in findings if f.get('vulnerability', {}).get('severity') == 'CRITICAL')
    high = sum(1 for f in findings if f.get('vulnerability', {}).get('severity') == 'HIGH')
    
    print(f"\n📊 Vulnerability Summary:")
    print(f"  Critical: {critical}")
    print(f"  High: {high}")
    print(f"  Total: {len(findings)}")
    
    # Fail ถ้ามี critical vulnerabilities
    if critical > 0:
        print(f"\n❌ Found {critical} critical vulnerabilities")
        sys.exit(1)
    
    print("\n✅ No critical vulnerabilities found")


if __name__ == '__main__':
    main()
```

---

## 8. Workshop: Implement Full Supply Chain Security

### Lab 1: SLSA Level 2 Pipeline

```yaml
# .github/workflows/slsa-level2.yml
name: SLSA Level 2 Build

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
      packages: write

    outputs:
      hashes: ${{ steps.hash.outputs.hashes }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4
        with:
          # Ensure we're checking out the tag
          ref: ${{ github.ref }}

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version-file: 'go.mod'
          cache: true

      - name: Build Binaries
        run: |
          mkdir -p dist
          
          # Build สำหรับหลาย platforms
          GOOS=linux GOARCH=amd64 go build -ldflags "-X main.version=${{ github.ref_name }}" \
            -o dist/myapp-linux-amd64 ./cmd/myapp
          GOOS=linux GOARCH=arm64 go build -ldflags "-X main.version=${{ github.ref_name }}" \
            -o dist/myapp-linux-arm64 ./cmd/myapp
          GOOS=darwin GOARCH=amd64 go build -ldflags "-X main.version=${{ github.ref_name }}" \
            -o dist/myapp-darwin-amd64 ./cmd/myapp
          GOOS=windows GOARCH=amd64 go build -ldflags "-X main.version=${{ github.ref_name }}" \
            -o dist/myapp-windows-amd64.exe ./cmd/myapp

      - name: Generate Checksums
        id: hash
        run: |
          cd dist
          sha256sum * > checksums.txt
          cat checksums.txt
          HASHES=$(cat checksums.txt | base64 -w0)
          echo "hashes=$HASHES" >> $GITHUB_OUTPUT

      - name: Build and Push Container
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}
          provenance: true
          sbom: true

      - name: Upload Release Assets
        uses: softprops/action-gh-release@v1
        with:
          files: dist/*

  # Generate SLSA Provenance
  provenance:
    needs: build
    permissions:
      actions: read
      id-token: write
      contents: write

    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: "${{ needs.build.outputs.hashes }}"
      upload-assets: true
      upload-tag-name: "${{ github.ref_name }}"

  # Verify provenance
  verify:
    needs: [build, provenance]
    runs-on: ubuntu-latest
    steps:
      - name: Install SLSA Verifier
        run: |
          curl -sLO https://github.com/slsa-framework/slsa-verifier/releases/download/v2.4.1/slsa-verifier-linux-amd64
          chmod +x slsa-verifier-linux-amd64
          sudo mv slsa-verifier-linux-amd64 /usr/local/bin/slsa-verifier

      - name: Download Release Assets
        run: |
          gh release download ${{ github.ref_name }} \
            --dir ./download
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Verify SLSA Provenance
        run: |
          cd download
          # ค้นหา provenance file
          PROVENANCE_FILE=$(ls *.intoto.jsonl 2>/dev/null | head -1)
          
          if [ -z "$PROVENANCE_FILE" ]; then
            echo "❌ ไม่พบ provenance file"
            exit 1
          fi
          
          # Verify แต่ละ binary
          for binary in myapp-linux-amd64 myapp-linux-arm64 myapp-darwin-amd64; do
            if [ -f "$binary" ]; then
              echo "Verifying $binary..."
              slsa-verifier verify-artifact \
                --provenance-path "$PROVENANCE_FILE" \
                --source-uri "github.com/${{ github.repository }}" \
                --source-tag "${{ github.ref_name }}" \
                "$binary"
              echo "✅ $binary verified"
            fi
          done
```

### Lab 2: Dependency Track Integration

```yaml
# .github/workflows/dependency-track.yml
name: SBOM and Dependency Track

on:
  push:
    branches: [main]

jobs:
  sbom-upload:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          path: .
          format: cyclonedx-json
          output-file: sbom.cyclonedx.json

      - name: Upload to Dependency Track
        env:
          DEPENDENCY_TRACK_URL: ${{ secrets.DEPENDENCY_TRACK_URL }}
          DEPENDENCY_TRACK_API_KEY: ${{ secrets.DEPENDENCY_TRACK_API_KEY }}
          PROJECT_NAME: ${{ github.repository }}
          PROJECT_VERSION: ${{ github.sha }}
        run: |
          python scripts/upload_to_dependency_track.py sbom.cyclonedx.json

      - name: Check Policy Violations
        run: |
          # ตรวจสอบว่ามี policy violations หรือไม่
          python scripts/check_dt_policy.py \
            --project "${{ github.repository }}" \
            --version "${{ github.sha }}" \
            --fail-on-critical
```

---

## 9. สรุปและ Best Practices

### Checklist Supply Chain Security

```markdown
## Supply Chain Security Checklist

### Source Code
- [ ] ใช้ branch protection rules
- [ ] กำหนด required reviewers
- [ ] Sign commits ด้วย GPG
- [ ] ใช้ dependency scanning

### Build Process
- [ ] Pin action versions ด้วย SHA
- [ ] ใช้ SLSA-compliant build generator
- [ ] สร้าง provenance ทุกครั้ง
- [ ] ไม่อนุญาต arbitrary code ใน pipeline

### Artifacts
- [ ] Sign ทุก artifact ด้วย Cosign
- [ ] สร้าง SBOM ทุก release
- [ ] Scan SBOMs สำหรับ vulnerabilities
- [ ] Store ใน OCI registry ที่ปลอดภัย

### Deployment
- [ ] Verify signatures ก่อน deploy
- [ ] ใช้ admission controllers ตรวจสอบ signatures
- [ ] Monitor transparency logs
- [ ] Maintain SBOM database
```

---

## อ้างอิง

- [SLSA Framework](https://slsa.dev)
- [Sigstore Documentation](https://docs.sigstore.dev)
- [in-toto Framework](https://in-toto.io)
- [CycloneDX Specification](https://cyclonedx.org)
- [SPDX Specification](https://spdx.dev)
- [Dependency Track](https://dependencytrack.org)
- [NIST SP 800-161r1: Cybersecurity Supply Chain Risk Management](https://doi.org/10.6028/NIST.SP.800-161r1)

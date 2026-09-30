# Part 38: Dependency Management และ Security Scanning

## บทนำ

Dependencies คือ third-party libraries และ packages ที่ project ของเรา depend upon ปัญหาด้านความปลอดภัยใน dependencies ที่เราใช้สามารถส่งผลกระทบต่อ application ของเราได้ ในปี 2021 ช่องโหว่ Log4Shell ใน Apache Log4j ทำให้ทั้งโลกต้องวิ่งแก้ปัญหากันอย่างเร่งด่วน ในบทนี้เราจะเรียนรู้การจัดการ dependencies อย่างมีประสิทธิภาพและปลอดภัย

## วัตถุประสงค์การเรียนรู้

- ใช้ npm audit, pip-audit, และ OWASP Dependency-Check
- ตั้งค่า Dependabot และ Renovate สำหรับ automated updates
- ตรวจสอบ licenses ของ dependencies
- สร้าง Software Bill of Materials (SBOM)
- จัดการ transitive dependencies
- ตั้งค่า dependency security gates ใน CI/CD

---

## 38.1 npm Audit

### Basic npm audit

```bash
# รัน audit
npm audit

# Audit แบบ JSON
npm audit --json

# Audit เฉพาะ production dependencies
npm audit --only=prod

# Audit ตาม severity level
npm audit --audit-level=critical
npm audit --audit-level=high
npm audit --audit-level=moderate

# Fix automatically (ระวัง: อาจ break changes)
npm audit fix

# Fix รวมถึง breaking changes
npm audit fix --force

# Generate report
npm audit --json > audit-report.json
```

### npm audit ใน CI Pipeline

```bash
#!/bin/bash
# npm-security-check.sh

set -euo pipefail

AUDIT_LEVEL=${AUDIT_LEVEL:-"high"}
REPORT_FILE="npm-audit-report.json"

echo "Running npm audit..."
npm audit --json > "$REPORT_FILE" 2>&1 || true

# Parse results
CRITICAL=$(cat "$REPORT_FILE" | jq '.metadata.vulnerabilities.critical // 0')
HIGH=$(cat "$REPORT_FILE" | jq '.metadata.vulnerabilities.high // 0')
MODERATE=$(cat "$REPORT_FILE" | jq '.metadata.vulnerabilities.moderate // 0')
LOW=$(cat "$REPORT_FILE" | jq '.metadata.vulnerabilities.low // 0')
TOTAL=$(cat "$REPORT_FILE" | jq '.metadata.vulnerabilities.total // 0')

echo "Vulnerability Summary:"
echo "  Critical: $CRITICAL"
echo "  High:     $HIGH"
echo "  Moderate: $MODERATE"
echo "  Low:      $LOW"
echo "  Total:    $TOTAL"

# แสดง vulnerable packages
if [ "$TOTAL" -gt "0" ]; then
    echo ""
    echo "Vulnerable Packages:"
    cat "$REPORT_FILE" | jq -r '
        .vulnerabilities | to_entries[] |
        select(.value.severity as $sev | ["high", "critical"] | contains([$sev])) |
        "  - \(.key)@\(.value.range) [\(.value.severity | ascii_upcase)]: \(.value.title)"
    '
fi

# ตรวจสอบ critical vulnerabilities
if [ "$CRITICAL" -gt "0" ]; then
    echo ""
    echo "FAIL: Found $CRITICAL critical vulnerabilities"
    exit 1
fi

if [ "$AUDIT_LEVEL" = "high" ] && [ "$HIGH" -gt "0" ]; then
    echo ""
    echo "FAIL: Found $HIGH high vulnerabilities"
    exit 1
fi

echo ""
echo "PASS: No significant vulnerabilities found"
```

### package.json Security Config

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    "audit": "npm audit --audit-level=high",
    "audit:fix": "npm audit fix",
    "audit:report": "npm audit --json > audit-report.json",
    "security": "npm run audit && npm run license-check"
  },
  "devDependencies": {
    "license-checker": "^25.0.1",
    "better-npm-audit": "^3.7.3"
  }
}
```

---

## 38.2 Python Dependency Security

### pip-audit

```bash
# ติดตั้ง
pip install pip-audit

# Scan requirements.txt
pip-audit -r requirements.txt

# Scan installed packages
pip-audit

# Output เป็น JSON
pip-audit -r requirements.txt --format json > pip-audit-report.json

# Fix vulnerabilities อัตโนมัติ
pip-audit -r requirements.txt --fix

# Skip specific vulnerabilities
pip-audit -r requirements.txt \
  --ignore-vuln PYSEC-2021-336 \
  --ignore-vuln GHSA-xxxx-xxxx-xxxx

# ใช้ specific vulnerability database
pip-audit -r requirements.txt \
  --vulnerability-service osv \
  --vulnerability-service pypi
```

### Safety (Python)

```bash
# ติดตั้ง
pip install safety

# Scan
safety check -r requirements.txt

# JSON output
safety check -r requirements.txt --json > safety-report.json

# Full output
safety check -r requirements.txt --full-report

# ใช้ API key สำหรับ full database
safety check -r requirements.txt --api-key=$SAFETY_API_KEY
```

### Python Security Automation

```python
#!/usr/bin/env python3
# check_dependencies.py

import json
import subprocess
import sys
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Vulnerability:
    package: str
    version: str
    vuln_id: str
    description: str
    severity: str
    fix_versions: List[str]
    
    @property
    def has_fix(self) -> bool:
        return len(self.fix_versions) > 0

class DependencyChecker:
    def __init__(
        self,
        requirements_file: str = "requirements.txt",
        fail_on: str = "high",
        ignore_ids: Optional[List[str]] = None
    ):
        self.requirements_file = requirements_file
        self.fail_on = fail_on
        self.ignore_ids = ignore_ids or []
    
    def run_pip_audit(self) -> dict:
        """รัน pip-audit และ return results"""
        result = subprocess.run(
            ["pip-audit", "-r", self.requirements_file, "--format", "json"],
            capture_output=True,
            text=True
        )
        
        try:
            return json.loads(result.stdout)
        except json.JSONDecodeError:
            print(f"Error parsing pip-audit output: {result.stderr}")
            return {}
    
    def parse_vulnerabilities(self, audit_data: dict) -> List[Vulnerability]:
        """Parse vulnerabilities จาก pip-audit output"""
        vulnerabilities = []
        
        for dep in audit_data.get("dependencies", []):
            for vuln in dep.get("vulns", []):
                if vuln["id"] in self.ignore_ids:
                    continue
                
                vulnerability = Vulnerability(
                    package=dep["name"],
                    version=dep["version"],
                    vuln_id=vuln["id"],
                    description=vuln.get("description", ""),
                    severity=self._get_severity(vuln),
                    fix_versions=vuln.get("fix_versions", [])
                )
                vulnerabilities.append(vulnerability)
        
        return vulnerabilities
    
    def _get_severity(self, vuln: dict) -> str:
        """ประเมิน severity จาก CVSS score"""
        aliases = vuln.get("aliases", [])
        
        # ดู CVE data
        for alias in aliases:
            if alias.startswith("CVE-"):
                # ในความเป็นจริงควรดึงข้อมูลจาก NVD API
                pass
        
        return "unknown"
    
    def should_fail(self, vulnerabilities: List[Vulnerability]) -> bool:
        """ตรวจสอบว่าควร fail หรือไม่"""
        severity_order = ["low", "moderate", "high", "critical"]
        threshold = self.fail_on
        
        if threshold not in severity_order:
            return False
        
        threshold_idx = severity_order.index(threshold)
        
        for vuln in vulnerabilities:
            if vuln.severity in severity_order:
                if severity_order.index(vuln.severity) >= threshold_idx:
                    return True
        
        return False
    
    def generate_report(self, vulnerabilities: List[Vulnerability]) -> str:
        """สร้าง report"""
        if not vulnerabilities:
            return "✅ No vulnerabilities found"
        
        lines = [
            f"🔍 Found {len(vulnerabilities)} vulnerabilities:",
            ""
        ]
        
        # Group by severity
        by_severity = {}
        for vuln in vulnerabilities:
            severity = vuln.severity.upper()
            if severity not in by_severity:
                by_severity[severity] = []
            by_severity[severity].append(vuln)
        
        # Sort by severity (critical first)
        severity_order = ["CRITICAL", "HIGH", "MODERATE", "LOW", "UNKNOWN"]
        
        for severity in severity_order:
            if severity in by_severity:
                lines.append(f"## {severity} ({len(by_severity[severity])})")
                for vuln in by_severity[severity]:
                    lines.append(f"  📦 {vuln.package}=={vuln.version}")
                    lines.append(f"     ID: {vuln.vuln_id}")
                    if vuln.description:
                        lines.append(f"     {vuln.description[:150]}...")
                    if vuln.has_fix:
                        lines.append(f"     Fix: upgrade to {', '.join(vuln.fix_versions)}")
                    lines.append("")
        
        return "\n".join(lines)
    
    def check(self) -> bool:
        """รัน check และ return True ถ้า pass"""
        print(f"Checking dependencies in {self.requirements_file}...")
        
        audit_data = self.run_pip_audit()
        vulnerabilities = self.parse_vulnerabilities(audit_data)
        
        report = self.generate_report(vulnerabilities)
        print(report)
        
        if self.should_fail(vulnerabilities):
            print(f"\n❌ FAIL: Vulnerabilities above threshold ({self.fail_on}) found")
            return False
        
        print("\n✅ PASS: Dependencies check passed")
        return True

if __name__ == "__main__":
    checker = DependencyChecker(
        requirements_file="requirements.txt",
        fail_on="high",
        ignore_ids=[]  # เพิ่ม false positive IDs ที่นี่
    )
    
    success = checker.check()
    sys.exit(0 if success else 1)
```

---

## 38.3 OWASP Dependency-Check

### การใช้งาน OWASP Dependency-Check

```bash
# Docker
docker run --rm \
  -v $(pwd):/src \
  -v $(pwd)/reports:/report \
  owasp/dependency-check \
  --scan /src \
  --format HTML \
  --format JSON \
  --out /report \
  --project "My Application" \
  --nvdApiKey $NVD_API_KEY

# Maven Plugin
# pom.xml
# <plugin>
#   <groupId>org.owasp</groupId>
#   <artifactId>dependency-check-maven</artifactId>
#   <version>8.4.0</version>
#   <configuration>
#     <failBuildOnCVSS>7</failBuildOnCVSS>
#     <format>ALL</format>
#   </configuration>
# </plugin>

# Gradle Plugin
# build.gradle
# plugins {
#   id 'org.owasp.dependencycheck' version '8.4.0'
# }
# dependencyCheck {
#   failBuildOnCVSS = 7
#   format = 'ALL'
# }
```

---

## 38.4 Snyk Integration

```bash
# ติดตั้ง Snyk CLI
npm install -g snyk

# Authenticate
snyk auth

# Test Node.js project
snyk test

# Test Python project
snyk test --file=requirements.txt

# Monitor project (เพิ่มใน dashboard)
snyk monitor

# Test Docker image
snyk container test nginx:latest

# IaC scan
snyk iac test k8s/

# Fix vulnerabilities
snyk fix
```

### Snyk Configuration

```yaml
# .snyk
version: v1.0.0

# ไม่สนใจ vulnerabilities ที่เป็น false positive
ignore:
  SNYK-JS-LODASH-567746:
    - '*':
        reason: Not applicable in our use case
        expires: 2025-01-01T00:00:00.000Z
  
  SNYK-PYTHON-REQUESTS-123456:
    - '*':
        reason: Protected by WAF
        expires: 2024-12-31T00:00:00.000Z

# ตั้งค่า patch
patch:
  SNYK-JS-MINIMATCH-3050818:
    - minimatch > minimatch:
        patched: 2024-01-01T00:00:00.000Z
```

---

## 38.5 Dependabot

### .github/dependabot.yml

```yaml
# .github/dependabot.yml
version: 2

updates:
  # npm dependencies
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Bangkok"
    
    # จำนวน PRs สูงสุดที่ออกพร้อมกัน
    open-pull-requests-limit: 10
    
    # Reviewers
    reviewers:
      - "security-team"
    
    # Assignees
    assignees:
      - "devops-lead"
    
    # Labels
    labels:
      - "dependencies"
      - "security"
    
    # Group updates เพื่อลด noise
    groups:
      # Group minor/patch updates ไว้ด้วยกัน
      minor-and-patch:
        patterns:
          - "*"
        update-types:
          - "minor"
          - "patch"
      
      # Security updates แยก group
      security-updates:
        patterns:
          - "*"
        dependency-type: "production"
    
    # Ignore ไม่อัพเดต major versions โดยอัตโนมัติ
    ignore:
      - dependency-name: "react"
        update-types: ["version-update:semver-major"]
      - dependency-name: "express"
        update-types: ["version-update:semver-major"]
    
    # Commit message settings
    commit-message:
      prefix: "chore(deps)"
      prefix-development: "chore(devdeps)"
      include: "scope"
    
    # Target branch
    target-branch: "main"
    
    # Milestone
    milestone: 1
  
  # Python pip
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "tuesday"
    open-pull-requests-limit: 5
    labels:
      - "dependencies"
      - "python"
  
  # Docker
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "docker"
      - "dependencies"
  
  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
    labels:
      - "github-actions"
      - "dependencies"
  
  # Terraform
  - package-ecosystem: "terraform"
    directory: "/infrastructure"
    schedule:
      interval: "monthly"
    labels:
      - "terraform"
      - "dependencies"
```

---

## 38.6 Renovate

### Renovate Configuration

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:base",
    ":dependencyDashboard",
    ":semanticCommits",
    "group:monorepos",
    "security:openssf-scorecard"
  ],
  
  "timezone": "Asia/Bangkok",
  "schedule": ["before 9am on Monday"],
  
  "labels": ["dependencies"],
  "reviewers": ["team:devops"],
  
  // Auto-merge สำหรับ minor/patch
  "automerge": false,
  "automergeType": "pr",
  
  "packageRules": [
    // Security updates - merge เร็ว
    {
      "matchCategories": ["security"],
      "labels": ["security", "dependencies"],
      "prPriority": 10,
      "automerge": true,
      "automergeSchedule": ["at any time"]
    },
    
    // Dev dependencies - ยืดหยุ่นกว่า
    {
      "matchDepTypes": ["devDependencies"],
      "automerge": true,
      "automergeSchedule": ["before 9am on Monday"]
    },
    
    // Group testing frameworks
    {
      "groupName": "testing frameworks",
      "matchPackagePatterns": [
        "^jest",
        "^@testing-library",
        "^playwright",
        "^cypress"
      ]
    },
    
    // Group ESLint plugins
    {
      "groupName": "eslint",
      "matchPackagePatterns": [
        "^eslint",
        "^@eslint"
      ]
    },
    
    // Major versions - ต้อง manual review
    {
      "matchUpdateTypes": ["major"],
      "labels": ["major-upgrade"],
      "automerge": false,
      "reviewers": ["team:senior-devs"],
      "assignees": ["tech-lead"]
    },
    
    // Pin Docker images
    {
      "matchDatasources": ["docker"],
      "pinDigests": true
    },
    
    // GitHub Actions - pin to digest
    {
      "matchManagers": ["github-actions"],
      "pinDigests": true
    }
  ],
  
  // Dependency Dashboard
  "dependencyDashboard": true,
  "dependencyDashboardTitle": "Dependency Updates",
  
  // Commit messages
  "commitMessagePrefix": "chore(deps):",
  "commitMessageAction": "update",
  
  // PR settings
  "prBodyColumns": ["Package", "Type", "Update", "Change", "References"],
  "prBodyNotes": [
    "Run `npm test` after merging to verify no breaking changes."
  ],
  
  // Ignore patterns
  "ignoreDeps": [
    "node"
  ],
  
  "ignorePaths": [
    "docs/**",
    "examples/**"
  ]
}
```

---

## 38.7 License Checking

### License Checker (Node.js)

```bash
# ติดตั้ง
npm install -g license-checker

# List all licenses
license-checker --json > licenses.json

# ตรวจสอบ incompatible licenses
license-checker \
  --excludePackages "internal-package@1.0.0" \
  --failOn "GPL-2.0;GPL-3.0;AGPL-3.0"

# เฉพาะ production deps
license-checker --production --json > prod-licenses.json
```

### License Checker Script

```python
#!/usr/bin/env python3
# check_licenses.py

import json
import subprocess
import sys
from typing import List, Set, Dict

# Licenses ที่ยอมรับได้
ALLOWED_LICENSES = {
    "MIT",
    "Apache-2.0",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "0BSD",
    "Unlicense",
    "CC0-1.0",
    "Python-2.0",
}

# Licenses ที่ไม่ยอมรับ (copyleft)
DENIED_LICENSES = {
    "GPL-2.0",
    "GPL-3.0",
    "LGPL-2.0",
    "LGPL-2.1",
    "LGPL-3.0",
    "AGPL-3.0",
    "EUPL-1.1",
    "EUPL-1.2",
}

def check_npm_licenses() -> Dict[str, str]:
    """ดึง licenses จาก npm packages"""
    result = subprocess.run(
        ["license-checker", "--json", "--production"],
        capture_output=True,
        text=True
    )
    
    if result.returncode != 0:
        print(f"Error: {result.stderr}")
        return {}
    
    return json.loads(result.stdout)

def check_python_licenses() -> Dict[str, str]:
    """ดึง licenses จาก Python packages"""
    result = subprocess.run(
        ["pip-licenses", "--format", "json"],
        capture_output=True,
        text=True
    )
    
    if result.returncode != 0:
        print(f"Error: {result.stderr}")
        return {}
    
    packages = json.loads(result.stdout)
    return {pkg["Name"]: pkg["License"] for pkg in packages}

def analyze_licenses(licenses: Dict[str, str]) -> tuple:
    """วิเคราะห์ licenses และแยกเป็น allowed/denied/unknown"""
    allowed = []
    denied = []
    unknown = []
    
    for package, license_str in licenses.items():
        # license-checker อาจมีหลาย licenses คั่นด้วย ;
        package_licenses = {l.strip() for l in license_str.split(";")}
        
        # ตรวจสอบ denied licenses ก่อน
        denied_found = package_licenses & DENIED_LICENSES
        if denied_found:
            denied.append({
                "package": package,
                "license": license_str,
                "denied_licenses": list(denied_found)
            })
            continue
        
        # ตรวจสอบ unknown licenses
        unknown_licenses = package_licenses - ALLOWED_LICENSES - DENIED_LICENSES
        if unknown_licenses:
            unknown.append({
                "package": package,
                "license": license_str,
                "unknown_licenses": list(unknown_licenses)
            })
            continue
        
        allowed.append(package)
    
    return allowed, denied, unknown

def main():
    print("Checking licenses...")
    
    # Check npm licenses
    npm_licenses = check_npm_licenses()
    _, npm_denied, npm_unknown = analyze_licenses(npm_licenses)
    
    # Check Python licenses
    py_licenses = check_python_licenses()
    _, py_denied, py_unknown = analyze_licenses(py_licenses)
    
    has_issues = False
    
    # Report denied licenses
    all_denied = npm_denied + py_denied
    if all_denied:
        has_issues = True
        print(f"\n❌ Found {len(all_denied)} package(s) with DENIED licenses:")
        for pkg in all_denied:
            print(f"  - {pkg['package']}: {pkg['license']}")
            print(f"    Denied: {', '.join(pkg['denied_licenses'])}")
    
    # Report unknown licenses
    all_unknown = npm_unknown + py_unknown
    if all_unknown:
        print(f"\n⚠️  Found {len(all_unknown)} package(s) with UNKNOWN licenses:")
        for pkg in all_unknown[:10]:  # แสดงแค่ 10 อัน
            print(f"  - {pkg['package']}: {pkg['license']}")
    
    if not has_issues:
        print("\n✅ All licenses are compliant!")
        sys.exit(0)
    else:
        print("\n❌ License compliance check FAILED")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

## 38.8 SBOM Generation ด้วย Syft

### Syft Commands

```bash
# ติดตั้ง Syft
brew install syft

# สร้าง SBOM จาก Docker image
syft nginx:latest

# Output เป็น SPDX
syft nginx:latest -o spdx-json > sbom.spdx.json

# Output เป็น CycloneDX
syft nginx:latest -o cyclonedx-json > sbom.cdx.json

# Scan filesystem
syft dir:/path/to/project -o spdx-json > sbom.spdx.json

# Scan ด้วย depth
syft dir:. -o table --scope all-layers

# Attest SBOM กับ image
cosign attest --predicate sbom.spdx.json \
  --type https://spdx.dev/Document \
  registry.example.com/my-app:v1.0.0
```

### SBOM Validation

```python
#!/usr/bin/env python3
# validate_sbom.py

import json
import sys
from datetime import datetime
from typing import List, Dict

def load_spdx_sbom(filepath: str) -> dict:
    """โหลด SPDX SBOM file"""
    with open(filepath) as f:
        return json.load(f)

def validate_sbom(sbom: dict) -> List[str]:
    """ตรวจสอบ SBOM ว่า valid หรือไม่"""
    issues = []
    
    # ตรวจสอบ required fields
    required_fields = ["spdxVersion", "dataLicense", "SPDXID", "name", "documentNamespace"]
    for field in required_fields:
        if field not in sbom:
            issues.append(f"Missing required field: {field}")
    
    # ตรวจสอบ creation info
    if "creationInfo" not in sbom:
        issues.append("Missing creationInfo")
    else:
        creation_info = sbom["creationInfo"]
        if "created" not in creation_info:
            issues.append("Missing creationInfo.created")
        else:
            # ตรวจสอบว่า SBOM ไม่เก่าเกินไป
            created = datetime.fromisoformat(creation_info["created"].replace("Z", "+00:00"))
            age_days = (datetime.now(created.tzinfo) - created).days
            if age_days > 30:
                issues.append(f"SBOM is {age_days} days old (max: 30 days)")
    
    # ตรวจสอบ packages
    packages = sbom.get("packages", [])
    if not packages:
        issues.append("No packages found in SBOM")
    else:
        # ตรวจสอบ required package fields
        for pkg in packages:
            if "name" not in pkg:
                issues.append(f"Package missing name field")
            if "versionInfo" not in pkg and pkg.get("SPDXID") != "SPDXRef-DOCUMENT":
                issues.append(f"Package {pkg.get('name', 'unknown')} missing version")
            if "downloadLocation" not in pkg:
                issues.append(f"Package {pkg.get('name', 'unknown')} missing downloadLocation")
    
    return issues

def check_for_vulnerabilities(sbom: dict, grype_report: dict) -> List[Dict]:
    """Cross-reference SBOM กับ vulnerability report"""
    # Extract package list จาก SBOM
    sbom_packages = {}
    for pkg in sbom.get("packages", []):
        if pkg.get("name") and pkg.get("versionInfo"):
            sbom_packages[pkg["name"]] = pkg["versionInfo"]
    
    # Cross-reference กับ Grype results
    vulnerable_packages = []
    for match in grype_report.get("matches", []):
        artifact = match.get("artifact", {})
        pkg_name = artifact.get("name", "")
        pkg_version = artifact.get("version", "")
        
        if pkg_name in sbom_packages:
            vuln = match.get("vulnerability", {})
            vulnerable_packages.append({
                "package": pkg_name,
                "version": pkg_version,
                "vuln_id": vuln.get("id", ""),
                "severity": vuln.get("severity", "unknown"),
                "fix_version": vuln.get("fix", {}).get("versions", [])
            })
    
    return vulnerable_packages

def main():
    if len(sys.argv) < 2:
        print("Usage: validate_sbom.py <sbom.spdx.json> [grype-report.json]")
        sys.exit(1)
    
    sbom_file = sys.argv[1]
    grype_file = sys.argv[2] if len(sys.argv) > 2 else None
    
    print(f"Validating SBOM: {sbom_file}")
    sbom = load_spdx_sbom(sbom_file)
    
    # Validate structure
    issues = validate_sbom(sbom)
    
    if issues:
        print("\n❌ SBOM validation issues:")
        for issue in issues:
            print(f"  - {issue}")
    else:
        pkg_count = len(sbom.get("packages", []))
        print(f"\n✅ SBOM is valid ({pkg_count} packages)")
    
    # Cross-reference กับ vulnerabilities
    if grype_file:
        print(f"\nCross-referencing with Grype report: {grype_file}")
        with open(grype_file) as f:
            grype_data = json.load(f)
        
        vulnerable = check_for_vulnerabilities(sbom, grype_data)
        
        if vulnerable:
            critical = [v for v in vulnerable if v["severity"].lower() == "critical"]
            high = [v for v in vulnerable if v["severity"].lower() == "high"]
            
            print(f"\nVulnerable packages: {len(vulnerable)}")
            print(f"  Critical: {len(critical)}")
            print(f"  High: {len(high)}")
            
            if critical:
                print("\nCritical vulnerabilities:")
                for v in critical:
                    print(f"  - {v['package']}@{v['version']}: {v['vuln_id']}")
                    if v['fix_version']:
                        print(f"    Fix: upgrade to {', '.join(v['fix_version'])}")
        else:
            print("✅ No vulnerabilities found in SBOM packages")
    
    sys.exit(0 if not issues else 1)

if __name__ == "__main__":
    main()
```

---

## 38.9 GitHub Actions Integration

```yaml
# .github/workflows/dependency-security.yml
name: Dependency Security

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 8 * * 1'  # ทุก Monday 8am

jobs:
  audit:
    name: Dependency Audit
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        ecosystem:
          - name: "Node.js"
            check: "[ -f 'package.json' ]"
          - name: "Python"
            check: "[ -f 'requirements.txt' ]"
    
    steps:
      - uses: actions/checkout@v4
      
      # npm audit
      - name: Setup Node.js
        if: hashFiles('package*.json') != ''
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: npm Audit
        if: hashFiles('package*.json') != ''
        run: |
          npm ci
          npm audit --json > npm-audit.json || true
          
          CRITICAL=$(cat npm-audit.json | jq '.metadata.vulnerabilities.critical // 0')
          HIGH=$(cat npm-audit.json | jq '.metadata.vulnerabilities.high // 0')
          
          echo "Critical: $CRITICAL, High: $HIGH"
          
          if [ "$CRITICAL" -gt "0" ]; then
            echo "::error::Found $CRITICAL critical npm vulnerabilities"
            exit 1
          fi
      
      # pip-audit
      - name: Setup Python
        if: hashFiles('requirements*.txt') != ''
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: pip Audit
        if: hashFiles('requirements*.txt') != ''
        run: |
          pip install pip-audit
          pip-audit -r requirements.txt --format json -o pip-audit.json || true
          
          CRITICAL=$(cat pip-audit.json | jq '[.[] | select(.vulns | map(.fix_versions | length > 0) | any)] | length')
          echo "Packages with fixes: $CRITICAL"
          
          pip-audit -r requirements.txt --format markdown
      
      - name: Upload audit reports
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: audit-reports
          path: |
            npm-audit.json
            pip-audit.json
  
  license-check:
    name: License Compliance
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        if: hashFiles('package.json') != ''
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Check npm licenses
        if: hashFiles('package.json') != ''
        run: |
          npm ci
          npx license-checker --json > npm-licenses.json
          
          # ตรวจสอบ forbidden licenses
          FORBIDDEN=$(cat npm-licenses.json | jq '
            to_entries |
            map(select(.value.licenses | 
              contains("GPL") or 
              contains("AGPL") or 
              contains("LGPL"))) |
            length
          ')
          
          if [ "$FORBIDDEN" -gt "0" ]; then
            echo "::error::Found $FORBIDDEN packages with copyleft licenses"
            cat npm-licenses.json | jq '
              to_entries |
              map(select(.value.licenses | 
                contains("GPL") or 
                contains("AGPL"))) |
              .[] | {package: .key, license: .value.licenses}
            '
            exit 1
          fi
      
      - name: Check Python licenses
        if: hashFiles('requirements.txt') != ''
        run: |
          pip install pip-licenses
          pip install -r requirements.txt
          pip-licenses --format json > py-licenses.json
          
          python3 check_licenses.py
  
  sbom-generation:
    name: Generate SBOM
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Syft
        uses: anchore/sbom-action/download-syft@v0
      
      - name: Generate SBOM
        run: |
          syft dir:. -o spdx-json=sbom.spdx.json -o cyclonedx-json=sbom.cdx.json
      
      - name: Validate SBOM
        run: |
          # ตรวจสอบว่า SBOM valid
          python3 validate_sbom.py sbom.spdx.json
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom-${{ github.sha }}
          path: |
            sbom.spdx.json
            sbom.cdx.json
          retention-days: 365
      
      - name: Attach SBOM to Release
        if: startsWith(github.ref, 'refs/tags/')
        uses: softprops/action-gh-release@v1
        with:
          files: |
            sbom.spdx.json
            sbom.cdx.json
  
  snyk-scan:
    name: Snyk Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif
        continue-on-error: true
      
      - name: Upload Snyk SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: snyk.sarif
```

---

## 38.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Vulnerability Assessment

1. สร้าง Node.js project ที่มี outdated dependencies
2. รัน `npm audit` และวิเคราะห์ผลลัพธ์
3. Fix vulnerabilities ที่แก้ได้โดยไม่มี breaking changes
4. Document vulnerabilities ที่แก้ไม่ได้และเหตุผล

### แบบฝึกหัดที่ 2: Dependabot Configuration

สร้าง Dependabot configuration สำหรับ project ที่มี:
- Frontend (React/Next.js)
- Backend (Python/Django)
- Docker containers
- GitHub Actions

โดยต้องการ:
- Security updates merge อัตโนมัติ
- Minor/patch updates รวมกัน
- Major updates ต้องการ manual review

### แบบฝึกหัดที่ 3: SBOM Pipeline

สร้าง pipeline ที่:
1. Generate SBOM ทุกครั้งที่ merge ไปยัง main
2. Sign SBOM ด้วย Cosign
3. Attach SBOM ไปกับ GitHub Release
4. Scan SBOM สำหรับ known vulnerabilities

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **npm audit, pip-audit**: ตรวจสอบ vulnerabilities ใน dependencies
- **OWASP Dependency-Check**: Comprehensive dependency scanning
- **Snyk**: Integrated security platform
- **Dependabot**: Automated dependency updates
- **Renovate**: Advanced dependency management with grouping
- **License Checking**: ตรวจสอบ license compliance
- **SBOM**: สร้าง Software Bill of Materials ด้วย Syft

บทต่อไป (Part 39) เราจะเรียนรู้เรื่อง **Database Migrations ใน CI/CD**

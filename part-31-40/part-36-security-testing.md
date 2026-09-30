# Part 36: Security Testing ใน CI/CD Pipeline

## บทนำ

Security Testing เป็นขั้นตอนสำคัญใน CI/CD pipeline ที่ช่วยตรวจจับช่องโหว่ด้านความปลอดภัยก่อนที่ code จะถูก deploy ไปยัง production การทำ "Shift Left Security" หรือ DevSecOps ช่วยให้ทีมสามารถค้นพบและแก้ไขปัญหาได้เร็วขึ้น ในบทนี้เราจะเรียนรู้ SAST, DAST, Secret Scanning, และเครื่องมืออื่นๆ

## วัตถุประสงค์การเรียนรู้

- เข้าใจ SAST และ DAST แตกต่างกันอย่างไร
- ใช้ Semgrep สำหรับ SAST
- ตั้งค่า OWASP ZAP สำหรับ DAST
- ค้นหา secrets ที่ถูก commit โดยไม่ตั้งใจ
- สร้าง security gates ใน pipeline
- เข้าใจ OWASP Top 10

---

## 36.1 Security Testing Fundamentals

### SAST vs DAST

| | SAST | DAST |
|--|------|------|
| ย่อมาจาก | Static Application Security Testing | Dynamic Application Security Testing |
| วิเคราะห์อะไร | Source code | Running application |
| เมื่อไร | Build time | Deploy time |
| ต้องการ | Source code access | Running app URL |
| ค้นพบ | Injection flaws ใน code, insecure patterns | Runtime vulnerabilities, misconfigurations |
| เครื่องมือ | Semgrep, Bandit, ESLint | OWASP ZAP, Nikto, Burp Suite |

---

## 36.2 SAST ด้วย Semgrep

### การติดตั้ง Semgrep

```bash
# pip
pip install semgrep

# Docker
docker pull returntocorp/semgrep

# Homebrew
brew install semgrep
```

### Semgrep Rules

```yaml
# rules/security.yaml
rules:
  # SQL Injection
  - id: sql-injection-raw-query
    patterns:
      - pattern: |
          cursor.execute("..." + $INPUT)
      - pattern: |
          cursor.execute(f"...{$INPUT}...")
    message: |
      Potential SQL injection detected. Use parameterized queries instead.
      Replace: cursor.execute("SELECT * FROM users WHERE id = " + user_id)
      With: cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    languages: [python]
    severity: ERROR
    metadata:
      category: security
      cwe: "CWE-89"
      owasp: "A03:2021"
  
  # Hard-coded passwords
  - id: hardcoded-password
    patterns:
      - pattern: password = "..."
      - pattern: PASSWORD = "..."
      - pattern: secret = "..."
      - pattern-not: password = ""
      - pattern-not: password = "test"
    message: |
      Hard-coded password detected. Use environment variables or secrets manager.
    languages: [python, javascript, go, java]
    severity: ERROR
    metadata:
      category: security
      cwe: "CWE-259"
  
  # Insecure random
  - id: insecure-random
    patterns:
      - pattern: random.random()
      - pattern: random.randint(...)
    message: |
      Using insecure random number generator for security purposes.
      Use secrets.token_bytes() or os.urandom() instead.
    languages: [python]
    severity: WARNING
    metadata:
      cwe: "CWE-338"
  
  # Path traversal
  - id: path-traversal
    patterns:
      - pattern: open($INPUT, ...)
    message: |
      Potential path traversal. Validate and sanitize file paths.
    languages: [python]
    severity: WARNING
    metadata:
      cwe: "CWE-22"
  
  # XSS - Unsafe HTML rendering
  - id: xss-dangerouslysetinnerhtml
    pattern: dangerouslySetInnerHTML={{ __html: $VAR }}
    message: |
      Using dangerouslySetInnerHTML can lead to XSS.
      Ensure $VAR is properly sanitized.
    languages: [javascript, typescript]
    severity: WARNING
    metadata:
      cwe: "CWE-79"
  
  # Insecure JWT
  - id: jwt-none-algorithm
    patterns:
      - pattern: jwt.decode($TOKEN, options={"algorithms": ["none"]})
    message: |
      JWT decoded with 'none' algorithm - critical security vulnerability.
    languages: [python]
    severity: ERROR
    metadata:
      cwe: "CWE-347"
```

### Semgrep ใน Python Project

```python
# ตัวอย่าง code ที่มีช่องโหว่ (เพื่อการเรียนรู้เท่านั้น)

# ❌ Bad: SQL Injection
def get_user_bad(user_id):
    query = f"SELECT * FROM users WHERE id = {user_id}"
    cursor.execute(query)
    return cursor.fetchone()

# ✅ Good: Parameterized query
def get_user_good(user_id):
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    return cursor.fetchone()

# ❌ Bad: Hard-coded secret
DATABASE_PASSWORD = "super_secret_password_123"

# ✅ Good: Environment variable
import os
DATABASE_PASSWORD = os.environ.get("DATABASE_PASSWORD")

# ❌ Bad: Weak random
import random
token = str(random.randint(100000, 999999))

# ✅ Good: Cryptographically secure
import secrets
token = secrets.token_urlsafe(32)

# ❌ Bad: Path traversal
def serve_file(filename):
    with open(f"/uploads/{filename}", 'rb') as f:
        return f.read()

# ✅ Good: Sanitized path
import os
import pathlib

def serve_file_safe(filename):
    upload_dir = pathlib.Path("/uploads")
    requested_path = (upload_dir / filename).resolve()
    
    # ตรวจสอบว่า path อยู่ใน upload directory เท่านั้น
    if not str(requested_path).startswith(str(upload_dir)):
        raise ValueError("Invalid file path")
    
    with open(requested_path, 'rb') as f:
        return f.read()
```

### Bandit สำหรับ Python

```bash
# ติดตั้ง
pip install bandit

# รัน basic scan
bandit -r src/

# รัน พร้อม custom config
bandit -r src/ -c bandit.yaml

# Output เป็น JSON
bandit -r src/ -f json -o bandit-report.json

# รัน specific tests
bandit -r src/ -t B101,B201,B301
```

```yaml
# bandit.yaml
skips: []
tests:
  - B101  # assert_used
  - B201  # flask_debug_true
  - B301  # pickle
  - B302  # marshal
  - B303  # md5/sha1
  - B304  # des/rc2/rc4 ciphers
  - B305  # cipher_modes
  - B306  # mktemp
  - B307  # eval
  - B308  # mark_safe
  - B310  # urllib_urlopen
  - B311  # random
  - B312  # telnetlib
  - B313  # xml_bad_cElementTree
  - B320  # xml_bad_expat
  - B321  # ftp_lib
  - B322  # input
  - B323  # unverified_context
  - B324  # hashlib_new_insecure_functions
  - B325  # tempnam
  - B401  # import_telnetlib
  - B402  # import_ftplib
  - B403  # import_pickle
  - B404  # import_subprocess
  - B405  # import_xml_etree
  - B411  # import_xmlrpclib
  - B412  # import_httpoxy
  - B413  # import_pycrypto
  - B501  # request_with_no_cert_validation
  - B502  # ssl_with_bad_version
  - B503  # ssl_with_bad_defaults
  - B504  # ssl_with_no_version
  - B505  # weak_cryptographic_key
  - B506  # yaml_load
  - B507  # ssh_no_host_key_verification
  - B601  # paramiko_calls
  - B602  # subprocess_popen_with_shell_equals_true
  - B603  # subprocess_without_shell_equals_true
  - B604  # any_other_function_with_shell_equals_true
  - B605  # start_process_with_a_shell
  - B606  # start_process_with_no_shell
  - B607  # start_process_with_partial_path
  - B608  # hardcoded_sql_expressions
  - B609  # linux_commands_wildcard_injection
  - B610  # django_extra
  - B611  # django_rawsql_used
  - B701  # jinja2_autoescape_false
  - B702  # use_of_mako_templates
  - B703  # django_mark_safe

exclude_dirs:
  - tests
  - .git
  - node_modules
  - venv

assert_used:
  skips: ['*_test.py', 'test_*.py']
```

### ESLint Security Plugin

```javascript
// .eslintrc.js
module.exports = {
  plugins: ['security', 'no-unsanitized'],
  extends: [
    'plugin:security/recommended',
  ],
  rules: {
    // Prevent eval()
    'no-eval': 'error',
    
    // Prevent dangerous innerHTML
    'no-unsanitized/method': 'error',
    'no-unsanitized/property': 'error',
    
    // Security rules
    'security/detect-object-injection': 'warn',
    'security/detect-non-literal-regexp': 'warn',
    'security/detect-unsafe-regex': 'error',
    'security/detect-buffer-noassert': 'error',
    'security/detect-child-process': 'warn',
    'security/detect-disable-mustache-escape': 'error',
    'security/detect-eval-with-expression': 'error',
    'security/detect-new-buffer': 'error',
    'security/detect-no-csrf-before-method-override': 'error',
    'security/detect-non-literal-fs-filename': 'warn',
    'security/detect-non-literal-require': 'warn',
    'security/detect-possible-timing-attacks': 'warn',
    'security/detect-pseudoRandomBytes': 'error',
  }
};
```

---

## 36.3 Secret Scanning

### Gitleaks

```bash
# ติดตั้ง
brew install gitleaks

# Scan repository
gitleaks detect --source . --verbose

# Scan specific commit range
gitleaks detect --source . --log-opts="HEAD~10..HEAD"

# Scan staged changes
gitleaks protect --staged

# Generate report
gitleaks detect --source . --report-format json --report-path gitleaks-report.json
```

```toml
# .gitleaks.toml
[extend]
useDefault = true

[[rules]]
id = "thai-national-id"
description = "Thai National ID"
regex = '''\b\d{1}-\d{4}-\d{5}-\d{2}-\d{1}\b'''
secretGroup = 0
keywords = ["national_id", "national-id", "thaiid"]

[[rules]]
id = "jwt-token"
description = "JWT Token"
regex = '''eyJ[A-Za-z0-9-_]+\.eyJ[A-Za-z0-9-_]+\.[A-Za-z0-9-_]+'''
secretGroup = 0

[[rules]]
id = "stripe-live-key"
description = "Stripe Live Key"
regex = '''sk_live_[0-9a-zA-Z]{24}'''
secretGroup = 0
keywords = ["stripe"]

[allowlist]
description = "Allow test and example patterns"
regexes = [
  '''example\.com''',
  '''test_token_\w+''',
  '''PLACEHOLDER''',
]
paths = [
  ".*/tests/.*",
  ".*/fixtures/.*",
  ".*test.*\\.py",
]
```

### TruffleHog

```bash
# ติดตั้ง
pip install trufflehog

# Scan git history
trufflehog git https://github.com/org/repo.git --json > trufflehog-report.json

# Scan local repo
trufflehog filesystem . --json

# Scan specific branch
trufflehog git . --branch feature/new-feature
```

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  # Secret scanning
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
        name: Detect secrets
        description: Scan for secrets before commit
  
  # Python security
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.5
    hooks:
      - id: bandit
        args: ["-c", "bandit.yaml"]
        exclude: tests/
  
  # Semgrep
  - repo: https://github.com/returntocorp/semgrep
    rev: v1.45.0
    hooks:
      - id: semgrep
        args: ['--config', 'rules/security.yaml', '--error']
  
  # General
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
      - id: detect-aws-credentials
```

---

## 36.4 DAST ด้วย OWASP ZAP

### ZAP Baseline Scan

```bash
# Docker
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t https://target-app.example.com \
  -r zap-report.html \
  -J zap-report.json

# Full scan
docker run -t owasp/zap2docker-stable zap-full-scan.py \
  -t https://target-app.example.com \
  -r zap-report.html

# API scan
docker run -t owasp/zap2docker-stable zap-api-scan.py \
  -t https://api.example.com/openapi.json \
  -f openapi \
  -r zap-api-report.html
```

### ZAP Automation Framework

```yaml
# zap-automation.yml
env:
  contexts:
    - name: "Target Application"
      urls:
        - "https://target-app.example.com"
      authentication:
        method: "form"
        parameters:
          loginUrl: "https://target-app.example.com/login"
          loginRequestData: "email={%username%}&password={%password%}"
          usernameParameter: "email"
          passwordParameter: "password"
      sessionManagement:
        method: "cookie"
      users:
        - name: "test_user"
          credentials:
            username: "testuser@example.com"
            password: "TestPassword123!"
  
  parameters:
    failOnError: true
    failOnWarning: false
    progressToStdout: true

jobs:
  # Spider
  - type: spider
    parameters:
      context: "Target Application"
      user: "test_user"
      maxDuration: 10
      maxDepth: 5
  
  # Ajax Spider (for JavaScript-heavy apps)
  - type: spiderAjax
    parameters:
      context: "Target Application"
      user: "test_user"
      maxDuration: 5
  
  # Active scan
  - type: activeScan
    parameters:
      context: "Target Application"
      user: "test_user"
      policy: "Default Policy"
      maxRuleDurationInMins: 2
  
  # Report
  - type: report
    parameters:
      template: "traditional-html"
      reportDir: "/tmp/reports"
      reportFile: "zap-report"
      reportTitle: "ZAP Security Report"
    risks:
      - high
      - medium
      - low
      - informational
```

### ZAP API Testing

```python
#!/usr/bin/env python3
# zap_api_test.py - ใช้ ZAP Python API

from zapv2 import ZAPv2
import time
import sys

def run_zap_scan(target_url: str, api_key: str = None):
    """รัน ZAP scan บน target URL"""
    
    # Connect ไปยัง ZAP proxy
    zap = ZAPv2(
        apikey=api_key,
        proxies={'http': 'http://localhost:8080', 'https': 'http://localhost:8080'}
    )
    
    print(f"Starting scan of {target_url}")
    
    # Spider
    print("Starting Spider...")
    scan_id = zap.spider.scan(target_url)
    
    while int(zap.spider.status(scan_id)) < 100:
        print(f"Spider progress: {zap.spider.status(scan_id)}%")
        time.sleep(5)
    
    print("Spider completed")
    
    # Active scan
    print("Starting Active Scan...")
    scan_id = zap.ascan.scan(target_url)
    
    while int(zap.ascan.status(scan_id)) < 100:
        print(f"Active scan progress: {zap.ascan.status(scan_id)}%")
        time.sleep(10)
    
    print("Active scan completed")
    
    # Get alerts
    alerts = zap.core.alerts()
    
    # Categorize by risk
    high_risk = [a for a in alerts if a['risk'] == 'High']
    medium_risk = [a for a in alerts if a['risk'] == 'Medium']
    low_risk = [a for a in alerts if a['risk'] == 'Low']
    
    print(f"\nScan Results:")
    print(f"High Risk: {len(high_risk)}")
    print(f"Medium Risk: {len(medium_risk)}")
    print(f"Low Risk: {len(low_risk)}")
    
    # Print high risk findings
    if high_risk:
        print("\nHigh Risk Findings:")
        for alert in high_risk:
            print(f"  - {alert['name']}")
            print(f"    URL: {alert['url']}")
            print(f"    Description: {alert['description'][:200]}")
    
    # Generate HTML report
    report = zap.core.htmlreport()
    with open('zap-report.html', 'wb') as f:
        f.write(report)
    
    # Fail if high risk found
    if len(high_risk) > 0:
        print(f"\nFAIL: {len(high_risk)} high risk vulnerability(s) found")
        return False
    
    print("\nPASS: No high risk vulnerabilities found")
    return True

if __name__ == "__main__":
    target = sys.argv[1] if len(sys.argv) > 1 else "http://localhost:3000"
    success = run_zap_scan(target)
    sys.exit(0 if success else 1)
```

---

## 36.5 OWASP Top 10 Testing

### A01: Broken Access Control

```python
# tests/security/test_access_control.py
import pytest
import requests

BASE_URL = "http://localhost:8080"

class TestAccessControl:
    """ทดสอบ Broken Access Control (OWASP A01)"""
    
    def test_unauthorized_access_to_admin(self):
        """ไม่มี auth token ไม่ควรเข้า admin"""
        response = requests.get(f"{BASE_URL}/api/admin/users")
        assert response.status_code == 401
    
    def test_regular_user_cannot_access_admin(self, regular_user_token):
        """Regular user ไม่ควรเข้า admin"""
        headers = {"Authorization": f"Bearer {regular_user_token}"}
        response = requests.get(f"{BASE_URL}/api/admin/users", headers=headers)
        assert response.status_code == 403
    
    def test_user_cannot_access_other_user_data(self, user1_token, user2_id):
        """User ไม่ควรเข้าถึงข้อมูลของ user อื่น"""
        headers = {"Authorization": f"Bearer {user1_token}"}
        response = requests.get(
            f"{BASE_URL}/api/users/{user2_id}/private-data",
            headers=headers
        )
        assert response.status_code in [403, 404]
    
    def test_idor_protection(self, user1_token):
        """ทดสอบ Insecure Direct Object Reference"""
        headers = {"Authorization": f"Bearer {user1_token}"}
        
        # พยายาม access orders ของ user อื่น
        for order_id in range(1, 100):
            response = requests.get(
                f"{BASE_URL}/api/orders/{order_id}",
                headers=headers
            )
            
            if response.status_code == 200:
                # ถ้าเข้าถึงได้ ต้องเป็น order ของ user นี้เท่านั้น
                order = response.json()
                assert order['userId'] == user1_token_user_id, \
                    f"IDOR vulnerability: User accessed order {order_id} belonging to another user"
```

### A02: Cryptographic Failures

```python
class TestCryptography:
    """ทดสอบ Cryptographic Failures (OWASP A02)"""
    
    def test_password_not_in_response(self, admin_token):
        """Password ไม่ควรปรากฏใน API response"""
        headers = {"Authorization": f"Bearer {admin_token}"}
        response = requests.get(f"{BASE_URL}/api/users/1", headers=headers)
        
        data = response.json()
        assert 'password' not in data
        assert 'password_hash' not in data
    
    def test_sensitive_data_not_in_logs(self):
        """ข้อมูล sensitive ไม่ควรอยู่ใน logs"""
        # Login และดู response headers
        response = requests.post(f"{BASE_URL}/api/auth/login", json={
            "email": "test@example.com",
            "password": "TestPassword123!"
        })
        
        # ตรวจสอบว่า token ไม่อยู่ใน URL
        assert response.url == f"{BASE_URL}/api/auth/login"
        
        # ตรวจสอบ HTTPS enforcement headers
        # (ทำงานใน production environment)
    
    def test_api_uses_https_in_production(self):
        """API ควรใช้ HTTPS ใน production"""
        if os.environ.get('APP_ENV') == 'production':
            response = requests.get(BASE_URL, allow_redirects=False)
            # Redirect HTTP -> HTTPS
            assert response.status_code in [301, 302]
            assert response.headers['Location'].startswith('https://')

class TestInputValidation:
    """ทดสอบ Injection (OWASP A03)"""
    
    def test_sql_injection_prevention(self):
        """ทดสอบ SQL Injection"""
        sql_payloads = [
            "' OR '1'='1",
            "'; DROP TABLE users; --",
            "1; SELECT * FROM users WHERE 1=1",
            "admin'--",
        ]
        
        for payload in sql_payloads:
            response = requests.post(f"{BASE_URL}/api/auth/login", json={
                "email": payload,
                "password": "anything"
            })
            # ควรได้ 401 หรือ 400 ไม่ใช่ 200 หรือ 500
            assert response.status_code in [400, 401], \
                f"Potential SQL injection with payload: {payload}"
    
    def test_xss_prevention(self, admin_token):
        """ทดสอบ XSS"""
        xss_payloads = [
            "<script>alert('xss')</script>",
            "<img src=x onerror=alert('xss')>",
            "javascript:alert('xss')",
        ]
        
        headers = {"Authorization": f"Bearer {admin_token}"}
        
        for payload in xss_payloads:
            # ลอง create product ที่มี XSS ใน name
            response = requests.post(
                f"{BASE_URL}/api/products",
                json={"name": payload, "price": 100},
                headers=headers
            )
            
            if response.status_code in [200, 201]:
                product = response.json()
                # ตรวจสอบว่า payload ถูก sanitize
                assert '<script>' not in product.get('name', '')
                assert 'onerror=' not in product.get('name', '')
```

---

## 36.6 Security Gates ใน Pipeline

```yaml
# .github/workflows/security-scan.yml
name: Security Scanning

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # ทุก Monday เวลา 6am

permissions:
  contents: read
  security-events: write
  actions: read

jobs:
  sast:
    name: Static Analysis (SAST)
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      # Semgrep
      - name: Semgrep Security Scan
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten
            rules/security.yaml
          generateSarif: '1'
          auditOn: push
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
      
      - name: Upload Semgrep SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: semgrep.sarif
      
      # Bandit (Python)
      - name: Bandit Security Scan
        if: hashFiles('**/*.py') != ''
        run: |
          pip install bandit
          bandit -r src/ -f json -o bandit-report.json -ll || true
          
          # ตรวจสอบ high severity issues
          HIGH_COUNT=$(cat bandit-report.json | jq '.results | map(select(.issue_severity == "HIGH")) | length')
          echo "High severity issues: $HIGH_COUNT"
          
          if [ "$HIGH_COUNT" -gt "0" ]; then
            echo "::error::Found $HIGH_COUNT high severity security issues"
            cat bandit-report.json | jq '.results[] | select(.issue_severity == "HIGH") | {test_name, issue_text, filename, line_number}'
            exit 1
          fi
      
      # ESLint Security (JavaScript/TypeScript)
      - name: ESLint Security Scan
        if: hashFiles('**/*.js', '**/*.ts') != ''
        run: |
          npm install eslint eslint-plugin-security eslint-plugin-no-unsanitized
          npx eslint . --ext .js,.ts,.jsx,.tsx \
            --plugin security \
            --rule 'security/detect-eval-with-expression: error' \
            --rule 'security/detect-non-literal-regexp: warn' \
            --format json \
            --output-file eslint-security-report.json || true
          
          # ตรวจสอบ errors
          ERROR_COUNT=$(cat eslint-security-report.json | jq '[.[].messages[] | select(.severity == 2)] | length')
          echo "Security errors: $ERROR_COUNT"
          
          if [ "$ERROR_COUNT" -gt "0" ]; then
            echo "::error::Found $ERROR_COUNT security errors"
            exit 1
          fi
  
  secret-scanning:
    name: Secret Scanning
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Gitleaks Secret Scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_ENABLE_COMMENTS: true
      
      - name: TruffleHog Secret Scan
        uses: trufflesecurity/trufflehog@v3
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --json --only-verified
  
  dependency-audit:
    name: Dependency Audit
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # npm audit
      - name: npm Security Audit
        if: hashFiles('package.json') != ''
        run: |
          npm audit --audit-level=high --json > npm-audit.json || true
          
          HIGH_VULNS=$(cat npm-audit.json | jq '.metadata.vulnerabilities.high // 0')
          CRITICAL_VULNS=$(cat npm-audit.json | jq '.metadata.vulnerabilities.critical // 0')
          
          echo "High vulnerabilities: $HIGH_VULNS"
          echo "Critical vulnerabilities: $CRITICAL_VULNS"
          
          if [ "$CRITICAL_VULNS" -gt "0" ]; then
            echo "::error::Found $CRITICAL_VULNS critical vulnerabilities"
            exit 1
          fi
      
      # pip-audit (Python)
      - name: pip Security Audit
        if: hashFiles('requirements*.txt') != ''
        run: |
          pip install pip-audit
          pip-audit -r requirements.txt --format json -o pip-audit.json || true
          
          CRITICAL=$(cat pip-audit.json | jq '[.[] | select(.vulns[].fix_versions | length > 0)] | length')
          
          if [ "$CRITICAL" -gt "0" ]; then
            echo "::error::Found $CRITICAL packages with vulnerabilities"
            cat pip-audit.json | jq '.[] | select(.vulns | length > 0) | {name, version, vulns: [.vulns[].id]}'
            exit 1
          fi
  
  dast:
    name: Dynamic Analysis (DAST)
    runs-on: ubuntu-latest
    needs: [sast, dependency-audit]
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Start Application
        run: |
          docker-compose -f docker-compose.test.yml up -d
          ./scripts/wait-for-services.sh
      
      - name: OWASP ZAP Scan
        uses: zaproxy/action-baseline@v0.11.0
        with:
          target: 'http://localhost:8080'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a'
          allow_issue_writing: false
          artifact_name: zap-report
      
      - name: Upload ZAP Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: zap-security-report
          path: report_html.html
      
      - name: Check ZAP Results
        run: |
          # Parse ZAP report สำหรับ high risk issues
          HIGH_RISK=$(cat zap-report.json | jq '[.site[].alerts[] | select(.riskcode >= "3")] | length')
          
          if [ "$HIGH_RISK" -gt "0" ]; then
            echo "::error::ZAP found $HIGH_RISK high risk security issues"
            exit 1
          fi
  
  security-report:
    name: Consolidate Security Report
    runs-on: ubuntu-latest
    needs: [sast, secret-scanning, dependency-audit]
    if: always()
    
    steps:
      - name: Download all reports
        uses: actions/download-artifact@v3
      
      - name: Generate Security Summary
        run: |
          cat << 'EOF' > security-summary.md
          # Security Scan Summary
          
          ## Results
          
          | Check | Status |
          |-------|--------|
          | SAST (Semgrep) | ${{ needs.sast.result }} |
          | Secret Scanning | ${{ needs.secret-scanning.result }} |
          | Dependency Audit | ${{ needs.dependency-audit.result }} |
          EOF
      
      - name: Post PR Comment
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const summary = fs.readFileSync('security-summary.md', 'utf8');
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: summary
            });
```

---

## 36.7 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Fix Security Issues

ตรวจสอบ code ต่อไปนี้และแก้ไขช่องโหว่:

```python
# มีช่องโหว่ซ่อนอยู่ กี่จุด?
from flask import Flask, request, render_template_string
import sqlite3
import hashlib
import random

app = Flask(__name__)
SECRET_KEY = "my_secret_key_123"  # ← ช่องโหว่ที่ 1?

@app.route('/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']
    
    # ← ช่องโหว่ที่ 2?
    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    query = f"SELECT * FROM users WHERE username='{username}' AND password='{hashlib.md5(password.encode()).hexdigest()}'"
    cursor.execute(query)
    user = cursor.fetchone()
    
    if user:
        token = str(random.randint(100000, 999999))  # ← ช่องโหว่ที่ 3?
        return {'token': token, 'user': {'id': user[0], 'password': user[2]}}  # ← ช่องโหว่ที่ 4?

@app.route('/profile')
def profile():
    user_id = request.args.get('id')
    name = request.args.get('name', '')
    
    # ← ช่องโหว่ที่ 5?
    template = f"<h1>Hello {name}</h1><p>Your ID: {user_id}</p>"
    return render_template_string(template)
```

### แบบฝึกหัดที่ 2: Setup Security Pipeline

สร้าง GitHub Actions workflow ที่ทำ:
1. Semgrep scan พร้อม custom rules
2. Gitleaks secret scanning
3. npm/pip audit
4. Upload SARIF report ไปยัง GitHub Security tab

### แบบฝึกหัดที่ 3: Custom Semgrep Rules

เขียน Semgrep rules สำหรับตรวจจับ:
1. การใช้ `eval()` ใน JavaScript
2. การ log ข้อมูล sensitive (email, token)
3. การใช้ HTTP แทน HTTPS ใน config files
4. Missing CSRF protection ใน form handlers

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **SAST**: Semgrep, Bandit, ESLint security rules
- **DAST**: OWASP ZAP automation
- **Secret Scanning**: Gitleaks, TruffleHog, pre-commit hooks
- **Security Gates**: กำหนด pipeline ที่ fail เมื่อพบ vulnerabilities
- **OWASP Top 10**: Test cases สำหรับช่องโหว่ที่พบบ่อย
- **CI Integration**: ทำ security scanning อัตโนมัติใน every PR

บทต่อไป (Part 37) เราจะเรียนรู้เรื่อง **Container Security**

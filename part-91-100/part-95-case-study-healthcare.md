# Part 95: Case Study — Healthcare CI/CD Compliance

## บทนำ

Healthcare เป็นอุตสาหกรรมที่มีกฎระเบียบเข้มงวดที่สุด เพราะเกี่ยวข้องกับชีวิตและข้อมูลสุขภาพของผู้ป่วย บทนี้ศึกษา case study ของ **"MediConnect Thailand"** — แพลตฟอร์ม health tech ที่ให้บริการ telemedicine, EHR (Electronic Health Records), และ lab results management

---

## 95.1 Context และ Requirements

### ข้อมูลองค์กร

```yaml
organization:
  name: "MediConnect Thailand"
  services:
    - "Telemedicine Platform (video consultations)"
    - "EHR Management System"
    - "Lab Results Portal"
    - "Prescription Management"
    - "Pharmacy Integration"
    
  scale:
    registered_patients: 500_000
    doctors_registered: 8_000
    consultations_per_day: 15_000
    health_records_managed: 2_000_000
    
  regulatory_frameworks:
    thailand:
      - "พรบ. คุ้มครองข้อมูลส่วนบุคคล (PDPA) 2562"
      - "ประกาศกระทรวงสาธารณสุข เรื่อง ระบบสารสนเทศทางการแพทย์"
      - "ข้อกำหนดของแพทยสภา เรื่อง Telemedicine"
      - "ระเบียบ อย. เรื่อง Electronic Prescription"
    international:
      - "HIPAA (สำหรับ international partnerships)"
      - "ISO 27001"
      - "ISO 27799 (Health informatics security)"
      - "HL7 FHIR Standards"
```

### Critical System Classification

```yaml
# system-classification.yml
# จำแนกระบบตาม criticality

critical_systems:
  tier_1_life_critical:
    - name: "Emergency Alert System"
      description: "ระบบแจ้งเตือนเหตุฉุกเฉิน"
      downtime_tolerance: "Zero"
      data_sensitivity: "Critical PHI"
      deployment_approval: "Medical Director + CTO"
      
  tier_2_patient_safety:
    - name: "Prescription Validation System"
      description: "ตรวจสอบความถูกต้องของใบสั่งยา"
      downtime_tolerance: "< 5 minutes"
      data_sensitivity: "PHI"
      deployment_approval: "Chief Medical Officer + Engineering"
      
    - name: "Drug Interaction Checker"
      description: "ตรวจสอบ drug interactions"
      downtime_tolerance: "< 5 minutes"
      
  tier_3_clinical:
    - name: "EHR System"
      description: "Electronic Health Records"
      downtime_tolerance: "< 30 minutes"
      data_sensitivity: "PHI"
      
    - name: "Lab Results Portal"
      description: "ระบบผลการตรวจ Lab"
      downtime_tolerance: "< 1 hour"
      
  tier_4_operational:
    - name: "Admin Portal"
      description: "ระบบจัดการภายใน"
      downtime_tolerance: "< 4 hours"
      data_sensitivity: "Internal only"
```

---

## 95.2 HIPAA/PDPA Compliance ใน CI/CD Pipeline

### Data Protection Requirements

```python
# data_protection.py
# Implement HIPAA/PDPA compliant data handling ใน CI/CD

class PHIProtectionControls:
    """
    Protected Health Information (PHI) controls
    ป้องกัน patient data ไม่ให้รั่วไหลใน CI/CD pipeline
    """
    
    # ข้อมูลที่ถือว่าเป็น PHI ตาม HIPAA
    PHI_PATTERNS = [
        r'\b\d{3}-\d{2}-\d{4}\b',          # Thai National ID (format: xxx-xx-xxxxx)
        r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',  # Email
        r'\b\d{10,13}\b',                    # Phone number
        r'HN\d{6,}',                         # Hospital Number
        r'AN\d{6,}',                         # Admission Number
        r'\b\d{4}-\d{2}-\d{2}\b',           # Date of birth (could be)
        r'(ชื่อ|นามสกุล|เบอร์โทร|วันเกิด)',  # Thai PII keywords
    ]
    
    def scan_for_phi(self, content: str) -> list:
        """ค้นหา PHI ใน content"""
        import re
        findings = []
        
        for pattern in self.PHI_PATTERNS:
            matches = re.finditer(pattern, content)
            for match in matches:
                findings.append({
                    'pattern': pattern,
                    'match': match.group(),
                    'position': match.start()
                })
        
        return findings
    
    def scan_test_data(self, test_data_path: str) -> dict:
        """ตรวจสอบ test data ว่ามี real PHI หรือไม่"""
        import os
        import json
        
        results = {
            'files_scanned': 0,
            'files_with_phi': [],
            'total_phi_found': 0
        }
        
        for root, dirs, files in os.walk(test_data_path):
            for file in files:
                if file.endswith(('.json', '.sql', '.csv', '.xml')):
                    filepath = os.path.join(root, file)
                    with open(filepath, 'r', encoding='utf-8', errors='ignore') as f:
                        content = f.read()
                    
                    findings = self.scan_for_phi(content)
                    results['files_scanned'] += 1
                    
                    if findings:
                        results['files_with_phi'].append({
                            'file': filepath,
                            'phi_count': len(findings)
                        })
                        results['total_phi_found'] += len(findings)
        
        return results
    
    def anonymize_test_data(self, data: dict) -> dict:
        """Anonymize patient data สำหรับ testing"""
        import copy
        import hashlib
        
        anonymized = copy.deepcopy(data)
        
        # Fields ที่ต้อง anonymize
        phi_fields = ['patient_name', 'national_id', 'phone', 'email', 
                      'date_of_birth', 'address', 'hospital_number']
        
        def anonymize_value(value: str, field: str) -> str:
            if field == 'patient_name':
                return 'ผู้ป่วยทดสอบ'
            elif field == 'national_id':
                return 'X-XX-XXXXX-XX-X'
            elif field == 'phone':
                return '0X-XXXX-XXXX'
            elif field == 'email':
                # Hash the email but keep domain format
                local = hashlib.md5(value.encode()).hexdigest()[:8]
                return f"test_{local}@example.com"
            elif field == 'date_of_birth':
                return '1990-01-01'  # Fixed test date
            else:
                return f"[ANONYMIZED_{field.upper()}]"
        
        def anonymize_dict(d: dict) -> dict:
            for key, value in d.items():
                if key in phi_fields and isinstance(value, str):
                    d[key] = anonymize_value(value, key)
                elif isinstance(value, dict):
                    d[key] = anonymize_dict(value)
                elif isinstance(value, list):
                    d[key] = [anonymize_dict(item) if isinstance(item, dict) else item for item in value]
            return d
        
        return anonymize_dict(anonymized)


class HealthcareTestDataManager:
    """จัดการ test data สำหรับ healthcare ที่ compliant"""
    
    def __init__(self, phi_protection: PHIProtectionControls):
        self.phi_protection = phi_protection
    
    def create_synthetic_patient(self, scenario: str = 'default') -> dict:
        """สร้าง synthetic patient data ที่ไม่ใช่ข้อมูลจริง"""
        import random
        import uuid
        
        scenarios = {
            'default': {
                'patient_name': 'นาย ทดสอบ ระบบ',
                'national_id': '0-0000-00000-00-0',  # Format ที่ invalid จริง
                'hospital_number': f'HN{random.randint(100000, 999999)}',
                'age': random.randint(20, 80),
                'gender': random.choice(['M', 'F']),
                'blood_type': random.choice(['A', 'B', 'AB', 'O']),
                'allergies': [],
                'is_synthetic': True,
                'created_for': 'testing'
            },
            'diabetic_patient': {
                'patient_name': 'นาง ผู้ป่วยเบาหวาน ทดสอบ',
                'national_id': '0-0000-00001-00-0',
                'conditions': ['Type 2 Diabetes'],
                'medications': ['Metformin 500mg'],
                'is_synthetic': True
            },
            'emergency_case': {
                'patient_name': 'นาย ฉุกเฉิน ทดสอบ',
                'admission_type': 'Emergency',
                'chief_complaint': 'Chest pain (synthetic)',
                'is_synthetic': True
            }
        }
        
        return scenarios.get(scenario, scenarios['default'])
    
    def validate_no_real_phi_in_pipeline(self) -> bool:
        """Validate ก่อน pipeline รันว่าไม่มี real PHI"""
        scan_result = self.phi_protection.scan_test_data('./tests/fixtures')
        
        if scan_result['files_with_phi']:
            print("❌ PHI FOUND IN TEST DATA:")
            for file_info in scan_result['files_with_phi']:
                print(f"   {file_info['file']}: {file_info['phi_count']} PHI instances")
            return False
        
        print(f"✅ No PHI found in test data ({scan_result['files_scanned']} files scanned)")
        return True
```

---

## 95.3 Healthcare CI/CD Pipeline

### EHR System Pipeline

```yaml
# .github/workflows/ehr-pipeline.yml
# Pipeline สำหรับ Electronic Health Records System

name: EHR System CI/CD

on:
  push:
    branches: [main, develop, 'hotfix/**']
  pull_request:
    branches: [main, develop]

env:
  SERVICE_NAME: ehr-service
  TIER: "tier_3_clinical"  # กำหนด tier สำหรับ approval requirements

jobs:
  # ===== PRE-CHECKS =====
  phi-check:
    name: PHI Data Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check for PHI in test fixtures
        run: |
          python scripts/phi_scanner.py --scan-path tests/fixtures/ --fail-on-phi
      
      - name: Verify synthetic data only
        run: |
          python scripts/verify_synthetic_data.py --path tests/fixtures/

  # ===== CODE QUALITY =====
  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: phi-check
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
          
      - name: Run Linting
        run: |
          flake8 src/ tests/ --max-line-length=120
          black src/ tests/ --check
          
      - name: Type Checking
        run: mypy src/ --strict
        
      - name: SonarQube Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # ===== SECURITY =====
  security:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: phi-check
    steps:
      - uses: actions/checkout@v4
      
      - name: Secret Detection
        run: |
          pip install detect-secrets
          detect-secrets scan --baseline .secrets.baseline
          
      - name: PHI Pattern Check in Code
        run: |
          # ตรวจหา hardcoded patient data ใน source code
          python scripts/phi_code_scanner.py --path src/
          
      - name: Dependency Vulnerability Check
        run: |
          pip install safety
          safety check -r requirements.txt --json > safety-report.json
          python scripts/evaluate_safety_report.py safety-report.json
          
      - name: SAST Analysis
        run: |
          pip install bandit
          bandit -r src/ -f json -o bandit-report.json -ll
          python scripts/evaluate_bandit.py bandit-report.json
          
      - name: Check HIPAA Compliance Patterns
        run: |
          # ตรวจสอบว่า code ใช้ encryption สำหรับ PHI fields
          python scripts/check_phi_encryption.py src/

  # ===== TESTS =====
  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: [code-quality, security]
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: mediconnect_test
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup test database with SYNTHETIC data only
        run: |
          python scripts/setup_test_db.py \
            --synthetic-only \
            --anonymize \
            --db-url postgresql://testuser:testpass@localhost:5432/mediconnect_test
      
      - name: Run unit tests
        run: |
          pytest tests/unit/ \
            --cov=src \
            --cov-report=xml \
            --cov-fail-under=75
            
      - name: Run integration tests
        run: |
          pytest tests/integration/ -v
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/mediconnect_test
          
      - name: Run FHIR compliance tests
        run: |
          # ทดสอบว่า API responses conform to HL7 FHIR R4
          pytest tests/fhir_compliance/ -v
          
      - name: Run data privacy tests
        run: |
          # ทดสอบ PHI masking, encryption, access control
          pytest tests/privacy/ -v --tb=short

  # ===== COMPLIANCE CHECKS =====
  compliance-check:
    name: Compliance Verification
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      
      - name: PDPA Compliance Check
        run: |
          python scripts/pdpa_compliance_check.py \
            --check-consent-management \
            --check-data-retention \
            --check-deletion-capability \
            --check-export-capability
            
      - name: Audit Log Completeness Check
        run: |
          # ตรวจสอบว่า audit logging ครบทุก PHI access
          python scripts/check_audit_completeness.py src/
          
      - name: Encryption Standards Check
        run: |
          # ตรวจสอบว่าใช้ AES-256 สำหรับ PHI at rest
          python scripts/check_encryption_standards.py \
            --min-key-size=256 \
            --check-transport=TLS_1_3

  # ===== BUILD =====
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: compliance-check
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        uses: docker/build-push-action@v5
        id: build
        with:
          context: .
          push: true
          tags: ${{ env.ECR_REGISTRY }}/${{ env.SERVICE_NAME }}:${{ github.sha }}
          
      - name: Container Security Scan
        run: |
          trivy image \
            --exit-code 1 \
            --severity CRITICAL,HIGH \
            --ignore-unfixed \
            ${{ env.ECR_REGISTRY }}/${{ env.SERVICE_NAME }}:${{ github.sha }}

  # ===== DEPLOY STAGING =====
  deploy-staging:
    name: Deploy to Staging (with UAT)
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: staging
    
    steps:
      - name: Deploy to staging
        run: |
          # Staging deployment
          kubectl set image deployment/${{ env.SERVICE_NAME }} \
            ${{ env.SERVICE_NAME }}=${{ env.ECR_REGISTRY }}/${{ env.SERVICE_NAME }}:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/${{ env.SERVICE_NAME }} -n staging
          
      - name: Notify QA team for UAT
        uses: 8398a7/action-slack@v3
        with:
          status: custom
          custom_payload: |
            {
              "text": "🏥 EHR Service deployed to staging - Please perform UAT",
              "attachments": [{
                "color": "good",
                "fields": [
                  {"title": "Service", "value": "${{ env.SERVICE_NAME }}", "short": true},
                  {"title": "Version", "value": "${{ github.sha }}", "short": true},
                  {"title": "UAT Checklist", "value": "https://confluence.mediconnect.th/uat-checklist"}
                ]
              }]
            }

  # ===== DEPLOY PRODUCTION =====
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Verify UAT sign-off
        run: |
          python scripts/check_uat_signoff.py \
            --version ${{ github.sha }} \
            --require-medical-approval
            
      - name: Pre-deployment backup
        run: |
          # Backup ก่อนทุก production deployment (HIPAA requirement)
          python scripts/create_deployment_backup.py \
            --service ${{ env.SERVICE_NAME }} \
            --tag pre-deploy-${{ github.sha }}
            
      - name: Deploy with health monitoring
        run: |
          python scripts/healthcare_deploy.py \
            --service ${{ env.SERVICE_NAME }} \
            --image ${{ env.ECR_REGISTRY }}/${{ env.SERVICE_NAME }}:${{ github.sha }} \
            --monitor-duration=600 \
            --max-error-rate=0.1
            
      - name: Post-deployment verification
        run: |
          pytest tests/post_deploy/ \
            --base-url=https://ehr.mediconnect.th \
            --critical-path-only
```

---

## 95.4 PHI Encryption และ Key Management

### Encryption at Rest

```python
# phi_encryption.py
# PHI encryption implementation

from cryptography.hazmat.primitives.ciphers.aead import AESGCM
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import boto3
import base64
import os

class PHIEncryptionService:
    """
    Encrypt/decrypt PHI data
    ใช้ AES-256-GCM (authenticated encryption)
    Keys managed ด้วย AWS KMS
    """
    
    def __init__(self, kms_key_id: str, region: str = 'ap-southeast-1'):
        self.kms = boto3.client('kms', region_name=region)
        self.kms_key_id = kms_key_id
    
    def encrypt_phi(self, plaintext: str, context: dict = None) -> str:
        """
        Encrypt PHI ด้วย envelope encryption:
        1. Generate data key จาก KMS
        2. Encrypt data ด้วย data key
        3. Store encrypted data key กับ ciphertext
        """
        
        # Generate data key
        encryption_context = context or {}
        response = self.kms.generate_data_key(
            KeyId=self.kms_key_id,
            KeySpec='AES_256',
            EncryptionContext=encryption_context
        )
        
        plaintext_key = response['Plaintext']
        encrypted_key = response['CiphertextBlob']
        
        # Encrypt data
        aesgcm = AESGCM(plaintext_key)
        nonce = os.urandom(12)
        ciphertext = aesgcm.encrypt(
            nonce,
            plaintext.encode('utf-8'),
            None
        )
        
        # Package: nonce + encrypted_key + ciphertext
        result = {
            'nonce': base64.b64encode(nonce).decode(),
            'encrypted_key': base64.b64encode(encrypted_key).decode(),
            'ciphertext': base64.b64encode(ciphertext).decode(),
            'key_id': self.kms_key_id
        }
        
        # Zero out plaintext key from memory
        del plaintext_key
        
        return base64.b64encode(str(result).encode()).decode()
    
    def decrypt_phi(self, encrypted_data: str, context: dict = None, 
                    accessor: str = None, purpose: str = None) -> str:
        """
        Decrypt PHI พร้อม audit logging
        """
        import json
        
        # Log access (HIPAA requirement)
        self._log_phi_access(
            accessor=accessor or 'unknown',
            purpose=purpose or 'unknown',
            action='decrypt'
        )
        
        data = json.loads(base64.b64decode(encrypted_data).decode())
        
        # Decrypt data key
        response = self.kms.decrypt(
            CiphertextBlob=base64.b64decode(data['encrypted_key']),
            EncryptionContext=context or {}
        )
        
        plaintext_key = response['Plaintext']
        
        # Decrypt data
        aesgcm = AESGCM(plaintext_key)
        plaintext = aesgcm.decrypt(
            base64.b64decode(data['nonce']),
            base64.b64decode(data['ciphertext']),
            None
        )
        
        del plaintext_key
        return plaintext.decode('utf-8')
    
    def _log_phi_access(self, accessor: str, purpose: str, action: str):
        """Audit log ทุกครั้งที่มีการ access PHI"""
        import logging
        
        audit_logger = logging.getLogger('phi_audit')
        audit_logger.info(json.dumps({
            'timestamp': datetime.utcnow().isoformat(),
            'action': action,
            'accessor': accessor,
            'purpose': purpose,
            'phi_type': 'encrypted_field'
        }))

# Model ที่ใช้ PHI Encryption
class PatientRecord:
    def __init__(self, encryption_service: PHIEncryptionService):
        self.encryption = encryption_service
    
    def create_record(self, patient_data: dict) -> dict:
        """สร้าง patient record พร้อม PHI encryption"""
        
        phi_fields = ['national_id', 'full_name', 'date_of_birth', 'phone', 'address']
        
        encrypted_record = {}
        for key, value in patient_data.items():
            if key in phi_fields:
                # Encrypt PHI fields
                encrypted_record[f'{key}_encrypted'] = self.encryption.encrypt_phi(
                    str(value),
                    context={'patient_id': patient_data.get('hospital_number', ''), 'field': key}
                )
            else:
                encrypted_record[key] = value
        
        return encrypted_record
```

---

## 95.5 Change Control สำหรับ Clinical Systems

### Clinical Change Management

```python
# clinical_change_manager.py

from enum import Enum
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional

class ClinicalImpact(Enum):
    NO_IMPACT = "no_impact"
    LOW = "low"           # UI changes, performance improvements
    MEDIUM = "medium"     # Feature additions, non-critical flows
    HIGH = "high"         # Clinical decision support, prescription
    CRITICAL = "critical" # Patient safety systems

@dataclass  
class ClinicalChangeRequest:
    service: str
    change_description: str
    clinical_impact: ClinicalImpact
    affected_workflows: List[str]
    tested_by: str
    medical_reviewer: Optional[str] = None
    it_reviewer: Optional[str] = None
    
    @property
    def requires_medical_review(self) -> bool:
        return self.clinical_impact in [ClinicalImpact.HIGH, ClinicalImpact.CRITICAL]
    
    @property
    def deployment_window(self) -> str:
        """กำหนด maintenance window ตาม clinical impact"""
        windows = {
            ClinicalImpact.NO_IMPACT: "Any time",
            ClinicalImpact.LOW: "Off-peak hours (22:00-06:00)",
            ClinicalImpact.MEDIUM: "Tuesday/Thursday 02:00-04:00",
            ClinicalImpact.HIGH: "Sunday 02:00-04:00 (lowest patient load)",
            ClinicalImpact.CRITICAL: "Scheduled maintenance window only"
        }
        return windows[self.clinical_impact]

class ClinicalChangeValidator:
    
    CLINICAL_CHECKLIST = {
        ClinicalImpact.HIGH: [
            "Clinical workflow diagram updated",
            "Physician walkthrough completed",
            "Test with synthetic patient data in staging",
            "Rollback plan tested",
            "Medical Director approval obtained",
            "HL7 FHIR compliance verified",
            "Drug interaction database integrity checked",
            "Audit trail completeness verified"
        ],
        ClinicalImpact.CRITICAL: [
            "Full clinical validation by medical team",
            "Hospital Medical Committee approval",
            "Regulatory notification (if required)",
            "Parallel run period planned",
            "Emergency rollback protocol documented",
            "24/7 on-call physician designated during rollout",
            "Patient safety committee sign-off"
        ]
    }
    
    def validate_change(self, change: ClinicalChangeRequest) -> dict:
        checklist = self.CLINICAL_CHECKLIST.get(change.clinical_impact, [])
        
        missing_items = []
        if change.requires_medical_review and not change.medical_reviewer:
            missing_items.append("Medical reviewer not assigned")
        
        return {
            'approved': len(missing_items) == 0,
            'missing_requirements': missing_items,
            'checklist': checklist,
            'deployment_window': change.deployment_window,
            'requires_notification': change.clinical_impact == ClinicalImpact.CRITICAL
        }
    
    def generate_change_documentation(self, change: ClinicalChangeRequest) -> str:
        """สร้าง change documentation สำหรับ compliance"""
        return f"""
CLINICAL SYSTEM CHANGE REQUEST
════════════════════════════════════════
Service: {change.service}
Date: {datetime.now().strftime('%Y-%m-%d %H:%M')}
Clinical Impact: {change.clinical_impact.value.upper()}
Deployment Window: {change.deployment_window}

CHANGE DESCRIPTION
{change.change_description}

AFFECTED CLINICAL WORKFLOWS
{chr(10).join('- ' + wf for wf in change.affected_workflows)}

APPROVALS REQUIRED
{'✅ Medical Review Required' if change.requires_medical_review else '✅ IT Review Only'}
Medical Reviewer: {change.medical_reviewer or 'Not assigned'}
IT Reviewer: {change.it_reviewer or 'Not assigned'}

COMPLIANCE CHECKLIST
{chr(10).join('☐ ' + item for item in self.CLINICAL_CHECKLIST.get(change.clinical_impact, ['Standard IT review']))}
"""
```

---

## 95.6 Audit และ Compliance Reporting

### HIPAA Audit Report

```python
# hipaa_audit_reporter.py

from datetime import date, datetime, timedelta

class HIPAAAuditReporter:
    """สร้าง HIPAA compliance reports สำหรับ audit"""
    
    def __init__(self, audit_db, phi_access_log):
        self.audit_db = audit_db
        self.phi_access_log = phi_access_log
    
    def generate_phi_access_report(self, start: date, end: date) -> dict:
        """
        รายงาน PHI access สำหรับ HIPAA audit
        ต้องแสดงว่าใครเข้าถึง patient data เมื่อไร ทำไม
        """
        accesses = self.phi_access_log.query(start, end)
        
        # Categorize accesses
        treatment_access = [a for a in accesses if a['purpose'] == 'treatment']
        payment_access = [a for a in accesses if a['purpose'] == 'payment']
        operations_access = [a for a in accesses if a['purpose'] == 'healthcare_operations']
        unauthorized = [a for a in accesses if a['purpose'] not in ['treatment', 'payment', 'healthcare_operations']]
        
        return {
            'period': f"{start.isoformat()} to {end.isoformat()}",
            'total_phi_accesses': len(accesses),
            'by_purpose': {
                'treatment': len(treatment_access),
                'payment': len(payment_access),
                'operations': len(operations_access),
                'other': len(unauthorized)
            },
            'potentially_unauthorized': unauthorized,
            'unique_accessors': len(set(a['accessor'] for a in accesses)),
            'compliance_status': 'COMPLIANT' if len(unauthorized) == 0 else 'REVIEW REQUIRED'
        }
    
    def generate_breach_assessment_report(self) -> dict:
        """
        ประเมินว่ามี data breach หรือไม่ (HIPAA Breach Notification Rule)
        ต้อง notify ผู้ป่วยภายใน 60 วัน ถ้ามี breach
        """
        
        # ตรวจสอบ potential breaches
        pipeline_incidents = self.audit_db.get_security_incidents()
        potential_breaches = []
        
        for incident in pipeline_incidents:
            if self._is_potential_phi_breach(incident):
                potential_breaches.append({
                    'incident_id': incident['id'],
                    'date': incident['date'],
                    'description': incident['description'],
                    'affected_phi': incident.get('affected_data_types', []),
                    'notification_required': True,
                    'notification_deadline': (
                        datetime.fromisoformat(incident['date']) + timedelta(days=60)
                    ).isoformat()
                })
        
        return {
            'assessment_date': datetime.now().isoformat(),
            'potential_breaches': potential_breaches,
            'notification_required': len(potential_breaches) > 0,
            'hipaa_compliant': len(potential_breaches) == 0
        }
    
    def _is_potential_phi_breach(self, incident: dict) -> bool:
        """ประเมินว่า incident เป็น PHI breach หรือไม่"""
        phi_related_keywords = [
            'patient', 'phi', 'health_record', 'medical', 
            'prescription', 'diagnosis', 'personal_data'
        ]
        description = incident.get('description', '').lower()
        return any(keyword in description for keyword in phi_related_keywords)
```

---

## 95.7 แบบฝึกหัด

### แบบฝึกหัดที่ 1: PHI Scanner

**โจทย์:** สร้าง PHI scanner สำหรับ test fixtures

```python
# TODO: Implement PHI scanner

class PHIScanner:
    # Thai Healthcare specific patterns
    THAI_PHI_PATTERNS = {
        'hn': r'HN\d{6,8}',         # Hospital Number
        'citizen_id': r'\d-\d{4}-\d{5}-\d{2}-\d',  # Thai ID
        'hn_format2': r'AN\d{6,8}',  # Admission Number
    }
    
    def scan_file(self, filepath: str) -> list:
        """TODO: Scan file สำหรับ PHI patterns"""
        pass
    
    def scan_directory(self, dirpath: str) -> dict:
        """TODO: Scan directory recursively"""
        pass
    
    def generate_report(self, scan_results: dict) -> str:
        """TODO: Generate HTML report"""
        pass

# Test
scanner = PHIScanner()
# TODO: Test กับ sample data
```

### แบบฝึกหัดที่ 2: Clinical Change Request

**โจทย์:** สร้าง Clinical Change Request สำหรับ "เพิ่ม drug interaction alert"

```python
# TODO: สร้าง ClinicalChangeRequest
# Service: prescription-service
# Change: เพิ่ม real-time drug interaction checking
# Impact: HIGH (เกี่ยวกับ clinical decision)
# Affected workflows: Prescription workflow, Doctor's order

change = ClinicalChangeRequest(
    service="prescription-service",
    change_description="""
    TODO: เขียนคำอธิบาย change ที่ละเอียด:
    - สิ่งที่เปลี่ยน
    - วิธีที่ระบบทำงาน
    - เหตุผลที่ต้องเปลี่ยน
    """,
    clinical_impact=ClinicalImpact.HIGH,
    affected_workflows=[],  # TODO: list workflows
    tested_by="",  # TODO: กำหนด tester
    medical_reviewer=""  # TODO: กำหนด medical reviewer
)

validator = ClinicalChangeValidator()
result = validator.validate_change(change)
print(validator.generate_change_documentation(change))
```

### แบบฝึกหัดที่ 3: PDPA Compliance Pipeline

**โจทย์:** เพิ่ม PDPA compliance check ใน pipeline

```yaml
# TODO: เพิ่ม job ใน .github/workflows/ehr-pipeline.yml

pdpa-compliance:
  name: PDPA Compliance Check
  runs-on: ubuntu-latest
  needs: test
  steps:
    - uses: actions/checkout@v4
    
    # TODO: เพิ่ม steps สำหรับ:
    # 1. ตรวจสอบ consent management implementation
    # 2. ตรวจสอบ data retention policies ใน code
    # 3. ตรวจสอบ data export capability (Right of Access)
    # 4. ตรวจสอบ data deletion capability (Right to Erasure)
    # 5. ตรวจสอบ audit logging completeness
    
    - name: Check consent management
      run: |
        # TODO: implement
        echo "Check consent management"
        
    - name: Verify data retention
      run: |
        # TODO: implement
        echo "Verify data retention policies"
```

---

## สรุป

Healthcare CI/CD ต้องสมดุลระหว่าง:

1. **Speed** - Deliver improvements เร็ว
2. **Safety** - Patient safety ต้องมาก่อน
3. **Compliance** - PDPA, HIPAA, กฎระเบียบกระทรวงสาธารณสุข
4. **Privacy** - PHI protection ทุกขั้นตอน

Key takeaways:
- **PHI never leaves protected environment** — test data ต้องเป็น synthetic เสมอ
- **Medical review required** สำหรับ clinical system changes
- **Audit trail ทุก PHI access** — Who, When, Why
- **Encryption at rest AND in transit** สำหรับทุก PHI
- **Clinical impact drives deployment window** — ไม่ deploy ช่วง peak clinical hours

---

**ต่อไป:** Part 96 - Case Study: Gaming Platform CI/CD

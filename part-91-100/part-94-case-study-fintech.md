# Part 94: Case Study — FinTech CI/CD

## บทนำ

FinTech มีความท้าทายพิเศษสำหรับ CI/CD เนื่องจากต้องปฏิบัติตามกฎระเบียบที่เข้มงวด มี zero tolerance สำหรับ errors ทางการเงิน และต้องการ audit trail ที่สมบูรณ์ บทนี้ศึกษา case study ของ **"PayFast Thailand"** — fintech startup ที่ให้บริการ digital payment และ lending

---

## 94.1 Context และ Requirements

### ข้อมูลองค์กร

```yaml
organization:
  name: "PayFast Thailand"
  type: "Payment Service Provider + Digital Lending"
  scale:
    users: 2_000_000
    daily_transactions: 500_000
    transaction_volume_thb: 5_000_000_000  # ฿5 พันล้าน/เดือน
    
  regulatory_framework:
    - "ธปท. (Bank of Thailand) - Payment System License"
    - "SEC - Securities & Exchange Commission (lending products)"
    - "PCI-DSS Level 1 (ปริมาณธุรกรรม)"
    - "ISO 27001"
    - "PDPA (Personal Data Protection Act)"
    
  engineering:
    developers: 45
    product_teams: 6
    services: 28
    
  key_systems:
    - "Payment Processing Engine (critical)"
    - "Fraud Detection System (real-time ML)"
    - "Loan Origination System"
    - "Core Banking Integration"
    - "Customer Portal"
    - "Merchant API"
```

### Regulatory Requirements ที่กระทบ CI/CD

```markdown
## กฎระเบียบที่ต้องปฏิบัติตาม

### 1. ธปท. (Bank of Thailand) Requirements
- Change management process ต้องมี formal approval
- Production changes ต้องมี rollback plan
- Critical system changes ต้อง notify ธปท. ล่วงหน้า 30 วัน
- Incident reporting ภายใน 2 ชั่วโมง

### 2. PCI-DSS Level 1
- Requirement 6: Secure software development
  - Code review บังคับสำหรับ payment code
  - SAST/DAST ใน development process
  - Penetration testing รายปี
- Requirement 7: Restrict access to cardholder data
  - Role-based access สำหรับ deployment
  - No developer access to production
- Requirement 10: Log and monitor all access
  - Audit logs ทุก production change
  - Immutable audit trail

### 3. ISO 27001
- Change management process (A.12.1.2)
- Test environments สำหรับ all changes (A.12.6.1)
- Backup ก่อนทุก critical change

### 4. PDPA
- Personal data ต้องไม่อยู่ใน logs โดยไม่จำเป็น
- Data masking ใน non-production environments
- Right to erasure ต้องทำงานได้
```

---

## 94.2 Compliance-First Pipeline Architecture

### Architecture Overview

```
PayFast CI/CD Architecture (Compliance-Focused)
═══════════════════════════════════════════════

Developer Workstation
       │
       │ git push
       ▼
  GitHub Enterprise
  (On-premises + air-gapped)
       │
       │ webhook (internal only)
       ▼
  Jenkins (self-hosted, air-gapped CI)
       │
  ┌────┴─────────────────────────────────┐
  │           CI STAGES                  │
  │  1. Code Quality (SonarQube)        │
  │  2. Security Scan (Checkmarx SAST)  │
  │  3. Dependency Check (Snyk/NVD)     │
  │  4. Unit + Integration Tests        │
  │  5. Container Build + Scan          │
  └────────────────┬─────────────────────┘
                   │
                   │ Artifact to internal registry
                   ▼
            Internal Registry
            (Harbor - air-gapped)
                   │
           ┌───────┴───────┐
           │               │
           ▼               ▼
       DEV/QA         STAGING
     (Auto-deploy)  (Auto-deploy)
           │               │
           │               │ UAT Sign-off
           │               │ Security Review
           │               │ Compliance Check
           │               ▼
           │        Change Advisory Board
           │        (CAB) Approval
           │               │
           └───────────────┘
                           │
                           ▼
                    PRODUCTION
               (Manual deployment by
                Ops team only, not Dev)
```

### Network Segregation

```
Network Zones สำหรับ CI/CD
══════════════════════════

Zone A: Developer Network
├── Source code repositories
├── CI/CD pipeline trigger
└── ไม่มี access ไปยัง production

Zone B: Build Zone
├── CI runners (isolated VMs)
├── Artifact scanning tools
└── Internal package registry

Zone C: Non-Production
├── Dev/QA environments
├── Staging environment
└── Test databases (masked data)

Zone D: Production (Restricted)
├── Production workloads
├── Production databases (encrypted)
├── Payment processing systems
└── Access: Ops team only + break-glass

Separation Controls:
- Physical network isolation
- Firewall rules - no CI/CD direct access to prod
- VPN + MFA required for Zone D
- All actions logged to immutable SIEM
```

---

## 94.3 Change Management Process

### Change Types และ Approval Levels

```yaml
# change-management-policy.yml

change_types:
  emergency_change:
    description: "Critical security fix หรือ severe incident fix"
    approval_required:
      - "CTO or delegate"
      - "Head of Compliance"
    deployment_window: "Any time"
    notification: "Post-deployment CAB notification ภายใน 2 ชั่วโมง"
    rollback_plan: "Required, tested"
    
  standard_change:
    description: "Pre-approved routine changes (hotfixes, config updates)"
    approval_required:
      - "Engineering Manager"
      - "One Peer Reviewer"
    deployment_window: "Tuesday-Thursday 10:00-16:00 ICT"
    advance_notice: "48 hours"
    
  normal_change:
    description: "Feature releases, significant updates"
    approval_required:
      - "Engineering Manager"
      - "Product Owner"
      - "Security Review"
      - "CAB (Change Advisory Board)"
    deployment_window: "Tuesday 10:00-14:00 ICT"
    advance_notice: "5 business days"
    rollback_plan: "Required, tested, approved"
    notification: "ธปท. notification ถ้า critical system"
    
  major_change:
    description: "Architecture changes, new systems, critical system changes"
    approval_required:
      - "Engineering Director"
      - "CISO"
      - "Head of Operations"
      - "CAB"
      - "ธปท. notification (30 days)"
    deployment_window: "Scheduled maintenance window"
    advance_notice: "30+ days"
```

### CAB (Change Advisory Board) Automation

```python
# cab_automation.py
# Automate CAB workflow สำหรับ CI/CD

import json
from datetime import datetime, date
from enum import Enum
from dataclasses import dataclass, field
from typing import List, Optional

class ChangeType(Enum):
    EMERGENCY = "emergency"
    STANDARD = "standard"
    NORMAL = "normal"
    MAJOR = "major"

class ApprovalStatus(Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    EXPIRED = "expired"

@dataclass
class ChangeRequest:
    id: str
    title: str
    description: str
    change_type: ChangeType
    requester: str
    service: str
    image_tag: str
    planned_deployment: datetime
    rollback_plan: str
    test_evidence: str
    approvals: dict = field(default_factory=dict)
    created_at: datetime = field(default_factory=datetime.now)
    
    @property
    def is_approved(self) -> bool:
        required = self.get_required_approvers()
        return all(
            self.approvals.get(approver) == ApprovalStatus.APPROVED
            for approver in required
        )
    
    def get_required_approvers(self) -> List[str]:
        approver_map = {
            ChangeType.EMERGENCY: ['cto', 'head_compliance'],
            ChangeType.STANDARD: ['eng_manager', 'peer_reviewer'],
            ChangeType.NORMAL: ['eng_manager', 'product_owner', 'security_team', 'cab'],
            ChangeType.MAJOR: ['eng_director', 'ciso', 'head_ops', 'cab']
        }
        return approver_map[self.change_type]

class CABWorkflow:
    def __init__(self, jira_client, slack_client, email_client):
        self.jira = jira_client
        self.slack = slack_client
        self.email = email_client
        self.pending_changes = {}
    
    def submit_change_request(self, change: ChangeRequest) -> str:
        """Submit change request และ notify approvers"""
        
        # สร้าง Jira ticket
        ticket_id = self.jira.create_issue({
            'project': 'CAB',
            'issuetype': 'Change Request',
            'summary': f"[{change.change_type.value.upper()}] {change.title}",
            'description': self.format_cr_description(change),
            'priority': 'High' if change.change_type == ChangeType.EMERGENCY else 'Medium'
        })
        
        change.id = ticket_id
        self.pending_changes[ticket_id] = change
        
        # Notify approvers
        for approver in change.get_required_approvers():
            self.notify_approver(approver, change)
        
        # Post to CAB channel
        self.slack.post_message(
            channel='#change-advisory-board',
            message=self.format_cab_message(change)
        )
        
        return ticket_id
    
    def approve_change(self, change_id: str, approver: str, notes: str = "") -> bool:
        """บันทึก approval"""
        change = self.pending_changes.get(change_id)
        if not change:
            raise ValueError(f"Change {change_id} not found")
        
        if approver not in change.get_required_approvers():
            raise ValueError(f"{approver} is not a required approver for this change")
        
        change.approvals[approver] = ApprovalStatus.APPROVED
        
        # Update Jira
        self.jira.add_comment(change_id, f"Approved by {approver}: {notes}")
        
        # ตรวจสอบว่า fully approved หรือยัง
        if change.is_approved:
            self.on_change_fully_approved(change)
        
        return change.is_approved
    
    def on_change_fully_approved(self, change: ChangeRequest):
        """เมื่อ change ได้รับ approval ครบทุกคน"""
        self.slack.post_message(
            channel='#deployments',
            message=f"✅ Change {change.id} fully approved!\n"
                   f"Service: {change.service}\n"
                   f"Planned: {change.planned_deployment.strftime('%Y-%m-%d %H:%M ICT')}\n"
                   f"Requester: {change.requester}"
        )
        
        # Update Jira status
        self.jira.transition_issue(change.id, 'Approved')
        
        # Schedule deployment notification
        self.schedule_deployment_reminder(change)
    
    def format_cr_description(self, change: ChangeRequest) -> str:
        return f"""
## Change Summary
- Service: {change.service}
- Type: {change.change_type.value}
- Image: {change.image_tag}
- Planned Deployment: {change.planned_deployment.strftime('%Y-%m-%d %H:%M ICT')}

## Change Description
{change.description}

## Rollback Plan
{change.rollback_plan}

## Test Evidence
{change.test_evidence}

## Required Approvers
{', '.join(change.get_required_approvers())}
"""
    
    def generate_compliance_report(self, start_date: date, end_date: date) -> dict:
        """สร้าง compliance report สำหรับ audit"""
        changes_in_period = [
            c for c in self.pending_changes.values()
            if start_date <= c.created_at.date() <= end_date
        ]
        
        return {
            'period': f"{start_date} to {end_date}",
            'total_changes': len(changes_in_period),
            'by_type': {
                change_type.value: len([c for c in changes_in_period if c.change_type == change_type])
                for change_type in ChangeType
            },
            'approval_compliance': len([c for c in changes_in_period if c.is_approved]) / len(changes_in_period) * 100 if changes_in_period else 100,
            'changes': [
                {
                    'id': c.id,
                    'service': c.service,
                    'type': c.change_type.value,
                    'approved': c.is_approved,
                    'date': c.created_at.isoformat()
                }
                for c in changes_in_period
            ]
        }
```

---

## 94.4 Security Controls ใน Pipeline

### Comprehensive Security Pipeline

```yaml
# jenkins/Jenkinsfile.payfast
// Jenkins pipeline พร้อม compliance controls

pipeline {
    agent none
    
    options {
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
        ansiColor('xterm')
    }
    
    environment {
        SERVICE_NAME = "${env.JOB_BASE_NAME}"
        REGISTRY = 'harbor.payfast.internal'
        SONAR_HOST = 'https://sonar.payfast.internal'
        VAULT_ADDR = 'https://vault.payfast.internal'
        
        // Compliance settings
        CHANGE_TICKET = "${params.CHANGE_TICKET_ID}"
        REQUIRE_CAB_APPROVAL = true
    }
    
    parameters {
        string(name: 'CHANGE_TICKET_ID', description: 'CAB Ticket ID (required for production)')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'production'])
        booleanParam(name: 'EMERGENCY_CHANGE', defaultValue: false)
    }

    stages {
        stage('Compliance Pre-Check') {
            agent { label 'compliance-runner' }
            steps {
                script {
                    if (params.ENVIRONMENT == 'production') {
                        // ตรวจสอบว่ามี approved CAB ticket
                        def cabStatus = sh(
                            script: "python3 scripts/check_cab_approval.py ${params.CHANGE_TICKET_ID}",
                            returnStdout: true
                        ).trim()
                        
                        if (cabStatus != 'APPROVED' && !params.EMERGENCY_CHANGE) {
                            error("Production deployment requires approved CAB ticket. Current status: ${cabStatus}")
                        }
                        
                        if (params.EMERGENCY_CHANGE) {
                            // Emergency: ต้อง notify post-deployment
                            slackSend(
                                channel: '#emergency-changes',
                                message: "⚠️ EMERGENCY CHANGE initiated for ${SERVICE_NAME} by ${env.BUILD_USER}"
                            )
                        }
                    }
                }
            }
        }
        
        stage('Code Analysis') {
            agent { label 'build-runner' }
            parallel {
                stage('SAST - Checkmarx') {
                    steps {
                        // Checkmarx สำหรับ financial code
                        step([$class: 'CxScanBuilder',
                            credentialsId: 'checkmarx-creds',
                            projectName: "${SERVICE_NAME}",
                            preset: '36',  // PCI-DSS preset
                            generatePdfReport: true,
                            failBuildOnNewResults: true,
                            failBuildOnNewSeverity: 'HIGH'
                        ])
                    }
                }
                
                stage('Secret Detection') {
                    steps {
                        sh '''
                            # ตรวจสอบ secrets ที่ specific สำหรับ fintech
                            trufflehog filesystem . \
                                --regex \
                                --rules config/trufflehog-rules.yml \
                                --fail
                        '''
                    }
                }
                
                stage('Dependency Security') {
                    steps {
                        sh '''
                            # OWASP Dependency Check
                            dependency-check.sh \
                                --project ${SERVICE_NAME} \
                                --scan . \
                                --format JSON \
                                --out reports/ \
                                --failOnCVSS 7
                            
                            # Check for known exploited vulnerabilities (CISA KEV)
                            python3 scripts/check-kev.py reports/dependency-check-report.json
                        '''
                    }
                }
            }
        }
        
        stage('Test') {
            agent { label 'test-runner' }
            stages {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            # ต้องการ 80% coverage สำหรับ payment code
                            jest --coverage \
                                --coverageThreshold='{"global":{"lines":80,"branches":75}}' \
                                --testPathPattern="unit"
                        '''
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        sh '''
                            # Integration tests กับ mock external services
                            jest --testPathPattern="integration" \
                                --runInBand \
                                --forceExit
                        '''
                    }
                }
                
                stage('Financial Logic Tests') {
                    steps {
                        sh '''
                            # Tests เฉพาะสำหรับ financial calculations
                            # Zero tolerance สำหรับ rounding errors
                            jest --testPathPattern="financial" \
                                --coverageThreshold='{"global":{"lines":100,"branches":100}}'
                        '''
                    }
                }
                
                stage('Compliance Tests') {
                    steps {
                        sh '''
                            # Tests สำหรับ regulatory requirements
                            pytest tests/compliance/ \
                                --html=compliance-report.html \
                                --self-contained-html \
                                -v
                        '''
                    }
                }
            }
        }
        
        stage('Build & Scan Container') {
            agent { label 'docker-runner' }
            steps {
                sh '''
                    # Build image
                    docker build \
                        --build-arg VERSION=${BUILD_NUMBER} \
                        --build-arg GIT_COMMIT=${GIT_COMMIT} \
                        -t ${REGISTRY}/${SERVICE_NAME}:${GIT_COMMIT} .
                    
                    # Scan ด้วย Trivy
                    trivy image \
                        --exit-code 1 \
                        --severity CRITICAL \
                        --no-progress \
                        ${REGISTRY}/${SERVICE_NAME}:${GIT_COMMIT}
                    
                    # Scan ด้วย Snyk
                    snyk container test \
                        ${REGISTRY}/${SERVICE_NAME}:${GIT_COMMIT} \
                        --severity-threshold=high
                    
                    # Sign image (สำหรับ integrity)
                    cosign sign \
                        --key vault://cosign-key \
                        ${REGISTRY}/${SERVICE_NAME}:${GIT_COMMIT}
                    
                    # Push
                    docker push ${REGISTRY}/${SERVICE_NAME}:${GIT_COMMIT}
                '''
            }
        }
        
        stage('Deploy') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    def environment = params.ENVIRONMENT
                    
                    if (environment == 'production') {
                        // Production requires additional verification
                        input message: "Confirm production deployment of ${SERVICE_NAME}?",
                              ok: "Deploy to Production",
                              submitterParameter: "APPROVED_BY"
                        
                        // บันทึกว่าใครอนุมัติ (audit trail)
                        sh """
                            python3 scripts/record_deployment.py \
                                --service ${SERVICE_NAME} \
                                --image ${REGISTRY}/${SERVICE_NAME}:${GIT_COMMIT} \
                                --environment production \
                                --approved-by ${env.APPROVED_BY} \
                                --change-ticket ${params.CHANGE_TICKET_ID}
                        """
                    }
                    
                    // Deploy ด้วย Ansible (เพราะ Kubernetes access restricted)
                    sh """
                        ansible-playbook deploy.yml \
                            -e environment=${environment} \
                            -e service=${SERVICE_NAME} \
                            -e image_tag=${GIT_COMMIT} \
                            --vault-password-file /etc/vault/ansible-vault-pass
                    """
                }
            }
        }
        
        stage('Post-Deployment Verification') {
            when { expression { params.ENVIRONMENT == 'production' } }
            steps {
                sh '''
                    # Smoke tests
                    pytest tests/smoke/ \
                        --base-url=https://api.payfast.th \
                        -v
                    
                    # Financial transaction tests (small amounts)
                    pytest tests/post_deploy/ \
                        --run-financial-checks \
                        -v
                    
                    # Update change ticket status
                    python3 scripts/update_cab_ticket.py \
                        --ticket-id ${CHANGE_TICKET} \
                        --status "Deployed Successfully" \
                        --deployed-by ${env.BUILD_USER}
                '''
            }
        }
    }
    
    post {
        always {
            // Archive all compliance artifacts
            archiveArtifacts artifacts: 'reports/**/*', fingerprint: true
            
            // Send to SIEM สำหรับ audit
            sh '''
                python3 scripts/send_to_siem.py \
                    --build-id ${BUILD_ID} \
                    --status ${currentBuild.result}
            '''
        }
        
        failure {
            // Notify ทุก failure
            script {
                slackSend(
                    channel: '#cicd-alerts',
                    color: 'danger',
                    message: "❌ Build FAILED: ${SERVICE_NAME} #${BUILD_NUMBER}\n${env.BUILD_URL}"
                )
                
                if (params.ENVIRONMENT == 'production') {
                    // Production failure = urgent notification
                    pagerdutyTrigger(
                        integrationKey: env.PAGERDUTY_KEY,
                        incSummary: "Production deployment failed: ${SERVICE_NAME}",
                        incSeverity: 'critical'
                    )
                }
            }
        }
    }
}
```

---

## 94.5 Audit Trail Implementation

### Immutable Audit Log

```python
# audit_logger.py
# Immutable audit logging สำหรับ CI/CD events

import hashlib
import json
import hmac
import time
from datetime import datetime
from typing import Dict, Any

class ImmutableAuditLogger:
    """
    Audit logger ที่ไม่สามารถแก้ไขได้ย้อนหลัง
    ใช้ hash chain เพื่อ verify integrity
    """
    
    def __init__(self, siem_endpoint: str, signing_key: str):
        self.siem_endpoint = siem_endpoint
        self.signing_key = signing_key
        self.previous_hash = "0" * 64  # Genesis hash
    
    def log_event(self, event_type: str, data: Dict[str, Any], actor: str) -> str:
        """บันทึก audit event พร้อม hash chain"""
        
        event = {
            'event_id': self._generate_event_id(),
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': event_type,
            'actor': actor,
            'data': data,
            'previous_hash': self.previous_hash
        }
        
        # สร้าง hash ของ event นี้
        event_hash = self._compute_hash(event)
        event['hash'] = event_hash
        
        # Sign event
        event['signature'] = self._sign_event(event)
        
        # Update chain
        self.previous_hash = event_hash
        
        # ส่งไปยัง immutable SIEM
        self._send_to_siem(event)
        
        return event_hash
    
    def _compute_hash(self, event: dict) -> str:
        """คำนวณ SHA-256 hash ของ event"""
        event_str = json.dumps(event, sort_keys=True, default=str)
        return hashlib.sha256(event_str.encode()).hexdigest()
    
    def _sign_event(self, event: dict) -> str:
        """Sign event ด้วย HMAC"""
        event_str = json.dumps(event, sort_keys=True, default=str)
        return hmac.new(
            self.signing_key.encode(),
            event_str.encode(),
            hashlib.sha256
        ).hexdigest()
    
    def _generate_event_id(self) -> str:
        """สร้าง unique event ID"""
        return f"evt_{int(time.time() * 1000)}_{hashlib.md5(str(time.time()).encode()).hexdigest()[:8]}"
    
    def _send_to_siem(self, event: dict):
        """ส่ง event ไปยัง SIEM (เช่น Splunk, IBM QRadar)"""
        import requests
        requests.post(
            self.siem_endpoint,
            json=event,
            headers={'X-API-Key': self.signing_key},
            timeout=5
        )
    
    def verify_chain_integrity(self, events: list) -> bool:
        """ตรวจสอบ integrity ของ audit chain"""
        prev_hash = "0" * 64
        
        for event in events:
            if event['previous_hash'] != prev_hash:
                return False
            
            # Verify hash
            stored_hash = event.pop('hash')
            computed_hash = self._compute_hash(event)
            event['hash'] = stored_hash
            
            if stored_hash != computed_hash:
                return False
            
            prev_hash = stored_hash
        
        return True

# Deployment Audit Events
class DeploymentAuditor:
    def __init__(self, audit_logger: ImmutableAuditLogger):
        self.logger = audit_logger
    
    def log_deployment_started(self, deployment_info: dict, actor: str):
        return self.logger.log_event(
            event_type='DEPLOYMENT_STARTED',
            data={
                'service': deployment_info['service'],
                'environment': deployment_info['environment'],
                'image_tag': deployment_info['image_tag'],
                'change_ticket': deployment_info.get('change_ticket'),
                'git_commit': deployment_info['git_commit'],
                'git_author': deployment_info['git_author']
            },
            actor=actor
        )
    
    def log_deployment_completed(self, deployment_id: str, service: str, success: bool, actor: str):
        return self.logger.log_event(
            event_type='DEPLOYMENT_COMPLETED',
            data={
                'deployment_id': deployment_id,
                'service': service,
                'success': success,
                'duration_seconds': deployment_info.get('duration')
            },
            actor=actor
        )
    
    def log_production_access(self, user: str, reason: str, resource: str):
        """Log ทุกครั้งที่มีคนเข้าถึง production"""
        return self.logger.log_event(
            event_type='PRODUCTION_ACCESS',
            data={
                'resource': resource,
                'reason': reason,
                'ip_address': get_client_ip(),
                'session_id': get_session_id()
            },
            actor=user
        )
    
    def log_rollback(self, service: str, from_version: str, to_version: str, reason: str, actor: str):
        return self.logger.log_event(
            event_type='ROLLBACK_EXECUTED',
            data={
                'service': service,
                'from_version': from_version,
                'to_version': to_version,
                'reason': reason,
                'emergency': 'EMERGENCY' in reason.upper()
            },
            actor=actor
        )
```

---

## 94.6 Zero-Downtime Deployment สำหรับ Financial Systems

### Payment Processing Deployment Strategy

```python
# payment_deployment.py
# Zero-downtime deployment สำหรับ payment processing

import time
import requests
from enum import Enum

class PaymentDeploymentManager:
    """
    Special deployment manager สำหรับ payment processing services
    ต้องระวังเป็นพิเศษเพราะมี in-flight transactions
    """
    
    def __init__(self, service_name: str, k8s_client, metrics_client):
        self.service_name = service_name
        self.k8s = k8s_client
        self.metrics = metrics_client
    
    def safe_deploy(self, new_image: str, change_ticket: str) -> bool:
        """
        Deploy อย่างปลอดภัยโดย:
        1. ตรวจสอบ in-flight transactions
        2. Graceful shutdown
        3. Rolling update
        4. Verification
        """
        
        print(f"🏦 Starting safe deployment for {self.service_name}")
        
        # 1. ตรวจสอบ current load
        load_info = self.check_current_load()
        if load_info['tps'] > 1000:
            print(f"⚠️ High TPS detected ({load_info['tps']}). Waiting for lower traffic...")
            self.wait_for_low_traffic()
        
        # 2. Enable deployment mode (drain new requests)
        self.enable_deployment_mode()
        
        # 3. รอให้ in-flight transactions complete
        print("⏳ Waiting for in-flight transactions to complete...")
        self.wait_for_in_flight_completion(timeout=300)
        
        # 4. Start rolling update
        success = self.rolling_update(new_image)
        
        # 5. Disable deployment mode
        self.disable_deployment_mode()
        
        if success:
            # 6. Verify payment processing works
            self.verify_payment_processing()
            print("✅ Deployment completed successfully")
        else:
            print("❌ Deployment failed, initiating rollback")
            self.rollback()
        
        return success
    
    def check_current_load(self) -> dict:
        """ตรวจสอบ current traffic load"""
        tps = self.metrics.query('rate(payment_transactions_total[1m])')
        error_rate = self.metrics.query('rate(payment_errors_total[5m]) / rate(payment_transactions_total[5m])')
        
        return {
            'tps': tps,
            'error_rate': error_rate,
            'safe_to_deploy': tps < 500 and error_rate < 0.001
        }
    
    def wait_for_low_traffic(self, target_tps: int = 500, timeout: int = 3600):
        """รอให้ traffic ลดลงก่อน deploy"""
        start = time.time()
        while time.time() - start < timeout:
            load = self.check_current_load()
            if load['tps'] < target_tps:
                return
            print(f"   Current TPS: {load['tps']}, waiting...")
            time.sleep(60)
        
        raise TimeoutError(f"Traffic did not drop below {target_tps} TPS within {timeout}s")
    
    def wait_for_in_flight_completion(self, timeout: int = 300):
        """รอให้ transactions ที่กำลังประมวลผลเสร็จ"""
        start = time.time()
        while time.time() - start < timeout:
            in_flight = self.metrics.query('payment_in_flight_transactions')
            if in_flight == 0:
                print("   All in-flight transactions completed")
                return
            print(f"   {in_flight} transactions still in-flight...")
            time.sleep(5)
        
        raise TimeoutError("In-flight transactions did not complete")
    
    def enable_deployment_mode(self):
        """เปิด deployment mode: หยุดรับ new transactions"""
        # Update ConfigMap
        self.k8s.patch_configmap(
            'payment-config',
            {'data': {'DEPLOYMENT_MODE': 'true'}}
        )
        print("🔒 Deployment mode enabled (new transactions paused)")
    
    def disable_deployment_mode(self):
        """ปิด deployment mode: กลับมารับ transactions"""
        self.k8s.patch_configmap(
            'payment-config',
            {'data': {'DEPLOYMENT_MODE': 'false'}}
        )
        print("🔓 Deployment mode disabled (transactions resumed)")
    
    def rolling_update(self, new_image: str) -> bool:
        """Rolling update ทีละ pod"""
        deployment = self.k8s.get_deployment(self.service_name)
        total_replicas = deployment.spec.replicas
        
        # Update image
        self.k8s.set_image(
            self.service_name,
            f'{self.service_name}={new_image}'
        )
        
        # Monitor rollout
        for i in range(60):  # รอสูงสุด 10 นาที
            status = self.k8s.get_rollout_status(self.service_name)
            
            if status.updated_replicas == total_replicas and status.available_replicas == total_replicas:
                print(f"✅ All {total_replicas} replicas updated successfully")
                return True
            
            print(f"   Updated: {status.updated_replicas}/{total_replicas} replicas")
            
            if status.unavailable_replicas > total_replicas * 0.1:  # > 10% unavailable
                print("❌ Too many unavailable replicas")
                return False
            
            time.sleep(10)
        
        return False
    
    def verify_payment_processing(self):
        """ทดสอบว่า payment processing ทำงานได้หลัง deploy"""
        # ส่ง test payment (฿1 บาท ไปยัง test account)
        response = requests.post(
            f'https://api-internal.payfast.th/v1/transactions/test',
            json={
                'amount': 1,
                'currency': 'THB',
                'test': True
            },
            headers={'X-API-Key': 'deployment-verification-key'}
        )
        
        assert response.status_code == 200, f"Payment test failed: {response.text}"
        assert response.json()['status'] == 'success', "Payment test transaction failed"
        
        print("✅ Payment processing verification passed")
```

---

## 94.7 Compliance Reporting

### Automated Compliance Dashboard

```python
# compliance_dashboard.py
# สร้าง compliance report สำหรับ auditors

from datetime import datetime, date, timedelta
import pandas as pd

class ComplianceDashboard:
    def __init__(self, audit_db, pipeline_db):
        self.audit_db = audit_db
        self.pipeline_db = pipeline_db
    
    def generate_monthly_compliance_report(self, year: int, month: int) -> dict:
        """สร้าง monthly report สำหรับ audit committee"""
        
        start = date(year, month, 1)
        end = date(year, month + 1, 1) if month < 12 else date(year + 1, 1, 1)
        
        return {
            'period': f"{start.strftime('%B %Y')}",
            'generated_at': datetime.now().isoformat(),
            'generated_by': 'Automated Compliance System',
            
            'change_management': self.analyze_change_management(start, end),
            'security_scan_results': self.analyze_security_scans(start, end),
            'deployment_statistics': self.analyze_deployments(start, end),
            'access_control': self.analyze_access_control(start, end),
            'incidents': self.analyze_incidents(start, end),
            'audit_trail_integrity': self.verify_audit_integrity(start, end)
        }
    
    def analyze_change_management(self, start: date, end: date) -> dict:
        changes = self.audit_db.query_events(
            event_type='DEPLOYMENT_COMPLETED',
            start_date=start,
            end_date=end
        )
        
        approved_changes = [c for c in changes if c.get('change_ticket')]
        emergency_changes = [c for c in changes if c.get('emergency')]
        
        compliance_rate = len(approved_changes) / len(changes) * 100 if changes else 100
        
        return {
            'total_changes': len(changes),
            'approved_changes': len(approved_changes),
            'emergency_changes': len(emergency_changes),
            'compliance_rate': round(compliance_rate, 2),
            'compliant': compliance_rate >= 99,  # ต้องการ 99% compliance
            'non_compliant_changes': [
                c for c in changes if not c.get('change_ticket')
            ]
        }
    
    def analyze_security_scans(self, start: date, end: date) -> dict:
        pipelines = self.pipeline_db.get_pipelines(start, end)
        
        with_sast = [p for p in pipelines if p.get('sast_passed')]
        with_container_scan = [p for p in pipelines if p.get('container_scan_passed')]
        critical_vulns_found = [p for p in pipelines if p.get('critical_vulns', 0) > 0]
        
        return {
            'total_pipelines': len(pipelines),
            'sast_coverage': round(len(with_sast) / len(pipelines) * 100, 2) if pipelines else 0,
            'container_scan_coverage': round(len(with_container_scan) / len(pipelines) * 100, 2) if pipelines else 0,
            'pipelines_with_critical_vulns': len(critical_vulns_found),
            'pci_dss_compliant': len(with_sast) == len(pipelines) and len(critical_vulns_found) == 0
        }
    
    def format_for_auditors(self, report: dict) -> str:
        """Format report สำหรับ auditors"""
        lines = [
            f"=" * 60,
            f"PAYFAST THAILAND - CI/CD COMPLIANCE REPORT",
            f"Period: {report['period']}",
            f"Generated: {report['generated_at']}",
            f"=" * 60,
            "",
            "1. CHANGE MANAGEMENT COMPLIANCE",
            "-" * 40,
        ]
        
        cm = report['change_management']
        lines.extend([
            f"Total Changes: {cm['total_changes']}",
            f"Changes with Approval: {cm['approved_changes']} ({cm['compliance_rate']}%)",
            f"Emergency Changes: {cm['emergency_changes']}",
            f"Status: {'✅ COMPLIANT' if cm['compliant'] else '❌ NON-COMPLIANT'}",
            ""
        ])
        
        return "\n".join(lines)
```

---

## 94.8 แบบฝึกหัด

### แบบฝึกหัดที่ 1: CAB Workflow

**โจทย์:** สร้าง Change Request สำหรับ "deploy payment service v2.5"

```python
# TODO: สร้าง ChangeRequest object
from datetime import datetime

change = ChangeRequest(
    id="",  # จะกำหนดโดย CAB system
    title="",  # TODO: กำหนดชื่อที่ชัดเจน
    description="""
    TODO: อธิบาย:
    - สิ่งที่เปลี่ยนแปลง
    - เหตุผลที่ต้องเปลี่ยน
    - Impact ที่คาดหวัง
    """,
    change_type=ChangeType.NORMAL,  # TODO: เลือก type ที่เหมาะสม
    requester="your_name",
    service="payment-service",
    image_tag="payment-service:v2.5.0",
    planned_deployment=datetime(2024, 3, 19, 10, 0),  # Tuesday 10:00 ICT
    rollback_plan="""
    TODO: เขียน rollback plan ที่ชัดเจน:
    1. ขั้นตอนการ rollback
    2. เวลาที่ต้องใช้
    3. วิธีตรวจสอบว่า rollback สำเร็จ
    """,
    test_evidence="""
    TODO: บอก test evidence:
    - Test results
    - Staging test results
    - Performance test results
    """
)
```

### แบบฝึกหัดที่ 2: Audit Trail

**โจทย์:** สร้าง audit trail สำหรับ deployment event

```python
# TODO: Implement audit logging
logger = ImmutableAuditLogger(
    siem_endpoint="https://siem.payfast.internal/events",
    signing_key="your-signing-key"
)

auditor = DeploymentAuditor(logger)

# TODO: Log deployment events:
# 1. Deployment started
# 2. Deployment completed
# 3. Post-deployment verification
# 4. Change ticket updated

# Bonus: Implement verify_chain_integrity ใน ImmutableAuditLogger
```

### แบบฝึกหัดที่ 3: Zero-Downtime Payment Deployment

**โจทย์:** Mock implementation ของ PaymentDeploymentManager

```python
# TODO: สร้าง mock ของ payment deployment
# Mock metrics, K8s client
# Test:
# 1. Normal deployment (TPS < 500)
# 2. High TPS scenario (ต้องรอก่อน)
# 3. In-flight transaction scenario
# 4. Failed deployment → rollback

class MockMetricsClient:
    def query(self, query: str) -> float:
        # TODO: implement mock
        pass

class MockK8sClient:
    def get_deployment(self, name: str):
        # TODO: implement mock
        pass
    
    def set_image(self, deployment: str, image: str):
        # TODO: implement mock
        pass

# สร้าง integration test
def test_payment_deployment():
    metrics = MockMetricsClient()
    k8s = MockK8sClient()
    
    manager = PaymentDeploymentManager("payment-service", k8s, metrics)
    
    # Test 1: Normal deployment
    result = manager.safe_deploy("payment-service:v2.5.0", "CAB-001")
    assert result == True
    
    # TODO: Test 2 and 3
```

---

## สรุป

FinTech CI/CD มีความท้าทายพิเศษ:

1. **Regulatory Compliance** - ธปท., PCI-DSS, PDPA requirements
2. **Change Management** - CAB process ที่ formal แต่ต้องไม่ช้าเกินไป
3. **Zero-Downtime** - In-flight transactions ต้องได้รับการจัดการ
4. **Audit Trail** - Immutable, verifiable, tamper-evident
5. **Security** - Multi-layer security scanning
6. **Emergency Changes** - Process ที่รองรับ emergency โดยไม่ละทิ้ง accountability

---

**ต่อไป:** Part 95 - Case Study: Healthcare CI/CD Compliance

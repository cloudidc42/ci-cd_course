# Part 82: CI/CD Governance & Compliance at Scale

## บทนำ

ในองค์กรที่อยู่ภายใต้กฎระเบียบ เช่น ธนาคาร ระบบสุขภาพ หรือบริษัทที่จดทะเบียนในตลาดหลักทรัพย์ การทำ CI/CD ไม่ใช่แค่เรื่องของความเร็ว แต่ต้องทำให้ถูกต้องตามกฎหมายและข้อบังคับต่างๆ ด้วย บทนี้จะพาไปเรียนรู้การออกแบบระบบ Governance และ Compliance สำหรับ CI/CD pipeline

## สารบัญ

1. [ทำความเข้าใจ Compliance Framework](#compliance-framework)
2. [Policy Enforcement as Code](#policy-as-code)
3. [Audit Trails ที่สมบูรณ์แบบ](#audit-trails)
4. [SOX Compliance ใน CI/CD](#sox)
5. [PCI-DSS Compliance ใน CI/CD](#pci-dss)
6. [HIPAA Compliance ใน CI/CD](#hipaa)
7. [Compliance Gates ในทุก Stage](#compliance-gates)
8. [Evidence Collection Automation](#evidence-collection)
9. [Regulated Industry Case Studies](#case-studies)
10. [แบบฝึกหัด](#exercises)

---

## 1. ทำความเข้าใจ Compliance Framework {#compliance-framework}

### ทำไม Compliance ถึงสำคัญใน CI/CD?

```
ตัวอย่าง: ธนาคารถูก audit โดย OCC (Office of the Comptroller)
พบว่า:
- ไม่มี change management documentation
- Developer สามารถ deploy โดยตรงไป production
- ไม่มี approval process ที่ documented
- Audit trail ขาดหาย

ผล:
- ปรับเงิน $50 ล้าน
- ต้องแก้ไขภายใน 90 วัน
- Reputation damage
```

### Compliance Framework ที่สำคัญ

| Framework | Industry | ความเกี่ยวข้องกับ CI/CD |
|-----------|----------|----------------------|
| SOX | Public Companies | Change management, audit trail |
| PCI-DSS | Payment Card | Code security, access control |
| HIPAA | Healthcare | PHI protection, audit log |
| GDPR | EU Data | Data handling in pipelines |
| SOC2 | SaaS | Security controls |
| FedRAMP | US Government | Stringent controls |
| ISO 27001 | General | ISMS integration |

### Compliance vs Security vs Governance

```
┌─────────────────────────────────────────────────────┐
│                    GOVERNANCE                        │
│   (What we decide to do and how we make decisions)   │
│                                                      │
│  ┌─────────────────────┐  ┌──────────────────────┐  │
│  │    COMPLIANCE       │  │     SECURITY         │  │
│  │  (Meeting external  │  │  (Protecting our     │  │
│  │   requirements)     │  │   systems & data)    │  │
│  └─────────────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────┘

ทั้ง 3 ต้องทำงานร่วมกัน แต่มีวัตถุประสงค์ต่างกัน
```

---

## 2. Policy Enforcement as Code {#policy-as-code}

### แนวคิด Policy as Code

แทนที่จะเขียน policy เป็นเอกสาร เราเขียนเป็น code ที่ enforce ได้:

```
ก่อนหน้า (Policy เป็นเอกสาร):
1. เขียน policy document
2. ส่งให้ทีม Developer อ่าน
3. Developer อาจจำหรืออาจลืม
4. ไม่มีการ enforce อัตโนมัติ
5. Audit ต้องตรวจด้วยมือ

หลังจากใช้ Policy as Code:
1. เขียน policy เป็น code (OPA/Rego)
2. CI/CD pipeline ตรวจสอบอัตโนมัติ
3. ถ้าไม่ผ่าน → build fail ทันที
4. 100% enforcement
5. Audit trail อัตโนมัติ
```

### Open Policy Agent (OPA) สำหรับ CI/CD

```rego
# policies/cicd/deployment-policy.rego

package cicd.deployment

import future.keywords.if
import future.keywords.in

# Main decision: อนุญาต deploy หรือไม่
default allow = false

allow if {
    count(deny) == 0
}

# รวบรวมทุก violation
deny[msg] {
    check := checks[_]
    not check.passed
    msg := check.message
}

checks[check] {
    check := {
        "name": "requires_approval",
        "passed": has_required_approvals,
        "message": "Production deployment requires at least 2 approvals"
    }
}

checks[check] {
    check := {
        "name": "security_scan",
        "passed": has_security_scan_passed,
        "message": "Security scan must pass before deployment"
    }
}

checks[check] {
    check := {
        "name": "no_critical_vulnerabilities",
        "passed": no_critical_cvs,
        "message": "No critical vulnerabilities allowed in production"
    }
}

checks[check] {
    check := {
        "name": "change_ticket",
        "passed": has_change_ticket,
        "message": "Must have approved change ticket (SOX requirement)"
    }
}

# Helper rules
has_required_approvals if {
    input.environment == "production"
    count(input.approvals) >= 2
    input.approvals[_].role == "tech-lead"
}

has_required_approvals if {
    input.environment != "production"
    count(input.approvals) >= 1
}

has_security_scan_passed if {
    input.security_scan.status == "passed"
    input.security_scan.timestamp > time.now_ns() - (24 * 60 * 60 * 1000000000)
}

no_critical_cvs if {
    not input.security_scan.vulnerabilities.critical
}

no_critical_cvs if {
    count(input.security_scan.vulnerabilities.critical) == 0
}

has_change_ticket if {
    input.change_ticket != null
    input.change_ticket.status == "approved"
    input.change_ticket.approver_role == "change-advisory-board"
}
```

```yaml
# .github/workflows/policy-check.yml

name: Policy Compliance Check

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  policy-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup OPA
        run: |
          curl -L -o opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
          chmod +x opa
          sudo mv opa /usr/local/bin/
      
      - name: Build Policy Input
        id: build-input
        run: |
          cat > /tmp/policy-input.json << EOF
          {
            "environment": "${{ env.TARGET_ENV }}",
            "service": "${{ env.SERVICE_NAME }}",
            "deployer": "${{ github.actor }}",
            "approvals": ${{ toJson(env.APPROVALS) }},
            "security_scan": {
              "status": "${{ env.SCAN_STATUS }}",
              "timestamp": $(date +%s%N),
              "vulnerabilities": {
                "critical": ${{ env.CRITICAL_CVE_COUNT }},
                "high": ${{ env.HIGH_CVE_COUNT }}
              }
            },
            "change_ticket": {
              "id": "${{ env.CHANGE_TICKET_ID }}",
              "status": "${{ env.CHANGE_TICKET_STATUS }}",
              "approver_role": "${{ env.CHANGE_TICKET_APPROVER_ROLE }}"
            }
          }
          EOF
      
      - name: Evaluate Policy
        run: |
          RESULT=$(opa eval \
            -d policies/cicd/ \
            -i /tmp/policy-input.json \
            'data.cicd.deployment.allow')
          
          if echo "$RESULT" | jq -e '.result[0].expressions[0].value == true' > /dev/null; then
            echo "✅ Policy check passed"
          else
            echo "❌ Policy check failed:"
            opa eval \
              -d policies/cicd/ \
              -i /tmp/policy-input.json \
              'data.cicd.deployment.deny' \
              --format pretty
            exit 1
          fi
```

### Conftest สำหรับ Kubernetes Manifests

```rego
# policy/k8s/security.rego

package kubernetes.security

deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.securityContext.runAsNonRoot
    msg := sprintf("Container '%v' must run as non-root", [container.name])
}

deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    container.image == "latest"
    msg := sprintf("Container '%v' must not use 'latest' tag", [container.name])
}

deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.resources.limits.memory
    msg := sprintf("Container '%v' must have memory limits", [container.name])
}

deny[msg] {
    input.kind == "Deployment"
    container := input.spec.template.spec.containers[_]
    not container.resources.limits.cpu
    msg := sprintf("Container '%v' must have CPU limits", [container.name])
}

warn[msg] {
    input.kind == "Deployment"
    not input.spec.template.spec.affinity
    msg := "Deployment should define affinity rules for high availability"
}
```

```yaml
# ใช้ Conftest ใน pipeline
- name: Check Kubernetes Policies
  run: |
    conftest test k8s/ \
      --policy policy/k8s/ \
      --all-namespaces \
      --output tap
```

---

## 3. Audit Trails ที่สมบูรณ์แบบ {#audit-trails}

### สิ่งที่ต้อง Record ใน Audit Trail

```
ทุก Event ที่เกิดขึ้นใน CI/CD ต้องบันทึก:

1. WHAT: อะไรเกิดขึ้น
   - Action type (build, test, deploy, rollback)
   - Service/component ที่ได้รับผลกระทบ
   - Configuration changes
   
2. WHO: ใครทำ
   - Developer ที่ commit code
   - Approvers ที่ approve
   - System account ที่ execute
   
3. WHEN: เมื่อไหร่
   - Start time, end time
   - Timezone (ต้อง UTC เสมอ)
   
4. WHERE: ที่ไหน
   - Environment (dev/staging/prod)
   - Region/datacenter
   - Cluster/namespace
   
5. WHY: ทำไม
   - Change ticket reference
   - PR/Issue link
   - Business justification
   
6. HOW: ทำอย่างไร
   - Method used
   - Pipeline version
   - Configuration used
```

### Immutable Audit Log Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  Pipeline Execution                       │
│                                                          │
│  Build → Test → Security Scan → Approve → Deploy        │
│    │        │         │             │          │         │
│    └────────┴─────────┴─────────────┴──────────┘         │
│                       │                                   │
└───────────────────────┼───────────────────────────────────┘
                        │ Events
                        ▼
┌──────────────────────────────────────────────────────────┐
│              Audit Event Bus (Kafka)                      │
│                                                          │
│  Topic: cicd.audit.events                                │
│  Retention: 7 years (compliance requirement)             │
│  Replication Factor: 3                                    │
└──────────────────────┬───────────────────────────────────┘
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
┌─────────────────┐   ┌─────────────────────────────────┐
│   SIEM System   │   │    Immutable Audit Storage       │
│   (Splunk/ELK)  │   │                                 │
│                 │   │  - WORM Storage (Write Once)    │
│  Real-time      │   │  - Cryptographic signing        │
│  alerting       │   │  - Tamper detection             │
└─────────────────┘   └─────────────────────────────────┘
```

### Audit Event Schema

```json
{
  "event_id": "evt_01HK2QMXYZ123",
  "event_version": "2.0",
  "event_type": "deployment.production.started",
  "timestamp": "2024-01-15T14:30:00.000Z",
  
  "actor": {
    "type": "human",
    "id": "user_jane_doe",
    "email": "jane.doe@company.com",
    "ip_address": "192.168.1.100",
    "session_id": "sess_abc123"
  },
  
  "subject": {
    "type": "service",
    "id": "payment-service",
    "version": "2.4.1",
    "environment": "production",
    "region": "us-east-1"
  },
  
  "action": {
    "type": "deploy",
    "method": "rolling_update",
    "previous_version": "2.4.0",
    "new_version": "2.4.1"
  },
  
  "context": {
    "change_ticket": "CHG-2024-0042",
    "pipeline_run_id": "run_xyz789",
    "git_commit": "abc123def456",
    "pr_number": 1234,
    "approved_by": [
      {"id": "user_tech_lead_1", "timestamp": "2024-01-15T14:15:00Z"},
      {"id": "user_tech_lead_2", "timestamp": "2024-01-15T14:20:00Z"}
    ]
  },
  
  "compliance": {
    "sox_ticket": "SOX-2024-Q1-042",
    "risk_level": "medium",
    "change_type": "standard"
  },
  
  "signature": {
    "algorithm": "SHA256withRSA",
    "value": "base64-encoded-signature",
    "certificate": "cert-fingerprint"
  }
}
```

### Immutable Log Implementation

```python
# audit/logger.py
import hashlib
import json
import time
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding

class ImmutableAuditLogger:
    """Logger ที่ไม่สามารถแก้ไขหรือลบ records ได้"""
    
    def __init__(self, private_key_path: str, storage_backend):
        self.private_key = self._load_private_key(private_key_path)
        self.storage = storage_backend
        self.previous_hash = self._get_latest_hash()
    
    def log(self, event: dict) -> str:
        """บันทึก audit event พร้อม chain hash"""
        
        # เพิ่ม metadata
        event['event_id'] = self._generate_event_id()
        event['timestamp'] = time.strftime('%Y-%m-%dT%H:%M:%S.000Z', time.gmtime())
        event['previous_event_hash'] = self.previous_hash
        
        # สร้าง event hash (chain ต่อกัน)
        event_json = json.dumps(event, sort_keys=True)
        event_hash = hashlib.sha256(event_json.encode()).hexdigest()
        event['event_hash'] = event_hash
        
        # Sign ด้วย private key
        signature = self.private_key.sign(
            event_hash.encode(),
            padding.PSS(
                mgf=padding.MGF1(hashes.SHA256()),
                salt_length=padding.PSS.MAX_LENGTH
            ),
            hashes.SHA256()
        )
        event['signature'] = signature.hex()
        
        # บันทึก (WORM storage - ไม่สามารถแก้ไขได้)
        self.storage.write(event_id=event['event_id'], data=event)
        
        # Update chain
        self.previous_hash = event_hash
        
        return event['event_id']
    
    def verify(self, event_id: str) -> bool:
        """ตรวจสอบว่า event ไม่ถูกแก้ไข"""
        event = self.storage.read(event_id)
        
        # ตรวจสอบ hash
        event_copy = {k: v for k, v in event.items() 
                     if k not in ['event_hash', 'signature']}
        computed_hash = hashlib.sha256(
            json.dumps(event_copy, sort_keys=True).encode()
        ).hexdigest()
        
        if computed_hash != event['event_hash']:
            return False
        
        # ตรวจสอบ signature
        public_key = self.private_key.public_key()
        try:
            public_key.verify(
                bytes.fromhex(event['signature']),
                event['event_hash'].encode(),
                padding.PSS(
                    mgf=padding.MGF1(hashes.SHA256()),
                    salt_length=padding.PSS.MAX_LENGTH
                ),
                hashes.SHA256()
            )
            return True
        except Exception:
            return False
```

---

## 4. SOX Compliance ใน CI/CD {#sox}

### SOX ต้องการอะไรจาก CI/CD?

Sarbanes-Oxley Act (SOX) มีผลต่อ IT controls:

```
SOX IT Controls ที่เกี่ยวกับ CI/CD:

ITGC (IT General Controls):
1. Change Management
   - ทุก change ต้องมี documented approval
   - Separation of Duties (Dev ≠ Approver ≠ Deployer)
   - Emergency change process
   
2. Access Management
   - Production access ต้อง minimal
   - Privileged access monitored
   - Regular access review
   
3. Computer Operations
   - Monitoring & alerting
   - Incident management
   - Backup & recovery
   
4. Program Development
   - SDLC controls
   - Testing requirements
   - Code review requirements
```

### Separation of Duties ใน CI/CD

```
SOX Separation of Duties:

Developer (writes code)
  │
  │ Cannot approve own code
  ▼
Code Reviewer (peer review)
  │
  │ Cannot deploy to production
  ▼
Tech Lead/Architect (approves deployment)
  │
  │ Cannot initiate deployment
  ▼
Release Manager (initiates deployment)
  │
  │ Cannot override security controls
  ▼
Security Team (monitors deployment)
```

```yaml
# SOX-compliant pipeline configuration
name: SOX Compliant Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
      change_ticket:
        required: true
        description: 'SOX Change Ticket Number (CHG-XXXX)'
      business_justification:
        required: true

jobs:
  validate-change-ticket:
    runs-on: ubuntu-latest
    steps:
      - name: Validate SOX Change Ticket
        run: |
          # ตรวจสอบ change ticket ใน ServiceNow
          TICKET_STATUS=$(curl -s \
            -H "Authorization: Bearer ${{ secrets.SERVICENOW_TOKEN }}" \
            "https://company.service-now.com/api/now/table/change_request" \
            "?sysparm_query=number=${{ inputs.change_ticket }}" \
            | jq -r '.result[0].state')
          
          if [ "$TICKET_STATUS" != "approved" ]; then
            echo "❌ Change ticket ${{ inputs.change_ticket }} is not approved (status: ${TICKET_STATUS})"
            echo "SOX Violation: Unapproved change cannot proceed"
            exit 1
          fi
          
          echo "✅ Change ticket approved"
  
  sox-approval-gate:
    needs: validate-change-ticket
    environment:
      name: sox-approval
    runs-on: ubuntu-latest
    steps:
      - name: Wait for Required Approvals
        run: |
          echo "Waiting for required SOX approvals..."
          echo "Required: Tech Lead + Release Manager"
          echo "Note: Developer cannot approve own deployment (SOD requirement)"
  
  deploy-with-audit:
    needs: sox-approval-gate
    runs-on: ubuntu-latest
    steps:
      - name: Create SOX Audit Record (Pre-deployment)
        run: |
          curl -X POST ${{ secrets.AUDIT_API }}/events \
            -d '{
              "event_type": "production.deployment.initiated",
              "change_ticket": "${{ inputs.change_ticket }}",
              "initiator": "${{ github.actor }}",
              "approvers": ${{ toJson(env.APPROVERS) }},
              "business_justification": "${{ inputs.business_justification }}",
              "sox_control_id": "ITGC-CM-001"
            }'
      
      - name: Deploy Application
        id: deploy
        run: ./scripts/deploy.sh ${{ inputs.environment }}
      
      - name: Create SOX Audit Record (Post-deployment)
        if: always()
        run: |
          curl -X POST ${{ secrets.AUDIT_API }}/events \
            -d '{
              "event_type": "production.deployment.completed",
              "status": "${{ steps.deploy.outcome }}",
              "change_ticket": "${{ inputs.change_ticket }}",
              "sox_control_id": "ITGC-CM-001"
            }'
```

### SOX Evidence Collection

```python
# sox/evidence_collector.py

class SOXEvidenceCollector:
    """รวบรวม evidence สำหรับ SOX audit"""
    
    def collect_change_management_evidence(
        self, 
        quarter: str, 
        year: int
    ) -> dict:
        """รวบรวม evidence สำหรับ Change Management controls"""
        
        evidence = {
            "control": "ITGC-CM-001",
            "period": f"{year} Q{quarter}",
            "collection_date": datetime.now().isoformat(),
            "evidence": {}
        }
        
        # 1. รวบรวมทุก production deployments
        deployments = self.db.query("""
            SELECT 
                deployment_id,
                service_name,
                version,
                deployed_at,
                deployed_by,
                change_ticket,
                approved_by,
                approval_timestamp
            FROM deployments
            WHERE environment = 'production'
            AND deployed_at BETWEEN %s AND %s
        """, (self.get_quarter_start(year, quarter), 
              self.get_quarter_end(year, quarter)))
        
        # 2. ตรวจสอบ Separation of Duties
        sod_violations = []
        for dep in deployments:
            if dep['deployed_by'] in dep['approved_by']:
                sod_violations.append({
                    "deployment_id": dep['deployment_id'],
                    "violation": "Deployer approved own deployment",
                    "deployer": dep['deployed_by']
                })
        
        # 3. ตรวจสอบ Change Ticket Approval
        missing_tickets = [
            dep for dep in deployments 
            if not dep['change_ticket'] or not dep['approval_timestamp']
        ]
        
        evidence['evidence'] = {
            "total_deployments": len(deployments),
            "deployments_with_approved_tickets": len(deployments) - len(missing_tickets),
            "compliance_rate": (len(deployments) - len(missing_tickets)) / len(deployments) * 100,
            "sod_violations": sod_violations,
            "missing_tickets": missing_tickets,
            "sample_deployments": deployments[:10]  # ตัวอย่าง 10 รายการ
        }
        
        return evidence
    
    def generate_sox_report(self, quarter: str, year: int) -> str:
        """สร้าง SOX compliance report"""
        
        evidence = self.collect_change_management_evidence(quarter, year)
        
        report = f"""
# SOX Compliance Report
## Period: {year} Q{quarter}

### Change Management Control (ITGC-CM-001)

**Summary:**
- Total Production Deployments: {evidence['evidence']['total_deployments']}
- Deployments with Approved Tickets: {evidence['evidence']['deployments_with_approved_tickets']}
- Compliance Rate: {evidence['evidence']['compliance_rate']:.1f}%
- SOD Violations: {len(evidence['evidence']['sod_violations'])}

### Control Assessment:
{'✅ EFFECTIVE' if evidence['evidence']['compliance_rate'] >= 95 and 
 not evidence['evidence']['sod_violations'] else '❌ DEFICIENCY IDENTIFIED'}

### Findings:
"""
        
        if evidence['evidence']['sod_violations']:
            report += "\n**Separation of Duties Violations:**\n"
            for v in evidence['evidence']['sod_violations']:
                report += f"- Deployment {v['deployment_id']}: {v['violation']}\n"
        
        if evidence['evidence']['missing_tickets']:
            report += "\n**Missing Change Tickets:**\n"
            for m in evidence['evidence']['missing_tickets']:
                report += f"- Deployment {m['deployment_id']} on {m['deployed_at']}\n"
        
        return report
```

---

## 5. PCI-DSS Compliance ใน CI/CD {#pci-dss}

### PCI-DSS Requirements ที่เกี่ยวกับ CI/CD

```
PCI-DSS v4.0 Requirements สำหรับ CI/CD:

Requirement 6: Develop and Maintain Secure Systems
  6.2 - Bespoke and custom software are developed securely
    6.2.4 - Software engineering techniques or methods 
            prevent or mitigate common software attacks
  
  6.3 - Security vulnerabilities are identified and addressed
    6.3.1 - Vulnerability scanning มีใน pipeline
    6.3.2 - Automated security testing
    6.3.3 - All identified security vulnerabilities addressed
  
  6.4 - Third-party software is protected from tampering
    6.4.1 - Scripts, functions verified before deployment

Requirement 7: Restrict Access to System Components
  7.2 - Access is assigned based on classification
  
Requirement 10: Log and Monitor All Access
  10.2.1 - Audit logs capture specific events
  10.3 - Audit logs are protected from destruction
```

### PCI-DSS Pipeline Implementation

```yaml
# pci-pipeline.yml

name: PCI-DSS Compliant Pipeline

on:
  push:
    branches: [main]
  pull_request:

jobs:
  pci-security-gates:
    name: PCI Security Controls
    runs-on: ubuntu-latest
    
    steps:
      # PCI Req 6.2.4 - SAST scanning
      - name: SAST Scan (PCI 6.2.4)
        uses: checkmarx/ast-github-action@main
        with:
          project-name: ${{ github.repository }}
          cx-tenant: ${{ secrets.CX_TENANT }}
          cx-client-id: ${{ secrets.CX_CLIENT_ID }}
          cx-client-secret: ${{ secrets.CX_CLIENT_SECRET }}
          
      # PCI Req 6.3.1 - Vulnerability scanning
      - name: SCA Scan (PCI 6.3.1)
        run: |
          # ไม่อนุญาต Critical/High ใน cardholder data environment
          snyk test \
            --severity-threshold=high \
            --fail-on=all \
            --json-file-output=sca-results.json
          
          # Log ผลลัพธ์ไปยัง PCI audit log
          python3 scripts/log-pci-scan.py sca-results.json
      
      # PCI Req 6.4.1 - Verify third-party scripts
      - name: Verify Dependencies (PCI 6.4.1)
        run: |
          # ตรวจสอบ integrity ของ dependencies
          npm ci --audit
          
          # ตรวจสอบ signature ของ Docker base images
          docker trust inspect \
            --pretty \
            ${{ env.BASE_IMAGE }}
      
      # PCI Req 6.3.3 - Address vulnerabilities
      - name: Check Vulnerability Exceptions
        run: |
          python3 scripts/check-vulnerability-exceptions.py \
            --scan-results sca-results.json \
            --exceptions-file vulnerability-exceptions.json \
            --approvals-required 2
  
  pci-access-control:
    name: PCI Access Controls
    runs-on: ubuntu-latest
    needs: pci-security-gates
    
    steps:
      # PCI Req 7.2 - Verify minimal access
      - name: Verify Deployment Permissions (PCI 7.2)
        run: |
          # ตรวจสอบว่า service account มี minimal permissions
          python3 scripts/verify-pci-permissions.py \
            --service-account ${{ env.DEPLOY_SA }} \
            --environment ${{ env.TARGET_ENV }}
      
      # Deploy ใน PCI environment
      - name: Deploy to CDE (Cardholder Data Environment)
        run: |
          # Log การ deploy
          log_pci_event "deployment.started" \
            --ticket "${{ env.CHANGE_TICKET }}" \
            --environment "cde-production" \
            --deployer "${{ github.actor }}"
          
          # Deploy
          kubectl apply -f k8s/cde/ \
            --record \
            --namespace cde-production
          
          # Log ผลลัพธ์
          log_pci_event "deployment.completed" \
            --status "success"
```

### PCI-DSS Segmentation Verification

```python
# pci/segmentation_test.py

import subprocess
import ipaddress

class PCISegmentationVerifier:
    """ตรวจสอบว่า CDE แยกออกจาก out-of-scope systems"""
    
    def __init__(self, cde_subnets: list, oos_subnets: list):
        self.cde_subnets = [ipaddress.ip_network(s) for s in cde_subnets]
        self.oos_subnets = [ipaddress.ip_network(s) for s in oos_subnets]
    
    def verify_no_cross_access(self) -> dict:
        """ตรวจสอบว่าไม่มี unauthorized access ระหว่าง zones"""
        
        violations = []
        
        # ทดสอบ connectivity จาก out-of-scope → CDE
        for oos_subnet in self.oos_subnets:
            for cde_subnet in self.cde_subnets:
                # ทดสอบ common ports
                for port in [22, 3306, 5432, 6379, 27017]:
                    result = self._test_connectivity(
                        source_subnet=str(oos_subnet),
                        dest_subnet=str(cde_subnet),
                        port=port
                    )
                    
                    if result['accessible']:
                        violations.append({
                            "violation_type": "unauthorized_access",
                            "source": str(oos_subnet),
                            "destination": str(cde_subnet),
                            "port": port,
                            "pci_requirement": "1.3.2",
                            "severity": "CRITICAL"
                        })
        
        return {
            "test_timestamp": datetime.now().isoformat(),
            "violations": violations,
            "compliant": len(violations) == 0
        }
    
    def run_quarterly_test(self) -> str:
        """PCI Req 11.4.1: Quarterly penetration test"""
        
        results = self.verify_no_cross_access()
        
        # บันทึกผลลัพธ์สำหรับ audit
        report_path = f"pci-segmentation-test-{datetime.now().strftime('%Y%m%d')}.json"
        with open(report_path, 'w') as f:
            json.dump(results, f, indent=2)
        
        # Upload ไปยัง compliance storage
        self._upload_to_compliance_storage(report_path)
        
        return report_path
```

---

## 6. HIPAA Compliance ใน CI/CD {#hipaa}

### HIPAA ใน Software Deployment

```
HIPAA Technical Safeguards ที่เกี่ยวกับ CI/CD:

§164.312(a) - Access Controls
  - Pipeline access ต้อง role-based
  - PHI environments ต้อง strictly controlled

§164.312(b) - Audit Controls
  - ทุก access ต้อง logged
  - Logs ต้อง immutable และ encrypted

§164.312(c) - Integrity
  - Software integrity verification
  - Signed deployments

§164.312(d) - Authentication
  - MFA สำหรับ PHI environment access

§164.312(e) - Transmission Security
  - Encrypted channels ทั้งหมด
```

### HIPAA-Compliant Pipeline

```yaml
# hipaa-pipeline.yml

name: HIPAA Compliant Deployment

jobs:
  hipaa-controls:
    steps:
      # ป้องกันไม่ให้ PHI ใน code/logs
      - name: PHI Detection Scan
        run: |
          # สแกนหา PHI patterns ใน source code
          python3 scripts/phi-scanner.py \
            --paths "src/ tests/" \
            --patterns phi_patterns.yml \
            --fail-on-detect
          
          # ตรวจสอบ logs ที่อาจมี PHI
          grep -r \
            -E "(SSN|DOB|patient|diagnosis|prescription)" \
            src/logging/ && {
            echo "❌ Potential PHI logging detected"
            exit 1
          } || echo "✅ No PHI logging patterns detected"
      
      # ตรวจสอบ encryption
      - name: Verify Encryption Requirements
        run: |
          # ตรวจสอบว่าทุก PHI data ถูก encrypt
          python3 scripts/encryption-audit.py \
            --config-file encryption-requirements.yml \
            --hipaa-mode
      
      # Minimum Necessary Access
      - name: Verify Minimum Necessary Access (HIPAA §164.514)
        run: |
          # ตรวจสอบว่า service มี access เฉพาะ PHI ที่จำเป็น
          python3 scripts/hipaa-access-audit.py \
            --service ${{ env.SERVICE_NAME }} \
            --policy hipaa-access-policy.yml
      
      - name: Deploy to HIPAA Environment
        run: |
          # ทุก deployment ต้อง encrypted in transit
          kubectl apply \
            --server-side \
            --field-manager=hipaa-controller \
            -f k8s/hipaa/ \
            --namespace hipaa-production
          
          # Log deployment event
          python3 scripts/hipaa-audit-log.py \
            --event deployment.completed \
            --details '{"service":"${{ env.SERVICE_NAME }}"}'
```

---

## 7. Compliance Gates ในทุก Stage {#compliance-gates}

### Compliance Gate Framework

```
Pipeline ที่มี Compliance Gates ครบถ้วน:

Commit Stage:
  ├── Pre-commit hooks (local)
  │     ├── Secret scanning
  │     ├── License check
  │     └── Code formatting
  │
CI Stage:
  ├── SAST Scan (Semgrep/Checkmarx)
  ├── SCA Scan (Snyk/Dependabot)
  ├── Secrets Detection (Gitleaks/TruffleHog)
  ├── License Compliance
  ├── Code Coverage Check
  └── Policy Compliance (OPA)
  
Build Stage:
  ├── Container Scanning (Trivy)
  ├── Container Signing (Sigstore)
  ├── SBOM Generation
  └── Artifact Integrity Verification

Deployment Stage:
  ├── Change Ticket Verification (SOX)
  ├── Approval Gates
  ├── Infrastructure Policy Check (Conftest)
  ├── Network Policy Verification
  └── Compliance Evidence Collection

Runtime:
  ├── Continuous Vulnerability Scanning
  ├── Runtime Security (Falco)
  ├── Access Anomaly Detection
  └── Compliance Reporting
```

### Automated Compliance Reporting

```python
# compliance/reporter.py

class ComplianceReporter:
    """สร้าง compliance reports อัตโนมัติ"""
    
    def generate_quarterly_report(
        self,
        frameworks: list,
        quarter: int,
        year: int
    ) -> dict:
        """สร้าง quarterly compliance report"""
        
        report = {
            "period": f"{year} Q{quarter}",
            "generated_at": datetime.now().isoformat(),
            "frameworks": {}
        }
        
        for framework in frameworks:
            if framework == "SOX":
                report["frameworks"]["SOX"] = self._sox_report(quarter, year)
            elif framework == "PCI-DSS":
                report["frameworks"]["PCI-DSS"] = self._pci_report(quarter, year)
            elif framework == "HIPAA":
                report["frameworks"]["HIPAA"] = self._hipaa_report(quarter, year)
            elif framework == "SOC2":
                report["frameworks"]["SOC2"] = self._soc2_report(quarter, year)
        
        # คำนวณ overall compliance score
        scores = [
            fw_data["compliance_score"] 
            for fw_data in report["frameworks"].values()
        ]
        report["overall_score"] = sum(scores) / len(scores)
        
        return report
    
    def _sox_report(self, quarter: int, year: int) -> dict:
        """รายงาน SOX compliance"""
        
        deployments = self.db.get_deployments(quarter, year)
        
        return {
            "control": "Change Management (ITGC-CM-001)",
            "total_changes": len(deployments),
            "authorized_changes": sum(1 for d in deployments if d.has_approval),
            "sod_violations": self._check_sod_violations(deployments),
            "compliance_score": self._calculate_sox_score(deployments),
            "evidence_links": self._get_evidence_links(deployments)
        }
    
    def send_to_auditor(self, report: dict, auditor_email: str):
        """ส่ง report ไปยัง auditor"""
        
        # สร้าง PDF report
        pdf_path = self._generate_pdf(report)
        
        # เข้ารหัส PDF
        encrypted_pdf = self._encrypt_file(
            pdf_path,
            recipient_key=self._get_auditor_public_key(auditor_email)
        )
        
        # ส่ง email พร้อมลายเซ็นดิจิทัล
        self.email_service.send(
            to=auditor_email,
            subject=f"Compliance Report Q{report['period'].split()[-1]} {datetime.now().year}",
            body="Please find attached the compliance report",
            attachments=[encrypted_pdf],
            sign_with=self.company_private_key
        )
```

---

## 8. Evidence Collection Automation {#evidence-collection}

### Automated Evidence Collection

สำหรับ audit ประจำปี เราต้องรวบรวม evidence มากมาย:

```python
# evidence/collector.py

EVIDENCE_REQUIREMENTS = {
    "SOX": {
        "ITGC-CM-001": {
            "description": "Change Management",
            "evidence_types": [
                "deployment_approvals",
                "change_tickets",
                "sod_verification"
            ],
            "retention_years": 7
        },
        "ITGC-AM-001": {
            "description": "Access Management",
            "evidence_types": [
                "access_reviews",
                "privileged_access_logs",
                "provisioning_deprovisioning"
            ],
            "retention_years": 7
        }
    },
    "PCI-DSS": {
        "REQ-6": {
            "description": "Secure Systems Development",
            "evidence_types": [
                "vulnerability_scans",
                "code_review_evidence",
                "patch_management_records"
            ],
            "retention_years": 1
        }
    }
}

class EvidenceCollector:
    def collect_for_audit(
        self,
        framework: str,
        period_start: datetime,
        period_end: datetime
    ) -> str:
        """รวบรวม evidence ทั้งหมดสำหรับ audit period"""
        
        evidence_package = {
            "framework": framework,
            "period": {
                "start": period_start.isoformat(),
                "end": period_end.isoformat()
            },
            "collected_at": datetime.now().isoformat(),
            "evidence": {}
        }
        
        requirements = EVIDENCE_REQUIREMENTS.get(framework, {})
        
        for control_id, control_info in requirements.items():
            evidence_package["evidence"][control_id] = {
                "description": control_info["description"],
                "items": []
            }
            
            for evidence_type in control_info["evidence_types"]:
                items = self._collect_evidence(
                    evidence_type=evidence_type,
                    start=period_start,
                    end=period_end
                )
                evidence_package["evidence"][control_id]["items"].extend(items)
        
        # บันทึกและ sign evidence package
        package_path = self._save_evidence_package(evidence_package, framework)
        signed_package = self._sign_package(package_path)
        
        # Upload ไปยัง long-term storage
        storage_url = self._upload_to_compliance_storage(
            signed_package,
            retention_years=requirements.get("retention_years", 7)
        )
        
        return storage_url
```

---

## 9. Regulated Industry Case Studies {#case-studies}

### Case Study 1: Investment Bank - SOX Automation

**บริบท:**
- Global investment bank, 15,000 employees
- 500 application developers
- Annual SOX audit cost: $2M
- Manual evidence collection: 3 weeks/quarter

**โซลูชัน:**

```
Phase 1: Compliance Pipeline Framework
- ออกแบบ pipeline template ที่ built-in compliance controls
- สร้าง automated audit trail
- Integrate กับ ServiceNow (change management)

Phase 2: Evidence Automation
- Automated quarterly evidence collection
- Auto-generate SOX report
- Direct Auditor portal access

Phase 3: Continuous Compliance
- Real-time compliance dashboard
- Automated violation alerting
- Quarterly compliance drills
```

**ผลลัพธ์:**
```
Annual audit cost: $2M → $400K (80% reduction)
Evidence collection: 3 weeks → 2 days (93% reduction)
SOX violations (prior year): 12 → 0
Auditor satisfaction: Significant improvement
Developer productivity: +25% (less manual compliance work)
```

### Case Study 2: Healthcare System - HIPAA Pipeline

**บริบท:**
- Regional hospital system
- 50 developers, 30 applications
- Handle PHI for 2M patients
- Previous HIPAA audit findings: 8 issues

**ความท้าทาย:**
```
- Developer ไม่เข้าใจ HIPAA requirements
- PHI accidentally logged หลายครั้ง
- Insufficient audit trail
- No automated scanning for PHI
```

**โซลูชัน:**
```yaml
# hipaa-developer-guardrails.yml

pre-commit:
  hooks:
    - name: PHI Pattern Scanner
      entry: scripts/phi-scan.py
      language: python
      types: [python, java, javascript]
      
    - name: Encryption Checker
      entry: scripts/check-encryption.py
      
    - name: Logging Reviewer
      entry: scripts/check-logging.py
      description: "Prevents PHI in log statements"
```

**ผลลัพธ์:**
```
PHI exposure incidents: 0 (vs 5 in previous year)
HIPAA audit findings: 0 (vs 8 previous year)
Developer HIPAA awareness: Improved significantly
Time to deploy PHI-safe code: Reduced by 40%
```

---

## 10. แบบฝึกหัด {#exercises}

### แบบฝึกหัดที่ 1: Policy as Code

**งาน:** เขียน OPA policy ที่ enforce:
1. ทุก production deployment ต้องมี approved change ticket
2. ห้าม deploy ระหว่าง 22:00-06:00 UTC (maintenance window)
3. Production deployments ต้องมี 2 approvals
4. Security scan ต้องเสร็จใน 24 ชั่วโมงที่ผ่านมา

```rego
# เริ่มต้น skeleton
package cicd.enterprise.policy

# TODO: implement your policies here

default allow = false

# Hint: ใช้ time.now_ns() สำหรับ time-based checks
# Hint: input.approvals เป็น array ของ objects
```

### แบบฝึกหัดที่ 2: SOX Evidence Report

**งาน:** สร้าง script ที่:
1. Query deployment history จาก GitHub API
2. ตรวจสอบ SOD violations
3. Link กับ change tickets
4. สร้าง HTML report สำหรับ auditor

```python
# เริ่มต้น
import requests
from datetime import datetime, timedelta

GITHUB_TOKEN = "your-token"
ORG = "your-org"
QUARTER_START = datetime(2024, 1, 1)
QUARTER_END = datetime(2024, 3, 31)

def collect_sox_evidence():
    # TODO: Implement evidence collection
    pass

def check_sod_violations(deployments):
    # TODO: Check if deployer == approver
    pass

def generate_report(evidence):
    # TODO: Create HTML report
    pass
```

### แบบฝึกหัดที่ 3: HIPAA PHI Detection

**งาน:** สร้าง pre-commit hook ที่:
1. สแกนหา PHI patterns ใน code
2. ตรวจสอบ logging statements
3. บล็อก commit ถ้าพบ PHI
4. สร้าง report สำหรับ Security team

**PHI Patterns ที่ต้องตรวจจับ:**
```python
PHI_PATTERNS = [
    r'\bSSN\b|\b\d{3}-\d{2}-\d{4}\b',  # Social Security Number
    r'\bDOB\b|date.of.birth|birthdate',  # Date of Birth
    r'\bdiagnosis\b|\bmedical.record\b',  # Medical terms
    r'\bpatient.name\b|\bpatient.id\b',  # Patient identifiers
    r'\bprescription\b|\bmedication\b',  # Medication info
]
```

### แบบฝึกหัดที่ 4: Compliance Dashboard

**งาน:** สร้าง compliance dashboard ที่แสดง:
1. Real-time compliance score สำหรับแต่ละ framework
2. Recent violations และ status
3. Upcoming audit dates
4. Evidence collection progress

---

## สรุป

การทำ Compliance ใน CI/CD ไม่ต้องเป็นอุปสรรคต่อ Developer Productivity ถ้าออกแบบถูกต้อง:

1. **Shift-Left Compliance** - ตรวจสอบ early ในกระบวนการ ไม่ใช่ตอน deploy
2. **Automate Everything** - Manual compliance checks ทั้งแพงและ error-prone
3. **Policy as Code** - Policy ที่ executable และ testable
4. **Immutable Audit Trails** - ไม่สามารถแก้ไขหรือลบได้
5. **Developer-Friendly Controls** - Compliance ต้องไม่ขัดขวาง velocity

## อ่านเพิ่มเติม

- NIST Cybersecurity Framework: https://www.nist.gov/cyberframework
- PCI Security Standards Council: https://www.pcisecuritystandards.org
- HIPAA Security Rule: https://www.hhs.gov/hipaa/for-professionals/security
- SOX IT Controls Guide: ISACA, PCAOB standards

---

*Part 82 จาก 100 | CI/CD Mastery Course*

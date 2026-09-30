# Part 68: Compliance Automation

## บทนำ: Compliance ในโลก DevOps

ก่อน DevOps, compliance เป็นกระบวนการที่ช้า manual intensive และมักเป็น blocker สำหรับ releases แต่ด้วย "Compliance as Code" เราสามารถทำให้ compliance เป็น automated, continuous, และ fast feedback process

### Compliance Frameworks หลักที่เราจะพูดถึง

| Framework | อุตสาหกรรม | โฟกัส |
|-----------|-----------|-------|
| SOC 2 | Software/Cloud | Security, Availability, Confidentiality |
| PCI-DSS | Payment/Fintech | Cardholder data protection |
| HIPAA | Healthcare | Patient data privacy |
| ISO 27001 | ทุกอุตสาหกรรม | Information security management |
| GDPR | EU businesses | Personal data protection |

### Traditional vs Automated Compliance

```
Traditional:
├── Annual/quarterly audits
├── Manual evidence collection (เดือนต่อเดือน)
├── Audit takes 3-6 months
├── High cost consultants
└── Reactive (find issues after the fact)

Automated (Compliance as Code):
├── Continuous monitoring
├── Automated evidence collection (real-time)
├── Always audit-ready
├── Lower cost
└── Proactive (prevent issues before they happen)
```

---

## 1. SOC 2 Compliance Automation

### 1.1 SOC 2 Trust Service Criteria

SOC 2 มี 5 Trust Service Criteria (TSC):
- **Security**: ป้องกัน unauthorized access
- **Availability**: ระบบ available ตามที่ตกลงกัน
- **Processing Integrity**: ประมวลผลถูกต้อง สมบูรณ์ ทันเวลา
- **Confidentiality**: ข้อมูลลับได้รับการปกป้อง
- **Privacy**: ข้อมูลส่วนตัวถูกจัดการอย่างถูกต้อง

### 1.2 Compliance Controls Mapping

```yaml
# compliance/soc2/controls.yaml
controls:
  # CC6: Logical and Physical Access Controls
  CC6.1:
    title: "Access to information assets is restricted"
    category: "Security"
    automated_checks:
      - iam_mfa_enabled
      - iam_password_policy
      - s3_bucket_not_public
      - rds_not_publicly_accessible
    evidence_collection:
      - type: aws_config
        rule: IAM_USER_MFA_ENABLED
      - type: aws_security_hub
        standard: CIS_AWS_FOUNDATIONS
  
  CC6.2:
    title: "Prior to issuing credentials, authorization is required"
    category: "Security"
    automated_checks:
      - iam_no_root_access_key
      - iam_user_has_groups
    evidence_collection:
      - type: aws_iam_report
  
  CC7.1:
    title: "Vulnerability management process is in place"
    category: "Security"
    automated_checks:
      - ecr_image_scanning_enabled
      - inspector_enabled
      - security_hub_enabled
    evidence_collection:
      - type: aws_inspector
      - type: snyk_report
      - type: dependabot_alerts
  
  # A1: Availability
  A1.2:
    title: "Recovery objectives are established and met"
    category: "Availability"
    automated_checks:
      - rds_multi_az_enabled
      - backup_enabled
      - dr_tested
    evidence_collection:
      - type: aws_backup
      - type: cloudwatch_metrics
```

### 1.3 Automated Evidence Collection

```python
# compliance/scripts/collect_evidence.py
import boto3
import json
import os
from datetime import datetime, timedelta
from pathlib import Path

class ComplianceEvidenceCollector:
    
    def __init__(self, output_dir: str = "evidence"):
        self.aws_session = boto3.Session()
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)
        self.evidence_manifest = []
    
    def collect_all_evidence(self, period_start: str, period_end: str):
        """Collect all compliance evidence"""
        print(f"Collecting evidence from {period_start} to {period_end}")
        
        collectors = [
            self.collect_iam_evidence,
            self.collect_network_evidence,
            self.collect_encryption_evidence,
            self.collect_logging_evidence,
            self.collect_backup_evidence,
            self.collect_vulnerability_evidence,
            self.collect_access_logs,
        ]
        
        for collector in collectors:
            try:
                collector(period_start, period_end)
            except Exception as e:
                print(f"Warning: {collector.__name__} failed: {e}")
        
        # Generate manifest
        self._save_manifest()
        print(f"Evidence collected: {len(self.evidence_manifest)} items")
    
    def collect_iam_evidence(self, period_start: str, period_end: str):
        """Collect IAM-related evidence"""
        iam = self.aws_session.client('iam')
        
        # MFA Status
        users = iam.list_users()['Users']
        mfa_data = []
        
        for user in users:
            mfa_devices = iam.list_mfa_devices(UserName=user['UserName'])['MFADevices']
            mfa_data.append({
                'username': user['UserName'],
                'mfa_enabled': len(mfa_devices) > 0,
                'mfa_devices': len(mfa_devices),
                'created_date': user['CreateDate'].isoformat(),
                'password_last_used': user.get('PasswordLastUsed', 'Never').isoformat() if user.get('PasswordLastUsed') else 'Never'
            })
        
        self._save_evidence(
            'iam_mfa_status',
            mfa_data,
            control='CC6.1',
            description='MFA status for all IAM users'
        )
        
        # Password Policy
        try:
            policy = iam.get_account_password_policy()['PasswordPolicy']
            self._save_evidence(
                'iam_password_policy',
                policy,
                control='CC6.1',
                description='IAM account password policy'
            )
        except iam.exceptions.NoSuchEntityException:
            self._save_evidence(
                'iam_password_policy',
                {'error': 'No password policy configured'},
                control='CC6.1',
                description='IAM account password policy - MISSING'
            )
        
        # Access Key Age
        access_key_data = []
        for user in users:
            keys = iam.list_access_keys(UserName=user['UserName'])['AccessKeyMetadata']
            for key in keys:
                created_date = key['CreateDate']
                age_days = (datetime.now(created_date.tzinfo) - created_date).days
                
                access_key_data.append({
                    'username': user['UserName'],
                    'access_key_id': key['AccessKeyId'][:6] + '...',
                    'status': key['Status'],
                    'age_days': age_days,
                    'compliant': age_days <= 90  # Keys should rotate every 90 days
                })
        
        self._save_evidence(
            'iam_access_keys',
            access_key_data,
            control='CC6.1',
            description='IAM access key age and rotation status'
        )
    
    def collect_encryption_evidence(self, period_start: str, period_end: str):
        """Collect encryption-related evidence"""
        
        # S3 Encryption
        s3 = self.aws_session.client('s3')
        
        buckets = s3.list_buckets()['Buckets']
        s3_encryption_data = []
        
        for bucket in buckets:
            bucket_name = bucket['Name']
            
            try:
                encryption = s3.get_bucket_encryption(Bucket=bucket_name)
                rules = encryption['ServerSideEncryptionConfiguration']['Rules']
                
                s3_encryption_data.append({
                    'bucket': bucket_name,
                    'encrypted': True,
                    'encryption_type': rules[0]['ApplyServerSideEncryptionByDefault']['SSEAlgorithm']
                })
            except s3.exceptions.NoSuchBucketEncryption:
                s3_encryption_data.append({
                    'bucket': bucket_name,
                    'encrypted': False,
                    'encryption_type': 'NONE'
                })
        
        self._save_evidence(
            's3_encryption',
            s3_encryption_data,
            control='CC6.7',
            description='S3 bucket encryption status'
        )
        
        # RDS Encryption
        rds = self.aws_session.client('rds')
        instances = rds.describe_db_instances()['DBInstances']
        
        rds_encryption_data = [{
            'identifier': db['DBInstanceIdentifier'],
            'engine': db['Engine'],
            'encrypted': db['StorageEncrypted'],
            'kms_key': db.get('KmsKeyId', 'default') if db['StorageEncrypted'] else 'N/A',
            'multi_az': db['MultiAZ'],
            'backup_retention': db['BackupRetentionPeriod']
        } for db in instances]
        
        self._save_evidence(
            'rds_configuration',
            rds_encryption_data,
            control='CC6.7',
            description='RDS instance security configuration'
        )
    
    def collect_logging_evidence(self, period_start: str, period_end: str):
        """Collect logging and monitoring evidence"""
        
        # CloudTrail Status
        cloudtrail = self.aws_session.client('cloudtrail')
        trails = cloudtrail.describe_trails()['trailList']
        
        trail_data = []
        for trail in trails:
            status = cloudtrail.get_trail_status(Name=trail['TrailARN'])
            trail_data.append({
                'name': trail['Name'],
                'is_logging': status['IsLogging'],
                'multi_region': trail.get('IsMultiRegionTrail', False),
                'include_management_events': True,
                's3_bucket': trail.get('S3BucketName'),
                'log_file_validation': trail.get('LogFileValidationEnabled', False)
            })
        
        self._save_evidence(
            'cloudtrail_status',
            trail_data,
            control='CC7.2',
            description='CloudTrail logging status'
        )
        
        # CloudWatch Alarms
        cloudwatch = self.aws_session.client('cloudwatch')
        alarms = cloudwatch.describe_alarms()['MetricAlarms']
        
        self._save_evidence(
            'cloudwatch_alarms',
            [{'name': a['AlarmName'], 'state': a['StateValue'], 
              'metric': a['MetricName']} for a in alarms],
            control='CC7.2',
            description='CloudWatch alarms configuration'
        )
    
    def collect_vulnerability_evidence(self, period_start: str, period_end: str):
        """Collect vulnerability management evidence"""
        
        # AWS Inspector findings
        inspector = self.aws_session.client('inspector2')
        
        findings = inspector.list_findings(
            filterCriteria={
                'updatedAt': [{
                    'startInclusive': datetime.fromisoformat(period_start),
                    'endInclusive': datetime.fromisoformat(period_end)
                }]
            }
        )
        
        severity_counts = {'CRITICAL': 0, 'HIGH': 0, 'MEDIUM': 0, 'LOW': 0, 'INFORMATIONAL': 0}
        
        for finding in findings.get('findings', []):
            severity = finding['severity']
            severity_counts[severity] = severity_counts.get(severity, 0) + 1
        
        self._save_evidence(
            'inspector_findings',
            {
                'period': f"{period_start} to {period_end}",
                'total_findings': sum(severity_counts.values()),
                'severity_breakdown': severity_counts,
                'compliant': severity_counts['CRITICAL'] == 0
            },
            control='CC7.1',
            description='AWS Inspector vulnerability findings summary'
        )
    
    def _save_evidence(self, name: str, data: dict, control: str, description: str):
        """Save evidence to file"""
        evidence_file = self.output_dir / f"{name}.json"
        
        evidence = {
            'evidence_id': f"{name}_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
            'control': control,
            'description': description,
            'collected_at': datetime.now().isoformat(),
            'data': data
        }
        
        with open(evidence_file, 'w') as f:
            json.dump(evidence, f, indent=2, default=str)
        
        self.evidence_manifest.append({
            'id': evidence['evidence_id'],
            'control': control,
            'file': str(evidence_file),
            'description': description
        })
        
        print(f"✓ Collected: {name} (Control: {control})")
    
    def _save_manifest(self):
        """Save evidence manifest"""
        manifest = {
            'collected_at': datetime.now().isoformat(),
            'total_evidence_items': len(self.evidence_manifest),
            'items': self.evidence_manifest
        }
        
        with open(self.output_dir / 'manifest.json', 'w') as f:
            json.dump(manifest, f, indent=2)

if __name__ == '__main__':
    import argparse
    
    parser = argparse.ArgumentParser()
    parser.add_argument('--start', required=True, help='Period start (ISO 8601)')
    parser.add_argument('--end', required=True, help='Period end (ISO 8601)')
    parser.add_argument('--output', default='evidence', help='Output directory')
    args = parser.parse_args()
    
    collector = ComplianceEvidenceCollector(output_dir=args.output)
    collector.collect_all_evidence(args.start, args.end)
```

---

## 2. PCI-DSS Compliance

### 2.1 PCI-DSS Requirements Checklist

```yaml
# compliance/pci-dss/requirements.yaml
pci_requirements:
  # Requirement 1: Network security controls
  requirement_1:
    title: "Install and maintain network security controls"
    automated_checks:
      - check_security_groups_restrict_inbound
      - check_no_unrestricted_internet_access_to_cardholder_data
      - check_firewall_rules_documented
    
  # Requirement 2: Secure configurations
  requirement_2:
    title: "Apply secure configurations to all system components"
    automated_checks:
      - check_no_default_passwords
      - check_unnecessary_services_disabled
      - check_security_parameters_documented
  
  # Requirement 3: Protect stored account data
  requirement_3:
    title: "Protect stored account data"
    automated_checks:
      - check_pan_data_encrypted_at_rest
      - check_no_pan_in_logs
      - check_card_data_retention_policy
    
  # Requirement 8: Identify users and authenticate access
  requirement_8:
    title: "Identify users and authenticate access to system components"
    automated_checks:
      - check_mfa_enabled_for_all_users
      - check_account_lockout_policy
      - check_session_timeout_configured
      - check_privileged_access_review
```

### 2.2 PCI-DSS Automated Checks

```python
# compliance/pci_dss/checks.py
import boto3
import json
from typing import List, Dict, Any

class PCIDSSChecker:
    
    def __init__(self):
        self.session = boto3.Session()
        self.findings = []
    
    def run_all_checks(self) -> List[Dict]:
        """Run all PCI-DSS checks"""
        checks = [
            self.check_no_pan_in_logs,
            self.check_encryption_at_rest,
            self.check_network_segmentation,
            self.check_access_controls,
            self.check_audit_logs,
        ]
        
        for check in checks:
            try:
                result = check()
                self.findings.extend(result)
            except Exception as e:
                self.findings.append({
                    'check': check.__name__,
                    'status': 'ERROR',
                    'message': str(e),
                    'pci_requirement': 'UNKNOWN'
                })
        
        return self.findings
    
    def check_no_pan_in_logs(self) -> List[Dict]:
        """
        PCI Req 3.4: ตรวจสอบว่าไม่มี PAN (Primary Account Number) 
        ใน application logs
        """
        findings = []
        
        logs = self.session.client('logs')
        
        # ค้นหา log groups ที่เกี่ยวกับ payment
        payment_log_groups = [
            '/aws/lambda/payment-processor',
            '/application/checkout-service',
            '/application/payment-api'
        ]
        
        # PAN pattern (simplified - real implementation needs more sophisticated detection)
        pan_patterns = [
            r'\b4[0-9]{12}(?:[0-9]{3})?\b',          # Visa
            r'\b5[1-5][0-9]{14}\b',                   # Mastercard
            r'\b3[47][0-9]{13}\b',                    # American Express
        ]
        
        for log_group in payment_log_groups:
            for pattern in pan_patterns:
                try:
                    response = logs.filter_log_events(
                        logGroupName=log_group,
                        filterPattern=pattern,
                        startTime=int((datetime.now() - timedelta(hours=24)).timestamp() * 1000),
                        endTime=int(datetime.now().timestamp() * 1000)
                    )
                    
                    if response['events']:
                        findings.append({
                            'check': 'check_no_pan_in_logs',
                            'status': 'FAIL',
                            'severity': 'CRITICAL',
                            'pci_requirement': '3.4',
                            'message': f'PAN detected in log group: {log_group}',
                            'evidence': {
                                'log_group': log_group,
                                'event_count': len(response['events'])
                            }
                        })
                except Exception:
                    pass  # Log group doesn't exist
        
        if not findings:
            findings.append({
                'check': 'check_no_pan_in_logs',
                'status': 'PASS',
                'pci_requirement': '3.4',
                'message': 'No PAN detected in logs'
            })
        
        return findings
    
    def check_encryption_at_rest(self) -> List[Dict]:
        """
        PCI Req 3.5: ตรวจสอบ encryption at rest
        """
        findings = []
        
        # ตรวจสอบ RDS
        rds = self.session.client('rds')
        dbs = rds.describe_db_instances()['DBInstances']
        
        for db in dbs:
            if not db.get('StorageEncrypted', False):
                findings.append({
                    'check': 'check_encryption_at_rest',
                    'status': 'FAIL',
                    'severity': 'HIGH',
                    'pci_requirement': '3.5.1',
                    'message': f"RDS {db['DBInstanceIdentifier']} not encrypted",
                    'resource': db['DBInstanceArn']
                })
        
        # ตรวจสอบ EBS volumes
        ec2 = self.session.client('ec2')
        volumes = ec2.describe_volumes()['Volumes']
        
        for vol in volumes:
            if not vol.get('Encrypted', False):
                findings.append({
                    'check': 'check_encryption_at_rest',
                    'status': 'FAIL',
                    'severity': 'HIGH',
                    'pci_requirement': '3.5.1',
                    'message': f"EBS volume {vol['VolumeId']} not encrypted",
                    'resource': vol['VolumeId']
                })
        
        if not any(f['status'] == 'FAIL' for f in findings):
            findings.append({
                'check': 'check_encryption_at_rest',
                'status': 'PASS',
                'pci_requirement': '3.5.1',
                'message': 'All storage resources are encrypted'
            })
        
        return findings
    
    def generate_report(self) -> Dict:
        """Generate compliance report"""
        pass_count = sum(1 for f in self.findings if f['status'] == 'PASS')
        fail_count = sum(1 for f in self.findings if f['status'] == 'FAIL')
        
        critical_failures = [f for f in self.findings 
                             if f['status'] == 'FAIL' and f.get('severity') == 'CRITICAL']
        
        return {
            'generated_at': datetime.now().isoformat(),
            'summary': {
                'total_checks': len(self.findings),
                'passed': pass_count,
                'failed': fail_count,
                'compliance_percentage': (pass_count / len(self.findings) * 100) 
                                          if self.findings else 0,
                'critical_failures': len(critical_failures)
            },
            'findings': self.findings,
            'compliant': fail_count == 0
        }
```

---

## 3. Audit Trail Automation

### 3.1 Immutable Audit Logs

```python
# compliance/audit/audit_trail.py
import json
import hashlib
import hmac
from datetime import datetime
import boto3

class AuditTrail:
    """
    Tamper-evident audit trail using hash chaining
    (คล้าย blockchain ขนาดเล็ก)
    """
    
    def __init__(self, table_name: str, signing_key: str):
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table(table_name)
        self.signing_key = signing_key.encode()
    
    def log_event(self, event_type: str, actor: str, resource: str, 
                  action: str, metadata: dict = None):
        """Log an audit event with tamper detection"""
        
        # Get last event for chaining
        previous_hash = self._get_last_hash()
        
        # Create event
        timestamp = datetime.utcnow().isoformat()
        event_data = {
            'event_type': event_type,
            'actor': actor,
            'resource': resource,
            'action': action,
            'timestamp': timestamp,
            'metadata': metadata or {},
            'previous_hash': previous_hash
        }
        
        # Calculate hash
        event_hash = self._calculate_hash(event_data)
        
        # HMAC signature for additional tamper detection
        signature = hmac.new(
            self.signing_key,
            json.dumps(event_data, sort_keys=True).encode(),
            hashlib.sha256
        ).hexdigest()
        
        # Store event
        self.table.put_item(Item={
            'event_id': f"{timestamp}-{event_hash[:8]}",
            'timestamp': timestamp,
            'event_data': json.dumps(event_data),
            'event_hash': event_hash,
            'signature': signature,
            'sequence': self._get_next_sequence()
        })
        
        return event_hash
    
    def verify_integrity(self, start_sequence: int, end_sequence: int) -> dict:
        """Verify audit trail integrity"""
        events = self._get_events_in_range(start_sequence, end_sequence)
        
        violations = []
        previous_hash = None
        
        for event in sorted(events, key=lambda e: e['sequence']):
            event_data = json.loads(event['event_data'])
            
            # Verify hash chain
            if previous_hash and event_data['previous_hash'] != previous_hash:
                violations.append({
                    'event_id': event['event_id'],
                    'issue': 'Hash chain broken - possible tampering detected'
                })
            
            # Verify HMAC
            expected_signature = hmac.new(
                self.signing_key,
                json.dumps(event_data, sort_keys=True).encode(),
                hashlib.sha256
            ).hexdigest()
            
            if event['signature'] != expected_signature:
                violations.append({
                    'event_id': event['event_id'],
                    'issue': 'Signature mismatch - event may have been modified'
                })
            
            # Verify stored hash
            calculated_hash = self._calculate_hash(event_data)
            if calculated_hash != event['event_hash']:
                violations.append({
                    'event_id': event['event_id'],
                    'issue': 'Hash mismatch - event data corrupted'
                })
            
            previous_hash = event['event_hash']
        
        return {
            'events_verified': len(events),
            'violations_found': len(violations),
            'violations': violations,
            'integrity_intact': len(violations) == 0
        }
    
    def _calculate_hash(self, data: dict) -> str:
        return hashlib.sha256(
            json.dumps(data, sort_keys=True).encode()
        ).hexdigest()
    
    def _get_last_hash(self) -> str:
        # ดึง hash ของ event ล่าสุด
        response = self.table.query(
            IndexName='sequence-index',
            ScanIndexForward=False,
            Limit=1
        )
        
        if response['Items']:
            return response['Items'][0]['event_hash']
        return '0' * 64  # Genesis hash
    
    def _get_next_sequence(self) -> int:
        # Simple incrementing sequence (ใน production ควรใช้ atomic counter)
        response = self.table.scan(Select='COUNT')
        return response['Count'] + 1
    
    def _get_events_in_range(self, start: int, end: int) -> list:
        response = self.table.scan(
            FilterExpression='#seq BETWEEN :start AND :end',
            ExpressionAttributeNames={'#seq': 'sequence'},
            ExpressionAttributeValues={':start': start, ':end': end}
        )
        return response['Items']
```

---

## 4. Continuous Compliance Monitoring

### 4.1 AWS Config Rules

```python
# compliance/aws_config/custom_rules.py
import json
import boto3

def evaluate_mfa_enabled(event, context):
    """
    AWS Config Custom Rule: ตรวจสอบว่า IAM users ทุกคนมี MFA
    """
    config = boto3.client('config')
    iam = boto3.client('iam')
    
    invoking_event = json.loads(event['invokingEvent'])
    configuration_item = invoking_event.get('configurationItem', {})
    
    if configuration_item.get('resourceType') != 'AWS::IAM::User':
        return
    
    user_name = configuration_item['resourceName']
    
    # ตรวจสอบ MFA
    try:
        mfa_devices = iam.list_mfa_devices(UserName=user_name)['MFADevices']
        compliant = len(mfa_devices) > 0
    except iam.exceptions.NoSuchEntityException:
        compliant = False
    
    # Report ผล
    config.put_evaluations(
        Evaluations=[
            {
                'ComplianceResourceType': 'AWS::IAM::User',
                'ComplianceResourceId': user_name,
                'ComplianceType': 'COMPLIANT' if compliant else 'NON_COMPLIANT',
                'Annotation': f"MFA {'enabled' if compliant else 'not enabled'} for user {user_name}",
                'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
            }
        ],
        ResultToken=event['resultToken']
    )
```

```yaml
# infrastructure/compliance/aws-config.tf (Terraform)
# AWS Config Rules สำหรับ SOC2/PCI-DSS

resource "aws_config_config_rule" "iam_mfa_enabled" {
  name        = "iam-user-mfa-enabled"
  description = "SOC2 CC6.1: All IAM users must have MFA enabled"

  source {
    owner             = "AWS"
    source_identifier = "IAM_USER_MFA_ENABLED"
  }

  tags = {
    Compliance = "SOC2"
    Control    = "CC6.1"
  }
}

resource "aws_config_config_rule" "rds_encrypted" {
  name        = "rds-storage-encrypted"
  description = "SOC2/PCI-DSS: RDS instances must have storage encryption"

  source {
    owner             = "AWS"
    source_identifier = "RDS_STORAGE_ENCRYPTED"
  }

  tags = {
    Compliance = "SOC2,PCI-DSS"
    Control    = "CC6.7,3.5.1"
  }
}

resource "aws_config_config_rule" "s3_bucket_not_public" {
  name        = "s3-bucket-public-access-prohibited"
  description = "SOC2 CC6.1: S3 buckets must not allow public access"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
}

# Remediation for non-compliant resources
resource "aws_config_remediation_configuration" "enable_s3_block_public" {
  config_rule_name = aws_config_config_rule.s3_bucket_not_public.name
  resource_type    = "AWS::S3::Bucket"
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-DisableS3BucketPublicReadWrite"
  automatic        = true
  maximum_automatic_attempts = 5
  
  parameter {
    name           = "S3BucketName"
    resource_value = "RESOURCE_ID"
  }
}
```

---

## 5. Compliance CI/CD Pipeline

```yaml
# .github/workflows/compliance.yml
name: Compliance Automation

on:
  push:
    branches: [main]
  schedule:
    # Run daily compliance checks
    - cron: '0 1 * * *'  # 08:00 Bangkok time
  workflow_dispatch:
    inputs:
      report_type:
        description: 'Compliance report type'
        type: choice
        options: [soc2, pci-dss, hipaa, all]
        default: all

jobs:
  # ==================== Policy Compliance ====================
  policy-compliance:
    name: Check Policy Compliance
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Terraform compliance checks
        run: |
          cd infrastructure
          terraform plan -out=tfplan.binary
          terraform show -json tfplan.binary > tfplan.json
          conftest test tfplan.json --policy compliance/policies/
      
      - name: Run Kubernetes compliance checks
        run: |
          conftest test k8s/ --policy compliance/policies/kubernetes/
      
      - name: Run Checkov (Infrastructure security)
        uses: bridgecrewio/checkov-action@master
        with:
          directory: infrastructure/
          framework: terraform
          check: CKV_AWS_1,CKV_AWS_2,CKV_AWS_3
          output_format: json
          output_file_path: compliance/reports/checkov-report.json

  # ==================== Evidence Collection ====================
  collect-evidence:
    name: Collect Compliance Evidence
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.COMPLIANCE_AUDIT_ROLE_ARN }}
          aws-region: ap-southeast-1
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
      
      - name: Install dependencies
        run: pip install -r compliance/requirements.txt
      
      - name: Collect evidence
        run: |
          PERIOD_START=$(date -d "30 days ago" +%Y-%m-%dT%H:%M:%S)
          PERIOD_END=$(date +%Y-%m-%dT%H:%M:%S)
          
          python compliance/scripts/collect_evidence.py \
            --start "$PERIOD_START" \
            --end "$PERIOD_END" \
            --output compliance/evidence/$(date +%Y-%m)
      
      - name: Run PCI-DSS checks
        if: inputs.report_type == 'pci-dss' || inputs.report_type == 'all'
        run: |
          python compliance/pci_dss/checks.py \
            --output compliance/reports/pci-dss-$(date +%Y%m%d).json
      
      - name: Generate compliance report
        run: |
          python compliance/scripts/generate_report.py \
            --evidence-dir compliance/evidence/$(date +%Y-%m) \
            --output compliance/reports/compliance-$(date +%Y%m%d).html
      
      - name: Upload evidence to S3
        run: |
          aws s3 sync compliance/evidence/ \
            s3://${{ secrets.COMPLIANCE_BUCKET }}/evidence/ \
            --sse aws:kms \
            --delete
      
      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report-${{ github.run_id }}
          path: compliance/reports/
          retention-days: 365  # Keep for 1 year
      
      - name: Send report to Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Compliance Report Generated",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Daily Compliance Report*\n:white_check_mark: Evidence collected successfully\n:link: <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Details>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_COMPLIANCE_WEBHOOK }}
```

---

## 6. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: SOC 2 Compliance Dashboard

```
สร้าง compliance dashboard ที่แสดง:
1. Current compliance status สำหรับ SOC 2 controls
2. Trend ของ compliance over time (30/60/90 days)
3. Top non-compliant resources
4. Evidence collection status

Implementation:
- Python script สำหรับ collect evidence
- GitHub Actions workflow รัน daily
- S3 bucket สำหรับ store evidence
- Simple HTML dashboard
```

### แบบฝึกหัดที่ 2: Automated Audit Trail

```python
# Implement audit trail system:

class AuditTrailSystem:
    """
    สร้าง audit trail ที่:
    1. Log ทุก deployment event
    2. Log user access patterns
    3. Hash chain verification
    4. Generate audit report สำหรับ period
    5. Alert เมื่อ tamper detected
    """
    
    def log_deployment(self, env, version, deployer, timestamp):
        pass
    
    def log_access(self, resource, user, action, success):
        pass
    
    def verify_period(self, start_date, end_date):
        pass
    
    def generate_audit_report(self, period, output_format='pdf'):
        pass
```

### แบบฝึกหัดที่ 3: Compliance as Code Integration

```
สร้าง PR check ที่:
1. ตรวจสอบ Terraform changes ไม่ละเมิด:
   - SOC 2 CC6.1 (access controls)
   - PCI-DSS 3.5.1 (encryption)
   - PCI-DSS 1.3 (network controls)

2. Block merge ถ้า compliance check fail

3. Generate compliance impact report:
   - Resources affected
   - Controls impacted
   - Risk level (Low/Medium/High/Critical)

4. ส่ง notification ไปยัง compliance team
```

### สรุปบทที่ 68

ในบทนี้เราได้เรียนรู้:
- **Compliance Frameworks**: SOC 2, PCI-DSS, HIPAA requirements
- **Compliance as Code**: นำ policies มาเขียนเป็น code
- **Automated Evidence Collection**: Python scripts สำหรับ collect evidence
- **Audit Trail**: Tamper-evident logging ด้วย hash chaining
- **AWS Config**: Continuous compliance monitoring
- **CI/CD Integration**: GitHub Actions สำหรับ daily compliance checks
- **Reporting**: Generate reports สำหรับ auditors

บทถัดไปเราจะเรียนรู้ Cost Optimization ใน CI/CD pipelines

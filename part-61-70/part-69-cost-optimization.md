# Part 69: Cost Optimization ใน CI/CD

## บทนำ: ทำไม CI/CD ถึงแพง?

CI/CD pipeline เป็น infrastructure ที่รันตลอด 24/7 และมักถูกมองข้ามในแง่ cost optimization ทั้งที่จริงๆ แล้ว CI/CD costs สามารถสูงถึง 20-40% ของ total cloud spend ในบาง organizations

### CI/CD Cost Breakdown

```
ต้นทุนหลักของ CI/CD:
1. Compute (CI runners/agents)     - 40-60% ของ CI cost
2. Storage (artifacts, caches)     - 15-25%
3. Network (data transfer)         - 10-15%
4. Container Registry              - 5-10%
5. Test environments               - 10-20%

Total CI/CD cost สำหรับ mid-size team:
- 10 developers, 50 PRs/day
- GitHub Actions: ~$500-1,500/เดือน
- Self-hosted runners: ~$200-800/เดือน
- Cloud test environments: ~$500-2,000/เดือน
```

---

## 1. Cloud Cost Analysis

### 1.1 Cost Attribution สำหรับ CI/CD

```bash
# ดึง AWS cost ที่เกี่ยวกับ CI/CD ด้วย Cost Explorer API
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --filter '{
    "Tags": {
      "Key": "Purpose",
      "Values": ["ci-cd", "build", "test"]
    }
  }' \
  --metrics "BlendedCost" "UsageQuantity" \
  --group-by Type=TAG,Key=Service
```

```python
# scripts/ci_cost_analysis.py
import boto3
import json
from datetime import datetime, timedelta
from collections import defaultdict

def analyze_ci_cd_costs(days: int = 30):
    """Analyze CI/CD related costs"""
    
    ce = boto3.client('ce', region_name='us-east-1')
    
    end_date = datetime.now().strftime('%Y-%m-%d')
    start_date = (datetime.now() - timedelta(days=days)).strftime('%Y-%m-%d')
    
    # Get costs by service and tagged resources
    response = ce.get_cost_and_usage(
        TimePeriod={
            'Start': start_date,
            'End': end_date
        },
        Granularity='DAILY',
        Filter={
            'Or': [
                # EC2 instances used for CI
                {
                    'Tags': {
                        'Key': 'Name',
                        'Values': ['github-runner', 'jenkins-agent', 'buildkite-agent']
                    }
                },
                # CodeBuild projects
                {
                    'Dimensions': {
                        'Key': 'SERVICE',
                        'Values': ['AWS CodeBuild']
                    }
                },
                # S3 buckets สำหรับ CI artifacts
                {
                    'Tags': {
                        'Key': 'Purpose',
                        'Values': ['ci-artifacts', 'build-cache']
                    }
                }
            ]
        },
        Metrics=['UnblendedCost', 'UsageQuantity'],
        GroupBy=[
            {'Type': 'DIMENSION', 'Key': 'SERVICE'},
            {'Type': 'DIMENSION', 'Key': 'USAGE_TYPE'}
        ]
    )
    
    # Process and summarize
    cost_summary = defaultdict(float)
    
    for result in response['ResultsByTime']:
        for group in result['Groups']:
            service = group['Keys'][0]
            cost = float(group['Metrics']['UnblendedCost']['Amount'])
            cost_summary[service] += cost
    
    # Sort by cost descending
    sorted_costs = sorted(cost_summary.items(), key=lambda x: x[1], reverse=True)
    
    total_cost = sum(cost for _, cost in sorted_costs)
    
    print(f"\n{'='*60}")
    print(f"CI/CD Cost Analysis ({start_date} to {end_date})")
    print(f"{'='*60}")
    print(f"\n{'Service':<40} {'Cost':>10} {'%':>6}")
    print(f"{'-'*58}")
    
    for service, cost in sorted_costs:
        percentage = (cost / total_cost * 100) if total_cost > 0 else 0
        print(f"{service:<40} ${cost:>9.2f} {percentage:>5.1f}%")
    
    print(f"{'-'*58}")
    print(f"{'Total':<40} ${total_cost:>9.2f} 100.0%")
    
    return sorted_costs

def get_runner_utilization():
    """Analyze EC2 runner utilization"""
    
    cloudwatch = boto3.client('cloudwatch')
    ec2 = boto3.client('ec2')
    
    # Get runners tagged as CI agents
    runners = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Purpose', 'Values': ['ci-runner']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )
    
    utilization_data = []
    
    for reservation in runners['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            
            # Get CPU utilization last 7 days
            response = cloudwatch.get_metric_statistics(
                Namespace='AWS/EC2',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                StartTime=datetime.now() - timedelta(days=7),
                EndTime=datetime.now(),
                Period=3600,
                Statistics=['Average', 'Maximum']
            )
            
            datapoints = response['Datapoints']
            
            if datapoints:
                avg_cpu = sum(d['Average'] for d in datapoints) / len(datapoints)
                max_cpu = max(d['Maximum'] for d in datapoints)
                
                utilization_data.append({
                    'instance_id': instance_id,
                    'instance_type': instance['InstanceType'],
                    'avg_cpu': avg_cpu,
                    'max_cpu': max_cpu,
                    'underutilized': avg_cpu < 20,  # < 20% average = underutilized
                    'recommendation': get_recommendation(avg_cpu, max_cpu, instance['InstanceType'])
                })
    
    return utilization_data

def get_recommendation(avg_cpu: float, max_cpu: float, instance_type: str) -> str:
    if avg_cpu < 5:
        return "Consider switching to spot/preemptible or serverless CI"
    elif avg_cpu < 20:
        return f"Downsize instance type (currently {instance_type})"
    elif avg_cpu > 80:
        return f"Upsize instance type or add auto-scaling"
    else:
        return "Optimal utilization"

if __name__ == '__main__':
    analyze_ci_cd_costs(days=30)
    
    utilization = get_runner_utilization()
    print(f"\n{'='*60}")
    print("Runner Utilization Analysis")
    print(f"{'='*60}")
    
    for runner in utilization:
        print(f"\nInstance: {runner['instance_id']} ({runner['instance_type']})")
        print(f"  Avg CPU: {runner['avg_cpu']:.1f}%")
        print(f"  Max CPU: {runner['max_cpu']:.1f}%")
        print(f"  Recommendation: {runner['recommendation']}")
```

---

## 2. Spot/Preemptible Instances สำหรับ CI

### 2.1 GitHub Actions Self-Hosted Runners บน Spot Instances

```python
# scripts/spot_runner_manager.py
import boto3
import time
import json
import logging
from typing import Optional

logger = logging.getLogger(__name__)

class SpotRunnerManager:
    """จัดการ GitHub Actions runners บน AWS Spot Instances"""
    
    def __init__(self, region: str = 'ap-southeast-1'):
        self.ec2 = boto3.client('ec2', region_name=region)
        self.region = region
    
    def launch_spot_runner(
        self,
        runner_type: str = 'medium',  # small, medium, large
        github_token: str = None,
        github_org: str = None,
        max_price: float = None
    ) -> Optional[str]:
        """Launch a spot instance as GitHub Actions runner"""
        
        configs = {
            'small': {'instance_types': ['t3.small', 't3a.small', 't2.small'], 'max_price': 0.02},
            'medium': {'instance_types': ['c5.xlarge', 'c5a.xlarge', 'm5.xlarge'], 'max_price': 0.05},
            'large': {'instance_types': ['c5.2xlarge', 'c5a.2xlarge', 'm5.2xlarge'], 'max_price': 0.10},
        }
        
        config = configs.get(runner_type, configs['medium'])
        spot_price = max_price or config['max_price']
        
        # User data สำหรับ auto-configure runner
        user_data = f"""#!/bin/bash
set -e

# Update system
apt-get update -y
apt-get install -y curl jq docker.io

# Install GitHub Actions runner
mkdir -p /opt/github-runner
cd /opt/github-runner

# Download runner
RUNNER_VERSION=$(curl -s https://api.github.com/repos/actions/runner/releases/latest | jq -r '.tag_name' | sed 's/v//')
curl -O -L "https://github.com/actions/runner/releases/download/v${{RUNNER_VERSION}}/actions-runner-linux-x64-${{RUNNER_VERSION}}.tar.gz"
tar xzf ./actions-runner-linux-x64-${{RUNNER_VERSION}}.tar.gz

# Get registration token
REG_TOKEN=$(curl -s -X POST \\
  -H "Authorization: token {github_token}" \\
  -H "Accept: application/vnd.github.v3+json" \\
  "https://api.github.com/orgs/{github_org}/actions/runners/registration-token" \\
  | jq -r '.token')

# Configure runner
./config.sh \\
  --url "https://github.com/{github_org}" \\
  --token "$REG_TOKEN" \\
  --name "spot-runner-$(hostname)" \\
  --labels "spot,linux,{runner_type}" \\
  --unattended \\
  --ephemeral  # Runner terminates after one job

# Start runner
./run.sh &

# Terminate instance after job completes or after max 1 hour
sleep 3600 && aws ec2 terminate-instances --instance-ids $(curl -s http://169.254.169.254/latest/meta-data/instance-id) &
"""
        
        try:
            # Request spot instance
            response = self.ec2.request_spot_instances(
                SpotPrice=str(spot_price),
                InstanceCount=1,
                Type='one-time',
                LaunchSpecification={
                    'ImageId': self._get_latest_ubuntu_ami(),
                    'InstanceType': config['instance_types'][0],
                    'SecurityGroupIds': ['sg-ci-runners'],
                    'SubnetId': 'subnet-private-a',
                    'IamInstanceProfile': {
                        'Name': 'github-runner-profile'
                    },
                    'UserData': user_data,
                    'BlockDeviceMappings': [
                        {
                            'DeviceName': '/dev/xvda',
                            'Ebs': {
                                'VolumeSize': 50,
                                'VolumeType': 'gp3',
                                'DeleteOnTermination': True
                            }
                        }
                    ]
                }
            )
            
            request_id = response['SpotInstanceRequests'][0]['SpotInstanceRequestId']
            logger.info(f"Spot request created: {request_id}")
            
            # Wait for fulfillment
            instance_id = self._wait_for_spot_fulfillment(request_id)
            
            if instance_id:
                # Tag instance
                self.ec2.create_tags(
                    Resources=[instance_id],
                    Tags=[
                        {'Key': 'Purpose', 'Value': 'ci-runner'},
                        {'Key': 'RunnerType', 'Value': runner_type},
                        {'Key': 'Name', 'Value': f'github-spot-runner-{runner_type}'}
                    ]
                )
                
                logger.info(f"Spot runner launched: {instance_id}")
                return instance_id
            
        except self.ec2.exceptions.InsufficientInstanceCapacity:
            logger.warning("Spot capacity unavailable, trying different instance types...")
            # Try other instance types
            for instance_type in config['instance_types'][1:]:
                try:
                    # Retry with different instance type
                    pass
                except Exception:
                    continue
        
        return None
    
    def _wait_for_spot_fulfillment(self, request_id: str, timeout: int = 300) -> Optional[str]:
        """รอจนกว่า spot instance จะถูก fulfill"""
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            response = self.ec2.describe_spot_instance_requests(
                SpotInstanceRequestIds=[request_id]
            )
            
            request = response['SpotInstanceRequests'][0]
            state = request['State']
            
            if state == 'active' and request.get('InstanceId'):
                return request['InstanceId']
            elif state in ['cancelled', 'failed']:
                logger.error(f"Spot request {state}: {request.get('Status', {})}")
                return None
            
            time.sleep(10)
        
        logger.error(f"Spot request timeout after {timeout}s")
        return None
    
    def _get_latest_ubuntu_ami(self) -> str:
        """ดึง AMI ID ของ Ubuntu 22.04 ล่าสุด"""
        response = self.ec2.describe_images(
            Owners=['099720109477'],  # Canonical
            Filters=[
                {'Name': 'name', 'Values': ['ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*']},
                {'Name': 'state', 'Values': ['available']}
            ]
        )
        
        images = sorted(
            response['Images'],
            key=lambda x: x['CreationDate'],
            reverse=True
        )
        
        return images[0]['ImageId']

    def get_spot_price_history(self, instance_type: str, days: int = 7) -> list:
        """ดู spot price history สำหรับ instance type"""
        from datetime import datetime, timedelta
        
        response = self.ec2.describe_spot_price_history(
            InstanceTypes=[instance_type],
            ProductDescriptions=['Linux/UNIX'],
            StartTime=datetime.now() - timedelta(days=days),
            EndTime=datetime.now()
        )
        
        prices = [
            {
                'timestamp': p['Timestamp'].isoformat(),
                'az': p['AvailabilityZone'],
                'price': float(p['SpotPrice'])
            }
            for p in response['SpotPriceHistory']
        ]
        
        if prices:
            avg_price = sum(p['price'] for p in prices) / len(prices)
            min_price = min(p['price'] for p in prices)
            max_price = max(p['price'] for p in prices)
            
            print(f"\nSpot Price History for {instance_type} (last {days} days):")
            print(f"  Average: ${avg_price:.4f}/hr")
            print(f"  Min: ${min_price:.4f}/hr")
            print(f"  Max: ${max_price:.4f}/hr")
            
            # On-demand comparison
            on_demand_prices = {
                'c5.xlarge': 0.17,
                'c5.2xlarge': 0.34,
                'm5.xlarge': 0.192,
            }
            
            if instance_type in on_demand_prices:
                on_demand = on_demand_prices[instance_type]
                savings = (1 - avg_price / on_demand) * 100
                print(f"  On-demand: ${on_demand:.4f}/hr")
                print(f"  Savings: {savings:.1f}%")
        
        return prices
```

### 2.2 Auto-scaling CI Runners

```yaml
# infrastructure/ci-runners/autoscaling.yaml
# AWS Auto Scaling Group สำหรับ CI runners

resource "aws_autoscaling_group" "ci_runners" {
  name                = "github-ci-runners"
  min_size            = 0
  max_size            = 20
  desired_capacity    = 0
  
  mixed_instances_policy {
    instances_distribution {
      # ใช้ spot ให้มากที่สุด
      on_demand_base_capacity                  = 0
      on_demand_percentage_above_base_capacity = 0
      spot_allocation_strategy                 = "capacity-optimized"
    }
    
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.ci_runner.id
        version            = "$Latest"
      }
      
      # Instance type overrides (จาก cheapest ที่ capable)
      override {
        instance_type = "c5.xlarge"
      }
      override {
        instance_type = "c5a.xlarge"
      }
      override {
        instance_type = "c4.xlarge"
      }
      override {
        instance_type = "m5.xlarge"
      }
      override {
        instance_type = "m5a.xlarge"
      }
    }
  }
  
  tag {
    key                 = "Purpose"
    value               = "ci-runner"
    propagate_at_launch = true
  }
  
  # Scale to 0 when idle
  lifecycle {
    create_before_destroy = true
  }
}

# Lambda function ที่ scale up เมื่อมี job รอ
resource "aws_lambda_function" "scale_runners" {
  function_name = "scale-ci-runners"
  role          = aws_iam_role.runner_scaler.arn
  handler       = "index.handler"
  runtime       = "nodejs18.x"
  timeout       = 60
  
  environment {
    variables = {
      ASG_NAME         = aws_autoscaling_group.ci_runners.name
      GITHUB_TOKEN     = var.github_token
      GITHUB_ORG       = var.github_org
      MAX_RUNNERS      = "20"
      JOBS_PER_RUNNER  = "1"
    }
  }
}
```

---

## 3. Pipeline Resource Optimization

### 3.1 Optimizing GitHub Actions

```yaml
# ก่อน Optimize: ช้า แพง
jobs:
  test:
    runs-on: ubuntu-latest  # ไม่จำเป็นต้อง latest
    steps:
      - uses: actions/checkout@v4
      - run: npm install        # ไม่มี cache
      - run: npm test           # รัน test ทุกตัว

# หลัง Optimize: เร็วขึ้น ถูกลง
jobs:
  test:
    runs-on: ubuntu-22.04  # Pin version = reproducible + cheaper
    
    steps:
      - uses: actions/checkout@v4
        with:
          # ดึงแค่ shallow clone ถ้าไม่ต้องการ history
          fetch-depth: 1
      
      # Cache node_modules
      - name: Cache dependencies
        uses: actions/cache@v4
        id: cache-deps
        with:
          path: ~/.npm
          key: node-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            node-${{ runner.os }}-
      
      - name: Install dependencies
        if: steps.cache-deps.outputs.cache-hit != 'true'
        run: npm ci
      
      # Run only changed tests
      - name: Run affected tests
        run: |
          CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
          npm test -- --findRelatedTests $CHANGED_FILES
```

### 3.2 Caching Strategies

```yaml
# .github/workflows/optimized-pipeline.yml
name: Optimized CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-22.04
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      
      # ==================== Multi-level caching ====================
      
      # Level 1: Node modules cache (by lockfile hash)
      - name: Cache node modules
        uses: actions/cache@v4
        id: npm-cache
        with:
          path: |
            ~/.npm
            node_modules
          key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
          restore-keys: npm-${{ runner.os }}-
      
      # Level 2: Build cache (by source hash)
      - name: Cache build
        uses: actions/cache@v4
        id: build-cache
        with:
          path: .next/cache
          key: nextjs-${{ runner.os }}-${{ hashFiles('**.[jt]s', '**.[jt]sx') }}
          restore-keys: nextjs-${{ runner.os }}-
      
      # Level 3: Test cache
      - name: Cache jest
        uses: actions/cache@v4
        with:
          path: .jest-cache
          key: jest-${{ runner.os }}-${{ hashFiles('jest.config.*') }}-${{ github.sha }}
          restore-keys: |
            jest-${{ runner.os }}-${{ hashFiles('jest.config.*') }}-
            jest-${{ runner.os }}-
      
      - name: Install (if not cached)
        if: steps.npm-cache.outputs.cache-hit != 'true'
        run: npm ci
      
      - name: Run tests
        run: |
          npx jest \
            --cacheDirectory=.jest-cache \
            --maxWorkers=4 \
            --ci
      
      - name: Build (if not cached)
        if: steps.build-cache.outputs.cache-hit != 'true'
        run: npm run build

  # ==================== Parallel jobs ====================
  
  lint:
    runs-on: ubuntu-22.04
    # ทำงาน parallel กับ test
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 1
      
      - name: Cache node modules
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: npm-${{ runner.os }}-${{ hashFiles('package-lock.json') }}
      
      - run: npm ci --ignore-scripts
      - run: npm run lint
      - run: npm run type-check

  # ==================== Skip unchanged paths ====================
  
  backend-test:
    runs-on: ubuntu-22.04
    # Skip ถ้า backend ไม่มีการเปลี่ยนแปลง
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Check if backend changed
        id: backend-changed
        run: |
          git diff --name-only HEAD~1 HEAD | grep -q "^backend/" && echo "changed=true" >> $GITHUB_OUTPUT || echo "changed=false" >> $GITHUB_OUTPUT
      
      - name: Run backend tests
        if: steps.backend-changed.outputs.changed == 'true'
        run: |
          cd backend
          npm test
```

### 3.3 Docker Build Optimization

```dockerfile
# Dockerfile ที่ optimize แล้ว

# ==================== Build stage ====================
# ใช้ build cache อย่างมีประสิทธิภาพ
FROM node:20-alpine AS deps
WORKDIR /app

# Copy lockfile ก่อน (แยก layer ออกมาเพื่อ cache)
COPY package.json package-lock.json ./
RUN npm ci --only=production

# ==================== Build stage ====================
FROM node:20-alpine AS builder
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build
RUN npm run build

# ==================== Production stage ====================
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production

# Create non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs

# Copy only necessary files
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

```yaml
# GitHub Actions ด้วย Docker layer caching
- name: Setup Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: registry.io/myapp:${{ github.sha }}
    cache-from: |
      type=registry,ref=registry.io/myapp:buildcache
      type=registry,ref=registry.io/myapp:latest
    cache-to: type=registry,ref=registry.io/myapp:buildcache,mode=max
```

---

## 4. Test Environment Cost Optimization

### 4.1 Ephemeral Test Environments

```python
# scripts/ephemeral_env.py
import boto3
import subprocess
import os
from datetime import datetime, timedelta

class EphemeralEnvironment:
    """สร้าง/ลบ test environments อัตโนมัติ"""
    
    def __init__(self, pr_number: int, project: str):
        self.pr_number = pr_number
        self.project = project
        self.env_name = f"{project}-pr-{pr_number}"
        self.cf = boto3.client('cloudformation')
        self.ssm = boto3.client('ssm')
    
    def create(self, stack_template: str) -> dict:
        """สร้าง ephemeral environment สำหรับ PR"""
        
        print(f"Creating ephemeral environment: {self.env_name}")
        
        # Store metadata
        self.ssm.put_parameter(
            Name=f"/ephemeral/{self.env_name}/metadata",
            Value=str({
                'created_at': datetime.now().isoformat(),
                'pr_number': self.pr_number,
                'expires_at': (datetime.now() + timedelta(hours=4)).isoformat()
            }),
            Type='String',
            Overwrite=True
        )
        
        # Deploy stack
        response = self.cf.create_stack(
            StackName=self.env_name,
            TemplateBody=stack_template,
            Parameters=[
                {'ParameterKey': 'Environment', 'ParameterValue': 'ephemeral'},
                {'ParameterKey': 'PRNumber', 'ParameterValue': str(self.pr_number)},
            ],
            Tags=[
                {'Key': 'Type', 'Value': 'ephemeral'},
                {'Key': 'PRNumber', 'Value': str(self.pr_number)},
                {'Key': 'ExpiresAt', 'Value': (datetime.now() + timedelta(hours=4)).isoformat()},
                {'Key': 'Cost:Ephemeral', 'Value': 'true'}
            ],
            OnFailure='DELETE'  # Auto-delete หาก create ล้มเหลว
        )
        
        # Wait for completion
        waiter = self.cf.get_waiter('stack_create_complete')
        waiter.wait(StackName=self.env_name)
        
        # Get outputs
        stack = self.cf.describe_stacks(StackName=self.env_name)['Stacks'][0]
        outputs = {o['OutputKey']: o['OutputValue'] for o in stack.get('Outputs', [])}
        
        print(f"Environment created: {outputs}")
        return outputs
    
    def destroy(self) -> bool:
        """ลบ ephemeral environment"""
        
        try:
            print(f"Destroying ephemeral environment: {self.env_name}")
            
            self.cf.delete_stack(StackName=self.env_name)
            
            waiter = self.cf.get_waiter('stack_delete_complete')
            waiter.wait(StackName=self.env_name)
            
            # Clean up SSM parameter
            self.ssm.delete_parameter(
                Name=f"/ephemeral/{self.env_name}/metadata"
            )
            
            print(f"Environment {self.env_name} destroyed successfully")
            return True
            
        except self.cf.exceptions.ClientError as e:
            if 'does not exist' in str(e):
                print(f"Environment {self.env_name} already deleted")
                return True
            raise

def cleanup_expired_environments():
    """ลบ environments ที่หมดอายุแล้ว"""
    
    cf = boto3.client('cloudformation')
    ssm = boto3.client('ssm')
    
    # List all ephemeral stacks
    paginator = cf.get_paginator('list_stacks')
    
    for page in paginator.paginate(StackStatusFilter=['CREATE_COMPLETE', 'UPDATE_COMPLETE']):
        for stack in page['StackSummaries']:
            # Check if ephemeral
            tags = {t['Key']: t['Value'] for t in 
                   cf.describe_stacks(StackName=stack['StackName'])['Stacks'][0].get('Tags', [])}
            
            if tags.get('Type') == 'ephemeral':
                expires_at_str = tags.get('ExpiresAt')
                
                if expires_at_str:
                    expires_at = datetime.fromisoformat(expires_at_str)
                    
                    if datetime.now() > expires_at:
                        print(f"Deleting expired environment: {stack['StackName']}")
                        cf.delete_stack(StackName=stack['StackName'])
                        
                        total_hours = (datetime.now() - stack['CreationTime'].replace(tzinfo=None)).total_seconds() / 3600
                        print(f"  Existed for {total_hours:.1f} hours")
```

---

## 5. Monitoring และ Cost Alerts

### 5.1 Cost Anomaly Detection

```python
# monitoring/cost_monitor.py
import boto3
import json
from datetime import datetime

def setup_cost_anomaly_monitoring():
    """Setup AWS Cost Anomaly Detection สำหรับ CI/CD"""
    
    ce = boto3.client('ce', region_name='us-east-1')
    sns = boto3.client('sns')
    
    # Create SNS topic for alerts
    topic_response = sns.create_topic(Name='ci-cd-cost-alerts')
    topic_arn = topic_response['TopicArn']
    
    # Subscribe email
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='email',
        Endpoint='devops@company.com'
    )
    
    # Create cost anomaly monitor
    monitor_response = ce.create_anomaly_monitor(
        AnomalyMonitor={
            'MonitorName': 'CI-CD-Cost-Monitor',
            'MonitorType': 'DIMENSIONAL',
            'MonitorDimension': 'SERVICE'
        }
    )
    
    monitor_arn = monitor_response['MonitorArn']
    
    # Create subscription
    ce.create_anomaly_subscription(
        AnomalySubscription={
            'MonitorArnList': [monitor_arn],
            'Subscribers': [
                {
                    'Address': topic_arn,
                    'Type': 'SNS'
                }
            ],
            'Threshold': 20.0,  # Alert เมื่อ cost เพิ่มขึ้น > $20
            'Frequency': 'DAILY',
            'SubscriptionName': 'CI-CD-Daily-Anomaly-Alert'
        }
    )
    
    print(f"Cost anomaly monitoring setup complete")
    print(f"Monitor ARN: {monitor_arn}")
    print(f"SNS Topic ARN: {topic_arn}")

def create_budget_alert(monthly_budget: float = 500.0):
    """สร้าง budget alert สำหรับ CI/CD costs"""
    
    budgets = boto3.client('budgets', region_name='us-east-1')
    account_id = boto3.client('sts').get_caller_identity()['Account']
    
    budgets.create_budget(
        AccountId=account_id,
        Budget={
            'BudgetName': 'CI-CD-Monthly-Budget',
            'BudgetLimit': {
                'Amount': str(monthly_budget),
                'Unit': 'USD'
            },
            'TimeUnit': 'MONTHLY',
            'BudgetType': 'COST',
            'CostFilters': {
                'TagKeyValue': ['user:Purpose$ci-cd']
            }
        },
        NotificationsWithSubscribers=[
            {
                'Notification': {
                    'NotificationType': 'ACTUAL',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 80,  # Alert ที่ 80% ของ budget
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': 'devops@company.com'
                    }
                ]
            },
            {
                'Notification': {
                    'NotificationType': 'FORECASTED',
                    'ComparisonOperator': 'GREATER_THAN',
                    'Threshold': 100,  # Alert เมื่อ forecast เกิน 100%
                    'ThresholdType': 'PERCENTAGE'
                },
                'Subscribers': [
                    {
                        'SubscriptionType': 'EMAIL',
                        'Address': 'engineering-lead@company.com'
                    }
                ]
            }
        ]
    )
    
    print(f"Budget alert created: ${monthly_budget}/month")
```

---

## 6. Cost Optimization Checklist

### 6.1 GitHub Actions Optimization Checklist

```markdown
## GitHub Actions Cost Optimization Checklist

### Immediate Wins (Low effort, high impact)
- [ ] Enable dependency caching (npm, pip, maven, gradle)
- [ ] Use `fetch-depth: 1` สำหรับ shallow clones
- [ ] Cancel outdated runs เมื่อมี push ใหม่
- [ ] ใช้ path filters เพื่อ skip irrelevant jobs

### Cache Strategy
- [ ] Cache dependency installations (node_modules, .venv, etc)
- [ ] Cache build outputs (.next/cache, target/, build/)
- [ ] Cache Docker layers ด้วย registry cache
- [ ] Cache test results ด้วย Jest cache

### Resource Right-sizing
- [ ] วิเคราะห์ runner utilization
- [ ] ลด maxWorkers สำหรับ parallel tests
- [ ] ใช้ smaller runners สำหรับ lint/type-check
- [ ] ใช้ larger runners สำหรับ builds เท่านั้น

### Test Optimization
- [ ] Run only affected tests (Jest --findRelatedTests)
- [ ] Parallel test execution
- [ ] Test splitting ด้วย split-tests
- [ ] Skip tests ถ้า no relevant file changed

### Pipeline Design
- [ ] ใช้ concurrent jobs แทน sequential
- [ ] Fast fail: run quick checks ก่อน
- [ ] Reuse build artifacts ระหว่าง jobs
- [ ] ลบ unused workflows
```

---

## 7. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Cost Audit Pipeline

```
สร้าง GitHub Actions workflow ที่รัน weekly และ:
1. ดึง CI/CD costs จาก AWS Cost Explorer
2. เปรียบเทียบกับ previous week
3. Highlight cost spikes > 20%
4. Generate optimization recommendations
5. ส่ง Slack report พร้อม breakdown ตาม:
   - Team
   - Project
   - Resource type

Budget thresholds:
- Per-team monthly: $200
- Per-project weekly: $50
- Single pipeline run: $5
```

### แบบฝึกหัดที่ 2: Cache Hit Rate Dashboard

```python
# สร้าง script ที่:
# 1. ดึง cache hit/miss statistics จาก GitHub Actions API
# 2. คำนวณ estimated time saved
# 3. คำนวณ estimated cost saved
# 4. แสดง recommendations ว่า cache ตัวไหนไม่ efficient

class CacheAnalyzer:
    
    def analyze_cache_effectiveness(self, workflow_runs: list) -> dict:
        """
        Returns:
        {
            "cache_hit_rate": 0.75,
            "avg_time_saved_per_run": 180,  # seconds
            "monthly_cost_savings": 45.0,   # USD
            "inefficient_caches": [
                {"key_pattern": "npm-...", "hit_rate": 0.2, "recommendation": "..."}
            ]
        }
        """
        pass
```

### แบบฝึกหัดที่ 3: Spot Runner Auto-scaling

```
Setup auto-scaling สำหรับ GitHub Actions runners:

1. สร้าง Lambda ที่ poll GitHub API ทุก 30 วินาที:
   - จำนวน queued jobs
   - จำนวน available runners
   
2. Auto-scale เมื่อ:
   - Queued > 0 → เพิ่ม runners
   - All jobs complete → ลด runners เป็น 0
   
3. Implement spot interruption handling:
   - Receive instance interruption warning (2 minutes)
   - Gracefully complete current job หรือ re-queue

4. Cost tracking:
   - บันทึก spot vs on-demand cost
   - คำนวณ savings per month
```

### สรุปบทที่ 69

ในบทนี้เราได้เรียนรู้:
- **Cost Analysis**: วิเคราะห์และ track CI/CD costs
- **Spot Instances**: ประหยัด 60-90% ด้วย spot instances สำหรับ CI runners
- **Caching**: Multi-level caching strategies สำหรับ dependencies, builds, tests
- **Pipeline Optimization**: Parallelism, path filters, shallow clones
- **Ephemeral Environments**: สร้าง/ลบ test environments อัตโนมัติ
- **Cost Monitoring**: Budget alerts และ anomaly detection
- **Docker Optimization**: Multi-stage builds, layer caching

บทถัดไปเราจะเรียนรู้ Developer Experience (DX) Optimization - บทสุดท้ายของ series นี้

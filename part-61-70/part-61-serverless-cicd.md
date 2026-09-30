# Part 61: Serverless CI/CD

## บทนำ: ทำไม Serverless จึงเปลี่ยนวิธี Deploy?

Serverless computing เปลี่ยนวิธีที่เราสร้างและ deploy แอปพลิเคชันอย่างสิ้นเชิง แทนที่จะต้องจัดการ server, container หรือ VM เราสามารถโฟกัสที่โค้ดและ business logic ได้เลย อย่างไรก็ตาม Serverless มาพร้อมกับความท้าทายด้าน CI/CD ที่แตกต่างจาก traditional deployment อย่างมาก

### ความแตกต่างหลักระหว่าง Traditional และ Serverless CI/CD

| ด้าน | Traditional | Serverless |
|------|-------------|------------|
| Unit of Deployment | Container/VM | Function |
| State Management | Persistent | Stateless |
| Scaling | Manual/Auto-scaling | Automatic |
| Cold Start | ไม่มี | มีผลกระทบ |
| Testing | Integration เต็มรูปแบบ | Mock/Local emulation |
| Cost Model | Pay for uptime | Pay per invocation |
| Deployment Time | นาที | วินาที |

### Challenges ของ Serverless CI/CD

1. **Cold Start Problem**: Function ที่ไม่ได้ถูกเรียกนานอาจใช้เวลาเริ่มต้นนาน
2. **Local Development Parity**: ยากต่อการจำลอง cloud environment ใน local
3. **Integration Testing**: ต้องทดสอบการทำงานร่วมกันระหว่าง services หลายตัว
4. **Dependency Management**: แต่ละ function มี dependencies ของตัวเอง
5. **Environment Variables**: จัดการ secrets และ config อย่างปลอดภัย

---

## 1. AWS Lambda และ SAM (Serverless Application Model)

### 1.1 AWS SAM คืออะไร?

AWS Serverless Application Model (SAM) คือ open-source framework สำหรับสร้าง serverless applications บน AWS โดย extend CloudFormation template ให้มี syntax ที่กระชับกว่า

### 1.2 โครงสร้าง SAM Project

```
my-serverless-app/
├── template.yaml          # SAM template หลัก
├── samconfig.toml         # SAM CLI configuration
├── src/
│   ├── handlers/
│   │   ├── createUser/
│   │   │   ├── index.js
│   │   │   └── package.json
│   │   ├── getUser/
│   │   │   ├── index.js
│   │   │   └── package.json
│   │   └── deleteUser/
│   │       ├── index.js
│   │       └── package.json
│   └── shared/
│       ├── database.js
│       └── utils.js
├── events/                # Test events
│   ├── create-user.json
│   └── get-user.json
├── tests/
│   ├── unit/
│   └── integration/
└── .github/
    └── workflows/
        └── deploy.yml
```

### 1.3 SAM Template ตัวอย่าง

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: User Management API

Globals:
  Function:
    Timeout: 30
    MemorySize: 256
    Runtime: nodejs18.x
    Environment:
      Variables:
        TABLE_NAME: !Ref UsersTable
        LOG_LEVEL: !Ref LogLevel
    Tracing: Active
    Layers:
      - !Ref SharedLayer

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev
  LogLevel:
    Type: String
    Default: INFO

Resources:
  # API Gateway
  UsersApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: !Ref Environment
      Cors:
        AllowMethods: "'GET,POST,PUT,DELETE,OPTIONS'"
        AllowHeaders: "'Content-Type,Authorization'"
        AllowOrigin: "'*'"
      Auth:
        DefaultAuthorizer: JwtAuthorizer
        Authorizers:
          JwtAuthorizer:
            JwtConfiguration:
              issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${UserPool}"
              audience:
                - !Ref UserPoolClient
            IdentitySource: "$request.header.Authorization"

  # Lambda Functions
  CreateUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/handlers/createUser/
      Handler: index.handler
      Description: Create new user
      Events:
        ApiEvent:
          Type: Api
          Properties:
            RestApiId: !Ref UsersApi
            Path: /users
            Method: POST
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref UsersTable
        - SNSPublishMessagePolicy:
            TopicName: !GetAtt UserEventsTopic.TopicName

  GetUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/handlers/getUser/
      Handler: index.handler
      Events:
        ApiEvent:
          Type: Api
          Properties:
            RestApiId: !Ref UsersApi
            Path: /users/{userId}
            Method: GET
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref UsersTable

  # Shared Lambda Layer
  SharedLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: shared-utilities
      ContentUri: src/shared/
      CompatibleRuntimes:
        - nodejs18.x
    Metadata:
      BuildMethod: nodejs18.x

  # DynamoDB Table
  UsersTable:
    Type: AWS::DynamoDB::Table
    DeletionPolicy: Retain
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: userId
          AttributeType: S
        - AttributeName: email
          AttributeType: S
      KeySchema:
        - AttributeName: userId
          KeyType: HASH
      GlobalSecondaryIndexes:
        - IndexName: email-index
          KeySchema:
            - AttributeName: email
              KeyType: HASH
          Projection:
            ProjectionType: ALL
      PointInTimeRecoverySpecification:
        PointInTimeRecoveryEnabled: true

  # SNS Topic
  UserEventsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub "user-events-${Environment}"

Outputs:
  ApiUrl:
    Description: API Gateway endpoint URL
    Value: !Sub "https://${UsersApi}.execute-api.${AWS::Region}.amazonaws.com/${Environment}"
  CreateUserFunctionArn:
    Value: !GetAtt CreateUserFunction.Arn
```

### 1.4 Handler Code ตัวอย่าง

```javascript
// src/handlers/createUser/index.js
const { DynamoDBClient, PutItemCommand } = require('@aws-sdk/client-dynamodb');
const { marshall } = require('@aws-sdk/util-dynamodb');
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');
const { v4: uuidv4 } = require('uuid');

const dynamodb = new DynamoDBClient({});
const sns = new SNSClient({});

const TABLE_NAME = process.env.TABLE_NAME;
const TOPIC_ARN = process.env.TOPIC_ARN;

exports.handler = async (event) => {
  try {
    const body = JSON.parse(event.body);
    
    // Validation
    if (!body.email || !body.name) {
      return {
        statusCode: 400,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ error: 'email and name are required' })
      };
    }

    const userId = uuidv4();
    const now = new Date().toISOString();

    const user = {
      userId,
      email: body.email,
      name: body.name,
      createdAt: now,
      updatedAt: now
    };

    // Save to DynamoDB
    await dynamodb.send(new PutItemCommand({
      TableName: TABLE_NAME,
      Item: marshall(user),
      ConditionExpression: 'attribute_not_exists(email)'
    }));

    // Publish event
    await sns.send(new PublishCommand({
      TopicArn: TOPIC_ARN,
      Message: JSON.stringify({ event: 'USER_CREATED', data: user }),
      MessageAttributes: {
        eventType: {
          DataType: 'String',
          StringValue: 'USER_CREATED'
        }
      }
    }));

    return {
      statusCode: 201,
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(user)
    };
  } catch (error) {
    if (error.name === 'ConditionalCheckFailedException') {
      return {
        statusCode: 409,
        body: JSON.stringify({ error: 'Email already exists' })
      };
    }
    
    console.error('Error creating user:', error);
    return {
      statusCode: 500,
      body: JSON.stringify({ error: 'Internal server error' })
    };
  }
};
```

### 1.5 SAM CLI Configuration

```toml
# samconfig.toml
version = 0.1

[default.global.parameters]
stack_name = "user-management-api"
region = "ap-southeast-1"

[default.build.parameters]
cached = true
parallel = true

[default.validate.parameters]
lint = true

[dev.deploy.parameters]
stack_name = "user-management-api-dev"
s3_bucket = "my-sam-deployments-dev"
s3_prefix = "user-management-api"
parameter_overrides = "Environment=dev LogLevel=DEBUG"
capabilities = "CAPABILITY_IAM CAPABILITY_AUTO_EXPAND"
confirm_changeset = false
fail_on_empty_changeset = false

[staging.deploy.parameters]
stack_name = "user-management-api-staging"
s3_bucket = "my-sam-deployments-staging"
parameter_overrides = "Environment=staging LogLevel=INFO"
capabilities = "CAPABILITY_IAM CAPABILITY_AUTO_EXPAND"
confirm_changeset = true

[prod.deploy.parameters]
stack_name = "user-management-api-prod"
s3_bucket = "my-sam-deployments-prod"
parameter_overrides = "Environment=prod LogLevel=WARN"
capabilities = "CAPABILITY_IAM CAPABILITY_AUTO_EXPAND"
confirm_changeset = true
```

---

## 2. GitHub Actions สำหรับ AWS SAM

### 2.1 Complete CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Serverless CI/CD Pipeline

on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'template.yaml'
      - '.github/workflows/**'
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'dev'
        type: choice
        options: [dev, staging, prod]

env:
  AWS_REGION: ap-southeast-1
  NODE_VERSION: '18'
  SAM_VERSION: '1.96.0'

jobs:
  # ==================== Test ====================
  test:
    name: Test Lambda Functions
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: '**/package-lock.json'
      
      - name: Install dependencies
        run: |
          for dir in src/handlers/*/; do
            echo "Installing deps in $dir"
            npm ci --prefix "$dir"
          done
          npm ci --prefix src/shared
      
      - name: Run unit tests
        run: |
          npm test -- --coverage --ci
        env:
          TABLE_NAME: test-table
          TOPIC_ARN: arn:aws:sns:ap-southeast-1:123456789:test-topic
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
          fail_ci_if_error: false

  # ==================== Security Scan ====================
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run npm audit
        run: |
          for dir in src/handlers/*/; do
            npm audit --prefix "$dir" --audit-level=high
          done
      
      - name: Scan with Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          file: template.yaml
          framework: cloudformation
          output_format: sarif
          output_file_path: checkov.sarif
      
      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: checkov.sarif

  # ==================== Build ====================
  build:
    name: Build SAM Application
    runs-on: ubuntu-latest
    needs: [test, security]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python (for SAM CLI)
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install SAM CLI
        run: |
          pip install aws-sam-cli==${{ env.SAM_VERSION }}
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Validate SAM template
        run: sam validate --lint
      
      - name: Build SAM application
        run: |
          sam build \
            --parallel \
            --cached \
            --use-container
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: sam-build-${{ github.sha }}
          path: .aws-sam/build/
          retention-days: 7

  # ==================== Deploy Dev ====================
  deploy-dev:
    name: Deploy to Dev
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop' || github.event_name == 'workflow_dispatch'
    environment: dev
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: sam-build-${{ github.sha }}
          path: .aws-sam/build/
      
      - name: Install SAM CLI
        run: pip install aws-sam-cli==${{ env.SAM_VERSION }}
      
      - name: Configure AWS credentials (Dev)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.DEV_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.DEV_AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to Dev
        run: |
          sam deploy \
            --config-env dev \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset \
            --resolve-s3
      
      - name: Run smoke tests
        run: |
          API_URL=$(aws cloudformation describe-stacks \
            --stack-name user-management-api-dev \
            --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" \
            --output text)
          
          # Test health endpoint
          curl -f "${API_URL}/health" || exit 1
          echo "Smoke tests passed!"

  # ==================== Deploy Staging ====================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: sam-build-${{ github.sha }}
          path: .aws-sam/build/
      
      - name: Install SAM CLI
        run: pip install aws-sam-cli==${{ env.SAM_VERSION }}
      
      - name: Configure AWS credentials (Staging)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.STAGING_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to Staging
        run: |
          sam deploy \
            --config-env staging \
            --no-fail-on-empty-changeset
      
      - name: Run integration tests
        run: |
          API_URL=$(aws cloudformation describe-stacks \
            --stack-name user-management-api-staging \
            --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" \
            --output text)
          
          npm run test:integration -- --api-url="${API_URL}"

  # ==================== Deploy Production ====================
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: sam-build-${{ github.sha }}
          path: .aws-sam/build/
      
      - name: Install SAM CLI
        run: pip install aws-sam-cli==${{ env.SAM_VERSION }}
      
      - name: Configure AWS credentials (Prod)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.PROD_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to Production with traffic shifting
        run: |
          sam deploy \
            --config-env prod \
            --no-fail-on-empty-changeset
      
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Production deployment ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment* ${{ job.status == 'success' && ':white_check_mark:' || ':x:' }}\n*Commit:* ${{ github.sha }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 3. Serverless Framework

### 3.1 Serverless Framework vs SAM

```
เปรียบเทียบ:
- SAM: AWS-specific, native CloudFormation, free
- Serverless Framework: Multi-provider, plugin ecosystem, community large
```

### 3.2 Serverless Framework Configuration

```yaml
# serverless.yml
service: user-api

frameworkVersion: '3'

useDotenv: true

provider:
  name: aws
  runtime: nodejs18.x
  region: ${opt:region, 'ap-southeast-1'}
  stage: ${opt:stage, 'dev'}
  
  # IAM Role
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
            - dynamodb:Query
            - dynamodb:Scan
          Resource:
            - !GetAtt UsersTable.Arn
            - !Sub "${UsersTable.Arn}/index/*"
  
  # Environment variables
  environment:
    TABLE_NAME: !Ref UsersTable
    STAGE: ${self:provider.stage}
    LOG_LEVEL: ${self:custom.logLevel.${self:provider.stage}}
  
  # X-Ray tracing
  tracing:
    apiGateway: true
    lambda: true
  
  # Logs
  logs:
    restApi: true

# Custom variables
custom:
  logLevel:
    dev: DEBUG
    staging: INFO
    prod: WARN
  
  # Pruning old versions
  prune:
    automatic: true
    number: 3
  
  # Bundle optimization
  esbuild:
    bundle: true
    minify: true
    sourcemap: true
    target: node18
    exclude:
      - '@aws-sdk/*'
    watch:
      pattern: ['src/**/*.ts']
  
  # Offline plugin settings
  serverless-offline:
    httpPort: 3000
    lambdaPort: 3002

functions:
  createUser:
    handler: src/handlers/createUser.handler
    description: Create new user
    timeout: 30
    memorySize: 256
    events:
      - http:
          path: /users
          method: POST
          cors: true
          request:
            schemas:
              application/json:
                schema: ${file(schemas/createUser.json)}
                name: CreateUserModel
    
  getUser:
    handler: src/handlers/getUser.handler
    timeout: 15
    memorySize: 128
    events:
      - http:
          path: /users/{userId}
          method: GET
          cors: true
          authorizer:
            name: jwtAuthorizer
            resultTtlInSeconds: 300

  # Scheduled function
  cleanupExpiredSessions:
    handler: src/handlers/cleanup.handler
    timeout: 300
    memorySize: 512
    events:
      - schedule:
          rate: rate(1 hour)
          enabled: ${self:custom.scheduleEnabled.${self:provider.stage}}

resources:
  Resources:
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: users-${self:provider.stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: userId
            AttributeType: S
        KeySchema:
          - AttributeName: userId
            KeyType: HASH
        TimeToLiveSpecification:
          AttributeName: ttl
          Enabled: true

plugins:
  - serverless-esbuild
  - serverless-offline
  - serverless-prune-plugin
  - serverless-plugin-canary-deployments
```

### 3.3 Canary Deployment สำหรับ Lambda

```yaml
# เพิ่มใน serverless.yml functions section
functions:
  createUser:
    handler: src/handlers/createUser.handler
    deploymentSettings:
      type: Linear10PercentEvery1Minute  # หรือ Canary10Percent5Minutes
      alias: Live
      preTrafficHook: preDeploymentCheck
      postTrafficHook: postDeploymentCheck
      alarms:
        - createUserErrorAlarm
        - createUserThrottleAlarm

resources:
  Resources:
    # CloudWatch Alarm สำหรับ monitor การ deploy
    createUserErrorAlarm:
      Type: AWS::CloudWatch::Alarm
      Properties:
        AlarmName: CreateUserErrors-${self:provider.stage}
        MetricName: Errors
        Namespace: AWS/Lambda
        Statistic: Sum
        Period: 60
        EvaluationPeriods: 1
        Threshold: 5
        ComparisonOperator: GreaterThanThreshold
        Dimensions:
          - Name: FunctionName
            Value: !Ref CreateUserLambdaFunction

    # Pre/Post deployment hooks
    PreDeploymentCheckFunction:
      Type: AWS::Lambda::Function
      Properties:
        FunctionName: preDeploymentCheck-${self:provider.stage}
        Handler: index.handler
        Runtime: nodejs18.x
        Code:
          ZipFile: |
            const AWS = require('aws-sdk');
            const codedeploy = new AWS.CodeDeploy();
            
            exports.handler = async (event) => {
              console.log('Running pre-deployment checks...');
              
              // Add your health checks here
              const healthy = await checkDependencies();
              
              if (!healthy) {
                await codedeploy.putLifecycleEventHookExecutionStatus({
                  deploymentId: event.DeploymentId,
                  lifecycleEventHookExecutionId: event.LifecycleEventHookExecutionId,
                  status: 'Failed'
                }).promise();
                return;
              }
              
              await codedeploy.putLifecycleEventHookExecutionStatus({
                deploymentId: event.DeploymentId,
                lifecycleEventHookExecutionId: event.LifecycleEventHookExecutionId,
                status: 'Succeeded'
              }).promise();
            };
```

---

## 4. Google Cloud Functions CI/CD

### 4.1 Cloud Functions Structure

```
gcp-functions/
├── package.json
├── index.js
├── functions/
│   ├── users/
│   │   ├── create.js
│   │   ├── get.js
│   │   └── delete.js
│   └── events/
│       └── process.js
└── .github/
    └── workflows/
        └── deploy-gcp.yml
```

### 4.2 Google Cloud Function Code

```javascript
// functions/users/create.js
const { Firestore } = require('@google-cloud/firestore');
const { PubSub } = require('@google-cloud/pubsub');

const firestore = new Firestore();
const pubsub = new PubSub();

exports.createUser = async (req, res) => {
  // CORS handling
  res.set('Access-Control-Allow-Origin', '*');
  
  if (req.method === 'OPTIONS') {
    res.set('Access-Control-Allow-Methods', 'POST');
    res.set('Access-Control-Allow-Headers', 'Content-Type,Authorization');
    res.status(204).send('');
    return;
  }
  
  if (req.method !== 'POST') {
    res.status(405).json({ error: 'Method not allowed' });
    return;
  }

  try {
    const { email, name } = req.body;
    
    if (!email || !name) {
      res.status(400).json({ error: 'email and name are required' });
      return;
    }

    // Check if user exists
    const existing = await firestore
      .collection('users')
      .where('email', '==', email)
      .limit(1)
      .get();
    
    if (!existing.empty) {
      res.status(409).json({ error: 'Email already exists' });
      return;
    }

    const userId = firestore.collection('users').doc().id;
    const now = new Date();
    
    const userData = {
      userId,
      email,
      name,
      createdAt: now,
      updatedAt: now
    };

    await firestore.collection('users').doc(userId).set(userData);

    // Publish event to Pub/Sub
    const topic = pubsub.topic('user-events');
    await topic.publishMessage({
      data: Buffer.from(JSON.stringify({
        event: 'USER_CREATED',
        data: userData
      })),
      attributes: {
        eventType: 'USER_CREATED'
      }
    });

    res.status(201).json(userData);
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
};
```

### 4.3 GitHub Actions สำหรับ GCP Functions

```yaml
# .github/workflows/deploy-gcp.yml
name: Deploy to Google Cloud Functions

on:
  push:
    branches: [main]
    paths: ['functions/**', 'package.json']

env:
  PROJECT_ID: my-project-id
  REGION: asia-southeast1

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # สำหรับ Workload Identity Federation
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}
      
      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v2
      
      - name: Run tests
        run: |
          npm ci
          npm test
      
      - name: Deploy createUser function
        run: |
          gcloud functions deploy createUser \
            --gen2 \
            --runtime=nodejs18 \
            --region=${{ env.REGION }} \
            --source=. \
            --entry-point=createUser \
            --trigger-http \
            --allow-unauthenticated \
            --min-instances=0 \
            --max-instances=100 \
            --memory=256MB \
            --timeout=30s \
            --set-env-vars="PROJECT_ID=${{ env.PROJECT_ID }}" \
            --service-account=functions-sa@${{ env.PROJECT_ID }}.iam.gserviceaccount.com
      
      - name: Deploy getUser function
        run: |
          gcloud functions deploy getUser \
            --gen2 \
            --runtime=nodejs18 \
            --region=${{ env.REGION }} \
            --source=. \
            --entry-point=getUser \
            --trigger-http \
            --allow-unauthenticated \
            --memory=128MB \
            --timeout=10s
```

---

## 5. Azure Functions CI/CD

### 5.1 Azure Functions Project Structure

```
azure-functions/
├── host.json
├── local.settings.json
├── package.json
├── CreateUser/
│   ├── function.json
│   └── index.js
├── GetUser/
│   ├── function.json
│   └── index.js
└── .github/
    └── workflows/
        └── deploy-azure.yml
```

### 5.2 Azure Function Configuration

```json
// host.json
{
  "version": "2.0",
  "logging": {
    "applicationInsights": {
      "samplingSettings": {
        "isEnabled": true,
        "excludedTypes": "Request"
      }
    }
  },
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[3.*, 4.0.0)"
  },
  "functionTimeout": "00:05:00",
  "retry": {
    "strategy": "exponentialBackoff",
    "maxRetryCount": 3,
    "minimumInterval": "00:00:02",
    "maximumInterval": "00:00:10"
  }
}
```

```json
// CreateUser/function.json
{
  "bindings": [
    {
      "authLevel": "function",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": ["post"],
      "route": "users"
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    },
    {
      "name": "cosmosOutput",
      "type": "cosmosDB",
      "direction": "out",
      "databaseName": "UserDB",
      "collectionName": "Users",
      "createIfNotExists": true,
      "connectionStringSetting": "CosmosDBConnectionString"
    }
  ]
}
```

### 5.3 GitHub Actions สำหรับ Azure Functions

```yaml
# .github/workflows/deploy-azure.yml
name: Deploy Azure Functions

on:
  push:
    branches: [main]

env:
  AZURE_FUNCTIONAPP_NAME: my-user-api
  AZURE_FUNCTIONAPP_PACKAGE_PATH: '.'
  NODE_VERSION: '18.x'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Install dependencies and run tests
        run: |
          npm ci
          npm run build --if-present
          npm test
      
      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Deploy to Azure Functions
        uses: Azure/functions-action@v1
        with:
          app-name: ${{ env.AZURE_FUNCTIONAPP_NAME }}
          package: ${{ env.AZURE_FUNCTIONAPP_PACKAGE_PATH }}
          publish-profile: ${{ secrets.AZURE_FUNCTIONAPP_PUBLISH_PROFILE }}
          respect-funcignore: true
      
      - name: Run smoke tests
        run: |
          FUNCTION_URL="https://${{ env.AZURE_FUNCTIONAPP_NAME }}.azurewebsites.net"
          curl -f "${FUNCTION_URL}/api/health" || exit 1
```

---

## 6. การทดสอบ Serverless Applications

### 6.1 Unit Testing

```javascript
// tests/unit/createUser.test.js
const { mockClient } = require('aws-sdk-client-mock');
const { DynamoDBClient, PutItemCommand } = require('@aws-sdk/client-dynamodb');
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');
const { handler } = require('../../src/handlers/createUser/index');

// Mock AWS clients
const dynamoMock = mockClient(DynamoDBClient);
const snsMock = mockClient(SNSClient);

describe('createUser Lambda handler', () => {
  beforeEach(() => {
    dynamoMock.reset();
    snsMock.reset();
    process.env.TABLE_NAME = 'test-table';
    process.env.TOPIC_ARN = 'arn:aws:sns:us-east-1:123:test';
  });

  it('should create user successfully', async () => {
    // Arrange
    dynamoMock.on(PutItemCommand).resolves({});
    snsMock.on(PublishCommand).resolves({ MessageId: 'test-id' });
    
    const event = {
      body: JSON.stringify({
        email: 'test@example.com',
        name: 'Test User'
      })
    };

    // Act
    const result = await handler(event);

    // Assert
    expect(result.statusCode).toBe(201);
    const body = JSON.parse(result.body);
    expect(body.email).toBe('test@example.com');
    expect(body.name).toBe('Test User');
    expect(body.userId).toBeDefined();
  });

  it('should return 400 when email is missing', async () => {
    const event = {
      body: JSON.stringify({ name: 'Test User' })
    };
    
    const result = await handler(event);
    
    expect(result.statusCode).toBe(400);
    expect(JSON.parse(result.body).error).toContain('email');
  });

  it('should return 409 when email already exists', async () => {
    const { ConditionalCheckFailedException } = require('@aws-sdk/client-dynamodb');
    dynamoMock.on(PutItemCommand).rejects(
      new ConditionalCheckFailedException({ message: 'Conditional check failed', $metadata: {} })
    );
    
    const event = {
      body: JSON.stringify({
        email: 'existing@example.com',
        name: 'Test User'
      })
    };
    
    const result = await handler(event);
    
    expect(result.statusCode).toBe(409);
  });

  it('should return 500 on unexpected error', async () => {
    dynamoMock.on(PutItemCommand).rejects(new Error('Connection timeout'));
    
    const event = {
      body: JSON.stringify({
        email: 'test@example.com',
        name: 'Test User'
      })
    };
    
    const result = await handler(event);
    
    expect(result.statusCode).toBe(500);
  });
});
```

### 6.2 Integration Testing ด้วย SAM Local

```bash
# รัน SAM API locally
sam local start-api --env-vars env.json --port 3000

# รัน function แบบ one-off
sam local invoke CreateUserFunction \
  --event events/create-user.json \
  --env-vars env.json
```

```json
// env.json สำหรับ local testing
{
  "CreateUserFunction": {
    "TABLE_NAME": "local-users-table",
    "TOPIC_ARN": "arn:aws:sns:ap-southeast-1:123456789:local-topic",
    "LOG_LEVEL": "DEBUG"
  }
}
```

```javascript
// tests/integration/api.test.js
const axios = require('axios');

const API_URL = process.env.API_URL || 'http://localhost:3000';

describe('User API Integration Tests', () => {
  let createdUserId;

  it('should create a user', async () => {
    const response = await axios.post(`${API_URL}/users`, {
      email: `test-${Date.now()}@example.com`,
      name: 'Integration Test User'
    });
    
    expect(response.status).toBe(201);
    expect(response.data.userId).toBeDefined();
    createdUserId = response.data.userId;
  });

  it('should get the created user', async () => {
    const response = await axios.get(`${API_URL}/users/${createdUserId}`);
    
    expect(response.status).toBe(200);
    expect(response.data.name).toBe('Integration Test User');
  });

  it('should handle duplicate email', async () => {
    const email = `duplicate-${Date.now()}@example.com`;
    
    await axios.post(`${API_URL}/users`, { email, name: 'First User' });
    
    try {
      await axios.post(`${API_URL}/users`, { email, name: 'Second User' });
      fail('Should have thrown error');
    } catch (error) {
      expect(error.response.status).toBe(409);
    }
  });
});
```

### 6.3 Load Testing สำหรับ Cold Start

```javascript
// tests/performance/cold-start.test.js
const AWS = require('aws-sdk');

const lambda = new AWS.Lambda({ region: 'ap-southeast-1' });
const functionName = process.env.FUNCTION_NAME;

async function measureColdStart() {
  // Force cold start by updating environment variable
  await lambda.updateFunctionConfiguration({
    FunctionName: functionName,
    Environment: {
      Variables: {
        COLD_START_FORCE: Date.now().toString()
      }
    }
  }).promise();
  
  // Wait for update
  await new Promise(r => setTimeout(r, 3000));
  
  const start = Date.now();
  
  const result = await lambda.invoke({
    FunctionName: functionName,
    Payload: JSON.stringify({
      httpMethod: 'GET',
      path: '/health'
    })
  }).promise();
  
  const duration = Date.now() - start;
  
  return {
    duration,
    statusCode: JSON.parse(result.Payload).statusCode
  };
}

describe('Cold Start Performance', () => {
  it('cold start should be under 3 seconds', async () => {
    const result = await measureColdStart();
    
    console.log(`Cold start duration: ${result.duration}ms`);
    expect(result.duration).toBeLessThan(3000);
    expect(result.statusCode).toBe(200);
  }, 30000);
});
```

---

## 7. Cold Start Optimization

### 7.1 Provisioned Concurrency

```yaml
# template.yaml เพิ่ม Provisioned Concurrency
  CreateUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      AutoPublishAlias: live
      ProvisionedConcurrencyConfig:
        ProvisionedConcurrentExecutions: 5
```

### 7.2 Scheduled Warming

```javascript
// src/handlers/warmer/index.js
const AWS = require('aws-sdk');
const lambda = new AWS.Lambda();

const FUNCTIONS_TO_WARM = [
  process.env.CREATE_USER_FUNCTION_NAME,
  process.env.GET_USER_FUNCTION_NAME,
];

exports.handler = async () => {
  const warmingPromises = FUNCTIONS_TO_WARM.map(functionName =>
    lambda.invoke({
      FunctionName: functionName,
      InvocationType: 'Event',
      Payload: JSON.stringify({ source: 'serverless-plugin-warmup' })
    }).promise()
  );
  
  await Promise.all(warmingPromises);
  console.log('Warmed functions:', FUNCTIONS_TO_WARM);
};
```

```yaml
# SAM Template สำหรับ warmer
  WarmerFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/handlers/warmer/index.handler
      Events:
        WarmSchedule:
          Type: Schedule
          Properties:
            Schedule: rate(5 minutes)
            Enabled: true
      Environment:
        Variables:
          CREATE_USER_FUNCTION_NAME: !Ref CreateUserFunction
          GET_USER_FUNCTION_NAME: !Ref GetUserFunction
      Policies:
        - LambdaInvokePolicy:
            FunctionName: !Ref CreateUserFunction
        - LambdaInvokePolicy:
            FunctionName: !Ref GetUserFunction
```

### 7.3 การ Optimize Lambda Package Size

```javascript
// webpack.config.js
const path = require('path');
const TerserPlugin = require('terser-webpack-plugin');

module.exports = {
  entry: './src/handlers/createUser/index.js',
  target: 'node',
  mode: 'production',
  output: {
    path: path.resolve(__dirname, 'dist/createUser'),
    filename: 'index.js',
    libraryTarget: 'commonjs2'
  },
  externals: {
    '@aws-sdk/client-dynamodb': '@aws-sdk/client-dynamodb',
    '@aws-sdk/client-sns': '@aws-sdk/client-sns'
  },
  optimization: {
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            dead_code: true,
            drop_console: process.env.NODE_ENV === 'production'
          }
        }
      })
    ]
  }
};
```

---

## 8. Deployment Strategies สำหรับ Serverless

### 8.1 Blue/Green Deployment

```python
# deploy_blue_green.py
import boto3
import time

client = boto3.client('lambda')
cloudwatch = boto3.client('cloudwatch')

def deploy_with_blue_green(function_name, new_code_s3_bucket, new_code_s3_key):
    """
    Implement blue/green deployment using Lambda aliases
    """
    # Step 1: Deploy new code as a new version
    print("Deploying new Lambda version...")
    
    response = client.update_function_code(
        FunctionName=function_name,
        S3Bucket=new_code_s3_bucket,
        S3Key=new_code_s3_key,
        Publish=True
    )
    
    new_version = response['Version']
    print(f"New version: {new_version}")
    
    # Step 2: Update 'green' alias to point to new version
    try:
        client.get_alias(FunctionName=function_name, Name='green')
        client.update_alias(
            FunctionName=function_name,
            Name='green',
            FunctionVersion=new_version
        )
    except client.exceptions.ResourceNotFoundException:
        client.create_alias(
            FunctionName=function_name,
            Name='green',
            FunctionVersion=new_version
        )
    
    print(f"Green alias pointing to version {new_version}")
    
    # Step 3: Run smoke tests on green
    print("Running smoke tests on green...")
    if not run_smoke_tests(function_name, 'green'):
        print("Smoke tests failed! Rolling back...")
        rollback_green(function_name)
        return False
    
    # Step 4: Gradually shift traffic
    print("Starting traffic shift...")
    
    for percentage in [10, 25, 50, 75, 100]:
        shift_traffic(function_name, new_version, percentage)
        
        # Monitor for 2 minutes at each step
        print(f"Monitoring at {percentage}% traffic for 2 minutes...")
        if not monitor_health(function_name, duration=120):
            print(f"Issues detected at {percentage}%! Rolling back...")
            rollback_traffic(function_name)
            return False
        
        print(f"Health check passed at {percentage}%")
    
    print("Deployment successful!")
    return True

def shift_traffic(function_name, new_version, percentage):
    """Shift specified percentage of traffic to new version"""
    current_alias = client.get_alias(
        FunctionName=function_name,
        Name='live'
    )
    
    old_version = current_alias['FunctionVersion']
    
    if percentage == 100:
        client.update_alias(
            FunctionName=function_name,
            Name='live',
            FunctionVersion=new_version
        )
    else:
        client.update_alias(
            FunctionName=function_name,
            Name='live',
            FunctionVersion=old_version,
            RoutingConfig={
                'AdditionalVersionWeights': {
                    new_version: percentage / 100
                }
            }
        )

def monitor_health(function_name, duration=120):
    """Monitor Lambda error rate"""
    start_time = time.time()
    check_interval = 15
    
    while time.time() - start_time < duration:
        metrics = cloudwatch.get_metric_statistics(
            Namespace='AWS/Lambda',
            MetricName='Errors',
            Dimensions=[{'Name': 'FunctionName', 'Value': function_name}],
            StartTime=time.strftime('%Y-%m-%dT%H:%M:%S', 
                                    time.gmtime(time.time() - check_interval)),
            EndTime=time.strftime('%Y-%m-%dT%H:%M:%S', time.gmtime()),
            Period=check_interval,
            Statistics=['Sum']
        )
        
        total_invocations = get_invocations(function_name, check_interval)
        total_errors = sum(p['Sum'] for p in metrics['Datapoints'])
        
        if total_invocations > 0:
            error_rate = total_errors / total_invocations
            print(f"Error rate: {error_rate:.2%}")
            
            if error_rate > 0.05:  # 5% threshold
                return False
        
        time.sleep(check_interval)
    
    return True
```

---

## 9. Monitoring และ Observability

### 9.1 Structured Logging

```javascript
// src/shared/logger.js
const LOG_LEVEL = process.env.LOG_LEVEL || 'INFO';
const LOG_LEVELS = { DEBUG: 0, INFO: 1, WARN: 2, ERROR: 3 };
const CURRENT_LEVEL = LOG_LEVELS[LOG_LEVEL] || 1;

function log(level, message, data = {}) {
  if (LOG_LEVELS[level] >= CURRENT_LEVEL) {
    console[level.toLowerCase()]({
      timestamp: new Date().toISOString(),
      level,
      message,
      requestId: process.env._X_AMZN_TRACE_ID,
      functionName: process.env.AWS_LAMBDA_FUNCTION_NAME,
      functionVersion: process.env.AWS_LAMBDA_FUNCTION_VERSION,
      ...data
    });
  }
}

module.exports = {
  debug: (msg, data) => log('DEBUG', msg, data),
  info: (msg, data) => log('INFO', msg, data),
  warn: (msg, data) => log('WARN', msg, data),
  error: (msg, data) => log('ERROR', msg, data)
};
```

### 9.2 CloudWatch Dashboard ผ่าน CloudFormation

```yaml
# cloudwatch-dashboard.yaml
Resources:
  LambdaDashboard:
    Type: AWS::CloudWatch::Dashboard
    Properties:
      DashboardName: ServerlessAPIMetrics
      DashboardBody: !Sub |
        {
          "widgets": [
            {
              "type": "metric",
              "properties": {
                "title": "Lambda Invocations",
                "metrics": [
                  ["AWS/Lambda", "Invocations", "FunctionName", "${CreateUserFunction}"],
                  ["AWS/Lambda", "Invocations", "FunctionName", "${GetUserFunction}"]
                ],
                "period": 300,
                "stat": "Sum"
              }
            },
            {
              "type": "metric",
              "properties": {
                "title": "Lambda Errors",
                "metrics": [
                  ["AWS/Lambda", "Errors", "FunctionName", "${CreateUserFunction}"],
                  ["AWS/Lambda", "Errors", "FunctionName", "${GetUserFunction}"]
                ],
                "period": 300,
                "stat": "Sum"
              }
            },
            {
              "type": "metric",
              "properties": {
                "title": "Lambda Duration (p99)",
                "metrics": [
                  ["AWS/Lambda", "Duration", "FunctionName", "${CreateUserFunction}", {"stat": "p99"}],
                  ["AWS/Lambda", "Duration", "FunctionName", "${GetUserFunction}", {"stat": "p99"}]
                ]
              }
            },
            {
              "type": "metric",
              "properties": {
                "title": "Cold Starts",
                "metrics": [
                  ["AWS/Lambda", "InitDuration", "FunctionName", "${CreateUserFunction}"]
                ]
              }
            }
          ]
        }
```

---

## 10. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: สร้าง Serverless TODO API

สร้าง serverless TODO list API ด้วย AWS SAM ที่มีฟีเจอร์ดังนี้:

```
Requirements:
1. POST /todos - สร้าง todo ใหม่
2. GET /todos - ดึง todos ทั้งหมด (with pagination)
3. GET /todos/{id} - ดึง todo เดี่ยว
4. PUT /todos/{id} - อัพเดท todo
5. DELETE /todos/{id} - ลบ todo

Technical Requirements:
- DynamoDB สำหรับ data storage
- API Gateway HTTP API (v2)
- Lambda authorizer ด้วย JWT
- CloudWatch Logs structured
- X-Ray tracing เปิดใช้งาน
- Unit tests ครอบคลุม > 80%
- SAM pipeline กับ GitHub Actions
```

### แบบฝึกหัดที่ 2: Implement Blue/Green Deployment

```
Tasks:
1. สร้าง Lambda function ที่มี 2 versions (v1, v2)
2. ตั้งค่า aliases: live, canary
3. สร้าง script สำหรับ gradually shift traffic
4. ตั้ง CloudWatch alarm สำหรับ auto-rollback
5. ทดสอบด้วยการ inject error ใน v2

Metrics ที่ต้อง monitor:
- Error rate > 5% → rollback
- P99 latency > 2000ms → rollback
- Throttle count > 10 → pause deployment
```

### แบบฝึกหัดที่ 3: Performance Testing Pipeline

```javascript
// performance-test.yml (k6 script)
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '1m', target: 10 },   // Ramp up
    { duration: '5m', target: 100 },  // Stay at peak
    { duration: '2m', target: 0 }     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],  // 95% under 2s
    errors: ['rate<0.05'],              // Error rate under 5%
  }
};

export default function() {
  const BASE_URL = __ENV.API_URL;
  
  // Create user
  const createRes = http.post(`${BASE_URL}/users`, JSON.stringify({
    email: `user-${Math.random()}@test.com`,
    name: 'Load Test User'
  }), {
    headers: { 'Content-Type': 'application/json' }
  });
  
  const createSuccess = check(createRes, {
    'create status is 201': (r) => r.status === 201
  });
  
  errorRate.add(!createSuccess);
  
  if (createSuccess) {
    const userId = createRes.json('userId');
    
    // Get user
    const getRes = http.get(`${BASE_URL}/users/${userId}`);
    
    check(getRes, {
      'get status is 200': (r) => r.status === 200,
      'response time < 500ms': (r) => r.timings.duration < 500
    });
  }
  
  sleep(1);
}
```

### สรุปบทที่ 61

ในบทนี้เราได้เรียนรู้:
- **AWS SAM**: สร้างและ deploy serverless applications บน AWS
- **Serverless Framework**: Multi-cloud serverless deployment พร้อม plugins
- **GCP/Azure Functions**: CI/CD สำหรับ providers อื่น
- **Testing Strategies**: Unit, integration, และ performance testing
- **Cold Start Optimization**: Provisioned concurrency และ warming strategies
- **Deployment Strategies**: Blue/green, canary deployment สำหรับ Lambda
- **Monitoring**: Structured logging, CloudWatch dashboards

บทถัดไปเราจะเรียนรู้ MLOps Pipeline ซึ่งนำ CI/CD concepts มาใช้กับ Machine Learning workflows

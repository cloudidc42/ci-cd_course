# Part 16: Jenkins

## บทนำ

Jenkins เป็น open-source automation server ที่มีมาตั้งแต่ปี 2004 (เดิมชื่อ Hudson) และยังคงเป็น CI/CD tool ที่ได้รับความนิยมสูงสุดในองค์กรขนาดใหญ่ ด้วย ecosystem ของ plugins กว่า 1,800 รายการ Jenkins สามารถ integrate กับเกือบทุก tool ในโลก DevOps

---

## 16.1 ติดตั้ง Jenkins ด้วย Docker

### Jenkins บน Docker (Development)

```bash
# รัน Jenkins ด้วย Docker อย่างง่าย
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk17

# ดู initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Jenkins ด้วย Docker Compose (Production-ready)

```yaml
# docker-compose.yml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    restart: unless-stopped
    privileged: true
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/local/bin/docker:/usr/local/bin/docker
    environment:
      - JENKINS_OPTS=--httpPort=8080
      - JAVA_OPTS=-Xmx2g -Xms512m -XX:MaxPermSize=512m
    networks:
      - jenkins-net

  jenkins-agent:
    image: jenkins/inbound-agent:latest
    container_name: jenkins-agent
    restart: unless-stopped
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_SECRET=${AGENT_SECRET}
      - JENKINS_AGENT_NAME=docker-agent
      - JENKINS_AGENT_WORKDIR=/home/jenkins/agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - agent_workdir:/home/jenkins/agent
    networks:
      - jenkins-net
    depends_on:
      - jenkins

networks:
  jenkins-net:
    driver: bridge

volumes:
  jenkins_home:
  agent_workdir:
```

### Jenkins Configuration as Code (JCasC)

```yaml
# jenkins.yaml - Configuration as Code
jenkins:
  systemMessage: "Jenkins configured automatically by JCasC"
  numExecutors: 0
  mode: EXCLUSIVE
  
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: admin
          name: "Admin User"
          password: "${JENKINS_ADMIN_PASSWORD}"
  
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            assignments:
              - "admin"
          - name: "developer"
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Read"
            assignments:
              - "developers"

  clouds:
    - docker:
        name: "docker"
        dockerApi:
          dockerHost:
            uri: "unix:///var/run/docker.sock"
        templates:
          - labelString: "docker-agent"
            dockerTemplateBase:
              image: "jenkins/inbound-agent:latest"
            remoteFs: "/home/jenkins/agent"
            connector:
              attach:
                user: "jenkins"

unclassified:
  location:
    url: "http://jenkins.mycompany.com:8080"
    adminAddress: "admin@mycompany.com"
  
  globalLibraries:
    libraries:
      - name: "shared-library"
        retriever:
          modernSCM:
            scm:
              git:
                remote: "https://github.com/myorg/jenkins-shared-library.git"
                credentialsId: "github-credentials"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: github-credentials
              username: "${GITHUB_USERNAME}"
              password: "${GITHUB_TOKEN}"
          - string:
              scope: GLOBAL
              id: slack-webhook
              secret: "${SLACK_WEBHOOK_URL}"
```

### Dockerfile สำหรับ Jenkins พร้อม Plugins

```dockerfile
FROM jenkins/jenkins:lts-jdk17

# ติดตั้ง plugins
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

# Copy JCasC configuration
COPY jenkins.yaml /usr/share/jenkins/ref/jenkins.yaml
ENV CASC_JENKINS_CONFIG=/usr/share/jenkins/ref/jenkins.yaml

USER root

# ติดตั้ง tools เพิ่มเติม
RUN apt-get update && apt-get install -y \
    docker.io \
    kubectl \
    helm \
    awscli \
    && rm -rf /var/lib/apt/lists/*

USER jenkins

# Skip setup wizard
ENV JAVA_OPTS=-Djenkins.install.runSetupWizard=false
```

```
# plugins.txt
git:latest
github:latest
pipeline-stage-view:latest
blueocean:latest
docker-workflow:latest
kubernetes:latest
credentials-binding:latest
ws-cleanup:latest
build-timeout:latest
timestamper:latest
ansicolor:latest
slack:latest
role-strategy:latest
configuration-as-code:latest
job-dsl:latest
pipeline-utility-steps:latest
http_request:latest
```

---

## 16.2 Jenkins UI Overview

### การ Navigate ใน Jenkins

```
Jenkins Dashboard
├── All Jobs - รายการ jobs ทั้งหมด
├── New Item - สร้าง job ใหม่
├── People - ผู้ใช้ทั้งหมด
├── Build History - ประวัติ builds
├── Manage Jenkins
│   ├── System Configuration
│   │   ├── System - Global settings
│   │   ├── Tools - JDK, Maven, Gradle, etc.
│   │   └── Plugins - Plugin manager
│   ├── Security
│   │   ├── Users - จัดการผู้ใช้
│   │   ├── Credentials - เก็บ credentials
│   │   └── Manage Old Data
│   ├── Status Information
│   │   ├── System Information
│   │   └── System Log
│   └── Nodes - Jenkins agents
└── Blue Ocean - Modern UI
```

### Job Types

1. **Freestyle Project** - การตั้งค่าผ่าน UI
2. **Pipeline** - ใช้ Jenkinsfile
3. **Multi-configuration Project** - Matrix builds
4. **Multibranch Pipeline** - Pipeline สำหรับหลาย branches
5. **Organization Folder** - Pipeline สำหรับทั้ง organization
6. **Folder** - จัดกลุ่ม jobs

---

## 16.3 Jenkinsfile - Declarative Pipeline

### โครงสร้าง Declarative Pipeline

```groovy
// Jenkinsfile

pipeline {
    // กำหนด agent ที่จะรัน pipeline
    agent any
    
    // กำหนด options ของ pipeline
    options {
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timestamps()
        ansiColor('xterm')
    }
    
    // กำหนด tools
    tools {
        nodejs 'NodeJS-20'
        jdk 'JDK17'
    }
    
    // กำหนด environment variables
    environment {
        APP_NAME = 'myapp'
        DOCKER_REGISTRY = 'registry.mycompany.com'
        SLACK_CHANNEL = '#deployments'
        
        // ดึง credentials
        DOCKER_CREDENTIALS = credentials('docker-registry')
        SLACK_WEBHOOK = credentials('slack-webhook')
    }
    
    // กำหนด parameters (สำหรับ manual trigger)
    parameters {
        string(name: 'BRANCH', defaultValue: 'main', description: 'Branch to build')
        choice(name: 'ENVIRONMENT', choices: ['staging', 'production'], description: 'Deploy environment')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run test suite')
        password(name: 'DEPLOY_KEY', defaultValue: '', description: 'Deployment key')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log --oneline -5'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Test') {
            when {
                expression { params.RUN_TESTS == true }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                    post {
                        always {
                            junit 'test-results/unit/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                    post {
                        always {
                            junit 'test-results/integration/*.xml'
                        }
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
                sh "docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ."
            }
        }
        
        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-registry',
                    usernameVariable: 'REGISTRY_USER',
                    passwordVariable: 'REGISTRY_PASS'
                )]) {
                    sh """
                        docker login -u ${REGISTRY_USER} -p ${REGISTRY_PASS} ${DOCKER_REGISTRY}
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}
                        docker tag ${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} ${DOCKER_REGISTRY}/${APP_NAME}:latest
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:latest
                    """
                }
            }
        }
        
        stage('Deploy Staging') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} \
                        -n staging
                    kubectl rollout status deployment/${APP_NAME} -n staging
                """
            }
        }
        
        stage('Deploy Production') {
            when {
                allOf {
                    branch 'main'
                    expression { params.ENVIRONMENT == 'production' }
                }
            }
            input {
                message "Deploy to production?"
                ok "Yes, deploy it!"
                submitter "admin,devops-lead"
                parameters {
                    string(name: 'REASON', description: 'Reason for deployment')
                }
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER} \
                        -n production
                    kubectl rollout status deployment/${APP_NAME} -n production
                """
            }
        }
    }
    
    post {
        always {
            // ทำเสมอ ไม่ว่าจะ success หรือ fail
            cleanWs()
        }
        success {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: "✅ Build ${BUILD_NUMBER} สำเร็จ: ${JOB_NAME}\n${BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: "❌ Build ${BUILD_NUMBER} ล้มเหลว: ${JOB_NAME}\n${BUILD_URL}"
            )
            emailext(
                to: 'team@mycompany.com',
                subject: "Jenkins Build Failed: ${JOB_NAME} #${BUILD_NUMBER}",
                body: "Build ล้มเหลว\nJob: ${JOB_NAME}\nBuild: ${BUILD_NUMBER}\nURL: ${BUILD_URL}"
            )
        }
        unstable {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'warning',
                message: "⚠️ Build ${BUILD_NUMBER} ไม่เสถียร: ${JOB_NAME}\n${BUILD_URL}"
            )
        }
    }
}
```

---

## 16.4 Declarative Pipeline ขั้นสูง

### Parallel Stages

```groovy
pipeline {
    agent any
    
    stages {
        stage('Parallel Testing') {
            parallel {
                stage('Test Chrome') {
                    agent {
                        docker {
                            image 'selenium/standalone-chrome:latest'
                            args '-v /dev/shm:/dev/shm'
                        }
                    }
                    steps {
                        sh 'npm run test:e2e:chrome'
                    }
                }
                
                stage('Test Firefox') {
                    agent {
                        docker {
                            image 'selenium/standalone-firefox:latest'
                            args '-v /dev/shm:/dev/shm'
                        }
                    }
                    steps {
                        sh 'npm run test:e2e:firefox'
                    }
                }
                
                stage('Test Safari') {
                    agent {
                        label 'macos'
                    }
                    steps {
                        sh 'npm run test:e2e:safari'
                    }
                }
            }
        }
        
        // Matrix stage (Jenkins 2.287+)
        stage('Matrix Build') {
            matrix {
                axes {
                    axis {
                        name 'PLATFORM'
                        values 'linux', 'windows', 'macos'
                    }
                    axis {
                        name 'NODE_VERSION'
                        values '18', '20', '21'
                    }
                }
                excludes {
                    exclude {
                        axis {
                            name 'PLATFORM'
                            values 'windows'
                        }
                        axis {
                            name 'NODE_VERSION'
                            values '18'
                        }
                    }
                }
                stages {
                    stage('Build') {
                        steps {
                            sh "echo Building on ${PLATFORM} with Node ${NODE_VERSION}"
                            sh 'npm test'
                        }
                    }
                }
            }
        }
    }
}
```

### Post Conditions

```groovy
pipeline {
    agent any
    
    stages {
        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                // รันหลัง stage นี้เสมอ
                always {
                    junit 'test-results/*.xml'
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'coverage/lcov-report',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
                // รันเฉพาะเมื่อ success
                success {
                    archiveArtifacts artifacts: 'dist/**/*', fingerprint: true
                }
                // รันเฉพาะเมื่อ failure
                failure {
                    sh 'npm run test:screenshot'
                    archiveArtifacts artifacts: 'screenshots/**/*'
                }
                // รันเฉพาะเมื่อ unstable (test failures)
                unstable {
                    emailext body: 'Tests are unstable', to: 'team@company.com'
                }
                // รันเมื่อเปลี่ยนจาก failure เป็น success
                fixed {
                    slackSend color: 'good', message: '✅ Build fixed!'
                }
                // รันเมื่อเปลี่ยนจาก success เป็น failure
                regression {
                    slackSend color: 'danger', message: '❌ Build regression!'
                }
            }
        }
    }
    
    post {
        cleanup {
            // ทำเสมอ แม้แต่หลัง always
            cleanWs()
            sh 'docker system prune -f'
        }
    }
}
```

---

## 16.5 Scripted Pipeline

```groovy
// Jenkinsfile - Scripted Pipeline
// ยืดหยุ่นกว่า Declarative แต่ซับซ้อนกว่า

node('linux') {
    def appName = 'myapp'
    def dockerRegistry = 'registry.mycompany.com'
    def buildTag = "${BUILD_NUMBER}-${GIT_COMMIT[0..6]}"
    
    try {
        stage('Checkout') {
            checkout scm
        }
        
        stage('Build') {
            sh 'npm ci'
            sh 'npm run build'
        }
        
        stage('Test') {
            // รัน tests แบบ parallel
            def tests = [:]
            
            tests['Unit Tests'] = {
                sh 'npm run test:unit'
            }
            
            tests['Integration Tests'] = {
                sh 'npm run test:integration'
            }
            
            tests['Security Scan'] = {
                sh 'npm audit --audit-level=high'
            }
            
            parallel tests
        }
        
        stage('Docker Build') {
            withDockerRegistry([credentialsId: 'docker-registry', url: "https://${dockerRegistry}"]) {
                def image = docker.build("${dockerRegistry}/${appName}:${buildTag}")
                image.push()
                image.push('latest')
            }
        }
        
        if (env.BRANCH_NAME == 'main') {
            stage('Deploy Staging') {
                deploy('staging', "${dockerRegistry}/${appName}:${buildTag}")
            }
            
            stage('Approval') {
                timeout(time: 24, unit: 'HOURS') {
                    input(
                        message: "Deploy ${buildTag} to production?",
                        ok: 'Deploy',
                        submitter: 'admin'
                    )
                }
            }
            
            stage('Deploy Production') {
                deploy('production', "${dockerRegistry}/${appName}:${buildTag}")
            }
        }
        
        currentBuild.result = 'SUCCESS'
        
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // ทำเสมอ
        cleanWs()
        notifySlack(currentBuild.result)
    }
}

// Helper functions
def deploy(String environment, String image) {
    sh """
        kubectl set image deployment/${appName} ${appName}=${image} -n ${environment}
        kubectl rollout status deployment/${appName} -n ${environment} --timeout=5m
    """
}

def notifySlack(String result) {
    def color = result == 'SUCCESS' ? 'good' : 'danger'
    def icon = result == 'SUCCESS' ? '✅' : '❌'
    slackSend(
        color: color,
        message: "${icon} Build ${BUILD_NUMBER}: ${result}\nJob: ${JOB_NAME}\n${BUILD_URL}"
    )
}
```

---

## 16.6 Agents และ Labels

### กำหนด Agents ประเภทต่างๆ

```groovy
pipeline {
    // agent any - ใช้ executor ใดก็ได้
    agent any
    
    stages {
        // Agent ระดับ stage
        stage('Linux Build') {
            agent {
                label 'linux'
            }
            steps {
                sh 'make build'
            }
        }
        
        stage('Windows Build') {
            agent {
                label 'windows'
            }
            steps {
                bat 'msbuild MyApp.sln'
            }
        }
        
        stage('Docker Stage') {
            agent {
                docker {
                    image 'maven:3.9-eclipse-temurin-17'
                    args '-v $HOME/.m2:/root/.m2'
                    // รัน container บน specific node
                    label 'docker-linux'
                }
            }
            steps {
                sh 'mvn package'
            }
        }
        
        stage('Kubernetes Pod') {
            agent {
                kubernetes {
                    yaml """
                        apiVersion: v1
                        kind: Pod
                        spec:
                          containers:
                          - name: maven
                            image: maven:3.9-eclipse-temurin-17
                            command:
                            - sleep
                            args:
                            - 99d
                          - name: docker
                            image: docker:24-dind
                            securityContext:
                              privileged: true
                    """
                    defaultContainer 'maven'
                }
            }
            steps {
                container('maven') {
                    sh 'mvn package'
                }
                container('docker') {
                    sh 'docker build -t myapp .'
                }
            }
        }
    }
}
```

---

## 16.7 Plugins ที่สำคัญ

### Blue Ocean

```groovy
// Blue Ocean ให้ UI ที่สวยงามสำหรับ Pipeline visualization
// ติดตั้งผ่าน Plugin Manager: blueocean

// Pipeline ที่ออกแบบมาดีจะดูสวยงามใน Blue Ocean
pipeline {
    agent any
    stages {
        stage('Build') { steps { sh 'make' } }
        stage('Test') {
            parallel {
                stage('Unit') { steps { sh 'make test-unit' } }
                stage('E2E') { steps { sh 'make test-e2e' } }
            }
        }
        stage('Deploy') { steps { sh 'make deploy' } }
    }
}
```

### Pipeline Plugin

```groovy
// ฟีเจอร์ Pipeline Plugin ที่ใช้บ่อย

// 1. archiveArtifacts
archiveArtifacts artifacts: 'dist/**/*', allowEmptyArchive: true

// 2. stash/unstash - ส่งไฟล์ระหว่าง stages
stash includes: 'dist/**/*', name: 'build-artifacts'
unstash 'build-artifacts'

// 3. withCredentials
withCredentials([
    usernamePassword(credentialsId: 'mydb', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS'),
    sshUserPrivateKey(credentialsId: 'server-key', keyFileVariable: 'SSH_KEY')
]) {
    sh 'ssh -i $SSH_KEY user@server deploy.sh'
}

// 4. timeout
timeout(time: 5, unit: 'MINUTES') {
    sh 'npm test'
}

// 5. retry
retry(3) {
    sh 'npm install'
}

// 6. waitUntil
waitUntil {
    def result = sh(script: 'curl -s http://myapp.com/health', returnStatus: true)
    return result == 0
}

// 7. httpRequest (ต้องติดตั้ง plugin)
def response = httpRequest(
    url: 'https://api.myapp.com/deploy',
    httpMode: 'POST',
    contentType: 'APPLICATION_JSON',
    requestBody: '{"environment": "staging"}',
    customHeaders: [[name: 'Authorization', value: "Bearer ${DEPLOY_TOKEN}"]]
)
echo "Status: ${response.status}"
```

### Git Plugin

```groovy
// การใช้งาน Git ขั้นสูง
stage('Checkout') {
    steps {
        checkout([
            $class: 'GitSCM',
            branches: [[name: '*/main']],
            extensions: [
                // Shallow clone สำหรับ large repos
                [$class: 'CloneOption', depth: 50, noTags: false, shallow: true],
                // Clean workspace ก่อน build
                [$class: 'CleanBeforeCheckout'],
                // Submodules
                [$class: 'SubmoduleOption', recursiveSubmodules: true]
            ],
            userRemoteConfigs: [[
                url: 'https://github.com/myorg/myrepo.git',
                credentialsId: 'github-credentials'
            ]]
        ])
        
        // ดึง Git info
        script {
            def gitInfo = sh(
                script: 'git log --oneline -1',
                returnStdout: true
            ).trim()
            env.GIT_COMMIT_MSG = gitInfo
        }
    }
}
```

---

## 16.8 Credentials Management

### ประเภทของ Credentials

```groovy
// 1. Username/Password
withCredentials([usernamePassword(
    credentialsId: 'docker-hub',
    usernameVariable: 'DOCKER_USER',
    passwordVariable: 'DOCKER_PASS'
)]) {
    sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
}

// 2. Secret Text
withCredentials([string(
    credentialsId: 'api-token',
    variable: 'API_TOKEN'
)]) {
    sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com'
}

// 3. SSH Private Key
withCredentials([sshUserPrivateKey(
    credentialsId: 'deploy-key',
    keyFileVariable: 'SSH_KEY_FILE',
    usernameVariable: 'SSH_USER'
)]) {
    sh '''
        ssh -i $SSH_KEY_FILE -o StrictHostKeyChecking=no $SSH_USER@server.com \
            "cd /app && ./deploy.sh"
    '''
}

// 4. Certificate
withCredentials([certificate(
    credentialsId: 'client-cert',
    keystoreVariable: 'KEYSTORE_FILE',
    passwordVariable: 'KEYSTORE_PASS'
)]) {
    sh 'curl --cert $KEYSTORE_FILE --pass $KEYSTORE_PASS https://secure.api.com'
}

// 5. File
withCredentials([file(
    credentialsId: 'kubeconfig',
    variable: 'KUBECONFIG_FILE'
)]) {
    sh 'kubectl --kubeconfig=$KUBECONFIG_FILE get pods'
}

// 6. ใช้หลาย credentials พร้อมกัน
withCredentials([
    usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS'),
    string(credentialsId: 'api-key', variable: 'API_KEY'),
    file(credentialsId: 'ssl-cert', variable: 'SSL_CERT')
]) {
    sh './deploy.sh'
}
```

### Credential Binding ใน Environment

```groovy
pipeline {
    agent any
    environment {
        // Bind credentials เป็น environment variables
        DOCKER_CREDS = credentials('docker-hub')
        // สร้าง DOCKER_CREDS_USR และ DOCKER_CREDS_PSW
        
        AWS_CREDS = credentials('aws-credentials')
        // สร้าง AWS_CREDS_USR (access key) และ AWS_CREDS_PSW (secret key)
    }
    
    stages {
        stage('Login') {
            steps {
                sh 'docker login -u $DOCKER_CREDS_USR -p $DOCKER_CREDS_PSW'
                sh 'aws configure set aws_access_key_id $AWS_CREDS_USR'
                sh 'aws configure set aws_secret_access_key $AWS_CREDS_PSW'
            }
        }
    }
}
```

---

## 16.9 Shared Libraries

### โครงสร้าง Shared Library

```
jenkins-shared-library/
├── vars/
│   ├── deployApp.groovy         # Global variable (เรียกใช้เหมือน function)
│   ├── buildDocker.groovy       # Docker build helper
│   └── notifySlack.groovy       # Slack notification
├── src/
│   └── com/
│       └── mycompany/
│           └── jenkins/
│               ├── Docker.groovy    # Docker class
│               └── Deploy.groovy   # Deployment class
└── resources/
    └── com/
        └── mycompany/
            └── scripts/
                └── deploy.sh
```

### `vars/deployApp.groovy`

```groovy
/**
 * Deploy application to environment
 * 
 * Usage:
 *   deployApp(
 *     app: 'myapp',
 *     environment: 'staging',
 *     image: 'registry/myapp:1.0',
 *     namespace: 'myapp-staging'
 *   )
 */
def call(Map config = [:]) {
    // ตรวจสอบ required parameters
    if (!config.app) error 'app parameter is required'
    if (!config.environment) error 'environment parameter is required'
    if (!config.image) error 'image parameter is required'
    
    def namespace = config.namespace ?: "${config.app}-${config.environment}"
    def timeout = config.timeout ?: '5m'
    
    echo "🚀 Deploying ${config.app} to ${config.environment}"
    
    withCredentials([file(credentialsId: "${config.environment}-kubeconfig", variable: 'KUBECONFIG')]) {
        sh """
            kubectl set image deployment/${config.app} \
                ${config.app}=${config.image} \
                --namespace=${namespace}
            
            kubectl rollout status deployment/${config.app} \
                --namespace=${namespace} \
                --timeout=${timeout}
        """
    }
    
    echo "✅ Successfully deployed ${config.app} to ${config.environment}"
    return [
        app: config.app,
        environment: config.environment,
        image: config.image,
        timestamp: new Date().toString()
    ]
}
```

### `vars/buildDocker.groovy`

```groovy
/**
 * Build and push Docker image
 * 
 * Usage:
 *   def result = buildDocker(
 *     image: 'myapp',
 *     registry: 'registry.mycompany.com',
 *     credentialsId: 'registry-creds',
 *     buildArgs: ['ENV=production', 'VERSION=1.0'],
 *     additionalTags: ['latest', 'stable']
 *   )
 *   echo result.tag  // full image:tag
 */
def call(Map config = [:]) {
    if (!config.image) error 'image parameter is required'
    
    def registry = config.registry ?: env.DOCKER_REGISTRY
    def credentialsId = config.credentialsId ?: 'docker-registry'
    def tag = config.tag ?: "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7)}"
    def fullImage = "${registry}/${config.image}:${tag}"
    
    // Build args
    def buildArgsStr = ''
    config.buildArgs?.each { arg ->
        buildArgsStr += " --build-arg ${arg}"
    }
    
    withDockerRegistry([credentialsId: credentialsId, url: "https://${registry}"]) {
        // Build image
        sh "docker build ${buildArgsStr} -t ${fullImage} ."
        
        // Push main tag
        sh "docker push ${fullImage}"
        
        // Push additional tags
        config.additionalTags?.each { additionalTag ->
            def additionalFullImage = "${registry}/${config.image}:${additionalTag}"
            sh "docker tag ${fullImage} ${additionalFullImage}"
            sh "docker push ${additionalFullImage}"
        }
    }
    
    return [
        image: config.image,
        registry: registry,
        tag: tag,
        fullTag: fullImage
    ]
}
```

### `src/com/mycompany/jenkins/Deploy.groovy`

```groovy
package com.mycompany.jenkins

class Deploy implements Serializable {
    def script
    String environment
    String kubeContext
    
    Deploy(script, String environment) {
        this.script = script
        this.environment = environment
        this.kubeContext = "k8s-${environment}"
    }
    
    def toKubernetes(String app, String image, String namespace) {
        script.sh """
            kubectl --context=${kubeContext} \
                set image deployment/${app} \
                ${app}=${image} \
                -n ${namespace}
            
            kubectl --context=${kubeContext} \
                rollout status deployment/${app} \
                -n ${namespace}
        """
    }
    
    def rollback(String app, String namespace) {
        script.sh """
            kubectl --context=${kubeContext} \
                rollout undo deployment/${app} \
                -n ${namespace}
        """
    }
    
    def getServiceUrl(String serviceName, String namespace) {
        return script.sh(
            script: """
                kubectl --context=${kubeContext} \
                    get service ${serviceName} \
                    -n ${namespace} \
                    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
            """,
            returnStdout: true
        ).trim()
    }
}
```

### การใช้งาน Shared Library ใน Jenkinsfile

```groovy
@Library('jenkins-shared-library@main') _

pipeline {
    agent any
    
    stages {
        stage('Build Docker') {
            steps {
                script {
                    def result = buildDocker(
                        image: 'myapp',
                        registry: 'registry.mycompany.com',
                        additionalTags: ['latest']
                    )
                    env.DOCKER_IMAGE = result.fullTag
                }
            }
        }
        
        stage('Deploy Staging') {
            steps {
                script {
                    deployApp(
                        app: 'myapp',
                        environment: 'staging',
                        image: env.DOCKER_IMAGE
                    )
                }
            }
        }
        
        stage('Deploy Production') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def deployer = new com.mycompany.jenkins.Deploy(this, 'production')
                    deployer.toKubernetes('myapp', env.DOCKER_IMAGE, 'myapp-production')
                    
                    def url = deployer.getServiceUrl('myapp', 'myapp-production')
                    echo "Application deployed at: ${url}"
                }
            }
        }
    }
    
    post {
        always {
            notifySlack(
                channel: '#deployments',
                status: currentBuild.result
            )
        }
    }
}
```

---

## 16.10 Triggers

### Webhook Trigger

```groovy
pipeline {
    agent any
    
    triggers {
        // GitHub webhook
        githubPush()
        
        // GitLab webhook  
        gitlab(triggerOnPush: true, triggerOnMergeRequest: true)
        
        // Bitbucket webhook
        bitbucketPush()
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
    }
}
```

### Cron Trigger

```groovy
pipeline {
    agent any
    
    triggers {
        // รัน nightly build ทุกวัน 2AM
        cron('0 2 * * *')
        
        // รัน ทุก 15 นาที
        cron('H/15 * * * *')
        
        // Poll SCM ทุก 5 นาที
        pollSCM('H/5 * * * *')
        
        // รัน nightly แค่วันจันทร์-ศุกร์
        cron('0 2 * * 1-5')
    }
    
    stages {
        stage('Nightly Tests') {
            steps {
                sh 'npm run test:full'
            }
        }
    }
}
```

### Upstream Trigger

```groovy
pipeline {
    agent any
    
    triggers {
        // Trigger เมื่อ upstream job เสร็จสิ้น
        upstream(upstreamProjects: 'build-myapp', threshold: hudson.model.Result.SUCCESS)
    }
    
    stages {
        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

---

## 16.11 Multibranch Pipeline

### Jenkinsfile สำหรับ Multibranch

```groovy
pipeline {
    agent any
    
    options {
        skipDefaultCheckout(true)
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('CI') {
            stages {
                stage('Build') {
                    steps {
                        sh 'npm ci && npm run build'
                    }
                }
                
                stage('Test') {
                    steps {
                        sh 'npm test'
                    }
                }
            }
        }
        
        // Deploy staging เฉพาะ develop branch
        stage('Deploy Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh './deploy.sh staging'
            }
            environment {
                name: staging
                url: https://staging.myapp.com
            }
        }
        
        // Deploy production เฉพาะ main branch
        stage('Deploy Production') {
            when {
                branch 'main'
            }
            input {
                message 'Deploy to production?'
                ok 'Deploy'
            }
            steps {
                sh './deploy.sh production'
            }
        }
        
        // PR testing
        stage('PR Validation') {
            when {
                changeRequest()
            }
            parallel {
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                }
                stage('Security') {
                    steps {
                        sh 'npm audit'
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
```

### การตั้งค่า Multibranch Pipeline

```groovy
// Jenkinsfile ที่ root ของ repository
// Jenkins จะ scan branches อัตโนมัติและสร้าง pipeline สำหรับแต่ละ branch

// ตั้งค่าผ่าน Job DSL หรือ Jenkins UI:
multibranchPipelineJob('my-multibranch') {
    branchSources {
        github {
            id('github-source')
            repoOwner('myorg')
            repository('myrepo')
            scanCredentialsId('github-credentials')
        }
    }
    
    orphanedItemStrategy {
        discardOldItems {
            daysToKeep(30)
            numToKeep(10)
        }
    }
    
    triggers {
        periodic(1)  // Scan ทุก 1 นาที
    }
}
```

---

## 16.12 Jenkins Agents Setup

### Static Agent

```bash
# เพิ่ม agent ผ่าน Jenkins UI
# Manage Jenkins > Manage Nodes and Clouds > New Node

# ตั้งค่า:
# - Node name: linux-builder-01
# - Remote root directory: /var/jenkins/agent
# - Labels: linux docker build
# - Launch method: Launch agents via SSH

# หรือใช้ Java Web Start
java -jar agent.jar \
  -jnlpUrl http://jenkins:8080/computer/linux-builder-01/slave-agent.jnlp \
  -secret AGENT_SECRET \
  -workDir /var/jenkins/agent
```

### Dynamic Agent (Kubernetes)

```groovy
// ใช้ Kubernetes plugin

pipeline {
    agent {
        kubernetes {
            cloud 'kubernetes'
            defaultContainer 'jnlp'
            yaml """
                apiVersion: v1
                kind: Pod
                metadata:
                  labels:
                    app: jenkins-agent
                spec:
                  serviceAccountName: jenkins-agent
                  containers:
                  - name: jnlp
                    image: jenkins/inbound-agent:latest
                    args: ['\$(JENKINS_SECRET)', '\$(JENKINS_NAME)']
                  - name: maven
                    image: maven:3.9-eclipse-temurin-17
                    command: ['cat']
                    tty: true
                    resources:
                      requests:
                        memory: "1Gi"
                        cpu: "500m"
                      limits:
                        memory: "2Gi"
                        cpu: "1"
                  - name: docker
                    image: docker:24-dind
                    securityContext:
                      privileged: true
                    env:
                    - name: DOCKER_TLS_CERTDIR
                      value: ""
                  volumes:
                  - name: docker-socket
                    emptyDir: {}
            """
        }
    }
    
    stages {
        stage('Build Java') {
            steps {
                container('maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Build Docker') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp .'
                }
            }
        }
    }
}
```

---

## 16.13 Backup และ Restore

### Backup Strategy

```bash
#!/bin/bash
# backup-jenkins.sh

JENKINS_HOME="/var/jenkins_home"
BACKUP_DIR="/backups/jenkins"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/jenkins_backup_$DATE.tar.gz"

# สร้าง backup directory
mkdir -p $BACKUP_DIR

# Backup Jenkins home
tar -czf $BACKUP_FILE \
    --exclude="$JENKINS_HOME/workspace" \
    --exclude="$JENKINS_HOME/logs" \
    --exclude="$JENKINS_HOME/.m2" \
    --exclude="$JENKINS_HOME/.npm" \
    $JENKINS_HOME

echo "Backup สำเร็จ: $BACKUP_FILE"

# ลบ backup เก่ากว่า 30 วัน
find $BACKUP_DIR -name "jenkins_backup_*.tar.gz" -mtime +30 -delete

# Upload ไปยัง S3
aws s3 cp $BACKUP_FILE s3://my-backups/jenkins/$DATE/

echo "Upload ไปยัง S3 สำเร็จ"
```

### Jenkins Backup Pipeline

```groovy
// Jenkinsfile สำหรับ backup
pipeline {
    agent any
    
    triggers {
        cron('0 1 * * *')  // รันทุกคืน 1AM
    }
    
    stages {
        stage('Backup Jenkins') {
            steps {
                sh '''
                    BACKUP_FILE="/tmp/jenkins_backup_$(date +%Y%m%d).tar.gz"
                    
                    tar -czf $BACKUP_FILE \
                        --exclude=/var/jenkins_home/workspace \
                        --exclude=/var/jenkins_home/logs \
                        /var/jenkins_home
                    
                    echo "Backup created: $BACKUP_FILE"
                    ls -lh $BACKUP_FILE
                '''
            }
        }
        
        stage('Upload to S3') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    credentialsId: 'aws-backup',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        BACKUP_FILE="/tmp/jenkins_backup_$(date +%Y%m%d).tar.gz"
                        aws s3 cp $BACKUP_FILE s3://my-company-backups/jenkins/
                        rm $BACKUP_FILE
                    '''
                }
            }
        }
        
        stage('Cleanup Old Backups') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    credentialsId: 'aws-backup',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        # ลบ backups เก่ากว่า 30 วัน
                        CUTOFF_DATE=$(date -d "30 days ago" +%Y%m%d)
                        aws s3 ls s3://my-company-backups/jenkins/ | while read -r line; do
                            FILE_DATE=$(echo $line | awk '{print $4}' | grep -o '[0-9]\{8\}')
                            if [ ! -z "$FILE_DATE" ] && [ "$FILE_DATE" -lt "$CUTOFF_DATE" ]; then
                                FILE=$(echo $line | awk '{print $4}')
                                aws s3 rm "s3://my-company-backups/jenkins/$FILE"
                                echo "Deleted: $FILE"
                            fi
                        done
                    '''
                }
            }
        }
    }
    
    post {
        success {
            slackSend channel: '#ops', color: 'good', message: '✅ Jenkins backup สำเร็จ'
        }
        failure {
            slackSend channel: '#ops', color: 'danger', message: '❌ Jenkins backup ล้มเหลว!'
        }
    }
}
```

---

## 16.14 Pipeline Performance Optimization

### Best Practices

```groovy
pipeline {
    agent any
    
    options {
        // ลดเวลา build ด้วย parallel compression
        parallelsAlwaysFailFast()
        
        // ตั้ง timeout เพื่อป้องกัน pipeline ค้าง
        timeout(time: 30, unit: 'MINUTES')
        
        // ลด log output
        quietPeriod(5)
        
        // Skip ถ้า SCM ไม่มี changes
        skipDefaultCheckout(false)
    }
    
    stages {
        stage('Checkout') {
            steps {
                // Shallow clone เพื่อเพิ่มความเร็ว
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: env.BRANCH_NAME]],
                    extensions: [[$class: 'CloneOption', depth: 10, shallow: true]],
                    userRemoteConfigs: [[url: 'https://github.com/myorg/myrepo.git']]
                ])
            }
        }
        
        stage('Build and Test') {
            parallel {
                stage('Frontend') {
                    agent { label 'node' }
                    steps {
                        dir('frontend') {
                            sh 'npm ci --cache /npm-cache'
                            sh 'npm run build'
                            sh 'npm test'
                        }
                    }
                }
                
                stage('Backend') {
                    agent { label 'java' }
                    steps {
                        dir('backend') {
                            // ใช้ Maven wrapper สำหรับ caching
                            sh './mvnw package -B -DskipTests=false'
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            // ทำความสะอาด workspace เสมอ
            cleanWs(deleteDirs: true, disableDeferredWipeout: false, notFailBuild: true)
        }
    }
}
```

---

## แบบฝึกหัด Part 16

### แบบฝึกหัดที่ 1: Jenkins Installation

ติดตั้ง Jenkins ด้วย Docker Compose ที่มี:
1. Jenkins controller
2. 2 static agents (Linux)
3. Nginx reverse proxy พร้อม SSL
4. Backup อัตโนมัติไปยัง S3

### แบบฝึกหัดที่ 2: Complete Pipeline

สร้าง Declarative Pipeline สำหรับ Java Spring Boot application ที่:
1. Checkout จาก GitHub พร้อม webhook trigger
2. Build ด้วย Maven
3. Test พร้อม parallel stages
4. SonarQube code analysis
5. Docker build และ push ไปยัง ECR
6. Deploy ไปยัง Kubernetes staging
7. Manual approval สำหรับ production
8. Rollback สำหรับ production

### แบบฝึกหัดที่ 3: Shared Library

สร้าง Shared Library ที่มี:
1. Reusable function สำหรับ Docker build
2. Kubernetes deployment helper
3. Slack notification utility
4. ทดสอบ library ใน test pipeline

### แบบฝึกหัดที่ 4: Multibranch Setup

ตั้งค่า Multibranch Pipeline ที่:
1. Scan branches จาก GitHub organization
2. รัน pipeline ที่แตกต่างกันสำหรับ main, develop, feature branches
3. Deploy review environment สำหรับ PR
4. ลบ environments อัตโนมัติเมื่อ branch ถูกลบ

### แบบฝึกหัดที่ 5: Jenkins as Code

กำหนดค่า Jenkins ทั้งหมดด้วย JCasC ที่รวม:
1. Security configuration
2. Plugin installation
3. Global credentials
4. Cloud configuration (Kubernetes)
5. Shared library configuration
6. Notification settings

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- **Jenkins Installation**: ติดตั้งด้วย Docker พร้อม configuration as code
- **Declarative Pipeline**: โครงสร้าง pipeline ที่ชัดเจนและ readable
- **Scripted Pipeline**: ความยืดหยุ่นสูงสุดด้วย Groovy
- **Agents**: static, dynamic (Docker, Kubernetes)
- **Plugins**: Blue Ocean, Pipeline, Git, Credentials
- **Shared Libraries**: code reuse สำหรับ pipeline
- **Triggers**: webhook, cron, upstream
- **Multibranch**: จัดการหลาย branches อัตโนมัติ
- **Backup**: ป้องกันข้อมูล Jenkins

ในบทถัดไป เราจะเรียนรู้แนวคิด Pipeline as Code ที่จะช่วยให้เราจัดการ pipeline ได้อย่างมีประสิทธิภาพมากขึ้น

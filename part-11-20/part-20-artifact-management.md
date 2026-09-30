# Part 20: Artifact Management & Storage - การจัดการ Artifacts ใน CI/CD

## สารบัญ

1. [บทนำ - Artifacts คืออะไร](#บทนำ)
2. [ประเภทของ Artifacts](#ประเภทของ-artifacts)
3. [Versioning Artifacts](#versioning-artifacts)
4. [Artifact Repositories](#artifact-repositories)
5. [npm Package Publishing](#npm-package-publishing)
6. [Python pip Package Publishing](#python-pip-publishing)
7. [Maven/Java Package Publishing](#maven-publishing)
8. [Container Image Registry](#container-registry)
9. [Artifact Promotion](#artifact-promotion)
10. [Retention Policies](#retention-policies)
11. [Artifact Signing & Provenance](#artifact-signing)
12. [GitHub Actions Artifacts](#github-actions-artifacts)
13. [Caching vs Artifacts](#caching-vs-artifacts)
14. [Cleanup Strategies](#cleanup-strategies)
15. [แบบฝึกหัด (Exercises)](#exercises)

---

## บทนำ - Artifacts คืออะไร {#บทนำ}

### ความหมายของ Artifact ใน Software Development

```
Artifact คือผลลัพธ์จากกระบวนการ build ที่สามารถ:
- Deploy ไปยัง server
- Share กับ developers คนอื่น
- Install เป็น dependency
- Archive สำหรับ audit/compliance

ตัวอย่าง Artifacts:
┌─────────────────────────────────────────────────────────┐
│ Source Code → Build Process → Artifacts                 │
│                                                         │
│ React App  → npm build    → dist/ folder (static files) │
│ Go App     → go build     → binary executable           │
│ Java App   → mvn package  → .jar / .war file            │
│ Python Pkg → python build → wheel (.whl) file           │
│ Docker     → docker build → container image             │
│ Helm Chart → helm package → .tgz chart                  │
└─────────────────────────────────────────────────────────┘
```

### ทำไม Artifact Management ถึงสำคัญ

```
ปัญหาที่เกิดขึ้นเมื่อไม่มี Artifact Management:

1. Reproducibility Problem:
   "Build ทำงานบน CI แต่ crash บน production"
   → ไม่รู้ว่า deploy artifact ไหน
   → Cannot reproduce the exact build

2. Supply Chain Security:
   "เราแน่ใจได้อย่างไรว่า artifact ที่ deploy ไม่ถูกแก้ไข?"
   → ไม่มี signing/verification
   → Malicious code injection risk

3. Rollback Issues:
   "อยากกลับไป version ก่อน แต่หา artifact ไม่เจอ"
   → ต้อง rebuild ซึ่งอาจได้ output ต่างกัน
   → Deployment ล่าช้า

4. Dependency Chaos:
   "npm install ได้ package ต่างกันทุกครั้ง"
   → ไม่มี lock file หรือ private registry
   → Security vulnerabilities ที่ไม่ตรวจสอบ
```

### Artifact Lifecycle

```
                Build
                  ↓
        ┌─────────────────┐
        │   Artifact Store │
        │  (Registry/Repo) │
        └─────────────────┘
                ↓
    ┌───────────────────────┐
    │      Promotion        │
    │  Dev → Stage → Prod   │
    └───────────────────────┘
                ↓
    ┌───────────────────────┐
    │     Deployment        │
    └───────────────────────┘
                ↓
    ┌───────────────────────┐
    │  Retention/Cleanup    │
    │  (Archive or Delete)  │
    └───────────────────────┘
```

---

## ประเภทของ Artifacts {#ประเภทของ-artifacts}

### 1. Binary Executables

```
ภาษา           ไฟล์ที่ได้                   ขนาดปกติ
Go          → binary (no extension)        5-50 MB
Rust        → binary                       2-20 MB
C/C++       → .exe (Windows) / binary      1-100 MB
.NET        → .exe / .dll                  5-100 MB
Java        → .jar / .war / .ear           5-200 MB
```

```yaml
# GitHub Actions: Build Go Binary
- name: Build Go binaries
  run: |
    # Build สำหรับหลาย platforms
    GOOS=linux GOARCH=amd64 go build -o dist/myapp-linux-amd64 ./cmd/myapp
    GOOS=linux GOARCH=arm64 go build -o dist/myapp-linux-arm64 ./cmd/myapp
    GOOS=darwin GOARCH=amd64 go build -o dist/myapp-darwin-amd64 ./cmd/myapp
    GOOS=darwin GOARCH=arm64 go build -o dist/myapp-darwin-arm64 ./cmd/myapp
    GOOS=windows GOARCH=amd64 go build -o dist/myapp-windows-amd64.exe ./cmd/myapp
    
    # สร้าง checksums
    cd dist && sha256sum * > checksums.txt

- name: Upload binaries
  uses: actions/upload-artifact@v4
  with:
    name: binaries
    path: dist/
    retention-days: 30
```

### 2. Container Images

```
Container Images เป็น artifact ที่นิยมมากที่สุดในยุคปัจจุบัน
- Self-contained (มี dependencies ทั้งหมด)
- Platform-independent (ทำงานได้ทุกที่ที่มี container runtime)
- Layered (efficient caching และ sharing)
- Immutable (ไม่เปลี่ยนหลัง build)

Image Naming Convention:
registry/namespace/image-name:tag

Examples:
docker.io/library/nginx:1.25
ghcr.io/myorg/myapp:v2.1.0
asia-southeast1-docker.pkg.dev/myproject/myrepo/myapp:latest
registry.example.com/myapp:abc123def
```

### 3. Language Packages

```
npm      → package.json + .tgz bundle → npmjs.com / GitHub Packages
pip      → setup.py + .whl + .tar.gz → PyPI / Artifact
Maven    → pom.xml + .jar            → Maven Central / Nexus
Gradle   → build.gradle + .aar/.jar  → Maven Central / Artifactory
NuGet    → .nuspec + .nupkg          → nuget.org
Cargo    → Cargo.toml               → crates.io
Composer → composer.json             → packagist.org
Helm     → Chart.yaml + .tgz        → Helm repository
```

### 4. Infrastructure Artifacts

```
Terraform:
- terraform plan output (.tfplan file)
- Terraform state (terraform.tfstate)
- Provider plugins

Ansible:
- Playbooks (ส่วนใหญ่ไม่ต้อง artifact - ใช้ git)
- Roles ที่เป็น packages

Packer:
- AMI ID (Amazon Machine Image)
- Vagrant boxes
- Docker images
```

---

## Versioning Artifacts {#versioning-artifacts}

### Semantic Versioning (SemVer)

```
Format: MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]

MAJOR: Breaking changes (API incompatible)
MINOR: New features (backward compatible)
PATCH: Bug fixes (backward compatible)

Examples:
1.0.0         ← Initial stable release
1.1.0         ← New feature added
1.1.1         ← Bug fix
2.0.0         ← Breaking change

Pre-release:
1.0.0-alpha.1
1.0.0-beta.1
1.0.0-rc.1    ← Release Candidate
1.0.0-rc.2

Build metadata:
1.0.0+20241201
1.0.0+build.123
```

### Versioning Strategies

```bash
# Strategy 1: Git Tag Based
# Version = git tag
git tag v1.2.3
git push origin v1.2.3

# CI reads version from git tag
VERSION=$(git describe --tags --abbrev=0)
echo "Building version: ${VERSION}"

# Strategy 2: Commit SHA Based
# เหมาะสำหรับ every-commit builds
VERSION="${GITHUB_SHA:0:7}"   # short SHA: abc1234
# หรือ full SHA: abc1234def5678901234567890123456789012345

# Strategy 3: CalVer (Calendar Versioning)
# ใช้ date เป็น version
VERSION="$(date +%Y.%m.%d)-$(git rev-parse --short HEAD)"
# เช่น: 2024.12.01-abc1234

# Strategy 4: Semantic Release (automated)
# ใช้ conventional commits เพื่อ auto-bump version
# feat: → minor bump
# fix:  → patch bump
# BREAKING CHANGE: → major bump
```

### Automated Semantic Versioning ด้วย GitHub Actions

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write
  packages: write

jobs:
  release:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # ต้องการ full history สำหรับ semantic-release
          token: ${{ secrets.RELEASE_TOKEN }}
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install semantic-release
        run: |
          npm install -g \
            semantic-release \
            @semantic-release/git \
            @semantic-release/changelog \
            @semantic-release/github \
            @semantic-release/exec
      
      - name: Run semantic release
        env:
          GITHUB_TOKEN: ${{ secrets.RELEASE_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: semantic-release
```

```json
// .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    [
      "@semantic-release/exec",
      {
        "prepareCmd": "echo ${nextRelease.version} > VERSION && make build VERSION=${nextRelease.version}"
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md", "VERSION"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

---

## Artifact Repositories {#artifact-repositories}

### เปรียบเทียบ Artifact Repositories

| Feature | GitHub Packages | Nexus Repository | JFrog Artifactory | AWS ECR |
|---------|-----------------|------------------|--------------------|---------|
| **ราคา** | Free (public) / Paid | Free OSS / Paid Pro | Paid | Pay per use |
| **Types รองรับ** | npm, Maven, Docker, etc. | npm, Maven, Docker, etc. | ทุกประเภท | Docker เท่านั้น |
| **Integration** | GitHub native | ต้องตั้งค่า | ต้องตั้งค่า | AWS native |
| **Proxy support** | ไม่มี | มี | มี | ไม่มี |
| **Storage** | รวมกับ GitHub plan | Dedicated | Dedicated | S3-backed |
| **สำหรับ** | GitHub users | On-premise | Enterprise | AWS users |

### Nexus Repository Setup

```yaml
# docker-compose.nexus.yml
version: '3.8'

services:
  nexus:
    image: sonatype/nexus3:3.62.0
    ports:
      - "8081:8081"     # Web UI + Maven/npm/PyPI
      - "8082:8082"     # Docker registry (hosted)
      - "8083:8083"     # Docker registry (proxy)
    volumes:
      - nexus_data:/nexus-data
    environment:
      INSTALL4J_ADD_VM_PARAMS: "-Xms1024m -Xmx2048m -XX:MaxDirectMemorySize=2703m"
    
    # Resource limits
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 2G

volumes:
  nexus_data:
    driver: local
```

```bash
# ตั้งค่า Nexus ผ่าน API (หลังจาก startup)

# สร้าง npm hosted repository
curl -X POST \
  "http://localhost:8081/service/rest/v1/repositories/npm/hosted" \
  -H "Content-Type: application/json" \
  -u "admin:admin123" \
  -d '{
    "name": "npm-releases",
    "online": true,
    "storage": {
      "blobStoreName": "default",
      "strictContentTypeValidation": true,
      "writePolicy": "ALLOW_ONCE"
    },
    "cleanup": {
      "policyNames": ["npm-cleanup-policy"]
    }
  }'

# สร้าง Docker hosted registry
curl -X POST \
  "http://localhost:8081/service/rest/v1/repositories/docker/hosted" \
  -H "Content-Type: application/json" \
  -u "admin:admin123" \
  -d '{
    "name": "docker-releases",
    "online": true,
    "storage": {
      "blobStoreName": "default",
      "strictContentTypeValidation": true,
      "writePolicy": "ALLOW_ONCE"
    },
    "docker": {
      "v1Enabled": false,
      "forceBasicAuth": true,
      "httpPort": 8082
    }
  }'
```

### JFrog Artifactory Configuration

```yaml
# artifactory สำหรับ local development/testing
# docker-compose.artifactory.yml
version: '3.8'

services:
  artifactory:
    image: releases-docker.jfrog.io/jfrog/artifactory-oss:7.71.11
    ports:
      - "8082:8082"
      - "8081:8081"
    volumes:
      - artifactory_data:/var/opt/jfrog/artifactory
    environment:
      JF_ROUTER_ENTRYPOINTS_EXTERNALPORT: 8082

  # PostgreSQL สำหรับ Artifactory
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: artifactory
      POSTGRES_USER: artifactory
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  artifactory_data:
  postgres_data:
```

---

## npm Package Publishing {#npm-package-publishing}

### โครงสร้าง npm Package

```
my-library/
├── src/
│   ├── index.ts
│   └── utils.ts
├── dist/                  ← compiled output (ไม่ commit)
├── package.json
├── tsconfig.json
├── .npmignore             ← files ที่ไม่ publish ไปยัง npm
└── README.md
```

```json
// package.json
{
  "name": "@myorg/my-library",
  "version": "1.2.3",
  "description": "My awesome library",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "prepublishOnly": "npm test && npm run build",
    "version": "npm run build && git add -A dist",
    "postversion": "git push && git push --tags"
  },
  "keywords": ["library", "utility"],
  "author": "Your Name",
  "license": "MIT",
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  }
}
```

### GitHub Actions: Publish npm Package

```yaml
# .github/workflows/npm-publish.yml
name: Publish npm Package

on:
  release:
    types: [published]

jobs:
  publish-npm:
    name: Publish to npm
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      id-token: write    # สำหรับ npm provenance
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build package
        run: npm run build
      
      - name: Verify package contents
        run: |
          # ตรวจสอบว่า package มี files ที่ถูกต้อง
          npm pack --dry-run
      
      - name: Publish to npm
        run: npm publish --provenance --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
      
      - name: Summary
        run: |
          PACKAGE_NAME=$(node -p "require('./package.json').name")
          PACKAGE_VERSION=$(node -p "require('./package.json').version")
          echo "Published ${PACKAGE_NAME}@${PACKAGE_VERSION} to npm"

  publish-github-packages:
    name: Publish to GitHub Packages
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js for GitHub Packages
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://npm.pkg.github.com'
          scope: '@${{ github.repository_owner }}'
      
      - run: npm ci
      - run: npm run build
      
      - name: Publish to GitHub Packages
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### การตั้งค่าสำหรับ Private npm Registry

```bash
# .npmrc (commit ได้ - ไม่มี token)
@myorg:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${NPM_TOKEN}

# สำหรับ Nexus
@myorg:registry=https://nexus.example.com/repository/npm-releases/
//nexus.example.com/repository/npm-releases/:_authToken=${NEXUS_TOKEN}
```

```yaml
# Consuming private packages ใน CI
- name: Setup npm with private registry
  run: |
    echo "@myorg:registry=https://npm.pkg.github.com" >> .npmrc
    echo "//npm.pkg.github.com/:_authToken=${NPM_TOKEN}" >> .npmrc
  env:
    NPM_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    
- name: Install dependencies
  run: npm ci
```

---

## Python pip Package Publishing {#python-pip-publishing}

### โครงสร้าง Python Package

```
my-python-lib/
├── src/
│   └── my_library/
│       ├── __init__.py
│       ├── core.py
│       └── utils.py
├── tests/
│   ├── test_core.py
│   └── test_utils.py
├── pyproject.toml          ← modern Python packaging
├── setup.cfg               ← หรือใช้ setup.cfg
├── MANIFEST.in
└── README.md
```

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-library"
version = "1.2.3"
description = "My awesome Python library"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.9"
authors = [
  {name = "Your Name", email = "you@example.com"},
]
classifiers = [
  "Development Status :: 4 - Beta",
  "Intended Audience :: Developers",
  "License :: OSI Approved :: MIT License",
  "Programming Language :: Python :: 3",
  "Programming Language :: Python :: 3.9",
  "Programming Language :: Python :: 3.10",
  "Programming Language :: Python :: 3.11",
  "Programming Language :: Python :: 3.12",
]
dependencies = [
  "requests>=2.28",
  "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
  "pytest>=7.0",
  "pytest-cov",
  "black",
  "ruff",
  "mypy",
]

[project.urls]
Homepage = "https://github.com/myorg/my-library"
Documentation = "https://docs.example.com"
Repository = "https://github.com/myorg/my-library"
Changelog = "https://github.com/myorg/my-library/CHANGELOG.md"

[tool.hatch.version]
path = "src/my_library/__init__.py"
```

### GitHub Actions: Publish Python Package

```yaml
# .github/workflows/pypi-publish.yml
name: Publish Python Package

on:
  release:
    types: [published]

jobs:
  build:
    name: Build distribution
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - name: Install build tools
        run: pip install build twine
      
      - name: Build package
        run: python -m build
      
      - name: Check package
        run: twine check dist/*
      
      - name: Upload distribution artifacts
        uses: actions/upload-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
  
  publish-pypi:
    name: Publish to PyPI
    needs: build
    runs-on: ubuntu-latest
    
    environment: pypi-release
    
    permissions:
      id-token: write    # สำหรับ Trusted Publisher (OIDC)
    
    steps:
      - name: Download distributions
        uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
      
      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        # ใช้ Trusted Publisher - ไม่ต้องใช้ API token!
        with:
          # ถ้าไม่ได้ตั้งค่า Trusted Publisher ใช้ token:
          # password: ${{ secrets.PYPI_API_TOKEN }}
          verbose: true
  
  publish-test-pypi:
    name: Publish to TestPyPI
    needs: build
    runs-on: ubuntu-latest
    
    # Publish to TestPyPI สำหรับทดสอบก่อน
    if: github.event_name == 'workflow_dispatch'
    
    permissions:
      id-token: write
    
    steps:
      - name: Download distributions
        uses: actions/download-artifact@v4
        with:
          name: python-package-distributions
          path: dist/
      
      - name: Publish to TestPyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          repository-url: https://test.pypi.org/legacy/
```

### Python Package ใน Private Registry

```bash
# ติดตั้งจาก private registry
pip install my-library \
  --index-url https://nexus.example.com/repository/pypi-releases/simple/ \
  --extra-index-url https://pypi.org/simple/

# หรือตั้งค่าใน pip.conf
cat > ~/.config/pip/pip.conf << 'EOF'
[global]
index-url = https://nexus.example.com/repository/pypi-proxy/simple/
extra-index-url = https://pypi.org/simple/
trusted-host = nexus.example.com
EOF

# Publish ไป private registry
twine upload \
  --repository-url https://nexus.example.com/repository/pypi-releases/ \
  --username ${NEXUS_USERNAME} \
  --password ${NEXUS_PASSWORD} \
  dist/*
```

---

## Maven/Java Package Publishing {#maven-publishing}

### pom.xml สำหรับ Publishing

```xml
<!-- pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>
  
  <groupId>com.example</groupId>
  <artifactId>my-library</artifactId>
  <version>1.2.3</version>
  <packaging>jar</packaging>
  
  <name>My Library</name>
  <description>My awesome Java library</description>
  <url>https://github.com/myorg/my-library</url>
  
  <licenses>
    <license>
      <name>MIT License</name>
      <url>https://www.opensource.org/licenses/mit-license.php</url>
    </license>
  </licenses>
  
  <developers>
    <developer>
      <name>Your Name</name>
      <email>you@example.com</email>
      <organization>MyOrg</organization>
      <organizationUrl>https://example.com</organizationUrl>
    </developer>
  </developers>
  
  <scm>
    <connection>scm:git:git://github.com/myorg/my-library.git</connection>
    <developerConnection>scm:git:ssh://github.com/myorg/my-library.git</developerConnection>
    <url>https://github.com/myorg/my-library/tree/main</url>
  </scm>
  
  <!-- Distribution management: กำหนดที่ publish -->
  <distributionManagement>
    <snapshotRepository>
      <id>ossrh</id>
      <url>https://s01.oss.sonatype.org/content/repositories/snapshots</url>
    </snapshotRepository>
    <repository>
      <id>ossrh</id>
      <url>https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/</url>
    </repository>
  </distributionManagement>
  
  <build>
    <plugins>
      <!-- Source attachment (required for Maven Central) -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-source-plugin</artifactId>
        <version>3.3.0</version>
        <executions>
          <execution>
            <id>attach-sources</id>
            <goals>
              <goal>jar-no-fork</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      
      <!-- Javadoc (required for Maven Central) -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-javadoc-plugin</artifactId>
        <version>3.6.3</version>
        <executions>
          <execution>
            <id>attach-javadocs</id>
            <goals>
              <goal>jar</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
      
      <!-- GPG signing (required for Maven Central) -->
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-gpg-plugin</artifactId>
        <version>3.1.0</version>
        <executions>
          <execution>
            <id>sign-artifacts</id>
            <phase>verify</phase>
            <goals>
              <goal>sign</goal>
            </goals>
            <configuration>
              <gpgArguments>
                <arg>--pinentry-mode</arg>
                <arg>loopback</arg>
              </gpgArguments>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

### GitHub Actions: Publish Maven Package

```yaml
# .github/workflows/maven-publish.yml
name: Maven Package Release

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          server-id: ossrh
          server-username: MAVEN_USERNAME
          server-password: MAVEN_PASSWORD
          gpg-private-key: ${{ secrets.GPG_PRIVATE_KEY }}
          gpg-passphrase: MAVEN_GPG_PASSPHRASE
      
      - name: Run tests
        run: mvn --batch-mode test
      
      - name: Publish to Maven Central
        run: mvn --batch-mode deploy -P release -DskipTests
        env:
          MAVEN_USERNAME: ${{ secrets.OSSRH_USERNAME }}
          MAVEN_PASSWORD: ${{ secrets.OSSRH_TOKEN }}
          MAVEN_GPG_PASSPHRASE: ${{ secrets.GPG_PASSPHRASE }}
      
      - name: Publish to GitHub Packages
        run: mvn --batch-mode deploy -P github-packages -DskipTests
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Container Image Registry {#container-registry}

### Docker Build Best Practices สำหรับ Artifacts

```dockerfile
# Dockerfile - Multi-stage build
# syntax=docker/dockerfile:1

# Stage 1: Dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Stage 2: Builder
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Runner (final artifact)
FROM node:20-alpine AS runner
WORKDIR /app

# Security: ไม่ run เป็น root
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built output เท่านั้น
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=deps --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY package.json ./

USER nextjs

EXPOSE 3000
ENV PORT 3000
ENV NODE_ENV production

CMD ["node", "dist/server.js"]
```

```yaml
# .github/workflows/container-build.yml
name: Build and Push Container

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-push:
    runs-on: ubuntu-latest
    
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
    
    outputs:
      image_digest: ${{ steps.build.outputs.digest }}
      image_tags: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            # Branch name: ghcr.io/myorg/myapp:main
            type=ref,event=branch
            # PR number: ghcr.io/myorg/myapp:pr-123
            type=ref,event=pr
            # Git tag: ghcr.io/myorg/myapp:v1.2.3
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
            # Short SHA: ghcr.io/myorg/myapp:sha-abc1234
            type=sha,prefix=sha-,format=short
          labels: |
            org.opencontainers.image.title=My Application
            org.opencontainers.image.description=My awesome application
            org.opencontainers.image.vendor=MyOrg
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64    # Multi-platform
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          # SBOM (Software Bill of Materials)
          sbom: true
          # Provenance attestation
          provenance: mode=max
      
      - name: Generate artifact attestation
        uses: actions/attest-build-provenance@v1
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
```

### Image Scanning ก่อน Push

```yaml
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
    format: 'sarif'
    output: 'trivy-results.sarif'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'    # Fail pipeline ถ้าพบ critical vulnerabilities

- name: Upload Trivy scan results
  uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: 'trivy-results.sarif'
```

---

## Artifact Promotion {#artifact-promotion}

### Promotion Pipeline

```yaml
# .github/workflows/artifact-promotion.yml
name: Promote Artifact

on:
  workflow_dispatch:
    inputs:
      artifact_version:
        description: 'Version to promote (e.g., v1.2.3 or sha-abc1234)'
        required: true
      target_environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  validate-artifact:
    name: Validate Artifact
    runs-on: ubuntu-latest
    
    outputs:
      validated: ${{ steps.validate.outputs.result }}
      full_image: ${{ steps.validate.outputs.full_image }}
    
    steps:
      - name: Validate artifact exists
        id: validate
        run: |
          REGISTRY="ghcr.io"
          IMAGE="${REGISTRY}/${{ github.repository }}:${{ inputs.artifact_version }}"
          
          # ตรวจสอบว่า image มีอยู่จริง
          if docker manifest inspect "${IMAGE}" > /dev/null 2>&1; then
            echo "✅ Artifact found: ${IMAGE}"
            echo "result=true" >> $GITHUB_OUTPUT
            echo "full_image=${IMAGE}" >> $GITHUB_OUTPUT
          else
            echo "❌ Artifact not found: ${IMAGE}"
            echo "result=false" >> $GITHUB_OUTPUT
            exit 1
          fi
      
      - name: Verify artifact passed tests
        run: |
          echo "Checking test status for ${{ inputs.artifact_version }}..."
          # ตรวจสอบ GitHub deployment status หรือ check runs
          # ในระบบจริงอาจ query database หรือ check GitHub API
  
  tag-for-environment:
    name: Tag Artifact for ${{ inputs.target_environment }}
    needs: validate-artifact
    runs-on: ubuntu-latest
    
    environment: ${{ inputs.target_environment }}
    
    permissions:
      packages: write
    
    steps:
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Tag artifact for environment
        run: |
          SOURCE_IMAGE="${{ needs.validate-artifact.outputs.full_image }}"
          ENV_TAG="ghcr.io/${{ github.repository }}:${{ inputs.target_environment }}"
          STABLE_TAG="ghcr.io/${{ github.repository }}:${{ inputs.target_environment }}-stable"
          TIMESTAMP=$(date +%Y%m%d-%H%M%S)
          DATED_TAG="ghcr.io/${{ github.repository }}:${{ inputs.target_environment }}-${TIMESTAMP}"
          
          echo "Promoting: ${SOURCE_IMAGE}"
          echo "  → ${ENV_TAG}"
          echo "  → ${DATED_TAG}"
          
          # ดึง image จาก registry
          docker pull "${SOURCE_IMAGE}"
          
          # Tag สำหรับ environment
          docker tag "${SOURCE_IMAGE}" "${ENV_TAG}"
          docker tag "${SOURCE_IMAGE}" "${DATED_TAG}"
          
          # Push tags ใหม่
          docker push "${ENV_TAG}"
          docker push "${DATED_TAG}"
          
          echo "✅ Promotion complete"
      
      - name: Create GitHub Deployment
        uses: actions/github-script@v7
        with:
          script: |
            const deployment = await github.rest.repos.createDeployment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              ref: '${{ inputs.artifact_version }}',
              environment: '${{ inputs.target_environment }}',
              description: 'Promoted via workflow_dispatch',
              auto_merge: false,
              required_contexts: [],
            });
            
            // Mark as successful
            await github.rest.repos.createDeploymentStatus({
              owner: context.repo.owner,
              repo: context.repo.repo,
              deployment_id: deployment.data.id,
              state: 'success',
              environment_url: context.payload.inputs.target_environment === 'production' 
                ? 'https://example.com' 
                : 'https://staging.example.com',
              description: 'Deployment successful',
            });
```

---

## Retention Policies {#retention-policies}

### ทำไมต้องมี Retention Policy

```
ปัญหาที่เกิดถ้าไม่มี Retention Policy:
- Storage costs พุ่ง (เก็บทุก artifact ตลอดไป)
- Registry ช้าลง (ต้อง scan/index artifacts เยอะ)
- ยากต่อการ find ว่า artifact ไหนใช้งาน
- Security risk (เก็บ artifacts เก่าที่มี vulnerabilities)

สิ่งที่ต้องคงไว้:
✅ Production deployments (เพื่อ rollback)
✅ Release versions (v1.0.0, v2.0.0)
✅ Artifacts ที่อยู่ใน staging/production environments
✅ Last N builds สำหรับ main branch

สิ่งที่สามารถลบได้:
❌ PR/feature branch builds > 7 วัน หลัง PR ปิด
❌ Development builds > 30 วัน
❌ Failed builds ทันที (optional)
❌ Snapshots/pre-release > 90 วัน
```

### GitHub Actions Artifact Retention

```yaml
# การตั้งค่า retention ใน upload-artifact
- name: Upload test results
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results/
    retention-days: 7          # เก็บแค่ 7 วัน

- name: Upload production artifact
  uses: actions/upload-artifact@v4
  with:
    name: production-build-${{ github.sha }}
    path: dist/
    retention-days: 90         # เก็บ 90 วันสำหรับ production
    
- name: Upload debug logs (short retention)
  uses: actions/upload-artifact@v4
  with:
    name: debug-logs
    path: logs/
    retention-days: 3          # แค่ 3 วันสำหรับ debug logs
```

### Container Image Retention Policy

```yaml
# .github/workflows/cleanup-images.yml
name: Cleanup Old Container Images

on:
  schedule:
    - cron: '0 3 * * 0'    # ทุกวันอาทิตย์ ตี 3

jobs:
  cleanup:
    runs-on: ubuntu-latest
    
    permissions:
      packages: write
    
    steps:
      - name: Delete old dev images
        uses: actions/github-script@v7
        with:
          script: |
            const owner = context.repo.owner;
            const packageName = 'myapp';  // ชื่อ package ใน GHCR
            
            // ดึง all versions
            const { data: versions } = await github.rest.packages.getAllPackageVersionsForPackageOwnedByOrg({
              package_type: 'container',
              package_name: packageName,
              org: owner,
              per_page: 100,
            });
            
            const thirtyDaysAgo = new Date();
            thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);
            
            let deletedCount = 0;
            
            for (const version of versions) {
              const tags = version.metadata?.container?.tags || [];
              const createdAt = new Date(version.created_at);
              
              // ลบ conditions:
              // 1. ไม่มี tag (dangling images)
              // 2. dev- prefix และเก่ากว่า 30 วัน
              // 3. pr- prefix และเก่ากว่า 14 วัน
              const shouldDelete = 
                (tags.length === 0 && createdAt < thirtyDaysAgo) ||
                (tags.some(t => t.startsWith('dev-')) && createdAt < thirtyDaysAgo) ||
                (tags.some(t => t.startsWith('pr-')) && createdAt < new Date(Date.now() - 14 * 86400000));
              
              // ห้ามลบ:
              // - production, staging tags
              // - semantic version tags (v1.2.3)
              // - latest
              const protectedTags = ['production', 'staging', 'latest'];
              const hasProtectedTag = tags.some(t => 
                protectedTags.includes(t) || /^v\d+\.\d+\.\d+/.test(t)
              );
              
              if (shouldDelete && !hasProtectedTag) {
                try {
                  await github.rest.packages.deletePackageVersionForOrg({
                    package_type: 'container',
                    package_name: packageName,
                    org: owner,
                    package_version_id: version.id,
                  });
                  
                  deletedCount++;
                  console.log(`Deleted version ${version.id} (tags: ${tags.join(', ')})`);
                } catch (err) {
                  console.error(`Failed to delete ${version.id}: ${err.message}`);
                }
              }
            }
            
            console.log(`Cleanup complete. Deleted ${deletedCount} versions.`);
```

### Nexus Cleanup Policy

```groovy
// Nexus Cleanup Policy (REST API)
// สร้างผ่าน POST /service/rest/v1/cleanup-policies

{
  "name": "cleanup-dev-images",
  "format": "docker",
  "notes": "Delete development Docker images older than 30 days",
  "criteria": {
    "lastDownloaded": 30,    // ไม่ได้ใช้ใน 30 วัน
    "lastBlobUpdated": 30,   // เก่ากว่า 30 วัน
    "regex": ".*dev.*"       // pattern ที่ match (dev images)
  }
}
```

---

## Artifact Signing & Provenance {#artifact-signing}

### ทำไมต้อง Sign Artifacts

```
Supply Chain Attacks เป็นหนึ่งในภัยคุกคามที่ใหญ่ที่สุดในปัจจุบัน

ตัวอย่าง:
- SolarWinds Breach (2020): Malicious code ถูก inject ใน build process
- XZ Utils backdoor (2024): Backdoor ใน compression library
- event-stream npm incident (2018): Malicious package ใน popular npm library

Artifact Signing ช่วย:
1. Verify ว่า artifact มาจาก trusted source
2. Detect tampering (ถ้า artifact ถูกแก้ไข signature จะ invalid)
3. Non-repudiation (ผู้ sign ไม่สามารถปฏิเสธได้)
4. Compliance requirements (SLSA, FedRAMP, etc.)
```

### Cosign สำหรับ Container Image Signing

```bash
# ติดตั้ง cosign
curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign

# Generate key pair
cosign generate-key-pair
# สร้าง cosign.key (private) และ cosign.pub (public)

# Sign image
cosign sign --key cosign.key ghcr.io/myorg/myapp:v1.2.3

# Verify signature
cosign verify --key cosign.pub ghcr.io/myorg/myapp:v1.2.3

# Keyless signing (ใช้ OIDC - แนะนำมากกว่า)
cosign sign ghcr.io/myorg/myapp:v1.2.3 \
  --identity-token=$(gcloud auth print-identity-token)

# Verify keyless
cosign verify \
  --certificate-identity-regexp=".*@myorg.com" \
  --certificate-oidc-issuer="https://accounts.google.com" \
  ghcr.io/myorg/myapp:v1.2.3
```

### GitHub Actions: Signing Container Images

```yaml
# .github/workflows/sign-and-publish.yml
name: Build, Sign, and Push

on:
  push:
    tags: ['v*']

permissions:
  contents: read
  packages: write
  id-token: write    # สำหรับ cosign keyless signing

jobs:
  build-sign-push:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install cosign
        uses: sigstore/cosign-installer@v3
      
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      
      - name: Build and push image
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          # Include SBOM สำหรับ provenance
          sbom: true
          provenance: mode=max
      
      - name: Sign the container image (keyless)
        run: |
          IMAGE="ghcr.io/${{ github.repository }}"
          DIGEST="${{ steps.build-push.outputs.digest }}"
          
          # Sign โดยใช้ image digest (immutable reference)
          cosign sign --yes "${IMAGE}@${DIGEST}"
          
          echo "✅ Signed: ${IMAGE}@${DIGEST}"
      
      - name: Generate SBOM (Software Bill of Materials)
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}@${{ steps.build-push.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json
      
      - name: Attach SBOM to image
        run: |
          cosign attach sbom \
            --sbom sbom.spdx.json \
            ghcr.io/${{ github.repository }}@${{ steps.build-push.outputs.digest }}
```

### SLSA (Supply chain Levels for Software Artifacts)

```yaml
# SLSA Level 2+ ด้วย GitHub Actions
# .github/workflows/slsa-build.yml

name: SLSA-compliant Build

on:
  release:
    types: [created]

jobs:
  build:
    outputs:
      hashes: ${{ steps.hash.outputs.hashes }}
    
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build artifacts
        run: make build
      
      - name: Generate hashes
        id: hash
        run: |
          HASHES=$(sha256sum ./dist/* | base64 -w0)
          echo "hashes=${HASHES}" >> $GITHUB_OUTPUT
  
  # SLSA Provenance Generator
  provenance:
    needs: [build]
    
    permissions:
      actions: read
      id-token: write
      contents: write
    
    # ใช้ official SLSA generator
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: "${{ needs.build.outputs.hashes }}"
      upload-assets: true
```

---

## GitHub Actions Artifacts {#github-actions-artifacts}

### Upload และ Download Artifacts

```yaml
# .github/workflows/artifact-example.yml
name: Artifact Examples

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build application
        run: |
          mkdir -p dist
          echo "Building..." 
          npm run build
      
      # Upload artifacts จาก job นี้
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-output-${{ github.sha }}
          path: |
            dist/
            !dist/**/*.map     # Exclude source maps
          retention-days: 30
          compression-level: 9  # Max compression
          if-no-files-found: error   # Fail ถ้าไม่มีไฟล์
  
  test:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # Download artifacts จาก job ก่อนหน้า
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output-${{ github.sha }}
          path: dist/
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()    # Upload แม้ tests fail
        with:
          name: test-results
          path: |
            test-results/
            coverage/
          retention-days: 7
  
  # Download artifacts จาก different job หรือ workflow
  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    
    steps:
      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          # Download ทุก artifacts (ไม่ระบุ name)
          path: artifacts/
      
      - name: List downloaded artifacts
        run: ls -la artifacts/
      
      - name: Deploy
        run: |
          cp -r artifacts/build-output-${{ github.sha }}/* /deploy/
```

### Artifacts ข้าม Workflows

```yaml
# workflow-a.yml - Upload artifact
- name: Upload shared artifact
  uses: actions/upload-artifact@v4
  with:
    name: shared-config
    path: config/shared/
    retention-days: 1    # Short retention สำหรับ inter-workflow

# workflow-b.yml - Download artifact จาก workflow อื่น
- name: Download artifact from other workflow
  uses: dawidd6/action-download-artifact@v3
  with:
    workflow: workflow-a.yml
    name: shared-config
    path: config/
```

### Matrix Build Artifacts

```yaml
name: Multi-platform Build

jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        arch: [amd64, arm64]
        exclude:
          - os: windows-latest
            arch: arm64
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build for ${{ matrix.os }}-${{ matrix.arch }}
        run: |
          # Build command แตกต่างกันตาม OS
          if [ "${{ matrix.os }}" == "windows-latest" ]; then
            go build -o dist/myapp.exe ./cmd/myapp
          else
            GOARCH=${{ matrix.arch }} go build -o dist/myapp-${{ matrix.arch }} ./cmd/myapp
          fi
      
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          # ชื่อ unique สำหรับแต่ละ matrix combination
          name: myapp-${{ matrix.os }}-${{ matrix.arch }}
          path: dist/
  
  release:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
      # Download ทุก artifacts จาก matrix
      - name: Download all platform artifacts
        uses: actions/download-artifact@v4
        with:
          path: all-artifacts/
          # Download ทุก artifacts ที่ขึ้นต้นด้วย "myapp-"
          pattern: myapp-*
          merge-multiple: false
      
      - name: Create release bundle
        run: |
          ls -la all-artifacts/
          # Pack ทุก binaries ลงใน release
```

---

## Caching vs Artifacts {#caching-vs-artifacts}

### ความแตกต่างระหว่าง Cache และ Artifact

```
┌────────────────────────────────────────────────────────────────────┐
│                         Cache vs Artifact                          │
├──────────────────┬─────────────────────┬───────────────────────────┤
│ Feature          │ Cache               │ Artifact                  │
├──────────────────┼─────────────────────┼───────────────────────────┤
│ วัตถุประสงค์     │ เร่งความเร็ว build  │ ส่งต่อผลลัพธ์ระหว่าง jobs│
│ อายุ             │ 7 วัน (auto expire)  │ กำหนดได้ (default 90 วัน)│
│ ขนาดจำกัด        │ 10 GB               │ 2 GB / artifact           │
│ Share ข้าม runs  │ ใช่                 │ ปกติไม่ (ใช้ชื่อ unique)  │
│ ใช้สำหรับ        │ dependencies, build │ build outputs, test results│
│                  │ caches, tools       │ reports, binaries          │
│ Invalidation     │ cache key เปลี่ยน   │ ไม่ auto-invalidate        │
│ Encryption       │ ไม่มี              │ ไม่มี                     │
└──────────────────┴─────────────────────┴───────────────────────────┘
```

### Caching Best Practices

```yaml
# .github/workflows/caching-example.yml
name: Build with Caching

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # Node.js - cache node_modules
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'    # Built-in cache สำหรับ npm/yarn/pnpm
      
      # Go - cache modules และ build cache
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'
          cache: true     # Built-in Go caching
      
      # Python - cache pip packages
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      
      # Docker layer caching
      - name: Build Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          cache-from: type=gha          # GitHub Actions cache
          cache-to: type=gha,mode=max   # Save cache
      
      # Custom cache (เช่น Gradle, Maven, Cargo)
      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          # Cache key รวม hash ของ build files
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            ${{ runner.os }}-gradle-
```

### Advanced Cache Strategies

```yaml
# Cache Layering Strategy
# Primary key: exact match (fastest)
# Restore keys: partial match (fallback)

- name: Cache npm dependencies
  uses: actions/cache@v4
  id: npm-cache
  with:
    path: ~/.npm
    # Primary key: exact match กับ lock file
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    # Fallback keys: ถ้า primary miss
    restore-keys: |
      ${{ runner.os }}-node-${{ hashFiles('**/package.json') }}
      ${{ runner.os }}-node-

- name: Install dependencies
  # ถ้า cache hit - ข้าม install
  if: steps.npm-cache.outputs.cache-hit != 'true'
  run: npm ci

# Cache Warming (สร้าง cache ล่วงหน้า)
- name: Pre-warm Docker cache
  run: |
    # Pull base images ก่อนเพื่อ warm layer cache
    docker pull node:20-alpine
    docker pull nginx:alpine
```

---

## Cleanup Strategies {#cleanup-strategies}

### ระดับการ Cleanup

```
1. CI/CD Level (GitHub Actions artifacts):
   - ตั้ง retention-days ที่เหมาะสม
   - Automated deletion หลัง workflow
   
2. Container Registry Level:
   - Tag lifecycle rules
   - Auto-delete untagged images
   - Environment-based retention
   
3. Package Registry Level (npm/pip/maven):
   - Archive deprecated versions
   - Enforce version limits
   
4. Storage Level (S3, GCS, Azure Blob):
   - Lifecycle policies based on age/access
   - Tiered storage (hot/warm/cold/archive)
```

### S3 Lifecycle Policy สำหรับ Artifacts

```json
{
  "Rules": [
    {
      "ID": "development-artifacts-cleanup",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "artifacts/dev/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"    // ย้ายไป Infrequent Access
        },
        {
          "Days": 60,
          "StorageClass": "GLACIER"        // ย้ายไป Glacier (เย็นลง)
        }
      ],
      "Expiration": {
        "Days": 90                         // ลบหลัง 90 วัน
      }
    },
    {
      "ID": "production-artifacts-retention",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "artifacts/production/"
      },
      "Transitions": [
        {
          "Days": 180,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 365,
          "StorageClass": "GLACIER_IR"
        }
      ]
      // ไม่มี Expiration - เก็บตลอดไป
    },
    {
      "ID": "delete-incomplete-uploads",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7           // ลบ incomplete uploads หลัง 7 วัน
      }
    }
  ]
}
```

### Automated Cleanup Workflow

```yaml
# .github/workflows/artifact-cleanup.yml
name: Artifact Cleanup

on:
  schedule:
    - cron: '0 4 * * 0'    # ทุกอาทิตย์ ตี 4
  workflow_dispatch:
    inputs:
      dry_run:
        type: boolean
        default: true
        description: 'Dry run mode'

jobs:
  cleanup-container-images:
    name: Cleanup Container Images
    runs-on: ubuntu-latest
    
    permissions:
      packages: write
    
    steps:
      - name: Get list of packages
        id: packages
        uses: actions/github-script@v7
        with:
          result-encoding: json
          script: |
            const response = await github.rest.packages.listPackagesForOrganization({
              package_type: 'container',
              org: context.repo.owner,
              per_page: 100,
            });
            return response.data.map(p => p.name);
      
      - name: Cleanup old versions
        uses: actions/github-script@v7
        env:
          DRY_RUN: ${{ inputs.dry_run || 'true' }}
        with:
          script: |
            const packages = ${{ steps.packages.outputs.result }};
            const dryRun = process.env.DRY_RUN === 'true';
            
            const retentionDays = {
              'dev': 7,
              'pr': 14,
              'staging': 60,
              'production': 365,
              'v': Infinity,    // semver tags เก็บตลอดไป
            };
            
            let stats = { scanned: 0, deleted: 0, kept: 0, errors: 0 };
            
            for (const pkg of packages) {
              const { data: versions } = await github.rest.packages.getAllPackageVersionsForPackageOwnedByOrg({
                package_type: 'container',
                package_name: pkg,
                org: context.repo.owner,
              });
              
              for (const version of versions) {
                stats.scanned++;
                const tags = version.metadata?.container?.tags || [];
                const ageInDays = (Date.now() - new Date(version.created_at)) / 86400000;
                
                let shouldKeep = false;
                let reason = '';
                
                for (const [prefix, days] of Object.entries(retentionDays)) {
                  if (tags.some(t => t.startsWith(prefix))) {
                    if (ageInDays <= days) {
                      shouldKeep = true;
                      reason = `tag with prefix '${prefix}' within ${days} days`;
                    }
                  }
                }
                
                // ถ้าไม่มี tag ใดๆ
                if (tags.length === 0 && ageInDays > 7) {
                  shouldKeep = false;
                  reason = 'untagged and older than 7 days';
                }
                
                if (shouldKeep) {
                  stats.kept++;
                } else {
                  stats.deleted++;
                  if (!dryRun) {
                    try {
                      await github.rest.packages.deletePackageVersionForOrg({
                        package_type: 'container',
                        package_name: pkg,
                        org: context.repo.owner,
                        package_version_id: version.id,
                      });
                    } catch (e) {
                      stats.errors++;
                      core.warning(`Failed to delete: ${e.message}`);
                    }
                  } else {
                    console.log(`[DRY RUN] Would delete: ${pkg}@${version.name} (tags: ${tags.join(',')})`);
                  }
                }
              }
            }
            
            console.log('\n=== Cleanup Summary ===');
            console.log(`Scanned: ${stats.scanned}`);
            console.log(`Would delete: ${stats.deleted}`);
            console.log(`Would keep: ${stats.kept}`);
            console.log(`Errors: ${stats.errors}`);
            console.log(`Mode: ${dryRun ? 'DRY RUN' : 'ACTUAL DELETE'}`);
```

### npm Package Deprecation

```bash
# Deprecate เก่า versions (ไม่ลบ แต่แสดง warning เมื่อ install)
npm deprecate @myorg/my-library@"< 2.0.0" "This version is deprecated. Please upgrade to >= 2.0.0"

# ลบ version (ทำได้ภายใน 72 ชั่วโมงหลัง publish)
npm unpublish @myorg/my-library@1.0.0

# ลบ package ทั้งหมด (สำหรับ packages ที่ไม่ได้ publish > 72 ชั่วโมง)
# ต้องติดต่อ npm support
```

---

## แบบฝึกหัด (Exercises) {#exercises}

### Exercise 1: Build Complete Artifact Pipeline

สร้าง CI/CD pipeline ที่:
1. Build application
2. Run tests
3. Build Docker image (multi-platform)
4. Sign image ด้วย cosign
5. Push to GitHub Container Registry
6. สร้าง GitHub Release พร้อม changelog

```yaml
# solution: .github/workflows/complete-artifact-pipeline.yml
name: Complete Artifact Pipeline

on:
  push:
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

permissions:
  contents: write
  packages: write
  id-token: write
  attestations: write

jobs:
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Extract version
        id: version
        run: echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      - run: npm test
      - run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
          retention-days: 30
  
  container:
    name: Build & Sign Container
    needs: build-test
    runs-on: ubuntu-latest
    
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
      
      - name: Install cosign
        uses: sigstore/cosign-installer@v3
      
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=semver,pattern={{major}}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          sbom: true
          provenance: mode=max
      
      - name: Sign image
        run: |
          cosign sign --yes \
            "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}"
      
      - name: Generate attestation
        uses: actions/attest-build-provenance@v1
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
  
  release:
    name: Create GitHub Release
    needs: [build-test, container]
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
      
      - name: Generate checksums
        run: |
          cd dist
          sha256sum * > SHA256SUMS
          cat SHA256SUMS
      
      - name: Create release
        uses: softprops/action-gh-release@v2
        with:
          files: |
            dist/*
            dist/SHA256SUMS
          generate_release_notes: true
          body: |
            ## Docker Images
            
            ```bash
            # Pull latest
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-test.outputs.version }}
            
            # Verify signature
            cosign verify \
              --certificate-identity-regexp="https://github.com/${{ github.repository }}" \
              --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
              ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ needs.container.outputs.digest }}
            ```
            
            ## Artifact Digest
            `${{ needs.container.outputs.digest }}`
```

### Exercise 2: Setup Artifact Retention Policy

สร้าง workflow ที่:
1. Build artifacts ด้วย different retention periods ตาม branch
2. Cleanup artifacts เก่าทุกสัปดาห์
3. Report storage usage

```yaml
# solution snippet - retention based on branch
- name: Upload artifact with branch-based retention
  uses: actions/upload-artifact@v4
  with:
    name: build-${{ github.sha }}
    path: dist/
    retention-days: >-
      ${{
        github.ref == 'refs/heads/main' && 90 ||
        github.ref_type == 'tag' && 365 ||
        startsWith(github.ref, 'refs/heads/feature/') && 7 ||
        30
      }}
```

### Exercise 3: Implement Package Publishing

**งาน:**
1. สร้าง npm library เล็กๆ (utility functions)
2. Setup GitHub Actions สำหรับ automatic publishing เมื่อ tag
3. Publish ทั้งไปยัง npm public registry และ GitHub Packages
4. Add automated versioning ด้วย semantic-release

```bash
# Template สำหรับ npm library
mkdir my-utils && cd my-utils
npm init -y

# สร้างไฟล์พื้นฐาน
mkdir src

cat > src/index.js << 'EOF'
/**
 * Format date to Thai locale
 * @param {Date} date
 * @returns {string}
 */
function formatThaiDate(date) {
  return new Intl.DateTimeFormat('th-TH', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(date);
}

/**
 * Truncate string with ellipsis
 * @param {string} str
 * @param {number} maxLength
 * @returns {string}
 */
function truncate(str, maxLength = 100) {
  if (str.length <= maxLength) return str;
  return str.slice(0, maxLength - 3) + '...';
}

module.exports = { formatThaiDate, truncate };
EOF

# สร้าง test
mkdir tests
cat > tests/index.test.js << 'EOF'
const { formatThaiDate, truncate } = require('../src');

test('formatThaiDate formats correctly', () => {
  const date = new Date('2024-01-15');
  const result = formatThaiDate(date);
  expect(result).toContain('2567');    // Thai year
});

test('truncate works correctly', () => {
  expect(truncate('hello world', 5)).toBe('he...');
  expect(truncate('hi', 10)).toBe('hi');
});
EOF
```

### Exercise 4: Artifact Verification

สร้าง script ที่:
1. Verify container image signature ด้วย cosign
2. Check SBOM สำหรับ known vulnerabilities
3. Verify checksums ของ binary artifacts
4. Generate audit report

```bash
#!/bin/bash
# solution/scripts/verify-artifact.sh

set -euo pipefail

IMAGE="${1:-}"
EXPECTED_DIGEST="${2:-}"

if [[ -z "${IMAGE}" ]]; then
  echo "Usage: $0 <image:tag> [expected_digest]"
  exit 1
fi

echo "=== Artifact Verification Report ==="
echo "Image: ${IMAGE}"
echo "Date: $(date)"
echo ""

# 1. Verify signature
echo "--- Verifying signature ---"
if cosign verify \
  --certificate-identity-regexp="https://github.com/myorg/myapp" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  "${IMAGE}" 2>/dev/null; then
  echo "✅ Signature valid"
else
  echo "❌ Signature verification failed!"
  exit 1
fi

# 2. Check digest ถ้าระบุมา
if [[ -n "${EXPECTED_DIGEST}" ]]; then
  echo ""
  echo "--- Verifying digest ---"
  ACTUAL_DIGEST=$(docker inspect "${IMAGE}" --format='{{index .RepoDigests 0}}' 2>/dev/null | cut -d'@' -f2)
  
  if [[ "${ACTUAL_DIGEST}" == "${EXPECTED_DIGEST}" ]]; then
    echo "✅ Digest matches: ${EXPECTED_DIGEST}"
  else
    echo "❌ Digest mismatch!"
    echo "   Expected: ${EXPECTED_DIGEST}"
    echo "   Actual:   ${ACTUAL_DIGEST}"
    exit 1
  fi
fi

# 3. Scan สำหรับ vulnerabilities
echo ""
echo "--- Scanning for vulnerabilities ---"
if command -v trivy &>/dev/null; then
  trivy image \
    --severity HIGH,CRITICAL \
    --exit-code 0 \
    --format table \
    "${IMAGE}" || echo "⚠️  Vulnerabilities found (see above)"
else
  echo "Trivy not installed, skipping scan"
fi

echo ""
echo "=== Verification Complete ==="
```

---

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:

1. **Artifacts คืออะไร** - ผลลัพธ์จาก build ที่พร้อม deploy/distribute
2. **ประเภทของ Artifacts** - Binaries, containers, packages, infrastructure
3. **Versioning** - SemVer, Git tags, automated semantic versioning
4. **Artifact Repositories** - Nexus, Artifactory, GitHub Packages, cloud registries
5. **Package Publishing** - npm, pip, Maven workflows
6. **Container Images** - Multi-platform builds, registries, best practices
7. **Artifact Promotion** - Build once, promote ข้าม environments
8. **Retention Policies** - ประหยัด storage, manage lifecycle
9. **Signing & Provenance** - cosign, SLSA, supply chain security
10. **GitHub Actions Artifacts** - Upload/download ระหว่าง jobs
11. **Caching vs Artifacts** - ใช้ให้ถูกวัตถุประสงค์
12. **Cleanup Strategies** - Automated deletion, lifecycle policies

### Key Takeaways

- **Build once, promote everywhere** - artifact เดิมผ่านทุก environment
- **Version everything** - ทั้ง semver และ immutable digests
- **Sign artifacts** - Supply chain security เป็นเรื่องสำคัญ
- **Retention policy จำเป็น** - ป้องกัน storage bloat และลด costs
- **Cache ≠ Artifact** - ใช้ต่างจุดประสงค์กัน
- **SBOM** - รู้ว่า artifact ของคุณมีอะไรอยู่ข้างใน
- **Automate cleanup** - manual cleanup ไม่ทำงานในระยะยาว

### Artifact Management Maturity Model

```
Level 1 (Basic):
- Manual upload/download
- ไม่มี versioning
- เก็บทุก artifacts ตลอดไป

Level 2 (Managed):
- Automated build and store
- Semantic versioning
- Basic retention policies

Level 3 (Optimized):
- Artifact promotion pipeline
- Automated cleanup
- Caching optimization

Level 4 (Advanced):
- Artifact signing
- SBOM generation
- Supply chain verification
- Provenance tracking

Level 5 (Expert):
- SLSA compliance
- Automated vulnerability scanning in artifacts
- Policy as code สำหรับ retention
- Cross-organization artifact sharing
```

---

## บทส่งท้าย - ภาพรวมของ Part 18-20

เราได้ครอบคลุม 3 เรื่องสำคัญที่เชื่อมโยงกัน:

```
Environment Management (Part 18)
         ↓
  กำหนด environments และ lifecycle ของ deployment
         ↓
Secrets Management (Part 19)
         ↓
  จัดการ credentials ที่ใช้ใน environments อย่างปลอดภัย
         ↓
Artifact Management (Part 20)
         ↓
  จัดการ output จาก build process ที่ถูก deploy ไปยัง environments
```

ทั้งสามส่วนนี้ทำงานร่วมกัน:
- **Environment** กำหนดว่า artifact ไหนถูก deploy ที่ไหน
- **Secrets** ให้ credentials ที่จำเป็นสำหรับแต่ละ environment
- **Artifacts** คือ software ที่ถูก deploy ไปยัง environments นั้น

การ Master ทั้งสามส่วนนี้จะทำให้ CI/CD pipeline ของคุณ:
- **Secure** - Secrets ไม่ถูก leak, Artifacts ถูก sign
- **Reliable** - Build once, test everywhere, deploy confidently
- **Efficient** - Caching และ retention policies ที่เหมาะสม
- **Auditable** - รู้ว่า version ไหนอยู่ที่ไหน เมื่อไหร่

---

*จบ Part 20 - Artifact Management & Storage*

*ขอบคุณที่เรียน CI/CD Course จนถึงบทนี้!*

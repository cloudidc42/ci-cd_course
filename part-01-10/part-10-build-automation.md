# Part 10: Build Automation & Makefile

> **หลักสูตร CI/CD สำหรับนักพัฒนา** | ระดับ: ปานกลาง-สูง | เวลาเรียน: ~5 ชั่วโมง

---

## สารบัญ

1. [Build Automation คืออะไร?](#1-build-automation-คืออะไร)
2. [Makefile: Syntax และ Targets](#2-makefile-syntax-และ-targets)
3. [npm/yarn Scripts](#3-npmyarn-scripts)
4. [Gradle Build Scripts](#4-gradle-build-scripts)
5. [Maven Build Scripts](#5-maven-build-scripts)
6. [Build Optimization](#6-build-optimization)
7. [Incremental Builds](#7-incremental-builds)
8. [Build Caching ใน CI](#8-build-caching-ใน-ci)
9. [Multi-stage Builds](#9-multi-stage-builds)
10. [Versioning Strategy (SemVer)](#10-versioning-strategy-semver)
11. [Changelog Generation](#11-changelog-generation)
12. [GitHub Actions Build Workflow](#12-github-actions-build-workflow)
13. [Artifact Creation](#13-artifact-creation)
14. [แบบฝึกหัด](#14-แบบฝึกหัด)

---

## 1. Build Automation คืออะไร?

### 1.1 นิยามและความสำคัญ

Build Automation คือการใช้ tools และ scripts เพื่อทำให้กระบวนการ build ซอฟต์แวร์เป็นอัตโนมัติ โดยไม่ต้องทำ manual

**กระบวนการ Build ทั่วไป:**
```
Source Code
    │
    ▼
┌──────────────────────────────────────────────────────┐
│                   Build Pipeline                      │
│                                                       │
│  Compile/Transpile → Bundle → Optimize → Package     │
│                                                       │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐            │
│  │ TypeScript│→│ Bundle  │→ │ Minify  │            │
│  │→JavaScript│  │ (webpack)│  │ (terser)│            │
│  └─────────┘   └─────────┘   └─────────┘            │
└──────────────────────────────────────────────────────┘
    │
    ▼
Deployable Artifact
(dist/, build/, .jar, .war, Docker image)
```

### 1.2 ทำไม Build Automation ถึงสำคัญ?

| ปัญหาไม่มี Automation | วิธีแก้ด้วย Automation |
|---------------------|----------------------|
| "Works on my machine" | Reproducible builds |
| Build steps แตกต่างกันระหว่าง developers | Standardized build process |
| การ deploy ต้องทำเอง (slow, error-prone) | Automated deployment pipeline |
| ไม่รู้ว่า version อะไรใน production | Versioning และ artifacts |
| Build ช้า | Build caching, parallel builds |

### 1.3 Build Tools ยอดนิยม

| Tool | ภาษา/Platform | ลักษณะ |
|------|--------------|--------|
| **Make** | Universal | Classic, declarative |
| **npm scripts** | JavaScript | Simple, built-in |
| **Gradle** | Java/Kotlin | Flexible, modern |
| **Maven** | Java | Convention over config |
| **Bazel** | Multi-language | Google's build system |
| **Task** | Universal | Modern Make alternative |

---

## 2. Makefile: Syntax และ Targets

### 2.1 Makefile Syntax พื้นฐาน

```makefile
# Makefile
# Syntax พื้นฐาน:
# target: dependencies
# 	command (ต้องใช้ TAB ไม่ใช่ spaces!)

# ── Variables ──────────────────────────────────────────────────────────────
APP_NAME    := myapp
VERSION     := $(shell cat VERSION 2>/dev/null || echo "0.0.1")
BUILD_DIR   := build
SRC_DIR     := src
DOCKER_REPO := myregistry.io/myapp

# Conditional variables
GOOS        ?= $(shell go env GOOS)
GOARCH      ?= $(shell go env GOARCH)
ENV         ?= development

# Colors สำหรับ output
RED    := \033[0;31m
GREEN  := \033[0;32m
YELLOW := \033[1;33m
NC     := \033[0m  # No Color

# ── Default target ─────────────────────────────────────────────────────────
.DEFAULT_GOAL := help

# ── PHONY targets (targets ที่ไม่ใช่ file) ────────────────────────────────
.PHONY: help build test lint clean install dev docker-build docker-push

# ── Help ───────────────────────────────────────────────────────────────────
help:  ## แสดง help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "  $(GREEN)%-20s$(NC) %s\n", $$1, $$2}'
	@echo ""
	@echo "Usage: make $(GREEN)<target>$(NC)"

# ── Development ────────────────────────────────────────────────────────────
install:  ## ติดตั้ง dependencies
	npm install

dev:  ## Run development server
	npm run dev

# ── Build ──────────────────────────────────────────────────────────────────
build: lint test  ## Build production bundle (lint + test ก่อน)
	@echo "$(GREEN)Building $(APP_NAME) v$(VERSION)...$(NC)"
	npm run build
	@echo "$(GREEN)Build complete! Output: $(BUILD_DIR)/$(NC)"

build-fast:  ## Build โดยไม่ lint/test (ใช้สำหรับ development เท่านั้น)
	npm run build

# ── Testing ────────────────────────────────────────────────────────────────
test:  ## Run tests
	npm test -- --ci --passWithNoTests

test-watch:  ## Run tests in watch mode
	npm test -- --watch

test-coverage:  ## Run tests with coverage report
	npm test -- --ci --coverage
	@echo "Coverage report: coverage/index.html"

# ── Linting ────────────────────────────────────────────────────────────────
lint:  ## Run linters
	npm run lint
	npm run format:check

lint-fix:  ## Auto-fix lint issues
	npm run lint:fix
	npm run format

# ── Clean ──────────────────────────────────────────────────────────────────
clean:  ## Remove build artifacts
	rm -rf $(BUILD_DIR) coverage node_modules/.cache
	@echo "$(YELLOW)Cleaned build artifacts$(NC)"

clean-all: clean  ## Remove all generated files including node_modules
	rm -rf node_modules
	@echo "$(YELLOW)Cleaned everything$(NC)"

# ── Docker ─────────────────────────────────────────────────────────────────
docker-build:  ## Build Docker image
	docker build \
		--tag $(DOCKER_REPO):$(VERSION) \
		--tag $(DOCKER_REPO):latest \
		--build-arg VERSION=$(VERSION) \
		--build-arg BUILD_DATE=$(shell date -u +%Y-%m-%dT%H:%M:%SZ) \
		.

docker-push: docker-build  ## Push Docker image to registry
	docker push $(DOCKER_REPO):$(VERSION)
	docker push $(DOCKER_REPO):latest

docker-run:  ## Run Docker container locally
	docker run --rm \
		--name $(APP_NAME) \
		--env-file .env \
		-p 3000:3000 \
		$(DOCKER_REPO):latest

# ── Database ───────────────────────────────────────────────────────────────
db-migrate:  ## Run database migrations
	npm run db:migrate

db-rollback:  ## Rollback last migration
	npm run db:rollback

db-seed:  ## Seed database with test data
	npm run db:seed

db-reset: db-rollback db-migrate db-seed  ## Reset database

# ── Release ────────────────────────────────────────────────────────────────
release-patch:  ## Release patch version (x.x.X)
	npm version patch -m "chore: bump version to %s"
	git push --follow-tags

release-minor:  ## Release minor version (x.X.0)
	npm version minor -m "chore: bump version to %s"
	git push --follow-tags

release-major:  ## Release major version (X.0.0)
	npm version major -m "chore: bump version to %s"
	git push --follow-tags

# ── CI/CD ──────────────────────────────────────────────────────────────────
ci: install lint test build  ## Run full CI pipeline locally
	@echo "$(GREEN)CI pipeline completed successfully!$(NC)"

# ── Info ───────────────────────────────────────────────────────────────────
version:  ## Show current version
	@echo "Version: $(VERSION)"
	@echo "App:     $(APP_NAME)"
	@echo "Env:     $(ENV)"

status:  ## Show project status
	@echo "Git branch: $(shell git rev-parse --abbrev-ref HEAD)"
	@echo "Git commit: $(shell git rev-parse --short HEAD)"
	@echo "Node:  $(shell node --version)"
	@echo "npm:   $(shell npm --version)"
```

### 2.2 Advanced Makefile Patterns

```makefile
# ── Pattern Rules ──────────────────────────────────────────────────────────
# Build ทุก .js ไฟล์จาก .ts
$(BUILD_DIR)/%.js: $(SRC_DIR)/%.ts
	tsc $< --outDir $(BUILD_DIR)

# ── Conditional Execution ──────────────────────────────────────────────────
check-env:
ifeq ($(ENV),production)
	@echo "Building for production..."
else
	@echo "Building for $(ENV)..."
endif

# ── Functions ──────────────────────────────────────────────────────────────
define check_tool
	@which $(1) > /dev/null 2>&1 || (echo "Error: $(1) is not installed" && exit 1)
endef

check-tools:  ## Check required tools are installed
	$(call check_tool,node)
	$(call check_tool,npm)
	$(call check_tool,docker)
	@echo "All tools available"

# ── Include ────────────────────────────────────────────────────────────────
# แยก Makefile ตาม environment
-include .make/$(ENV).mk
-include .make/local.mk  # local overrides (ไม่ commit)

# ── Automatic Variables ────────────────────────────────────────────────────
# $@ = target name
# $< = first dependency
# $^ = all dependencies
# $* = stem (เมื่อใช้ pattern rules)

build/%: src/%.ts
	@echo "Building $@ from $<"
	tsc $< --outFile $@
```

### 2.3 Makefile สำหรับ Multi-language Project

```makefile
# Makefile สำหรับ Monorepo
.PHONY: all build test lint clean

# Projects
FRONTEND_DIR := frontend
BACKEND_DIR  := backend
MOBILE_DIR   := mobile

# ── All Projects ───────────────────────────────────────────────────────────
all: install build test  ## Build and test all projects

install:  ## Install dependencies for all projects
	@echo "Installing all dependencies..."
	cd $(FRONTEND_DIR) && npm ci
	cd $(BACKEND_DIR) && pip install -r requirements.txt
	@echo "Done!"

build: build-frontend build-backend  ## Build all projects

build-frontend:  ## Build frontend
	cd $(FRONTEND_DIR) && npm run build

build-backend:  ## Build backend
	cd $(BACKEND_DIR) && python -m build

test: test-frontend test-backend  ## Test all projects

test-frontend:  ## Test frontend
	cd $(FRONTEND_DIR) && npm test -- --ci

test-backend:  ## Test backend
	cd $(BACKEND_DIR) && pytest tests/ -v

lint: lint-frontend lint-backend  ## Lint all projects

lint-frontend:  ## Lint frontend code
	cd $(FRONTEND_DIR) && npm run lint

lint-backend:  ## Lint backend code
	cd $(BACKEND_DIR) && ruff check src/ && black --check src/

clean:  ## Clean all build artifacts
	cd $(FRONTEND_DIR) && rm -rf dist/ node_modules/.cache/
	cd $(BACKEND_DIR) && find . -type d -name __pycache__ -exec rm -rf {} +
	cd $(BACKEND_DIR) && rm -rf dist/ build/ *.egg-info/
```

---

## 3. npm/yarn Scripts

### 3.1 package.json Scripts ครบถ้วน

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "scripts": {
    // ── Development ───────────────────────────────────────────────────
    "dev": "ts-node-dev --respawn --transpile-only src/index.ts",
    "dev:debug": "ts-node-dev --inspect=0.0.0.0:9229 src/index.ts",
    "start": "node dist/index.js",
    
    // ── Build ─────────────────────────────────────────────────────────
    "build": "npm run clean && npm run compile && npm run copy-assets",
    "build:prod": "npm run build -- --NODE_ENV=production",
    "build:analyze": "webpack-bundle-analyzer dist/stats.json",
    "compile": "tsc -p tsconfig.build.json",
    "copy-assets": "copyfiles -u 1 src/**/*.{json,html,css} dist/",
    "clean": "rimraf dist coverage",
    
    // ── Testing ───────────────────────────────────────────────────────
    "test": "jest",
    "test:watch": "jest --watch",
    "test:ci": "jest --ci --coverage --runInBand",
    "test:unit": "jest --testPathPattern=unit",
    "test:integration": "jest --testPathPattern=integration --runInBand",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    
    // ── Coverage ──────────────────────────────────────────────────────
    "coverage": "jest --coverage",
    "coverage:open": "open coverage/index.html",
    
    // ── Linting ───────────────────────────────────────────────────────
    "lint": "eslint src --ext .ts,.tsx",
    "lint:fix": "eslint src --ext .ts,.tsx --fix",
    "lint:report": "eslint src --ext .ts --format html -o reports/eslint.html",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "tsc --noEmit",
    
    // ── Database ──────────────────────────────────────────────────────
    "db:migrate": "knex migrate:latest",
    "db:rollback": "knex migrate:rollback",
    "db:seed": "knex seed:run",
    "db:reset": "npm run db:rollback && npm run db:migrate && npm run db:seed",
    
    // ── Docker ────────────────────────────────────────────────────────
    "docker:build": "docker build -t myapp:latest .",
    "docker:run": "docker run --rm -p 3000:3000 --env-file .env myapp:latest",
    "docker:compose:up": "docker compose up -d",
    "docker:compose:down": "docker compose down",
    
    // ── Release ───────────────────────────────────────────────────────
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major",
    "release:patch": "standard-version --release-as patch",
    "release:dry": "standard-version --dry-run",
    
    // ── CI ────────────────────────────────────────────────────────────
    "ci:lint": "npm run typecheck && npm run lint && npm run format:check",
    "ci:test": "npm run test:ci",
    "ci:build": "npm run build",
    "ci:all": "npm run ci:lint && npm run ci:test && npm run ci:build",
    
    // ── Utilities ─────────────────────────────────────────────────────
    "generate:types": "openapi-typescript api/openapi.yaml -o src/types/api.d.ts",
    "docs": "typedoc src/ --out docs/",
    "outdated": "npm outdated",
    "security": "npm audit"
  },
  
  // Pre/post hooks
  // pre<script> runs before <script>
  // post<script> runs after <script>
}
```

### 3.2 npm Scripts Hooks

```json
{
  "scripts": {
    "prebuild": "echo 'Starting build...' && npm run clean",
    "build": "tsc",
    "postbuild": "echo 'Build complete!' && node scripts/post-build.js",
    
    "pretest": "npm run lint:check",
    "test": "jest",
    "posttest": "npm run coverage:report",
    
    "prepare": "husky install",  // รันหลัง npm install
    "prepublishOnly": "npm run build && npm test"  // รันก่อน npm publish
  }
}
```

### 3.3 Cross-platform Scripts

```json
{
  "scripts": {
    // ❌ ไม่ทำงานบน Windows
    "clean": "rm -rf dist",
    
    // ✓ ทำงานบนทุก OS ด้วย rimraf
    "clean": "rimraf dist",
    
    // ❌ ไม่ทำงานบน Windows
    "set-env": "NODE_ENV=production node server.js",
    
    // ✓ ทำงานบนทุก OS ด้วย cross-env
    "set-env": "cross-env NODE_ENV=production node server.js"
  },
  "devDependencies": {
    "rimraf": "^5.0.0",
    "cross-env": "^7.0.3"
  }
}
```

### 3.4 Turborepo สำหรับ Monorepo

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],  // ^ หมายถึง dependencies ต้อง build ก่อน
      "outputs": ["dist/**", ".next/**"],
      "cache": true
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "lint": {
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    }
  }
}
```

```bash
# Run build สำหรับทุก packages
npx turbo run build

# Run build แค่ที่เปลี่ยน (incremental)
npx turbo run build --filter=[HEAD^1]

# Run parallel tests
npx turbo run test --parallel

# Run สำหรับ specific package
npx turbo run build --filter=@myapp/web
```

---

## 4. Gradle Build Scripts

### 4.1 build.gradle.kts (Kotlin DSL)

```kotlin
// build.gradle.kts
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.21"
    kotlin("plugin.spring") version "1.9.21"
    
    // Code quality
    id("org.sonarqube") version "4.4.1.3373"
    id("jacoco")
    id("com.diffplug.spotless") version "6.23.3"
    
    // Publishing
    id("maven-publish")
}

group = "com.example"
version = System.getenv("VERSION") ?: "0.0.1-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

// ── Dependencies ─────────────────────────────────────────────────────────
repositories {
    mavenCentral()
}

dependencies {
    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    
    // Kotlin
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    
    // Database
    runtimeOnly("org.postgresql:postgresql")
    
    // Test
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:postgresql")
}

// ── Compile ──────────────────────────────────────────────────────────────
tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "21"
    }
}

// ── Test ─────────────────────────────────────────────────────────────────
tasks.withType<Test> {
    useJUnitPlatform()
    
    testLogging {
        events("passed", "skipped", "failed")
        showExceptions = true
        showStandardStreams = false
    }
    
    // JVM args สำหรับ tests
    jvmArgs = listOf(
        "-Xmx512m",
        "-XX:+HeapDumpOnOutOfMemoryError",
    )
    
    // Environment variables สำหรับ tests
    environment(
        "SPRING_PROFILES_ACTIVE", "test",
        "DB_URL", "jdbc:postgresql://localhost:5432/testdb",
    )
}

// ── JaCoCo Coverage ───────────────────────────────────────────────────────
jacoco {
    toolVersion = "0.8.11"
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    
    reports {
        xml.required = true
        html.required = true
        csv.required = false
    }
    
    // Exclude generated code
    classDirectories.setFrom(
        files(classDirectories.files.map {
            fileTree(it).apply {
                exclude("**/generated/**", "**/model/**")
            }
        })
    )
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()
            }
        }
    }
}

// Coverage check รันหลัง test
tasks.check {
    dependsOn(tasks.jacocoTestCoverageVerification)
}

// ── Spotless Code Formatting ──────────────────────────────────────────────
spotless {
    kotlin {
        ktfmt("0.46").kotlinlangStyle()
        trimTrailingWhitespace()
        endWithNewline()
        licenseHeaderFile("${rootDir}/LICENSE_HEADER")
    }
    kotlinGradle {
        ktfmt("0.46")
    }
}

// ── SonarQube ────────────────────────────────────────────────────────────
sonarqube {
    properties {
        property("sonar.projectKey", "myorg_myapp")
        property("sonar.organization", "myorg")
        property("sonar.host.url", "https://sonarcloud.io")
        property("sonar.coverage.jacoco.xmlReportPaths", "$buildDir/reports/jacoco/test/jacocoTestReport.xml")
        property("sonar.exclusions", "**/generated/**,**/model/**")
    }
}

// ── Custom Tasks ─────────────────────────────────────────────────────────
tasks.register("buildInfo") {
    description = "Print build information"
    
    doLast {
        println("=== Build Info ===")
        println("Version: ${project.version}")
        println("Java: ${System.getProperty("java.version")}")
        println("Gradle: ${gradle.gradleVersion}")
        println("Build time: ${java.time.LocalDateTime.now()}")
    }
}

tasks.register("unitTest", Test::class) {
    description = "Runs unit tests only"
    group = "verification"
    
    useJUnitPlatform {
        includeTags("unit")
        excludeTags("integration", "e2e")
    }
}

tasks.register("integrationTest", Test::class) {
    description = "Runs integration tests"
    group = "verification"
    
    useJUnitPlatform {
        includeTags("integration")
    }
    
    shouldRunAfter(tasks.named("test"))
}
```

### 4.2 settings.gradle.kts

```kotlin
// settings.gradle.kts
rootProject.name = "my-spring-app"

// Multi-project setup
include(
    ":app",
    ":core",
    ":api",
    ":infra",
)

// Plugin management
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
    }
}

// Dependency version management
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        mavenCentral()
    }
}
```

### 4.3 gradle.properties

```properties
# gradle.properties
# JVM arguments สำหรับ Gradle daemon
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8 -XX:+HeapDumpOnOutOfMemoryError

# Parallel builds
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configureondemand=true

# Kotlin
kotlin.incremental=true
kotlin.incremental.useClasspathSnapshot=true

# Versions
springBootVersion=3.2.0
kotlinVersion=1.9.21
```

---

## 5. Maven Build Scripts

### 5.1 pom.xml ครบถ้วน

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>${revision}</version>
    <packaging>jar</packaging>
    
    <!-- ── Properties ──────────────────────────────────────────────────── -->
    <properties>
        <revision>1.0.0</revision>
        <java.version>21</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        
        <!-- Dependency versions -->
        <testcontainers.version>1.19.3</testcontainers.version>
        <jacoco.version>0.8.11</jacoco.version>
        
        <!-- Sonar -->
        <sonar.host.url>https://sonarcloud.io</sonar.host.url>
        <sonar.organization>myorg</sonar.organization>
        <sonar.projectKey>myorg_myapp</sonar.projectKey>
    </properties>
    
    <!-- ── Dependencies ────────────────────────────────────────────────── -->
    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${testcontainers.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <!-- ── Build ────────────────────────────────────────────────────────── -->
    <build>
        <plugins>
            <!-- Spring Boot plugin -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                    <!-- Build info สำหรับ actuator -->
                    <buildInfoProperties>
                        <property>
                            <name>build.version</name>
                            <value>${revision}</value>
                        </property>
                    </buildInfoProperties>
                </configuration>
            </plugin>
            
            <!-- JaCoCo Coverage -->
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>${jacoco.version}</version>
                <executions>
                    <execution>
                        <goals><goal>prepare-agent</goal></goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>test</phase>
                        <goals><goal>report</goal></goals>
                    </execution>
                    <execution>
                        <id>check</id>
                        <goals><goal>check</goal></goals>
                        <configuration>
                            <rules>
                                <rule>
                                    <element>BUNDLE</element>
                                    <limits>
                                        <limit>
                                            <counter>LINE</counter>
                                            <value>COVEREDRATIO</value>
                                            <minimum>0.80</minimum>
                                        </limit>
                                    </limits>
                                </rule>
                            </rules>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            
            <!-- Checkstyle -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-checkstyle-plugin</artifactId>
                <version>3.3.1</version>
                <configuration>
                    <configLocation>google_checks.xml</configLocation>
                    <failsOnError>true</failsOnError>
                </configuration>
                <executions>
                    <execution>
                        <goals><goal>check</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    
    <!-- ── Profiles ─────────────────────────────────────────────────────── -->
    <profiles>
        <profile>
            <id>dev</id>
            <activation><activeByDefault>true</activeByDefault></activation>
            <properties>
                <spring.profiles.active>dev</spring.profiles.active>
            </properties>
        </profile>
        
        <profile>
            <id>production</id>
            <properties>
                <spring.profiles.active>production</spring.profiles.active>
            </properties>
            <build>
                <plugins>
                    <!-- สำหรับ production: สร้าง layered jar -->
                    <plugin>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-maven-plugin</artifactId>
                        <configuration>
                            <layers><enabled>true</enabled></layers>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
        
        <profile>
            <id>coverage</id>
            <!-- Profile สำหรับ CI ที่ต้องการ coverage report -->
        </profile>
    </profiles>
    
</project>
```

---

## 6. Build Optimization

### 6.1 Webpack Optimization

```javascript
// webpack.config.js
const path = require('path');
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = (env, argv) => {
  const isProduction = argv.mode === 'production';
  
  return {
    mode: argv.mode || 'development',
    
    entry: {
      main: './src/index.ts',
      // Code splitting: แยก vendor bundle
      vendor: ['react', 'react-dom', 'lodash'],
    },
    
    output: {
      path: path.resolve(__dirname, 'dist'),
      filename: isProduction ? '[name].[contenthash].js' : '[name].js',
      chunkFilename: isProduction ? '[name].[contenthash].chunk.js' : '[name].chunk.js',
      clean: true,
    },
    
    optimization: {
      // ── Minimization ────────────────────────────────────────────────
      minimize: isProduction,
      minimizer: [
        new TerserPlugin({
          terserOptions: {
            compress: {
              drop_console: isProduction,  // Remove console.log in production
            },
          },
          parallel: true,  // ใช้ parallel processing
        }),
        new CssMinimizerPlugin(),
      ],
      
      // ── Code Splitting ───────────────────────────────────────────────
      splitChunks: {
        chunks: 'all',
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/]/,
            name: 'vendors',
            chunks: 'all',
          },
          common: {
            name: 'common',
            minChunks: 2,
            chunks: 'all',
          },
        },
      },
      
      // Module IDs ที่ stable (ไม่เปลี่ยนเมื่อ build ใหม่)
      moduleIds: 'deterministic',
      runtimeChunk: 'single',
    },
    
    // ── Caching ─────────────────────────────────────────────────────────
    cache: {
      type: 'filesystem',
      buildDependencies: {
        config: [__filename],
      },
    },
    
    // ── Performance ─────────────────────────────────────────────────────
    performance: {
      hints: isProduction ? 'error' : 'warning',
      maxEntrypointSize: 512000,   // 500KB
      maxAssetSize: 512000,
    },
    
    plugins: [
      // Bundle analyzer (เฉพาะเมื่อ ANALYZE=true)
      ...(process.env.ANALYZE === 'true' ? [new BundleAnalyzerPlugin()] : []),
    ],
  };
};
```

### 6.2 Vite Build Optimization

```javascript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    // Bundle visualizer
    visualizer({ filename: 'dist/stats.html', open: false }),
  ],
  
  build: {
    // Target browsers
    target: 'esnext',
    
    // Output directory
    outDir: 'dist',
    
    // Minification
    minify: 'esbuild',  // หรือ 'terser' สำหรับ better compression
    
    // Code splitting
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['react-router-dom'],
          utils: ['lodash', 'date-fns'],
        },
      },
    },
    
    // Report compressed size
    reportCompressedSize: true,
    
    // Chunk size warning limit
    chunkSizeWarningLimit: 500, // KB
  },
  
  // Build caching
  cacheDir: 'node_modules/.vite',
});
```

---

## 7. Incremental Builds

### 7.1 TypeScript Incremental Compilation

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    
    // ── Incremental Compilation ──────────────────────────────────────
    "incremental": true,
    "tsBuildInfoFile": ".tsbuildinfo",  // Cache file
    
    // ── Type Checking ────────────────────────────────────────────────
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    
    // ── Paths ────────────────────────────────────────────────────────
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

```bash
# First build: compile everything
tsc

# Second build: เฉพาะไฟล์ที่เปลี่ยน (ใช้ .tsbuildinfo)
tsc --incremental

# Faster: เฉพาะ type check, ไม่ emit
tsc --noEmit  
```

### 7.2 Makefile Incremental Builds

```makefile
# ── Incremental builds ด้วย Makefile ───────────────────────────────────────
BUILD_DIR := dist
SRC_DIR   := src

# สร้าง list ของ target files จาก source files
SOURCES := $(shell find $(SRC_DIR) -name "*.ts")
TARGETS := $(patsubst $(SRC_DIR)/%.ts, $(BUILD_DIR)/%.js, $(SOURCES))

# Build เฉพาะไฟล์ที่ .ts ใหม่กว่า .js
$(BUILD_DIR)/%.js: $(SRC_DIR)/%.ts
	@mkdir -p $(dir $@)
	tsc $< --outDir $(dir $@)
	@echo "Compiled: $<"

build: $(TARGETS)  ## Build only changed files
	@echo "Build complete"

# จะ build เฉพาะไฟล์ที่เปลี่ยนแปลง
# make จะตรวจสอบ timestamp ของ .ts vs .js
```

---

## 8. Build Caching ใน CI

### 8.1 npm/yarn Caching

```yaml
# .github/workflows/build-with-cache.yml
name: Build with Cache

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # ── Node.js Cache ─────────────────────────────────────────────────────
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'  # built-in npm cache
      
      # ── Additional Build Cache ────────────────────────────────────────────
      
      # TypeScript cache
      - name: Cache TypeScript build info
        uses: actions/cache@v4
        with:
          path: .tsbuildinfo
          key: tsbuildinfo-${{ runner.os }}-${{ hashFiles('tsconfig.json') }}-${{ hashFiles('src/**/*.ts') }}
          restore-keys: |
            tsbuildinfo-${{ runner.os }}-${{ hashFiles('tsconfig.json') }}-
            tsbuildinfo-${{ runner.os }}-
      
      # Webpack/Vite cache
      - name: Cache build cache
        uses: actions/cache@v4
        with:
          path: |
            node_modules/.cache
            node_modules/.vite
          key: build-cache-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('vite.config.ts', 'webpack.config.js') }}
          restore-keys: |
            build-cache-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-
      
      - run: npm ci
      - run: npm run build
```

### 8.2 Python/pip Caching

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'  # built-in pip cache
      
      # Python wheel cache
      - name: Cache pip wheels
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ runner.os }}-${{ hashFiles('requirements*.txt') }}
          restore-keys: |
            pip-${{ runner.os }}-
      
      # mypy cache
      - name: Cache mypy
        uses: actions/cache@v4
        with:
          path: .mypy_cache
          key: mypy-${{ runner.os }}-${{ hashFiles('mypy.ini', '**/*.py') }}
          restore-keys: mypy-${{ runner.os }}-
      
      - run: pip install -r requirements.txt
      - run: python -m build
```

### 8.3 Gradle/Maven Caching

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'  # built-in Gradle cache
          # หรือ
          cache: 'maven'   # built-in Maven cache
      
      # Gradle build cache (local)
      - name: Cache Gradle wrapper
        uses: actions/cache@v4
        with:
          path: ~/.gradle/wrapper
          key: gradle-wrapper-${{ hashFiles('**/gradle-wrapper.properties') }}
      
      - name: Cache Gradle caches
        uses: actions/cache@v4
        with:
          path: ~/.gradle/caches
          key: gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            gradle-${{ runner.os }}-
      
      - run: ./gradlew build
```

---

## 9. Multi-stage Builds

### 9.1 Dockerfile Multi-stage Build (Node.js)

```dockerfile
# ── Stage 1: Dependencies ─────────────────────────────────────────────────
FROM node:20-alpine AS deps
WORKDIR /app

# Copy package files เฉพาะ (cache layer)
COPY package.json package-lock.json ./
RUN npm ci --only=production

# ── Stage 2: Builder ──────────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app

# Install dev dependencies สำหรับ build
COPY package.json package-lock.json ./
RUN npm ci

# Copy source
COPY tsconfig.json ./
COPY src/ ./src/

# Build
ENV NODE_ENV=production
RUN npm run build

# ── Stage 3: Test ─────────────────────────────────────────────────────────
FROM builder AS test
RUN npm test -- --ci --passWithNoTests

# ── Stage 4: Production ───────────────────────────────────────────────────
FROM node:20-alpine AS production

# Security: ใช้ non-root user
RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodeuser

WORKDIR /app

# Copy production dependencies จาก deps stage
COPY --from=deps --chown=nodeuser:nodejs /app/node_modules ./node_modules

# Copy built code จาก builder stage
COPY --from=builder --chown=nodeuser:nodejs /app/dist ./dist

# Copy config files
COPY --chown=nodeuser:nodejs package.json ./

USER nodeuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD wget -qO- http://localhost:3000/health || exit 1

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### 9.2 Dockerfile สำหรับ Python

```dockerfile
# ── Stage 1: Builder ──────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

# Install build tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Install dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ── Stage 2: Test ─────────────────────────────────────────────────────────
FROM builder AS test
COPY requirements-dev.txt .
RUN pip install --user --no-cache-dir -r requirements-dev.txt
COPY . .
RUN pytest tests/ -v --tb=short

# ── Stage 3: Production ───────────────────────────────────────────────────
FROM python:3.12-slim AS production

# Security: non-root user
RUN groupadd --gid 1001 appgroup && \
    useradd --uid 1001 --gid appgroup --shell /bin/bash --create-home appuser

WORKDIR /app

# Copy installed packages
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appgroup src/ ./src/
COPY --chown=appuser:appgroup main.py .

USER appuser

# Make sure scripts in .local are usable
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

EXPOSE 8000
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 9.3 Docker Build Optimization

```yaml
# .github/workflows/docker-build.yml
name: Docker Build

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  docker:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      # ── Docker Setup ──────────────────────────────────────────────────────
      - name: Set up QEMU (cross-platform)
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      # ── Login ─────────────────────────────────────────────────────────────
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      # ── Meta ──────────────────────────────────────────────────────────────
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: myorg/myapp
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
            type=raw,value=latest,enable={{is_default_branch}}
      
      # ── Build & Push ──────────────────────────────────────────────────────
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          
          # Layer caching
          cache-from: type=gha
          cache-to: type=gha,mode=max
          
          # Build args
          build-args: |
            VERSION=${{ steps.meta.outputs.version }}
            BUILD_DATE=${{ github.event.repository.updated_at }}
            GIT_COMMIT=${{ github.sha }}
```

---

## 10. Versioning Strategy (SemVer)

### 10.1 Semantic Versioning

```
MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]
  │      │      │
  │      │      └── Bug fixes (backward compatible)
  │      └───────── New features (backward compatible)
  └──────────────── Breaking changes
  
ตัวอย่าง:
  1.2.3          → Stable release
  1.2.3-alpha.1  → Alpha pre-release
  1.2.3-beta.2   → Beta pre-release
  1.2.3-rc.1     → Release candidate
  1.2.3+20240115 → Build metadata
```

### 10.2 Version Management ด้วย npm

```bash
# ดู version ปัจจุบัน
npm version

# Bump versions
npm version patch    # 1.0.0 → 1.0.1
npm version minor    # 1.0.0 → 1.1.0
npm version major    # 1.0.0 → 2.0.0
npm version prepatch # 1.0.0 → 1.0.1-0
npm version prerelease # 1.0.0 → 1.0.1-0

# Bump และ commit + tag
npm version patch -m "chore: release v%s"
git push --follow-tags
```

### 10.3 Conventional Commits

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]

ตัวอย่าง:
feat: add Thai language support
fix(auth): resolve token expiration bug
docs: update API documentation
style: format code with prettier
refactor: extract payment logic to service
test: add unit tests for OrderService
chore: update dependencies
ci: add docker build workflow
perf: optimize database queries
break!: remove deprecated API endpoints
```

**เชื่อมโยง Conventional Commits กับ SemVer:**
- `fix:` → PATCH bump
- `feat:` → MINOR bump
- `feat!:` หรือ `BREAKING CHANGE:` → MAJOR bump

### 10.4 semantic-release

```bash
npm install --save-dev semantic-release \
  @semantic-release/changelog \
  @semantic-release/git \
  @semantic-release/github \
  @semantic-release/npm \
  @semantic-release/commit-analyzer \
  @semantic-release/release-notes-generator
```

**.releaserc.js:**
```javascript
// .releaserc.js
module.exports = {
  branches: [
    'main',
    { name: 'develop', prerelease: 'beta' },
    { name: 'staging', prerelease: 'rc' },
  ],
  
  plugins: [
    // 1. วิเคราะห์ commits เพื่อหา version bump
    [
      '@semantic-release/commit-analyzer',
      {
        preset: 'conventionalcommits',
        releaseRules: [
          { type: 'feat', release: 'minor' },
          { type: 'fix', release: 'patch' },
          { type: 'perf', release: 'patch' },
          { type: 'docs', release: 'patch' },
          { breaking: true, release: 'major' },
        ],
      },
    ],
    
    // 2. Generate release notes
    ['@semantic-release/release-notes-generator', {
      preset: 'conventionalcommits',
      presetConfig: {
        types: [
          { type: 'feat', section: 'Features' },
          { type: 'fix', section: 'Bug Fixes' },
          { type: 'perf', section: 'Performance' },
          { type: 'docs', section: 'Documentation' },
          { type: 'refactor', section: 'Refactoring' },
        ],
      },
    }],
    
    // 3. Update CHANGELOG.md
    ['@semantic-release/changelog', {
      changelogFile: 'CHANGELOG.md',
    }],
    
    // 4. Update package.json version
    ['@semantic-release/npm', {
      npmPublish: false,  // ไม่ publish ไป npm registry (แค่ bump version)
    }],
    
    // 5. Commit CHANGELOG.md และ package.json ที่อัพเดท
    ['@semantic-release/git', {
      assets: ['CHANGELOG.md', 'package.json'],
      message: 'chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}',
    }],
    
    // 6. สร้าง GitHub Release
    ['@semantic-release/github', {
      assets: [
        { path: 'dist/*.tar.gz', label: 'Compiled binaries' },
      ],
    }],
  ],
};
```

---

## 11. Changelog Generation

### 11.1 conventional-changelog

```bash
npm install --save-dev conventional-changelog-cli
```

```json
// package.json
{
  "scripts": {
    "changelog": "conventional-changelog -p angular -i CHANGELOG.md -s -r 0",
    "changelog:patch": "conventional-changelog -p angular -i CHANGELOG.md -s",
    "version": "conventional-changelog -p angular -i CHANGELOG.md -s && git add CHANGELOG.md"
  }
}
```

### 11.2 ตัวอย่าง CHANGELOG.md

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Thai language support for UI

## [2.1.0] - 2024-01-15

### Added
- ระบบส่ง OTP ผ่าน SMS
- รองรับการชำระเงินด้วย PromptPay
- Dark mode support

### Changed
- ปรับปรุง performance ของ search feature

### Fixed
- แก้ไข bug การคำนวณ VAT สำหรับสินค้านำเข้า
- แก้ไข session timeout ที่ไม่ถูกต้อง

### Security
- อัพเดท dependencies ที่มี vulnerabilities

## [2.0.0] - 2023-12-01

### Breaking Changes
- API v1 endpoints ถูกลบออก ใช้ v2 แทน
- เปลี่ยน authentication จาก session เป็น JWT

### Added
- REST API v2 ที่ออกแบบใหม่
- GraphQL endpoint
- Webhook support

[Unreleased]: https://github.com/myorg/myapp/compare/v2.1.0...HEAD
[2.1.0]: https://github.com/myorg/myapp/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/myorg/myapp/releases/tag/v2.0.0
```

---

## 12. GitHub Actions Build Workflow

### 12.1 Complete Build Workflow

```yaml
# .github/workflows/build.yml
name: Build & Release

on:
  push:
    branches: [ main, develop ]
    tags: [ 'v*.*.*' ]
  pull_request:
    branches: [ main ]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── Quality Checks ────────────────────────────────────────────────────────
  
  quality:
    name: Quality Checks
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - name: Type check
        run: npm run typecheck
      
      - name: Lint
        run: npm run lint
      
      - name: Format check
        run: npm run format:check
  
  # ── Tests ─────────────────────────────────────────────────────────────────
  
  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: quality
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - uses: actions/cache@v4
        with:
          path: node_modules/.cache/jest
          key: jest-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
      
      - run: npm ci
      
      - name: Run tests
        run: npm run test:ci
      
      - uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: coverage/lcov.info
  
  # ── Build ─────────────────────────────────────────────────────────────────
  
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    
    outputs:
      version: ${{ steps.version.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      # Build cache
      - uses: actions/cache@v4
        with:
          path: |
            node_modules/.cache
            .tsbuildinfo
          key: build-${{ runner.os }}-${{ hashFiles('**/package-lock.json', 'tsconfig.json') }}-${{ github.sha }}
          restore-keys: |
            build-${{ runner.os }}-${{ hashFiles('**/package-lock.json', 'tsconfig.json') }}-
      
      - run: npm ci
      
      - name: Get version
        id: version
        run: echo "version=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT
      
      - name: Build
        run: npm run build
        env:
          NODE_ENV: production
          VERSION: ${{ steps.version.outputs.version }}
          GIT_COMMIT: ${{ github.sha }}
          BUILD_DATE: ${{ github.event.repository.updated_at }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
          retention-days: 7
  
  # ── Docker Build ──────────────────────────────────────────────────────────
  
  docker:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push'
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
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
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          target: production  # Multi-stage build target
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ needs.build.outputs.version }}
            GIT_COMMIT=${{ github.sha }}
  
  # ── Release ───────────────────────────────────────────────────────────────
  
  release:
    name: Create Release
    runs-on: ubuntu-latest
    needs: [build, docker]
    if: startsWith(github.ref, 'refs/tags/v')
    
    permissions:
      contents: write
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: actions/download-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
      
      - name: Create release archive
        run: |
          tar -czf myapp-${{ needs.build.outputs.version }}.tar.gz dist/
      
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          files: myapp-${{ needs.build.outputs.version }}.tar.gz
          generate_release_notes: true
          draft: false
          prerelease: ${{ contains(github.ref, '-rc') || contains(github.ref, '-beta') || contains(github.ref, '-alpha') }}
```

---

## 13. Artifact Creation

### 13.1 สร้าง NPM Package

```bash
# โครงสร้างสำหรับ library
my-lib/
├── src/
│   ├── index.ts      ← entry point
│   └── utils/
├── dist/             ← compiled output
├── package.json
├── tsconfig.json
└── README.md
```

**package.json สำหรับ library:**
```json
{
  "name": "@myorg/my-lib",
  "version": "1.0.0",
  "description": "My reusable library",
  
  "main": "./dist/cjs/index.js",     // CommonJS
  "module": "./dist/esm/index.js",   // ES Modules
  "types": "./dist/types/index.d.ts", // TypeScript types
  
  "exports": {
    ".": {
      "require": "./dist/cjs/index.js",
      "import": "./dist/esm/index.js",
      "types": "./dist/types/index.d.ts"
    },
    "./utils": {
      "require": "./dist/cjs/utils/index.js",
      "import": "./dist/esm/utils/index.js"
    }
  },
  
  "files": [
    "dist/",
    "README.md",
    "CHANGELOG.md"
  ],
  
  "scripts": {
    "build:cjs": "tsc -p tsconfig.cjs.json",
    "build:esm": "tsc -p tsconfig.esm.json",
    "build:types": "tsc -p tsconfig.types.json",
    "build": "npm run clean && npm run build:cjs && npm run build:esm && npm run build:types",
    "prepublishOnly": "npm run build && npm test"
  },
  
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org"
  }
}
```

### 13.2 สร้าง Python Package

**pyproject.toml:**
```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "my-package"
version = "1.0.0"
description = "My Python package"
readme = "README.md"
license = {text = "MIT"}
requires-python = ">=3.10"

dependencies = [
    "requests>=2.28.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "black",
    "ruff",
]

[project.scripts]
# CLI commands
my-tool = "my_package.cli:main"

[tool.setuptools.packages.find]
where = ["src"]
```

```bash
# Build package
python -m build

# Outputs:
# dist/my-package-1.0.0.tar.gz
# dist/my_package-1.0.0-py3-none-any.whl

# Publish ไป PyPI
python -m twine upload dist/*

# หรือ test PyPI ก่อน
python -m twine upload --repository testpypi dist/*
```

### 13.3 GitHub Actions: Publish Package

```yaml
# .github/workflows/publish.yml
name: Publish Package

on:
  release:
    types: [published]

jobs:
  publish-npm:
    name: Publish to npm
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
      
      - run: npm ci
      - run: npm run build
      - run: npm test
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
  
  publish-pypi:
    name: Publish to PyPI
    runs-on: ubuntu-latest
    
    permissions:
      id-token: write  # สำหรับ trusted publishing
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      
      - run: pip install build
      - run: python -m build
      
      - name: Publish to PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        # ใช้ OIDC trusted publishing (ไม่ต้องใช้ token!)
```

---

## 14. แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Makefile ครบถ้วน

**โจทย์:** สร้าง Makefile สำหรับ project ที่มีทั้ง Node.js frontend และ Python backend

**ข้อกำหนด:**
1. มี `help` target ที่แสดงคำอธิบาย
2. `install` ติดตั้ง dependencies ทั้ง frontend และ backend
3. `build` สร้าง production builds
4. `test` รัน tests ทั้งสอง
5. `lint` ตรวจสอบ code quality
6. `clean` ลบ build artifacts
7. `docker-build` สร้าง Docker image
8. `ci` รัน full CI pipeline

### แบบฝึกหัดที่ 2: GitHub Actions Build Pipeline

**โจทย์:** สร้าง GitHub Actions workflow ที่:
1. Triggered เมื่อ push ไป main หรือเปิด PR
2. รัน lint, test, build แบบ parallel
3. Build Docker image เฉพาะ main branch
4. Cache dependencies และ build cache
5. Publish release เมื่อ push tag v*
6. Upload build artifacts สำหรับ 7 วัน

### แบบฝึกหัดที่ 3: SemVer และ Changelog

**โจทย์:**
1. ตั้งค่า Conventional Commits ในโปรเจค
2. ตั้งค่า commitlint + Husky
3. ตั้งค่า semantic-release
4. เขียน CHANGELOG.md
5. Bump version และ generate changelog อัตโนมัติ

**ตรวจสอบความเข้าใจ:**
- `git commit -m "feat: add payment via QR code"` → bump version อย่างไร?
- `git commit -m "fix: resolve login timeout bug"` → bump version อย่างไร?
- `git commit -m "feat!: change API response format"` → bump version อย่างไร?

**เฉลย:**
- `feat:` → MINOR bump (1.0.0 → 1.1.0)
- `fix:` → PATCH bump (1.0.0 → 1.0.1)
- `feat!:` (breaking) → MAJOR bump (1.0.0 → 2.0.0)

### แบบฝึกหัดที่ 4: Build Optimization

**โจทย์:** เพิ่ม build caching และ optimization สำหรับ GitHub Actions

**ปัจจุบัน:** build ใช้เวลา 8 นาที (ไม่มี cache)
**เป้าหมาย:** ลดเหลือ < 3 นาที ด้วยการ:
1. Cache npm dependencies
2. Cache build artifacts (TypeScript .tsbuildinfo)
3. Cache Docker layers
4. ใช้ incremental TypeScript compilation

---

## สรุป Part 10

ในบทนี้เราได้เรียนรู้:

1. **Build Automation** — ความสำคัญ, tools ที่มี
2. **Makefile** — syntax, patterns, incremental builds, multi-language
3. **npm/yarn scripts** — scripts ครบถ้วน, hooks, cross-platform
4. **Gradle** — build.gradle.kts, plugins, custom tasks
5. **Maven** — pom.xml, profiles, plugins
6. **Build Optimization** — Webpack, Vite, TypeScript incremental
7. **Build Caching** — GitHub Actions caching strategies
8. **Multi-stage Docker** — Node.js, Python optimized images
9. **SemVer** — Conventional Commits, semantic-release
10. **Changelog** — conventional-changelog, CHANGELOG.md format
11. **GitHub Actions** — Complete build pipeline
12. **Artifacts** — npm package, Python package, publishing

**การเรียนรู้ต่อไป:**

Part 11-20 จะครอบคลุม:
- Part 11: Docker & Containerization
- Part 12: Docker Compose
- Part 13: Kubernetes Basics
- Part 14: Helm Charts
- Part 15: GitOps with ArgoCD
- Part 16: Monitoring & Observability
- Part 17: Security in CI/CD
- Part 18: Performance Testing
- Part 19: Infrastructure as Code
- Part 20: Advanced CI/CD Patterns

---

*ปรับปรุงล่าสุด: 2024 | หลักสูตร CI/CD สำหรับนักพัฒนา*

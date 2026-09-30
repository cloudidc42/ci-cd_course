# Part 06: สร้าง CI Pipeline แรก

## บทนำ

ในบทนี้เราจะสร้าง CI (Continuous Integration) Pipeline จริงๆ โดยเริ่มตั้งแต่การสร้าง Node.js application ตัวอย่าง ไปจนถึงการเขียน GitHub Actions workflow ที่ครบครัน พร้อม best practices ที่ใช้ได้จริงในการทำงาน

**สิ่งที่จะได้เรียนรู้:**
- สร้าง Node.js app ตัวอย่างสำหรับ CI
- เขียน GitHub Actions CI workflow
- Checkout code, setup runtime, install dependencies
- รัน linter, tests, และ build
- Upload artifacts
- Branch-based triggers
- PR checks และ status badges
- CI สำหรับ Python, Java, และ Go
- Best practices สำหรับ CI pipelines

---

## 6.1 สร้าง Node.js Application สำหรับ CI

### โครงสร้าง Project

```
ci-demo-node/
├── src/
│   ├── app.js           ← Express application
│   ├── calculator.js    ← Module ที่จะ test
│   └── utils.js         ← Utility functions
├── tests/
│   ├── calculator.test.js
│   └── utils.test.js
├── .github/
│   └── workflows/
│       └── ci.yml       ← CI workflow
├── .eslintrc.js
├── .prettierrc
├── package.json
├── package-lock.json
└── README.md
```

### ขั้นตอนที่ 1: สร้าง Project

```bash
# สร้าง directory
mkdir ci-demo-node && cd ci-demo-node

# Initialize npm project
npm init -y

# ติดตั้ง dependencies
npm install express

# ติดตั้ง dev dependencies
npm install --save-dev \
  jest \
  supertest \
  eslint \
  eslint-config-prettier \
  prettier \
  @types/jest

# สร้าง directories
mkdir -p src tests .github/workflows
```

### ขั้นตอนที่ 2: สร้าง Source Files

**src/calculator.js**
```javascript
// src/calculator.js
'use strict';

/**
 * Calculator module สำหรับ demo CI
 */

/**
 * บวกตัวเลข 2 จำนวน
 * @param {number} a
 * @param {number} b
 * @returns {number}
 */
function add(a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw new TypeError('Arguments must be numbers');
  }
  return a + b;
}

/**
 * ลบตัวเลข 2 จำนวน
 * @param {number} a
 * @param {number} b
 * @returns {number}
 */
function subtract(a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw new TypeError('Arguments must be numbers');
  }
  return a - b;
}

/**
 * คูณตัวเลข 2 จำนวน
 * @param {number} a
 * @param {number} b
 * @returns {number}
 */
function multiply(a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw new TypeError('Arguments must be numbers');
  }
  return a * b;
}

/**
 * หารตัวเลข 2 จำนวน
 * @param {number} a
 * @param {number} b
 * @returns {number}
 */
function divide(a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw new TypeError('Arguments must be numbers');
  }
  if (b === 0) {
    throw new Error('Division by zero is not allowed');
  }
  return a / b;
}

module.exports = { add, subtract, multiply, divide };
```

**src/utils.js**
```javascript
// src/utils.js
'use strict';

/**
 * ตรวจสอบว่าเป็น palindrome หรือไม่
 * @param {string} str
 * @returns {boolean}
 */
function isPalindrome(str) {
  if (typeof str !== 'string') {
    throw new TypeError('Argument must be a string');
  }
  const cleaned = str.toLowerCase().replace(/[^a-z0-9]/g, '');
  return cleaned === cleaned.split('').reverse().join('');
}

/**
 * หาค่า Fibonacci
 * @param {number} n
 * @returns {number}
 */
function fibonacci(n) {
  if (typeof n !== 'number' || n < 0) {
    throw new TypeError('Argument must be a non-negative number');
  }
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

/**
 * Format ตัวเลขเป็น currency
 * @param {number} amount
 * @param {string} currency
 * @returns {string}
 */
function formatCurrency(amount, currency = 'THB') {
  return new Intl.NumberFormat('th-TH', {
    style: 'currency',
    currency,
  }).format(amount);
}

module.exports = { isPalindrome, fibonacci, formatCurrency };
```

**src/app.js**
```javascript
// src/app.js
'use strict';

const express = require('express');
const { add, subtract, multiply, divide } = require('./calculator');

const app = express();
app.use(express.json());

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'OK',
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version || '1.0.0',
  });
});

// Calculator endpoints
app.post('/calculate', (req, res) => {
  const { operation, a, b } = req.body;

  try {
    let result;
    switch (operation) {
      case 'add':
        result = add(a, b);
        break;
      case 'subtract':
        result = subtract(a, b);
        break;
      case 'multiply':
        result = multiply(a, b);
        break;
      case 'divide':
        result = divide(a, b);
        break;
      default:
        return res.status(400).json({ error: 'Invalid operation' });
    }

    res.json({ result, operation, a, b });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = app;
```

### ขั้นตอนที่ 3: สร้าง Test Files

**tests/calculator.test.js**
```javascript
// tests/calculator.test.js
'use strict';

const { add, subtract, multiply, divide } = require('../src/calculator');

describe('Calculator', () => {
  describe('add()', () => {
    test('บวก 2 + 3 = 5', () => {
      expect(add(2, 3)).toBe(5);
    });

    test('บวก 0 + 0 = 0', () => {
      expect(add(0, 0)).toBe(0);
    });

    test('บวก -1 + 1 = 0', () => {
      expect(add(-1, 1)).toBe(0);
    });

    test('บวก ตัวเลขทศนิยม', () => {
      expect(add(0.1, 0.2)).toBeCloseTo(0.3);
    });

    test('throw error เมื่อ argument ไม่ใช่ number', () => {
      expect(() => add('2', 3)).toThrow(TypeError);
      expect(() => add(2, '3')).toThrow(TypeError);
    });
  });

  describe('subtract()', () => {
    test('ลบ 5 - 3 = 2', () => {
      expect(subtract(5, 3)).toBe(2);
    });

    test('ลบ 0 - 5 = -5', () => {
      expect(subtract(0, 5)).toBe(-5);
    });
  });

  describe('multiply()', () => {
    test('คูณ 3 * 4 = 12', () => {
      expect(multiply(3, 4)).toBe(12);
    });

    test('คูณ -2 * 3 = -6', () => {
      expect(multiply(-2, 3)).toBe(-6);
    });

    test('คูณ 0 * 5 = 0', () => {
      expect(multiply(0, 5)).toBe(0);
    });
  });

  describe('divide()', () => {
    test('หาร 10 / 2 = 5', () => {
      expect(divide(10, 2)).toBe(5);
    });

    test('หาร 7 / 2 = 3.5', () => {
      expect(divide(7, 2)).toBe(3.5);
    });

    test('throw error เมื่อหารด้วย 0', () => {
      expect(() => divide(10, 0)).toThrow('Division by zero');
    });
  });
});
```

**tests/utils.test.js**
```javascript
// tests/utils.test.js
'use strict';

const { isPalindrome, fibonacci, formatCurrency } = require('../src/utils');

describe('Utils', () => {
  describe('isPalindrome()', () => {
    test('"racecar" เป็น palindrome', () => {
      expect(isPalindrome('racecar')).toBe(true);
    });

    test('"hello" ไม่ใช่ palindrome', () => {
      expect(isPalindrome('hello')).toBe(false);
    });

    test('"A man a plan a canal Panama" เป็น palindrome', () => {
      expect(isPalindrome('A man a plan a canal Panama')).toBe(true);
    });

    test('throw error เมื่อ argument ไม่ใช่ string', () => {
      expect(() => isPalindrome(123)).toThrow(TypeError);
    });
  });

  describe('fibonacci()', () => {
    test('fibonacci(0) = 0', () => {
      expect(fibonacci(0)).toBe(0);
    });

    test('fibonacci(1) = 1', () => {
      expect(fibonacci(1)).toBe(1);
    });

    test('fibonacci(10) = 55', () => {
      expect(fibonacci(10)).toBe(55);
    });
  });

  describe('formatCurrency()', () => {
    test('format 1000 เป็น THB', () => {
      const result = formatCurrency(1000);
      expect(result).toContain('1,000');
    });
  });
});
```

### ขั้นตอนที่ 4: ตั้งค่า package.json

```json
{
  "name": "ci-demo-node",
  "version": "1.0.0",
  "description": "CI Demo Node.js Application",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch",
    "lint": "eslint src tests",
    "lint:fix": "eslint src tests --fix",
    "format": "prettier --write 'src/**/*.js' 'tests/**/*.js'",
    "format:check": "prettier --check 'src/**/*.js' 'tests/**/*.js'",
    "build": "echo 'No build step for this project'"
  },
  "jest": {
    "testEnvironment": "node",
    "coverageDirectory": "coverage",
    "collectCoverageFrom": [
      "src/**/*.js",
      "!src/app.js"
    ],
    "coverageThresholds": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### ขั้นตอนที่ 5: ESLint Configuration

```javascript
// .eslintrc.js
module.exports = {
  env: {
    node: true,
    es2021: true,
    jest: true,
  },
  extends: ['eslint:recommended', 'prettier'],
  parserOptions: {
    ecmaVersion: 'latest',
  },
  rules: {
    'no-console': 'warn',
    'no-unused-vars': 'error',
    'no-var': 'error',
    'prefer-const': 'error',
    'eqeqeq': ['error', 'always'],
    'curly': 'error',
  },
};
```

### ขั้นตอนที่ 6: Prettier Configuration

```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2,
  "arrowParens": "always"
}
```

---

## 6.2 เขียน GitHub Actions CI Workflow

### Workflow พื้นฐาน

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Check Prettier formatting
        run: npm run format:check

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests with coverage
        run: npm run test:coverage

      - name: Upload coverage reports
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build project
        run: npm run build
```

### CI Workflow แบบสมบูรณ์

```yaml
# .github/workflows/ci-complete.yml
name: Complete CI Pipeline

on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'
      - 'hotfix/**'
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches:
      - main
      - develop
    types:
      - opened
      - synchronize
      - reopened

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  NODE_VERSION: '20'
  NODE_ENV: test

jobs:
  # =============================
  # Job 1: Code Quality
  # =============================
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0    # Full history สำหรับ analysis

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Check Prettier
        run: npm run format:check

      - name: Check for security vulnerabilities
        run: npm audit --audit-level=high

  # =============================
  # Job 2: Tests
  # =============================
  test:
    name: Tests (Node.js ${{ matrix.node-version }})
    runs-on: ${{ matrix.os }}
    timeout-minutes: 15

    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest]
        node-version: [18, 20, 21]
        include:
          - os: windows-latest
            node-version: 20
          - os: macos-latest
            node-version: 20

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm test

      - name: Run tests with coverage (Ubuntu + Node 20 only)
        if: matrix.os == 'ubuntu-latest' && matrix.node-version == 20
        run: npm run test:coverage

      - name: Upload coverage report
        if: matrix.os == 'ubuntu-latest' && matrix.node-version == 20
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 30

  # =============================
  # Job 3: Build
  # =============================
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [quality, test]
    timeout-minutes: 10

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build project
        run: npm run build

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: |
            dist/
            !dist/**/*.map
          retention-days: 7

      - name: Set build output
        id: build-info
        run: |
          echo "build-date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_OUTPUT
          echo "build-sha=${{ github.sha }}" >> $GITHUB_OUTPUT

  # =============================
  # Job 4: PR Comment
  # =============================
  pr-comment:
    name: PR Summary
    runs-on: ubuntu-latest
    needs: [quality, test, build]
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: write

    steps:
      - name: Comment PR
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const message = `## CI Pipeline Results ✅

            | Job | Status |
            |-----|--------|
            | Code Quality | ✅ Passed |
            | Tests | ✅ Passed |
            | Build | ✅ Passed |

            **Commit:** \`${{ github.sha }}\`
            **Branch:** \`${{ github.head_ref }}\`
            **Author:** @${{ github.actor }}
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: message
            });
```

---

## 6.3 Checkout Code

### actions/checkout

```yaml
steps:
  # Simple checkout
  - uses: actions/checkout@v4

  # Full clone (ไม่ shallow)
  - uses: actions/checkout@v4
    with:
      fetch-depth: 0

  # Checkout specific branch
  - uses: actions/checkout@v4
    with:
      ref: develop

  # Checkout specific commit
  - uses: actions/checkout@v4
    with:
      ref: ${{ github.sha }}

  # Checkout different repository
  - uses: actions/checkout@v4
    with:
      repository: other-org/other-repo
      token: ${{ secrets.PAT_TOKEN }}

  # Checkout with submodules
  - uses: actions/checkout@v4
    with:
      submodules: recursive
```

---

## 6.4 Setup Runtime Environments

### Node.js

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '20'
    node-version-file: '.nvmrc'    # อ่านจากไฟล์
    cache: 'npm'                   # cache npm
    # cache: 'yarn'
    # cache: 'pnpm'
    registry-url: 'https://registry.npmjs.org'
    scope: '@myorg'
```

### Python

```yaml
- name: Setup Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    python-version-file: '.python-version'
    cache: 'pip'
    # cache: 'poetry'
    # cache: 'pipenv'
```

### Go

```yaml
- name: Setup Go
  uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    go-version-file: 'go.mod'
    cache: true
```

### Java

```yaml
- name: Setup Java
  uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'
    # cache: 'gradle'
```

---

## 6.5 Install Dependencies

### npm

```bash
# สำหรับ CI (ใช้ package-lock.json)
npm ci

# ตรวจสอบ security
npm audit

# ตรวจสอบ audit level
npm audit --audit-level=high

# ดู dependencies ที่ outdated
npm outdated
```

### Yarn

```bash
# ติดตั้งด้วย Yarn
yarn install --frozen-lockfile

# หรือ Yarn Berry
yarn install --immutable
```

### pnpm

```bash
# ติดตั้ง pnpm ก่อน
npm install -g pnpm

# ติดตั้ง dependencies
pnpm install --frozen-lockfile
```

---

## 6.6 Run Linter

### ESLint

```yaml
steps:
  - name: Run ESLint
    run: npx eslint src --ext .js,.ts,.jsx,.tsx

  # บันทึก ESLint output เป็น SARIF
  - name: Run ESLint with SARIF output
    run: npx eslint src --format @microsoft/eslint-formatter-sarif --output-file results.sarif
    continue-on-error: true

  - name: Upload ESLint results
    uses: github/codeql-action/upload-sarif@v3
    with:
      sarif_file: results.sarif
```

### รวม linting tools

```yaml
steps:
  - name: Run all linters
    run: |
      echo "Running ESLint..."
      npm run lint

      echo "Checking Prettier..."
      npm run format:check

      echo "Type checking..."
      npx tsc --noEmit

      echo "All checks passed!"
```

---

## 6.7 Run Tests

### Jest

```yaml
steps:
  - name: Run tests
    run: npm test

  - name: Run tests with coverage
    run: npm run test:coverage

  - name: Run specific test file
    run: npx jest tests/calculator.test.js

  - name: Run tests in parallel
    run: npx jest --maxWorkers=4
```

### Coverage Report ใน PR

```yaml
steps:
  - name: Run tests with coverage
    run: npm run test:coverage

  - name: Upload coverage to Codecov
    uses: codecov/codecov-action@v4
    with:
      token: ${{ secrets.CODECOV_TOKEN }}
      directory: ./coverage
      fail_ci_if_error: true

  # หรือแสดงใน Job Summary
  - name: Coverage Summary
    run: |
      echo "## Test Coverage" >> $GITHUB_STEP_SUMMARY
      cat coverage/coverage-summary.json | \
        jq -r '"| Metric | Coverage |
      |--------|----------|
      | Lines | \(.total.lines.pct)% |
      | Statements | \(.total.statements.pct)% |
      | Functions | \(.total.functions.pct)% |
      | Branches | \(.total.branches.pct)% |"' >> $GITHUB_STEP_SUMMARY
```

---

## 6.8 Build Project

### Build Node.js Project

```yaml
steps:
  - name: Build
    run: npm run build

  - name: Check build output
    run: |
      echo "Build completed. Contents of dist/:"
      ls -la dist/

  - name: Verify build
    run: |
      if [ ! -f dist/index.js ]; then
        echo "Build failed: dist/index.js not found"
        exit 1
      fi
      echo "Build verification passed!"
```

### Docker Build

```yaml
steps:
  - name: Set up Docker Buildx
    uses: docker/setup-buildx-action@v3

  - name: Build Docker image
    uses: docker/build-push-action@v5
    with:
      context: .
      push: false
      tags: myapp:${{ github.sha }}
      cache-from: type=gha
      cache-to: type=gha,mode=max

  - name: Test Docker image
    run: |
      docker run --rm -d -p 3000:3000 --name test-app myapp:${{ github.sha }}
      sleep 3
      curl -f http://localhost:3000/health
      docker stop test-app
```

---

## 6.9 Upload Artifacts

### actions/upload-artifact

```yaml
steps:
  - name: Build
    run: npm run build

  # Upload ทุกไฟล์ใน dist
  - name: Upload build artifacts
    uses: actions/upload-artifact@v4
    with:
      name: build-output-${{ github.run_number }}
      path: dist/
      retention-days: 30

  # Upload หลาย paths
  - name: Upload multiple artifacts
    uses: actions/upload-artifact@v4
    with:
      name: all-artifacts
      path: |
        dist/
        coverage/
        reports/
      if-no-files-found: error

  # Upload ด้วย pattern
  - name: Upload logs
    uses: actions/upload-artifact@v4
    if: always()    # upload แม้ job fail
    with:
      name: logs-${{ github.run_id }}
      path: |
        **/*.log
        !node_modules/**
      retention-days: 7
```

### actions/download-artifact

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      - name: Deploy
        run: |
          echo "Deploying from dist/:"
          ls -la dist/
```

---

## 6.10 Branch-based Triggers

### Workflow สำหรับ Branch Strategy

```yaml
# .github/workflows/branch-ci.yml
name: Branch CI

on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'
      - 'hotfix/**'
      - 'release/**'

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm test

  deploy-staging:
    runs-on: ubuntu-latest
    needs: ci
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        run: echo "Deploying to staging..."

  deploy-production:
    runs-on: ubuntu-latest
    needs: ci
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://app.example.com
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: echo "Deploying to production..."
```

### Feature Branch Workflow

```yaml
# .github/workflows/feature-check.yml
name: Feature Branch Check

on:
  push:
    branches:
      - 'feature/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check branch name format
        run: |
          BRANCH="${{ github.ref_name }}"
          # ตรวจสอบว่า branch name อยู่ใน format feature/TICKET-123-description
          if ! echo "$BRANCH" | grep -qE '^feature/[A-Z]+-[0-9]+-[a-z0-9-]+$'; then
            echo "❌ Branch name '$BRANCH' does not follow naming convention"
            echo "Expected format: feature/TICKET-123-description"
            exit 1
          fi
          echo "✅ Branch name is valid: $BRANCH"

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm test
```

---

## 6.11 PR Checks

### Required Status Checks

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    branches: [main, develop]

jobs:
  # ตรวจสอบ PR title
  validate-pr:
    name: Validate PR
    runs-on: ubuntu-latest
    steps:
      - name: Check PR title
        uses: actions/github-script@v7
        with:
          script: |
            const title = context.payload.pull_request.title;
            const pattern = /^(feat|fix|docs|style|refactor|test|chore|ci)(\(.+\))?: .+/;

            if (!pattern.test(title)) {
              core.setFailed(`PR title "${title}" does not follow Conventional Commits format.
              Expected: type(scope): description
              Examples:
                feat(auth): add login functionality
                fix(api): resolve null pointer exception
                docs: update README`);
            } else {
              console.log(`✅ PR title is valid: ${title}`);
            }

  # Code quality checks
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run format:check

  # Tests
  test:
    name: Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:coverage

      - name: Coverage gate
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          MIN_COVERAGE=80
          if (( $(echo "$COVERAGE < $MIN_COVERAGE" | bc -l) )); then
            echo "❌ Coverage $COVERAGE% is below minimum $MIN_COVERAGE%"
            exit 1
          fi
          echo "✅ Coverage $COVERAGE% meets requirement"

  # Security check
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm audit --audit-level=high

  # Size check
  size-check:
    name: Bundle Size
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Check bundle size
        run: |
          SIZE=$(du -sh dist/ | cut -f1)
          echo "Bundle size: $SIZE"
          # Check ว่าไม่เกิน 10MB
          SIZE_BYTES=$(du -sb dist/ | cut -f1)
          MAX_BYTES=$((10 * 1024 * 1024))
          if [ $SIZE_BYTES -gt $MAX_BYTES ]; then
            echo "❌ Bundle size $SIZE exceeds 10MB limit"
            exit 1
          fi
          echo "✅ Bundle size $SIZE is within limits"
```

---

## 6.12 Job Summary

GitHub Actions รองรับการเขียน Markdown ลงใน Job Summary

```yaml
steps:
  - name: Run tests
    id: test
    run: npm run test:coverage

  - name: Generate summary
    if: always()
    run: |
      echo "## Test Results 🧪" >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY

      if [ "${{ steps.test.outcome }}" == "success" ]; then
        echo "✅ All tests passed!" >> $GITHUB_STEP_SUMMARY
      else
        echo "❌ Tests failed!" >> $GITHUB_STEP_SUMMARY
      fi

      echo "" >> $GITHUB_STEP_SUMMARY
      echo "### Coverage Report" >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY
      echo "| Metric | Coverage |" >> $GITHUB_STEP_SUMMARY
      echo "|--------|----------|" >> $GITHUB_STEP_SUMMARY

      # Parse coverage data
      LINES=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
      FUNCTIONS=$(cat coverage/coverage-summary.json | jq '.total.functions.pct')
      BRANCHES=$(cat coverage/coverage-summary.json | jq '.total.branches.pct')
      STATEMENTS=$(cat coverage/coverage-summary.json | jq '.total.statements.pct')

      echo "| Lines | ${LINES}% |" >> $GITHUB_STEP_SUMMARY
      echo "| Functions | ${FUNCTIONS}% |" >> $GITHUB_STEP_SUMMARY
      echo "| Branches | ${BRANCHES}% |" >> $GITHUB_STEP_SUMMARY
      echo "| Statements | ${STATEMENTS}% |" >> $GITHUB_STEP_SUMMARY

      echo "" >> $GITHUB_STEP_SUMMARY
      echo "**Run:** ${{ github.run_number }}" >> $GITHUB_STEP_SUMMARY
      echo "**Commit:** ${{ github.sha }}" >> $GITHUB_STEP_SUMMARY
```

---

## 6.13 Status Badges

### เพิ่ม Badge ใน README.md

```markdown
# My Project

[![CI Pipeline](https://github.com/username/repo/actions/workflows/ci.yml/badge.svg)](https://github.com/username/repo/actions/workflows/ci.yml)

[![CI Pipeline](https://github.com/username/repo/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/username/repo/actions/workflows/ci.yml)

[![CI Pipeline](https://github.com/username/repo/actions/workflows/ci.yml/badge.svg?event=push)](https://github.com/username/repo/actions/workflows/ci.yml)
```

### Codecov Badge

```markdown
[![codecov](https://codecov.io/gh/username/repo/branch/main/graph/badge.svg)](https://codecov.io/gh/username/repo)
```

### Badge Styles

```markdown
# Flat badge (default)
![CI](https://img.shields.io/github/actions/workflow/status/username/repo/ci.yml)

# With label
![CI](https://img.shields.io/github/actions/workflow/status/username/repo/ci.yml?label=CI%20Pipeline)

# Node version badge
![Node Version](https://img.shields.io/badge/node-%3E%3D18.0.0-brightgreen)

# License badge
![License](https://img.shields.io/github/license/username/repo)
```

---

## 6.14 CI Pipeline สำหรับ Python

### Python Application

```python
# src/calculator.py
"""Calculator module for CI demo."""


def add(a: float, b: float) -> float:
    """Add two numbers."""
    if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
        raise TypeError("Arguments must be numbers")
    return a + b


def subtract(a: float, b: float) -> float:
    """Subtract b from a."""
    if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
        raise TypeError("Arguments must be numbers")
    return a - b


def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
        raise TypeError("Arguments must be numbers")
    return a * b


def divide(a: float, b: float) -> float:
    """Divide a by b."""
    if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
        raise TypeError("Arguments must be numbers")
    if b == 0:
        raise ValueError("Division by zero is not allowed")
    return a / b
```

### Python Tests

```python
# tests/test_calculator.py
"""Tests for calculator module."""
import pytest
from src.calculator import add, subtract, multiply, divide


class TestAdd:
    """Tests for add function."""

    def test_add_positive_numbers(self):
        assert add(2, 3) == 5

    def test_add_negative_numbers(self):
        assert add(-1, -1) == -2

    def test_add_zero(self):
        assert add(0, 5) == 5

    def test_add_floats(self):
        assert add(0.1, 0.2) == pytest.approx(0.3)

    def test_add_invalid_type(self):
        with pytest.raises(TypeError):
            add("2", 3)


class TestDivide:
    """Tests for divide function."""

    def test_divide_normal(self):
        assert divide(10, 2) == 5.0

    def test_divide_by_zero(self):
        with pytest.raises(ValueError, match="Division by zero"):
            divide(10, 0)
```

### GitHub Actions สำหรับ Python

```yaml
# .github/workflows/python-ci.yml
name: Python CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  PYTHON_VERSION: '3.12'

jobs:
  lint:
    name: Lint Python
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'

      - name: Install linting tools
        run: |
          pip install ruff mypy

      - name: Run Ruff linter
        run: ruff check src tests

      - name: Run Ruff formatter check
        run: ruff format --check src tests

      - name: Run mypy type check
        run: mypy src

  test:
    name: Test Python ${{ matrix.python-version }}
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']

    steps:
      - uses: actions/checkout@v4

      - name: Setup Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run tests
        run: pytest tests/ -v

      - name: Run tests with coverage
        if: matrix.python-version == '3.12'
        run: pytest tests/ --cov=src --cov-report=xml --cov-report=html

      - name: Upload coverage
        if: matrix.python-version == '3.12'
        uses: actions/upload-artifact@v4
        with:
          name: python-coverage
          path: |
            coverage.xml
            htmlcov/

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Install safety
        run: pip install safety bandit

      - name: Check dependencies for vulnerabilities
        run: safety check

      - name: Run bandit security linter
        run: bandit -r src/ -ll
```

---

## 6.15 CI Pipeline สำหรับ Java

### Java Application

```java
// src/main/java/com/example/Calculator.java
package com.example;

public class Calculator {
    public double add(double a, double b) {
        return a + b;
    }

    public double subtract(double a, double b) {
        return a - b;
    }

    public double multiply(double a, double b) {
        return a * b;
    }

    public double divide(double a, double b) {
        if (b == 0) {
            throw new ArithmeticException("Division by zero");
        }
        return a / b;
    }
}
```

### Java Tests (JUnit 5)

```java
// src/test/java/com/example/CalculatorTest.java
package com.example;

import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {
    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }

    @Test
    @DisplayName("บวก 2 + 3 = 5")
    void testAdd() {
        assertEquals(5.0, calculator.add(2, 3));
    }

    @Test
    @DisplayName("หารด้วย 0 ต้อง throw exception")
    void testDivideByZero() {
        assertThrows(ArithmeticException.class, () -> {
            calculator.divide(10, 0);
        });
    }
}
```

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>ci-demo</artifactId>
    <version>1.0.0</version>

    <properties>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <junit.version>5.10.1</junit.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.3</version>
            </plugin>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.11</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>test</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

### GitHub Actions สำหรับ Java

```yaml
# .github/workflows/java-ci.yml
name: Java CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build-and-test:
    name: Build and Test (Java ${{ matrix.java-version }})
    runs-on: ubuntu-latest

    strategy:
      matrix:
        java-version: [17, 21]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Java ${{ matrix.java-version }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Build with Maven
        run: mvn clean compile

      - name: Run tests
        run: mvn test

      - name: Generate coverage report
        run: mvn jacoco:report

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-java${{ matrix.java-version }}
          path: |
            target/surefire-reports/
            target/site/jacoco/

      - name: Package application
        run: mvn package -DskipTests

      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: app-jar-java${{ matrix.java-version }}
          path: target/*.jar

  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Run Checkstyle
        run: mvn checkstyle:check

      - name: Run SpotBugs
        run: mvn spotbugs:check
```

---

## 6.16 CI Pipeline สำหรับ Go

### Go Application

```go
// calculator.go
package calculator

import "errors"

// Add returns the sum of two numbers
func Add(a, b float64) float64 {
	return a + b
}

// Subtract returns the difference of two numbers
func Subtract(a, b float64) float64 {
	return a - b
}

// Multiply returns the product of two numbers
func Multiply(a, b float64) float64 {
	return a * b
}

// Divide returns the quotient of two numbers
func Divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("division by zero")
	}
	return a / b, nil
}
```

### Go Tests

```go
// calculator_test.go
package calculator

import (
	"testing"
)

func TestAdd(t *testing.T) {
	tests := []struct {
		name string
		a, b float64
		want float64
	}{
		{"positive numbers", 2, 3, 5},
		{"negative numbers", -1, -1, -2},
		{"zero", 0, 5, 5},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := Add(tt.a, tt.b)
			if got != tt.want {
				t.Errorf("Add(%v, %v) = %v, want %v", tt.a, tt.b, got, tt.want)
			}
		})
	}
}

func TestDivide(t *testing.T) {
	t.Run("normal division", func(t *testing.T) {
		result, err := Divide(10, 2)
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if result != 5 {
			t.Errorf("expected 5, got %v", result)
		}
	})

	t.Run("division by zero", func(t *testing.T) {
		_, err := Divide(10, 0)
		if err == nil {
			t.Fatal("expected error, got nil")
		}
	})
}
```

### go.mod

```
module github.com/username/ci-demo-go

go 1.22
```

### GitHub Actions สำหรับ Go

```yaml
# .github/workflows/go-ci.yml
name: Go CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Run golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest

  test:
    name: Test (Go ${{ matrix.go-version }})
    runs-on: ubuntu-latest

    strategy:
      matrix:
        go-version: ['1.21', '1.22']

    steps:
      - uses: actions/checkout@v4

      - name: Setup Go ${{ matrix.go-version }}
        uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
          cache: true

      - name: Download dependencies
        run: go mod download

      - name: Verify dependencies
        run: go mod verify

      - name: Run tests
        run: go test ./... -v

      - name: Run tests with coverage
        if: matrix.go-version == '1.22'
        run: go test ./... -coverprofile=coverage.out -covermode=atomic

      - name: Display coverage
        if: matrix.go-version == '1.22'
        run: go tool cover -func=coverage.out

      - name: Upload coverage
        if: matrix.go-version == '1.22'
        uses: actions/upload-artifact@v4
        with:
          name: go-coverage
          path: coverage.out

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.22'

      - name: Build for multiple platforms
        run: |
          GOOS=linux GOARCH=amd64 go build -o dist/app-linux-amd64 .
          GOOS=darwin GOARCH=amd64 go build -o dist/app-darwin-amd64 .
          GOOS=windows GOARCH=amd64 go build -o dist/app-windows-amd64.exe .

      - name: Upload binaries
        uses: actions/upload-artifact@v4
        with:
          name: go-binaries
          path: dist/
```

---

## 6.17 Notification Setup

### Slack Notifications

```yaml
steps:
  - name: Notify Slack on success
    if: success()
    uses: slackapi/slack-github-action@v1.25.0
    with:
      channel-id: 'C12345678'
      slack-message: |
        ✅ CI Pipeline Passed!
        Repository: ${{ github.repository }}
        Branch: ${{ github.ref_name }}
        Commit: ${{ github.sha }}
        Author: ${{ github.actor }}
        Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
    env:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

  - name: Notify Slack on failure
    if: failure()
    uses: slackapi/slack-github-action@v1.25.0
    with:
      channel-id: 'C12345678'
      payload: |
        {
          "text": "❌ CI Pipeline Failed!",
          "attachments": [
            {
              "color": "#FF0000",
              "fields": [
                {
                  "title": "Repository",
                  "value": "${{ github.repository }}",
                  "short": true
                },
                {
                  "title": "Branch",
                  "value": "${{ github.ref_name }}",
                  "short": true
                },
                {
                  "title": "Run URL",
                  "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
                }
              ]
            }
          ]
        }
    env:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Email Notifications (ผ่าน GitHub)

GitHub ส่ง notification email อัตโนมัติเมื่อ workflow fail สามารถตั้งค่าที่:
Settings > Notifications > Actions

### Microsoft Teams

```yaml
steps:
  - name: Notify Teams
    if: failure()
    uses: jdcargile/ms-teams-notification@v1.4
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      ms-teams-webhook-uri: ${{ secrets.MS_TEAMS_WEBHOOK_URI }}
      notification-summary: CI Pipeline Failed
      notification-color: FF0000
      timezone: Asia/Bangkok
```

---

## 6.18 Best Practices สำหรับ CI Pipelines

### 1. Fast Feedback

```yaml
jobs:
  # ทำให้ quick checks รันก่อน
  quick-check:
    name: Quick Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint   # รันก่อน - เร็วกว่า tests
      - run: npm run format:check

  # tests รันหลัง quick-check ผ่าน
  test:
    needs: quick-check
    # ...
```

### 2. Cache ให้มากที่สุด

```yaml
steps:
  - name: Cache Node modules
    uses: actions/cache@v4
    with:
      path: |
        ~/.npm
        node_modules
      key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
      restore-keys: |
        ${{ runner.os }}-node-

  - name: Install (only if cache miss)
    run: npm ci
```

### 3. Fail Fast

```yaml
strategy:
  fail-fast: true   # หยุดทันทีเมื่อ matrix job ใดล้มเหลว
  matrix:
    node: [18, 20, 21]
```

### 4. แยก Concerns

```yaml
jobs:
  # แยก lint, test, build เป็น jobs ต่างๆ
  lint:    # Code quality
  test:    # Unit/integration tests
  build:   # Compile/package
  security: # Security checks
```

### 5. ใช้ Environments

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://app.example.com
    # ต้องมี reviewer approve ก่อน deploy
```

### 6. Meaningful Names

```yaml
# ❌ ชื่อไม่ชัดเจน
jobs:
  job1:
  job2:

# ✅ ชื่อชัดเจน
jobs:
  code-quality:
    name: Code Quality Check
  unit-tests:
    name: Unit Tests (Node ${{ matrix.node }})
```

### 7. Timeout ทุก Job

```yaml
jobs:
  build:
    timeout-minutes: 15
    steps:
      - name: Build
        timeout-minutes: 10
        run: npm run build
```

### 8. ใช้ continue-on-error สำหรับ Non-critical Steps

```yaml
steps:
  - name: Upload to Codecov
    uses: codecov/codecov-action@v4
    continue-on-error: true  # ไม่ให้ fail workflow ถ้า Codecov มีปัญหา
```

---

## 6.19 Full Production-ready CI Workflow

```yaml
# .github/workflows/production-ci.yml
name: Production CI

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
      - '.github/ISSUE_TEMPLATE/**'
  pull_request:
    branches: [main, develop]
    types: [opened, synchronize, reopened]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

defaults:
  run:
    shell: bash

env:
  NODE_VERSION: '20'
  NODE_ENV: test

jobs:
  # =============================
  # Validate PR
  # =============================
  validate:
    name: Validate
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    permissions:
      pull-requests: read
    steps:
      - name: Check PR title
        uses: actions/github-script@v7
        with:
          script: |
            const title = context.payload.pull_request.title;
            const regex = /^(feat|fix|docs|style|refactor|test|chore|ci|build|perf|revert)(\(.+\))?: .{3,}/;
            if (!regex.test(title)) {
              core.setFailed(`Invalid PR title: "${title}"`);
            }

  # =============================
  # Code Quality
  # =============================
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Format check
        run: npm run format:check

      - name: Type check
        run: npx tsc --noEmit
        continue-on-error: false

  # =============================
  # Security
  # =============================
  security:
    name: Security
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Audit dependencies
        run: npm audit --audit-level=high

  # =============================
  # Tests
  # =============================
  test:
    name: Test
    runs-on: ubuntu-latest
    needs: [quality]
    timeout-minutes: 15

    strategy:
      fail-fast: false
      matrix:
        node: [18, 20]

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test -- --forceExit

      - name: Coverage (Node 20 only)
        if: matrix.node == 20
        run: npm run test:coverage

      - name: Upload coverage
        if: matrix.node == 20
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/
          retention-days: 30

  # =============================
  # Build
  # =============================
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [quality, security, test]
    timeout-minutes: 15

    outputs:
      version: ${{ steps.version.outputs.version }}
      build-date: ${{ steps.build-meta.outputs.date }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Get version
        id: version
        run: |
          VERSION=$(node -p "require('./package.json').version")
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - name: Build
        run: npm run build

      - name: Build metadata
        id: build-meta
        run: |
          echo "date=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> $GITHUB_OUTPUT

      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ github.sha }}
          path: dist/
          retention-days: 7

  # =============================
  # Summary
  # =============================
  summary:
    name: CI Summary
    runs-on: ubuntu-latest
    needs: [quality, security, test, build]
    if: always()
    permissions:
      pull-requests: write

    steps:
      - name: Generate summary
        run: |
          echo "## CI Pipeline Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Job | Status |" >> $GITHUB_STEP_SUMMARY
          echo "|-----|--------|" >> $GITHUB_STEP_SUMMARY
          echo "| Code Quality | ${{ needs.quality.result == 'success' && '✅' || '❌' }} ${{ needs.quality.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Security | ${{ needs.security.result == 'success' && '✅' || '❌' }} ${{ needs.security.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Tests | ${{ needs.test.result == 'success' && '✅' || '❌' }} ${{ needs.test.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Build | ${{ needs.build.result == 'success' && '✅' || '❌' }} ${{ needs.build.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Commit:** \`${{ github.sha }}\`" >> $GITHUB_STEP_SUMMARY
          echo "**Triggered by:** ${{ github.actor }}" >> $GITHUB_STEP_SUMMARY
          echo "**Run:** [${{ github.run_number }}](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})" >> $GITHUB_STEP_SUMMARY

      - name: Comment PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const quality = '${{ needs.quality.result }}';
            const security = '${{ needs.security.result }}';
            const test = '${{ needs.test.result }}';
            const build = '${{ needs.build.result }}';

            const statusEmoji = (s) => s === 'success' ? '✅' : s === 'skipped' ? '⏭️' : '❌';

            const body = `## CI Pipeline Results

            | Job | Status |
            |-----|--------|
            | Code Quality | ${statusEmoji(quality)} ${quality} |
            | Security | ${statusEmoji(security)} ${security} |
            | Tests | ${statusEmoji(test)} ${test} |
            | Build | ${statusEmoji(build)} ${build} |

            **Commit:** \`${{ github.sha }}\`
            **Author:** @${{ github.actor }}
            **Run:** [${{ github.run_number }}](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})
            `;

            const { data: comments } = await github.rest.issues.listComments({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
            });

            // หา comment ที่เคย post ไว้แล้ว
            const existingComment = comments.find(c =>
              c.user.login === 'github-actions[bot]' &&
              c.body.includes('CI Pipeline Results')
            );

            if (existingComment) {
              // Update comment
              await github.rest.issues.updateComment({
                comment_id: existingComment.id,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body,
              });
            } else {
              // Create new comment
              await github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body,
              });
            }
```

---

## 6.20 Exercises

### Exercise 1: สร้าง Node.js CI Pipeline

**โจทย์:** สร้าง complete CI pipeline สำหรับ Node.js application ที่:
1. Clone repository ที่เราสร้างใน section 6.1
2. รัน ESLint และ Prettier check
3. รัน unit tests พร้อม coverage ไม่น้อยกว่า 80%
4. Build project
5. Upload artifacts
6. แสดง summary ใน Job Summary

**Checklist:**
```
[ ] สร้าง .github/workflows/ci.yml
[ ] ตั้งค่า triggers สำหรับ push และ PR
[ ] สร้าง lint job
[ ] สร้าง test job พร้อม coverage check
[ ] สร้าง build job
[ ] สร้าง summary job
[ ] ทดสอบด้วยการ push code
```

### Exercise 2: Matrix Testing

**โจทย์:** แก้ไข CI workflow ให้ทดสอบบน:
- Ubuntu 22.04 + Node.js 18, 20
- Windows Latest + Node.js 20
- macOS Latest + Node.js 20

### Exercise 3: PR Checks

**โจทย์:** สร้าง PR workflow ที่:
1. ตรวจสอบ PR title ตาม Conventional Commits
2. รัน tests
3. Check coverage
4. Comment ผลลัพธ์ลงใน PR

### Exercise 4: Python CI

**โจทย์:** สร้าง CI pipeline สำหรับ Python project ที่มี:
- Ruff linter
- mypy type checking  
- pytest พร้อม coverage
- Matrix test บน Python 3.10, 3.11, 3.12
- Security scan ด้วย safety

### Exercise 5: Complete CI with Notifications

**โจทย์:** สร้าง production-ready CI pipeline ที่:
1. รัน checks ทั้งหมด
2. Deploy ไป staging เมื่อ push ไป develop
3. ส่ง Slack notification เมื่อ pipeline success/fail
4. แสดง status badge ใน README

---

## 6.21 Troubleshooting ปัญหาที่พบบ่อย

### ปัญหา: npm ci fails

```yaml
# ตรวจสอบว่ามี package-lock.json
- name: Check lock file
  run: |
    if [ ! -f package-lock.json ]; then
      echo "package-lock.json not found!"
      exit 1
    fi

# ใช้ npm install แทน npm ci ถ้าไม่มี lock file
- name: Install
  run: |
    if [ -f package-lock.json ]; then
      npm ci
    else
      npm install
    fi
```

### ปัญหา: Tests timeout

```yaml
- name: Run tests
  run: npm test
  timeout-minutes: 10
  env:
    # เพิ่ม timeout สำหรับ Jest
    JEST_TIMEOUT: 30000
```

### ปัญหา: Permission denied

```yaml
- name: Make script executable
  run: chmod +x ./scripts/deploy.sh

- name: Run script
  run: ./scripts/deploy.sh
```

### ปัญหา: Out of disk space

```yaml
- name: Free disk space
  run: |
    sudo rm -rf /usr/share/dotnet
    sudo rm -rf /opt/ghc
    sudo rm -rf "/usr/local/share/boost"
    sudo rm -rf "$AGENT_TOOLSDIRECTORY"
    df -h
```

---

## 6.22 สรุป

ในบทนี้เราได้เรียนรู้:

1. **สร้าง Node.js application** พร้อม tests ที่ใช้สำหรับ CI
2. **เขียน GitHub Actions CI workflow** ที่ครบครัน
3. **Checkout และ Setup** runtime environments
4. **Install dependencies** อย่างถูกวิธี
5. **Run linter และ formatter** checks
6. **Run tests** พร้อม coverage reporting
7. **Build และ upload artifacts**
8. **Branch-based triggers** สำหรับ different environments
9. **PR checks** รวมถึง automated comments
10. **Status badges** สำหรับ README
11. **CI pipelines** สำหรับ Python, Java, Go
12. **Best practices** สำหรับ production CI

### CI Pipeline Checklist

```
[ ] Trigger บน push และ pull_request
[ ] Concurrency ป้องกัน duplicate runs
[ ] Code quality checks (lint, format)
[ ] Security audit
[ ] Unit tests พร้อม coverage
[ ] Matrix testing บนหลาย versions
[ ] Build artifacts
[ ] Upload artifacts
[ ] PR comments
[ ] Status badges
[ ] Notifications
[ ] Timeouts ทุก jobs
[ ] Fail fast configuration
```

### แหล่งข้อมูลเพิ่มเติม

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [actions/checkout](https://github.com/actions/checkout)
- [actions/setup-node](https://github.com/actions/setup-node)
- [actions/upload-artifact](https://github.com/actions/upload-artifact)
- [Codecov](https://codecov.io/)
- [GitHub Actions Examples](https://github.com/actions/starter-workflows)

---

**ต่อไป:** Part 07 - CD Pipeline และ Deployment Strategies

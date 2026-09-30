# Part 09: Code Quality, Linting & Formatting

> **หลักสูตร CI/CD สำหรับนักพัฒนา** | ระดับ: ปานกลาง | เวลาเรียน: ~4 ชั่วโมง

---

## สารบัญ

1. [Code Quality คืออะไร?](#1-code-quality-คืออะไร)
2. [ESLint สำหรับ JavaScript/TypeScript](#2-eslint-สำหรับ-javascript)
3. [Pylint, Flake8, Black สำหรับ Python](#3-pylint-flake8-black)
4. [Checkstyle สำหรับ Java](#4-checkstyle-สำหรับ-java)
5. [Prettier สำหรับ Code Formatting](#5-prettier)
6. [SonarQube และ SonarCloud](#6-sonarqube-และ-sonarcloud)
7. [Code Smell Detection](#7-code-smell-detection)
8. [Complexity Metrics](#8-complexity-metrics)
9. [Pre-commit Hooks ด้วย Husky](#9-pre-commit-hooks-ด้วย-husky)
10. [lint-staged](#10-lint-staged)
11. [GitHub Actions Lint Workflow](#11-github-actions-lint-workflow)
12. [Code Review Checklist](#12-code-review-checklist)
13. [แบบฝึกหัด](#13-แบบฝึกหัด)

---

## 1. Code Quality คืออะไร?

### 1.1 นิยาม Code Quality

Code Quality หมายถึงคุณสมบัติต่างๆ ของโค้ดที่ทำให้:
- **อ่านง่าย** (Readability) — นักพัฒนาคนอื่นเข้าใจได้เร็ว
- **บำรุงรักษาง่าย** (Maintainability) — แก้ไขได้โดยไม่ทำให้ส่วนอื่นพัง
- **ทดสอบได้** (Testability) — เขียน test ได้ง่าย
- **ปลอดภัย** (Security) — ไม่มีช่องโหว่ที่รู้จัก
- **มีประสิทธิภาพ** (Performance) — ทำงานได้รวดเร็ว

### 1.2 Code Quality Metrics

| Metric | คำอธิบาย | เป้าหมาย |
|--------|---------|---------|
| **Cyclomatic Complexity** | ความซับซ้อนของ logic | ≤ 10 ต่อ function |
| **Code Duplication** | % โค้ดที่ซ้ำกัน | < 3% |
| **Test Coverage** | % โค้ดที่ถูกทดสอบ | > 80% |
| **Technical Debt** | เวลาที่ต้องแก้ไขโค้ดไม่ดี | < 5% development time |
| **Code Churn** | ความถี่ในการเปลี่ยน file | Monitor patterns |
| **Coupling** | ความพึ่งพาระหว่าง modules | ต่ำ |
| **Cohesion** | ความเกี่ยวข้องของ code ใน module | สูง |

### 1.3 ทำไม Code Quality ถึงสำคัญใน CI/CD?

```
Code Quality ต่ำ → Technical Debt สูง → Velocity ลดลง → ยาก deploy → CI/CD ช้า

Code Quality สูง → Technical Debt ต่ำ → Velocity สูง → Deploy ได้เร็ว → CI/CD เร็ว
```

**ผลกระทบของ Code Quality ต่ำ:**
- Bug เยอะ → Test ล้มเหลวบ่อย → Pipeline ช้าลง
- อ่านยาก → Code review นาน → PR merge ช้า
- Coupling สูง → แก้ส่วนหนึ่งพังส่วนอื่น → Rollback บ่อย

---

## 2. ESLint สำหรับ JavaScript

### 2.1 ติดตั้ง ESLint

```bash
# ติดตั้ง ESLint
npm install --save-dev eslint

# ใช้ wizard ตั้งค่า (สำหรับ project ใหม่)
npx eslint --init

# ติดตั้ง plugins ยอดนิยม
npm install --save-dev \
  eslint-plugin-react \
  eslint-plugin-react-hooks \
  eslint-plugin-jsx-a11y \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser \
  eslint-plugin-import \
  eslint-plugin-node \
  eslint-plugin-promise \
  eslint-config-airbnb \
  eslint-config-airbnb-base
```

### 2.2 .eslintrc.js — การตั้งค่าพื้นฐาน

```javascript
// .eslintrc.js
module.exports = {
  root: true,
  
  // Environment
  env: {
    browser: true,
    es2021: true,
    node: true,
    jest: true,  // Jest globals (describe, test, expect)
  },
  
  // Parser สำหรับ syntax ต่างๆ
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module',
    ecmaFeatures: {
      jsx: true,
    },
    project: './tsconfig.json',  // สำหรับ TypeScript rules
  },
  
  // Plugins
  plugins: [
    '@typescript-eslint',
    'import',
    'promise',
  ],
  
  // Extends (ลำดับสำคัญ: ที่หลังจะ override ที่ก่อน)
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:import/recommended',
    'plugin:promise/recommended',
    'prettier',  // ต้องอยู่ท้ายสุด! ปิด rules ที่ conflict กับ prettier
  ],
  
  // Rules
  rules: {
    // ── Error Prevention ──────────────────────────────────────────
    'no-console': ['warn', { allow: ['warn', 'error'] }],
    'no-debugger': 'error',
    'no-unused-vars': 'off',  // ใช้ TypeScript version แทน
    '@typescript-eslint/no-unused-vars': [
      'error',
      { 
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      },
    ],
    'no-var': 'error',
    'prefer-const': 'error',
    'eqeqeq': ['error', 'always'],  // ต้องใช้ === ไม่ใช่ ==
    
    // ── Async/Await ───────────────────────────────────────────────
    'require-await': 'error',
    'no-return-await': 'error',
    'promise/catch-or-return': 'error',
    'promise/no-return-wrap': 'error',
    
    // ── Import Organization ───────────────────────────────────────
    'import/order': [
      'error',
      {
        groups: ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
        'newlines-between': 'always',
        alphabetize: { order: 'asc', caseInsensitive: true },
      },
    ],
    'import/no-unused-modules': 'warn',
    'import/no-duplicates': 'error',
    
    // ── Code Style ────────────────────────────────────────────────
    'max-lines': ['warn', { max: 300, skipBlankLines: true, skipComments: true }],
    'max-lines-per-function': ['warn', { max: 50, skipBlankLines: true }],
    'complexity': ['error', { max: 10 }],   // Cyclomatic complexity
    'max-depth': ['error', 4],               // Nesting depth
    'max-params': ['warn', 4],              // Function parameters
    
    // ── TypeScript Specific ───────────────────────────────────────
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/no-explicit-any': 'warn',
    '@typescript-eslint/no-non-null-assertion': 'warn',
  },
  
  // Override rules สำหรับไฟล์เฉพาะ
  overrides: [
    {
      // Test files: allow more relaxed rules
      files: ['**/*.test.ts', '**/*.test.js', '**/*.spec.ts'],
      rules: {
        'max-lines-per-function': 'off',
        '@typescript-eslint/no-explicit-any': 'off',
      },
    },
    {
      // Config files: allow require()
      files: ['*.config.js', '*.config.ts'],
      rules: {
        '@typescript-eslint/no-var-requires': 'off',
      },
    },
  ],
  
  // ไฟล์ที่ไม่ต้องตรวจ
  ignorePatterns: [
    'node_modules/',
    'dist/',
    'build/',
    'coverage/',
    '*.min.js',
    'public/',
  ],
};
```

### 2.3 .eslintrc.js สำหรับ React Project

```javascript
// .eslintrc.js (React + TypeScript)
module.exports = {
  root: true,
  env: {
    browser: true,
    es2021: true,
  },
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module',
    ecmaFeatures: { jsx: true },
  },
  plugins: ['react', 'react-hooks', 'jsx-a11y', '@typescript-eslint'],
  extends: [
    'eslint:recommended',
    'plugin:react/recommended',
    'plugin:react/jsx-runtime',   // React 17+: ไม่ต้อง import React
    'plugin:react-hooks/recommended',
    'plugin:jsx-a11y/recommended',
    'plugin:@typescript-eslint/recommended',
    'prettier',
  ],
  settings: {
    react: {
      version: 'detect',
    },
  },
  rules: {
    // React specific
    'react/prop-types': 'off',              // ใช้ TypeScript แทน
    'react/display-name': 'warn',
    'react-hooks/rules-of-hooks': 'error', // ต้องใช้ hooks ถูกต้อง
    'react-hooks/exhaustive-deps': 'warn', // dependency array ครบ
    
    // Accessibility
    'jsx-a11y/anchor-is-valid': 'warn',
    
    // TypeScript
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
  },
};
```

### 2.4 Scripts และ Auto-fix

```json
// package.json
{
  "scripts": {
    "lint": "eslint src --ext .ts,.tsx,.js,.jsx",
    "lint:fix": "eslint src --ext .ts,.tsx,.js,.jsx --fix",
    "lint:report": "eslint src --ext .ts,.tsx,.js,.jsx --format html -o reports/eslint.html"
  }
}
```

```bash
# ตรวจสอบ
npm run lint

# แก้อัตโนมัติที่แก้ได้
npm run lint:fix

# ดู errors เฉพาะ (ไม่ warnings)
npx eslint src --quiet
```

### 2.5 ESLint สำหรับ Node.js Backend

```javascript
// .eslintrc.js (Node.js)
module.exports = {
  env: {
    node: true,
    es2021: true,
  },
  extends: [
    'eslint:recommended',
    'plugin:node/recommended',
    'plugin:security/recommended',  // Security checks
    'prettier',
  ],
  plugins: ['node', 'security'],
  rules: {
    'node/exports-style': ['error', 'module.exports'],
    'node/prefer-global/buffer': ['error', 'always'],
    'node/prefer-global/console': ['error', 'always'],
    'node/prefer-global/process': ['error', 'always'],
    
    // Security rules
    'security/detect-sql-injection': 'error',
    'security/detect-non-literal-regexp': 'warn',
    'security/detect-unsafe-regex': 'error',
    'security/detect-possible-timing-attacks': 'warn',
  },
};
```

---

## 3. Pylint, Flake8, Black

### 3.1 Flake8 — Style Check

```bash
pip install flake8 flake8-bugbear flake8-comprehensions flake8-simplify
```

**.flake8:**
```ini
[flake8]
# Maximum line length
max-line-length = 100
max-doc-length = 100

# ไม่ตรวจ files เหล่านี้
exclude =
    .git,
    __pycache__,
    .venv,
    venv,
    build,
    dist,
    migrations,
    *.egg-info

# Plugins ที่ใช้
extend-select =
    B,      # flake8-bugbear
    C4,     # flake8-comprehensions
    SIM,    # flake8-simplify

# Rules ที่ยกเว้น (ต้องอธิบายเหตุผล)
ignore =
    E501,   # line too long (handled by black)
    W503,   # line break before binary operator (conflicts with black)
    B008,   # do not perform calls in default args (common pattern in FastAPI)

# Per-file ignores
per-file-ignores =
    __init__.py: F401   # Unused imports OK in __init__.py
    tests/*: S101       # assert OK in tests
    */migrations/*: E501,W291
```

```bash
# Run flake8
flake8 src/ tests/

# Run พร้อม statistics
flake8 --statistics --count src/

# ดู rules ที่ถูก triggered
flake8 --show-pep8 src/
```

### 3.2 Pylint — Comprehensive Linting

```bash
pip install pylint pylint-django  # หรือ pylint-flask
```

**.pylintrc:**
```ini
[MASTER]
# Run pylint สำหรับ job count
jobs = 4

# Python version
py-version = 3.11

# ไม่ตรวจไฟล์เหล่านี้
ignore = 
    migrations,
    .git,
    __pycache__

[MESSAGES CONTROL]
# ปิด messages ที่ไม่ต้องการ
disable =
    missing-docstring,     # ไม่บังคับ docstrings (ใช้ type hints แทน)
    too-few-public-methods, # dataclasses มักมี method น้อย
    C0114,                  # Missing module docstring
    R0903,                  # Too few public methods

# เปิด messages เพิ่มเติม
enable =
    useless-suppression

[FORMAT]
max-line-length = 100
indent-string = '    '
indent-after-paren = 4

[DESIGN]
# จำนวน arguments สูงสุด
max-args = 5
# จำนวน returns สูงสุด
max-returns = 3
# จำนวน branches สูงสุด (cyclomatic complexity)
max-branches = 10
# จำนวน statements สูงสุดใน function
max-statements = 40
# จำนวน attributes สูงสุดใน class
max-attributes = 10
# จำนวน public methods สูงสุดใน class
max-public-methods = 20

[IMPORTS]
# ต้องไม่ import wildcard (from module import *)
allow-wildcard-with-all = no

[TYPECHECK]
# Modules ที่อาจ generate attributes dynamically
generated-members = 
    numpy.*,
    torch.*,
    django.db.models.fields.*

[SIMILARITIES]
# ความยาวขั้นต่ำของ similar block
min-similarity-lines = 10
# ยกเว้น comments และ docstrings
ignore-comments = yes
ignore-docstrings = yes

[REPORTS]
# ให้ output เป็น text
output-format = text
# แสดง full report
reports = yes
# แสดง score
score = yes
```

```bash
# Run pylint
pylint src/ --rcfile=.pylintrc

# ดู score
pylint src/ --score=yes

# Report เป็น JSON
pylint src/ --output-format=json > pylint-report.json
```

### 3.3 Black — Code Formatter

```bash
pip install black
```

**pyproject.toml:**
```toml
[tool.black]
line-length = 100
target-version = ['py310', 'py311', 'py312']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.hg
  | \.mypy_cache
  | \.tox
  | \.venv
  | _build
  | buck-out
  | build
  | dist
  | migrations
)/
'''
```

```bash
# Format code
black src/

# Check without modifying (สำหรับ CI)
black --check src/

# ดูความแตกต่าง
black --diff src/

# Format เฉพาะ file
black src/services/user_service.py
```

**ตัวอย่าง Before/After Black:**
```python
# Before Black
def calculate_total(items, discount=0, tax_rate=0.07, include_shipping=True, shipping_cost=50):
    subtotal = sum(item['price'] * item['quantity'] for item in items)
    discounted = subtotal - discount
    tax = discounted * tax_rate
    total = discounted + tax
    if include_shipping:
        total += shipping_cost
    return total

# After Black
def calculate_total(
    items,
    discount=0,
    tax_rate=0.07,
    include_shipping=True,
    shipping_cost=50,
):
    subtotal = sum(item["price"] * item["quantity"] for item in items)
    discounted = subtotal - discount
    tax = discounted * tax_rate
    total = discounted + tax
    if include_shipping:
        total += shipping_cost
    return total
```

### 3.4 isort — Import Sorting

```bash
pip install isort
```

**pyproject.toml:**
```toml
[tool.isort]
profile = "black"           # Compatible with black
line_length = 100
multi_line_output = 3
include_trailing_comma = true
force_grid_wrap = 0
use_parentheses = true
ensure_newline_before_comments = true
known_third_party = ["fastapi", "pydantic", "sqlalchemy"]
known_first_party = ["src", "app"]
sections = ["FUTURE", "STDLIB", "THIRDPARTY", "FIRSTPARTY", "LOCALFOLDER"]
```

```bash
# Sort imports
isort src/

# Check
isort --check-only src/

# ดูความแตกต่าง
isort --diff src/
```

### 3.5 mypy — Type Checking

```bash
pip install mypy
```

**mypy.ini:**
```ini
[mypy]
python_version = 3.11
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
disallow_incomplete_defs = True
check_untyped_defs = True
no_implicit_optional = True
warn_redundant_casts = True
warn_unused_ignores = True
show_error_codes = True
strict = True

# Per-module settings
[mypy-third_party_lib.*]
ignore_missing_imports = True

[mypy-legacy_code.*]
ignore_errors = True
```

```bash
# Type check
mypy src/

# Ignore missing stubs
mypy src/ --ignore-missing-imports
```

### 3.6 Ruff — Modern Python Linter (เร็วมาก!)

```bash
pip install ruff
```

**pyproject.toml:**
```toml
[tool.ruff]
line-length = 100
target-version = "py311"

# รวม rules จากหลาย linters
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "SIM", # flake8-simplify
    "TCH", # flake8-type-checking
    "N",   # pep8-naming
]

ignore = [
    "E501",  # line too long (handled by black)
    "B008",  # do not perform function calls in argument defaults
]

exclude = [
    ".git",
    ".venv",
    "build",
    "dist",
    "migrations",
]

[tool.ruff.per-file-ignores]
"__init__.py" = ["F401"]
"tests/*" = ["S101", "S311"]

[tool.ruff.isort]
known-first-party = ["src"]
```

```bash
# Check
ruff check src/

# Fix
ruff check src/ --fix

# Format (เหมือน black)
ruff format src/
```

---

## 4. Checkstyle สำหรับ Java

### 4.1 ติดตั้ง Checkstyle

**pom.xml:**
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.0</version>
    <configuration>
        <configLocation>checkstyle.xml</configLocation>
        <consoleOutput>true</consoleOutput>
        <failsOnError>true</failsOnError>
        <includeTestSourceDirectory>true</includeTestSourceDirectory>
    </configuration>
    <executions>
        <execution>
            <id>validate</id>
            <phase>validate</phase>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 4.2 checkstyle.xml

```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">

<module name="Checker">
    <property name="severity" value="error"/>
    <property name="fileExtensions" value="java, properties, xml"/>
    
    <!-- ตรวจสอบ tab characters -->
    <module name="FileTabCharacter">
        <property name="eachLine" value="true"/>
    </module>
    
    <!-- Trailing whitespace -->
    <module name="RegexpSingleline">
        <property name="format" value="\s+$"/>
        <property name="message" value="Line has trailing whitespace"/>
    </module>
    
    <module name="TreeWalker">
        
        <!-- ── Naming Conventions ─────────────────────────────────── -->
        <module name="TypeName"/>
        <module name="MethodName">
            <property name="format" value="^[a-z][a-zA-Z0-9]*$"/>
        </module>
        <module name="ParameterName"/>
        <module name="LocalVariableName"/>
        <module name="ConstantName"/>
        <module name="PackageName">
            <property name="format" value="^[a-z]+(\.[a-z][a-z0-9]*)*$"/>
        </module>
        
        <!-- ── Code Size ──────────────────────────────────────────── -->
        <module name="MethodLength">
            <property name="max" value="50"/>
        </module>
        <module name="ParameterNumber">
            <property name="max" value="5"/>
        </module>
        <module name="LineLength">
            <property name="max" value="120"/>
            <property name="ignorePattern" value="^package.*|^import.*|a href|href|http://|https://|ftp://"/>
        </module>
        
        <!-- ── Complexity ─────────────────────────────────────────── -->
        <module name="CyclomaticComplexity">
            <property name="max" value="10"/>
        </module>
        <module name="NPathComplexity">
            <property name="max" value="200"/>
        </module>
        
        <!-- ── JavaDoc ────────────────────────────────────────────── -->
        <module name="JavadocMethod">
            <property name="accessModifiers" value="public"/>
        </module>
        <module name="JavadocType">
            <property name="scope" value="public"/>
        </module>
        
        <!-- ── Block Checks ──────────────────────────────────────── -->
        <module name="NeedBraces"/>
        <module name="LeftCurly"/>
        <module name="RightCurly"/>
        <module name="EmptyBlock">
            <property name="option" value="TEXT"/>
            <property name="tokens" value="LITERAL_TRY,LITERAL_FINALLY,LITERAL_IF,LITERAL_ELSE"/>
        </module>
        
        <!-- ── Whitespace ────────────────────────────────────────── -->
        <module name="NoWhitespaceAfter"/>
        <module name="NoWhitespaceBefore"/>
        <module name="WhitespaceAround"/>
        
        <!-- ── Imports ────────────────────────────────────────────── -->
        <module name="UnusedImports"/>
        <module name="AvoidStarImport"/>
        <module name="ImportOrder">
            <property name="groups" value="/^java\./,/^javax\./,/^org\./,*"/>
            <property name="separated" value="true"/>
        </module>
        
        <!-- ── Miscellaneous ─────────────────────────────────────── -->
        <module name="UpperEll"/>
        <module name="ArrayTypeStyle"/>
        <module name="TodoComment">
            <property name="format" value="(TODO:|FIXME:)"/>
        </module>
        
        <!-- ── Design ────────────────────────────────────────────── -->
        <module name="FinalClass"/>
        <module name="HideUtilityClassConstructor"/>
        <module name="VisibilityModifier">
            <property name="packageAllowed" value="false"/>
            <property name="protectedAllowed" value="true"/>
        </module>
        
    </module>
</module>
```

### 4.3 Google Java Style

```bash
# ใช้ Google Java Style (เป็นที่นิยม)
# checkstyle.xml
<module name="Checker">
    <module name="TreeWalker">
        <!-- Use Google Java Style -->
        <!-- Download: https://github.com/checkstyle/checkstyle/blob/master/src/main/resources/google_checks.xml -->
    </module>
</module>
```

---

## 5. Prettier

### 5.1 ติดตั้ง Prettier

```bash
npm install --save-dev prettier
```

### 5.2 .prettierrc.js

```javascript
// .prettierrc.js
/** @type {import("prettier").Config} */
module.exports = {
  // ── Basic ──────────────────────────────────────────────────────
  printWidth: 100,          // ความกว้างบรรทัดสูงสุด
  tabWidth: 2,              // indent ด้วย 2 spaces
  useTabs: false,           // ใช้ spaces ไม่ใช่ tabs
  semi: true,               // เพิ่ม semicolons
  singleQuote: true,        // ใช้ single quotes
  quoteProps: 'as-needed',  // quotes ใน object keys เฉพาะเมื่อจำเป็น
  
  // ── JSX ────────────────────────────────────────────────────────
  jsxSingleQuote: false,    // ใช้ double quotes ใน JSX
  
  // ── Arrays/Objects ────────────────────────────────────────────
  trailingComma: 'es5',    // trailing comma ตาม ES5 spec
  bracketSpacing: true,     // spaces ใน object literals: { foo: bar }
  bracketSameLine: false,   // ปิด JSX ใน next line
  
  // ── Arrow Functions ───────────────────────────────────────────
  arrowParens: 'always',   // (x) => แทน x =>
  
  // ── Markdown ──────────────────────────────────────────────────
  proseWrap: 'preserve',   // ไม่ wrap markdown prose
  
  // ── HTML ──────────────────────────────────────────────────────
  htmlWhitespaceSensitivity: 'css',
  
  // ── End of Line ───────────────────────────────────────────────
  endOfLine: 'lf',         // ใช้ LF (Unix-style)
  
  // ── Embedded Languages ────────────────────────────────────────
  embeddedLanguageFormatting: 'auto',
  
  // ── Overrides per file type ───────────────────────────────────
  overrides: [
    {
      files: '*.json',
      options: { printWidth: 80 },
    },
    {
      files: '*.md',
      options: {
        proseWrap: 'always',
        printWidth: 80,
      },
    },
    {
      files: ['*.yaml', '*.yml'],
      options: { tabWidth: 2, singleQuote: false },
    },
  ],
};
```

**.prettierignore:**
```
# Build outputs
dist/
build/
.next/
out/

# Dependencies
node_modules/

# Auto-generated
coverage/
*.min.js
*.min.css
package-lock.json

# Version control
.git/

# Specific files
CHANGELOG.md
```

### 5.3 Scripts

```json
// package.json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "format:changed": "prettier --write $(git diff --name-only --diff-filter=ACMR)"
  }
}
```

### 5.4 Prettier + ESLint Integration

```bash
npm install --save-dev eslint-config-prettier eslint-plugin-prettier
```

```javascript
// .eslintrc.js
module.exports = {
  extends: [
    'eslint:recommended',
    // ...other configs...
    'plugin:prettier/recommended',  // ต้องอยู่ท้ายสุด
  ],
  rules: {
    'prettier/prettier': 'error',  // Prettier violations เป็น ESLint errors
  },
};
```

---

## 6. SonarQube และ SonarCloud

### 6.1 SonarCloud Setup

```bash
# ติดตั้ง sonar-scanner
npm install --save-dev sonarqube-scanner
```

**sonar-project.properties:**
```properties
# Project identification
sonar.projectKey=myorg_myproject
sonar.organization=myorg
sonar.projectName=My Project
sonar.projectVersion=1.0

# Source code
sonar.sources=src
sonar.tests=tests
sonar.exclusions=**/node_modules/**,**/dist/**,**/*.test.js

# Language-specific
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.testExecutionReportPaths=test-results/junit.xml

# Quality Gate
sonar.qualitygate.wait=true

# Branch analysis
sonar.branch.name=${GITHUB_REF_NAME}

# Pull Request
sonar.pullrequest.key=${GITHUB_PR_NUMBER}
sonar.pullrequest.branch=${GITHUB_HEAD_REF}
sonar.pullrequest.base=${GITHUB_BASE_REF}
```

### 6.2 GitHub Actions + SonarCloud

```yaml
# .github/workflows/sonar.yml
name: SonarCloud Analysis

on:
  push:
    branches: [ main, develop ]
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  sonarcloud:
    name: SonarCloud
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # ต้องการ full history สำหรับ blame
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run tests with coverage
        run: npm test -- --ci --coverage
      
      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        with:
          args: >
            -Dsonar.projectKey=myorg_myproject
            -Dsonar.organization=myorg
            -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
```

### 6.3 Quality Gate Configuration

```yaml
# sonar-project.properties
# Quality Gate Conditions
# (ตั้งค่าใน SonarCloud UI หรือ sonar-project.properties)

# ต้องผ่าน:
# Coverage >= 80%
# Duplicated Lines <= 3%
# Code Smells Rating = A
# Security Rating = A
# Reliability Rating = A
# Maintainability Rating = A
```

---

## 7. Code Smell Detection

### 7.1 Code Smells ที่พบบ่อย

**Long Methods:**
```javascript
// ❌ Code Smell: Long Method (50+ lines)
function processOrder(order) {
  // 1. Validate order
  if (!order.customerId) throw new Error('No customer');
  if (!order.items?.length) throw new Error('No items');
  // ... 10 more validation lines
  
  // 2. Calculate prices
  let subtotal = 0;
  for (const item of order.items) {
    subtotal += item.price * item.quantity;
  }
  // ... 10 more calculation lines
  
  // 3. Apply discounts
  // ... 15 lines
  
  // 4. Save to DB
  // ... 10 lines
}

// ✓ Refactored: Extract Methods
function processOrder(order) {
  validateOrder(order);
  const pricing = calculatePricing(order.items, order.discountCode);
  return saveOrder({ ...order, ...pricing });
}

function validateOrder(order) { /* ... */ }
function calculatePricing(items, discountCode) { /* ... */ }
function saveOrder(orderData) { /* ... */ }
```

**Feature Envy:**
```javascript
// ❌ Code Smell: Feature Envy
class OrderProcessor {
  processShipping(order) {
    // ใช้ข้อมูลของ order มากเกินไป
    const weight = order.items.reduce((sum, item) => sum + item.weight * item.quantity, 0);
    const volume = order.items.reduce((sum, item) => sum + item.volume * item.quantity, 0);
    const distance = calculateDistance(order.address.zipCode, WAREHOUSE_ZIP);
    return calculateShippingCost(weight, volume, distance);
  }
}

// ✓ Refactored: ย้าย logic ไปที่ Order class
class Order {
  getTotalWeight() {
    return this.items.reduce((sum, item) => sum + item.weight * item.quantity, 0);
  }
  
  getTotalVolume() {
    return this.items.reduce((sum, item) => sum + item.volume * item.quantity, 0);
  }
  
  getShippingAddress() {
    return this.address;
  }
}

class ShippingCalculator {
  calculate(order) {
    const distance = this.getDistance(order.getShippingAddress().zipCode);
    return this.computeCost(order.getTotalWeight(), order.getTotalVolume(), distance);
  }
}
```

**Data Clumps:**
```python
# ❌ Code Smell: Data Clumps
def create_address(street, city, province, postal_code, country):
    pass

def validate_address(street, city, province, postal_code, country):
    pass

def format_address(street, city, province, postal_code, country):
    pass

# ✓ Refactored: Extract to dataclass
from dataclasses import dataclass

@dataclass
class Address:
    street: str
    city: str
    province: str
    postal_code: str
    country: str
    
    def is_valid(self) -> bool:
        return bool(self.street and self.city and self.postal_code)
    
    def format(self) -> str:
        return f"{self.street}, {self.city} {self.postal_code}"

def create_address(address: Address):
    pass
```

**God Object:**
```javascript
// ❌ Code Smell: God Object (ทำทุกอย่าง)
class ApplicationManager {
  handleUserLogin() { /* ... */ }
  validateCreditCard() { /* ... */ }
  sendEmail() { /* ... */ }
  processPayment() { /* ... */ }
  generateReport() { /* ... */ }
  manageInventory() { /* ... */ }
  calculateShipping() { /* ... */ }
  // ...50 more methods
}

// ✓ Refactored: Single Responsibility Principle
class AuthService { handleUserLogin() {} }
class PaymentService { validateCreditCard(); processPayment() {} }
class EmailService { sendEmail() {} }
class ReportService { generateReport() {} }
class InventoryService { manageInventory() {} }
class ShippingService { calculateShipping() {} }
```

---

## 8. Complexity Metrics

### 8.1 Cyclomatic Complexity

Cyclomatic Complexity วัดจำนวน paths ที่โค้ดสามารถเดินได้

```javascript
// Complexity = 1 (no branches)
function greet(name) {
  return `Hello, ${name}`;  // CC = 1
}

// Complexity = 4
function classify(score) {          // +1 (base)
  if (score >= 90) {                // +1 (if)
    return 'A';
  } else if (score >= 80) {         // +1 (else if)
    return 'B';
  } else if (score >= 70) {         // +1 (else if)
    return 'C';
  } else {
    return 'F';
  }
}  // CC = 4

// Complexity = 10 (maximum recommended)
function processTransaction(tx) {        // +1
  if (!tx.amount || tx.amount <= 0) {   // +1
    throw new Error('Invalid amount');
  }
  
  if (tx.type === 'debit') {             // +1
    if (account.balance < tx.amount) {   // +1
      if (account.hasOverdraft) {        // +1
        // overdraft processing
      } else {
        throw new Error('Insufficient');
      }
    }
  } else if (tx.type === 'credit') {     // +1
    // credit processing
  } else if (tx.type === 'transfer') {   // +1
    if (!tx.targetAccount) {             // +1
      throw new Error('No target');
    }
    if (tx.amount > MAX_TRANSFER) {      // +1
      throw new Error('Exceeds limit');
    }
  }
}  // CC = 10
```

### 8.2 Cognitive Complexity

Cognitive Complexity วัด "ความยากในการอ่านและเข้าใจ" ซึ่งแม่นยำกว่า Cyclomatic

```javascript
// Cognitive Complexity ต่ำ - อ่านง่าย
function getActiveUsers(users) {
  return users.filter(user => user.isActive);  // CC = 1
}

// Cognitive Complexity สูง - อ่านยาก
function processNestedData(data) {
  for (const category of data.categories) {  // +1
    for (const item of category.items) {      // +2 (nested)
      if (item.isActive) {                    // +3 (nested deeper)
        if (item.hasDiscount) {               // +4 (even deeper)
          try {
            applyDiscount(item);
          } catch (err) {                     // +2
            if (err instanceof DiscountError) { // +3
              handleDiscountError(err);
            }
          }
        }
      }
    }
  }
}  // Total Cognitive Complexity ≈ 15+
```

### 8.3 Measure Complexity ด้วย Tools

**JavaScript (ESLint):**
```javascript
// .eslintrc.js
rules: {
  'complexity': ['error', { max: 10 }],
  // Plugin: eslint-plugin-sonarjs
  'sonarjs/cognitive-complexity': ['error', 15],
}
```

**Python:**
```bash
# ติดตั้ง
pip install radon

# Cyclomatic Complexity
radon cc src/ -s -n B  # แสดงเฉพาะ grade B ขึ้นไป

# Maintainability Index
radon mi src/ -s

# Halstead Metrics
radon hal src/
```

---

## 9. Pre-commit Hooks ด้วย Husky

### 9.1 ติดตั้ง Husky

```bash
# ติดตั้ง
npm install --save-dev husky

# เปิดใช้งาน git hooks
npx husky init

# สร้าง pre-commit hook
echo "npx lint-staged" > .husky/pre-commit
```

### 9.2 .husky/pre-commit

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# 1. Run lint-staged (lint + format changed files)
npx lint-staged

# 2. Run tests (เฉพาะที่เกี่ยวข้อง)
# npx jest --findRelatedTests --passWithNoTests

# 3. Type check
# npm run typecheck
```

### 9.3 .husky/pre-push

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# ตรวจสอบก่อน push ไป remote
echo "Running pre-push checks..."

# Type check
npm run typecheck
if [ $? -ne 0 ]; then
  echo "❌ Type check failed. Push aborted."
  exit 1
fi

# Run all tests
npm test -- --passWithNoTests --ci
if [ $? -ne 0 ]; then
  echo "❌ Tests failed. Push aborted."
  exit 1
fi

echo "✓ All checks passed!"
```

### 9.4 .husky/commit-msg

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

# Validate commit message format (Conventional Commits)
npx --no-install commitlint --edit "$1"
```

**commitlint.config.js:**
```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    // Type ที่ allowed
    'type-enum': [
      2, 'always',
      [
        'feat',    // New feature
        'fix',     // Bug fix
        'docs',    // Documentation
        'style',   // Formatting
        'refactor', // Refactoring
        'test',    // Tests
        'chore',   // Build/tools
        'perf',    // Performance
        'ci',      // CI/CD
        'revert',  // Revert changes
      ],
    ],
    'subject-max-length': [2, 'always', 72],
    'subject-case': [2, 'always', 'lower-case'],
  },
};
```

### 9.5 Python pre-commit Hooks

```bash
pip install pre-commit
```

**.pre-commit-config.yaml:**
```yaml
# .pre-commit-config.yaml
repos:
  # ── General ────────────────────────────────────────────────────────────────
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-toml
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: debug-statements    # ไม่ให้ commit pdb/breakpoint
      - id: mixed-line-ending
  
  # ── Python ─────────────────────────────────────────────────────────────────
  - repo: https://github.com/psf/black
    rev: 23.12.0
    hooks:
      - id: black
        language_version: python3.11
  
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.1.9
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
  
  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.8.0
    hooks:
      - id: mypy
        additional_dependencies: [types-requests, types-PyYAML]
  
  # ── JavaScript ─────────────────────────────────────────────────────────────
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.56.0
    hooks:
      - id: eslint
        files: \.[jt]sx?$
        types: [file]
        additional_dependencies:
          - eslint@8.56.0
          - eslint-config-prettier@9.1.0
  
  # ── Security ───────────────────────────────────────────────────────────────
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.4.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
```

```bash
# ติดตั้ง hooks
pre-commit install

# Run manually ทุกไฟล์
pre-commit run --all-files

# Update hook versions
pre-commit autoupdate
```

---

## 10. lint-staged

### 10.1 ติดตั้ง lint-staged

```bash
npm install --save-dev lint-staged
```

### 10.2 Configuration

**package.json:**
```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,yaml,yml,md}": [
      "prettier --write"
    ],
    "*.css": [
      "prettier --write"
    ],
    "*.py": [
      "black",
      "ruff check --fix"
    ]
  }
}
```

**.lintstagedrc.js (ขั้นสูง):**
```javascript
// .lintstagedrc.js
module.exports = {
  // TypeScript: lint, format, type check
  '*.{ts,tsx}': [
    'eslint --fix',
    'prettier --write',
    // Type check ทั้ง project (ไม่ใช่แค่ changed files)
    () => 'tsc --noEmit',
  ],
  
  // JavaScript
  '*.{js,jsx}': [
    'eslint --fix',
    'prettier --write',
  ],
  
  // Tests: เฉพาะ format
  '*.test.{js,ts}': [
    'prettier --write',
  ],
  
  // Python
  '*.py': async (filenames) => {
    const files = filenames.join(' ');
    return [
      `black ${files}`,
      `ruff check --fix ${files}`,
      `mypy ${files} --ignore-missing-imports`,
    ];
  },
  
  // JSON/YAML
  '*.{json,yaml,yml}': [
    'prettier --write',
  ],
  
  // Markdown: lint
  '*.md': [
    'prettier --write',
    'markdownlint --fix',
  ],
};
```

### 10.3 ทดสอบ lint-staged

```bash
# Run manually บน staged files
npx lint-staged

# Debug mode
npx lint-staged --debug

# Dry run (แสดงว่าจะทำอะไร ไม่ execute จริง)
npx lint-staged --dry-run
```

---

## 11. GitHub Actions Lint Workflow

### 11.1 Comprehensive Lint Workflow

```yaml
# .github/workflows/code-quality.yml
name: Code Quality

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # ── JavaScript/TypeScript Lint ─────────────────────────────────────────────
  
  lint-javascript:
    name: ESLint & Prettier (JS/TS)
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run ESLint
        run: |
          npx eslint src --ext .ts,.tsx,.js,.jsx \
            --format=@microsoft/eslint-formatter-sarif \
            --output-file eslint-results.sarif
        continue-on-error: true
      
      - name: Upload ESLint results to GitHub
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: eslint-results.sarif
          wait-for-processing: true
      
      - name: Check Prettier formatting
        run: npx prettier --check "src/**/*.{ts,tsx,js,jsx,json,css,md}"
  
  # ── Python Lint ───────────────────────────────────────────────────────────
  
  lint-python:
    name: Ruff & Black (Python)
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      
      - run: pip install ruff black mypy
      
      - name: Run Ruff linting
        run: ruff check src/ --output-format=github
      
      - name: Check Black formatting
        run: black --check src/
      
      - name: Run mypy type checking
        run: mypy src/ --ignore-missing-imports
  
  # ── Security Lint ─────────────────────────────────────────────────────────
  
  security-lint:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: npm audit
        run: npm audit --audit-level=high
      
      - name: Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
  
  # ── SonarCloud ────────────────────────────────────────────────────────────
  
  sonarcloud:
    name: SonarCloud Analysis
    runs-on: ubuntu-latest
    needs: [lint-javascript, lint-python]
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci && npm test -- --ci --coverage
      
      - uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### 11.2 Reviewdog — Inline PR Comments

```yaml
# .github/workflows/reviewdog.yml
name: Reviewdog

on:
  pull_request:

jobs:
  eslint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      
      - uses: reviewdog/action-eslint@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review
          eslint_flags: 'src --ext .ts,.tsx,.js'
  
  actionlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: reviewdog/action-actionlint@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          reporter: github-pr-review
```

---

## 12. Code Review Checklist

### 12.1 Code Review Process

```
PR Submitted
    │
    ▼
Automated Checks (CI)
├── Lint ✓/✗
├── Tests ✓/✗
├── Coverage ✓/✗
└── Security ✓/✗
    │
    ▼
Human Review
├── Code Correctness
├── Code Quality
├── Security
├── Performance
└── Design
    │
    ▼
Approved → Merge
```

### 12.2 Code Review Checklist ครบถ้วน

```markdown
## Code Review Checklist

### ✅ Correctness
- [ ] โค้ดทำงานตรงตาม requirements
- [ ] Logic ถูกต้องและไม่มี off-by-one errors
- [ ] Null/undefined cases ถูกจัดการ
- [ ] Error cases ถูกจัดการอย่างเหมาะสม
- [ ] Boundary conditions ถูกทดสอบ

### ✅ Code Quality
- [ ] ชื่อตัวแปร/function/class สื่อความหมาย
- [ ] ไม่มี magic numbers/strings (ใช้ constants)
- [ ] ไม่มี code duplication (DRY principle)
- [ ] Function ทำแค่อย่างเดียว (SRP)
- [ ] ความยาว function ไม่เกิน 30-50 บรรทัด
- [ ] Complexity ไม่สูงเกินไป (≤ 10)
- [ ] Comments อธิบาย "ทำไม" ไม่ใช่ "อะไร"

### ✅ Tests
- [ ] มี tests สำหรับ business logic
- [ ] Tests ครอบคลุม happy path, edge cases, error cases
- [ ] Test names บอกว่าทดสอบอะไร
- [ ] Tests เป็น independent จากกัน
- [ ] Coverage ไม่ลดลงจาก PR นี้

### ✅ Security
- [ ] Input validation ครบถ้วน
- [ ] ไม่มี SQL injection vulnerabilities
- [ ] ไม่มี XSS vulnerabilities
- [ ] Sensitive data ไม่ถูก log
- [ ] Authentication/Authorization ถูกต้อง
- [ ] ไม่มี secrets/credentials ใน code

### ✅ Performance
- [ ] ไม่มี N+1 query problems
- [ ] การใช้ cache เหมาะสม
- [ ] ไม่มี unnecessary computation
- [ ] Database queries มี indexes
- [ ] ไม่มี memory leaks

### ✅ Compatibility
- [ ] API changes backward compatible
- [ ] Database migrations safe
- [ ] ไม่มี breaking changes ที่ไม่ได้ประกาศ

### ✅ Documentation
- [ ] README updated ถ้าจำเป็น
- [ ] API docs updated
- [ ] Comments มีสำหรับ complex logic

### ✅ Operations
- [ ] Logging เหมาะสม (ไม่น้อยหรือมากเกินไป)
- [ ] Error messages ช่วย debug
- [ ] Feature flags ถูกใช้สำหรับ risky changes
- [ ] Rollback plan มีถ้าจำเป็น
```

### 12.3 ตัวอย่าง Code Review Comments ที่ดี

```javascript
// ❌ Comment ที่ไม่ดี
// ❌ "This is wrong"
// ❌ "Why did you write this?"
// ❌ "You should know better"

// ✓ Comment ที่ดี
// ✓ "การใช้ for loop นี้อาจมี O(n²) complexity เมื่อ data เยอะ 
//    ลองพิจารณาใช้ Map เพื่อลดเป็น O(n): ..."
// ✓ "Nit: ชื่อตัวแปร 'd' ไม่ค่อยชัดเจน อาจใช้ 'discountAmount' แทน?"
// ✓ "Question: มีเหตุผลไหมที่เลือกใช้ async/await ตรงนี้แทน Promise.all? 
//    อาจ parallel ได้ถ้า..."
// ✓ "Suggestion: ส่วนนี้ดูซ้ำกับ UserService.findById() - อาจ extract method ได้?"
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Setup ESLint + Prettier

**โจทย์:** Setup code quality tools สำหรับ Node.js project

**ขั้นตอน:**
1. สร้าง project ใหม่: `mkdir quality-demo && cd quality-demo && npm init -y`
2. ติดตั้ง ESLint + Prettier + Husky + lint-staged
3. เขียน `.eslintrc.js` ที่ครอบคลุม
4. เขียน `.prettierrc.js`
5. ตั้งค่า Husky pre-commit hook
6. ตั้งค่า lint-staged

**Code ที่ต้องแก้:**
```javascript
// bad-code.js (ใช้ lint แล้วแก้)
var x = 1
const y = 2
function   add(a,b){
return a+b
}
const unused = "never used"
const result = add(x,y)
console.log(result)
if (result == 3) {
console.log("correct")
}
```

### แบบฝึกหัดที่ 2: Python Code Quality

**โจทย์:** Setup Ruff + Black + mypy สำหรับ Python project และแก้ code ต่อไปนี้:

```python
# bad_code.py
import os,sys
from typing import *

def   process_users(data,flag,extra,another,more):
    result=[]
    for i in range(len(data)):
        user=data[i]
        if user['active']==True:
            if user['age']>=18:
                if user['verified']==True:
                    result.append({'name':user['name'],'email':user['email']})
    return result

class userManager:
    def __init__(self):
        self.Users=[]
    def addUser(self,name,email,age,phone,address,city):
        self.Users.append({'name':name,'email':email,'age':age})
    def getActiveUsers(self):
        return [u for u in self.Users if u.get('active')]
```

### แบบฝึกหัดที่ 3: GitHub Actions Lint Workflow

**โจทย์:** สร้าง workflow ที่:
1. Run ESLint สำหรับ JavaScript files
2. Run Black + Ruff สำหรับ Python files
3. Check Prettier formatting
4. Run SonarCloud (mock ถ้าไม่มี account)
5. Post inline comments ใน PR เมื่อพบ issues

---

## สรุป Part 09

ในบทนี้เราได้เรียนรู้:

1. **Code Quality คืออะไร** — Metrics, ความสำคัญใน CI/CD
2. **ESLint** — การตั้งค่า `.eslintrc.js`, plugins, rules
3. **Pylint/Flake8/Black/Ruff** — Tools สำหรับ Python
4. **Checkstyle** — Java code style checking
5. **Prettier** — Code formatting ที่ opinionated
6. **SonarQube/SonarCloud** — Code quality analysis
7. **Code Smells** — Long Methods, God Object, Feature Envy
8. **Complexity Metrics** — Cyclomatic, Cognitive Complexity
9. **Husky** — Git hooks automation
10. **lint-staged** — Run linters เฉพาะ staged files
11. **GitHub Actions** — Lint workflows, reviewdog
12. **Code Review** — Process และ checklist

**บทต่อไป:** Part 10 จะดู Build Automation ด้วย Makefile, npm scripts และ Gradle/Maven

---

*ปรับปรุงล่าสุด: 2024 | หลักสูตร CI/CD สำหรับนักพัฒนา*

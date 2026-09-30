# Part 08: Automated Testing ใน CI/CD

> **หลักสูตร CI/CD สำหรับนักพัฒนา** | ระดับ: ปานกลาง-สูง | เวลาเรียน: ~5 ชั่วโมง

---

## สารบัญ

1. [Automated Testing คืออะไร?](#1-automated-testing-คืออะไร)
2. [Unit Testing ด้วย Jest (JavaScript)](#2-unit-testing-ด้วย-jest)
3. [Unit Testing ด้วย pytest (Python)](#3-unit-testing-ด้วย-pytest)
4. [Unit Testing ด้วย JUnit (Java)](#4-unit-testing-ด้วย-junit)
5. [Integration Testing](#5-integration-testing)
6. [API Testing ด้วย Supertest](#6-api-testing-ด้วย-supertest)
7. [API Testing ด้วย requests (Python)](#7-api-testing-ด้วย-requests)
8. [Test Coverage Reports](#8-test-coverage-reports)
9. [GitHub Actions สำหรับ Tests](#9-github-actions-สำหรับ-tests)
10. [Parallel Test Execution](#10-parallel-test-execution)
11. [Test Caching](#11-test-caching)
12. [Flaky Tests](#12-flaky-tests)
13. [Test Reporting](#13-test-reporting)
14. [Codecov Integration](#14-codecov-integration)
15. [แบบฝึกหัด](#15-แบบฝึกหัด)

---

## 1. Automated Testing คืออะไร?

### 1.1 Automated Testing vs Manual Testing

| ด้าน | Manual Testing | Automated Testing |
|------|--------------|-----------------|
| **ความเร็ว** | ช้า (นาที-ชั่วโมง) | เร็ว (วินาที-นาที) |
| **ความแม่นยำ** | Human error ได้ | Consistent |
| **การทำซ้ำ** | เบื่อ, ข้ามขั้นตอน | ทำซ้ำได้ตลอด |
| **ค่าใช้จ่าย** | แพงในระยะยาว | ลงทุนครั้งแรกสูง แต่ถูกในระยะยาว |
| **Feedback** | ช้า | เร็ว (minutes หลัง commit) |
| **Coverage** | Limited | กว้าง |

### 1.2 Automated Testing ใน CI/CD Pipeline

```
Developer Commits Code
        │
        ▼
   GitHub Actions Triggered
        │
        ▼
   ┌────────────────────┐
   │  1. Install deps   │ npm install / pip install
   ├────────────────────┤
   │  2. Lint & Format  │ eslint, prettier
   ├────────────────────┤
   │  3. Unit Tests     │ jest, pytest, junit
   ├────────────────────┤
   │  4. Coverage Check │ > 80% required
   ├────────────────────┤
   │  5. Integration    │ with test DB
   │     Tests          │
   ├────────────────────┤
   │  6. Build          │ compile, bundle
   └────────────────────┘
        │
        ▼
   ✓ All Pass → Deploy to Staging
   ✗ Any Fail → Block Merge, Notify Developer
```

---

## 2. Unit Testing ด้วย Jest

### 2.1 ติดตั้งและ Configure Jest

```bash
# สร้าง project ใหม่
mkdir jest-demo && cd jest-demo
npm init -y

# ติดตั้ง Jest
npm install --save-dev jest

# สำหรับ TypeScript
npm install --save-dev jest ts-jest @types/jest typescript

# สำหรับ ES Modules
npm install --save-dev @babel/preset-env babel-jest
```

**jest.config.js — การตั้งค่าสมบูรณ์:**
```javascript
/** @type {import('jest').Config} */
const config = {
  // ประเภท tests ที่จะ run
  testMatch: [
    '**/__tests__/**/*.[jt]s?(x)',
    '**/?(*.)+(spec|test).[jt]s?(x)',
  ],
  
  // ไฟล์ที่ไม่รวมใน tests
  testPathIgnorePatterns: [
    '/node_modules/',
    '/dist/',
    '/build/',
  ],
  
  // สภาพแวดล้อมการทดสอบ
  testEnvironment: 'node',  // หรือ 'jsdom' สำหรับ browser
  
  // Coverage settings
  collectCoverageFrom: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.d.ts',
    '!src/index.{js,ts}',
    '!src/**/*.stories.{js,ts}',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html', 'json'],
  
  // Coverage thresholds (จะ fail ถ้าต่ำกว่า)
  coverageThreshold: {
    global: {
      branches: 75,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  
  // Setup files
  setupFilesAfterFramework: ['<rootDir>/jest.setup.js'],
  
  // Module aliases
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  
  // Timeout สำหรับแต่ละ test
  testTimeout: 10000,
  
  // Verbose output
  verbose: true,
  
  // Force clear mocks ก่อนแต่ละ test
  clearMocks: true,
  restoreMocks: true,
};

module.exports = config;
```

**jest.setup.js:**
```javascript
// jest.setup.js
// Setup global test utilities, custom matchers, etc.

// Custom matcher
expect.extend({
  toBeWithinRange(received, floor, ceiling) {
    const pass = received >= floor && received <= ceiling;
    if (pass) {
      return {
        message: () => `expected ${received} not to be within range [${floor}, ${ceiling}]`,
        pass: true,
      };
    } else {
      return {
        message: () => `expected ${received} to be within range [${floor}, ${ceiling}]`,
        pass: false,
      };
    }
  },
});

// Global timeout สำหรับ async tests
jest.setTimeout(30000);

// Suppress console.error ใน tests (optional)
beforeAll(() => {
  jest.spyOn(console, 'error').mockImplementation(() => {});
});

afterAll(() => {
  console.error.mockRestore();
});
```

### 2.2 Advanced Jest Features

**Testing Promises:**
```javascript
// src/data-fetcher.js
async function fetchUserData(userId) {
  if (!userId) throw new Error('userId is required');
  
  const response = await fetch(`/api/users/${userId}`);
  if (!response.ok) throw new Error(`HTTP error ${response.status}`);
  
  return response.json();
}

// src/data-fetcher.test.js
const { fetchUserData } = require('./data-fetcher');

// Mock global fetch
global.fetch = jest.fn();

describe('fetchUserData', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  test('fetches user data successfully', async () => {
    const mockUser = { id: 1, name: 'สมชาย' };
    
    global.fetch.mockResolvedValue({
      ok: true,
      json: jest.fn().mockResolvedValue(mockUser),
    });
    
    const result = await fetchUserData(1);
    
    expect(result).toEqual(mockUser);
    expect(global.fetch).toHaveBeenCalledWith('/api/users/1');
  });
  
  test('throws error when userId is missing', async () => {
    await expect(fetchUserData()).rejects.toThrow('userId is required');
    expect(global.fetch).not.toHaveBeenCalled();
  });
  
  test('throws error when HTTP response is not ok', async () => {
    global.fetch.mockResolvedValue({
      ok: false,
      status: 404,
    });
    
    await expect(fetchUserData(999)).rejects.toThrow('HTTP error 404');
  });
});
```

**Mocking Modules:**
```javascript
// src/email-sender.js
const nodemailer = require('nodemailer');

async function sendEmail(to, subject, body) {
  const transporter = nodemailer.createTransporter({
    host: process.env.SMTP_HOST,
    port: process.env.SMTP_PORT,
  });
  
  return transporter.sendMail({ from: 'noreply@app.com', to, subject, html: body });
}

module.exports = { sendEmail };

// src/email-sender.test.js
// Mock ทั้ง module
jest.mock('nodemailer');
const nodemailer = require('nodemailer');
const { sendEmail } = require('./email-sender');

describe('sendEmail', () => {
  let mockSendMail;
  
  beforeEach(() => {
    mockSendMail = jest.fn().mockResolvedValue({ messageId: 'test-id' });
    nodemailer.createTransporter.mockReturnValue({
      sendMail: mockSendMail,
    });
  });
  
  test('sends email with correct parameters', async () => {
    await sendEmail('user@test.com', 'Subject', '<p>Body</p>');
    
    expect(mockSendMail).toHaveBeenCalledWith({
      from: 'noreply@app.com',
      to: 'user@test.com',
      subject: 'Subject',
      html: '<p>Body</p>',
    });
  });
  
  test('uses environment variables for SMTP config', async () => {
    process.env.SMTP_HOST = 'smtp.test.com';
    process.env.SMTP_PORT = '587';
    
    await sendEmail('user@test.com', 'Test', 'Body');
    
    expect(nodemailer.createTransporter).toHaveBeenCalledWith({
      host: 'smtp.test.com',
      port: '587',
    });
  });
});
```

**Snapshot Testing:**
```javascript
// src/formatters.js
function formatOrderSummary(order) {
  return {
    orderId: order.id,
    customerName: order.customer.name,
    itemCount: order.items.length,
    total: `฿${order.total.toFixed(2)}`,
    status: order.status.toUpperCase(),
    createdAt: new Date(order.createdAt).toLocaleDateString('th-TH'),
  };
}

// src/formatters.test.js
const { formatOrderSummary } = require('./formatters');

test('formats order summary correctly', () => {
  const order = {
    id: 'ORD-001',
    customer: { name: 'สมชาย มีทรัพย์' },
    items: [{ id: 1 }, { id: 2 }],
    total: 1250.50,
    status: 'confirmed',
    createdAt: '2024-01-15T10:00:00Z',
  };
  
  const result = formatOrderSummary(order);
  
  // Snapshot test - บันทึกผลครั้งแรก, เปรียบเทียบครั้งต่อไป
  expect(result).toMatchSnapshot();
});
```

### 2.3 Testing React Components ด้วย Jest + Testing Library

```bash
npm install --save-dev @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

```javascript
// src/components/LoginForm.jsx
import React, { useState } from 'react';

export function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  
  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!email || !password) {
      setError('กรุณากรอกข้อมูลให้ครบ');
      return;
    }
    try {
      await onSubmit({ email, password });
    } catch (err) {
      setError(err.message);
    }
  };
  
  return (
    <form onSubmit={handleSubmit} data-testid="login-form">
      <input
        type="email"
        placeholder="Email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        data-testid="email-input"
      />
      <input
        type="password"
        placeholder="Password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        data-testid="password-input"
      />
      {error && <p role="alert" data-testid="error-message">{error}</p>}
      <button type="submit" data-testid="submit-button">เข้าสู่ระบบ</button>
    </form>
  );
}

// src/components/LoginForm.test.jsx
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import '@testing-library/jest-dom';
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  const mockOnSubmit = jest.fn();
  
  beforeEach(() => {
    mockOnSubmit.mockClear();
  });
  
  test('renders login form with correct elements', () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    expect(screen.getByTestId('email-input')).toBeInTheDocument();
    expect(screen.getByTestId('password-input')).toBeInTheDocument();
    expect(screen.getByTestId('submit-button')).toHaveTextContent('เข้าสู่ระบบ');
  });
  
  test('shows error when submitting empty form', async () => {
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    fireEvent.click(screen.getByTestId('submit-button'));
    
    expect(await screen.findByTestId('error-message'))
      .toHaveTextContent('กรุณากรอกข้อมูลให้ครบ');
    expect(mockOnSubmit).not.toHaveBeenCalled();
  });
  
  test('calls onSubmit with credentials when form is valid', async () => {
    const user = userEvent.setup();
    mockOnSubmit.mockResolvedValue({ token: 'test-token' });
    
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByTestId('email-input'), 'test@example.com');
    await user.type(screen.getByTestId('password-input'), 'password123');
    await user.click(screen.getByTestId('submit-button'));
    
    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123',
      });
    });
  });
  
  test('shows API error message on login failure', async () => {
    const user = userEvent.setup();
    mockOnSubmit.mockRejectedValue(new Error('Email หรือ Password ไม่ถูกต้อง'));
    
    render(<LoginForm onSubmit={mockOnSubmit} />);
    
    await user.type(screen.getByTestId('email-input'), 'test@example.com');
    await user.type(screen.getByTestId('password-input'), 'wrong-pass');
    await user.click(screen.getByTestId('submit-button'));
    
    expect(await screen.findByRole('alert'))
      .toHaveTextContent('Email หรือ Password ไม่ถูกต้อง');
  });
});
```

---

## 3. Unit Testing ด้วย pytest

### 3.1 การตั้งค่า pytest

```bash
pip install pytest pytest-cov pytest-mock pytest-asyncio faker
```

**pyproject.toml:**
```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--verbose",
    "--tb=short",
    "--strict-markers",
    "-p", "no:warnings",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
asyncio_mode = "auto"
```

**conftest.py (shared fixtures):**
```python
# tests/conftest.py
import pytest
import asyncio
from unittest.mock import AsyncMock, MagicMock
from datetime import datetime


# ── Fixtures ──────────────────────────────────────────────────────────────────

@pytest.fixture
def sample_user():
    """สร้าง user data สำหรับ test"""
    return {
        "id": "user-001",
        "name": "สมชาย มีทรัพย์",
        "email": "somchai@test.com",
        "age": 30,
        "created_at": datetime(2024, 1, 15, 10, 0, 0),
    }


@pytest.fixture
def mock_db():
    """Mock database connection"""
    db = MagicMock()
    db.query = MagicMock(return_value=[])
    db.execute = MagicMock(return_value=None)
    return db


@pytest.fixture
def mock_email_service():
    """Mock email service"""
    service = MagicMock()
    service.send = AsyncMock(return_value={"message_id": "test-id"})
    return service


@pytest.fixture
def mock_cache():
    """Mock Redis cache"""
    cache = {}
    
    mock = MagicMock()
    mock.get = MagicMock(side_effect=lambda key: cache.get(key))
    mock.set = MagicMock(side_effect=lambda key, value, **kwargs: cache.update({key: value}))
    mock.delete = MagicMock(side_effect=lambda key: cache.pop(key, None))
    
    return mock


# ── Event loop สำหรับ async tests ─────────────────────────────────────────────

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for all async tests"""
    loop = asyncio.get_event_loop_policy().new_event_loop()
    yield loop
    loop.close()
```

### 3.2 Parametrize Tests

```python
# tests/test_validators.py
import pytest
from src.validators import validate_thai_phone, validate_email, validate_national_id


class TestThaiPhoneValidator:
    
    @pytest.mark.parametrize("phone,expected", [
        # Valid phones
        ("0891234567", True),
        ("0861234567", True),
        ("0991234567", True),
        ("+66891234567", True),
        ("66891234567", True),
        # Invalid phones
        ("1234567890", False),   # ไม่ขึ้นต้นด้วย 0 หรือ +66
        ("089123456", False),    # สั้นเกินไป
        ("08912345678", False),  # ยาวเกินไป
        ("089-123-4567", False), # มี dash
        ("", False),             # ว่างเปล่า
        (None, False),           # None
    ])
    def test_phone_validation(self, phone, expected):
        assert validate_thai_phone(phone) == expected
    
    def test_valid_format_starts_with_zero(self):
        """เบอร์โทรที่ขึ้นต้นด้วย 0 และมี 10 หลักถูกต้อง"""
        valid_phones = ["081", "082", "083", "084", "085", "086", "087", "088", "089"]
        for prefix in valid_phones:
            phone = f"{prefix}1234567"
            assert validate_thai_phone(phone), f"Phone {phone} should be valid"


class TestNationalIdValidator:
    
    @pytest.mark.parametrize("national_id", [
        "1234567890123",
        "9876543210987",
        "1000000000000",
    ])
    def test_valid_national_ids(self, national_id):
        assert validate_national_id(national_id) is True
    
    @pytest.mark.parametrize("invalid_id,error_message", [
        ("123", "too short"),
        ("12345678901234", "too long"),
        ("123456789012a", "non-numeric"),
        ("", "empty"),
    ])
    def test_invalid_national_ids_raise_error(self, invalid_id, error_message):
        with pytest.raises(ValueError, match=error_message):
            validate_national_id(invalid_id)
```

### 3.3 Fixtures ขั้นสูง

```python
# tests/test_user_service.py
import pytest
from unittest.mock import AsyncMock, patch, call
from src.services.user_service import UserService
from src.exceptions import UserNotFoundError, DuplicateEmailError


class TestUserService:
    
    @pytest.fixture
    def user_repo(self):
        from unittest.mock import MagicMock
        repo = MagicMock()
        repo.find_by_id = AsyncMock(return_value=None)
        repo.find_by_email = AsyncMock(return_value=None)
        repo.save = AsyncMock()
        repo.update = AsyncMock()
        repo.delete = AsyncMock()
        return repo
    
    @pytest.fixture
    def email_service(self):
        from unittest.mock import MagicMock
        service = MagicMock()
        service.send_welcome_email = AsyncMock(return_value=True)
        service.send_password_reset = AsyncMock(return_value=True)
        return service
    
    @pytest.fixture
    def service(self, user_repo, email_service):
        return UserService(
            user_repository=user_repo,
            email_service=email_service,
        )
    
    @pytest.fixture
    def existing_user(self):
        return {
            "id": "user-001",
            "name": "สมชาย",
            "email": "somchai@test.com",
            "password_hash": "hashed_password",
            "is_active": True,
        }
    
    # ── Registration Tests ────────────────────────────────────────────────────
    
    @pytest.mark.asyncio
    async def test_register_creates_new_user(self, service, user_repo, email_service):
        user_repo.find_by_email.return_value = None
        user_repo.save.return_value = {
            "id": "new-user-001",
            "email": "newuser@test.com",
        }
        
        result = await service.register("newuser@test.com", "SecurePass123!")
        
        assert result["id"] == "new-user-001"
        user_repo.save.assert_called_once()
        email_service.send_welcome_email.assert_called_once_with("newuser@test.com")
    
    @pytest.mark.asyncio
    async def test_register_raises_error_for_duplicate_email(self, service, user_repo, existing_user):
        user_repo.find_by_email.return_value = existing_user
        
        with pytest.raises(DuplicateEmailError):
            await service.register("somchai@test.com", "password123")
        
        user_repo.save.assert_not_called()
    
    @pytest.mark.asyncio
    async def test_register_hashes_password(self, service, user_repo):
        saved_user = None
        
        async def capture_save(user):
            nonlocal saved_user
            saved_user = user
            return {**user, "id": "new-id"}
        
        user_repo.find_by_email.return_value = None
        user_repo.save.side_effect = capture_save
        
        await service.register("test@test.com", "PlainTextPassword")
        
        assert saved_user is not None
        assert saved_user["password_hash"] != "PlainTextPassword"
        assert len(saved_user["password_hash"]) > 20  # hashed ยาวกว่า
    
    # ── Get User Tests ────────────────────────────────────────────────────────
    
    @pytest.mark.asyncio
    async def test_get_user_returns_user_data(self, service, user_repo, existing_user):
        user_repo.find_by_id.return_value = existing_user
        
        result = await service.get_user("user-001")
        
        assert result["email"] == "somchai@test.com"
        user_repo.find_by_id.assert_called_once_with("user-001")
    
    @pytest.mark.asyncio
    async def test_get_user_raises_error_when_not_found(self, service, user_repo):
        user_repo.find_by_id.return_value = None
        
        with pytest.raises(UserNotFoundError, match="user-999"):
            await service.get_user("user-999")
    
    @pytest.mark.asyncio
    async def test_get_user_does_not_return_password_hash(self, service, user_repo, existing_user):
        user_repo.find_by_id.return_value = existing_user
        
        result = await service.get_user("user-001")
        
        assert "password_hash" not in result
```

---

## 4. Unit Testing ด้วย JUnit

### 4.1 Setup Spring Boot Test

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 4.2 ตัวอย่าง JUnit 5 Tests

```java
// src/test/java/com/example/ProductServiceTest.java
package com.example;

import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Optional;
import java.math.BigDecimal;

import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("ProductService Tests")
class ProductServiceTest {
    
    @Mock
    private ProductRepository productRepository;
    
    @Mock
    private CategoryRepository categoryRepository;
    
    @InjectMocks
    private ProductService productService;
    
    private Product sampleProduct;
    
    @BeforeEach
    void setUp() {
        sampleProduct = new Product();
        sampleProduct.setId(1L);
        sampleProduct.setName("สินค้าทดสอบ");
        sampleProduct.setPrice(new BigDecimal("299.99"));
        sampleProduct.setStock(100);
    }
    
    @Test
    @DisplayName("findById returns product when exists")
    void findById_WhenExists_ReturnsProduct() {
        when(productRepository.findById(1L))
            .thenReturn(Optional.of(sampleProduct));
        
        Product result = productService.findById(1L);
        
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getName()).isEqualTo("สินค้าทดสอบ");
        verify(productRepository, times(1)).findById(1L);
    }
    
    @Test
    @DisplayName("findById throws exception when not found")
    void findById_WhenNotFound_ThrowsException() {
        when(productRepository.findById(999L))
            .thenReturn(Optional.empty());
        
        assertThatThrownBy(() -> productService.findById(999L))
            .isInstanceOf(ProductNotFoundException.class)
            .hasMessageContaining("999");
    }
    
    @Test
    @DisplayName("createProduct saves and returns new product")
    void createProduct_WithValidData_SavesAndReturns() {
        ProductDTO dto = new ProductDTO("สินค้าใหม่", new BigDecimal("150.00"), 50);
        Product savedProduct = new Product(1L, "สินค้าใหม่", new BigDecimal("150.00"), 50);
        
        when(productRepository.save(any(Product.class)))
            .thenReturn(savedProduct);
        
        Product result = productService.createProduct(dto);
        
        assertThat(result.getId()).isNotNull();
        assertThat(result.getName()).isEqualTo("สินค้าใหม่");
        
        ArgumentCaptor<Product> captor = ArgumentCaptor.forClass(Product.class);
        verify(productRepository).save(captor.capture());
        assertThat(captor.getValue().getPrice()).isEqualByComparingTo("150.00");
    }
    
    @Test
    @DisplayName("updateStock reduces stock correctly")
    void updateStock_WithValidQuantity_ReducesStock() {
        when(productRepository.findById(1L))
            .thenReturn(Optional.of(sampleProduct));
        when(productRepository.save(any()))
            .thenAnswer(i -> i.getArgument(0));
        
        productService.reduceStock(1L, 10);
        
        ArgumentCaptor<Product> captor = ArgumentCaptor.forClass(Product.class);
        verify(productRepository).save(captor.capture());
        assertThat(captor.getValue().getStock()).isEqualTo(90);
    }
    
    @Test
    @DisplayName("updateStock throws exception when insufficient")
    void updateStock_InsufficientStock_ThrowsException() {
        when(productRepository.findById(1L))
            .thenReturn(Optional.of(sampleProduct));
        
        assertThatThrownBy(() -> productService.reduceStock(1L, 200))
            .isInstanceOf(InsufficientStockException.class);
        
        verify(productRepository, never()).save(any());
    }
}
```

---

## 5. Integration Testing

### 5.1 Integration Tests ด้วย Node.js + Testcontainers

```bash
npm install --save-dev @testcontainers/postgresql pg
```

```javascript
// tests/integration/user-repository.integration.test.js
const { PostgreSqlContainer } = require('@testcontainers/postgresql');
const { Pool } = require('pg');
const { UserRepository } = require('../../src/repositories/user-repository');

describe('UserRepository Integration Tests', () => {
  let container;
  let pool;
  let repo;
  
  // เริ่ม PostgreSQL container ก่อน suite ทั้งหมด
  beforeAll(async () => {
    container = await new PostgreSqlContainer('postgres:16-alpine')
      .withDatabase('testdb')
      .withUsername('testuser')
      .withPassword('testpass')
      .start();
    
    pool = new Pool({
      connectionString: container.getConnectionUri(),
    });
    
    // Run migrations
    await pool.query(`
      CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        name VARCHAR(100) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
      )
    `);
    
    repo = new UserRepository(pool);
  }, 60000); // Timeout 60s สำหรับ start container
  
  // ล้างข้อมูลหลังแต่ละ test
  afterEach(async () => {
    await pool.query('DELETE FROM users');
  });
  
  // หยุด container หลังเสร็จสิ้น
  afterAll(async () => {
    await pool.end();
    await container.stop();
  });
  
  test('saves user to database', async () => {
    const user = await repo.save({
      name: 'สมชาย มีทรัพย์',
      email: 'somchai@test.com',
    });
    
    expect(user.id).toBeDefined();
    expect(user.name).toBe('สมชาย มีทรัพย์');
    
    // ตรวจสอบจาก DB จริง
    const { rows } = await pool.query('SELECT * FROM users WHERE id = $1', [user.id]);
    expect(rows[0].email).toBe('somchai@test.com');
  });
  
  test('finds user by email', async () => {
    await pool.query(
      'INSERT INTO users (name, email) VALUES ($1, $2)',
      ['สมหญิง', 'somying@test.com']
    );
    
    const user = await repo.findByEmail('somying@test.com');
    
    expect(user).toBeTruthy();
    expect(user.name).toBe('สมหญิง');
  });
  
  test('returns null for non-existent email', async () => {
    const user = await repo.findByEmail('notexist@test.com');
    expect(user).toBeNull();
  });
  
  test('throws error for duplicate email', async () => {
    await repo.save({ name: 'User 1', email: 'duplicate@test.com' });
    
    await expect(
      repo.save({ name: 'User 2', email: 'duplicate@test.com' })
    ).rejects.toThrow();
  });
});
```

---

## 6. API Testing ด้วย Supertest

### 6.1 ติดตั้งและ Setup

```bash
npm install --save-dev supertest
```

### 6.2 ตัวอย่าง Express API Tests

```javascript
// src/app.js
const express = require('express');
const { userRouter } = require('./routes/users');
const { authMiddleware } = require('./middleware/auth');

const app = express();
app.use(express.json());
app.use('/api/users', authMiddleware, userRouter);

module.exports = app;

// tests/api/users.api.test.js
const request = require('supertest');
const app = require('../../src/app');
const { db } = require('../../src/database');
const { generateToken } = require('../../src/auth');

describe('Users API', () => {
  let authToken;
  let adminToken;
  
  beforeAll(async () => {
    await db.migrate.latest();
    
    // สร้าง test users
    const [user] = await db('users').insert({
      name: 'Test User',
      email: 'testuser@test.com',
      role: 'user',
    }).returning('*');
    
    const [admin] = await db('users').insert({
      name: 'Admin User',
      email: 'admin@test.com',
      role: 'admin',
    }).returning('*');
    
    authToken = generateToken(user);
    adminToken = generateToken(admin);
  });
  
  afterEach(async () => {
    // ล้างข้อมูลที่สร้างใน tests (ยกเว้น test users)
    await db('users').where('email', 'like', '%created-in-test%').delete();
  });
  
  afterAll(async () => {
    await db('users').truncate();
    await db.destroy();
  });
  
  describe('GET /api/users', () => {
    test('returns list of users with valid auth', async () => {
      const response = await request(app)
        .get('/api/users')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(response.body).toHaveProperty('users');
      expect(Array.isArray(response.body.users)).toBe(true);
    });
    
    test('returns 401 without auth token', async () => {
      const response = await request(app)
        .get('/api/users')
        .expect(401);
      
      expect(response.body.error).toBe('Unauthorized');
    });
    
    test('supports pagination with page and limit params', async () => {
      const response = await request(app)
        .get('/api/users?page=1&limit=10')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(response.body).toHaveProperty('page', 1);
      expect(response.body).toHaveProperty('limit', 10);
      expect(response.body).toHaveProperty('total');
    });
  });
  
  describe('POST /api/users', () => {
    test('creates new user with valid data', async () => {
      const userData = {
        name: 'New User',
        email: 'newuser-created-in-test@test.com',
        password: 'SecurePass123!',
      };
      
      const response = await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .send(userData)
        .expect(201);
      
      expect(response.body.id).toBeDefined();
      expect(response.body.email).toBe(userData.email);
      expect(response.body.password).toBeUndefined(); // ไม่ควรส่ง password กลับ
    });
    
    test('returns 400 for invalid email', async () => {
      const response = await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .send({ name: 'Test', email: 'not-an-email', password: 'pass' })
        .expect(400);
      
      expect(response.body.errors).toBeDefined();
      expect(response.body.errors.email).toBeDefined();
    });
    
    test('returns 409 for duplicate email', async () => {
      const response = await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${adminToken}`)
        .send({
          name: 'Duplicate User',
          email: 'testuser@test.com', // email ที่มีอยู่แล้ว
          password: 'pass123',
        })
        .expect(409);
      
      expect(response.body.error).toContain('already exists');
    });
    
    test('returns 403 when non-admin tries to create user', async () => {
      await request(app)
        .post('/api/users')
        .set('Authorization', `Bearer ${authToken}`) // user token, not admin
        .send({ name: 'Test', email: 'test@test.com', password: 'pass' })
        .expect(403);
    });
  });
  
  describe('GET /api/users/:id', () => {
    test('returns user by ID', async () => {
      const response = await request(app)
        .get('/api/users/1')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      
      expect(response.body.id).toBeDefined();
    });
    
    test('returns 404 for non-existent user', async () => {
      await request(app)
        .get('/api/users/99999')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(404);
    });
  });
  
  describe('PUT /api/users/:id', () => {
    test('updates user name successfully', async () => {
      const response = await request(app)
        .put('/api/users/1')
        .set('Authorization', `Bearer ${adminToken}`)
        .send({ name: 'Updated Name' })
        .expect(200);
      
      expect(response.body.name).toBe('Updated Name');
    });
    
    test('ignores readonly fields in update', async () => {
      const response = await request(app)
        .put('/api/users/1')
        .set('Authorization', `Bearer ${adminToken}`)
        .send({ name: 'New Name', id: 99999, createdAt: '2020-01-01' })
        .expect(200);
      
      expect(response.body.id).toBe(1); // id ไม่เปลี่ยน
    });
  });
});
```

---

## 7. API Testing ด้วย requests (Python)

### 7.1 pytest + httpx (Async HTTP Client)

```bash
pip install pytest httpx pytest-asyncio
```

```python
# tests/api/test_product_api.py
import pytest
import httpx
from httpx import AsyncClient
from main import app  # FastAPI app


@pytest.fixture
async def client():
    """Async test client"""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac


@pytest.fixture
async def auth_headers(client):
    """ได้รับ auth headers สำหรับ test"""
    response = await client.post("/api/auth/login", json={
        "email": "testuser@test.com",
        "password": "TestPassword123!"
    })
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}


class TestProductAPI:
    
    @pytest.mark.asyncio
    async def test_list_products_returns_200(self, client, auth_headers):
        response = await client.get("/api/products", headers=auth_headers)
        
        assert response.status_code == 200
        data = response.json()
        assert "products" in data
        assert isinstance(data["products"], list)
    
    @pytest.mark.asyncio
    async def test_create_product_success(self, client, auth_headers):
        product_data = {
            "name": "สินค้าทดสอบ",
            "price": 299.99,
            "stock": 100,
            "category": "electronics"
        }
        
        response = await client.post(
            "/api/products",
            json=product_data,
            headers=auth_headers
        )
        
        assert response.status_code == 201
        created = response.json()
        assert created["name"] == "สินค้าทดสอบ"
        assert created["id"] is not None
        assert "price" in created
    
    @pytest.mark.asyncio
    async def test_create_product_invalid_price(self, client, auth_headers):
        response = await client.post(
            "/api/products",
            json={"name": "สินค้า", "price": -100, "stock": 10},
            headers=auth_headers
        )
        
        assert response.status_code == 422  # Unprocessable Entity
        errors = response.json()["detail"]
        price_errors = [e for e in errors if "price" in str(e)]
        assert len(price_errors) > 0
    
    @pytest.mark.asyncio
    async def test_get_product_not_found(self, client, auth_headers):
        response = await client.get(
            "/api/products/99999",
            headers=auth_headers
        )
        
        assert response.status_code == 404
        assert "not found" in response.json()["detail"].lower()
    
    @pytest.mark.asyncio
    async def test_search_products(self, client, auth_headers):
        response = await client.get(
            "/api/products?search=สินค้า&category=electronics&min_price=100",
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "products" in data
        assert "total" in data
        assert "page" in data
    
    @pytest.mark.asyncio
    async def test_unauthorized_access_returns_401(self, client):
        response = await client.get("/api/products")
        assert response.status_code == 401
    
    @pytest.mark.parametrize("endpoint", [
        "/api/products",
        "/api/orders",
        "/api/users",
    ])
    @pytest.mark.asyncio
    async def test_protected_endpoints_require_auth(self, client, endpoint):
        response = await client.get(endpoint)
        assert response.status_code == 401
```

---

## 8. Test Coverage Reports

### 8.1 Coverage ด้วย Jest

```bash
# Generate coverage report
npx jest --coverage --coverageReporters=text,lcov,html

# Coverage ใน CI (ไม่ต้องการ interactive)
CI=true npx jest --coverage
```

**coverage/index.html** จะมี report แบบกราฟิก

### 8.2 Coverage ด้วย pytest-cov

```bash
# Basic coverage
pytest --cov=src tests/

# Coverage with detailed report
pytest --cov=src --cov-report=term-missing --cov-report=html tests/

# Coverage threshold (fail ถ้าต่ำกว่า 80%)
pytest --cov=src --cov-fail-under=80 tests/
```

**.coveragerc:**
```ini
[run]
source = src
omit =
    src/migrations/*
    src/settings*.py
    */__init__.py
    */conftest.py

[report]
show_missing = True
skip_empty = True
precision = 2

[html]
directory = coverage_html_report

[xml]
output = coverage.xml
```

### 8.3 Coverage ด้วย JaCoCo (Java)

```xml
<!-- pom.xml -->
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
        <execution>
            <id>check</id>
            <goals>
                <goal>check</goal>
            </goals>
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
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.75</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

---

## 9. GitHub Actions สำหรับ Tests

### 9.1 Node.js Test Workflow

```yaml
# .github/workflows/test-nodejs.yml
name: Node.js Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x, 20.x, 22.x]
    
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
        run: npm test -- --ci --coverage --watchAll=false
        env:
          CI: true
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-node-${{ matrix.node-version }}
          fail_ci_if_error: true
```

### 9.2 Python Test Workflow

```yaml
# .github/workflows/test-python.yml
name: Python Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Python ${{ matrix.python-version }} Tests
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
      fail-fast: false  # ให้ test ทุก version แม้บางตัว fail
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run unit tests
        run: pytest tests/unit/ -v --tb=short -m "not integration"
      
      - name: Run integration tests
        run: pytest tests/integration/ -v --tb=short -m "integration"
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
      
      - name: Generate coverage report
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            --cov-fail-under=80
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage.xml
          flags: python-${{ matrix.python-version }}
```

### 9.3 Java Test Workflow

```yaml
# .github/workflows/test-java.yml
name: Java Tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    name: Java Tests
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Run tests
        run: mvn clean verify -Pcoverage
      
      - name: Upload JaCoCo coverage report
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: target/site/jacoco/jacoco.xml
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()  # Upload แม้ tests fail
        with:
          name: test-results
          path: target/surefire-reports/
          retention-days: 30
```

### 9.4 Multi-language Test Workflow

```yaml
# .github/workflows/test-all.yml
name: All Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # ── Unit Tests ────────────────────────────────────────────────────────────
  
  unit-tests-js:
    name: JS Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: 'frontend/package-lock.json'
      - run: cd frontend && npm ci
      - run: cd frontend && npm test -- --ci --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: js-coverage
          path: frontend/coverage/
  
  unit-tests-python:
    name: Python Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install -r backend/requirements-dev.txt
      - run: cd backend && pytest tests/unit/ --cov=src --cov-report=xml
      - uses: actions/upload-artifact@v4
        with:
          name: python-coverage
          path: backend/coverage.xml
  
  # ── Integration Tests ─────────────────────────────────────────────────────
  
  integration-tests:
    name: Integration Tests
    needs: [unit-tests-js, unit-tests-python]  # Run หลัง unit tests ผ่าน
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: --health-cmd pg_isready --health-interval 10s --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install -r backend/requirements-dev.txt
      - run: cd backend && pytest tests/integration/ -v
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
  
  # ── Coverage Report ───────────────────────────────────────────────────────
  
  coverage-report:
    name: Coverage Report
    needs: [unit-tests-js, unit-tests-python]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          name: js-coverage
          path: coverage/js/
      - uses: actions/download-artifact@v4
        with:
          name: python-coverage
          path: coverage/python/
      - uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: coverage/js/lcov.info,coverage/python/coverage.xml
```

---

## 10. Parallel Test Execution

### 10.1 Jest Parallel Tests

Jest รัน tests แบบ parallel ด้วย worker processes โดย default

```javascript
// jest.config.js
module.exports = {
  // จำนวน workers (default: OS CPU count - 1)
  maxWorkers: '50%',  // ใช้ 50% ของ CPUs
  // หรือ
  maxWorkers: 4,      // กำหนดจำนวน fixed
  
  // สำหรับ CI environment ที่มี resources จำกัด
  // maxWorkers: 2,
};
```

**สำหรับ tests ที่แชร์ resources:**
```javascript
// jest.config.js
module.exports = {
  projects: [
    {
      // Unit tests - parallel
      displayName: 'unit',
      testMatch: ['<rootDir>/src/**/*.unit.test.js'],
      maxWorkers: '50%',
    },
    {
      // Integration tests - sequential (ต่อ DB เดียวกัน)
      displayName: 'integration',
      testMatch: ['<rootDir>/tests/integration/**/*.test.js'],
      maxWorkers: 1,  // รันทีละตัว
    },
  ],
};
```

### 10.2 pytest-xdist (Python Parallel)

```bash
pip install pytest-xdist
```

```bash
# รัน 4 workers parallel
pytest -n 4 tests/

# รัน auto (ตาม CPU count)
pytest -n auto tests/

# รัน parallel เฉพาะ unit tests
pytest -n auto -m "unit" tests/
```

```python
# pytest.ini
[pytest]
addopts = -n auto  # default parallel สำหรับทุกครั้ง

# markers สำหรับแยก tests
markers =
    unit: unit tests (parallel)
    integration: integration tests (sequential)
    serial: must run sequentially
```

### 10.3 Parallel ใน GitHub Actions

```yaml
# .github/workflows/parallel-tests.yml
name: Parallel Tests

jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]  # แบ่งเป็น 4 shards
    
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      # รัน 1/4 ของ tests ต่อ runner
      - run: |
          npx jest \
            --shard=${{ matrix.shard }}/4 \
            --coverage \
            --ci
      
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-shard-${{ matrix.shard }}
          path: coverage/
  
  # Merge coverage จากทุก shard
  merge-coverage:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-shard-*
          merge-multiple: true
          path: coverage/
      - run: npx nyc merge coverage/ merged-coverage.json
      - uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

---

## 11. Test Caching

### 11.1 Dependency Caching

```yaml
# .github/workflows/cached-tests.yml
name: Cached Tests

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Cache node_modules
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'  # built-in cache ใน setup-node
      
      # Cache pip packages
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'  # built-in cache ใน setup-python
      
      # Cache Jest transform cache
      - name: Cache Jest
        uses: actions/cache@v4
        with:
          path: |
            node_modules/.cache/jest
            /tmp/jest_*
          key: jest-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-${{ hashFiles('**/*.js', '**/*.ts') }}
          restore-keys: |
            jest-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-
            jest-${{ runner.os }}-
      
      # Cache pytest cache
      - name: Cache pytest
        uses: actions/cache@v4
        with:
          path: |
            .pytest_cache
            **/__pycache__
          key: pytest-${{ runner.os }}-${{ hashFiles('**/requirements*.txt') }}-${{ hashFiles('**/*.py') }}
          restore-keys: |
            pytest-${{ runner.os }}-${{ hashFiles('**/requirements*.txt') }}-
      
      - run: npm ci
      - run: npm test -- --ci
      
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/ -v
```

### 11.2 Jest Cache Configuration

```javascript
// jest.config.js
module.exports = {
  // Cache location (สำหรับ CI)
  cacheDirectory: '/tmp/jest_cache',
  
  // Force clear cache เมื่อ test fail
  // (ใช้ CLI: jest --clearCache)
};
```

---

## 12. Flaky Tests

### 12.1 Flaky Test คืออะไร?

Flaky Test คือ test ที่บางครั้ง pass บางครั้ง fail โดยไม่มีการเปลี่ยนแปลงโค้ด

**สาเหตุหลัก:**
- **Race condition** — tests รัน async โดยไม่ wait อย่างถูกต้อง
- **Time dependency** — test ขึ้นอยู่กับเวลาปัจจุบัน
- **Random data** — ใช้ random values ที่อาจทำให้ assertions fail
- **External dependencies** — เรียก API จริงที่อาจ unstable
- **Test order dependency** — test ขึ้นอยู่กับ state ที่ test อื่น set
- **Resource leaks** — connections หรือ timeouts ค้างจาก test ก่อนหน้า

### 12.2 ตัวอย่างและการแก้

**Flaky: Race condition**
```javascript
// ❌ Flaky - race condition
test('sends notification after order', async () => {
  await createOrder();
  // หลังสร้าง order ต้องรอ notification แต่ไม่มีการ wait
  expect(notificationService.calls).toHaveLength(1);
});

// ✓ Fixed
test('sends notification after order', async () => {
  const notificationPromise = waitForNotification(); // wait สำหรับ notification
  await createOrder();
  await notificationPromise; // รอจน notification ส่ง
  
  expect(notificationService.calls).toHaveLength(1);
});
```

**Flaky: Time dependency**
```javascript
// ❌ Flaky - ขึ้นอยู่กับเวลาจริง
test('expires token after 1 hour', async () => {
  const token = createToken();
  
  // รอ 1 ชั่วโมง? ไม่ได้!
  await new Promise(r => setTimeout(r, 3600000));
  
  expect(token.isExpired()).toBe(true);
});

// ✓ Fixed - ใช้ fake timer
test('expires token after 1 hour', () => {
  jest.useFakeTimers();
  
  const token = createToken();
  
  // เร่งเวลา 1 ชั่วโมง
  jest.advanceTimersByTime(3600000);
  
  expect(token.isExpired()).toBe(true);
  
  jest.useRealTimers();
});
```

**Flaky: External API**
```javascript
// ❌ Flaky - เรียก API จริง
test('gets weather data', async () => {
  const weather = await weatherService.getWeather('Bangkok');
  expect(weather.city).toBe('Bangkok');
});

// ✓ Fixed - mock API call
test('gets weather data', async () => {
  global.fetch = jest.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ name: 'Bangkok', temp: 35 }),
  });
  
  const weather = await weatherService.getWeather('Bangkok');
  expect(weather.city).toBe('Bangkok');
});
```

### 12.3 Detection Strategy

```yaml
# .github/workflows/flaky-test-detection.yml
name: Flaky Test Detection

on:
  schedule:
    - cron: '0 2 * * *'  # Run ทุกคืน 2am

jobs:
  detect-flaky:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      
      # Run tests 3 ครั้ง เพื่อหา flaky
      - name: Run tests multiple times
        run: |
          for i in 1 2 3; do
            echo "=== Run $i ==="
            npx jest --ci --json --outputFile=results-$i.json || true
          done
      
      # เปรียบเทียบผล
      - name: Analyze flaky tests
        run: |
          node -e "
            const results = [1,2,3].map(i => 
              JSON.parse(require('fs').readFileSync('results-'+i+'.json'))
            );
            
            const allTests = {};
            results.forEach((run, runIdx) => {
              run.testResults.forEach(suite => {
                suite.testResults.forEach(test => {
                  const key = test.fullName;
                  if (!allTests[key]) allTests[key] = [];
                  allTests[key].push(test.status);
                });
              });
            });
            
            const flakyTests = Object.entries(allTests)
              .filter(([, statuses]) => {
                const unique = new Set(statuses);
                return unique.size > 1; // มีทั้ง pass และ fail
              });
            
            if (flakyTests.length > 0) {
              console.log('⚠️  Flaky Tests Detected:');
              flakyTests.forEach(([name, statuses]) => {
                console.log('  -', name, ':', statuses.join(', '));
              });
            } else {
              console.log('✓ No flaky tests detected');
            }
          "
```

---

## 13. Test Reporting

### 13.1 JUnit XML Report (Universal Format)

```javascript
// package.json
{
  "scripts": {
    "test:ci": "jest --ci --reporters=default --reporters=jest-junit"
  },
  "devDependencies": {
    "jest-junit": "^16.0.0"
  },
  "jest-junit": {
    "outputDirectory": "test-results",
    "outputName": "junit.xml",
    "classname": "{classname}",
    "title": "{title}",
    "ancestorSeparator": " > "
  }
}
```

### 13.2 HTML Test Report

```bash
# Jest HTML Report
npm install --save-dev jest-html-reporters
```

```javascript
// jest.config.js
module.exports = {
  reporters: [
    'default',
    [
      'jest-html-reporters',
      {
        publicPath: './reports',
        filename: 'test-report.html',
        openReport: false,
        pageTitle: 'Test Report',
        logoImgPath: './assets/logo.png',
        expand: true,
      },
    ],
  ],
};
```

### 13.3 GitHub Actions Test Summary

```yaml
# .github/workflows/test-with-report.yml
name: Tests with Report

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run tests
        run: npm test -- --ci --reporters=jest-junit
        env:
          JEST_JUNIT_OUTPUT_DIR: test-results
          JEST_JUNIT_OUTPUT_NAME: junit.xml
      
      # GitHub Test Summary (แสดงใน PR)
      - name: Test Summary
        uses: test-summary/action@v2
        with:
          paths: "test-results/junit.xml"
        if: always()
      
      # Upload artifacts
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: test-results/
          retention-days: 14
```

### 13.4 Allure Report

```bash
npm install --save-dev allure-jest allure-commandline
```

```javascript
// jest.config.js
module.exports = {
  reporters: [
    'default',
    ['allure-jest/reporter', { resultsDir: 'allure-results' }],
  ],
};
```

```yaml
# GitHub Actions with Allure
- name: Generate Allure report
  run: npx allure generate allure-results -o allure-report --clean
  if: always()

- name: Upload Allure report
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: allure-report
    path: allure-report/
```

---

## 14. Codecov Integration

### 14.1 ตั้งค่า Codecov

1. ไปที่ [codecov.io](https://codecov.io)
2. Login ด้วย GitHub account
3. Add repository
4. Copy CODECOV_TOKEN
5. เพิ่ม secret ใน GitHub: Settings > Secrets > CODECOV_TOKEN

### 14.2 codecov.yml

```yaml
# codecov.yml (ในroot ของ repo)
coverage:
  # ความต้องการขั้นต่ำ
  status:
    project:
      default:
        target: 80%        # coverage ต้องไม่ต่ำกว่า 80%
        threshold: 5%      # ยอมให้ลดได้ 5%
        base: auto
    patch:                 # สำหรับ code ที่เพิ่งเพิ่ม
      default:
        target: 70%
        threshold: 10%
  
  # ไม่นับ files เหล่านี้
  ignore:
    - "**/*.test.js"
    - "**/*.spec.js"
    - "**/node_modules/**"
    - "**/dist/**"
    - "coverage/**"
    - "scripts/**"

# Comment ใน PR
comment:
  layout: "reach,diff,flags,tree"
  behavior: default
  require_changes: true   # Comment เฉพาะเมื่อ coverage เปลี่ยน

# Flags สำหรับแยก coverage
flags:
  unit:
    paths:
      - src/
    carryforward: true
  integration:
    paths:
      - tests/integration/
    carryforward: false
```

### 14.3 GitHub Actions + Codecov ครบถ้วน

```yaml
# .github/workflows/test-and-coverage.yml
name: Tests & Coverage

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2  # ต้องการสำหรับ Codecov diff
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run tests with coverage
        run: npm test -- --ci --coverage
      
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          flags: unittests
          name: codecov-${{ github.sha }}
          fail_ci_if_error: true
          verbose: true
      
      # แสดง coverage summary ใน Actions log
      - name: Coverage Summary
        run: |
          echo "## Coverage Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          npx jest --coverage --coverageReporters=text-summary 2>&1 | \
            grep -A 10 "Coverage summary" >> $GITHUB_STEP_SUMMARY
```

---

## 15. แบบฝึกหัด

### แบบฝึกหัดที่ 1: เขียน API Tests ด้วย Supertest

**โจทย์:** มี Express API ดังนี้ เขียน integration tests ให้ครบ

```javascript
// src/routes/products.js
const express = require('express');
const router = express.Router();
const { productService } = require('../services/product-service');

router.get('/', async (req, res) => {
  const { category, page = 1, limit = 20 } = req.query;
  const products = await productService.list({ category, page, limit });
  res.json(products);
});

router.post('/', async (req, res) => {
  try {
    const product = await productService.create(req.body);
    res.status(201).json(product);
  } catch (err) {
    if (err.name === 'ValidationError') {
      return res.status(400).json({ error: err.message });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

router.delete('/:id', async (req, res) => {
  try {
    await productService.delete(req.params.id);
    res.status(204).send();
  } catch (err) {
    if (err.name === 'NotFoundError') {
      return res.status(404).json({ error: 'Product not found' });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

**สิ่งที่ต้องทำ:**
1. เขียน supertest tests สำหรับทุก endpoint
2. Test happy path และ error cases
3. ใช้ mock สำหรับ productService

### แบบฝึกหัดที่ 2: GitHub Actions Workflow

**โจทย์:** สร้าง workflow ที่:
1. รัน tests ทุกครั้งที่ push หรือ PR
2. Matrix testing กับ Node.js 18, 20, 22
3. Cache dependencies
4. Upload coverage ไป Codecov
5. Post test summary ใน PR
6. Fail ถ้า coverage ต่ำกว่า 75%

### แบบฝึกหัดที่ 3: แก้ Flaky Tests

**โจทย์:** test ต่อไปนี้ flaky เพราะอะไร และแก้อย่างไร?

```javascript
// test 1: flaky
test('creates order with timestamp', async () => {
  const order = await createOrder({ productId: 'P001' });
  const now = Date.now();
  
  expect(order.createdAt).toBeGreaterThan(now - 1000);
  expect(order.createdAt).toBeLessThan(now + 1000);
});

// test 2: flaky  
test('processes items in correct order', async () => {
  const items = ['A', 'B', 'C'];
  const processed = await Promise.all(items.map(processItem));
  
  expect(processed[0]).toBe('Processed: A');
  expect(processed[1]).toBe('Processed: B');
  expect(processed[2]).toBe('Processed: C');
});

// test 3: flaky
test('generates unique IDs', () => {
  const ids = new Set();
  for (let i = 0; i < 1000; i++) {
    ids.add(generateId());
  }
  expect(ids.size).toBe(1000);
});
```

---

## สรุป Part 08

ในบทนี้เราได้เรียนรู้:

1. **Automated Testing ใน CI/CD** — การ integrate tests เข้า pipeline
2. **Jest** — Advanced features: mocking modules, snapshot, React Testing Library
3. **pytest** — parametrize, fixtures, async tests, mocking
4. **JUnit** — Spring Boot tests, Mockito, AssertJ
5. **Integration Tests** — Testcontainers, real database testing
6. **API Testing** — Supertest, httpx สำหรับ REST API
7. **Coverage** — Jest, pytest-cov, JaCoCo configuration
8. **GitHub Actions** — Workflows สำหรับทุก language
9. **Parallel Tests** — sharding, matrix strategy
10. **Flaky Tests** — การ detect และแก้ไข
11. **Codecov** — setup, configuration, GitHub integration

**บทต่อไป:** Part 09 จะดู Code Quality, Linting และ Formatting เพื่อให้โค้ดมีมาตรฐาน

---

*ปรับปรุงล่าสุด: 2024 | หลักสูตร CI/CD สำหรับนักพัฒนา*

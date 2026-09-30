# Part 07: การทดสอบซอฟต์แวร์เบื้องต้น (Testing Fundamentals)

> **หลักสูตร CI/CD สำหรับนักพัฒนา** | ระดับ: ปานกลาง | เวลาเรียน: ~4 ชั่วโมง

---

## สารบัญ

1. [ทำไมต้องทดสอบซอฟต์แวร์?](#1-ทำไมต้องทดสอบซอฟต์แวร์)
2. [Test Pyramid](#2-test-pyramid)
3. [คำศัพท์ที่ต้องรู้ (Testing Terminology)](#3-testing-terminology)
4. [TDD - Test Driven Development](#4-tdd---test-driven-development)
5. [BDD - Behavior Driven Development](#5-bdd---behavior-driven-development)
6. [Test Doubles: Mock, Stub, Spy, Fake](#6-test-doubles-mock-stub-spy-fake)
7. [ตัวอย่าง Test Cases จริง (Node.js)](#7-ตัวอย่าง-test-cases-nodejs)
8. [ตัวอย่าง Test Cases จริง (Python)](#8-ตัวอย่าง-test-cases-python)
9. [ตัวอย่าง Test Cases จริง (Java)](#9-ตัวอย่าง-test-cases-java)
10. [Test Coverage](#10-test-coverage)
11. [มาตรฐานการเขียน Test](#11-มาตรฐานการเขียน-test)
12. [Test Naming Conventions](#12-test-naming-conventions)
13. [การจัดการ Test Suites](#13-การจัดการ-test-suites)
14. [แบบฝึกหัด](#14-แบบฝึกหัด)

---

## 1. ทำไมต้องทดสอบซอฟต์แวร์?

### 1.1 ปัญหาที่เกิดขึ้นเมื่อไม่มีการทดสอบ

ลองนึกภาพว่าคุณพัฒนาระบบ e-commerce ขนาดใหญ่โดยไม่มีการทดสอบอัตโนมัติ:

- **Bug ซ่อนตัวในโค้ด** — ฟังก์ชันคิดราคาสินค้ามีข้อผิดพลาดเล็กน้อย แต่ไม่มีใครสังเกตจนกว่าลูกค้าจะร้องเรียน
- **Regression Bug** — แก้ไขโค้ดส่วนหนึ่งแล้วพังโค้ดส่วนอื่นโดยไม่รู้ตัว
- **Confidence ต่ำ** — ทีมกลัวที่จะ deploy เพราะไม่แน่ใจว่าจะมีอะไรพัง
- **Debug ยาก** — เมื่อ bug เกิดขึ้น production ต้องใช้เวลานานในการหาสาเหตุ

### 1.2 ประโยชน์ของการทดสอบอัตโนมัติ

```
การทดสอบ = ตาข่ายนิรภัยสำหรับนักพัฒนา
```

**ประโยชน์หลัก:**

| ด้าน | ผลลัพธ์ |
|------|---------|
| **ความมั่นใจ** | Deploy ได้อย่างมั่นใจ รู้ว่าอะไรทำงานได้ |
| **เอกสาร** | Tests เป็นเอกสารที่มีชีวิต แสดงว่าโค้ดทำอะไร |
| **Design** | บังคับให้คิดถึง interface ก่อนเขียน implementation |
| **Refactoring** | แก้ไขโค้ดได้อย่างปลอดภัย |
| **ประหยัดเวลา** | ค้นหา bug ได้เร็วขึ้น แก้ได้ถูกจุด |
| **คุณภาพ** | โค้ดมีคุณภาพสูงขึ้น มีโครงสร้างดีขึ้น |

### 1.3 ต้นทุนของ Bug

ตาม NIST (National Institute of Standards and Technology) ต้นทุนในการแก้ bug เพิ่มขึ้นตามระยะที่พบ:

```
Requirements Phase:   $1
Design Phase:         $5
Coding Phase:         $10
Testing Phase:        $20
System Test Phase:    $50
Production Phase:     $200+
```

**ข้อสรุป:** การพบ bug ในระยะเริ่มต้นถูกกว่าการพบใน production ถึง 200 เท่า!

### 1.4 ตัวอย่างความเสียหายจากการไม่ทดสอบ

```javascript
// ตัวอย่างโค้ดที่ดูเหมือนถูกต้อง แต่มี bug
function calculateDiscount(price, discountPercent) {
  return price - (price * discountPercent / 100);
}

// ใช้งาน
console.log(calculateDiscount(100, 10));  // 90 ✓
console.log(calculateDiscount(100, 0));   // 100 ✓
console.log(calculateDiscount(100, 110)); // -10 ✗ ราคาติดลบ! Bug!

// ถ้ามี test:
test('should not allow discount over 100%', () => {
  expect(() => calculateDiscount(100, 110)).toThrow('Invalid discount');
});
// → Test fail จะแจ้งให้รู้ทันที
```

---

## 2. Test Pyramid

### 2.1 แนวคิด Test Pyramid

Test Pyramid คือโมเดลที่อธิบายสัดส่วนการทดสอบที่ดีในซอฟต์แวร์:

```
        /\
       /E2E\          ← น้อย, แพง, ช้า
      /------\
     /Integr-\        ← ปานกลาง
    /  ation  \
   /------------\
  /  Unit Tests  \    ← มาก, ถูก, เร็ว
 /________________\
```

### 2.2 Unit Tests (การทดสอบระดับหน่วย)

**คืออะไร:** ทดสอบฟังก์ชัน/คลาส/method เดียวในการแยกส่วน (isolation)

**ลักษณะ:**
- ทำงานเร็วมาก (milliseconds)
- ไม่มีการเชื่อมต่อ database, network, filesystem จริง
- ทำซ้ำได้ (reproducible)
- ควรมี 60-70% ของ tests ทั้งหมด

**ตัวอย่าง:**
```javascript
// unit ที่ทดสอบ: ฟังก์ชัน validateEmail
function validateEmail(email) {
  const regex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  return regex.test(email);
}

// unit test
describe('validateEmail', () => {
  test('returns true for valid email', () => {
    expect(validateEmail('user@example.com')).toBe(true);
  });
  
  test('returns false for email without @', () => {
    expect(validateEmail('userexample.com')).toBe(false);
  });
  
  test('returns false for empty string', () => {
    expect(validateEmail('')).toBe(false);
  });
});
```

### 2.3 Integration Tests (การทดสอบระดับการเชื่อมต่อ)

**คืออะไร:** ทดสอบการทำงานร่วมกันของหลาย component

**ลักษณะ:**
- ช้ากว่า unit tests
- อาจใช้ database จริง (หรือ in-memory)
- ทดสอบ API endpoints, database queries, service interactions
- ควรมี 20-30% ของ tests ทั้งหมด

**ตัวอย่าง:**
```javascript
// integration test: ทดสอบ API endpoint ที่ต่อ database
describe('POST /api/users', () => {
  beforeEach(async () => {
    await db.clear(); // ล้าง test database
  });
  
  test('creates user and returns 201', async () => {
    const response = await request(app)
      .post('/api/users')
      .send({ name: 'สมชาย', email: 'somchai@test.com' });
    
    expect(response.status).toBe(201);
    expect(response.body.name).toBe('สมชาย');
    
    // ตรวจสอบว่า save ลง database จริง
    const user = await User.findByEmail('somchai@test.com');
    expect(user).toBeTruthy();
  });
});
```

### 2.4 End-to-End Tests (E2E Tests)

**คืออะไร:** ทดสอบ user journey ตั้งแต่ต้นจนจบ เหมือนผู้ใช้จริง

**ลักษณะ:**
- ช้าที่สุด (หลาย seconds หรือหลาย minutes)
- ใช้ browser จริง (Playwright, Cypress, Selenium)
- ทดสอบ real environment
- ควรมี 5-10% ของ tests ทั้งหมด

**ตัวอย่าง:**
```javascript
// E2E test ด้วย Playwright
test('user can login and view dashboard', async ({ page }) => {
  await page.goto('https://myapp.com/login');
  
  await page.fill('[data-testid="email"]', 'user@test.com');
  await page.fill('[data-testid="password"]', 'password123');
  await page.click('[data-testid="login-button"]');
  
  await expect(page).toHaveURL('/dashboard');
  await expect(page.locator('h1')).toContainText('ยินดีต้อนรับ');
});
```

### 2.5 สัดส่วนที่แนะนำ

```
รวม 1000 tests ทั้งหมด:
├── Unit Tests:        700 tests  (70%) → ทำงาน: ~30 วินาที
├── Integration Tests: 200 tests  (20%) → ทำงาน: ~2 นาที
└── E2E Tests:         100 tests  (10%) → ทำงาน: ~10 นาที
```

### 2.6 Ice Cream Cone Anti-Pattern

สิ่งที่ **ไม่ควรทำ** คือ "Ice Cream Cone":

```
     /____\
    /  E2E  \     ← มากเกินไป! แพงมาก
   /----------\
  / Integration\  ← ปานกลาง
 /--------------\
/   Unit Tests   \ ← น้อยเกินไป!
```

ปัญหา: ถ้า E2E tests เยอะเกินไป:
- CI pipeline ช้ามาก
- หา bug ต้นสาเหตุยาก
- ค่าใช้จ่ายสูง

---

## 3. Testing Terminology

### 3.1 คำศัพท์พื้นฐาน

**Test Suite** — กลุ่มของ tests ที่เกี่ยวข้องกัน
```javascript
describe('UserService', () => {   // ← Test Suite
  // ...
});
```

**Test Case** — การทดสอบหนึ่งกรณี
```javascript
test('should create user successfully', () => {   // ← Test Case
  // ...
});
```

**Assertion** — การตรวจสอบผลลัพธ์
```javascript
expect(result).toBe(42);        // ← Assertion
expect(user.name).toEqual('สมชาย');
expect(errors).toHaveLength(0);
```

**Test Runner** — โปรแกรมที่ run tests
- JavaScript: Jest, Mocha, Vitest
- Python: pytest, unittest
- Java: JUnit, TestNG

**Test Fixture** — ข้อมูลหรือ state ที่ต้องเตรียมก่อน test

### 3.2 Test Lifecycle Hooks

```javascript
describe('Shopping Cart', () => {
  
  // วิ่งครั้งเดียวก่อน test suite ทั้งหมด
  beforeAll(async () => {
    await connectDatabase();
  });
  
  // วิ่งก่อน test แต่ละตัว
  beforeEach(() => {
    cart = new ShoppingCart();
  });
  
  // วิ่งหลัง test แต่ละตัว
  afterEach(() => {
    cart.clear();
  });
  
  // วิ่งครั้งเดียวหลัง test suite ทั้งหมด
  afterAll(async () => {
    await disconnectDatabase();
  });
  
  test('adds item to cart', () => {
    cart.add({ id: 1, price: 100 });
    expect(cart.items).toHaveLength(1);
  });
});
```

### 3.3 AAA Pattern (Arrange-Act-Assert)

รูปแบบมาตรฐานในการเขียน test:

```javascript
test('should calculate total price with tax', () => {
  // Arrange - เตรียมข้อมูล
  const items = [
    { price: 100, quantity: 2 },
    { price: 50, quantity: 1 },
  ];
  const taxRate = 0.07; // 7% VAT
  
  // Act - ทำงานที่ต้องการทดสอบ
  const total = calculateTotalWithTax(items, taxRate);
  
  // Assert - ตรวจสอบผลลัพธ์
  expect(total).toBe(267.50); // (200 + 50) * 1.07
});
```

### 3.4 Given-When-Then Pattern (BDD Style)

```javascript
test('given a full cart, when checkout, then order is created', async () => {
  // Given - สถานการณ์เริ่มต้น
  const cart = new Cart();
  cart.add({ productId: 'A1', quantity: 2 });
  const user = await createTestUser();
  
  // When - เหตุการณ์ที่เกิดขึ้น
  const order = await cart.checkout(user.id);
  
  // Then - ผลลัพธ์ที่คาดหวัง
  expect(order.status).toBe('pending');
  expect(order.items).toHaveLength(1);
  expect(order.userId).toBe(user.id);
});
```

### 3.5 Happy Path vs Edge Cases vs Error Cases

```javascript
describe('divide function', () => {
  // Happy Path - กรณีปกติที่ทุกอย่างถูกต้อง
  test('divides two positive numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });
  
  test('divides with decimal result', () => {
    expect(divide(10, 3)).toBeCloseTo(3.333);
  });
  
  // Edge Cases - กรณีขอบเขต
  test('returns 0 when dividing 0', () => {
    expect(divide(0, 5)).toBe(0);
  });
  
  test('handles negative numbers', () => {
    expect(divide(-10, 2)).toBe(-5);
  });
  
  // Error Cases - กรณีที่ต้องเกิด error
  test('throws error when dividing by zero', () => {
    expect(() => divide(10, 0)).toThrow('Division by zero');
  });
  
  test('throws error for non-numeric input', () => {
    expect(() => divide('10', 2)).toThrow('Invalid input');
  });
});
```

---

## 4. TDD - Test Driven Development

### 4.1 TDD คืออะไร?

TDD (Test Driven Development) คือกระบวนการพัฒนาซอฟต์แวร์ที่เขียน test ก่อน แล้วค่อยเขียน code

**วงจร Red-Green-Refactor:**

```
     ┌─────────────┐
     │   RED 🔴    │
     │  เขียน test │
     │  ที่ fail   │
     └──────┬──────┘
            │
            ▼
     ┌─────────────┐
     │  GREEN 🟢   │
     │ เขียน code  │
     │ ให้ test    │
     │  ผ่าน      │
     └──────┬──────┘
            │
            ▼
     ┌─────────────┐
     │ REFACTOR 🔵 │
     │  ปรับปรุง  │
     │   โค้ด     │
     └──────┬──────┘
            │
            └────────► วนซ้ำ
```

### 4.2 ตัวอย่าง TDD แบบ Step-by-Step

**โจทย์:** สร้างฟังก์ชัน `fibonacci(n)` ที่คืนค่า Fibonacci number

**Step 1: RED - เขียน test ที่ fail**
```javascript
// fibonacci.test.js
const { fibonacci } = require('./fibonacci');

test('fibonacci(0) returns 0', () => {
  expect(fibonacci(0)).toBe(0);
});
// ► Run test → FAIL (ยังไม่มี fibonacci.js)
```

**Step 2: GREEN - เขียน code น้อยสุดให้ผ่าน**
```javascript
// fibonacci.js
function fibonacci(n) {
  return 0; // code น้อยสุดที่ทำให้ test ผ่าน
}
module.exports = { fibonacci };
// ► Run test → PASS ✓
```

**Step 3: เพิ่ม test ใหม่**
```javascript
test('fibonacci(1) returns 1', () => {
  expect(fibonacci(1)).toBe(1);
});
// ► Run test → FAIL
```

**Step 4: แก้ code ให้ผ่าน**
```javascript
function fibonacci(n) {
  if (n === 0) return 0;
  return 1;
}
// ► Run test → PASS ✓
```

**Step 5: เพิ่ม test อีก**
```javascript
test('fibonacci(5) returns 5', () => {
  expect(fibonacci(5)).toBe(5);
});
test('fibonacci(10) returns 55', () => {
  expect(fibonacci(10)).toBe(55);
});
// ► Run test → FAIL
```

**Step 6: เขียน implementation จริง**
```javascript
function fibonacci(n) {
  if (n < 0) throw new Error('Input must be non-negative');
  if (n === 0) return 0;
  if (n === 1) return 1;
  return fibonacci(n - 1) + fibonacci(n - 2);
}
// ► Run test → PASS ✓
```

**Step 7: REFACTOR - ปรับปรุงโดยไม่เปลี่ยน behavior**
```javascript
// ปรับให้ใช้ memoization เพื่อ performance
function fibonacci(n, memo = {}) {
  if (n < 0) throw new Error('Input must be non-negative');
  if (n in memo) return memo[n];
  if (n <= 1) return n;
  
  memo[n] = fibonacci(n - 1, memo) + fibonacci(n - 2, memo);
  return memo[n];
}
// ► Run test → PASS ✓ (ยังผ่านหลัง refactor)
```

### 4.3 ประโยชน์ของ TDD

1. **Design ดีขึ้น** — คิดถึง interface ก่อนเขียน implementation
2. **โค้ดมีคุณภาพสูง** — มี test coverage สูงโดยอัตโนมัติ
3. **Bug น้อยลง** — ค้นพบ bug ระหว่างพัฒนา ไม่ใช่ใน production
4. **Refactor ได้ปลอดภัย** — มี test เป็น safety net
5. **เอกสารที่แม่นยำ** — tests แสดงว่าโค้ดทำอะไร

### 4.4 เมื่อไหร่ไม่ควรใช้ TDD

- Prototyping หรือ proof of concept
- UI/UX exploration
- โค้ดที่ต้องเปลี่ยนแปลงบ่อยมากในช่วงต้น
- เวลาเร่งรีบมาก (แต่ให้กลับมาเพิ่ม tests ภายหลัง)

---

## 5. BDD - Behavior Driven Development

### 5.1 BDD คืออะไร?

BDD (Behavior Driven Development) คือการพัฒนาโดยเน้นที่ behavior ของระบบจาก perspective ของผู้ใช้

**ภาษา Gherkin (Feature files):**
```gherkin
Feature: ระบบเข้าสู่ระบบ

  Scenario: ผู้ใช้ login สำเร็จ
    Given ผู้ใช้อยู่ที่หน้า login
    When ผู้ใช้ป้อน email "user@test.com" และ password ที่ถูกต้อง
    Then ผู้ใช้ถูก redirect ไปหน้า dashboard
    And เห็นข้อความ "ยินดีต้อนรับ, สมชาย"

  Scenario: ผู้ใช้ login ล้มเหลวด้วย password ผิด
    Given ผู้ใช้อยู่ที่หน้า login
    When ผู้ใช้ป้อน email "user@test.com" และ password ผิด
    Then เห็นข้อความ error "Email หรือ Password ไม่ถูกต้อง"
    And ผู้ใช้ยังอยู่ที่หน้า login
```

### 5.2 BDD ด้วย Jest (JavaScript)

```javascript
// user-login.test.js
const { loginUser } = require('./auth');

describe('User Login Behavior', () => {
  
  describe('เมื่อ credentials ถูกต้อง', () => {
    let result;
    
    beforeEach(async () => {
      result = await loginUser('user@test.com', 'correctPassword');
    });
    
    it('ควรคืน token', () => {
      expect(result.token).toBeDefined();
    });
    
    it('ควรคืน user data', () => {
      expect(result.user.email).toBe('user@test.com');
    });
    
    it('ควรบันทึก last login time', async () => {
      const user = await User.findByEmail('user@test.com');
      expect(user.lastLoginAt).toBeDefined();
    });
  });
  
  describe('เมื่อ password ผิด', () => {
    it('ควร throw InvalidCredentialsError', async () => {
      await expect(
        loginUser('user@test.com', 'wrongPassword')
      ).rejects.toThrow('InvalidCredentials');
    });
    
    it('ควรบันทึก failed login attempt', async () => {
      try {
        await loginUser('user@test.com', 'wrongPassword');
      } catch (e) {}
      
      const user = await User.findByEmail('user@test.com');
      expect(user.failedLoginCount).toBe(1);
    });
  });
});
```

### 5.3 BDD ด้วย Cucumber.js

```javascript
// features/login.feature
Feature: User Authentication
  
  Scenario: Successful login
    Given I am on the login page
    When I enter "user@test.com" as email
    And I enter "password123" as password
    And I click the login button
    Then I should be redirected to "/dashboard"
    And I should see "Welcome back"

// step_definitions/login.steps.js
const { Given, When, Then } = require('@cucumber/cucumber');
const { expect } = require('@playwright/test');

Given('I am on the login page', async function () {
  await this.page.goto('/login');
});

When('I enter {string} as email', async function (email) {
  await this.page.fill('[name="email"]', email);
});

When('I enter {string} as password', async function (password) {
  await this.page.fill('[name="password"]', password);
});

When('I click the login button', async function () {
  await this.page.click('[type="submit"]');
});

Then('I should be redirected to {string}', async function (path) {
  await expect(this.page).toHaveURL(path);
});

Then('I should see {string}', async function (text) {
  await expect(this.page.locator('body')).toContainText(text);
});
```

---

## 6. Test Doubles: Mock, Stub, Spy, Fake

### 6.1 ทำไมต้องใช้ Test Doubles?

เมื่อทำ unit tests เราต้องแยก unit ออกจาก dependencies:

```
โค้ดจริง:
UserService → EmailService → SMTP Server
           → Database → MySQL Server
           → PaymentGateway → Stripe API

ถ้าทำ unit test ของ UserService ต้องจัดการ dependencies ทั้งหมดด้วย
→ ใช้ Test Doubles แทน dependencies จริง
```

### 6.2 Stub

Stub คือ object ที่ return ค่าที่กำหนดไว้ล่วงหน้า ใช้เมื่อต้องการควบคุมค่าที่ได้รับ

```javascript
// ตัวอย่าง: stub สำหรับ database
const userRepositoryStub = {
  findById: jest.fn().mockResolvedValue({
    id: 1,
    name: 'สมชาย',
    email: 'somchai@test.com',
  }),
};

test('getUserProfile returns user data', async () => {
  const service = new UserService(userRepositoryStub);
  const profile = await service.getUserProfile(1);
  
  expect(profile.name).toBe('สมชาย');
});
```

### 6.3 Mock

Mock คือ object ที่ตรวจสอบว่า function ถูกเรียกถูกต้องหรือเปล่า (เน้นการ verify interactions)

```javascript
// ตัวอย่าง: mock สำหรับ email service
const emailServiceMock = {
  sendWelcomeEmail: jest.fn().mockResolvedValue(true),
};

test('registers user and sends welcome email', async () => {
  const service = new UserService(emailServiceMock);
  
  await service.register({
    email: 'newuser@test.com',
    password: 'pass123',
  });
  
  // ตรวจสอบว่า sendWelcomeEmail ถูกเรียก
  expect(emailServiceMock.sendWelcomeEmail).toHaveBeenCalledTimes(1);
  expect(emailServiceMock.sendWelcomeEmail).toHaveBeenCalledWith({
    to: 'newuser@test.com',
    subject: 'ยินดีต้อนรับสู่ระบบ',
  });
});
```

### 6.4 Spy

Spy คือ wrapper ที่บันทึกการเรียก function จริง สามารถ verify ได้โดยไม่ต้องแทนที่ implementation จริง

```javascript
// ตัวอย่าง: spy ดู method ที่เรียกโดยไม่เปลี่ยน behavior
const emailService = new EmailService();
const sendEmailSpy = jest.spyOn(emailService, 'sendEmail');

test('sends email when user registers', async () => {
  await userService.register({ email: 'test@test.com' });
  
  expect(sendEmailSpy).toHaveBeenCalledTimes(1);
  // สังเกต: emailService.sendEmail จริงยังถูกเรียก
});

afterEach(() => {
  sendEmailSpy.mockRestore(); // คืน implementation จริง
});
```

### 6.5 Fake

Fake คือ implementation จริงที่ง่ายกว่า เหมาะสำหรับ testing เช่น in-memory database

```javascript
// ตัวอย่าง: Fake in-memory database
class FakeUserRepository {
  constructor() {
    this.users = new Map();
    this.nextId = 1;
  }
  
  async save(userData) {
    const user = { ...userData, id: this.nextId++ };
    this.users.set(user.id, user);
    return user;
  }
  
  async findById(id) {
    return this.users.get(id) || null;
  }
  
  async findByEmail(email) {
    for (const user of this.users.values()) {
      if (user.email === email) return user;
    }
    return null;
  }
  
  async clear() {
    this.users.clear();
    this.nextId = 1;
  }
}

// ใช้ใน test
const fakeRepo = new FakeUserRepository();
const userService = new UserService(fakeRepo);

test('saves user to repository', async () => {
  await userService.createUser({ name: 'สมหญิง', email: 'somying@test.com' });
  
  const user = await fakeRepo.findByEmail('somying@test.com');
  expect(user.name).toBe('สมหญิง');
});
```

### 6.6 สรุปความแตกต่าง

| ประเภท | จุดประสงค์ | ตรวจสอบอะไร |
|--------|-----------|-------------|
| **Stub** | ควบคุม input/output | ผลลัพธ์ (state) |
| **Mock** | ตรวจสอบ interactions | ว่า function ถูกเรียก |
| **Spy** | สังเกต calls โดยไม่เปลี่ยน behavior | ว่า function ถูกเรียก |
| **Fake** | implementation จำลองที่ทำงานได้จริง | ผลลัพธ์ (state) |

---

## 7. ตัวอย่าง Test Cases จริง (Node.js)

### 7.1 Setup Project

```bash
mkdir node-testing-demo && cd node-testing-demo
npm init -y
npm install --save-dev jest @types/jest
```

**package.json:**
```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": [
      "src/**/*.js",
      "!src/**/*.test.js"
    ]
  }
}
```

### 7.2 Business Logic Tests

```javascript
// src/shopping-cart.js
class ShoppingCart {
  constructor() {
    this.items = [];
    this.discountCode = null;
  }
  
  addItem(product) {
    if (!product.id || !product.price || product.price < 0) {
      throw new Error('Invalid product');
    }
    
    const existingItem = this.items.find(i => i.id === product.id);
    if (existingItem) {
      existingItem.quantity += (product.quantity || 1);
    } else {
      this.items.push({
        id: product.id,
        name: product.name,
        price: product.price,
        quantity: product.quantity || 1,
      });
    }
  }
  
  removeItem(productId) {
    const index = this.items.findIndex(i => i.id === productId);
    if (index === -1) throw new Error('Item not found');
    this.items.splice(index, 1);
  }
  
  applyDiscount(code) {
    const discounts = {
      'SAVE10': 0.10,
      'SAVE20': 0.20,
      'HALFOFF': 0.50,
    };
    if (!discounts[code]) throw new Error('Invalid discount code');
    this.discountCode = code;
    this.discountPercent = discounts[code];
  }
  
  getSubtotal() {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
  
  getDiscount() {
    if (!this.discountCode) return 0;
    return this.getSubtotal() * this.discountPercent;
  }
  
  getTotal() {
    return this.getSubtotal() - this.getDiscount();
  }
  
  clear() {
    this.items = [];
    this.discountCode = null;
  }
}

module.exports = ShoppingCart;
```

```javascript
// src/shopping-cart.test.js
const ShoppingCart = require('./shopping-cart');

describe('ShoppingCart', () => {
  let cart;
  
  beforeEach(() => {
    cart = new ShoppingCart();
  });
  
  describe('addItem', () => {
    test('adds new item to cart', () => {
      // Arrange
      const product = { id: 'P001', name: 'สินค้า A', price: 100, quantity: 1 };
      
      // Act
      cart.addItem(product);
      
      // Assert
      expect(cart.items).toHaveLength(1);
      expect(cart.items[0]).toMatchObject(product);
    });
    
    test('increases quantity when adding same product', () => {
      cart.addItem({ id: 'P001', name: 'สินค้า A', price: 100, quantity: 1 });
      cart.addItem({ id: 'P001', name: 'สินค้า A', price: 100, quantity: 2 });
      
      expect(cart.items).toHaveLength(1);
      expect(cart.items[0].quantity).toBe(3);
    });
    
    test('defaults quantity to 1 when not specified', () => {
      cart.addItem({ id: 'P001', name: 'สินค้า A', price: 100 });
      
      expect(cart.items[0].quantity).toBe(1);
    });
    
    test('throws error for invalid product (no id)', () => {
      expect(() => cart.addItem({ name: 'สินค้า A', price: 100 }))
        .toThrow('Invalid product');
    });
    
    test('throws error for negative price', () => {
      expect(() => cart.addItem({ id: 'P001', price: -50 }))
        .toThrow('Invalid product');
    });
  });
  
  describe('removeItem', () => {
    beforeEach(() => {
      cart.addItem({ id: 'P001', name: 'สินค้า A', price: 100 });
    });
    
    test('removes item from cart', () => {
      cart.removeItem('P001');
      expect(cart.items).toHaveLength(0);
    });
    
    test('throws error when removing non-existent item', () => {
      expect(() => cart.removeItem('NOTEXIST')).toThrow('Item not found');
    });
  });
  
  describe('getTotal', () => {
    beforeEach(() => {
      cart.addItem({ id: 'P001', name: 'สินค้า A', price: 100, quantity: 2 });
      cart.addItem({ id: 'P002', name: 'สินค้า B', price: 50, quantity: 1 });
    });
    
    test('calculates correct subtotal', () => {
      expect(cart.getSubtotal()).toBe(250); // 100*2 + 50*1
    });
    
    test('returns subtotal when no discount', () => {
      expect(cart.getTotal()).toBe(250);
    });
    
    test('applies 10% discount with SAVE10 code', () => {
      cart.applyDiscount('SAVE10');
      expect(cart.getTotal()).toBe(225); // 250 * 0.90
    });
    
    test('applies 50% discount with HALFOFF code', () => {
      cart.applyDiscount('HALFOFF');
      expect(cart.getTotal()).toBe(125); // 250 * 0.50
    });
    
    test('throws error for invalid discount code', () => {
      expect(() => cart.applyDiscount('INVALID')).toThrow('Invalid discount code');
    });
    
    test('returns 0 for empty cart', () => {
      const emptyCart = new ShoppingCart();
      expect(emptyCart.getTotal()).toBe(0);
    });
  });
});
```

### 7.3 Async Tests

```javascript
// src/user-service.js
class UserService {
  constructor(userRepository, emailService) {
    this.userRepository = userRepository;
    this.emailService = emailService;
  }
  
  async register(userData) {
    // Validate
    if (!userData.email || !userData.password) {
      throw new Error('Email and password are required');
    }
    
    // Check duplicate
    const existing = await this.userRepository.findByEmail(userData.email);
    if (existing) {
      throw new Error('Email already registered');
    }
    
    // Save
    const user = await this.userRepository.save({
      ...userData,
      createdAt: new Date(),
    });
    
    // Send welcome email
    await this.emailService.sendWelcomeEmail(user.email);
    
    return user;
  }
}

module.exports = UserService;
```

```javascript
// src/user-service.test.js
const UserService = require('./user-service');

describe('UserService.register', () => {
  let mockRepository;
  let mockEmailService;
  let service;
  
  beforeEach(() => {
    mockRepository = {
      findByEmail: jest.fn(),
      save: jest.fn(),
    };
    
    mockEmailService = {
      sendWelcomeEmail: jest.fn(),
    };
    
    service = new UserService(mockRepository, mockEmailService);
  });
  
  test('registers new user successfully', async () => {
    // Arrange
    const userData = { email: 'new@test.com', password: 'pass123' };
    mockRepository.findByEmail.mockResolvedValue(null); // ไม่มี user เก่า
    mockRepository.save.mockResolvedValue({ id: 1, ...userData });
    mockEmailService.sendWelcomeEmail.mockResolvedValue(true);
    
    // Act
    const user = await service.register(userData);
    
    // Assert
    expect(user.id).toBe(1);
    expect(mockRepository.save).toHaveBeenCalledWith(
      expect.objectContaining({ email: 'new@test.com' })
    );
    expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith('new@test.com');
  });
  
  test('throws error for duplicate email', async () => {
    // Arrange
    mockRepository.findByEmail.mockResolvedValue({ id: 1, email: 'exist@test.com' });
    
    // Act & Assert
    await expect(
      service.register({ email: 'exist@test.com', password: 'pass' })
    ).rejects.toThrow('Email already registered');
    
    // ต้องไม่ save และไม่ส่ง email
    expect(mockRepository.save).not.toHaveBeenCalled();
    expect(mockEmailService.sendWelcomeEmail).not.toHaveBeenCalled();
  });
  
  test('throws error when email is missing', async () => {
    await expect(
      service.register({ password: 'pass123' })
    ).rejects.toThrow('Email and password are required');
  });
});
```

---

## 8. ตัวอย่าง Test Cases จริง (Python)

### 8.1 Setup Project

```bash
mkdir python-testing-demo && cd python-testing-demo
python -m venv venv
source venv/bin/activate  # Linux/Mac
pip install pytest pytest-cov pytest-mock
```

**pytest.ini:**
```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
```

### 8.2 ตัวอย่าง Business Logic

```python
# src/order_processor.py
from dataclasses import dataclass, field
from typing import List, Optional
from enum import Enum
from datetime import datetime


class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"


@dataclass
class OrderItem:
    product_id: str
    product_name: str
    quantity: int
    unit_price: float
    
    @property
    def total_price(self):
        return self.quantity * self.unit_price


@dataclass
class Order:
    customer_id: str
    items: List[OrderItem] = field(default_factory=list)
    status: OrderStatus = OrderStatus.PENDING
    created_at: datetime = field(default_factory=datetime.now)
    discount_amount: float = 0.0
    
    def add_item(self, item: OrderItem):
        if item.quantity <= 0:
            raise ValueError("Quantity must be positive")
        if item.unit_price < 0:
            raise ValueError("Price cannot be negative")
        self.items.append(item)
    
    @property
    def subtotal(self):
        return sum(item.total_price for item in self.items)
    
    @property
    def total(self):
        return max(0, self.subtotal - self.discount_amount)
    
    def apply_discount(self, discount: float):
        if discount < 0:
            raise ValueError("Discount cannot be negative")
        if discount > self.subtotal:
            raise ValueError("Discount cannot exceed subtotal")
        self.discount_amount = discount
    
    def confirm(self):
        if not self.items:
            raise ValueError("Cannot confirm empty order")
        if self.status != OrderStatus.PENDING:
            raise ValueError(f"Cannot confirm order with status {self.status.value}")
        self.status = OrderStatus.CONFIRMED
    
    def cancel(self):
        if self.status in (OrderStatus.SHIPPED, OrderStatus.DELIVERED):
            raise ValueError("Cannot cancel shipped or delivered order")
        self.status = OrderStatus.CANCELLED
```

```python
# tests/test_order_processor.py
import pytest
from datetime import datetime
from src.order_processor import Order, OrderItem, OrderStatus


class TestOrderItem:
    """Tests for OrderItem dataclass"""
    
    def test_total_price_calculation(self):
        item = OrderItem(
            product_id="P001",
            product_name="สินค้า A",
            quantity=3,
            unit_price=100.0
        )
        assert item.total_price == 300.0
    
    def test_total_price_with_decimal(self):
        item = OrderItem(
            product_id="P002",
            product_name="สินค้า B",
            quantity=2,
            unit_price=99.99
        )
        assert item.total_price == pytest.approx(199.98)


class TestOrder:
    """Tests for Order class"""
    
    @pytest.fixture
    def empty_order(self):
        """Fixture: empty order"""
        return Order(customer_id="C001")
    
    @pytest.fixture
    def order_with_items(self):
        """Fixture: order with 2 items"""
        order = Order(customer_id="C001")
        order.add_item(OrderItem("P001", "สินค้า A", 2, 100.0))
        order.add_item(OrderItem("P002", "สินค้า B", 1, 50.0))
        return order
    
    # === add_item tests ===
    
    def test_add_item_to_empty_order(self, empty_order):
        item = OrderItem("P001", "สินค้า A", 1, 100.0)
        empty_order.add_item(item)
        assert len(empty_order.items) == 1
    
    def test_add_multiple_items(self, empty_order):
        empty_order.add_item(OrderItem("P001", "สินค้า A", 1, 100.0))
        empty_order.add_item(OrderItem("P002", "สินค้า B", 2, 200.0))
        assert len(empty_order.items) == 2
    
    def test_add_item_with_zero_quantity_raises_error(self, empty_order):
        with pytest.raises(ValueError, match="Quantity must be positive"):
            empty_order.add_item(OrderItem("P001", "สินค้า A", 0, 100.0))
    
    def test_add_item_with_negative_price_raises_error(self, empty_order):
        with pytest.raises(ValueError, match="Price cannot be negative"):
            empty_order.add_item(OrderItem("P001", "สินค้า A", 1, -10.0))
    
    # === subtotal and total tests ===
    
    def test_subtotal_calculation(self, order_with_items):
        assert order_with_items.subtotal == 250.0  # 200 + 50
    
    def test_total_equals_subtotal_without_discount(self, order_with_items):
        assert order_with_items.total == 250.0
    
    def test_total_with_discount(self, order_with_items):
        order_with_items.apply_discount(50.0)
        assert order_with_items.total == 200.0
    
    def test_subtotal_is_zero_for_empty_order(self, empty_order):
        assert empty_order.subtotal == 0.0
    
    # === apply_discount tests ===
    
    def test_apply_valid_discount(self, order_with_items):
        order_with_items.apply_discount(25.0)
        assert order_with_items.discount_amount == 25.0
    
    def test_apply_discount_exceeding_subtotal_raises_error(self, order_with_items):
        with pytest.raises(ValueError, match="Discount cannot exceed subtotal"):
            order_with_items.apply_discount(300.0)
    
    def test_apply_negative_discount_raises_error(self, order_with_items):
        with pytest.raises(ValueError, match="Discount cannot be negative"):
            order_with_items.apply_discount(-10.0)
    
    # === status tests ===
    
    def test_new_order_is_pending(self, empty_order):
        assert empty_order.status == OrderStatus.PENDING
    
    def test_confirm_order_with_items(self, order_with_items):
        order_with_items.confirm()
        assert order_with_items.status == OrderStatus.CONFIRMED
    
    def test_confirm_empty_order_raises_error(self, empty_order):
        with pytest.raises(ValueError, match="Cannot confirm empty order"):
            empty_order.confirm()
    
    def test_cannot_confirm_already_confirmed_order(self, order_with_items):
        order_with_items.confirm()
        with pytest.raises(ValueError):
            order_with_items.confirm()
    
    def test_cancel_pending_order(self, order_with_items):
        order_with_items.cancel()
        assert order_with_items.status == OrderStatus.CANCELLED
    
    def test_cannot_cancel_shipped_order(self, order_with_items):
        order_with_items.confirm()
        order_with_items.status = OrderStatus.SHIPPED
        with pytest.raises(ValueError, match="Cannot cancel shipped"):
            order_with_items.cancel()


class TestOrderParametrized:
    """Parametrized tests"""
    
    @pytest.mark.parametrize("quantity,price,expected", [
        (1, 100.0, 100.0),
        (5, 20.0, 100.0),
        (10, 9.99, 99.90),
        (100, 0.50, 50.0),
    ])
    def test_order_item_total_price(self, quantity, price, expected):
        item = OrderItem("P001", "Test", quantity, price)
        assert item.total_price == pytest.approx(expected)
```

### 8.3 Mocking ใน Python

```python
# tests/test_notification_service.py
import pytest
from unittest.mock import Mock, patch, MagicMock
from src.notification_service import NotificationService


class TestNotificationService:
    
    @pytest.fixture
    def mock_email_client(self):
        return Mock()
    
    @pytest.fixture
    def mock_sms_client(self):
        return Mock()
    
    @pytest.fixture
    def service(self, mock_email_client, mock_sms_client):
        return NotificationService(
            email_client=mock_email_client,
            sms_client=mock_sms_client,
        )
    
    def test_sends_email_on_order_confirmed(self, service, mock_email_client):
        order_data = {
            "id": "ORD-001",
            "customer_email": "customer@test.com",
            "total": 500.0,
        }
        
        service.notify_order_confirmed(order_data)
        
        mock_email_client.send.assert_called_once_with(
            to="customer@test.com",
            subject="ยืนยันคำสั่งซื้อ #ORD-001",
            body=pytest.approx(str, abs=0),  # ตรวจสอบว่า body มีเนื้อหา
        )
    
    def test_sends_sms_when_enabled(self, service, mock_sms_client):
        order_data = {
            "id": "ORD-001",
            "customer_phone": "0891234567",
            "total": 500.0,
        }
        
        service.notify_order_confirmed(order_data, send_sms=True)
        
        assert mock_sms_client.send.called
    
    def test_does_not_send_sms_when_disabled(self, service, mock_sms_client):
        order_data = {
            "id": "ORD-001",
            "customer_phone": "0891234567",
            "total": 500.0,
        }
        
        service.notify_order_confirmed(order_data, send_sms=False)
        
        mock_sms_client.send.assert_not_called()
    
    @patch('src.notification_service.datetime')
    def test_includes_timestamp_in_notification(self, mock_datetime, service):
        from datetime import datetime
        mock_datetime.now.return_value = datetime(2024, 1, 15, 10, 30, 0)
        
        result = service.create_notification_message("ORD-001")
        
        assert "2024-01-15" in result
```

---

## 9. ตัวอย่าง Test Cases จริง (Java)

### 9.1 Setup Project (Maven)

**pom.xml:**
```xml
<dependencies>
    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.0</version>
        <scope>test</scope>
    </dependency>
    
    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>5.3.1</version>
        <scope>test</scope>
    </dependency>
    
    <!-- AssertJ (better assertions) -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.24.2</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 9.2 ตัวอย่าง Unit Tests ด้วย JUnit 5

```java
// src/main/java/com/example/BankAccount.java
package com.example;

import java.math.BigDecimal;

public class BankAccount {
    private final String accountId;
    private BigDecimal balance;
    private boolean frozen;
    
    public BankAccount(String accountId, BigDecimal initialBalance) {
        if (initialBalance.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.accountId = accountId;
        this.balance = initialBalance;
        this.frozen = false;
    }
    
    public void deposit(BigDecimal amount) {
        if (frozen) throw new IllegalStateException("Account is frozen");
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        balance = balance.add(amount);
    }
    
    public void withdraw(BigDecimal amount) {
        if (frozen) throw new IllegalStateException("Account is frozen");
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Withdrawal amount must be positive");
        }
        if (amount.compareTo(balance) > 0) {
            throw new IllegalStateException("Insufficient funds");
        }
        balance = balance.subtract(amount);
    }
    
    public void freeze() { this.frozen = true; }
    public void unfreeze() { this.frozen = false; }
    
    public BigDecimal getBalance() { return balance; }
    public String getAccountId() { return accountId; }
    public boolean isFrozen() { return frozen; }
}
```

```java
// src/test/java/com/example/BankAccountTest.java
package com.example;

import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;
import org.junit.jupiter.params.provider.CsvSource;
import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;

@DisplayName("BankAccount Tests")
class BankAccountTest {
    
    private BankAccount account;
    
    @BeforeEach
    void setUp() {
        account = new BankAccount("ACC-001", new BigDecimal("1000.00"));
    }
    
    @Nested
    @DisplayName("Constructor Tests")
    class ConstructorTests {
        
        @Test
        @DisplayName("creates account with correct initial balance")
        void createsAccountWithCorrectBalance() {
            BankAccount newAccount = new BankAccount("ACC-002", new BigDecimal("500.00"));
            assertThat(newAccount.getBalance()).isEqualByComparingTo("500.00");
        }
        
        @Test
        @DisplayName("throws exception for negative initial balance")
        void throwsExceptionForNegativeBalance() {
            assertThatThrownBy(() -> 
                new BankAccount("ACC-003", new BigDecimal("-100.00"))
            ).isInstanceOf(IllegalArgumentException.class)
             .hasMessageContaining("cannot be negative");
        }
    }
    
    @Nested
    @DisplayName("Deposit Tests")
    class DepositTests {
        
        @Test
        @DisplayName("increases balance after deposit")
        void increasesBalanceAfterDeposit() {
            account.deposit(new BigDecimal("500.00"));
            assertThat(account.getBalance()).isEqualByComparingTo("1500.00");
        }
        
        @ParameterizedTest(name = "deposit of {0} should increase balance")
        @ValueSource(strings = {"0.01", "100", "999.99", "10000"})
        @DisplayName("accepts various valid deposit amounts")
        void acceptsValidDepositAmounts(String amount) {
            BigDecimal depositAmount = new BigDecimal(amount);
            BigDecimal expected = new BigDecimal("1000.00").add(depositAmount);
            
            account.deposit(depositAmount);
            
            assertThat(account.getBalance()).isEqualByComparingTo(expected);
        }
        
        @Test
        @DisplayName("throws exception for zero deposit")
        void throwsExceptionForZeroDeposit() {
            assertThatThrownBy(() -> account.deposit(BigDecimal.ZERO))
                .isInstanceOf(IllegalArgumentException.class);
        }
        
        @Test
        @DisplayName("throws exception when account is frozen")
        void throwsExceptionWhenFrozen() {
            account.freeze();
            assertThatThrownBy(() -> account.deposit(new BigDecimal("100")))
                .isInstanceOf(IllegalStateException.class)
                .hasMessage("Account is frozen");
        }
    }
    
    @Nested
    @DisplayName("Withdrawal Tests")
    class WithdrawalTests {
        
        @Test
        @DisplayName("decreases balance after withdrawal")
        void decreasesBalanceAfterWithdrawal() {
            account.withdraw(new BigDecimal("300.00"));
            assertThat(account.getBalance()).isEqualByComparingTo("700.00");
        }
        
        @Test
        @DisplayName("throws exception for insufficient funds")
        void throwsExceptionForInsufficientFunds() {
            assertThatThrownBy(() -> account.withdraw(new BigDecimal("2000.00")))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("Insufficient funds");
        }
        
        @Test
        @DisplayName("allows withdrawal of entire balance")
        void allowsWithdrawalOfEntireBalance() {
            account.withdraw(new BigDecimal("1000.00"));
            assertThat(account.getBalance()).isEqualByComparingTo(BigDecimal.ZERO);
        }
    }
    
    @Nested
    @DisplayName("Account Freeze Tests")
    class FreezeTests {
        
        @Test
        @DisplayName("account is not frozen by default")
        void accountIsNotFrozenByDefault() {
            assertThat(account.isFrozen()).isFalse();
        }
        
        @Test
        @DisplayName("freezes account successfully")
        void freezesAccount() {
            account.freeze();
            assertThat(account.isFrozen()).isTrue();
        }
        
        @Test
        @DisplayName("unfreezes account successfully")
        void unfreezesAccount() {
            account.freeze();
            account.unfreeze();
            assertThat(account.isFrozen()).isFalse();
        }
    }
    
    @ParameterizedTest(name = "deposit {0}, withdraw {1}, expected balance {2}")
    @CsvSource({
        "500, 200, 1300",
        "1000, 500, 1500",
        "100, 100, 1000",
    })
    @DisplayName("balance after deposit then withdrawal")
    void balanceAfterDepositAndWithdrawal(
        String deposit, String withdrawal, String expectedBalance
    ) {
        account.deposit(new BigDecimal(deposit));
        account.withdraw(new BigDecimal(withdrawal));
        
        assertThat(account.getBalance()).isEqualByComparingTo(expectedBalance);
    }
}
```

---

## 10. Test Coverage

### 10.1 Test Coverage คืออะไร?

Test Coverage คือการวัดว่าโค้ดของเราถูกทดสอบแล้วกี่ %

**ประเภทของ Coverage:**

| ประเภท | คำอธิบาย |
|--------|---------|
| **Line Coverage** | กี่ % ของ lines ถูก execute |
| **Branch Coverage** | กี่ % ของ if/else branches ถูกทดสอบ |
| **Function Coverage** | กี่ % ของ functions ถูกเรียก |
| **Statement Coverage** | กี่ % ของ statements ถูก execute |

### 10.2 ตัวอย่าง Coverage Analysis

```javascript
// โค้ดที่ต้องการ coverage
function processPayment(amount, method) {      // line 1
  if (amount <= 0) {                          // line 2
    throw new Error('Invalid amount');        // line 3
  }                                           // line 4
  
  if (method === 'credit') {                  // line 5
    return chargeCard(amount);               // line 6 - branch A
  } else if (method === 'bank') {            // line 7
    return bankTransfer(amount);             // line 8 - branch B
  } else {                                   // line 9
    throw new Error('Unknown payment method'); // line 10 - branch C
  }
}
```

**ถ้าทดสอบแค่ credit card:**
- Line Coverage: 6/8 = 75% (ไม่รัน line 8, 10)
- Branch Coverage: 2/3 = 67% (ไม่ทดสอบ bank และ unknown)

**ถ้าทดสอบทั้งหมด:**
- Line Coverage: 8/8 = 100%
- Branch Coverage: 3/3 = 100%

### 10.3 เป้าหมาย Coverage

```
Best Practices:
├── Critical Business Logic: ≥ 90%
├── Core Services/Utilities: ≥ 80%
├── General Application Code: ≥ 70%
└── Legacy/Generated Code: ≥ 50%

"100% coverage ≠ 100% bug-free"
(สามารถมี 100% coverage แต่ยังมี bugs อยู่ได้)
```

### 10.4 Generate Coverage Report (Jest)

```bash
# Run tests with coverage
npx jest --coverage

# ตัวอย่าง output:
-----------------------|---------|----------|---------|---------|
File                   | % Stmts | % Branch | % Funcs | % Lines |
-----------------------|---------|----------|---------|---------|
All files              |   87.50 |    75.00 |   90.00 |   88.00 |
 shopping-cart.js      |   95.00 |    90.00 |  100.00 |   95.00 |
 user-service.js       |   80.00 |    60.00 |   80.00 |   81.00 |
-----------------------|---------|----------|---------|---------|
```

### 10.5 jest.config.js สำหรับ Coverage

```javascript
// jest.config.js
module.exports = {
  collectCoverage: false, // ไม่ collect ทุกครั้ง (ใช้ --coverage flag แทน)
  collectCoverageFrom: [
    'src/**/*.{js,jsx}',
    '!src/**/*.test.{js,jsx}',
    '!src/index.js',
    '!src/**/*.stories.js',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThresholds: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
    // ตั้งค่าสำหรับไฟล์สำคัญ
    './src/payment-service.js': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90,
    },
  },
};
```

---

## 11. มาตรฐานการเขียน Test

### 11.1 FIRST Principles

Tests ที่ดีต้องมีคุณสมบัติ FIRST:

- **F**ast — ทำงานเร็ว (unit tests ไม่ควรเกิน 100ms)
- **I**solated — แต่ละ test ทำงานอิสระจากกัน
- **R**epeatable — ผลลัพธ์เหมือนกันทุกครั้ง
- **S**elf-Validating — บอกได้เองว่า pass หรือ fail
- **T**imely — เขียน tests ตรงเวลา (TDD: ก่อนโค้ด)

### 11.2 สิ่งที่ทำให้ Test ไม่ดี

**Test ที่ไม่ดี:**
```javascript
// ❌ Test ที่มีปัญหาหลายอย่าง
test('user', async () => {
  const u = await db.query('SELECT * FROM users');
  expect(u.length).toBeGreaterThan(0); // Fragile: ขึ้นอยู่กับ DB state
  
  const user = await createUser({ email: 'test' + Date.now() + '@test.com' }); // Random data
  expect(user).toBeTruthy(); // Weak assertion
  
  await deleteUser(user.id);
  const u2 = await db.query('SELECT * FROM users WHERE id = ?', [user.id]);
  expect(u2.length).toBe(0);
  // ทดสอบหลายอย่างใน test เดียว!
});
```

**Test ที่ดี:**
```javascript
// ✓ Test ที่ดี: ทดสอบอย่างเดียว, ชื่อชัดเจน, isolate

describe('UserRepository', () => {
  let db;
  
  beforeEach(async () => {
    db = await createTestDatabase(); // database ใหม่สำหรับแต่ละ test
  });
  
  afterEach(async () => {
    await db.destroy(); // cleanup หลัง test
  });
  
  test('createUser saves user with correct email', async () => {
    const repo = new UserRepository(db);
    
    const user = await repo.createUser({ email: 'test@example.com' });
    
    const found = await repo.findById(user.id);
    expect(found.email).toBe('test@example.com');
  });
  
  test('deleteUser removes user from database', async () => {
    const repo = new UserRepository(db);
    const user = await repo.createUser({ email: 'todelete@example.com' });
    
    await repo.deleteUser(user.id);
    
    const found = await repo.findById(user.id);
    expect(found).toBeNull();
  });
});
```

### 11.3 Test Data Management

```javascript
// ดี: ใช้ test factories สำหรับสร้าง test data
// tests/factories/user.factory.js
const { faker } = require('@faker-js/faker');

function createUserData(overrides = {}) {
  return {
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    age: faker.number.int({ min: 18, max: 80 }),
    createdAt: new Date(),
    ...overrides,
  };
}

// ใช้ใน tests
test('updates user name', async () => {
  const userData = createUserData({ name: 'สมชาย มีทรัพย์' });
  const user = await repo.save(userData);
  
  const updated = await service.updateName(user.id, 'สมหญิง ดีใจ');
  
  expect(updated.name).toBe('สมหญิง ดีใจ');
});
```

---

## 12. Test Naming Conventions

### 12.1 Naming Patterns

**Pattern 1: should_when_**
```javascript
test('should_return_zero_when_cart_is_empty', () => {...});
test('should_throw_error_when_email_is_invalid', () => {...});
```

**Pattern 2: given_when_then**
```javascript
test('given_empty_cart_when_add_item_then_cart_has_one_item', () => {...});
```

**Pattern 3: descriptive sentence (แนะนำ)**
```javascript
test('returns empty array when no users exist', () => {...});
test('throws InvalidEmailError when email format is wrong', () => {...});
test('creates user with hashed password', () => {...});
```

### 12.2 ตัวอย่างชื่อ Test ที่ดีและไม่ดี

```javascript
// ❌ ชื่อไม่ดี
test('test1', () => {...});
test('works', () => {...});
test('user test', () => {...});
test('error', () => {...});

// ✓ ชื่อดี
test('creates user with valid credentials', () => {...});
test('throws error when email already exists', () => {...});
test('returns null when user ID does not exist', () => {...});
test('sends welcome email after successful registration', () => {...});
```

### 12.3 Test Organization Structure

```javascript
describe('ProductService', () => {           // Service/Class
  describe('createProduct', () => {          // Method/Function
    describe('with valid data', () => {      // Condition (optional)
      test('saves product to database', () => {...});
      test('returns created product', () => {...});
      test('generates unique SKU', () => {...});
    });
    
    describe('with invalid data', () => {
      test('throws error for missing name', () => {...});
      test('throws error for negative price', () => {...});
      test('throws error for invalid category', () => {...});
    });
  });
  
  describe('updateProduct', () => {
    test('updates allowed fields', () => {...});
    test('does not update readonly fields', () => {...});
    test('throws error when product not found', () => {...});
  });
});
```

---

## 13. การจัดการ Test Suites

### 13.1 การ Skip Tests

```javascript
// Skip test เดียว
test.skip('todo: implement this feature', () => {
  // ...
});

// Skip ทั้ง suite
describe.skip('PaymentService (not implemented yet)', () => {
  // ...
});

// Run เฉพาะ test นี้ (อย่าลืมเอาออกก่อน commit!)
test.only('debug this test', () => {
  // ...
});
```

### 13.2 Tagging Tests

```javascript
// ใช้ custom tags เพื่อ filter tests
// jest.config.js
module.exports = {
  testPathPattern: process.env.TEST_TYPE === 'unit'
    ? '.unit.test.'
    : process.env.TEST_TYPE === 'integration'
    ? '.integration.test.'
    : '',
};

// ตั้งชื่อไฟล์ตามประเภท
// user.unit.test.js
// user.integration.test.js
// user.e2e.test.js
```

```bash
# Run แค่ unit tests
TEST_TYPE=unit npm test

# Run แค่ integration tests
TEST_TYPE=integration npm test
```

### 13.3 Test Timeout

```javascript
// ตั้ง timeout สำหรับ async test ที่ช้า
test('processes large dataset', async () => {
  const result = await processLargeData(bigDataset);
  expect(result.count).toBe(10000);
}, 30000); // timeout 30 seconds

// ตั้ง default timeout ใน jest.config.js
module.exports = {
  testTimeout: 10000, // 10 seconds สำหรับทุก test
};
```

---

## 14. แบบฝึกหัด

### แบบฝึกหัดที่ 1: เขียน Unit Tests สำหรับ Calculator

**โจทย์:** เขียน tests สำหรับ class Calculator ต่อไปนี้

```javascript
// calculator.js
class Calculator {
  add(a, b) { return a + b; }
  subtract(a, b) { return a - b; }
  multiply(a, b) { return a * b; }
  divide(a, b) {
    if (b === 0) throw new Error('Division by zero');
    return a / b;
  }
  power(base, exponent) {
    if (!Number.isInteger(exponent)) throw new Error('Exponent must be integer');
    return Math.pow(base, exponent);
  }
}
```

**สิ่งที่ต้องทำ:**
1. เขียน test สำหรับแต่ละ method
2. ครอบคลุม happy path, edge cases, และ error cases
3. ใช้ describe blocks จัดกลุ่ม
4. ชื่อ test ต้องสื่อความหมาย

**เฉลย:**
```javascript
// calculator.test.js
const Calculator = require('./calculator');

describe('Calculator', () => {
  let calc;
  
  beforeEach(() => {
    calc = new Calculator();
  });
  
  describe('add', () => {
    test('adds two positive numbers', () => {
      expect(calc.add(2, 3)).toBe(5);
    });
    test('adds negative numbers', () => {
      expect(calc.add(-2, -3)).toBe(-5);
    });
    test('adds zero', () => {
      expect(calc.add(5, 0)).toBe(5);
    });
    test('adds decimal numbers', () => {
      expect(calc.add(0.1, 0.2)).toBeCloseTo(0.3);
    });
  });
  
  describe('divide', () => {
    test('divides two numbers', () => {
      expect(calc.divide(10, 2)).toBe(5);
    });
    test('throws error when dividing by zero', () => {
      expect(() => calc.divide(10, 0)).toThrow('Division by zero');
    });
    test('handles decimal result', () => {
      expect(calc.divide(1, 3)).toBeCloseTo(0.333);
    });
  });
  
  describe('power', () => {
    test('calculates power correctly', () => {
      expect(calc.power(2, 10)).toBe(1024);
    });
    test('throws error for non-integer exponent', () => {
      expect(() => calc.power(2, 1.5)).toThrow('Exponent must be integer');
    });
    test('handles zero exponent', () => {
      expect(calc.power(5, 0)).toBe(1);
    });
    test('handles negative exponent', () => {
      expect(calc.power(2, -1)).toBeCloseTo(0.5);
    });
  });
});
```

### แบบฝึกหัดที่ 2: TDD - เขียน Password Validator

**โจทย์:** ใช้ TDD เขียน `validatePassword(password)` ที่ตรวจสอบว่า password:
- ยาวอย่างน้อย 8 ตัวอักษร
- มีตัวใหญ่อย่างน้อย 1 ตัว
- มีตัวเลขอย่างน้อย 1 ตัว
- มี special character อย่างน้อย 1 ตัว (!@#$%^&*)

**ขั้นตอน:**
1. เขียน test ก่อน (RED)
2. เขียน code ให้ผ่าน (GREEN)
3. Refactor (BLUE)

### แบบฝึกหัดที่ 3: Mock External Service

**โจทย์:** เขียน test สำหรับ `WeatherService` ที่เรียก API ภายนอก โดยใช้ mock

```javascript
// weather-service.js
class WeatherService {
  constructor(httpClient) {
    this.httpClient = httpClient;
    this.baseUrl = 'https://api.weather.com';
  }
  
  async getWeather(city) {
    const response = await this.httpClient.get(`${this.baseUrl}/weather/${city}`);
    return {
      city: response.data.name,
      temp: response.data.main.temp,
      description: response.data.weather[0].description,
    };
  }
}
```

**เขียน tests ที่:**
1. Mock `httpClient.get` ให้ return ข้อมูลจำลอง
2. ตรวจสอบว่า URL ถูกต้อง
3. ตรวจสอบ error handling เมื่อ API ล้มเหลว

---

## สรุป Part 07

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องทดสอบ** — ลดต้นทุน bug, เพิ่มความมั่นใจในการ deploy
2. **Test Pyramid** — Unit (70%) > Integration (20%) > E2E (10%)
3. **Testing Terminology** — Test Suite, Test Case, Assertion, Hooks, AAA Pattern
4. **TDD** — Red-Green-Refactor cycle
5. **BDD** — Given-When-Then, Gherkin syntax
6. **Test Doubles** — Mock, Stub, Spy, Fake และเมื่อไหร่ใช้อะไร
7. **ตัวอย่างจริง** — Node.js, Python, Java
8. **Test Coverage** — Line, Branch, Function coverage
9. **มาตรฐาน** — FIRST principles, naming conventions

**บทต่อไป:** Part 08 จะดูการทำ Automated Testing ใน CI/CD Pipeline โดยใช้ GitHub Actions

---

*ปรับปรุงล่าสุด: 2024 | หลักสูตร CI/CD สำหรับนักพัฒนา*

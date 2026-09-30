# Part 34: End-to-End Testing อัตโนมัติ

## บทนำ

End-to-End (E2E) Testing เป็นการทดสอบที่จำลองการใช้งานจริงของ users ตั้งแต่ต้นจนจบ ตรวจสอบว่าทุก components ทำงานร่วมกันได้อย่างถูกต้องในสภาพแวดล้อมที่เหมือนกับ production ในบทนี้เราจะเรียนรู้การใช้ Playwright, Cypress, และ Selenium สำหรับ E2E testing พร้อมกับ CI integration

## วัตถุประสงค์การเรียนรู้

- เข้าใจหลักการ E2E Testing และเมื่อไรควรใช้
- ตั้งค่าและเขียน tests ด้วย Playwright
- ใช้ Page Object Model Pattern
- Integrate E2E tests เข้ากับ CI pipeline
- จัดการ screenshot และ video artifacts
- รัน parallel E2E tests

---

## 34.1 Playwright Setup

### การติดตั้ง Playwright

```bash
# สร้าง project ใหม่
npm create playwright@latest

# หรือเพิ่มเข้า project ที่มีอยู่
npm install -D @playwright/test
npx playwright install

# ติดตั้ง browsers
npx playwright install chromium firefox webkit
```

### playwright.config.ts

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';
import * as dotenv from 'dotenv';

dotenv.config({ path: `.env.${process.env.TEST_ENV || 'local'}` });

export default defineConfig({
  testDir: './tests/e2e',
  
  // Timeout สำหรับแต่ละ test
  timeout: 60000,
  
  // Timeout สำหรับ expect assertions
  expect: { timeout: 10000 },
  
  // รัน tests แบบ parallel
  fullyParallel: true,
  
  // จำนวน workers
  workers: process.env.CI ? 2 : undefined,
  
  // Retry ใน CI
  retries: process.env.CI ? 2 : 0,
  
  // Reporter
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/e2e-results.xml' }],
    ['list'],
    process.env.CI ? ['github'] : ['line'],
  ],
  
  // Global setup
  globalSetup: require.resolve('./tests/e2e/global-setup.ts'),
  globalTeardown: require.resolve('./tests/e2e/global-teardown.ts'),
  
  use: {
    // Base URL
    baseURL: process.env.APP_URL || 'http://localhost:3000',
    
    // Screenshot เมื่อ test fail
    screenshot: 'only-on-failure',
    
    // Video recording
    video: process.env.CI ? 'retain-on-failure' : 'off',
    
    // Trace
    trace: 'on-first-retry',
    
    // Ignore HTTPS errors ใน test env
    ignoreHTTPSErrors: true,
    
    // User agent
    userAgent: 'Playwright E2E Test',
    
    // Extra HTTP headers
    extraHTTPHeaders: {
      'x-test-request': '1'
    }
  },
  
  projects: [
    // Desktop browsers
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    
    // Mobile browsers
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
    
    // Specific test suites
    {
      name: 'auth',
      testMatch: /auth\.spec\.ts/,
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'checkout',
      testMatch: /checkout\.spec\.ts/,
      use: { ...devices['Desktop Chrome'] },
      dependencies: ['auth'],
    }
  ],
  
  // Local dev server
  webServer: process.env.CI ? undefined : {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    stdout: 'pipe',
    stderr: 'pipe',
  },
});
```

---

## 34.2 Page Object Model

### Base Page

```typescript
// tests/e2e/pages/BasePage.ts
import { Page, Locator } from '@playwright/test';

export class BasePage {
  protected page: Page;
  protected baseURL: string;
  
  constructor(page: Page) {
    this.page = page;
    this.baseURL = process.env.APP_URL || 'http://localhost:3000';
  }
  
  // Navigation
  async navigate(path: string = '') {
    await this.page.goto(`${this.baseURL}${path}`);
    await this.waitForPageLoad();
  }
  
  // Wait helpers
  async waitForPageLoad() {
    await this.page.waitForLoadState('networkidle');
  }
  
  async waitForElement(selector: string, timeout: number = 10000) {
    await this.page.waitForSelector(selector, { timeout });
  }
  
  // Common actions
  async clickButton(text: string) {
    await this.page.getByRole('button', { name: text }).click();
  }
  
  async fillInput(label: string, value: string) {
    await this.page.getByLabel(label).fill(value);
  }
  
  async selectOption(label: string, value: string) {
    await this.page.getByLabel(label).selectOption(value);
  }
  
  // Assertions helpers
  async expectVisible(text: string) {
    await this.page.getByText(text).isVisible();
  }
  
  async expectURL(path: string) {
    await this.page.waitForURL(`**${path}`);
  }
  
  // Screenshot
  async takeScreenshot(name: string) {
    await this.page.screenshot({ 
      path: `screenshots/${name}.png`,
      fullPage: true 
    });
  }
  
  // Scroll helpers
  async scrollToBottom() {
    await this.page.evaluate(() => window.scrollTo(0, document.body.scrollHeight));
  }
}
```

### Login Page Object

```typescript
// tests/e2e/pages/LoginPage.ts
import { Page, expect } from '@playwright/test';
import { BasePage } from './BasePage';

export class LoginPage extends BasePage {
  // Locators
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly loginButton: Locator;
  readonly rememberMeCheckbox: Locator;
  readonly forgotPasswordLink: Locator;
  readonly errorMessage: Locator;
  readonly googleLoginButton: Locator;
  
  constructor(page: Page) {
    super(page);
    
    this.emailInput = page.getByLabel('Email address');
    this.passwordInput = page.getByLabel('Password');
    this.loginButton = page.getByRole('button', { name: 'Sign in' });
    this.rememberMeCheckbox = page.getByLabel('Remember me');
    this.forgotPasswordLink = page.getByText('Forgot password?');
    this.errorMessage = page.getByRole('alert');
    this.googleLoginButton = page.getByRole('button', { name: 'Continue with Google' });
  }
  
  async goto() {
    await this.navigate('/login');
    await expect(this.loginButton).toBeVisible();
  }
  
  async login(email: string, password: string, rememberMe: boolean = false) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    
    if (rememberMe) {
      await this.rememberMeCheckbox.check();
    }
    
    await this.loginButton.click();
  }
  
  async loginAndExpectSuccess(email: string, password: string) {
    await this.login(email, password);
    await this.expectURL('/dashboard');
  }
  
  async expectLoginError(errorText: string) {
    await expect(this.errorMessage).toBeVisible();
    await expect(this.errorMessage).toContainText(errorText);
  }
  
  async expectFieldError(field: string, errorText: string) {
    const fieldLocator = this.page.getByLabel(field);
    const errorLocator = fieldLocator.locator('../..').getByRole('alert');
    await expect(errorLocator).toContainText(errorText);
  }
}
```

### E-Commerce Checkout Page

```typescript
// tests/e2e/pages/CheckoutPage.ts
import { Page, Locator, expect } from '@playwright/test';
import { BasePage } from './BasePage';

export interface CheckoutData {
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  address: string;
  city: string;
  province: string;
  postalCode: string;
  cardNumber?: string;
  cardExpiry?: string;
  cardCVV?: string;
}

export class CheckoutPage extends BasePage {
  // Cart
  readonly cartIcon: Locator;
  readonly cartItemCount: Locator;
  readonly proceedToCheckoutButton: Locator;
  
  // Shipping form
  readonly firstNameInput: Locator;
  readonly lastNameInput: Locator;
  readonly emailInput: Locator;
  readonly phoneInput: Locator;
  readonly addressInput: Locator;
  readonly cityInput: Locator;
  readonly provinceSelect: Locator;
  readonly postalCodeInput: Locator;
  
  // Payment
  readonly paymentMethodCreditCard: Locator;
  readonly paymentMethodBankTransfer: Locator;
  readonly cardNumberInput: Locator;
  readonly cardExpiryInput: Locator;
  readonly cardCVVInput: Locator;
  
  // Order summary
  readonly subtotalAmount: Locator;
  readonly shippingAmount: Locator;
  readonly totalAmount: Locator;
  readonly placeOrderButton: Locator;
  readonly orderConfirmationHeading: Locator;
  readonly orderNumber: Locator;
  
  constructor(page: Page) {
    super(page);
    
    this.cartIcon = page.getByTestId('cart-icon');
    this.cartItemCount = page.getByTestId('cart-item-count');
    this.proceedToCheckoutButton = page.getByRole('button', { name: 'Proceed to Checkout' });
    
    this.firstNameInput = page.getByLabel('First Name');
    this.lastNameInput = page.getByLabel('Last Name');
    this.emailInput = page.getByLabel('Email');
    this.phoneInput = page.getByLabel('Phone Number');
    this.addressInput = page.getByLabel('Address');
    this.cityInput = page.getByLabel('City');
    this.provinceSelect = page.getByLabel('Province');
    this.postalCodeInput = page.getByLabel('Postal Code');
    
    this.paymentMethodCreditCard = page.getByLabel('Credit Card');
    this.paymentMethodBankTransfer = page.getByLabel('Bank Transfer');
    this.cardNumberInput = page.getByLabel('Card Number');
    this.cardExpiryInput = page.getByLabel('Expiry Date');
    this.cardCVVInput = page.getByLabel('CVV');
    
    this.subtotalAmount = page.getByTestId('subtotal-amount');
    this.shippingAmount = page.getByTestId('shipping-amount');
    this.totalAmount = page.getByTestId('total-amount');
    this.placeOrderButton = page.getByRole('button', { name: 'Place Order' });
    this.orderConfirmationHeading = page.getByRole('heading', { name: /Order Confirmed/i });
    this.orderNumber = page.getByTestId('order-number');
  }
  
  async goto() {
    await this.navigate('/checkout');
  }
  
  async fillShippingInfo(data: Partial<CheckoutData>) {
    if (data.firstName) await this.firstNameInput.fill(data.firstName);
    if (data.lastName) await this.lastNameInput.fill(data.lastName);
    if (data.email) await this.emailInput.fill(data.email);
    if (data.phone) await this.phoneInput.fill(data.phone);
    if (data.address) await this.addressInput.fill(data.address);
    if (data.city) await this.cityInput.fill(data.city);
    if (data.province) await this.provinceSelect.selectOption(data.province);
    if (data.postalCode) await this.postalCodeInput.fill(data.postalCode);
  }
  
  async fillPaymentInfo(data: Partial<CheckoutData>) {
    await this.paymentMethodCreditCard.click();
    
    if (data.cardNumber) {
      // Handle credit card iframe
      const cardFrame = this.page.frameLocator('#card-iframe');
      await cardFrame.getByLabel('Card number').fill(data.cardNumber);
    }
    if (data.cardExpiry) await this.cardExpiryInput.fill(data.cardExpiry);
    if (data.cardCVV) await this.cardCVVInput.fill(data.cardCVV);
  }
  
  async placeOrder() {
    await this.placeOrderButton.click();
    await expect(this.orderConfirmationHeading).toBeVisible({ timeout: 30000 });
  }
  
  async getOrderNumber(): Promise<string> {
    return await this.orderNumber.textContent() || '';
  }
  
  async expectOrderTotal(expectedTotal: string) {
    await expect(this.totalAmount).toHaveText(expectedTotal);
  }
}
```

---

## 34.3 E2E Test Scenarios

### Authentication Tests

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

test.describe('Authentication', () => {
  let loginPage: LoginPage;
  
  test.beforeEach(async ({ page }) => {
    loginPage = new LoginPage(page);
    await loginPage.goto();
  });
  
  test('ควร login สำเร็จด้วย credentials ที่ถูกต้อง', async ({ page }) => {
    const dashboard = new DashboardPage(page);
    
    await loginPage.loginAndExpectSuccess(
      'user@example.com',
      'ValidPassword123!'
    );
    
    await expect(dashboard.welcomeMessage).toContainText('Welcome');
    await expect(page).toHaveURL('/dashboard');
  });
  
  test('ควรแสดง error เมื่อ password ไม่ถูกต้อง', async () => {
    await loginPage.login('user@example.com', 'wrongpassword');
    await loginPage.expectLoginError('Invalid email or password');
  });
  
  test('ควรแสดง validation error เมื่อ email ว่าง', async () => {
    await loginPage.loginButton.click();
    await loginPage.expectFieldError('Email address', 'Email is required');
  });
  
  test('ควร redirect ไปยัง protected page หลัง login', async ({ page }) => {
    // ไปที่ protected page ก่อน
    await page.goto('/orders');
    await expect(page).toHaveURL(/\/login/);
    
    // Login
    const loginPage = new LoginPage(page);
    await loginPage.login('user@example.com', 'ValidPassword123!');
    
    // ควร redirect กลับไปที่ orders
    await expect(page).toHaveURL('/orders');
  });
  
  test('ควร logout สำเร็จ', async ({ page }) => {
    await loginPage.loginAndExpectSuccess('user@example.com', 'ValidPassword123!');
    
    await page.getByTestId('user-menu').click();
    await page.getByRole('menuitem', { name: 'Sign out' }).click();
    
    await expect(page).toHaveURL('/login');
    
    // ตรวจสอบว่า session ถูกลบ
    await page.goto('/dashboard');
    await expect(page).toHaveURL(/\/login/);
  });
  
  test('ควรทำงานได้บน mobile', async ({ page, isMobile }) => {
    test.skip(!isMobile, 'Mobile test only');
    
    await loginPage.loginAndExpectSuccess('user@example.com', 'ValidPassword123!');
    await expect(page).toHaveURL('/dashboard');
  });
});

// Authentication fixture
test.describe('Authenticated tests', () => {
  test.use({ storageState: '.auth/user.json' });
  
  test('ควรแสดง dashboard', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page).not.toHaveURL(/\/login/);
  });
});
```

### Shopping Cart and Checkout Tests

```typescript
// tests/e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';
import { ProductPage } from './pages/ProductPage';
import { CartPage } from './pages/CartPage';
import { CheckoutPage } from './pages/CheckoutPage';

test.use({ storageState: '.auth/user.json' });

const testCheckoutData = {
  firstName: 'สมชาย',
  lastName: 'ใจดี',
  email: 'user@example.com',
  phone: '0812345678',
  address: '123 ถนนสุขุมวิท',
  city: 'กรุงเทพมหานคร',
  province: 'Bangkok',
  postalCode: '10110',
};

test.describe('Checkout Flow', () => {
  test('ควรทำ checkout สำเร็จแบบครบขั้นตอน', async ({ page }) => {
    const productPage = new ProductPage(page);
    const cartPage = new CartPage(page);
    const checkoutPage = new CheckoutPage(page);
    
    // Step 1: เพิ่ม product ไปยัง cart
    await productPage.goto('/products/1');
    await productPage.selectQuantity(2);
    await productPage.addToCart();
    
    await expect(checkoutPage.cartItemCount).toHaveText('2');
    
    // Step 2: ไปที่ cart
    await checkoutPage.cartIcon.click();
    await cartPage.expectItemCount(1);
    await cartPage.proceedToCheckout();
    
    // Step 3: กรอก shipping info
    await checkoutPage.fillShippingInfo(testCheckoutData);
    
    // Step 4: เลือก payment method
    await checkoutPage.fillPaymentInfo({
      cardNumber: '4242424242424242',
      cardExpiry: '12/26',
      cardCVV: '123'
    });
    
    // Step 5: ตรวจสอบ order summary
    await checkoutPage.expectOrderTotal('฿1,500.00');
    
    // Step 6: Place order
    await checkoutPage.placeOrder();
    
    // ตรวจสอบ confirmation
    const orderNumber = await checkoutPage.getOrderNumber();
    expect(orderNumber).toMatch(/^ORD-\d{6}$/);
    
    // ตรวจสอบ email confirmation (optional - ถ้ามี email mock)
    await expect(page.getByText('Confirmation email sent')).toBeVisible();
  });
  
  test('ควรคำนวณ shipping cost ถูกต้อง', async ({ page }) => {
    const productPage = new ProductPage(page);
    const checkoutPage = new CheckoutPage(page);
    
    await productPage.goto('/products/1');
    await productPage.addToCart();
    await page.goto('/checkout');
    
    // Free shipping สำหรับ orders > 500 บาท
    await checkoutPage.expectShippingCost('Free');
    
    // ตรวจสอบ regular shipping
    await productPage.goto('/products/2');  // ราคา 200 บาท
    await productPage.clearCart();
    await productPage.addToCart();
    await page.goto('/checkout');
    await checkoutPage.expectShippingCost('฿50.00');
  });
  
  test('ควรใช้ coupon code ได้', async ({ page }) => {
    const checkoutPage = new CheckoutPage(page);
    
    // เพิ่ม product และไปที่ checkout
    await page.goto('/checkout');
    
    // ใส่ coupon code
    await page.getByLabel('Coupon Code').fill('SAVE10');
    await page.getByRole('button', { name: 'Apply' }).click();
    
    await expect(page.getByText('10% discount applied')).toBeVisible();
    await expect(checkoutPage.totalAmount).toHaveText('฿1,350.00');
  });
  
  test('ควรแสดง error เมื่อ payment fail', async ({ page }) => {
    const checkoutPage = new CheckoutPage(page);
    
    await page.goto('/checkout');
    await checkoutPage.fillShippingInfo(testCheckoutData);
    
    // ใช้ test card ที่จะ fail
    await checkoutPage.fillPaymentInfo({
      cardNumber: '4000000000000002',  // Declined card
      cardExpiry: '12/26',
      cardCVV: '123'
    });
    
    await checkoutPage.placeOrderButton.click();
    
    await expect(page.getByText('Payment was declined')).toBeVisible();
    await expect(checkoutPage.orderConfirmationHeading).not.toBeVisible();
  });
});
```

---

## 34.4 Authentication State Management

### Global Setup

```typescript
// tests/e2e/global-setup.ts
import { chromium, FullConfig } from '@playwright/test';
import * as fs from 'fs';
import * as path from 'path';

async function globalSetup(config: FullConfig) {
  const browser = await chromium.launch();
  const page = await browser.newPage();
  
  const authDir = '.auth';
  if (!fs.existsSync(authDir)) {
    fs.mkdirSync(authDir, { recursive: true });
  }
  
  // Setup: User authentication
  await setupUserAuth(page, authDir);
  
  // Setup: Admin authentication
  await setupAdminAuth(page, authDir);
  
  await browser.close();
}

async function setupUserAuth(page: any, authDir: string) {
  // Login
  await page.goto(`${process.env.APP_URL}/login`);
  await page.getByLabel('Email address').fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  
  // รอให้ login สำเร็จ
  await page.waitForURL('**/dashboard');
  
  // Save authentication state
  await page.context().storageState({ 
    path: path.join(authDir, 'user.json') 
  });
  
  console.log('✓ User authentication state saved');
}

async function setupAdminAuth(page: any, authDir: string) {
  await page.goto(`${process.env.APP_URL}/login`);
  await page.getByLabel('Email address').fill(process.env.TEST_ADMIN_EMAIL!);
  await page.getByLabel('Password').fill(process.env.TEST_ADMIN_PASSWORD!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  
  await page.waitForURL('**/admin/dashboard');
  
  await page.context().storageState({ 
    path: path.join(authDir, 'admin.json') 
  });
  
  console.log('✓ Admin authentication state saved');
}

export default globalSetup;
```

---

## 34.5 Visual Testing

```typescript
// tests/e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Visual Regression Tests', () => {
  test('Homepage ควรดูเหมือนเดิม', async ({ page }) => {
    await page.goto('/');
    
    // รอให้ images โหลด
    await page.waitForLoadState('networkidle');
    
    // Screenshot comparison
    await expect(page).toHaveScreenshot('homepage.png', {
      fullPage: true,
      threshold: 0.1,  // 10% pixel difference allowed
    });
  });
  
  test('Product card ควรดูเหมือนเดิม', async ({ page }) => {
    await page.goto('/products');
    
    const productCard = page.getByTestId('product-card').first();
    
    await expect(productCard).toHaveScreenshot('product-card.png', {
      threshold: 0.05,
    });
  });
  
  test('Mobile checkout ควรดูเหมือนเดิม', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 812 });
    await page.goto('/checkout');
    
    await expect(page).toHaveScreenshot('mobile-checkout.png', {
      fullPage: true,
    });
  });
});
```

---

## 34.6 CI Integration สำหรับ Playwright

```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    # รัน E2E tests ทุกคืน
    - cron: '0 0 * * *'

jobs:
  e2e-tests:
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]  # แบ่ง tests เป็น 4 parts
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium
      
      - name: Start application
        run: |
          npm run build
          npm run start &
          npx wait-on http://localhost:3000 --timeout 60000
        env:
          NODE_ENV: test
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
      
      - name: Run E2E tests (Shard ${{ matrix.shard }}/4)
        run: |
          npx playwright test \
            --shard=${{ matrix.shard }}/4 \
            --project=chromium
        env:
          APP_URL: http://localhost:3000
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
          TEST_ADMIN_EMAIL: ${{ secrets.TEST_ADMIN_EMAIL }}
          TEST_ADMIN_PASSWORD: ${{ secrets.TEST_ADMIN_PASSWORD }}
        continue-on-error: true
      
      - name: Upload blob report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: blob-report-${{ matrix.shard }}
          path: blob-report
          retention-days: 1
  
  merge-reports:
    runs-on: ubuntu-latest
    needs: e2e-tests
    if: always()
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Download blob reports
        uses: actions/download-artifact@v3
        with:
          pattern: blob-report-*
          merge-multiple: true
          path: all-blob-reports
      
      - name: Merge reports
        run: npx playwright merge-reports --reporter html ./all-blob-reports
      
      - name: Upload HTML report
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
      
      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: "test-results/**/*.xml"
      
      - name: Check test results
        run: |
          FAILED=$(cat all-blob-reports/*/results.json | jq '[.[] | select(.status == "failed")] | length')
          echo "Failed tests: $FAILED"
          if [ "$FAILED" -gt "0" ]; then
            exit 1
          fi

  visual-tests:
    runs-on: ubuntu-latest
    needs: e2e-tests
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci && npx playwright install chromium
      
      - name: Run visual tests
        run: npx playwright test tests/e2e/visual.spec.ts --update-snapshots=none
        continue-on-error: true
      
      - name: Upload visual diff
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: visual-diffs
          path: test-results/
```

---

## 34.7 Cypress Alternative

### Cypress Setup

```javascript
// cypress.config.js
const { defineConfig } = require('cypress');

module.exports = defineConfig({
  e2e: {
    baseUrl: process.env.APP_URL || 'http://localhost:3000',
    specPattern: 'cypress/e2e/**/*.cy.js',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: true,
    videoCompression: 32,
    screenshotOnRunFailure: true,
    defaultCommandTimeout: 10000,
    pageLoadTimeout: 30000,
    requestTimeout: 10000,
    
    setupNodeEvents(on, config) {
      // Tasks
      on('task', {
        clearDatabase() {
          // Clear test database
          return null;
        },
        seedDatabase(data) {
          // Seed test data
          return null;
        }
      });
      
      // Code coverage
      require('@cypress/code-coverage/task')(on, config);
      
      return config;
    }
  },
  
  component: {
    devServer: {
      framework: 'react',
      bundler: 'vite',
    },
  }
});
```

### Cypress Custom Commands

```javascript
// cypress/support/commands.js

// Login command
Cypress.Commands.add('login', (email, password) => {
  cy.session([email, password], () => {
    cy.visit('/login');
    cy.get('[data-testid="email-input"]').type(email);
    cy.get('[data-testid="password-input"]').type(password);
    cy.get('[data-testid="login-button"]').click();
    cy.url().should('include', '/dashboard');
  });
});

// API Login (faster)
Cypress.Commands.add('loginByAPI', (email, password) => {
  cy.request({
    method: 'POST',
    url: '/api/auth/login',
    body: { email, password }
  }).then(({ body }) => {
    window.localStorage.setItem('auth_token', body.token);
    cy.setCookie('session', body.session_cookie);
  });
});

// Intercept and mock API
Cypress.Commands.add('mockAPI', (method, url, response, statusCode = 200) => {
  cy.intercept(method, url, {
    statusCode,
    body: response,
    delay: 100
  });
});

// Add to cart
Cypress.Commands.add('addToCart', (productId, quantity = 1) => {
  cy.request({
    method: 'POST',
    url: '/api/cart/items',
    body: { productId, quantity },
    headers: {
      Authorization: `Bearer ${localStorage.getItem('auth_token')}`
    }
  });
});

// Clear cart
Cypress.Commands.add('clearCart', () => {
  cy.request({
    method: 'DELETE',
    url: '/api/cart',
    headers: {
      Authorization: `Bearer ${localStorage.getItem('auth_token')}`
    }
  });
});
```

### Cypress E2E Test

```javascript
// cypress/e2e/checkout.cy.js
describe('Checkout Flow', () => {
  beforeEach(() => {
    cy.loginByAPI('user@example.com', 'password123');
    cy.clearCart();
    cy.addToCart(1, 2);
  });
  
  it('ควรทำ checkout สำเร็จ', () => {
    cy.visit('/checkout');
    
    // ตรวจสอบ cart items
    cy.get('[data-testid="cart-item"]').should('have.length', 1);
    
    // กรอก shipping info
    cy.get('[data-testid="first-name"]').type('สมชาย');
    cy.get('[data-testid="last-name"]').type('ใจดี');
    cy.get('[data-testid="phone"]').type('0812345678');
    cy.get('[data-testid="address"]').type('123 ถนนสุขุมวิท');
    cy.get('[data-testid="province"]').select('Bangkok');
    cy.get('[data-testid="postal-code"]').type('10110');
    
    // เลือก payment method
    cy.get('[data-testid="payment-credit-card"]').click();
    
    // กรอก card info ใน iframe
    cy.getWithinIframe('[data-testid="card-number"]', '#card-iframe')
      .type('4242424242424242');
    
    cy.get('[data-testid="place-order-button"]').click();
    
    // ตรวจสอบ confirmation
    cy.get('[data-testid="order-confirmation"]').should('be.visible');
    cy.get('[data-testid="order-number"]').should('match', /ORD-\d{6}/);
    
    // ตรวจสอบ URL
    cy.url().should('include', '/order-confirmation');
  });
  
  it('ควรแสดง error เมื่อ card expired', () => {
    cy.visit('/checkout');
    cy.fillCheckoutForm();
    
    cy.get('[data-testid="payment-credit-card"]').click();
    cy.get('[data-testid="card-number"]').type('4000000000000069');  // Expired card
    cy.get('[data-testid="card-expiry"]').type('12/20');
    cy.get('[data-testid="card-cvv"]').type('123');
    
    cy.get('[data-testid="place-order-button"]').click();
    
    cy.get('[data-testid="payment-error"]')
      .should('be.visible')
      .should('contain', 'Your card has expired');
  });
  
  it('ควรบันทึก video เมื่อ test fail', () => {
    // Test ที่จะ fail เพื่อ demo video recording
    cy.visit('/checkout');
    cy.get('[data-testid="non-existent-element"]').should('be.visible');
  });
});
```

---

## 34.8 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Page Object Model

สร้าง Page Object สำหรับ User Profile page:

```typescript
// แบบฝึกหัด: สร้าง UserProfilePage.ts
// ควรมี:
// 1. Locators สำหรับทุก elements ในหน้า
// 2. Methods: updateProfile(), changePassword(), uploadAvatar()
// 3. Assertion methods: expectProfileUpdated(), expectAvatarChanged()

export class UserProfilePage extends BasePage {
  // TODO: เพิ่ม locators และ methods
}

// Tests
test('ควรอัพเดต profile ได้', async ({ page }) => {
  // TODO: implement test
});

test('ควรเปลี่ยน password ได้', async ({ page }) => {
  // TODO: implement test
});
```

### แบบฝึกหัดที่ 2: Test Data Management

สร้าง test fixtures สำหรับ E2E tests:

```typescript
// แบบฝึกหัด: สร้าง test-data.ts
// 1. สร้าง users ประเภทต่างๆ (regular, admin, premium)
// 2. สร้าง products และ categories
// 3. สร้าง orders ในสถานะต่างๆ
// 4. เพิ่ม helper functions สำหรับ cleanup

export const testUsers = {
  regular: { email: 'test@example.com', password: '...' },
  admin: { email: 'admin@example.com', password: '...' },
};
```

### แบบฝึกหัดที่ 3: API Mocking

เขียน E2E tests ที่ใช้ API mocking:

```typescript
// แบบฝึกหัด
test('ควรแสดง promotions เมื่อ API ส่งข้อมูล', async ({ page }) => {
  // TODO: Mock /api/promotions endpoint
  // Return test promotions data
  // ตรวจสอบว่า promotions แสดงถูกต้อง
});

test('ควรแสดง error state เมื่อ API fail', async ({ page }) => {
  // TODO: Mock /api/products ให้ return 500 error
  // ตรวจสอบว่า error message แสดงถูกต้อง
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

- **Playwright**: เครื่องมือ E2E testing ที่ทรงพลังพร้อม multi-browser support
- **Page Object Model**: Pattern สำหรับจัดการ page interactions อย่างมีระเบียบ
- **Authentication State**: จัดการ login state อย่างมีประสิทธิภาพ
- **Visual Testing**: Screenshot comparison สำหรับ regression detection
- **Parallel Testing**: รัน tests แบบ parallel ด้วย sharding
- **CI Integration**: รัน E2E tests ใน GitHub Actions พร้อม artifact collection
- **Cypress**: ทางเลือก E2E framework ที่นิยมใช้

บทต่อไป (Part 35) เราจะเรียนรู้เรื่อง **Performance Testing ใน CI/CD**

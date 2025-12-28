# E2E 테스트 (End-to-End Testing)

## 목차
1. [E2E 테스트 개요](#e2e-테스트-개요)
2. [주요 도구: Selenium, Cypress, Playwright](#주요-도구-selenium-cypress-playwright)
3. [테스트 전략](#테스트-전략)
4. [Page Object Pattern](#page-object-pattern)
5. [테스트 안정성](#테스트-안정성)
---

## E2E 테스트 개요

### 정의

E2E(End-to-End) 테스트는 전체 애플리케이션을 실제 사용자 관점에서 테스트하는 방법이다. 프론트엔드부터 백엔드, 데이터베이스까지 모든 컴포넌트가 함께 동작하는 것을 검증한다.

```
사용자 → 브라우저 → 프론트엔드 → API → 백엔드 → 데이터베이스
   ↑                                                    |
   └────────────── 응답 ────────────────────────────────┘
                    │
         E2E 테스트: 전체 흐름 검증
```

### 테스트 피라미드에서의 위치

```
        /\
       /E2E\        ← 여기! (10%)
      /────\
     /통합  \
    /────────\
   /  단위    \
  /────────────\
```

### E2E 테스트의 특징

| 장점 | 단점 |
|-----|-----|
| 실제 사용자 시나리오 검증 | 실행 속도 느림 |
| 시스템 전체 동작 확인 | 유지보수 비용 높음 |
| 통합 문제 발견 | Flaky 테스트 발생 가능 |
| 회귀 테스트에 유용 | 디버깅 어려움 |

---

## 주요 도구: Selenium, Cypress, Playwright

### 비교 개요

| 특성 | Selenium | Cypress | Playwright |
|-----|----------|---------|------------|
| 언어 지원 | Java, Python, C#, JS 등 | JavaScript/TypeScript | JS, TS, Python, Java, C# |
| 브라우저 | Chrome, Firefox, Safari, Edge | Chrome, Firefox, Edge | Chrome, Firefox, Safari, Edge |
| 아키텍처 | WebDriver 프로토콜 | 브라우저 내부 실행 | CDP/Browser-specific |
| 병렬 실행 | 추가 설정 필요 | 유료 플랜 필요 | 네이티브 지원 |
| 자동 대기 | 수동 구현 | 내장 | 내장 |
| 속도 | 보통 | 빠름 | 매우 빠름 |

### 1. Selenium

가장 오래된 브라우저 자동화 도구. WebDriver 프로토콜 기반.

```java
// Java + Selenium WebDriver
import org.openqa.selenium.*;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.support.ui.WebDriverWait;
import org.openqa.selenium.support.ui.ExpectedConditions;

public class LoginTest {

    private WebDriver driver;
    private WebDriverWait wait;

    @BeforeEach
    void setUp() {
        driver = new ChromeDriver();
        wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        driver.manage().window().maximize();
    }

    @AfterEach
    void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }

    @Test
    void shouldLoginSuccessfully() {
        // Given
        driver.get("https://example.com/login");

        // When
        WebElement emailInput = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.id("email"))
        );
        emailInput.sendKeys("user@example.com");

        WebElement passwordInput = driver.findElement(By.id("password"));
        passwordInput.sendKeys("password123");

        WebElement loginButton = driver.findElement(By.cssSelector("button[type='submit']"));
        loginButton.click();

        // Then
        WebElement welcomeMessage = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.className("welcome-message"))
        );
        assertTrue(welcomeMessage.getText().contains("Welcome"));
    }

    @Test
    void shouldShowErrorForInvalidCredentials() {
        driver.get("https://example.com/login");

        driver.findElement(By.id("email")).sendKeys("wrong@example.com");
        driver.findElement(By.id("password")).sendKeys("wrongpassword");
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        WebElement errorMessage = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.className("error-message"))
        );
        assertEquals("Invalid credentials", errorMessage.getText());
    }
}
```

```python
# Python + Selenium
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import pytest

class TestLogin:

    @pytest.fixture(autouse=True)
    def setup(self):
        self.driver = webdriver.Chrome()
        self.wait = WebDriverWait(self.driver, 10)
        self.driver.maximize_window()
        yield
        self.driver.quit()

    def test_should_login_successfully(self):
        # Given
        self.driver.get("https://example.com/login")

        # When
        email_input = self.wait.until(
            EC.visibility_of_element_located((By.ID, "email"))
        )
        email_input.send_keys("user@example.com")

        password_input = self.driver.find_element(By.ID, "password")
        password_input.send_keys("password123")

        login_button = self.driver.find_element(By.CSS_SELECTOR, "button[type='submit']")
        login_button.click()

        # Then
        welcome_message = self.wait.until(
            EC.visibility_of_element_located((By.CLASS_NAME, "welcome-message"))
        )
        assert "Welcome" in welcome_message.text
```

### 2. Cypress

JavaScript 기반의 현대적인 E2E 테스트 도구. 브라우저 내부에서 실행.

```javascript
// cypress/e2e/login.cy.js
describe('Login Flow', () => {

    beforeEach(() => {
        cy.visit('/login');
    });

    it('should login successfully with valid credentials', () => {
        // Given - 페이지가 이미 로드됨

        // When
        cy.get('[data-testid="email-input"]')
            .type('user@example.com');

        cy.get('[data-testid="password-input"]')
            .type('password123');

        cy.get('[data-testid="login-button"]')
            .click();

        // Then
        cy.url().should('include', '/dashboard');
        cy.get('.welcome-message')
            .should('contain', 'Welcome');

        // 로컬 스토리지 확인
        cy.window()
            .its('localStorage.authToken')
            .should('exist');
    });

    it('should show error for invalid credentials', () => {
        cy.get('[data-testid="email-input"]')
            .type('wrong@example.com');

        cy.get('[data-testid="password-input"]')
            .type('wrongpassword');

        cy.get('[data-testid="login-button"]')
            .click();

        cy.get('.error-message')
            .should('be.visible')
            .and('contain', 'Invalid credentials');

        cy.url().should('include', '/login');
    });

    it('should validate required fields', () => {
        cy.get('[data-testid="login-button"]')
            .click();

        cy.get('[data-testid="email-error"]')
            .should('contain', 'Email is required');

        cy.get('[data-testid="password-error"]')
            .should('contain', 'Password is required');
    });
});

// 네트워크 요청 가로채기
describe('Login with API mocking', () => {

    it('should handle server error gracefully', () => {
        cy.intercept('POST', '/api/login', {
            statusCode: 500,
            body: { message: 'Internal Server Error' }
        }).as('loginRequest');

        cy.visit('/login');
        cy.get('[data-testid="email-input"]').type('user@example.com');
        cy.get('[data-testid="password-input"]').type('password123');
        cy.get('[data-testid="login-button"]').click();

        cy.wait('@loginRequest');
        cy.get('.error-message')
            .should('contain', 'Something went wrong');
    });

    it('should show loading state during login', () => {
        cy.intercept('POST', '/api/login', {
            delay: 2000,
            statusCode: 200,
            body: { token: 'fake-token' }
        }).as('slowLogin');

        cy.visit('/login');
        cy.get('[data-testid="email-input"]').type('user@example.com');
        cy.get('[data-testid="password-input"]').type('password123');
        cy.get('[data-testid="login-button"]').click();

        cy.get('[data-testid="loading-spinner"]')
            .should('be.visible');

        cy.wait('@slowLogin');
        cy.get('[data-testid="loading-spinner"]')
            .should('not.exist');
    });
});
```

### 3. Playwright

Microsoft에서 개발한 최신 E2E 테스트 도구. 크로스 브라우저 지원 우수.

```typescript
// tests/login.spec.ts
import { test, expect, Page } from '@playwright/test';

test.describe('Login Flow', () => {

    test.beforeEach(async ({ page }) => {
        await page.goto('/login');
    });

    test('should login successfully', async ({ page }) => {
        // Given - 페이지가 이미 로드됨

        // When
        await page.fill('[data-testid="email-input"]', 'user@example.com');
        await page.fill('[data-testid="password-input"]', 'password123');
        await page.click('[data-testid="login-button"]');

        // Then
        await expect(page).toHaveURL(/.*dashboard/);
        await expect(page.locator('.welcome-message')).toContainText('Welcome');

        // 로컬 스토리지 확인
        const token = await page.evaluate(() => localStorage.getItem('authToken'));
        expect(token).toBeTruthy();
    });

    test('should show error for invalid credentials', async ({ page }) => {
        await page.fill('[data-testid="email-input"]', 'wrong@example.com');
        await page.fill('[data-testid="password-input"]', 'wrongpassword');
        await page.click('[data-testid="login-button"]');

        await expect(page.locator('.error-message')).toBeVisible();
        await expect(page.locator('.error-message')).toContainText('Invalid credentials');
    });
});

// 다중 브라우저 테스트
test.describe('Cross-browser login', () => {

    test('should work on Chrome', async ({ page, browserName }) => {
        test.skip(browserName !== 'chromium', 'Chrome only');
        await page.goto('/login');
        // Chrome 특화 테스트
    });

    test('should work on Safari', async ({ page, browserName }) => {
        test.skip(browserName !== 'webkit', 'Safari only');
        await page.goto('/login');
        // Safari 특화 테스트
    });
});

// API 모킹
test.describe('Login with API mocking', () => {

    test('should handle network error', async ({ page }) => {
        await page.route('**/api/login', route => {
            route.abort('failed');
        });

        await page.goto('/login');
        await page.fill('[data-testid="email-input"]', 'user@example.com');
        await page.fill('[data-testid="password-input"]', 'password123');
        await page.click('[data-testid="login-button"]');

        await expect(page.locator('.error-message'))
            .toContainText('Network error');
    });

    test('should handle slow response', async ({ page }) => {
        await page.route('**/api/login', async route => {
            await new Promise(resolve => setTimeout(resolve, 3000));
            await route.fulfill({
                status: 200,
                body: JSON.stringify({ token: 'fake-token' })
            });
        });

        await page.goto('/login');
        await page.fill('[data-testid="email-input"]', 'user@example.com');
        await page.fill('[data-testid="password-input"]', 'password123');
        await page.click('[data-testid="login-button"]');

        await expect(page.locator('[data-testid="loading-spinner"]'))
            .toBeVisible();
    });
});

// 스크린샷 및 비주얼 테스트
test('should match visual snapshot', async ({ page }) => {
    await page.goto('/login');
    await expect(page).toHaveScreenshot('login-page.png');
});
```

```python
# Python Playwright
import pytest
from playwright.sync_api import Page, expect

class TestLogin:

    def test_should_login_successfully(self, page: Page):
        # Given
        page.goto("/login")

        # When
        page.fill('[data-testid="email-input"]', "user@example.com")
        page.fill('[data-testid="password-input"]', "password123")
        page.click('[data-testid="login-button"]')

        # Then
        expect(page).to_have_url(re.compile(r".*dashboard"))
        expect(page.locator(".welcome-message")).to_contain_text("Welcome")

    def test_should_show_error_for_invalid_credentials(self, page: Page):
        page.goto("/login")

        page.fill('[data-testid="email-input"]', "wrong@example.com")
        page.fill('[data-testid="password-input"]', "wrongpassword")
        page.click('[data-testid="login-button"]')

        expect(page.locator(".error-message")).to_be_visible()
        expect(page.locator(".error-message")).to_contain_text("Invalid credentials")

    def test_should_handle_network_error(self, page: Page):
        # API 모킹
        page.route("**/api/login", lambda route: route.abort())

        page.goto("/login")
        page.fill('[data-testid="email-input"]', "user@example.com")
        page.fill('[data-testid="password-input"]', "password123")
        page.click('[data-testid="login-button"]')

        expect(page.locator(".error-message")).to_contain_text("Network error")


# conftest.py - 공통 설정
@pytest.fixture(scope="function")
def page(browser):
    context = browser.new_context(
        viewport={"width": 1920, "height": 1080},
        locale="ko-KR"
    )
    page = context.new_page()
    yield page
    context.close()
```

### 도구 선택 가이드

```
Selenium 선택:
- 다양한 언어 지원 필요
- 레거시 시스템 통합
- 광범위한 브라우저 지원

Cypress 선택:
- JavaScript/TypeScript 프로젝트
- 빠른 피드백 필요
- 개발자 친화적 디버깅

Playwright 선택:
- 크로스 브라우저 테스트 필수
- 다국어 지원 필요
- 모바일 에뮬레이션 필요
- 대규모 병렬 실행
```

---

## 테스트 전략

### 테스트 범위 결정

```
전체 시나리오    vs    핵심 플로우만
    ↓                    ↓
유지보수 부담 높음    관리 가능한 수준

권장: 핵심 사용자 여정(Critical User Journey)에 집중
```

### Critical User Journey 식별

```typescript
// 전자상거래 사이트의 핵심 사용자 여정
const criticalJourneys = [
    '회원가입 → 로그인',
    '상품 검색 → 상세 페이지 → 장바구니 추가',
    '장바구니 → 결제 → 주문 완료',
    '주문 조회 → 주문 취소/반품'
];

// 각 여정별 E2E 테스트 작성
```

```typescript
// tests/e2e/checkout-journey.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Checkout Journey', () => {

    test('complete purchase flow', async ({ page }) => {
        // 1. 상품 검색
        await page.goto('/');
        await page.fill('[data-testid="search-input"]', '노트북');
        await page.click('[data-testid="search-button"]');

        // 2. 상품 선택
        await page.click('[data-testid="product-card"]:first-child');
        await expect(page).toHaveURL(/.*product\/.+/);

        // 3. 장바구니 추가
        await page.click('[data-testid="add-to-cart"]');
        await expect(page.locator('[data-testid="cart-count"]'))
            .toHaveText('1');

        // 4. 장바구니로 이동
        await page.click('[data-testid="cart-icon"]');
        await expect(page).toHaveURL(/.*cart/);

        // 5. 결제 진행
        await page.click('[data-testid="checkout-button"]');

        // 6. 배송 정보 입력
        await page.fill('[data-testid="address"]', '서울시 강남구');
        await page.fill('[data-testid="phone"]', '010-1234-5678');
        await page.click('[data-testid="next-step"]');

        // 7. 결제 정보 입력
        await page.fill('[data-testid="card-number"]', '1234567890123456');
        await page.fill('[data-testid="card-expiry"]', '12/25');
        await page.fill('[data-testid="card-cvv"]', '123');

        // 8. 주문 완료
        await page.click('[data-testid="place-order"]');
        await expect(page).toHaveURL(/.*order-confirmation/);
        await expect(page.locator('[data-testid="order-number"]'))
            .toBeVisible();
    });
});
```

### 테스트 데이터 관리

```typescript
// fixtures/test-data.ts
export const testUsers = {
    standard: {
        email: 'test-user@example.com',
        password: 'TestPassword123!'
    },
    premium: {
        email: 'premium-user@example.com',
        password: 'PremiumPassword123!'
    }
};

export const testProducts = {
    laptop: {
        id: 'PROD-001',
        name: 'MacBook Pro',
        price: 2000000
    }
};

// 테스트에서 사용
test('premium user should see discount', async ({ page }) => {
    await loginAs(page, testUsers.premium);
    await page.goto(`/product/${testProducts.laptop.id}`);

    await expect(page.locator('.discount-badge'))
        .toContainText('10% 할인');
});
```

### 환경별 설정

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
    testDir: './tests',
    fullyParallel: true,
    forbidOnly: !!process.env.CI,
    retries: process.env.CI ? 2 : 0,
    workers: process.env.CI ? 4 : undefined,

    reporter: [
        ['html'],
        ['junit', { outputFile: 'results/junit.xml' }]
    ],

    use: {
        baseURL: process.env.BASE_URL || 'http://localhost:3000',
        trace: 'on-first-retry',
        screenshot: 'only-on-failure',
        video: 'retain-on-failure'
    },

    projects: [
        {
            name: 'chromium',
            use: { ...devices['Desktop Chrome'] }
        },
        {
            name: 'firefox',
            use: { ...devices['Desktop Firefox'] }
        },
        {
            name: 'webkit',
            use: { ...devices['Desktop Safari'] }
        },
        {
            name: 'mobile-chrome',
            use: { ...devices['Pixel 5'] }
        },
        {
            name: 'mobile-safari',
            use: { ...devices['iPhone 12'] }
        }
    ],

    webServer: {
        command: 'npm run start',
        url: 'http://localhost:3000',
        reuseExistingServer: !process.env.CI
    }
});
```

---

## Page Object Pattern

### 개념

Page Object Pattern은 웹 페이지나 컴포넌트를 클래스로 추상화하여 테스트 코드의 유지보수성을 높이는 디자인 패턴이다.

```
기존 방식:
Test1: page.click('#login-btn')
Test2: page.click('#login-btn')
Test3: page.click('#login-btn')
       ↓
버튼 ID가 변경되면 모든 테스트 수정 필요

Page Object 방식:
LoginPage.clickLogin()
       ↓
페이지 객체 내부만 수정하면 됨
```

### Playwright Page Object 구현

```typescript
// pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
    readonly page: Page;

    // Locators
    readonly emailInput: Locator;
    readonly passwordInput: Locator;
    readonly loginButton: Locator;
    readonly errorMessage: Locator;
    readonly forgotPasswordLink: Locator;

    constructor(page: Page) {
        this.page = page;
        this.emailInput = page.locator('[data-testid="email-input"]');
        this.passwordInput = page.locator('[data-testid="password-input"]');
        this.loginButton = page.locator('[data-testid="login-button"]');
        this.errorMessage = page.locator('.error-message');
        this.forgotPasswordLink = page.locator('[data-testid="forgot-password"]');
    }

    async goto() {
        await this.page.goto('/login');
    }

    async login(email: string, password: string) {
        await this.emailInput.fill(email);
        await this.passwordInput.fill(password);
        await this.loginButton.click();
    }

    async expectErrorMessage(message: string) {
        await expect(this.errorMessage).toBeVisible();
        await expect(this.errorMessage).toContainText(message);
    }

    async expectLoginSuccessful() {
        await expect(this.page).toHaveURL(/.*dashboard/);
    }
}
```

```typescript
// pages/DashboardPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class DashboardPage {
    readonly page: Page;

    // Locators
    readonly welcomeMessage: Locator;
    readonly userMenu: Locator;
    readonly logoutButton: Locator;
    readonly navigationItems: Locator;

    constructor(page: Page) {
        this.page = page;
        this.welcomeMessage = page.locator('.welcome-message');
        this.userMenu = page.locator('[data-testid="user-menu"]');
        this.logoutButton = page.locator('[data-testid="logout-button"]');
        this.navigationItems = page.locator('nav a');
    }

    async expectWelcomeMessage(name: string) {
        await expect(this.welcomeMessage).toContainText(`Welcome, ${name}`);
    }

    async logout() {
        await this.userMenu.click();
        await this.logoutButton.click();
    }

    async navigateTo(section: string) {
        await this.navigationItems.filter({ hasText: section }).click();
    }
}
```

```typescript
// pages/ProductPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class ProductPage {
    readonly page: Page;

    // Locators
    readonly productTitle: Locator;
    readonly productPrice: Locator;
    readonly quantityInput: Locator;
    readonly addToCartButton: Locator;
    readonly successToast: Locator;

    constructor(page: Page) {
        this.page = page;
        this.productTitle = page.locator('[data-testid="product-title"]');
        this.productPrice = page.locator('[data-testid="product-price"]');
        this.quantityInput = page.locator('[data-testid="quantity-input"]');
        this.addToCartButton = page.locator('[data-testid="add-to-cart"]');
        this.successToast = page.locator('.toast-success');
    }

    async goto(productId: string) {
        await this.page.goto(`/product/${productId}`);
    }

    async addToCart(quantity: number = 1) {
        await this.quantityInput.fill(quantity.toString());
        await this.addToCartButton.click();
        await expect(this.successToast).toBeVisible();
    }

    async getPrice(): Promise<number> {
        const priceText = await this.productPrice.textContent();
        return parseInt(priceText?.replace(/[^0-9]/g, '') || '0');
    }
}
```

### 테스트에서 Page Object 사용

```typescript
// tests/login.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';

test.describe('Login Flow', () => {
    let loginPage: LoginPage;
    let dashboardPage: DashboardPage;

    test.beforeEach(async ({ page }) => {
        loginPage = new LoginPage(page);
        dashboardPage = new DashboardPage(page);
    });

    test('should login successfully', async () => {
        await loginPage.goto();
        await loginPage.login('user@example.com', 'password123');

        await loginPage.expectLoginSuccessful();
        await dashboardPage.expectWelcomeMessage('User');
    });

    test('should show error for invalid credentials', async () => {
        await loginPage.goto();
        await loginPage.login('wrong@example.com', 'wrongpassword');

        await loginPage.expectErrorMessage('Invalid credentials');
    });

    test('should logout successfully', async ({ page }) => {
        // 로그인 상태로 시작
        await loginPage.goto();
        await loginPage.login('user@example.com', 'password123');

        // 로그아웃
        await dashboardPage.logout();

        // 로그인 페이지로 리다이렉트 확인
        await expect(page).toHaveURL(/.*login/);
    });
});
```

### Page Object Fixture 패턴 (Playwright)

```typescript
// fixtures/pages.ts
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { DashboardPage } from '../pages/DashboardPage';
import { ProductPage } from '../pages/ProductPage';
import { CartPage } from '../pages/CartPage';

type Pages = {
    loginPage: LoginPage;
    dashboardPage: DashboardPage;
    productPage: ProductPage;
    cartPage: CartPage;
};

export const test = base.extend<Pages>({
    loginPage: async ({ page }, use) => {
        await use(new LoginPage(page));
    },
    dashboardPage: async ({ page }, use) => {
        await use(new DashboardPage(page));
    },
    productPage: async ({ page }, use) => {
        await use(new ProductPage(page));
    },
    cartPage: async ({ page }, use) => {
        await use(new CartPage(page));
    }
});

export { expect } from '@playwright/test';
```

```typescript
// tests/checkout.spec.ts
import { test, expect } from '../fixtures/pages';

test.describe('Checkout Flow', () => {

    test('should add product to cart', async ({ loginPage, productPage, cartPage }) => {
        // 로그인
        await loginPage.goto();
        await loginPage.login('user@example.com', 'password123');

        // 상품 추가
        await productPage.goto('PROD-001');
        await productPage.addToCart(2);

        // 장바구니 확인
        await cartPage.goto();
        await cartPage.expectItemCount(1);
        await cartPage.expectProductQuantity('PROD-001', 2);
    });
});
```

### Cypress Page Object 구현

```javascript
// cypress/support/pages/LoginPage.js
class LoginPage {
    visit() {
        cy.visit('/login');
        return this;
    }

    getEmailInput() {
        return cy.get('[data-testid="email-input"]');
    }

    getPasswordInput() {
        return cy.get('[data-testid="password-input"]');
    }

    getLoginButton() {
        return cy.get('[data-testid="login-button"]');
    }

    getErrorMessage() {
        return cy.get('.error-message');
    }

    login(email, password) {
        this.getEmailInput().type(email);
        this.getPasswordInput().type(password);
        this.getLoginButton().click();
        return this;
    }

    expectLoginSuccess() {
        cy.url().should('include', '/dashboard');
        return this;
    }

    expectError(message) {
        this.getErrorMessage()
            .should('be.visible')
            .and('contain', message);
        return this;
    }
}

export default new LoginPage();
```

```javascript
// cypress/e2e/login.cy.js
import loginPage from '../support/pages/LoginPage';
import dashboardPage from '../support/pages/DashboardPage';

describe('Login Flow', () => {

    it('should login successfully', () => {
        loginPage
            .visit()
            .login('user@example.com', 'password123')
            .expectLoginSuccess();

        dashboardPage.expectWelcomeMessage('User');
    });

    it('should show error for invalid credentials', () => {
        loginPage
            .visit()
            .login('wrong@example.com', 'wrongpassword')
            .expectError('Invalid credentials');
    });
});
```

---

## 테스트 안정성

### Flaky Test란?

동일한 코드에서 때때로 성공하고 때때로 실패하는 테스트.

```
실행 1: PASS ✓
실행 2: PASS ✓
실행 3: FAIL ✗  ← 코드 변경 없이 실패
실행 4: PASS ✓
```

### Flaky Test의 원인과 해결책

#### 1. 타이밍 문제

```typescript
// 문제: 하드코딩된 대기 시간
await page.click('#submit');
await page.waitForTimeout(3000);  // 3초 대기
const result = await page.locator('.result').textContent();

// 해결: 조건 기반 대기
await page.click('#submit');
await page.waitForSelector('.result', { state: 'visible' });
const result = await page.locator('.result').textContent();
```

```javascript
// Cypress - 자동 재시도 활용
// 문제
cy.get('.result').should('have.text', 'Success');  // 이미 좋음

// 더 복잡한 조건
cy.get('.result', { timeout: 10000 })
    .should('be.visible')
    .and('contain', 'Success');
```

#### 2. 동적 콘텐츠

```typescript
// 문제: 동적으로 로드되는 요소
await page.click('.item');  // 아직 렌더링 안 됨

// 해결: 요소가 준비될 때까지 대기
await page.waitForSelector('.item', { state: 'attached' });
await page.click('.item');

// 또는 auto-waiting 활용 (Playwright 기본 동작)
await page.locator('.item').click();  // 자동으로 대기
```

#### 3. 테스트 순서 의존성

```typescript
// 문제: 테스트 간 데이터 공유
test('should create user', async ({ page }) => {
    // 사용자 생성
    createdUserId = '123';  // 전역 변수!
});

test('should delete user', async ({ page }) => {
    // createdUserId 사용 - 순서 의존!
});

// 해결: 각 테스트의 독립성 보장
test('should delete user', async ({ page }) => {
    // 테스트 내에서 사용자 생성
    const userId = await createTestUser();
    await deleteUser(userId);
});
```

#### 4. 네트워크 불안정

```typescript
// 문제: 네트워크 오류에 취약
test('should load data', async ({ page }) => {
    await page.goto('/dashboard');
    // 네트워크 오류 시 실패
});

// 해결 1: 재시도 설정
// playwright.config.ts
{
    retries: process.env.CI ? 2 : 0
}

// 해결 2: 네트워크 요청 대기
await page.goto('/dashboard');
await page.waitForResponse(response =>
    response.url().includes('/api/data') && response.status() === 200
);
```

#### 5. 애니메이션

```typescript
// 문제: 애니메이션 중 클릭
await page.click('.animated-button');

// 해결 1: 애니메이션 비활성화
await page.addStyleTag({
    content: `*, *::before, *::after {
        animation-duration: 0s !important;
        transition-duration: 0s !important;
    }`
});

// 해결 2: force 옵션 사용
await page.click('.animated-button', { force: true });

// 해결 3: 애니메이션 완료 대기
await page.locator('.animated-button')
    .waitFor({ state: 'visible' });
await page.click('.animated-button');
```

### 안정성 향상 전략

```typescript
// 1. 견고한 로케이터 사용
// 나쁨
page.locator('.btn.primary.large.submit');
page.locator('body > div:nth-child(3) > button');

// 좋음
page.locator('[data-testid="submit-button"]');
page.getByRole('button', { name: 'Submit' });
page.getByText('Submit');
```

```typescript
// 2. 테스트 격리
test.beforeEach(async ({ page }) => {
    // 각 테스트 전 상태 초기화
    await page.evaluate(() => localStorage.clear());
    await resetDatabase();
});
```

```typescript
// 3. 실패 시 디버깅 정보 수집
// playwright.config.ts
{
    use: {
        trace: 'on-first-retry',
        screenshot: 'only-on-failure',
        video: 'retain-on-failure'
    }
}
```

```typescript
// 4. 커스텀 재시도 로직
async function retryAction<T>(
    action: () => Promise<T>,
    maxRetries: number = 3,
    delay: number = 1000
): Promise<T> {
    let lastError: Error | undefined;

    for (let i = 0; i < maxRetries; i++) {
        try {
            return await action();
        } catch (error) {
            lastError = error as Error;
            await new Promise(resolve => setTimeout(resolve, delay));
        }
    }

    throw lastError;
}

// 사용
await retryAction(async () => {
    await page.click('[data-testid="flaky-button"]');
    await expect(page.locator('.result')).toBeVisible();
});
```

### CI/CD 파이프라인 설정

```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps

      - name: Run E2E tests
        run: npx playwright test
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Upload test videos
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: test-videos
          path: test-results/
          retention-days: 7
```

---

## 참고 자료

- Playwright Documentation - https://playwright.dev/docs/intro
- Cypress Documentation - https://docs.cypress.io/
- Selenium Documentation - https://www.selenium.dev/documentation/
- Page Object Model Guide - https://playwright.dev/docs/pom
- Testing Best Practices - https://www.browserstack.com/guide/testing-pyramid-for-test-automation
- Flaky Test Prevention - https://playwright.dev/docs/test-retries

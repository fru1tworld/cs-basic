# TDD와 BDD (Test-Driven Development & Behavior-Driven Development)

## 목차
1. [TDD 개요](#tdd-개요)
2. [Red-Green-Refactor 사이클](#red-green-refactor-사이클)
3. [BDD와 Given-When-Then 패턴](#bdd와-given-when-then-패턴)
4. [BDD 프레임워크: Cucumber, Behave](#bdd-프레임워크-cucumber-behave)
5. [TDD의 장단점](#tdd의-장단점)
---

## TDD 개요

### 정의

TDD(Test-Driven Development)는 Kent Beck이 1990년대 후반 Extreme Programming의 일부로 개발한 소프트웨어 개발 방법론이다. **테스트를 먼저 작성하고, 그 테스트를 통과하는 코드를 작성**하는 개발 방식이다.

### 전통적 개발 vs TDD

```
전통적 개발:
요구사항 분석 → 설계 → 구현 → 테스트 작성 → 버그 수정

TDD:
요구사항 분석 → 테스트 작성 → 구현 → 리팩토링 → 반복
```

### TDD의 핵심 원칙

Kent Beck의 "Test-Driven Development by Example"에서 정의한 두 가지 규칙:

1. **새로운 코드는 실패하는 자동화된 테스트가 있을 때만 작성한다**
2. **중복을 제거한다**

```java
// 규칙 1: 실패하는 테스트 먼저
@Test
void shouldAddTwoNumbers() {
    Calculator calc = new Calculator();
    assertEquals(5, calc.add(2, 3));  // Calculator 클래스 없음 - 컴파일 에러!
}

// 규칙 2: 테스트를 통과한 후 중복 제거 (리팩토링)
```

### Canon TDD (Kent Beck, 2023)

Kent Beck이 최근 명확히 정의한 TDD의 목표:

```
TDD를 통해 달성하고자 하는 상태:
1. 기존에 동작하던 모든 것이 여전히 동작한다
2. 새로운 동작이 예상대로 작동한다
3. 시스템이 다음 변경을 받아들일 준비가 되어 있다
4. 프로그래머와 동료들이 위 사항들에 확신을 가진다
```

---

## Red-Green-Refactor 사이클

### 개념

TDD의 핵심 워크플로우는 세 단계로 구성된다:

```
    ┌─────────────────────────────────────────┐
    │                                         │
    ▼                                         │
┌───────┐     ┌───────┐     ┌──────────┐     │
│  RED  │────►│ GREEN │────►│ REFACTOR │─────┘
│(실패) │     │(성공) │     │ (정리)   │
└───────┘     └───────┘     └──────────┘
```

### 1단계: Red (실패하는 테스트 작성)

**목표**: 구현하려는 기능을 정의하는 테스트 작성

```java
// 요구사항: 사용자 비밀번호 유효성 검증
// - 최소 8자 이상
// - 대문자 1개 이상 포함
// - 숫자 1개 이상 포함

@Test
void shouldRejectPasswordShorterThan8Characters() {
    PasswordValidator validator = new PasswordValidator();

    boolean result = validator.isValid("Short1A");

    assertFalse(result);
}

// 컴파일 에러! PasswordValidator 클래스가 없음
```

**중요**: 테스트가 실패하는 것을 확인해야 한다. 이를 통해:
- 테스트가 실제로 실행되는지 확인
- 테스트가 의도한 것을 검증하는지 확인

### 2단계: Green (테스트 통과)

**목표**: 테스트를 통과하는 최소한의 코드 작성

```java
// 1차 구현: 가장 단순한 방법
public class PasswordValidator {

    public boolean isValid(String password) {
        return password.length() >= 8;  // 일단 길이만 체크
    }
}

// 테스트 통과! 하지만 요구사항의 일부만 구현
```

Kent Beck의 조언:
> "Make it work quickly, committing whatever sins necessary in the process."
> (빠르게 동작하게 만들어라. 과정에서 어떤 죄를 저질러도 좋다.)

### 3단계: Refactor (리팩토링)

**목표**: 중복 제거, 코드 정리, 설계 개선

```java
// 추가 테스트로 요구사항 완성
@Test
void shouldRejectPasswordWithoutUppercase() {
    assertFalse(validator.isValid("password1"));
}

@Test
void shouldRejectPasswordWithoutNumber() {
    assertFalse(validator.isValid("Password"));
}

@Test
void shouldAcceptValidPassword() {
    assertTrue(validator.isValid("Password1"));
}

// 리팩토링된 구현
public class PasswordValidator {

    private static final int MIN_LENGTH = 8;

    public boolean isValid(String password) {
        return hasMinimumLength(password)
            && hasUppercase(password)
            && hasNumber(password);
    }

    private boolean hasMinimumLength(String password) {
        return password.length() >= MIN_LENGTH;
    }

    private boolean hasUppercase(String password) {
        return password.chars().anyMatch(Character::isUpperCase);
    }

    private boolean hasNumber(String password) {
        return password.chars().anyMatch(Character::isDigit);
    }
}
```

### 실전 예제: 장바구니 기능

```java
// Step 1: RED - 실패하는 테스트
public class ShoppingCartTest {

    @Test
    void shouldBeEmptyWhenCreated() {
        ShoppingCart cart = new ShoppingCart();

        assertTrue(cart.isEmpty());
        assertEquals(0, cart.getTotalItems());
    }
}

// Step 2: GREEN - 최소 구현
public class ShoppingCart {
    public boolean isEmpty() {
        return true;
    }

    public int getTotalItems() {
        return 0;
    }
}

// Step 3: 다음 테스트 추가
@Test
void shouldNotBeEmptyAfterAddingItem() {
    ShoppingCart cart = new ShoppingCart();
    cart.addItem(new Item("Apple", 1000));

    assertFalse(cart.isEmpty());
    assertEquals(1, cart.getTotalItems());
}

// Step 4: 테스트 통과를 위한 구현
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();

    public void addItem(Item item) {
        items.add(item);
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public int getTotalItems() {
        return items.size();
    }
}

// Step 5: 계속해서 테스트 추가
@Test
void shouldCalculateTotalPrice() {
    ShoppingCart cart = new ShoppingCart();
    cart.addItem(new Item("Apple", 1000));
    cart.addItem(new Item("Banana", 500));

    assertEquals(1500, cart.getTotalPrice());
}

// Step 6: 구현 및 리팩토링
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();

    public void addItem(Item item) {
        items.add(item);
    }

    public int getTotalPrice() {
        return items.stream()
            .mapToInt(Item::getPrice)
            .sum();
    }

    // ... 기존 메서드
}
```

### Python 예제

```python
# test_shopping_cart.py
import pytest
from shopping_cart import ShoppingCart, Item

class TestShoppingCart:

    # RED: 장바구니 생성 테스트
    def test_should_be_empty_when_created(self):
        cart = ShoppingCart()

        assert cart.is_empty() is True
        assert cart.total_items == 0

    # RED: 아이템 추가 테스트
    def test_should_not_be_empty_after_adding_item(self):
        cart = ShoppingCart()
        cart.add_item(Item("Apple", 1000))

        assert cart.is_empty() is False
        assert cart.total_items == 1

    # RED: 총액 계산 테스트
    def test_should_calculate_total_price(self):
        cart = ShoppingCart()
        cart.add_item(Item("Apple", 1000))
        cart.add_item(Item("Banana", 500))

        assert cart.total_price == 1500

    # RED: 할인 적용 테스트
    def test_should_apply_percentage_discount(self):
        cart = ShoppingCart()
        cart.add_item(Item("Laptop", 1000000))

        cart.apply_discount(10)  # 10% 할인

        assert cart.total_price == 900000

    # RED: 동일 상품 수량 증가 테스트
    def test_should_increase_quantity_for_same_item(self):
        cart = ShoppingCart()
        cart.add_item(Item("Apple", 1000))
        cart.add_item(Item("Apple", 1000))

        assert cart.total_items == 1  # 아이템 종류는 1개
        assert cart.get_quantity("Apple") == 2  # 수량은 2개
        assert cart.total_price == 2000


# shopping_cart.py - GREEN + REFACTOR 후 최종 구현
from dataclasses import dataclass
from typing import Dict

@dataclass
class Item:
    name: str
    price: int

class ShoppingCart:
    def __init__(self):
        self._items: Dict[str, Dict] = {}
        self._discount_percent: int = 0

    def add_item(self, item: Item) -> None:
        if item.name in self._items:
            self._items[item.name]['quantity'] += 1
        else:
            self._items[item.name] = {
                'item': item,
                'quantity': 1
            }

    @property
    def is_empty(self) -> bool:
        return len(self._items) == 0

    @property
    def total_items(self) -> int:
        return len(self._items)

    @property
    def total_price(self) -> int:
        subtotal = sum(
            data['item'].price * data['quantity']
            for data in self._items.values()
        )
        discount = subtotal * self._discount_percent // 100
        return subtotal - discount

    def get_quantity(self, item_name: str) -> int:
        return self._items.get(item_name, {}).get('quantity', 0)

    def apply_discount(self, percent: int) -> None:
        if not 0 <= percent <= 100:
            raise ValueError("Discount must be between 0 and 100")
        self._discount_percent = percent
```

### JavaScript/TypeScript 예제

```typescript
// cart.test.ts
import { ShoppingCart, Item } from './cart';

describe('ShoppingCart', () => {

    describe('when created', () => {
        it('should be empty', () => {
            const cart = new ShoppingCart();

            expect(cart.isEmpty()).toBe(true);
            expect(cart.totalItems).toBe(0);
        });
    });

    describe('when adding items', () => {
        it('should not be empty after adding an item', () => {
            const cart = new ShoppingCart();

            cart.addItem(new Item('Apple', 1000));

            expect(cart.isEmpty()).toBe(false);
            expect(cart.totalItems).toBe(1);
        });

        it('should calculate total price correctly', () => {
            const cart = new ShoppingCart();

            cart.addItem(new Item('Apple', 1000));
            cart.addItem(new Item('Banana', 500));

            expect(cart.totalPrice).toBe(1500);
        });
    });

    describe('when applying discount', () => {
        it('should apply percentage discount', () => {
            const cart = new ShoppingCart();
            cart.addItem(new Item('Laptop', 1000000));

            cart.applyDiscount(10);

            expect(cart.totalPrice).toBe(900000);
        });

        it('should throw error for invalid discount', () => {
            const cart = new ShoppingCart();

            expect(() => cart.applyDiscount(150))
                .toThrow('Discount must be between 0 and 100');
        });
    });
});
```

---

## BDD와 Given-When-Then 패턴

### BDD란?

BDD(Behavior-Driven Development)는 Dan North가 2006년에 제안한 개발 방법론으로, TDD를 비즈니스 관점에서 발전시킨 것이다. 기술적 테스트 대신 **비즈니스 행동(Behavior)**에 초점을 맞춘다.

### TDD vs BDD

```
TDD:
"이 함수가 올바른 값을 반환하는가?"
"이 메서드가 예외를 던지는가?"

BDD:
"사용자가 로그인하면 대시보드를 볼 수 있다"
"VIP 고객이 주문하면 10% 할인을 받는다"
```

### Given-When-Then 패턴

```
Given: 초기 상태 (시나리오의 전제 조건)
When:  행동 (테스트하려는 동작)
Then:  기대 결과 (예상되는 결과)
```

```gherkin
Feature: 장바구니
  온라인 쇼핑몰 사용자로서
  상품을 장바구니에 담을 수 있어야 한다
  구매를 진행하기 위해

  Scenario: 장바구니에 상품 추가
    Given 빈 장바구니가 있다
    When 사용자가 "MacBook Pro"를 장바구니에 추가한다
    Then 장바구니에 1개의 상품이 있어야 한다
    And 장바구니 총액은 2,000,000원이어야 한다

  Scenario: 같은 상품을 여러 번 추가
    Given 장바구니에 "AirPods"가 1개 있다
    When 사용자가 "AirPods"를 장바구니에 추가한다
    Then 장바구니에 1개의 상품이 있어야 한다
    And "AirPods"의 수량은 2개여야 한다

  Scenario: VIP 고객 할인 적용
    Given "김철수" 고객은 VIP 등급이다
    And 장바구니에 100,000원 상당의 상품이 있다
    When 주문을 생성한다
    Then 10% 할인이 적용되어야 한다
    And 최종 결제 금액은 90,000원이어야 한다
```

### Gherkin 문법

```gherkin
Feature: 기능 설명
  사용자 스토리 형식으로 작성

  Background:
    # 모든 시나리오에서 공통으로 실행되는 전제 조건
    Given 사용자가 로그인되어 있다

  Scenario: 시나리오 이름
    Given 전제 조건
    And 추가 조건
    When 행동
    Then 결과
    But 제외 조건

  Scenario Outline: 파라미터화된 시나리오
    Given <상품>을 <수량>개 주문한다
    When 결제를 진행한다
    Then <금액>원이 청구되어야 한다

    Examples:
      | 상품       | 수량 | 금액     |
      | MacBook   | 1    | 2000000  |
      | iPhone    | 2    | 2400000  |
      | AirPods   | 3    | 750000   |

  @wip @slow
  Scenario: 태그가 있는 시나리오
    # @wip: 작업 중
    # @slow: 느린 테스트
    Given ...
```

---

## BDD 프레임워크: Cucumber, Behave

### 1. Cucumber (Java)

```java
// features/shopping_cart.feature
Feature: 장바구니
  Scenario: 상품 추가
    Given 빈 장바구니가 있다
    When 사용자가 "MacBook Pro"를 2000000원에 추가한다
    Then 장바구니에 1개의 상품이 있어야 한다
    And 총액은 2000000원이어야 한다
```

```java
// step_definitions/ShoppingCartSteps.java
import io.cucumber.java.en.*;
import static org.junit.jupiter.api.Assertions.*;

public class ShoppingCartSteps {

    private ShoppingCart cart;

    @Given("빈 장바구니가 있다")
    public void emptyCart() {
        cart = new ShoppingCart();
    }

    @Given("장바구니에 {string}가 {int}개 있다")
    public void cartWithItem(String itemName, int quantity) {
        cart = new ShoppingCart();
        for (int i = 0; i < quantity; i++) {
            cart.addItem(new Item(itemName, 0));
        }
    }

    @When("사용자가 {string}를 {int}원에 추가한다")
    public void addItemToCart(String itemName, int price) {
        cart.addItem(new Item(itemName, price));
    }

    @When("사용자가 {string}를 장바구니에 추가한다")
    public void addItemToCart(String itemName) {
        cart.addItem(new Item(itemName, 0));
    }

    @Then("장바구니에 {int}개의 상품이 있어야 한다")
    public void verifyItemCount(int expectedCount) {
        assertEquals(expectedCount, cart.getTotalItems());
    }

    @Then("총액은 {int}원이어야 한다")
    public void verifyTotalPrice(int expectedTotal) {
        assertEquals(expectedTotal, cart.getTotalPrice());
    }

    @Then("{string}의 수량은 {int}개여야 한다")
    public void verifyItemQuantity(String itemName, int expectedQuantity) {
        assertEquals(expectedQuantity, cart.getQuantity(itemName));
    }
}
```

```java
// Cucumber 실행 설정
// src/test/java/RunCucumberTest.java
import io.cucumber.junit.platform.engine.Cucumber;
import org.junit.platform.suite.api.*;

@Suite
@IncludeEngines("cucumber")
@SelectPackages("features")
@ConfigurationParameter(key = "cucumber.glue", value = "step_definitions")
@ConfigurationParameter(key = "cucumber.plugin", value = "pretty, html:target/cucumber-reports.html")
public class RunCucumberTest {
}
```

### 2. Behave (Python)

```gherkin
# features/shopping_cart.feature
Feature: 장바구니
  사용자가 상품을 장바구니에 담고 구매할 수 있다

  Background:
    Given 사용자가 로그인되어 있다

  Scenario: 상품 추가
    Given 빈 장바구니가 있다
    When 사용자가 "MacBook Pro"를 2000000원에 추가한다
    Then 장바구니에 1개의 상품이 있어야 한다
    And 총액은 2000000원이어야 한다

  Scenario Outline: 할인 적용
    Given 장바구니에 <금액>원 상당의 상품이 있다
    And 사용자 등급이 "<등급>"이다
    When 할인을 적용한다
    Then 최종 금액은 <최종금액>원이어야 한다

    Examples:
      | 금액    | 등급     | 최종금액 |
      | 100000 | 일반    | 100000   |
      | 100000 | VIP     | 90000    |
      | 100000 | VVIP    | 80000    |
```

```python
# features/steps/shopping_cart_steps.py
from behave import given, when, then
from shopping_cart import ShoppingCart, Item, User

@given('사용자가 로그인되어 있다')
def step_user_logged_in(context):
    context.user = User(name="테스트유저", grade="일반")

@given('빈 장바구니가 있다')
def step_empty_cart(context):
    context.cart = ShoppingCart()

@given('장바구니에 {price:d}원 상당의 상품이 있다')
def step_cart_with_items(context, price):
    context.cart = ShoppingCart()
    context.cart.add_item(Item("상품", price))

@given('사용자 등급이 "{grade}"이다')
def step_user_grade(context, grade):
    context.user.grade = grade

@when('사용자가 "{item_name}"를 {price:d}원에 추가한다')
def step_add_item(context, item_name, price):
    context.cart.add_item(Item(item_name, price))

@when('할인을 적용한다')
def step_apply_discount(context):
    discount_rates = {
        "일반": 0,
        "VIP": 10,
        "VVIP": 20
    }
    rate = discount_rates.get(context.user.grade, 0)
    context.cart.apply_discount(rate)

@then('장바구니에 {count:d}개의 상품이 있어야 한다')
def step_verify_item_count(context, count):
    assert context.cart.total_items == count, \
        f"Expected {count} items, got {context.cart.total_items}"

@then('총액은 {price:d}원이어야 한다')
def step_verify_total(context, price):
    assert context.cart.total_price == price, \
        f"Expected {price}, got {context.cart.total_price}"

@then('최종 금액은 {price:d}원이어야 한다')
def step_verify_final_price(context, price):
    assert context.cart.total_price == price, \
        f"Expected {price}, got {context.cart.total_price}"
```

```python
# features/environment.py - 환경 설정
def before_scenario(context, scenario):
    """각 시나리오 전에 실행"""
    context.cart = None
    context.user = None

def after_scenario(context, scenario):
    """각 시나리오 후에 실행"""
    pass

def before_feature(context, feature):
    """각 피처 파일 전에 실행"""
    pass
```

### 3. Cucumber.js (JavaScript/TypeScript)

```gherkin
# features/login.feature
Feature: 로그인
  사용자가 서비스에 로그인할 수 있다

  Scenario: 성공적인 로그인
    Given 회원가입된 사용자 "user@example.com"이 있다
    When 사용자가 "user@example.com"과 "password123"으로 로그인한다
    Then 대시보드 페이지가 표시되어야 한다
    And 환영 메시지에 사용자 이름이 포함되어야 한다

  Scenario: 잘못된 비밀번호
    Given 회원가입된 사용자 "user@example.com"이 있다
    When 사용자가 "user@example.com"과 "wrongpassword"로 로그인한다
    Then 에러 메시지 "Invalid credentials"가 표시되어야 한다
```

```typescript
// features/step_definitions/login.steps.ts
import { Given, When, Then } from '@cucumber/cucumber';
import { expect } from '@playwright/test';
import { LoginPage } from '../../pages/LoginPage';
import { DashboardPage } from '../../pages/DashboardPage';

let loginPage: LoginPage;
let dashboardPage: DashboardPage;

Given('회원가입된 사용자 {string}이 있다', async function(email: string) {
    // 테스트 사용자 생성 또는 확인
    await this.createTestUser(email, 'password123');
    loginPage = new LoginPage(this.page);
});

When('사용자가 {string}과 {string}으로 로그인한다',
    async function(email: string, password: string) {
        await loginPage.goto();
        await loginPage.login(email, password);
    }
);

Then('대시보드 페이지가 표시되어야 한다', async function() {
    dashboardPage = new DashboardPage(this.page);
    await expect(this.page).toHaveURL(/.*dashboard/);
});

Then('환영 메시지에 사용자 이름이 포함되어야 한다', async function() {
    await dashboardPage.expectWelcomeMessage();
});

Then('에러 메시지 {string}가 표시되어야 한다', async function(message: string) {
    await loginPage.expectErrorMessage(message);
});
```

```typescript
// cucumber.js 설정
module.exports = {
    default: {
        require: ['features/step_definitions/**/*.ts'],
        requireModule: ['ts-node/register'],
        format: [
            'progress-bar',
            ['html', 'reports/cucumber-report.html'],
            ['json', 'reports/cucumber-report.json']
        ],
        formatOptions: { snippetInterface: 'async-await' }
    }
};
```

---

## TDD의 장단점

### 장점

#### 1. 설계 개선

```java
// TDD 없이 작성된 코드 - 테스트하기 어려움
public class OrderService {
    public Order createOrder(Long userId) {
        User user = new UserRepository().findById(userId);  // 직접 생성
        LocalDateTime now = LocalDateTime.now();             // 시간 직접 호출
        EmailService emailService = new EmailService();      // 직접 생성
        // ...
    }
}

// TDD로 작성된 코드 - 테스트하기 쉬움
public class OrderService {
    private final UserRepository userRepository;
    private final Clock clock;
    private final EmailService emailService;

    public OrderService(UserRepository userRepository, Clock clock, EmailService emailService) {
        this.userRepository = userRepository;
        this.clock = clock;
        this.emailService = emailService;
    }

    public Order createOrder(Long userId) {
        User user = userRepository.findById(userId);
        LocalDateTime now = LocalDateTime.now(clock);
        // ...
    }
}
```

#### 2. 문서화 효과

```java
// 테스트가 명세 역할
@Test
void 신규_회원가입시_포인트_1000점_지급() {
    // Given
    MemberService service = new MemberService();
    MemberJoinRequest request = new MemberJoinRequest("user@example.com");

    // When
    Member member = service.join(request);

    // Then
    assertEquals(1000, member.getPoint());
}

@Test
void VIP_등급은_10만원_이상_구매시_자동_승급() {
    // 테스트 이름만 봐도 비즈니스 규칙을 알 수 있음
}
```

#### 3. 회귀 버그 방지

```java
// 6개월 전에 작성한 코드를 수정할 때
// 기존 테스트가 있으면 안전하게 리팩토링 가능

@Test
void 기존_기능이_여전히_동작한다() {
    // 새 코드 추가 후에도 이 테스트가 통과해야 함
}
```

#### 4. 디버깅 시간 단축

```
TDD 없이: 버그 발생 → 원인 추적 (어디서?) → 수정 → 수동 테스트

TDD로:    테스트 실패 → 실패한 테스트가 원인 지점 → 수정 → 자동 검증
```

### 단점

#### 1. 초기 개발 속도 저하

```
기능 구현에 1시간 걸리는 경우:
- TDD 없이: 구현 1시간
- TDD로:    테스트 30분 + 구현 1시간 + 리팩토링 30분 = 2시간

하지만 장기적으로는 버그 수정/리팩토링 시간 감소
```

#### 2. 잘못된 TDD 적용

```java
// 안티패턴: 구현을 그대로 테스트
@Test
void testGetName() {
    User user = new User("John");
    assertEquals("John", user.getName());  // 의미 없는 테스트
}

// 안티패턴: private 메서드 테스트
@Test
void testCalculateInternalValue() {
    // private 메서드는 public 메서드를 통해 간접 테스트해야 함
}

// 안티패턴: 테스트와 구현의 과도한 결합
@Test
void testUserService() {
    UserService service = new UserService();
    service.createUser("John");

    // 내부 구현에 의존
    verify(mockRepo, times(1)).save(any());
    verify(mockEmail, times(1)).send(any());
}
```

#### 3. 테스트 유지보수 부담

```java
// 요구사항 변경 시 많은 테스트 수정 필요
// Before: 사용자명은 필수
@Test
void shouldFailWithoutName() {
    assertThrows(ValidationException.class,
        () -> userService.createUser(null, "email@example.com"));
}

// After: 사용자명은 선택 사항으로 변경
// → 관련된 모든 테스트 수정 필요
```

#### 4. 과도한 테스트로 인한 리팩토링 저항

```java
// 너무 많은 테스트가 구현 세부사항에 의존하면
// 리팩토링할 때 많은 테스트가 깨짐

// 해결: 행동(behavior) 테스트에 집중
// - "무엇을" 테스트할지 정하고, "어떻게"는 자유롭게
```

### 언제 TDD를 사용할까?

```
TDD 권장:
✓ 복잡한 비즈니스 로직
✓ 요구사항이 명확한 경우
✓ 레거시 코드 수정
✓ 버그 수정 (재발 방지)

TDD 보류:
✗ 프로토타이핑/탐색 단계
✗ UI/화면 레이아웃
✗ 빠르게 변경되는 요구사항
✗ 단순 CRUD
```

---

## 참고 자료

- Kent Beck, "Test-Driven Development: By Example" (2002)
- Kent Beck, "Canon TDD" (2023) - https://tidyfirst.substack.com/p/canon-tdd
- Dan North, "Introducing BDD" (2006)
- Martin Fowler, "TestDrivenDevelopment" - https://martinfowler.com/bliki/TestDrivenDevelopment.html
- Michael Feathers, "Working Effectively with Legacy Code"
- Cucumber Documentation - https://cucumber.io/docs/
- Behave Documentation - https://behave.readthedocs.io/
- Robert C. Martin (Uncle Bob), "The Cycles of TDD" - https://blog.cleancoder.com/uncle-bob/2014/12/17/TheCyclesOfTDD.html

# 단위 테스트 (Unit Testing)

## 목차
1. [테스트 피라미드](#테스트-피라미드)
2. [테스트 더블: Mock, Stub, Spy, Fake](#테스트-더블-mock-stub-spy-fake)
3. [테스트 커버리지](#테스트-커버리지)
4. [주요 테스트 프레임워크](#주요-테스트-프레임워크)
5. [AAA 패턴](#aaa-패턴-arrange-act-assert)
---

## 테스트 피라미드

### 개념

테스트 피라미드는 Mike Cohn이 그의 저서 "Succeeding with Agile"에서 처음 소개한 개념으로, 자동화된 테스트의 계층 구조를 시각화한 것이다.

```
        /\
       /  \        E2E 테스트 (10%)
      /----\       - 전체 시스템 검증
     /      \      - 느리고 비용이 높음
    /--------\
   /          \    통합 테스트 (20%)
  /            \   - 컴포넌트 간 상호작용
 /--------------\  - 중간 속도와 비용
/                \
/------------------\ 단위 테스트 (70%)
                     - 개별 함수/메서드
                     - 빠르고 비용이 낮음
```

### 권장 비율

| 테스트 유형 | 권장 비율 | 실행 속도 | 유지보수 비용 |
|------------|----------|----------|--------------|
| 단위 테스트 | 70% | 매우 빠름 (ms) | 낮음 |
| 통합 테스트 | 20% | 보통 (초) | 중간 |
| E2E 테스트 | 10% | 느림 (분) | 높음 |

### 단위 테스트의 특징

```java
// 좋은 단위 테스트의 특징: FIRST
// Fast - 빠르게 실행
// Independent - 다른 테스트에 의존하지 않음
// Repeatable - 어떤 환경에서도 동일한 결과
// Self-validating - 자동으로 성공/실패 판단
// Timely - 적시에 작성 (코드 작성 전후)

@Test
void shouldCalculateTotalPrice() {
    // 빠름: 외부 의존성 없음
    // 독립적: 다른 테스트에 영향받지 않음
    // 반복 가능: 항상 동일한 결과
    ShoppingCart cart = new ShoppingCart();
    cart.addItem(new Item("Apple", 1000));
    cart.addItem(new Item("Banana", 500));

    int total = cart.calculateTotal();

    assertEquals(1500, total);
}
```

### 안티패턴: 아이스크림 콘

```
    __________
   /          \     E2E 테스트 (많음)
  /            \    - 느리고 불안정
 /--------------\
/                \  통합 테스트 (중간)
\                /
 \              /   단위 테스트 (적음)
  \            /    - 피라미드의 역전!
   \__________/
```

아이스크림 콘 패턴은 E2E 테스트가 많고 단위 테스트가 적은 안티패턴이다. 이는 다음과 같은 문제를 야기한다:
- 테스트 실행 시간 증가
- 불안정한 테스트 (Flaky Tests)
- 버그 위치 파악 어려움
- 유지보수 비용 증가

---

## 테스트 더블: Mock, Stub, Spy, Fake

Gerard Meszaros가 "xUnit Test Patterns"에서 정의한 용어로, 테스트에서 실제 객체를 대체하는 객체들을 통칭한다.

### 1. Stub (스텁)

**정의**: 미리 정해진 응답을 반환하는 객체. 호출 여부는 검증하지 않는다.

```java
// Java with Mockito
public class OrderServiceTest {
    @Test
    void shouldApplyDiscountForPremiumUser() {
        // Stub: 항상 Premium 사용자를 반환
        UserRepository userRepository = mock(UserRepository.class);
        when(userRepository.findById(1L))
            .thenReturn(new User(1L, "John", UserType.PREMIUM));

        OrderService orderService = new OrderService(userRepository);
        Order order = orderService.createOrder(1L, 10000);

        assertEquals(9000, order.getFinalPrice()); // 10% 할인 적용
    }
}
```

```python
# Python with pytest
def test_should_apply_discount_for_premium_user(mocker):
    # Stub: find_by_id가 호출되면 Premium 사용자 반환
    mock_repo = mocker.Mock()
    mock_repo.find_by_id.return_value = User(1, "John", UserType.PREMIUM)

    order_service = OrderService(mock_repo)
    order = order_service.create_order(1, 10000)

    assert order.final_price == 9000
```

```javascript
// JavaScript with Jest
describe('OrderService', () => {
    it('should apply discount for premium user', () => {
        // Stub
        const userRepository = {
            findById: jest.fn().mockReturnValue({
                id: 1,
                name: 'John',
                type: 'PREMIUM'
            })
        };

        const orderService = new OrderService(userRepository);
        const order = orderService.createOrder(1, 10000);

        expect(order.finalPrice).toBe(9000);
    });
});
```

### 2. Mock (목)

**정의**: 기대하는 호출을 미리 정의하고, 실제 호출이 기대와 일치하는지 검증하는 객체.

```java
// Java with Mockito
@Test
void shouldSendEmailWhenOrderIsCreated() {
    EmailService emailService = mock(EmailService.class);
    OrderService orderService = new OrderService(emailService);

    orderService.createOrder(new Order(1L, "Product A"));

    // Mock 검증: 특정 메서드가 특정 인자로 호출되었는지 확인
    verify(emailService).sendEmail(
        eq("order-confirmation@example.com"),
        contains("Order #1 confirmed")
    );
}

@Test
void shouldNotSendEmailForDraftOrder() {
    EmailService emailService = mock(EmailService.class);
    OrderService orderService = new OrderService(emailService);

    orderService.saveDraft(new Order(1L, "Product A"));

    // 호출되지 않았음을 검증
    verify(emailService, never()).sendEmail(any(), any());
}
```

```python
# Python with unittest.mock
from unittest.mock import Mock, call

def test_should_send_email_when_order_created():
    email_service = Mock()
    order_service = OrderService(email_service)

    order_service.create_order(Order(1, "Product A"))

    # Mock 검증
    email_service.send_email.assert_called_once_with(
        "order-confirmation@example.com",
        "Order #1 confirmed"
    )
```

### 3. Spy (스파이)

**정의**: 실제 객체를 감싸서 호출 정보를 기록하면서 실제 메서드도 실행하는 객체.

```java
// Java with Mockito
@Test
void shouldLogAllDatabaseQueries() {
    // 실제 객체를 spy로 감싸기
    QueryLogger realLogger = new QueryLogger();
    QueryLogger spyLogger = spy(realLogger);

    DatabaseService dbService = new DatabaseService(spyLogger);
    dbService.executeQuery("SELECT * FROM users");
    dbService.executeQuery("SELECT * FROM orders");

    // 실제 로깅이 수행되면서 호출 정보도 기록됨
    verify(spyLogger, times(2)).log(any());

    // 특정 호출 순서 검증
    InOrder inOrder = inOrder(spyLogger);
    inOrder.verify(spyLogger).log(contains("users"));
    inOrder.verify(spyLogger).log(contains("orders"));
}
```

```javascript
// JavaScript with Jest
describe('Spy example', () => {
    it('should track method calls while executing real logic', () => {
        const calculator = {
            add: (a, b) => a + b,
            multiply: (a, b) => a * b
        };

        // Spy 생성
        const addSpy = jest.spyOn(calculator, 'add');

        const result = calculator.add(2, 3);

        expect(result).toBe(5); // 실제 로직 실행
        expect(addSpy).toHaveBeenCalledWith(2, 3); // 호출 정보 추적
        expect(addSpy).toHaveBeenCalledTimes(1);

        addSpy.mockRestore(); // 원래 상태로 복원
    });
});
```

### 4. Fake (페이크)

**정의**: 실제로 동작하는 구현체이지만, 프로덕션에서는 사용하기 어려운 단순화된 버전.

```java
// Fake: 인메모리 데이터베이스 구현
public class FakeUserRepository implements UserRepository {
    private Map<Long, User> storage = new HashMap<>();
    private AtomicLong idGenerator = new AtomicLong(0);

    @Override
    public User save(User user) {
        if (user.getId() == null) {
            user.setId(idGenerator.incrementAndGet());
        }
        storage.put(user.getId(), user);
        return user;
    }

    @Override
    public Optional<User> findById(Long id) {
        return Optional.ofNullable(storage.get(id));
    }

    @Override
    public List<User> findByName(String name) {
        return storage.values().stream()
            .filter(user -> user.getName().equals(name))
            .collect(Collectors.toList());
    }

    @Override
    public void deleteById(Long id) {
        storage.remove(id);
    }
}

// 테스트에서 사용
@Test
void shouldManageUsers() {
    UserRepository fakeRepo = new FakeUserRepository();
    UserService userService = new UserService(fakeRepo);

    User saved = userService.createUser("John", "john@example.com");
    User found = userService.findUser(saved.getId());

    assertEquals("John", found.getName());
}
```

```python
# Python Fake 예시
class FakeEmailService:
    def __init__(self):
        self.sent_emails = []

    def send(self, to: str, subject: str, body: str) -> bool:
        """실제 이메일은 보내지 않고 기록만 함"""
        self.sent_emails.append({
            'to': to,
            'subject': subject,
            'body': body
        })
        return True

    def get_sent_count(self) -> int:
        return len(self.sent_emails)

    def was_sent_to(self, email: str) -> bool:
        return any(e['to'] == email for e in self.sent_emails)


def test_notification_service():
    fake_email = FakeEmailService()
    notification_service = NotificationService(fake_email)

    notification_service.notify_user("user@example.com", "Welcome!")

    assert fake_email.get_sent_count() == 1
    assert fake_email.was_sent_to("user@example.com")
```

### 5. Dummy (더미)

**정의**: 전달은 되지만 실제로 사용되지 않는 객체. 매개변수를 채우기 위해 사용.

```java
@Test
void shouldCreateUserWithDefaultSettings() {
    // Dummy: 실제로 사용되지 않지만 생성자에 필요
    Logger dummyLogger = mock(Logger.class);

    UserService userService = new UserService(dummyLogger);
    User user = userService.createUser("John");

    assertNotNull(user);
}
```

### 비교표

| 유형 | 행동 제어 | 호출 검증 | 실제 로직 | 사용 시점 |
|-----|---------|---------|---------|---------|
| Dummy | X | X | X | 매개변수 채우기용 |
| Stub | O | X | X | 특정 응답 반환 필요 |
| Spy | O | O | O | 실제 객체 호출 추적 |
| Mock | O | O | X | 행동 검증 필요 |
| Fake | O | X | O (단순화) | 경량 대체 구현 필요 |

---

## 테스트 커버리지

### 1. Line Coverage (라인 커버리지)

**정의**: 테스트 중 실행된 코드 라인의 비율

```java
public int calculateDiscount(int price, boolean isPremium) {
    int discount = 0;           // Line 1
    if (isPremium) {            // Line 2
        discount = price / 10;  // Line 3
    }
    return price - discount;    // Line 4
}

// 테스트: calculateDiscount(1000, true)
// 실행된 라인: 1, 2, 3, 4
// Line Coverage: 4/4 = 100%

// 테스트: calculateDiscount(1000, false)
// 실행된 라인: 1, 2, 4
// Line Coverage: 3/4 = 75%
```

**장점**: 측정이 간단하고 직관적
**단점**: 조건문의 모든 분기를 검증하지 못함

### 2. Branch Coverage (분기 커버리지)

**정의**: 모든 조건문의 참/거짓 분기가 실행된 비율

```java
public String classify(int score) {
    if (score >= 90) {
        return "A";
    } else if (score >= 80) {
        return "B";
    } else if (score >= 70) {
        return "C";
    } else {
        return "F";
    }
}

// 분기 수: 8개 (4개의 조건 × 2가지 경로)
// score=95: 분기 1 (>=90 true)
// score=85: 분기 2 (>=90 false), 분기 3 (>=80 true)
// score=75: 분기 2, 분기 4 (>=80 false), 분기 5 (>=70 true)
// score=65: 분기 2, 분기 4, 분기 6 (>=70 false), 분기 7 (else)

// 모든 분기 커버에 필요한 테스트 케이스: 4개
```

```python
# 분기 커버리지가 중요한 예
def validate_user(user):
    if user is None:               # Branch 1-2
        return False
    if user.age < 0:               # Branch 3-4
        return False
    if user.age > 150:             # Branch 5-6
        return False
    if not user.name:              # Branch 7-8
        return False
    return True

# 100% 라인 커버리지지만 50% 분기 커버리지인 테스트:
def test_valid_user():
    user = User(name="John", age=30)
    assert validate_user(user) == True

# 100% 분기 커버리지를 위한 테스트:
def test_none_user():
    assert validate_user(None) == False

def test_negative_age():
    assert validate_user(User("John", -1)) == False

def test_excessive_age():
    assert validate_user(User("John", 200)) == False

def test_empty_name():
    assert validate_user(User("", 30)) == False

def test_valid_user():
    assert validate_user(User("John", 30)) == True
```

### 3. Path Coverage (경로 커버리지)

**정의**: 코드의 모든 가능한 실행 경로가 테스트된 비율

```java
public void process(boolean condA, boolean condB) {
    if (condA) {
        doA();
    }
    if (condB) {
        doB();
    }
}

// 가능한 경로:
// Path 1: condA=true, condB=true   → doA(), doB()
// Path 2: condA=true, condB=false  → doA()
// Path 3: condA=false, condB=true  → doB()
// Path 4: condA=false, condB=false → (nothing)

// Branch Coverage 100%: 2개 테스트 케이스로 가능
// Path Coverage 100%: 4개 테스트 케이스 필요
```

### 경로 폭발 문제

```java
public void complexMethod(boolean a, boolean b, boolean c, boolean d, boolean e) {
    if (a) { /* ... */ }
    if (b) { /* ... */ }
    if (c) { /* ... */ }
    if (d) { /* ... */ }
    if (e) { /* ... */ }
}

// 가능한 경로 수: 2^5 = 32개
// n개의 독립적인 조건 → 2^n 개의 경로
```

### 커버리지 도구 설정

```xml
<!-- Maven: JaCoCo 설정 -->
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
    <configuration>
        <rules>
            <rule>
                <element>CLASS</element>
                <limits>
                    <limit>
                        <counter>LINE</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.80</minimum>
                    </limit>
                    <limit>
                        <counter>BRANCH</counter>
                        <value>COVEREDRATIO</value>
                        <minimum>0.70</minimum>
                    </limit>
                </limits>
            </rule>
        </rules>
    </configuration>
</plugin>
```

```javascript
// Jest 커버리지 설정 (jest.config.js)
module.exports = {
    collectCoverage: true,
    coverageDirectory: 'coverage',
    coverageReporters: ['text', 'lcov', 'html'],
    coverageThreshold: {
        global: {
            branches: 70,
            functions: 80,
            lines: 80,
            statements: 80
        }
    },
    collectCoverageFrom: [
        'src/**/*.{js,ts}',
        '!src/**/*.test.{js,ts}',
        '!src/index.{js,ts}'
    ]
};
```

```ini
# pytest 커버리지 설정 (pytest.ini)
[pytest]
addopts = --cov=src --cov-report=html --cov-fail-under=80
```

### 커버리지의 한계

```java
// 100% 커버리지지만 버그가 있는 코드
public int divide(int a, int b) {
    return a / b;  // b=0 일 때 예외 발생!
}

@Test
void testDivide() {
    assertEquals(5, divide(10, 2));
    // Line Coverage: 100%
    // 하지만 b=0 케이스를 테스트하지 않음
}
```

> "높은 커버리지가 품질 좋은 테스트를 보장하지 않는다.
> 하지만 낮은 커버리지는 테스트가 부족하다는 것을 보장한다."

---

## 주요 테스트 프레임워크

### 1. JUnit 5 (Java)

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

class UserServiceTest {

    private UserRepository userRepository;
    private UserService userService;

    @BeforeEach
    void setUp() {
        userRepository = mock(UserRepository.class);
        userService = new UserService(userRepository);
    }

    @Test
    @DisplayName("사용자 생성 시 ID가 할당되어야 한다")
    void shouldAssignIdWhenCreatingUser() {
        // Given
        when(userRepository.save(any(User.class)))
            .thenAnswer(inv -> {
                User user = inv.getArgument(0);
                user.setId(1L);
                return user;
            });

        // When
        User user = userService.createUser("John", "john@example.com");

        // Then
        assertNotNull(user.getId());
        assertEquals("John", user.getName());
    }

    @ParameterizedTest
    @DisplayName("잘못된 이메일 형식은 예외를 발생시켜야 한다")
    @ValueSource(strings = {"invalid", "no-at-sign", "@missing-local", "missing-domain@"})
    void shouldThrowExceptionForInvalidEmail(String invalidEmail) {
        assertThrows(IllegalArgumentException.class, () ->
            userService.createUser("John", invalidEmail)
        );
    }

    @ParameterizedTest
    @CsvSource({
        "100, BRONZE",
        "500, SILVER",
        "1000, GOLD",
        "5000, PLATINUM"
    })
    void shouldAssignCorrectMembershipLevel(int points, String expectedLevel) {
        User user = new User("John", "john@example.com", points);
        assertEquals(expectedLevel, user.getMembershipLevel().name());
    }

    @Nested
    @DisplayName("사용자 검색 테스트")
    class FindUserTests {

        @Test
        void shouldFindUserById() {
            when(userRepository.findById(1L))
                .thenReturn(Optional.of(new User(1L, "John")));

            User found = userService.findById(1L);

            assertEquals("John", found.getName());
        }

        @Test
        void shouldThrowExceptionWhenUserNotFound() {
            when(userRepository.findById(anyLong()))
                .thenReturn(Optional.empty());

            assertThrows(UserNotFoundException.class, () ->
                userService.findById(999L)
            );
        }
    }

    @Test
    @Timeout(value = 500, unit = TimeUnit.MILLISECONDS)
    void shouldCompleteWithinTimeout() {
        // 성능 테스트
        userService.processLargeDataSet();
    }
}
```

### 2. pytest (Python)

```python
import pytest
from unittest.mock import Mock, patch, MagicMock
from datetime import datetime
from user_service import UserService, User, UserNotFoundException

class TestUserService:

    @pytest.fixture
    def mock_repository(self):
        """각 테스트마다 새로운 mock repository 생성"""
        return Mock()

    @pytest.fixture
    def user_service(self, mock_repository):
        return UserService(mock_repository)

    def test_should_create_user_with_id(self, user_service, mock_repository):
        # Given
        mock_repository.save.return_value = User(id=1, name="John", email="john@example.com")

        # When
        user = user_service.create_user("John", "john@example.com")

        # Then
        assert user.id == 1
        assert user.name == "John"
        mock_repository.save.assert_called_once()

    @pytest.mark.parametrize("invalid_email", [
        "invalid",
        "no-at-sign",
        "@missing-local",
        "missing-domain@"
    ])
    def test_should_raise_for_invalid_email(self, user_service, invalid_email):
        with pytest.raises(ValueError, match="Invalid email"):
            user_service.create_user("John", invalid_email)

    @pytest.mark.parametrize("points,expected_level", [
        (100, "BRONZE"),
        (500, "SILVER"),
        (1000, "GOLD"),
        (5000, "PLATINUM"),
    ])
    def test_should_assign_correct_membership(self, points, expected_level):
        user = User(name="John", email="john@example.com", points=points)
        assert user.membership_level == expected_level

    def test_should_find_user_by_id(self, user_service, mock_repository):
        mock_repository.find_by_id.return_value = User(id=1, name="John")

        found = user_service.find_by_id(1)

        assert found.name == "John"

    def test_should_raise_when_user_not_found(self, user_service, mock_repository):
        mock_repository.find_by_id.return_value = None

        with pytest.raises(UserNotFoundException):
            user_service.find_by_id(999)

    @pytest.mark.slow
    def test_should_process_large_dataset(self, user_service):
        """느린 테스트는 별도 마커로 관리"""
        result = user_service.process_large_dataset()
        assert result.success is True

    @patch('user_service.datetime')
    def test_should_set_created_at_to_current_time(self, mock_datetime, user_service, mock_repository):
        # Given
        fixed_time = datetime(2024, 1, 15, 10, 30, 0)
        mock_datetime.now.return_value = fixed_time
        mock_repository.save.side_effect = lambda u: u

        # When
        user = user_service.create_user("John", "john@example.com")

        # Then
        assert user.created_at == fixed_time


class TestUserValidation:
    """사용자 유효성 검증 테스트 그룹"""

    def test_name_should_not_be_empty(self):
        with pytest.raises(ValueError, match="Name cannot be empty"):
            User(name="", email="john@example.com")

    def test_name_should_not_exceed_max_length(self):
        long_name = "a" * 101
        with pytest.raises(ValueError, match="Name too long"):
            User(name=long_name, email="john@example.com")
```

### 3. Jest (JavaScript/TypeScript)

```typescript
import { UserService } from './UserService';
import { UserRepository } from './UserRepository';
import { EmailService } from './EmailService';
import { User, UserNotFoundException } from './types';

// Mock 모듈
jest.mock('./UserRepository');
jest.mock('./EmailService');

describe('UserService', () => {
    let userService: UserService;
    let mockRepository: jest.Mocked<UserRepository>;
    let mockEmailService: jest.Mocked<EmailService>;

    beforeEach(() => {
        // 각 테스트 전 mock 초기화
        jest.clearAllMocks();

        mockRepository = new UserRepository() as jest.Mocked<UserRepository>;
        mockEmailService = new EmailService() as jest.Mocked<EmailService>;
        userService = new UserService(mockRepository, mockEmailService);
    });

    describe('createUser', () => {
        it('should create user with generated id', async () => {
            // Given
            mockRepository.save.mockResolvedValue({
                id: 1,
                name: 'John',
                email: 'john@example.com'
            });

            // When
            const user = await userService.createUser('John', 'john@example.com');

            // Then
            expect(user.id).toBe(1);
            expect(mockRepository.save).toHaveBeenCalledWith(
                expect.objectContaining({
                    name: 'John',
                    email: 'john@example.com'
                })
            );
        });

        it('should send welcome email after user creation', async () => {
            // Given
            mockRepository.save.mockResolvedValue({
                id: 1,
                name: 'John',
                email: 'john@example.com'
            });
            mockEmailService.send.mockResolvedValue(true);

            // When
            await userService.createUser('John', 'john@example.com');

            // Then
            expect(mockEmailService.send).toHaveBeenCalledWith(
                'john@example.com',
                'Welcome!',
                expect.stringContaining('John')
            );
        });

        it.each([
            ['invalid', 'no @ symbol'],
            ['no-at-sign', 'no @ symbol'],
            ['@missing-local', 'missing local part'],
            ['missing-domain@', 'missing domain'],
        ])('should throw error for invalid email: %s (%s)', async (email, _reason) => {
            await expect(
                userService.createUser('John', email)
            ).rejects.toThrow('Invalid email');
        });
    });

    describe('findById', () => {
        it('should return user when found', async () => {
            // Given
            mockRepository.findById.mockResolvedValue({
                id: 1,
                name: 'John',
                email: 'john@example.com'
            });

            // When
            const user = await userService.findById(1);

            // Then
            expect(user.name).toBe('John');
        });

        it('should throw UserNotFoundException when not found', async () => {
            // Given
            mockRepository.findById.mockResolvedValue(null);

            // When & Then
            await expect(userService.findById(999)).rejects.toThrow(
                UserNotFoundException
            );
        });
    });

    describe('membership level calculation', () => {
        it.each([
            [100, 'BRONZE'],
            [500, 'SILVER'],
            [1000, 'GOLD'],
            [5000, 'PLATINUM'],
        ])('should assign %s points to %s level', (points, expectedLevel) => {
            const user: User = { id: 1, name: 'John', email: 'john@example.com', points };
            expect(userService.calculateMembershipLevel(user)).toBe(expectedLevel);
        });
    });
});

// 비동기 테스트 고급 패턴
describe('Async patterns', () => {
    it('should handle concurrent operations', async () => {
        const promises = Array.from({ length: 10 }, (_, i) =>
            userService.createUser(`User${i}`, `user${i}@example.com`)
        );

        const users = await Promise.all(promises);

        expect(users).toHaveLength(10);
        expect(mockRepository.save).toHaveBeenCalledTimes(10);
    });

    it('should timeout for slow operations', async () => {
        mockRepository.findById.mockImplementation(
            () => new Promise(resolve => setTimeout(resolve, 5000))
        );

        await expect(
            Promise.race([
                userService.findById(1),
                new Promise((_, reject) =>
                    setTimeout(() => reject(new Error('Timeout')), 1000)
                )
            ])
        ).rejects.toThrow('Timeout');
    }, 10000);
});
```

---

## AAA 패턴 (Arrange-Act-Assert)

### 개념

AAA 패턴은 테스트를 세 단계로 구조화하는 패턴이다:
- **Arrange**: 테스트 환경 설정 (준비)
- **Act**: 테스트 대상 실행 (실행)
- **Assert**: 결과 검증 (검증)

### 기본 예시

```java
@Test
void shouldCalculateTotalWithDiscount() {
    // Arrange (Given) - 테스트 환경 준비
    ShoppingCart cart = new ShoppingCart();
    cart.addItem(new Item("Laptop", 1000000));
    cart.addItem(new Item("Mouse", 50000));
    Discount discount = new PercentageDiscount(10);

    // Act (When) - 테스트 대상 실행
    int total = cart.calculateTotal(discount);

    // Assert (Then) - 결과 검증
    assertEquals(945000, total);  // (1000000 + 50000) * 0.9
}
```

### Given-When-Then vs AAA

```python
# BDD 스타일: Given-When-Then
def test_should_apply_discount_for_bulk_order():
    # Given: 10개 이상의 상품이 담긴 장바구니
    cart = ShoppingCart()
    for i in range(10):
        cart.add_item(Item(f"Product{i}", 10000))

    # When: 총액 계산
    total = cart.calculate_total()

    # Then: 대량 구매 할인 적용
    assert total == 90000  # 10% 할인

# 동일한 테스트를 AAA 스타일로
def test_bulk_order_discount():
    # Arrange
    cart = ShoppingCart()
    for i in range(10):
        cart.add_item(Item(f"Product{i}", 10000))

    # Act
    total = cart.calculate_total()

    # Assert
    assert total == 90000
```

### 좋은 AAA 패턴 적용

```typescript
describe('OrderService', () => {
    describe('processOrder', () => {
        it('should decrease inventory and send confirmation email', async () => {
            // ========== Arrange ==========
            // 의존성 설정
            const mockInventory = {
                decrease: jest.fn().mockResolvedValue(true),
                getStock: jest.fn().mockReturnValue(100)
            };
            const mockEmailService = {
                send: jest.fn().mockResolvedValue(true)
            };
            const orderService = new OrderService(mockInventory, mockEmailService);

            // 테스트 데이터 준비
            const order = {
                id: 'ORD-001',
                items: [
                    { productId: 'PRD-001', quantity: 2 },
                    { productId: 'PRD-002', quantity: 1 }
                ],
                customerEmail: 'customer@example.com'
            };

            // ========== Act ==========
            const result = await orderService.processOrder(order);

            // ========== Assert ==========
            // 결과 검증
            expect(result.status).toBe('COMPLETED');
            expect(result.processedAt).toBeInstanceOf(Date);

            // 재고 감소 검증
            expect(mockInventory.decrease).toHaveBeenCalledTimes(2);
            expect(mockInventory.decrease).toHaveBeenCalledWith('PRD-001', 2);
            expect(mockInventory.decrease).toHaveBeenCalledWith('PRD-002', 1);

            // 이메일 발송 검증
            expect(mockEmailService.send).toHaveBeenCalledWith(
                'customer@example.com',
                expect.stringContaining('Order Confirmation'),
                expect.any(String)
            );
        });
    });
});
```

### AAA 패턴 안티패턴

```java
// 안티패턴 1: Act가 여러 개
@Test
void badTest() {
    // Arrange
    User user = new User("John");

    // Act 1
    user.setAge(25);
    assertEquals(25, user.getAge());  // Assert 1

    // Act 2
    user.setEmail("john@example.com");
    assertEquals("john@example.com", user.getEmail());  // Assert 2

    // 문제: 하나의 테스트에서 여러 동작을 검증
    // 해결: 각각 별도의 테스트로 분리
}

// 안티패턴 2: Assert가 너무 많음
@Test
void tooManyAsserts() {
    Order order = orderService.createOrder(customer, items);

    assertNotNull(order);
    assertNotNull(order.getId());
    assertEquals("PENDING", order.getStatus());
    assertEquals(customer.getId(), order.getCustomerId());
    assertEquals(3, order.getItems().size());
    assertTrue(order.getTotal() > 0);
    assertNotNull(order.getCreatedAt());
    // ... 더 많은 assertion

    // 문제: 하나의 실패가 다른 assertion을 가림
    // 해결: 논리적으로 관련된 assertion 그룹화 또는 분리
}

// 개선된 버전
@Test
void shouldCreateOrderWithCorrectCustomer() {
    Order order = orderService.createOrder(customer, items);
    assertEquals(customer.getId(), order.getCustomerId());
}

@Test
void shouldCreateOrderWithPendingStatus() {
    Order order = orderService.createOrder(customer, items);
    assertEquals("PENDING", order.getStatus());
}

@Test
void shouldCalculateOrderTotal() {
    Order order = orderService.createOrder(customer, items);
    assertEquals(expectedTotal, order.getTotal());
}
```

### 테스트 헬퍼 메서드 활용

```java
class OrderServiceTest {

    // Arrange 헬퍼 메서드
    private Customer createTestCustomer() {
        return Customer.builder()
            .id(1L)
            .name("Test Customer")
            .email("test@example.com")
            .build();
    }

    private List<OrderItem> createTestItems(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> new OrderItem("Product" + i, 1000 * (i + 1), 1))
            .collect(Collectors.toList());
    }

    // Assert 헬퍼 메서드
    private void assertOrderIsValid(Order order) {
        assertAll("Order validation",
            () -> assertNotNull(order.getId(), "Order ID should not be null"),
            () -> assertNotNull(order.getCreatedAt(), "Created date should not be null"),
            () -> assertTrue(order.getTotal() > 0, "Total should be positive")
        );
    }

    @Test
    void shouldCreateValidOrder() {
        // Arrange - 헬퍼 메서드로 간결하게
        Customer customer = createTestCustomer();
        List<OrderItem> items = createTestItems(3);

        // Act
        Order order = orderService.createOrder(customer, items);

        // Assert - 헬퍼 메서드로 가독성 향상
        assertOrderIsValid(order);
        assertEquals(customer.getId(), order.getCustomerId());
    }
}
```

---

## 참고 자료

- Kent Beck, "Test-Driven Development by Example"
- Gerard Meszaros, "xUnit Test Patterns"
- Martin Fowler, "Mocks Aren't Stubs" - https://martinfowler.com/articles/mocksArentStubs.html
- Mike Cohn, "Succeeding with Agile"
- JUnit 5 User Guide - https://junit.org/junit5/docs/current/user-guide/
- pytest Documentation - https://docs.pytest.org/
- Jest Documentation - https://jestjs.io/docs/getting-started

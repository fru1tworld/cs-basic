# 도메인 주도 설계 (Domain-Driven Design, DDD)

## 목차
1. [개요](#개요)
2. [전략적 설계](#전략적-설계)
3. [전술적 설계](#전술적-설계)
4. [유비쿼터스 언어](#유비쿼터스-언어)
5. [Domain Event](#domain-event)
6. [Anti-Corruption Layer](#anti-corruption-layer)
7. [Event Storming](#event-storming)
8. [실무 구현 예제](#실무-구현-예제)
---

## 개요

도메인 주도 설계(DDD)는 Eric Evans가 2003년에 제안한 소프트웨어 개발 방법론으로, **복잡한 도메인 문제를 해결하기 위해 도메인 모델을 중심으로 설계**하는 접근법이다.

### DDD의 핵심 철학

```
┌─────────────────────────────────────────────────────────────────────┐
│                       DDD의 핵심 철학                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  "소프트웨어의 복잡성은 도메인 자체의 복잡성에서 비롯된다"              │
│                                            - Eric Evans              │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     DDD의 3가지 핵심                         │    │
│  │                                                              │    │
│  │  1. 도메인 전문가와 개발자의 긴밀한 협업                       │    │
│  │     → 지식을 코드로 표현                                     │    │
│  │                                                              │    │
│  │  2. 유비쿼터스 언어 (Ubiquitous Language)                    │    │
│  │     → 모든 이해관계자가 동일한 용어 사용                       │    │
│  │                                                              │    │
│  │  3. 모델 주도 설계 (Model-Driven Design)                     │    │
│  │     → 도메인 모델이 곧 코드                                   │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  DDD의 구성:                                                         │
│  ┌─────────────────┐     ┌─────────────────┐                        │
│  │  전략적 설계    │     │  전술적 설계     │                        │
│  │  (Strategic)   │     │  (Tactical)     │                        │
│  ├─────────────────┤     ├─────────────────┤                        │
│  │ - Bounded Context│    │ - Entity        │                        │
│  │ - Context Map   │     │ - Value Object  │                        │
│  │ - Subdomain     │     │ - Aggregate     │                        │
│  │ - Core Domain   │     │ - Repository    │                        │
│  │                 │     │ - Domain Event  │                        │
│  │                 │     │ - Domain Service│                        │
│  └─────────────────┘     └─────────────────┘                        │
│                                                                      │
│  WHERE (어디에?)           WHAT & HOW (무엇을? 어떻게?)              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 전략적 설계

전략적 설계는 큰 그림에서 도메인을 어떻게 나누고 조직화할지 결정하는 단계이다.

### Bounded Context (경계 컨텍스트)

Bounded Context는 **특정 도메인 모델이 유효한 범위**를 정의한다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Bounded Context 예시                            │
│                     (E-Commerce 도메인)                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  동일한 "상품(Product)"이 컨텍스트마다 다른 의미를 가짐                │
│                                                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  상품 카탈로그    │  │    주문 관리      │  │    배송 관리      │   │
│  │   Context       │  │    Context       │  │    Context       │   │
│  ├──────────────────┤  ├──────────────────┤  ├──────────────────┤   │
│  │                  │  │                  │  │                  │   │
│  │  Product:        │  │  Product:        │  │  Product:        │   │
│  │  - id           │  │  - id            │  │  - id            │   │
│  │  - name         │  │  - name          │  │  - weight        │   │
│  │  - description  │  │  - price         │  │  - dimensions    │   │
│  │  - images       │  │  - quantity      │  │  - fragile       │   │
│  │  - category     │  │                  │  │  - hazardous     │   │
│  │  - attributes   │  │                  │  │                  │   │
│  │                  │  │                  │  │                  │   │
│  │  [상세 정보]     │  │  [주문에 필요한]  │  │  [배송에 필요한]  │   │
│  │  [마케팅 관점]   │  │  [핵심 정보]      │  │  [물류 정보]      │   │
│  │                  │  │                  │  │                  │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘   │
│                                                                      │
│  ★ 핵심: 각 Context에서 "상품"은 서로 다른 속성과 행동을 가짐          │
│         같은 용어라도 Context가 다르면 다른 모델                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Subdomain (서브도메인)

서브도메인은 비즈니스 도메인을 더 작은 영역으로 나눈 것이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        서브도메인 분류                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Core Domain (핵심 도메인)                  │    │
│  │                                                               │    │
│  │  • 비즈니스 차별화 요소                                       │    │
│  │  • 가장 높은 투자 우선순위                                     │    │
│  │  • 가장 뛰어난 개발자 배치                                     │    │
│  │  • 예: 가격 책정 엔진, 추천 알고리즘, 매칭 시스템               │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │               Supporting Domain (지원 도메인)                 │    │
│  │                                                               │    │
│  │  • 비즈니스에 특화되어 있지만 핵심은 아님                       │    │
│  │  • 외부 솔루션으로 대체 어려움                                 │    │
│  │  • 중간 수준 투자                                             │    │
│  │  • 예: 재고 관리, 고객 지원, 주문 처리                         │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                Generic Domain (일반 도메인)                   │    │
│  │                                                               │    │
│  │  • 범용 솔루션으로 해결 가능                                   │    │
│  │  • 외부 서비스나 오픈소스 활용                                 │    │
│  │  • 최소 투자                                                  │    │
│  │  • 예: 인증(Auth0), 결제(Stripe), 이메일(SendGrid)            │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ★ 중요: 서브도메인과 Bounded Context는 1:1 대응이 이상적            │
│         하지만 실제로는 다를 수 있음 (레거시 시스템 등)               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Context Map (컨텍스트 맵)

Context Map은 여러 Bounded Context 간의 관계를 시각화한 것이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Context Map 패턴                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Shared Kernel (공유 커널)                                        │
│     ┌────────────┐         ┌────────────┐                           │
│     │ Context A  │◄───────►│ Context B  │                           │
│     └─────┬──────┘         └──────┬─────┘                           │
│           └──────────┬───────────┘                                  │
│                ┌─────┴─────┐                                        │
│                │  Shared   │  ← 양쪽이 공유하는 코드/모델             │
│                │  Kernel   │    변경 시 양쪽 동의 필요               │
│                └───────────┘                                        │
│                                                                      │
│  2. Customer-Supplier (고객-공급자)                                  │
│     ┌────────────┐                                                  │
│     │  Supplier  │ ──Upstream──►                                    │
│     │ (공급자)   │             ┌────────────┐                       │
│     └────────────┘             │  Customer  │                       │
│                    ◄──Downstream──│  (고객)   │                       │
│                                └────────────┘                       │
│     공급자가 고객의 요구사항을 수용                                    │
│                                                                      │
│  3. Conformist (순응자)                                              │
│     ┌────────────┐                                                  │
│     │  Upstream  │ ──────────►                                      │
│     │  (상류)    │            ┌────────────┐                        │
│     └────────────┘            │ Conformist │                        │
│                               │  (순응자)   │                        │
│                               └────────────┘                        │
│     하류가 상류 모델을 그대로 따름 (협상력 없음)                        │
│                                                                      │
│  4. Anti-Corruption Layer (부패 방지 계층)                           │
│     ┌────────────┐     ┌─────────┐     ┌────────────┐              │
│     │  Legacy    │────►│   ACL   │────►│    New     │              │
│     │  System    │     │ (변환)   │     │  Context   │              │
│     └────────────┘     └─────────┘     └────────────┘              │
│     외부/레거시 모델을 내 모델로 변환                                  │
│                                                                      │
│  5. Open Host Service + Published Language                          │
│     ┌────────────┐                                                  │
│     │   Service  │ ═══ API (REST/gRPC) ═══►  다수의 클라이언트       │
│     │  Provider  │     + 공개된 언어                                 │
│     └────────────┘     (JSON Schema, Protobuf)                      │
│                                                                      │
│  6. Separate Ways (분리된 길)                                        │
│     ┌────────────┐     ┌────────────┐                               │
│     │ Context A  │     │ Context B  │                               │
│     │            │  X  │            │  ← 의도적으로 통합 안 함        │
│     └────────────┘     └────────────┘                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Context Map 예시

```
┌─────────────────────────────────────────────────────────────────────┐
│                 E-Commerce Context Map 예시                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│              ┌─────────────────────────────────────┐                │
│              │         Product Catalog             │                │
│              │         (Core Domain)               │                │
│              └───────────────┬─────────────────────┘                │
│                              │ OHS/PL                               │
│                              │ (Open Host Service)                  │
│                              ▼                                       │
│     ┌────────────────────────┼────────────────────────┐             │
│     │                        │                        │              │
│     ▼                        ▼                        ▼              │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐         │
│  │   Order     │      │  Inventory  │      │   Search    │         │
│  │  Context    │      │  Context    │      │  Context    │         │
│  │(Supporting) │      │(Supporting) │      │(Supporting) │         │
│  └──────┬──────┘      └──────┬──────┘      └─────────────┘         │
│         │                    │                                      │
│         │ C/S                │ C/S                                  │
│         │(Customer/Supplier) │                                      │
│         ▼                    │                                      │
│  ┌─────────────┐             │                                      │
│  │  Payment    │◄────────────┘                                      │
│  │  Context    │                                                    │
│  │ (Generic)   │                                                    │
│  └──────┬──────┘                                                    │
│         │ ACL (Anti-Corruption Layer)                               │
│         ▼                                                           │
│  ┌─────────────┐                                                    │
│  │  External   │                                                    │
│  │   Payment   │  (Stripe, PG사 등)                                 │
│  │   Gateway   │                                                    │
│  └─────────────┘                                                    │
│                                                                      │
│  범례:                                                               │
│  OHS/PL = Open Host Service / Published Language                    │
│  C/S = Customer/Supplier                                            │
│  ACL = Anti-Corruption Layer                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 전술적 설계

전술적 설계는 각 Bounded Context 내에서 도메인 모델을 어떻게 구현할지 결정한다.

### Entity (엔티티)

엔티티는 **고유한 식별자(ID)를 가지며 생명주기 동안 연속성을 유지**하는 객체이다.

```java
// Entity - 식별자로 구분되는 도메인 객체
public class Order {

    // 식별자 (Identity) - 엔티티를 구분하는 기준
    private final OrderId id;

    // 상태 (Mutable)
    private CustomerId customerId;
    private List<OrderLine> orderLines;
    private OrderStatus status;
    private Money totalAmount;
    private LocalDateTime createdAt;
    private LocalDateTime confirmedAt;

    // 생성자 - 불변식(Invariant) 보장
    private Order(OrderId id, CustomerId customerId) {
        this.id = Objects.requireNonNull(id, "Order ID cannot be null");
        this.customerId = Objects.requireNonNull(customerId, "Customer ID cannot be null");
        this.orderLines = new ArrayList<>();
        this.status = OrderStatus.DRAFT;
        this.totalAmount = Money.ZERO;
        this.createdAt = LocalDateTime.now();
    }

    // 정적 팩토리 메서드
    public static Order create(CustomerId customerId) {
        return new Order(OrderId.generate(), customerId);
    }

    // 도메인 행동 (비즈니스 로직을 엔티티가 직접 수행)
    public void addOrderLine(Product product, Quantity quantity) {
        validateDraftStatus();
        validatePositiveQuantity(quantity);

        OrderLine newLine = OrderLine.create(this.id, product, quantity);
        this.orderLines.add(newLine);
        recalculateTotalAmount();
    }

    public void confirm() {
        validateDraftStatus();
        validateNotEmpty();

        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = LocalDateTime.now();
    }

    public void cancel(String reason) {
        validateCancellable();
        this.status = OrderStatus.CANCELLED;
    }

    // 불변식 검증 메서드
    private void validateDraftStatus() {
        if (this.status != OrderStatus.DRAFT) {
            throw new InvalidOrderStateException(
                "주문을 수정하려면 초안 상태여야 합니다. 현재 상태: " + this.status
            );
        }
    }

    private void validateCancellable() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new OrderCannotBeCancelledException(
                "배송 시작된 주문은 취소할 수 없습니다"
            );
        }
    }

    // 동등성: ID 기반
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order order = (Order) o;
        return id.equals(order.id);  // ID로만 비교!
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

### Value Object (값 객체)

Value Object는 **식별자가 없고, 속성의 조합으로 동등성을 판단**하는 불변 객체이다.

```java
// Value Object - 불변, 속성으로 동등성 비교
public final class Money {

    private final BigDecimal amount;
    private final Currency currency;

    // 정적 팩토리
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.KRW);

    public static Money of(BigDecimal amount, Currency currency) {
        return new Money(amount, currency);
    }

    public static Money won(long amount) {
        return new Money(BigDecimal.valueOf(amount), Currency.KRW);
    }

    // private 생성자 (불변식 검증)
    private Money(BigDecimal amount, Currency currency) {
        if (amount == null) {
            throw new IllegalArgumentException("금액은 null일 수 없습니다");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("금액은 0 이상이어야 합니다");
        }
        if (currency == null) {
            throw new IllegalArgumentException("통화는 null일 수 없습니다");
        }
        this.amount = amount;
        this.currency = currency;
    }

    // 비즈니스 연산 - 새 객체 반환 (불변성)
    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        validateSameCurrency(other);
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new InsufficientFundsException("결과 금액이 음수입니다");
        }
        return new Money(result, this.currency);
    }

    public Money multiply(int multiplier) {
        return new Money(
            this.amount.multiply(BigDecimal.valueOf(multiplier)),
            this.currency
        );
    }

    public Money multiply(BigDecimal multiplier) {
        return new Money(
            this.amount.multiply(multiplier).setScale(0, RoundingMode.HALF_UP),
            this.currency
        );
    }

    public boolean isGreaterThan(Money other) {
        validateSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    private void validateSameCurrency(Money other) {
        if (this.currency != other.currency) {
            throw new CurrencyMismatchException(
                "통화가 다릅니다: " + this.currency + " vs " + other.currency
            );
        }
    }

    // 동등성: 모든 속성으로 비교
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 &&
               currency == money.currency;
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return currency.getSymbol() + " " +
               amount.setScale(0, RoundingMode.HALF_UP).toPlainString();
    }
}

// 다른 Value Object 예시
public final class Address {
    private final String street;
    private final String city;
    private final String zipCode;
    private final String country;

    // 생성자, 검증, equals, hashCode...

    // 변경이 필요하면 새 객체 반환
    public Address withCity(String newCity) {
        return new Address(this.street, newCity, this.zipCode, this.country);
    }
}

public final class EmailAddress {
    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[A-Za-z0-9+_.-]+@(.+)$");

    private final String value;

    public EmailAddress(String value) {
        if (!EMAIL_PATTERN.matcher(value).matches()) {
            throw new InvalidEmailException("잘못된 이메일 형식: " + value);
        }
        this.value = value.toLowerCase();
    }

    public String getValue() {
        return value;
    }

    public String getDomain() {
        return value.substring(value.indexOf('@') + 1);
    }

    // equals, hashCode...
}
```

### Entity vs Value Object 비교

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Entity vs Value Object                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  특성              │    Entity          │    Value Object           │
│  ─────────────────┼───────────────────┼──────────────────────────  │
│  식별자           │    있음 (ID)       │    없음                    │
│  동등성           │    ID로 비교       │    모든 속성으로 비교       │
│  변경가능성        │    가변 (Mutable)  │    불변 (Immutable)        │
│  생명주기         │    있음            │    없음                    │
│  교체 가능성       │    교체 불가       │    언제든 교체 가능         │
│                                                                      │
│  예시:                                                               │
│  ─────────────────────────────────────────────────────────────────  │
│  Order (주문)          ←── Entity (주문번호로 식별)                  │
│  Customer (고객)       ←── Entity (고객ID로 식별)                    │
│  Money (금액)          ←── Value Object (10000원 == 10000원)        │
│  Address (주소)        ←── Value Object (속성이 같으면 같은 주소)     │
│  DateRange (기간)      ←── Value Object                             │
│  Quantity (수량)       ←── Value Object                             │
│                                                                      │
│  선택 기준:                                                          │
│  ─────────────────────────────────────────────────────────────────  │
│  • 연속성이 필요한가? → Entity                                       │
│  • 속성 값 자체가 의미인가? → Value Object                           │
│  • 교체해도 되는가? → Value Object                                   │
│                                                                      │
│  "의심스러우면 Value Object로 시작하라" - Vaughn Vernon              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Aggregate (애그리게이트)

Aggregate는 **데이터 변경의 단위로 취급되는 연관 객체들의 클러스터**이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Aggregate 개념                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Order Aggregate                           │    │
│  │  ┌───────────────────────────────────────────────────────┐  │    │
│  │  │                                                        │  │    │
│  │  │  ┌─────────────────┐                                  │  │    │
│  │  │  │  Order (Root)   │ ◄── Aggregate Root               │  │    │
│  │  │  │  - orderId      │     (유일한 진입점)               │  │    │
│  │  │  │  - status       │                                  │  │    │
│  │  │  │  - totalAmount  │                                  │  │    │
│  │  │  └────────┬────────┘                                  │  │    │
│  │  │           │                                           │  │    │
│  │  │           │ contains                                  │  │    │
│  │  │           ▼                                           │  │    │
│  │  │  ┌─────────────────┐                                  │  │    │
│  │  │  │   OrderLine     │ ◄── 내부 Entity                  │  │    │
│  │  │  │   - lineId      │     (외부에서 직접 접근 불가)      │  │    │
│  │  │  │   - productId   │                                  │  │    │
│  │  │  │   - quantity    │                                  │  │    │
│  │  │  │   - price       │ ◄── Value Object                 │  │    │
│  │  │  └─────────────────┘                                  │  │    │
│  │  │                                                        │  │    │
│  │  └───────────────────────────────────────────────────────┘  │    │
│  │                      트랜잭션 경계                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Aggregate 규칙:                                                     │
│  1. Root만 외부에서 참조 가능                                        │
│  2. 내부 객체는 Root를 통해서만 접근                                  │
│  3. 하나의 트랜잭션에서 하나의 Aggregate만 수정                       │
│  4. 다른 Aggregate는 ID로만 참조 (객체 참조 X)                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// Aggregate Root
public class Order {

    private final OrderId id;
    private CustomerId customerId;  // 다른 Aggregate는 ID로만 참조!
    private List<OrderLine> orderLines;  // 내부 Entity
    private ShippingAddress shippingAddress;  // Value Object
    private OrderStatus status;
    private Money totalAmount;

    // Aggregate 외부에서는 Root를 통해서만 내부 객체에 접근
    public void addOrderLine(ProductId productId, String productName,
                            Money price, Quantity quantity) {
        validateDraftStatus();

        // 내부 Entity 생성은 Root가 담당
        OrderLine line = new OrderLine(
            OrderLineId.generate(),
            productId,
            productName,
            price,
            quantity
        );

        this.orderLines.add(line);
        recalculateTotalAmount();
    }

    // 내부 Entity 수정도 Root를 통해
    public void updateOrderLineQuantity(OrderLineId lineId, Quantity newQuantity) {
        validateDraftStatus();

        OrderLine line = findOrderLine(lineId);
        line.updateQuantity(newQuantity);  // 내부적으로 수정
        recalculateTotalAmount();
    }

    public void removeOrderLine(OrderLineId lineId) {
        validateDraftStatus();

        OrderLine line = findOrderLine(lineId);
        this.orderLines.remove(line);
        recalculateTotalAmount();
    }

    // 불변 컬렉션 반환 (외부에서 직접 수정 방지)
    public List<OrderLine> getOrderLines() {
        return Collections.unmodifiableList(orderLines);
    }

    private OrderLine findOrderLine(OrderLineId lineId) {
        return orderLines.stream()
            .filter(line -> line.getId().equals(lineId))
            .findFirst()
            .orElseThrow(() -> new OrderLineNotFoundException(lineId));
    }

    private void recalculateTotalAmount() {
        this.totalAmount = orderLines.stream()
            .map(OrderLine::getLineTotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// 내부 Entity (Aggregate 외부에서 직접 생성/접근 불가)
class OrderLine {  // package-private

    private final OrderLineId id;
    private final ProductId productId;  // 다른 Aggregate 참조는 ID로
    private final String productName;   // 스냅샷 데이터
    private final Money unitPrice;
    private Quantity quantity;

    OrderLine(OrderLineId id, ProductId productId, String productName,
              Money unitPrice, Quantity quantity) {
        this.id = id;
        this.productId = productId;
        this.productName = productName;
        this.unitPrice = unitPrice;
        this.quantity = quantity;
    }

    void updateQuantity(Quantity newQuantity) {
        if (newQuantity.getValue() <= 0) {
            throw new InvalidQuantityException("수량은 1 이상이어야 합니다");
        }
        this.quantity = newQuantity;
    }

    Money getLineTotal() {
        return unitPrice.multiply(quantity.getValue());
    }

    // getter는 public (조회는 허용)
    public OrderLineId getId() {
        return id;
    }

    public ProductId getProductId() {
        return productId;
    }
}
```

### Repository (리포지토리)

Repository는 **Aggregate의 영속성을 추상화**하는 패턴이다.

```java
// Repository Interface (도메인 레이어에 위치)
public interface OrderRepository {

    // Aggregate Root 단위로 조회/저장
    Optional<Order> findById(OrderId id);

    Order save(Order order);

    void delete(Order order);

    // 복잡한 쿼리는 Specification 패턴 또는 전용 메서드
    List<Order> findByCustomerId(CustomerId customerId);

    List<Order> findByStatus(OrderStatus status);

    // 페이징
    Page<Order> findAll(Pageable pageable);
}

// Repository 구현체 (인프라 레이어에 위치)
@Repository
@RequiredArgsConstructor
public class JpaOrderRepository implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain);
    }

    @Override
    @Transactional
    public Order save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        OrderJpaEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    @Transactional
    public void delete(Order order) {
        jpaRepository.deleteById(order.getId().getValue());
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepository.findByCustomerId(customerId.getValue())
            .stream()
            .map(mapper::toDomain)
            .collect(toList());
    }
}
```

### Domain Service (도메인 서비스)

도메인 서비스는 **특정 Entity나 Value Object에 속하지 않는 도메인 로직**을 캡슐화한다.

```java
// Domain Service - 여러 Aggregate에 걸친 비즈니스 로직
@DomainService
public class OrderPricingService {

    private final DiscountPolicyRepository discountPolicyRepository;

    public Money calculateFinalPrice(Order order, Customer customer) {
        Money basePrice = order.getTotalAmount();

        // 여러 정책 적용
        DiscountPolicy memberDiscount = getMemberDiscount(customer);
        DiscountPolicy eventDiscount = getActiveEventDiscount();

        Money afterMemberDiscount = memberDiscount.apply(basePrice);
        Money afterEventDiscount = eventDiscount.apply(afterMemberDiscount);

        return afterEventDiscount;
    }

    private DiscountPolicy getMemberDiscount(Customer customer) {
        return discountPolicyRepository
            .findByMemberGrade(customer.getGrade())
            .orElse(DiscountPolicy.NONE);
    }
}

// Domain Service - 외부 시스템과 연동이 필요한 도메인 로직
@DomainService
public class FraudDetectionService {

    public boolean isSuspiciousOrder(Order order, Customer customer) {
        // 도메인 규칙 기반 사기 탐지
        return hasTooManyOrdersToday(customer, 10) ||
               isUnusuallyHighAmount(order, customer) ||
               isNewAccountWithLargeOrder(customer, order);
    }

    private boolean hasTooManyOrdersToday(Customer customer, int limit) {
        // ...
    }
}
```

---

## 유비쿼터스 언어

유비쿼터스 언어(Ubiquitous Language)는 **도메인 전문가와 개발자가 공유하는 공통 언어**이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      유비쿼터스 언어                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  "우리 도메인에서 사용하는 용어를 코드에 그대로 반영한다"        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ❌ 잘못된 예시 (기술 용어 사용):                                    │
│  ─────────────────────────────────────────────────────────────────  │
│  • OrderDAO, OrderDTO, OrderEntity                                  │
│  • updateOrderStatus(), setStatus()                                 │
│  • isValid(), processData()                                         │
│                                                                      │
│  ✓ 올바른 예시 (도메인 용어 사용):                                   │
│  ─────────────────────────────────────────────────────────────────  │
│  • Order (주문), Customer (고객), Product (상품)                    │
│  • confirmOrder() - 주문 확정                                       │
│  • cancelOrder() - 주문 취소                                        │
│  • shipOrder() - 배송 시작                                          │
│  • placeOrder() - 주문하다                                          │
│                                                                      │
│  유비쿼터스 언어 사전 예시 (E-Commerce):                              │
│  ─────────────────────────────────────────────────────────────────  │
│  용어            │ 정의                          │ 코드              │
│  ─────────────────────────────────────────────────────────────────  │
│  주문(Order)     │ 고객이 상품을 구매하는 행위    │ Order.class      │
│  장바구니(Cart)  │ 구매 전 임시 보관함            │ ShoppingCart     │
│  결제(Payment)   │ 주문에 대한 대금 지불          │ Payment.class    │
│  배송(Shipping)  │ 상품을 고객에게 전달           │ Shipment.class   │
│  환불(Refund)    │ 결제 취소 및 금액 반환         │ Refund.class     │
│  재고(Stock)     │ 보유 중인 상품 수량            │ Inventory        │
│  품절(OutOfStock)│ 재고가 0인 상태               │ OutOfStockEx     │
│                                                                      │
│  ★ 핵심: 도메인 전문가와 대화할 때 사용하는 용어 = 코드의 클래스/메서드명│
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// 유비쿼터스 언어를 반영한 코드

// ❌ 나쁜 예: 기술적 용어
public class OrderManager {
    public void updateStatus(Order order, int statusCode) { }
    public void processPayment(OrderData data) { }
    public boolean validateOrder(OrderDTO dto) { }
}

// ✓ 좋은 예: 도메인 용어
public class Order {
    // 도메인 전문가도 이해할 수 있는 메서드명
    public void confirm() { }          // "주문을 확정한다"
    public void cancel(String reason) { } // "주문을 취소한다"
    public void ship() { }             // "배송을 시작한다"
    public void complete() { }         // "주문을 완료한다"

    public void addItem(Product product, Quantity quantity) { }
    public void removeItem(OrderLineId lineId) { }

    public boolean canBeCancelled() { }
    public boolean isOverdue() { }
}

public class Customer {
    public void placeOrder(Order order) { }  // "주문하다"
    public void requestRefund(Order order, String reason) { }  // "환불 요청하다"
    public void upgradeToGold() { }  // "골드 등급으로 승급하다"
}

// 예외도 도메인 언어로
public class OrderAlreadyShippedException extends DomainException { }
public class InsufficientStockException extends DomainException { }
public class PaymentDeclinedException extends DomainException { }
```

---

## Domain Event

Domain Event는 **도메인에서 발생한 중요한 비즈니스 사건**을 나타낸다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Domain Event                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  특징:                                                               │
│  • 과거에 발생한 사실 (Past Tense)                                   │
│  • 불변 (Immutable)                                                  │
│  • 비즈니스적으로 의미 있는 사건                                      │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  "~가 발생했다" (Something happened)                         │    │
│  │                                                               │    │
│  │  • OrderPlaced (주문이 접수되었다)                            │    │
│  │  • OrderConfirmed (주문이 확정되었다)                          │    │
│  │  • PaymentReceived (결제가 완료되었다)                        │    │
│  │  • OrderShipped (배송이 시작되었다)                           │    │
│  │  • InventoryDepleted (재고가 소진되었다)                      │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  도메인 이벤트의 활용:                                                │
│  ─────────────────────────────────────────────────────────────────  │
│  1. 느슨한 결합: 서비스 간 직접 의존성 제거                           │
│  2. 감사 추적: 시스템에서 발생한 모든 변경 기록                        │
│  3. 이벤트 소싱: 이벤트로부터 현재 상태 재구성                         │
│  4. 통합: 다른 Bounded Context에 변경 알림                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// Domain Event 기본 인터페이스
public interface DomainEvent {
    String getEventId();
    String getAggregateId();
    String getEventType();
    Instant getOccurredAt();
}

// 추상 기본 클래스
@Getter
public abstract class BaseDomainEvent implements DomainEvent {

    private final String eventId;
    private final Instant occurredAt;

    protected BaseDomainEvent() {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = Instant.now();
    }

    @Override
    public String getEventType() {
        return this.getClass().getSimpleName();
    }
}

// 구체적인 Domain Event
@Getter
public class OrderConfirmedEvent extends BaseDomainEvent {

    private final String orderId;
    private final String customerId;
    private final BigDecimal totalAmount;
    private final List<OrderLineSnapshot> orderLines;

    public OrderConfirmedEvent(Order order) {
        super();
        this.orderId = order.getId().getValue().toString();
        this.customerId = order.getCustomerId().getValue().toString();
        this.totalAmount = order.getTotalAmount().getAmount();
        this.orderLines = order.getOrderLines().stream()
            .map(OrderLineSnapshot::from)
            .collect(toList());
    }

    @Override
    public String getAggregateId() {
        return orderId;
    }

    // 스냅샷 (이벤트 발생 시점의 데이터)
    @Value
    public static class OrderLineSnapshot {
        String productId;
        String productName;
        int quantity;
        BigDecimal unitPrice;

        public static OrderLineSnapshot from(OrderLine line) {
            return new OrderLineSnapshot(
                line.getProductId().getValue().toString(),
                line.getProductName(),
                line.getQuantity().getValue(),
                line.getUnitPrice().getAmount()
            );
        }
    }
}

// Aggregate에서 이벤트 발행
public abstract class AggregateRoot<ID> {

    @Getter(AccessLevel.PROTECTED)
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected void registerEvent(DomainEvent event) {
        this.domainEvents.add(event);
    }

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearDomainEvents() {
        this.domainEvents.clear();
    }
}

// Order에서 이벤트 등록
public class Order extends AggregateRoot<OrderId> {

    public void confirm() {
        validateDraftStatus();
        validateNotEmpty();

        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = LocalDateTime.now();

        // 도메인 이벤트 등록
        registerEvent(new OrderConfirmedEvent(this));
    }

    public void cancel(String reason) {
        validateCancellable();
        this.status = OrderStatus.CANCELLED;

        registerEvent(new OrderCancelledEvent(this, reason));
    }
}

// Application Service에서 이벤트 발행
@Service
@Transactional
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    public void confirmOrder(ConfirmOrderCommand command) {
        Order order = orderRepository.findById(new OrderId(command.getOrderId()))
            .orElseThrow(() -> new OrderNotFoundException(command.getOrderId()));

        order.confirm();

        orderRepository.save(order);

        // 이벤트 발행
        order.getDomainEvents().forEach(eventPublisher::publish);
        order.clearDomainEvents();
    }
}

// 이벤트 핸들러 (다른 Bounded Context 또는 같은 Context)
@Component
public class OrderEventHandler {

    private final InventoryService inventoryService;
    private final NotificationService notificationService;

    @EventHandler  // 또는 @EventListener, @KafkaListener 등
    public void handle(OrderConfirmedEvent event) {
        // 재고 차감
        inventoryService.decreaseStock(event.getOrderLines());

        // 알림 발송
        notificationService.sendOrderConfirmation(event.getCustomerId());
    }
}
```

---

## Anti-Corruption Layer

Anti-Corruption Layer(ACL)는 **외부 시스템의 모델이 내 도메인 모델을 오염시키지 않도록 보호**하는 계층이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Anti-Corruption Layer (ACL)                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────┐     ┌─────────────────┐     ┌────────────────┐ │
│  │   외부 시스템    │     │      ACL        │     │   내 도메인    │ │
│  │  (Legacy/3rd)  │────►│    (변환 계층)   │────►│    (Clean)    │ │
│  │                │     │                 │     │               │ │
│  │ • 다른 용어     │     │ • Facade       │     │ • 내 용어      │ │
│  │ • 다른 구조     │     │ • Adapter      │     │ • 내 구조      │ │
│  │ • 다른 규칙     │     │ • Translator   │     │ • 내 규칙      │ │
│  │                │     │                 │     │               │ │
│  └─────────────────┘     └─────────────────┘     └────────────────┘ │
│                                                                      │
│  적용 상황:                                                          │
│  • 레거시 시스템과 통합                                              │
│  • 외부 API 연동 (결제, 배송 등)                                     │
│  • 다른 팀의 서비스 호출                                             │
│  • 서드파티 라이브러리 래핑                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// ===== 외부 결제 시스템의 모델 (우리가 제어할 수 없음) =====
// 외부 API 응답 (snake_case, 다른 용어)
public class ExternalPaymentResponse {
    public String tx_id;
    public String payment_status;  // "SUCCESS", "FAIL", "PENDING"
    public long amount_in_cents;   // 센트 단위!
    public String failure_reason;
    public String created_datetime;  // "2024-01-15T10:30:00Z"
}

// ===== ACL: 외부 모델 → 내 도메인 모델 변환 =====

// 내 도메인 모델
@Value
public class PaymentResult {
    PaymentId paymentId;
    PaymentStatus status;
    Money amount;
    String failureReason;
    LocalDateTime processedAt;
}

public enum PaymentStatus {
    COMPLETED, FAILED, PENDING
}

// Anti-Corruption Layer (Adapter + Translator)
@Component
public class PaymentGatewayAdapter implements PaymentGateway {

    private final ExternalPaymentClient externalClient;  // Feign/RestTemplate
    private final PaymentTranslator translator;

    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        // 1. 내 모델 → 외부 모델 변환
        ExternalPaymentRequest externalRequest = translator.toExternal(request);

        // 2. 외부 시스템 호출
        ExternalPaymentResponse externalResponse = externalClient.pay(externalRequest);

        // 3. 외부 모델 → 내 도메인 모델 변환
        return translator.toDomain(externalResponse);
    }
}

// 변환 로직 (Translator)
@Component
public class PaymentTranslator {

    public PaymentResult toDomain(ExternalPaymentResponse external) {
        return new PaymentResult(
            new PaymentId(external.tx_id),
            translateStatus(external.payment_status),
            translateAmount(external.amount_in_cents),
            external.failure_reason,
            parseDateTime(external.created_datetime)
        );
    }

    private PaymentStatus translateStatus(String externalStatus) {
        return switch (externalStatus) {
            case "SUCCESS" -> PaymentStatus.COMPLETED;
            case "FAIL" -> PaymentStatus.FAILED;
            case "PENDING" -> PaymentStatus.PENDING;
            default -> throw new UnknownPaymentStatusException(externalStatus);
        };
    }

    private Money translateAmount(long amountInCents) {
        // 센트 → 원화 변환
        BigDecimal amountInWon = BigDecimal.valueOf(amountInCents)
            .divide(BigDecimal.valueOf(100), 0, RoundingMode.HALF_UP);
        return Money.of(amountInWon, Currency.KRW);
    }

    private LocalDateTime parseDateTime(String datetime) {
        return LocalDateTime.parse(datetime, DateTimeFormatter.ISO_DATE_TIME);
    }

    public ExternalPaymentRequest toExternal(PaymentRequest request) {
        ExternalPaymentRequest external = new ExternalPaymentRequest();
        external.merchant_id = "MY_MERCHANT_ID";
        external.order_ref = request.getOrderId().getValue().toString();
        external.amount_cents = request.getAmount().getAmount()
            .multiply(BigDecimal.valueOf(100)).longValue();
        external.currency_code = "KRW";
        external.customer_email = request.getCustomerEmail();
        return external;
    }
}

// 도메인 계층의 Port (ACL과 분리)
public interface PaymentGateway {
    PaymentResult processPayment(PaymentRequest request);
}
```

### 레거시 시스템 ACL 예시

```java
// 레거시 시스템의 복잡한 데이터 구조
public class LegacyOrderData {
    public int orderNo;         // 숫자 ID
    public int custCd;          // 고객 코드
    public String ordDt;        // "20240115" 형식
    public int ordStat;         // 1=접수, 2=확정, 3=배송, 4=완료, 9=취소
    public List<LegacyOrderItem> items;
    public int totAmt;          // 정수형 총액
}

// ACL로 레거시 모델 변환
@Component
public class LegacyOrderAdapter implements LegacyOrderGateway {

    private final LegacySystemClient legacyClient;

    @Override
    public Optional<Order> findById(OrderId orderId) {
        LegacyOrderData legacyData = legacyClient.getOrder(
            Integer.parseInt(orderId.getValue().toString())
        );

        if (legacyData == null) {
            return Optional.empty();
        }

        return Optional.of(translateToOrder(legacyData));
    }

    private Order translateToOrder(LegacyOrderData legacy) {
        return Order.reconstitute(
            new OrderId(Long.valueOf(legacy.orderNo)),
            new CustomerId(Long.valueOf(legacy.custCd)),
            translateStatus(legacy.ordStat),
            translateDate(legacy.ordDt),
            translateItems(legacy.items),
            Money.won(legacy.totAmt)
        );
    }

    private OrderStatus translateStatus(int legacyStatus) {
        return switch (legacyStatus) {
            case 1 -> OrderStatus.DRAFT;
            case 2 -> OrderStatus.CONFIRMED;
            case 3 -> OrderStatus.SHIPPED;
            case 4 -> OrderStatus.COMPLETED;
            case 9 -> OrderStatus.CANCELLED;
            default -> throw new UnknownLegacyStatusException(legacyStatus);
        };
    }

    private LocalDate translateDate(String dateStr) {
        // "20240115" → LocalDate
        return LocalDate.parse(dateStr, DateTimeFormatter.ofPattern("yyyyMMdd"));
    }
}
```

---

## Event Storming

Event Storming은 **도메인 이벤트를 중심으로 비즈니스 프로세스를 탐색**하는 워크숍 기법이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Event Storming 프로세스                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  참여자: 도메인 전문가 + 개발자 + 디자이너                             │
│  준비물: 넓은 벽, 포스트잇(여러 색상), 마커                            │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    포스트잇 색상 규칙                         │    │
│  │                                                               │    │
│  │  🟧 주황색: Domain Event (과거형, "~했다")                    │    │
│  │  🟦 파란색: Command (명령, "~해라")                           │    │
│  │  🟨 노란색: Aggregate (명사)                                  │    │
│  │  🟪 보라색: Policy/Process Manager ("~하면 ~한다")            │    │
│  │  🟩 녹색:  Read Model (조회용 데이터)                         │    │
│  │  🟥 빨간색: Hotspot/문제점                                    │    │
│  │  📝 하늘색: External System                                   │    │
│  │  👤 작은 노란색: Actor (사용자)                               │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  진행 단계:                                                          │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  1단계: Big Picture (30분-1시간)                                    │
│     - 모든 도메인 이벤트를 자유롭게 붙이기                            │
│     - 시간 순서대로 배치                                             │
│     - 중복/충돌 허용                                                 │
│                                                                      │
│  2단계: 타임라인 정리 (30분)                                         │
│     - 이벤트를 시간 순으로 정렬                                      │
│     - 중복 제거, 누락 이벤트 추가                                    │
│     - Hotspot(문제점/의문점) 표시                                    │
│                                                                      │
│  3단계: Command & Actor 추가 (30분)                                 │
│     - 각 이벤트를 발생시키는 Command 추가                            │
│     - Command를 실행하는 Actor 추가                                  │
│                                                                      │
│  4단계: Aggregate 식별 (30분)                                       │
│     - 관련 이벤트/커맨드를 그룹화                                    │
│     - 각 그룹에 Aggregate 이름 부여                                  │
│                                                                      │
│  5단계: Bounded Context 도출 (30분)                                 │
│     - 관련 Aggregate를 묶어 Context 경계 그리기                      │
│     - Context 간 관계 정의                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Event Storming 결과 예시

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Event Storming 결과 예시 (주문 도메인)               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Timeline ────────────────────────────────────────────────►         │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Shopping Cart Context                                       │    │
│  │                                                               │    │
│  │  👤 Customer                                                  │    │
│  │       │                                                       │    │
│  │       ▼                                                       │    │
│  │  🟦 Add to Cart    🟦 Remove Item    🟦 Update Qty           │    │
│  │       │                  │                │                   │    │
│  │       ▼                  ▼                ▼                   │    │
│  │  🟨 Shopping Cart ────────────────────────────                │    │
│  │       │                                                       │    │
│  │       ▼                                                       │    │
│  │  🟧 Item Added     🟧 Item Removed   🟧 Qty Updated          │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                         │                                           │
│                         ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Order Context                                               │    │
│  │                                                               │    │
│  │  👤 Customer                                                  │    │
│  │       │                                                       │    │
│  │       ▼                                                       │    │
│  │  🟦 Place Order ─────────────────────────────────────────    │    │
│  │       │                                                       │    │
│  │       ▼                                                       │    │
│  │  🟨 Order                                                     │    │
│  │       │                                                       │    │
│  │       ▼                                                       │    │
│  │  🟧 Order Placed ────► 🟪 Policy: When Order Placed          │    │
│  │                              │                                │    │
│  │                              ├──► 🟦 Reserve Stock           │    │
│  │                              └──► 🟦 Process Payment         │    │
│  │                                                               │    │
│  │  🟧 Order Confirmed ◄── 🟪 Policy: When Payment Completed    │    │
│  │       │                                                       │    │
│  │       ▼                                                       │    │
│  │  🟦 Ship Order                                                │    │
│  │       │                                                       │    │
│  │       ▼                                                       │    │
│  │  🟧 Order Shipped                                             │    │
│  │                                                               │    │
│  │  🟥 HOTSPOT: 재고 부족 시 어떻게?                             │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                         │                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Inventory Context        Payment Context                    │    │
│  │                                                               │    │
│  │  🟨 Inventory             🟨 Payment                         │    │
│  │       │                        │                              │    │
│  │  🟧 Stock Reserved        🟧 Payment Completed               │    │
│  │  🟧 Stock Depleted        🟧 Payment Failed                  │    │
│  │                                                               │    │
│  │  📝 Stripe (External)                                        │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 실무 구현 예제

### 전체 프로젝트 구조

```
src/main/java/com/example/order/
├── domain/                           # 도메인 레이어
│   ├── model/
│   │   ├── order/
│   │   │   ├── Order.java           # Aggregate Root
│   │   │   ├── OrderId.java         # Value Object (ID)
│   │   │   ├── OrderLine.java       # Entity
│   │   │   ├── OrderLineId.java
│   │   │   └── OrderStatus.java     # Enum
│   │   ├── customer/
│   │   │   ├── CustomerId.java
│   │   │   └── Customer.java
│   │   └── product/
│   │       ├── ProductId.java
│   │       └── ProductSnapshot.java  # Value Object
│   ├── vo/                           # Shared Value Objects
│   │   ├── Money.java
│   │   ├── Quantity.java
│   │   └── Address.java
│   ├── event/
│   │   ├── DomainEvent.java
│   │   ├── OrderPlacedEvent.java
│   │   ├── OrderConfirmedEvent.java
│   │   └── OrderCancelledEvent.java
│   ├── repository/
│   │   └── OrderRepository.java      # Repository Interface
│   ├── service/
│   │   └── OrderPricingService.java  # Domain Service
│   └── exception/
│       ├── DomainException.java
│       └── InvalidOrderStateException.java
│
├── application/                      # 애플리케이션 레이어
│   ├── port/
│   │   ├── in/
│   │   │   ├── PlaceOrderUseCase.java
│   │   │   └── CancelOrderUseCase.java
│   │   └── out/
│   │       ├── LoadOrderPort.java
│   │       ├── SaveOrderPort.java
│   │       └── PaymentGateway.java
│   ├── service/
│   │   ├── PlaceOrderService.java
│   │   └── CancelOrderService.java
│   └── dto/
│       ├── PlaceOrderCommand.java
│       └── OrderResult.java
│
├── infrastructure/                   # 인프라 레이어
│   ├── persistence/
│   │   ├── entity/
│   │   │   └── OrderJpaEntity.java
│   │   ├── repository/
│   │   │   └── OrderJpaRepository.java
│   │   ├── adapter/
│   │   │   └── OrderPersistenceAdapter.java
│   │   └── mapper/
│   │       └── OrderMapper.java
│   ├── external/
│   │   ├── payment/
│   │   │   ├── PaymentGatewayAdapter.java  # ACL
│   │   │   └── StripePaymentClient.java
│   │   └── shipping/
│   │       └── ShippingAdapter.java
│   └── messaging/
│       └── KafkaEventPublisher.java
│
└── interfaces/                       # 인터페이스 레이어
    ├── rest/
    │   ├── OrderController.java
    │   ├── request/
    │   │   └── PlaceOrderRequest.java
    │   └── response/
    │       └── OrderResponse.java
    └── event/
        └── OrderEventListener.java
```

### 통합 예제 코드

```java
// ===== Domain Layer =====

// Order Aggregate Root
public class Order extends AggregateRoot<OrderId> {

    private final OrderId id;
    private final CustomerId customerId;
    private List<OrderLine> orderLines;
    private ShippingAddress shippingAddress;
    private OrderStatus status;
    private Money totalAmount;
    private final LocalDateTime createdAt;
    private LocalDateTime confirmedAt;

    public static Order place(CustomerId customerId,
                             List<OrderLineRequest> lineRequests,
                             ShippingAddress shippingAddress) {

        Order order = new Order(OrderId.generate(), customerId, shippingAddress);

        for (OrderLineRequest request : lineRequests) {
            order.addOrderLine(request.getProduct(), request.getQuantity());
        }

        order.registerEvent(new OrderPlacedEvent(order));
        return order;
    }

    private void addOrderLine(ProductSnapshot product, Quantity quantity) {
        OrderLine line = OrderLine.create(product, quantity);
        this.orderLines.add(line);
        recalculateTotalAmount();
    }

    public void confirm() {
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new InvalidOrderStateException(
                "결제 대기 상태에서만 확정 가능합니다. 현재: " + this.status
            );
        }

        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = LocalDateTime.now();

        registerEvent(new OrderConfirmedEvent(this));
    }

    public void cancel(String reason) {
        if (this.status.isNotCancellable()) {
            throw new OrderCannotBeCancelledException(
                "취소 불가능한 상태입니다: " + this.status
            );
        }

        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(this, reason));
    }

    // ... getter, 검증 메서드 등
}

// ===== Application Layer =====

@UseCase
@RequiredArgsConstructor
@Transactional
public class PlaceOrderService implements PlaceOrderUseCase {

    private final LoadCustomerPort loadCustomerPort;
    private final LoadProductPort loadProductPort;
    private final SaveOrderPort saveOrderPort;
    private final PaymentGateway paymentGateway;
    private final DomainEventPublisher eventPublisher;

    @Override
    public OrderResult execute(PlaceOrderCommand command) {
        // 1. 고객 조회 및 검증
        Customer customer = loadCustomerPort.findById(command.getCustomerId())
            .orElseThrow(() -> new CustomerNotFoundException(command.getCustomerId()));

        // 2. 상품 조회 및 스냅샷 생성
        List<OrderLineRequest> lineRequests = command.getItems().stream()
            .map(item -> {
                Product product = loadProductPort.findById(item.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
                return new OrderLineRequest(
                    ProductSnapshot.from(product),
                    new Quantity(item.getQuantity())
                );
            })
            .collect(toList());

        // 3. 주문 생성 (도메인 로직)
        ShippingAddress address = command.getShippingAddress();
        Order order = Order.place(customer.getId(), lineRequests, address);

        // 4. 저장
        Order savedOrder = saveOrderPort.save(order);

        // 5. 이벤트 발행
        savedOrder.getDomainEvents().forEach(eventPublisher::publish);
        savedOrder.clearDomainEvents();

        return OrderResult.from(savedOrder);
    }
}

// ===== Infrastructure Layer (ACL) =====

@Component
@RequiredArgsConstructor
public class PaymentGatewayAdapter implements PaymentGateway {

    private final StripeClient stripeClient;
    private final PaymentTranslator translator;

    @Override
    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        // 내 모델 → Stripe 모델
        StripePaymentIntent intent = translator.toStripeIntent(request);

        // Stripe 호출
        StripePaymentResult stripeResult = stripeClient.createPayment(intent);

        // Stripe 모델 → 내 모델
        return translator.toPaymentResult(stripeResult);
    }

    private PaymentResult paymentFallback(PaymentRequest request, Exception ex) {
        log.error("Payment service unavailable", ex);
        return PaymentResult.failed("결제 서비스 일시 장애");
    }
}
```

---

## 참고 자료

- Eric Evans, "Domain-Driven Design: Tackling Complexity in the Heart of Software", 2003
- Vaughn Vernon, "Implementing Domain-Driven Design", 2013
- Vaughn Vernon, "Domain-Driven Design Distilled", 2016
- [Domain-Driven Design - Wikipedia](https://en.wikipedia.org/wiki/Domain-driven_design)
- [Event Storming - Lucidchart](https://www.lucidchart.com/blog/ddd-event-storming)
- [Domain-Driven Design - IBM](https://ibm-cloud-architecture.github.io/refarch-eda/methodology/domain-driven-design/)
- [Strategic Domain Driven Design - DEV](https://dev.to/indcoder/strategic-domain-driven-design-efn)
- Alberto Brandolini, "Introducing EventStorming", 2021

---

*마지막 업데이트: 2025년 1월*

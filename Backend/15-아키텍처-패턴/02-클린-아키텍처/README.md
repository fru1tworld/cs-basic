# 클린 아키텍처 (Clean Architecture)

## 목차
1. [개요](#개요)
2. [아키텍처 진화의 역사](#아키텍처-진화의-역사)
3. [의존성 규칙](#의존성-규칙)
4. [레이어 구조](#레이어-구조)
5. [포트와 어댑터 (Hexagonal Architecture)](#포트와-어댑터-hexagonal-architecture)
6. [의존성 역전 원칙](#의존성-역전-원칙)
7. [실무 구현 예제](#실무-구현-예제)
8. [테스트 전략](#테스트-전략)
9. [실무 적용 가이드](#실무-적용-가이드)
---

## 개요

클린 아키텍처는 Robert C. Martin(Uncle Bob)이 2012년에 제안한 소프트웨어 아키텍처로, **비즈니스 로직을 외부 기술적 세부사항으로부터 독립**시키는 것을 목표로 한다. 이를 통해 프레임워크, 데이터베이스, UI 등의 변경이 비즈니스 로직에 영향을 주지 않도록 한다.

### 핵심 목표

```
┌─────────────────────────────────────────────────────────────────────┐
│                    클린 아키텍처의 핵심 목표                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Framework Independent (프레임워크 독립성)                        │
│     → Spring, Express 등 프레임워크에 종속되지 않음                   │
│                                                                      │
│  2. Testable (테스트 용이성)                                        │
│     → UI, DB, 외부 서비스 없이 비즈니스 로직 테스트 가능               │
│                                                                      │
│  3. UI Independent (UI 독립성)                                      │
│     → Web UI를 Mobile UI로 교체해도 비즈니스 로직 변경 없음           │
│                                                                      │
│  4. Database Independent (데이터베이스 독립성)                       │
│     → Oracle에서 MongoDB로 변경해도 비즈니스 로직 변경 없음           │
│                                                                      │
│  5. External Agency Independent (외부 시스템 독립성)                 │
│     → 비즈니스 로직은 외부 세계에 대해 알지 못함                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 아키텍처 진화의 역사

### 관련 아키텍처 패턴 비교

```
┌─────────────────────────────────────────────────────────────────────┐
│                    아키텍처 패턴의 진화                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  2005  Hexagonal Architecture (Alistair Cockburn)                   │
│        └── Ports and Adapters                                       │
│                     │                                                │
│                     ▼                                                │
│  2008  Onion Architecture (Jeffrey Palermo)                         │
│        └── Domain Model at Center                                   │
│                     │                                                │
│                     ▼                                                │
│  2012  Clean Architecture (Robert C. Martin)                        │
│        └── 위 아키텍처들의 통합 + 명확한 규칙                         │
│                     │                                                │
│                     ▼                                                │
│  2024  Ports and Adapters 책 출간 (Cockburn & Garrido)              │
│        └── 실무 적용 가이드 상세화                                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                   공통 핵심 원칙                              │    │
│  │                                                               │    │
│  │  ✓ 의존성은 바깥에서 안쪽으로만 향함                          │    │
│  │  ✓ 비즈니스 로직이 중심에 위치                                │    │
│  │  ✓ 외부 세부사항(DB, UI)은 플러그인처럼 교체 가능              │    │
│  │  ✓ Dependency Inversion Principle (DIP) 적용                 │    │
│  │                                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 의존성 규칙

### The Dependency Rule

> "Source code dependencies must point only inward, toward higher-level policies."
> 소스 코드 의존성은 반드시 안쪽으로, 상위 레벨 정책을 향해야 한다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Clean Architecture 동심원                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│         ┌─────────────────────────────────────────────────┐         │
│         │     Frameworks & Drivers (최외곽)                │         │
│         │  ┌─────────────────────────────────────────┐    │         │
│         │  │    Interface Adapters                    │    │         │
│         │  │  ┌─────────────────────────────────┐    │    │         │
│         │  │  │      Application Business       │    │    │         │
│         │  │  │          (Use Cases)            │    │    │         │
│         │  │  │  ┌─────────────────────────┐    │    │    │         │
│         │  │  │  │   Enterprise Business   │    │    │    │         │
│         │  │  │  │      (Entities)         │    │    │    │         │
│         │  │  │  │                         │    │    │    │         │
│         │  │  │  │    ★ 핵심 도메인 ★      │    │    │    │         │
│         │  │  │  │                         │    │    │    │         │
│         │  │  │  └─────────────────────────┘    │    │    │         │
│         │  │  └─────────────────────────────────┘    │    │         │
│         │  └─────────────────────────────────────────┘    │         │
│         └─────────────────────────────────────────────────┘         │
│                                                                      │
│                  의존성 방향: 바깥 → 안쪽 (ONLY!)                     │
│                                                                      │
│  ╔═══════════════════════════════════════════════════════════════╗  │
│  ║  ✗ 안쪽 레이어는 바깥 레이어에 대해 아무것도 모름               ║  │
│  ║  ✗ Entity는 Use Case를 알지 못함                              ║  │
│  ║  ✗ Use Case는 Controller나 DB를 알지 못함                     ║  │
│  ╚═══════════════════════════════════════════════════════════════╝  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 데이터 흐름 vs 의존성 방향

```
┌─────────────────────────────────────────────────────────────────────┐
│                 데이터 흐름과 의존성 방향의 분리                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  데이터 흐름 (Runtime):                                              │
│  Controller → Use Case → Entity → Repository → Database             │
│  Controller ← Use Case ← Entity ← Repository ← Database             │
│                                                                      │
│  의존성 방향 (Compile-time):                                         │
│                                                                      │
│     Controller                  Entity                               │
│         │                          ▲                                 │
│         ▼                          │                                 │
│     Use Case  ───────────────────► │                                 │
│         │                          │                                 │
│         │                          │                                 │
│         ▼                          │                                 │
│   Repository ◄─────────────────────┘                                 │
│    Interface                                                         │
│         ▲                                                            │
│         │ (DIP: 의존성 역전)                                          │
│         │                                                            │
│   Repository                                                         │
│     Impl                                                             │
│         │                                                            │
│         ▼                                                            │
│     Database                                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 레이어 구조

### 1. Entities (Enterprise Business Rules)

엔티티는 **핵심 비즈니스 규칙**을 캡슐화한다. 가장 변경 가능성이 낮은 레이어이다.

```java
// Domain Entity - 순수한 비즈니스 로직
// 어떤 프레임워크에도 의존하지 않음
public class Order {

    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderLine> orderLines;
    private OrderStatus status;
    private Money totalAmount;

    private Order(OrderId id, CustomerId customerId) {
        this.id = id;
        this.customerId = customerId;
        this.orderLines = new ArrayList<>();
        this.status = OrderStatus.DRAFT;
        this.totalAmount = Money.ZERO;
    }

    // Factory Method
    public static Order create(CustomerId customerId) {
        return new Order(OrderId.generate(), customerId);
    }

    // 비즈니스 규칙: 주문 라인 추가
    public void addOrderLine(Product product, Quantity quantity) {
        validateDraftStatus();
        validateQuantity(quantity);

        OrderLine line = OrderLine.create(product, quantity);
        this.orderLines.add(line);
        recalculateTotalAmount();
    }

    // 비즈니스 규칙: 주문 확정
    public void confirm() {
        validateDraftStatus();
        validateHasOrderLines();

        this.status = OrderStatus.CONFIRMED;
        // Domain Event 발생 (선택적)
        registerEvent(new OrderConfirmedEvent(this.id));
    }

    // 비즈니스 규칙: 주문 취소
    public void cancel(String reason) {
        if (status == OrderStatus.SHIPPED) {
            throw new OrderAlreadyShippedException(
                "배송 시작된 주문은 취소할 수 없습니다."
            );
        }
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(this.id, reason));
    }

    // 불변식(Invariant) 검증
    private void validateDraftStatus() {
        if (status != OrderStatus.DRAFT) {
            throw new InvalidOrderStateException(
                "초안 상태에서만 수정 가능합니다."
            );
        }
    }

    private void validateHasOrderLines() {
        if (orderLines.isEmpty()) {
            throw new EmptyOrderException("주문 항목이 없습니다.");
        }
    }

    private void recalculateTotalAmount() {
        this.totalAmount = orderLines.stream()
            .map(OrderLine::getLineTotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Value Object - 불변, 동등성 비교
public final class Money {

    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.KRW);

    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidMoneyException("금액은 0 이상이어야 합니다.");
        }
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int multiplier) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(multiplier)), this.currency);
    }

    // equals, hashCode 반드시 구현 (Value Object)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 &&
               currency == money.currency;
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}
```

### 2. Use Cases (Application Business Rules)

Use Case는 **애플리케이션 고유의 비즈니스 규칙**을 포함한다. 엔티티로/로부터 데이터 흐름을 조율한다.

```java
// Use Case Input Port (Interface)
public interface CreateOrderUseCase {
    OrderOutput execute(CreateOrderCommand command);
}

// Command (Input DTO)
@Value  // Lombok - 불변 객체
public class CreateOrderCommand {
    Long customerId;
    List<OrderLineItem> items;

    @Value
    public static class OrderLineItem {
        Long productId;
        int quantity;
    }
}

// Output (응답 DTO)
@Value
public class OrderOutput {
    Long orderId;
    String status;
    BigDecimal totalAmount;
    LocalDateTime createdAt;

    public static OrderOutput from(Order order) {
        return new OrderOutput(
            order.getId().getValue(),
            order.getStatus().name(),
            order.getTotalAmount().getAmount(),
            order.getCreatedAt()
        );
    }
}

// Use Case 구현체 (Application Service / Interactor)
@UseCase  // 커스텀 어노테이션 (Spring @Service 대신)
@RequiredArgsConstructor
@Transactional
public class CreateOrderService implements CreateOrderUseCase {

    // 출력 포트 (인터페이스에 의존)
    private final LoadCustomerPort loadCustomerPort;
    private final LoadProductPort loadProductPort;
    private final SaveOrderPort saveOrderPort;
    private final OrderEventPublisher eventPublisher;

    @Override
    public OrderOutput execute(CreateOrderCommand command) {
        // 1. 고객 조회 및 검증
        Customer customer = loadCustomerPort.loadById(
            new CustomerId(command.getCustomerId())
        ).orElseThrow(() -> new CustomerNotFoundException(command.getCustomerId()));

        // 2. 주문 생성 (도메인 엔티티)
        Order order = Order.create(customer.getId());

        // 3. 주문 라인 추가
        for (CreateOrderCommand.OrderLineItem item : command.getItems()) {
            Product product = loadProductPort.loadById(
                new ProductId(item.getProductId())
            ).orElseThrow(() -> new ProductNotFoundException(item.getProductId()));

            order.addOrderLine(product, new Quantity(item.getQuantity()));
        }

        // 4. 주문 확정
        order.confirm();

        // 5. 저장
        Order savedOrder = saveOrderPort.save(order);

        // 6. 이벤트 발행
        savedOrder.getDomainEvents().forEach(eventPublisher::publish);

        return OrderOutput.from(savedOrder);
    }
}
```

### 3. Interface Adapters

외부 인터페이스(Web, DB 등)와 Use Case 사이를 변환하는 레이어이다.

```java
// ===== Input Adapter (Driving Adapter) =====

// Web Controller - Input Adapter
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;
    private final GetOrderUseCase getOrderUseCase;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {

        // Request DTO → Command 변환
        CreateOrderCommand command = new CreateOrderCommand(
            request.getCustomerId(),
            request.getItems().stream()
                .map(item -> new CreateOrderCommand.OrderLineItem(
                    item.getProductId(),
                    item.getQuantity()
                ))
                .collect(toList())
        );

        // Use Case 실행
        OrderOutput output = createOrderUseCase.execute(command);

        // Output → Response DTO 변환
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(OrderResponse.from(output));
    }
}

// Request DTO (Web Layer)
@Data
public class CreateOrderRequest {
    @NotNull
    private Long customerId;

    @NotEmpty
    @Valid
    private List<OrderLineRequest> items;

    @Data
    public static class OrderLineRequest {
        @NotNull
        private Long productId;

        @Min(1)
        private int quantity;
    }
}

// ===== Output Adapter (Driven Adapter) =====

// Repository Implementation - Output Adapter
@Repository
@RequiredArgsConstructor
public class OrderJpaAdapter implements SaveOrderPort, LoadOrderPort {

    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    @Override
    public Order save(Order order) {
        OrderJpaEntity entity = mapper.toJpaEntity(order);
        OrderJpaEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<Order> loadById(OrderId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain);
    }
}

// JPA Entity (Infrastructure Layer)
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderJpaEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long customerId;

    @Enumerated(EnumType.STRING)
    private String status;

    @Column(precision = 19, scale = 2)
    private BigDecimal totalAmount;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderLineJpaEntity> orderLines = new ArrayList<>();

    private LocalDateTime createdAt;
}

// Mapper (Domain <-> JPA Entity 변환)
@Component
public class OrderMapper {

    public Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(
            new OrderId(entity.getId()),
            new CustomerId(entity.getCustomerId()),
            OrderStatus.valueOf(entity.getStatus()),
            entity.getOrderLines().stream()
                .map(this::toOrderLine)
                .collect(toList()),
            new Money(entity.getTotalAmount(), Currency.KRW),
            entity.getCreatedAt()
        );
    }

    public OrderJpaEntity toJpaEntity(Order order) {
        // ... 변환 로직
    }
}
```

### 4. Frameworks & Drivers

가장 바깥 레이어로, 프레임워크와 도구들이 위치한다.

```java
// Spring Configuration
@Configuration
@EnableJpaRepositories(basePackages = "com.example.adapter.out.persistence")
@EntityScan(basePackages = "com.example.adapter.out.persistence")
public class PersistenceConfig {

    @Bean
    public DataSource dataSource() {
        // 데이터소스 설정
    }
}

// Spring Security Configuration
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    // 보안 설정
}
```

### 전체 패키지 구조

```
src/main/java/com/example/
├── domain/                          # Enterprise Business Rules
│   ├── model/
│   │   ├── Order.java
│   │   ├── OrderLine.java
│   │   ├── OrderId.java
│   │   └── OrderStatus.java
│   ├── vo/                          # Value Objects
│   │   ├── Money.java
│   │   ├── Quantity.java
│   │   └── Address.java
│   ├── event/                       # Domain Events
│   │   ├── OrderConfirmedEvent.java
│   │   └── OrderCancelledEvent.java
│   └── exception/
│       ├── InvalidOrderStateException.java
│       └── EmptyOrderException.java
│
├── application/                     # Application Business Rules
│   ├── port/
│   │   ├── in/                      # Input Ports (Use Cases)
│   │   │   ├── CreateOrderUseCase.java
│   │   │   ├── GetOrderUseCase.java
│   │   │   └── CancelOrderUseCase.java
│   │   └── out/                     # Output Ports
│   │       ├── LoadOrderPort.java
│   │       ├── SaveOrderPort.java
│   │       ├── LoadCustomerPort.java
│   │       └── OrderEventPublisher.java
│   ├── service/                     # Use Case Implementations
│   │   ├── CreateOrderService.java
│   │   ├── GetOrderService.java
│   │   └── CancelOrderService.java
│   └── dto/                         # Command & Output DTOs
│       ├── CreateOrderCommand.java
│       ├── CancelOrderCommand.java
│       └── OrderOutput.java
│
├── adapter/                         # Interface Adapters
│   ├── in/                          # Input (Driving) Adapters
│   │   ├── web/
│   │   │   ├── OrderController.java
│   │   │   ├── dto/
│   │   │   │   ├── CreateOrderRequest.java
│   │   │   │   └── OrderResponse.java
│   │   │   └── exception/
│   │   │       └── GlobalExceptionHandler.java
│   │   └── message/
│   │       └── OrderEventListener.java
│   └── out/                         # Output (Driven) Adapters
│       ├── persistence/
│       │   ├── OrderJpaAdapter.java
│       │   ├── OrderJpaRepository.java
│       │   ├── entity/
│       │   │   └── OrderJpaEntity.java
│       │   └── mapper/
│       │       └── OrderMapper.java
│       └── messaging/
│           └── KafkaOrderEventPublisher.java
│
└── config/                          # Frameworks & Drivers
    ├── WebConfig.java
    ├── PersistenceConfig.java
    └── KafkaConfig.java
```

---

## 포트와 어댑터 (Hexagonal Architecture)

### 헥사고날 아키텍처 개념

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Hexagonal Architecture                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│     Primary/Driving Adapters            Secondary/Driven Adapters    │
│     (외부 → 애플리케이션)                (애플리케이션 → 외부)          │
│                                                                      │
│  ┌────────────────┐                         ┌────────────────┐      │
│  │  REST API      │                         │  PostgreSQL    │      │
│  │  Controller    │──────┐         ┌───────│  Repository    │      │
│  └────────────────┘      │         │        └────────────────┘      │
│                          │         │                                 │
│  ┌────────────────┐      │         │        ┌────────────────┐      │
│  │  GraphQL       │      ▼         ▼        │  Redis         │      │
│  │  Resolver      │─────►╔═══════════╗◄─────│  Cache         │      │
│  └────────────────┘      ║           ║      └────────────────┘      │
│                          ║   Core    ║                               │
│  ┌────────────────┐      ║  Domain   ║      ┌────────────────┐      │
│  │  CLI           │─────►║    +      ║◄─────│  Kafka         │      │
│  │  Command       │      ║ Use Cases ║      │  Publisher     │      │
│  └────────────────┘      ║           ║      └────────────────┘      │
│                          ╚═══════════╝                               │
│  ┌────────────────┐      ▲         ▲        ┌────────────────┐      │
│  │  Kafka         │      │         │        │  External      │      │
│  │  Consumer      │──────┘         └───────│  API Client    │      │
│  └────────────────┘                         └────────────────┘      │
│                                                                      │
│              Ports = 인터페이스 (계약)                                │
│              Adapters = 포트의 구현체                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 포트 정의

```java
// ===== Input Ports (Primary Ports) =====
// 애플리케이션이 제공하는 기능을 정의
// Use Case 인터페이스 = Input Port

public interface CreateOrderUseCase {
    OrderOutput execute(CreateOrderCommand command);
}

public interface CancelOrderUseCase {
    void execute(CancelOrderCommand command);
}

public interface GetOrderQuery {
    Optional<OrderOutput> findById(Long orderId);
    List<OrderOutput> findByCustomerId(Long customerId);
}

// ===== Output Ports (Secondary Ports) =====
// 애플리케이션이 외부에 요청하는 기능을 정의

// 영속성 포트
public interface LoadOrderPort {
    Optional<Order> loadById(OrderId id);
    List<Order> loadByCustomerId(CustomerId customerId);
}

public interface SaveOrderPort {
    Order save(Order order);
}

// 외부 서비스 포트
public interface LoadCustomerPort {
    Optional<Customer> loadById(CustomerId id);
}

public interface PaymentPort {
    PaymentResult processPayment(PaymentRequest request);
}

// 이벤트 발행 포트
public interface OrderEventPublisher {
    void publish(DomainEvent event);
}
```

### 어댑터 구현

```java
// ===== Primary Adapter (Driving Adapter) =====

// Web Adapter
@RestAdapter  // 커스텀 어노테이션
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderWebAdapter {

    private final CreateOrderUseCase createOrderUseCase;
    private final CancelOrderUseCase cancelOrderUseCase;
    private final GetOrderQuery getOrderQuery;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = request.toCommand();
        OrderOutput output = createOrderUseCase.execute(command);
        return ResponseEntity.created(URI.create("/api/v1/orders/" + output.getOrderId()))
                             .body(OrderResponse.from(output));
    }

    @DeleteMapping("/{orderId}")
    public ResponseEntity<Void> cancelOrder(
            @PathVariable Long orderId,
            @RequestBody CancelOrderRequest request) {
        cancelOrderUseCase.execute(new CancelOrderCommand(orderId, request.getReason()));
        return ResponseEntity.noContent().build();
    }
}

// Message Consumer Adapter
@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentEventAdapter {

    private final CompleteOrderUseCase completeOrderUseCase;

    @KafkaListener(topics = "payment-completed")
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        log.info("Payment completed for order: {}", event.getOrderId());
        completeOrderUseCase.execute(
            new CompleteOrderCommand(event.getOrderId(), event.getPaymentId())
        );
    }
}

// ===== Secondary Adapter (Driven Adapter) =====

// Persistence Adapter
@PersistenceAdapter  // 커스텀 어노테이션
@RequiredArgsConstructor
public class OrderPersistenceAdapter implements LoadOrderPort, SaveOrderPort {

    private final OrderJpaRepository orderRepository;
    private final OrderMapper orderMapper;

    @Override
    public Optional<Order> loadById(OrderId id) {
        return orderRepository.findById(id.getValue())
                .map(orderMapper::toDomain);
    }

    @Override
    @Transactional
    public Order save(Order order) {
        OrderJpaEntity entity = orderMapper.toEntity(order);
        OrderJpaEntity savedEntity = orderRepository.save(entity);
        return orderMapper.toDomain(savedEntity);
    }
}

// External API Adapter
@Component
@RequiredArgsConstructor
public class CustomerApiAdapter implements LoadCustomerPort {

    private final CustomerFeignClient customerClient;

    @Override
    @CircuitBreaker(name = "customer-service", fallbackMethod = "getCustomerFallback")
    public Optional<Customer> loadById(CustomerId id) {
        CustomerResponse response = customerClient.getCustomer(id.getValue());
        return Optional.ofNullable(response)
                .map(this::toDomain);
    }

    private Optional<Customer> getCustomerFallback(CustomerId id, Exception e) {
        log.warn("Customer service unavailable, using fallback for id: {}", id);
        return Optional.empty();
    }
}

// Event Publisher Adapter
@Component
@RequiredArgsConstructor
public class KafkaOrderEventPublisher implements OrderEventPublisher {

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Override
    public void publish(DomainEvent event) {
        String topic = resolveTopic(event);
        String key = event.getAggregateId().toString();

        kafkaTemplate.send(topic, key, toKafkaMessage(event))
            .addCallback(
                result -> log.info("Event published: {}", event.getEventType()),
                ex -> log.error("Failed to publish event: {}", event.getEventType(), ex)
            );
    }
}
```

---

## 의존성 역전 원칙

### Dependency Inversion Principle (DIP)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    의존성 역전 원칙 (DIP)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  전통적인 레이어드 아키텍처 (DIP 미적용):                             │
│                                                                      │
│  ┌─────────────────┐                                                │
│  │  Presentation   │                                                │
│  └────────┬────────┘                                                │
│           │ depends on                                              │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │    Business     │                                                │
│  └────────┬────────┘                                                │
│           │ depends on                                              │
│           ▼                                                          │
│  ┌─────────────────┐                                                │
│  │  Data Access    │◄──── 비즈니스가 인프라에 의존! (문제)           │
│  └─────────────────┘                                                │
│                                                                      │
│                                                                      │
│  클린 아키텍처 (DIP 적용):                                           │
│                                                                      │
│  ┌─────────────────┐                                                │
│  │   Controller    │                                                │
│  │ (Web Adapter)   │                                                │
│  └────────┬────────┘                                                │
│           │ depends on (interface)                                  │
│           ▼                                                          │
│  ╔═════════════════╗                                                │
│  ║    Use Case     ║◄──── Interface 정의                            │
│  ║   (Interface)   ║                                                │
│  ╚════════╤════════╝                                                │
│           │ depends on (interface)                                  │
│           ▼                                                          │
│  ╔═════════════════╗                                                │
│  ║  Repository     ║◄──── Port (Interface)                          │
│  ║   (Interface)   ║                                                │
│  ╚════════╤════════╝                                                │
│           ▲ implements                                               │
│           │                                                          │
│  ┌────────┴────────┐                                                │
│  │  JPA Adapter    │◄──── 구현체가 인터페이스에 의존!                 │
│  │ (Implementation)│                                                │
│  └─────────────────┘                                                │
│                                                                      │
│  ★ 핵심: 인터페이스를 상위 레벨(도메인)에서 정의                       │
│         구현체가 인터페이스에 의존 (역전!)                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### DIP 구현 코드

```java
// ===== 1. 도메인 레이어에서 인터페이스 정의 =====
// application/port/out/OrderRepository.java
public interface OrderRepository {  // Port (Interface)
    Optional<Order> findById(OrderId id);
    Order save(Order order);
}

// ===== 2. Use Case가 인터페이스에 의존 =====
// application/service/CreateOrderService.java
@Service
@RequiredArgsConstructor
public class CreateOrderService implements CreateOrderUseCase {

    // 인터페이스에 의존 (구체 구현체 모름)
    private final OrderRepository orderRepository;  // Port
    private final CustomerRepository customerRepository;

    @Override
    public OrderOutput execute(CreateOrderCommand command) {
        // 비즈니스 로직
        Order order = Order.create(/*...*/);
        Order saved = orderRepository.save(order);  // 어떤 DB인지 모름
        return OrderOutput.from(saved);
    }
}

// ===== 3. 구현체가 인터페이스 구현 (의존성 역전!) =====
// adapter/out/persistence/OrderJpaAdapter.java
@Repository
@RequiredArgsConstructor
public class OrderJpaAdapter implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.getValue())
                .map(mapper::toDomain);
    }

    @Override
    public Order save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        return mapper.toDomain(jpaRepository.save(entity));
    }
}

// ===== 4. 의존성 주입 (Spring) =====
// config/BeanConfig.java
@Configuration
public class BeanConfig {

    @Bean
    public OrderRepository orderRepository(
            OrderJpaRepository jpaRepository,
            OrderMapper mapper) {
        return new OrderJpaAdapter(jpaRepository, mapper);
    }
}

// 또는 @Repository 어노테이션으로 자동 주입
```

### 인터페이스 위치의 중요성

```
┌─────────────────────────────────────────────────────────────────────┐
│                 인터페이스 정의 위치가 중요한 이유                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ✗ 잘못된 방식: 인터페이스가 인프라 레이어에 있음                     │
│                                                                      │
│     Domain Layer                    Infrastructure Layer             │
│  ┌─────────────────┐              ┌─────────────────────────┐       │
│  │  OrderService   │──depends───►│  OrderRepository        │       │
│  │                 │      on     │  (Interface)            │       │
│  └─────────────────┘              │  OrderJpaImpl          │       │
│                                   │  (Implementation)       │       │
│                                   └─────────────────────────┘       │
│                                                                      │
│     → 도메인이 인프라에 의존! (DIP 위반)                              │
│                                                                      │
│                                                                      │
│  ✓ 올바른 방식: 인터페이스가 도메인/애플리케이션 레이어에 있음         │
│                                                                      │
│     Domain/Application Layer      Infrastructure Layer              │
│  ┌─────────────────────────┐    ┌─────────────────────────┐        │
│  │  OrderService           │    │                          │        │
│  │  OrderRepository (Port) │◄───│  OrderJpaAdapter        │        │
│  │  (Interface)            │impl│  (Implementation)        │        │
│  └─────────────────────────┘ents└─────────────────────────┘        │
│                                                                      │
│     → 인프라가 도메인에 의존! (DIP 적용)                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 실무 구현 예제

### 전체 흐름 예제: 주문 생성

```
┌─────────────────────────────────────────────────────────────────────┐
│                     주문 생성 요청 흐름                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. HTTP Request                                                    │
│     │                                                               │
│     ▼                                                               │
│  2. OrderController (Input Adapter)                                 │
│     │  - Request DTO 검증                                           │
│     │  - Command 객체로 변환                                         │
│     ▼                                                               │
│  3. CreateOrderUseCase (Input Port)                                 │
│     │                                                               │
│     ▼                                                               │
│  4. CreateOrderService (Use Case 구현)                              │
│     │  - 비즈니스 규칙 검증                                          │
│     │  - 도메인 엔티티 생성/조작                                      │
│     ▼                                                               │
│  5. Order (Domain Entity)                                           │
│     │  - 불변식(Invariant) 검증                                      │
│     │  - 도메인 이벤트 생성                                          │
│     ▼                                                               │
│  6. SaveOrderPort (Output Port)                                     │
│     │                                                               │
│     ▼                                                               │
│  7. OrderJpaAdapter (Output Adapter)                                │
│     │  - 도메인 → JPA Entity 변환                                    │
│     │  - 데이터베이스 저장                                           │
│     ▼                                                               │
│  8. OrderEventPublisher (Output Port)                               │
│     │                                                               │
│     ▼                                                               │
│  9. KafkaEventPublisher (Output Adapter)                            │
│     │  - 이벤트 발행                                                 │
│     ▼                                                               │
│  10. Response (역방향)                                               │
│      Output DTO → Response DTO → HTTP Response                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 완전한 예제 코드

```java
// ===== Domain Layer =====

// Order.java
public class Order {
    private OrderId id;
    private CustomerId customerId;
    private List<OrderLine> orderLines;
    private OrderStatus status;
    private Money totalAmount;
    private List<DomainEvent> domainEvents = new ArrayList<>();

    public static Order create(CustomerId customerId) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.customerId = customerId;
        order.orderLines = new ArrayList<>();
        order.status = OrderStatus.DRAFT;
        order.totalAmount = Money.ZERO;
        return order;
    }

    public void addLine(Product product, Quantity quantity) {
        validateDraftStatus();
        OrderLine line = OrderLine.create(this.id, product, quantity);
        this.orderLines.add(line);
        recalculateTotal();
    }

    public void confirm() {
        validateDraftStatus();
        if (orderLines.isEmpty()) {
            throw new EmptyOrderException("주문 항목이 없습니다");
        }
        this.status = OrderStatus.CONFIRMED;
        registerEvent(new OrderConfirmedEvent(this.id, this.totalAmount));
    }

    protected void registerEvent(DomainEvent event) {
        this.domainEvents.add(event);
    }

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearEvents() {
        this.domainEvents.clear();
    }

    private void validateDraftStatus() {
        if (this.status != OrderStatus.DRAFT) {
            throw new InvalidOrderStateException("초안 상태에서만 수정 가능합니다");
        }
    }

    private void recalculateTotal() {
        this.totalAmount = orderLines.stream()
            .map(OrderLine::getLineTotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// ===== Application Layer =====

// CreateOrderCommand.java
@Value
public class CreateOrderCommand {
    Long customerId;
    List<LineItem> items;

    @Value
    public static class LineItem {
        Long productId;
        int quantity;
    }
}

// CreateOrderUseCase.java
public interface CreateOrderUseCase {
    OrderOutput execute(CreateOrderCommand command);
}

// CreateOrderService.java
@UseCase
@RequiredArgsConstructor
@Transactional
public class CreateOrderService implements CreateOrderUseCase {

    private final LoadCustomerPort loadCustomerPort;
    private final LoadProductPort loadProductPort;
    private final SaveOrderPort saveOrderPort;
    private final DomainEventPublisher eventPublisher;

    @Override
    public OrderOutput execute(CreateOrderCommand command) {
        // 1. 고객 조회
        Customer customer = loadCustomerPort.findById(new CustomerId(command.getCustomerId()))
            .orElseThrow(() -> new CustomerNotFoundException(command.getCustomerId()));

        // 2. 주문 생성
        Order order = Order.create(customer.getId());

        // 3. 주문 라인 추가
        for (var item : command.getItems()) {
            Product product = loadProductPort.findById(new ProductId(item.getProductId()))
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
            order.addLine(product, new Quantity(item.getQuantity()));
        }

        // 4. 주문 확정
        order.confirm();

        // 5. 저장
        Order saved = saveOrderPort.save(order);

        // 6. 도메인 이벤트 발행
        saved.getDomainEvents().forEach(eventPublisher::publish);
        saved.clearEvents();

        return OrderOutput.from(saved);
    }
}

// ===== Adapter Layer =====

// OrderController.java (Input Adapter)
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {

        CreateOrderCommand command = CreateOrderCommand.builder()
            .customerId(request.getCustomerId())
            .items(request.getItems().stream()
                .map(i -> new CreateOrderCommand.LineItem(i.getProductId(), i.getQuantity()))
                .collect(toList()))
            .build();

        OrderOutput output = createOrderUseCase.execute(command);

        return ResponseEntity.created(URI.create("/api/orders/" + output.getOrderId()))
            .body(OrderResponse.from(output));
    }
}

// OrderPersistenceAdapter.java (Output Adapter)
@PersistenceAdapter
@RequiredArgsConstructor
public class OrderPersistenceAdapter implements LoadOrderPort, SaveOrderPort {

    private final OrderJpaRepository repository;
    private final OrderMapper mapper;

    @Override
    public Optional<Order> findById(OrderId id) {
        return repository.findById(id.getValue())
            .map(mapper::toDomain);
    }

    @Override
    public Order save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        OrderJpaEntity saved = repository.save(entity);
        return mapper.toDomain(saved);
    }
}
```

---

## 테스트 전략

### 테스트 피라미드

```
┌─────────────────────────────────────────────────────────────────────┐
│                        테스트 피라미드                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                          /\                                          │
│                         /  \                                         │
│                        / E2E\        ◄── 적음, 느림, 비용 높음        │
│                       /──────\                                       │
│                      /        \                                      │
│                     /Integration\    ◄── 중간                        │
│                    /─────────────\                                   │
│                   /               \                                  │
│                  /      Unit       \  ◄── 많음, 빠름, 비용 낮음       │
│                 /───────────────────\                                │
│                                                                      │
│  클린 아키텍처의 테스트 이점:                                         │
│  - 도메인/Use Case를 외부 의존성 없이 단위 테스트 가능                 │
│  - Adapter를 교체하여 다양한 환경에서 테스트 가능                      │
│  - 모킹이 최소화됨 (Port를 통한 명확한 경계)                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 레이어별 테스트

```java
// ===== 1. Domain Layer 단위 테스트 =====
class OrderTest {

    @Test
    void 주문에_상품_추가시_총액이_계산된다() {
        // Given
        Order order = Order.create(new CustomerId(1L));
        Product product = Product.of(new ProductId(1L), "테스트 상품",
            new Money(BigDecimal.valueOf(10000), Currency.KRW));

        // When
        order.addLine(product, new Quantity(2));

        // Then
        assertThat(order.getTotalAmount())
            .isEqualTo(new Money(BigDecimal.valueOf(20000), Currency.KRW));
    }

    @Test
    void 확정된_주문은_수정할_수_없다() {
        // Given
        Order order = Order.create(new CustomerId(1L));
        order.addLine(createProduct(), new Quantity(1));
        order.confirm();

        // When & Then
        assertThatThrownBy(() -> order.addLine(createProduct(), new Quantity(1)))
            .isInstanceOf(InvalidOrderStateException.class);
    }

    @Test
    void 빈_주문은_확정할_수_없다() {
        // Given
        Order order = Order.create(new CustomerId(1L));

        // When & Then
        assertThatThrownBy(() -> order.confirm())
            .isInstanceOf(EmptyOrderException.class);
    }
}

// ===== 2. Use Case 단위 테스트 =====
@ExtendWith(MockitoExtension.class)
class CreateOrderServiceTest {

    @Mock
    private LoadCustomerPort loadCustomerPort;
    @Mock
    private LoadProductPort loadProductPort;
    @Mock
    private SaveOrderPort saveOrderPort;
    @Mock
    private DomainEventPublisher eventPublisher;

    @InjectMocks
    private CreateOrderService createOrderService;

    @Test
    void 주문을_생성한다() {
        // Given
        CreateOrderCommand command = new CreateOrderCommand(
            1L,
            List.of(new CreateOrderCommand.LineItem(1L, 2))
        );

        Customer customer = new Customer(new CustomerId(1L), "고객");
        Product product = Product.of(new ProductId(1L), "상품",
            new Money(BigDecimal.valueOf(10000), Currency.KRW));

        given(loadCustomerPort.findById(any())).willReturn(Optional.of(customer));
        given(loadProductPort.findById(any())).willReturn(Optional.of(product));
        given(saveOrderPort.save(any())).willAnswer(inv -> inv.getArgument(0));

        // When
        OrderOutput result = createOrderService.execute(command);

        // Then
        assertThat(result.getStatus()).isEqualTo("CONFIRMED");
        assertThat(result.getTotalAmount()).isEqualByComparingTo("20000");
        verify(eventPublisher, times(1)).publish(any(OrderConfirmedEvent.class));
    }

    @Test
    void 존재하지_않는_고객이면_예외를_던진다() {
        // Given
        CreateOrderCommand command = new CreateOrderCommand(999L, List.of());
        given(loadCustomerPort.findById(any())).willReturn(Optional.empty());

        // When & Then
        assertThatThrownBy(() -> createOrderService.execute(command))
            .isInstanceOf(CustomerNotFoundException.class);
    }
}

// ===== 3. Adapter 통합 테스트 =====
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@Testcontainers
class OrderPersistenceAdapterTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Autowired
    private OrderJpaRepository orderJpaRepository;

    private OrderPersistenceAdapter adapter;

    @BeforeEach
    void setUp() {
        OrderMapper mapper = new OrderMapper();
        adapter = new OrderPersistenceAdapter(orderJpaRepository, mapper);
    }

    @Test
    void 주문을_저장하고_조회한다() {
        // Given
        Order order = Order.create(new CustomerId(1L));
        order.addLine(createProduct(), new Quantity(2));
        order.confirm();

        // When
        Order saved = adapter.save(order);
        Optional<Order> found = adapter.findById(saved.getId());

        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }
}

// ===== 4. Controller 테스트 =====
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private CreateOrderUseCase createOrderUseCase;

    @Test
    void 주문_생성_요청을_처리한다() throws Exception {
        // Given
        OrderOutput output = new OrderOutput(1L, "CONFIRMED",
            BigDecimal.valueOf(20000), LocalDateTime.now());
        given(createOrderUseCase.execute(any())).willReturn(output);

        // When & Then
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerId": 1,
                        "items": [
                            {"productId": 1, "quantity": 2}
                        ]
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.orderId").value(1))
            .andExpect(jsonPath("$.status").value("CONFIRMED"));
    }
}
```

---

## 실무 적용 가이드

### 점진적 도입 전략

```
┌─────────────────────────────────────────────────────────────────────┐
│                    점진적 클린 아키텍처 도입                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Phase 1: 패키지 구조 정리 (1-2주)                                   │
│  ├── 기존 코드를 domain, application, adapter로 분류                │
│  ├── 순환 의존성 제거                                               │
│  └── 컴파일 의존성 방향 확인                                         │
│                                                                      │
│  Phase 2: Port 도입 (2-3주)                                         │
│  ├── Repository 인터페이스를 application 레이어로 이동               │
│  ├── Output Port 정의                                               │
│  └── 기존 구현체가 Port 구현하도록 수정                              │
│                                                                      │
│  Phase 3: Use Case 분리 (2-4주)                                     │
│  ├── 거대한 Service 클래스를 Use Case 단위로 분리                    │
│  ├── Command/Query 객체 도입                                        │
│  └── Input Port 정의                                                │
│                                                                      │
│  Phase 4: Domain 강화 (지속)                                        │
│  ├── Anemic Domain을 Rich Domain으로 전환                           │
│  ├── Value Object 도입                                              │
│  └── Domain Event 도입                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 주의사항

```
┌─────────────────────────────────────────────────────────────────────┐
│                        주의사항 및 트레이드오프                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ✗ Over-engineering 주의                                            │
│    - 단순한 CRUD에는 과도한 추상화                                   │
│    - 팀의 역량과 프로젝트 복잡도 고려                                 │
│                                                                      │
│  ✗ 무분별한 Mapper 생성 주의                                        │
│    - Entity → Domain → DTO 변환 과정에서 성능 이슈 가능             │
│    - 필요한 경우에만 변환 레이어 도입                                 │
│                                                                      │
│  ✓ 적합한 경우                                                       │
│    - 복잡한 비즈니스 로직                                            │
│    - 장기 유지보수가 필요한 프로젝트                                  │
│    - 여러 팀이 협업하는 대규모 프로젝트                               │
│    - 외부 시스템 변경이 잦은 환경                                     │
│                                                                      │
│  ✓ 적용 시 이점                                                      │
│    - 테스트 용이성                                                   │
│    - 기술 변경 용이성                                                │
│    - 명확한 책임 분리                                                │
│    - 도메인 로직 보호                                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- Robert C. Martin, "Clean Architecture: A Craftsman's Guide to Software Structure and Design", 2017
- [Hexagonal Architecture - Alistair Cockburn](https://alistair.cockburn.us/hexagonal-architecture/)
- [Hexagonal Architecture and Clean Architecture with examples - DEV](https://dev.to/dyarleniber/hexagonal-architecture-and-clean-architecture-with-examples-48oi)
- [Clean Architecture vs Onion vs Hexagonal - CCD Akademie](https://ccd-akademie.de/en/clean-architecture-vs-onion-architecture-vs-hexagonal-architecture/)
- [Ports and Adapters in Python - Software Patterns Lexicon](https://softwarepatternslexicon.com/python/architectural-patterns/hexagonal-architecture-ports-and-adapters/)
- [From Hexagonal to Clean Architecture - CodeArtify](https://codeartify.substack.com/p/from-hexagonal-to-clean-architecture)

---

*마지막 업데이트: 2025년 1월*

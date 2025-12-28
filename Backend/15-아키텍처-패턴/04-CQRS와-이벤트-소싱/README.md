# CQRS와 이벤트 소싱 (CQRS & Event Sourcing)

## 목차
1. [개요](#개요)
2. [CQRS 패턴 상세](#cqrs-패턴-상세)
3. [Command와 Query 분리](#command와-query-분리)
4. [이벤트 소싱 (Event Sourcing)](#이벤트-소싱-event-sourcing)
5. [이벤트 저장소](#이벤트-저장소)
6. [프로젝션과 Read Model](#프로젝션과-read-model)
7. [최종 일관성](#최종-일관성)
8. [실무 구현 예제](#실무-구현-예제)
---

## 개요

CQRS(Command Query Responsibility Segregation)와 Event Sourcing은 복잡한 도메인에서 **읽기와 쓰기를 분리**하고, **상태 변경을 이벤트로 저장**하는 아키텍처 패턴이다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CQRS + Event Sourcing 개요                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  전통적인 CRUD                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Client  ←→  Service  ←→  Repository  ←→  Database          │    │
│  │         (읽기/쓰기 동일 모델, 동일 경로)                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  CQRS                                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              ┌─────────────────────────────────────┐        │    │
│  │  Command ───►│  Command Handler  ───►  Write DB   │        │    │
│  │              └─────────────────────────────────────┘        │    │
│  │                                                              │    │
│  │              ┌─────────────────────────────────────┐        │    │
│  │  Query   ───►│  Query Handler    ───►  Read DB    │        │    │
│  │              └─────────────────────────────────────┘        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  CQRS + Event Sourcing                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                                              │    │
│  │  Command ───► Aggregate ───► Events ───► Event Store        │    │
│  │                                    │                         │    │
│  │                                    ▼                         │    │
│  │                              Projection                      │    │
│  │                                    │                         │    │
│  │                                    ▼                         │    │
│  │  Query   ───► Query Handler ───► Read Model (DB/Cache)      │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 왜 CQRS와 Event Sourcing을 사용하는가?

```
┌─────────────────────────────────────────────────────────────────────┐
│                         사용 이유                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  CQRS의 장점:                                                        │
│  ─────────────────────────────────────────────────────────────────  │
│  • 읽기/쓰기 독립 확장 (읽기가 많은 시스템에서 Read DB만 스케일아웃)    │
│  • 각각에 최적화된 데이터 모델 사용 가능                              │
│  • 복잡한 도메인 로직과 간단한 조회 로직 분리                         │
│  • 보안: 읽기/쓰기에 다른 권한 적용 용이                              │
│                                                                      │
│  Event Sourcing의 장점:                                              │
│  ─────────────────────────────────────────────────────────────────  │
│  • 완전한 감사 추적 (Audit Trail) - 모든 변경 기록                    │
│  • 시간 여행 (Time Travel) - 과거 어느 시점 상태 재현                 │
│  • 디버깅 - 버그 발생 경위 추적                                       │
│  • 이벤트 재생 (Replay) - 새로운 Read Model 생성                     │
│  • 도메인 이벤트 자연스럽게 도출                                      │
│                                                                      │
│  적합한 사용 사례:                                                    │
│  ─────────────────────────────────────────────────────────────────  │
│  • 금융 시스템 (거래 내역 추적 필수)                                  │
│  • 예약 시스템 (변경 이력 중요)                                       │
│  • 협업 도구 (실시간 동기화)                                          │
│  • 쇼핑몰 장바구니/주문 (상태 변경 추적)                              │
│                                                                      │
│  부적합한 사용 사례:                                                  │
│  ─────────────────────────────────────────────────────────────────  │
│  • 단순 CRUD 애플리케이션                                            │
│  • 실시간 일관성이 필수인 시스템                                      │
│  • 팀에 경험이 부족한 경우                                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## CQRS 패턴 상세

### 기본 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CQRS 아키텍처                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                          Client                                      │
│                            │                                         │
│              ┌─────────────┴─────────────┐                          │
│              │                           │                           │
│              ▼                           ▼                           │
│     ┌─────────────────┐         ┌─────────────────┐                 │
│     │    Commands     │         │     Queries     │                 │
│     │   (POST, PUT,   │         │   (GET)         │                 │
│     │    DELETE)      │         │                 │                 │
│     └────────┬────────┘         └────────┬────────┘                 │
│              │                           │                           │
│              ▼                           ▼                           │
│     ┌─────────────────┐         ┌─────────────────┐                 │
│     │ Command Handler │         │  Query Handler  │                 │
│     │                 │         │                 │                 │
│     │ - Validation    │         │ - Simple Read   │                 │
│     │ - Domain Logic  │         │ - No Side Effect│                 │
│     │ - State Change  │         │                 │                 │
│     └────────┬────────┘         └────────┬────────┘                 │
│              │                           │                           │
│              ▼                           ▼                           │
│     ┌─────────────────┐         ┌─────────────────┐                 │
│     │   Write Model   │         │   Read Model    │                 │
│     │                 │         │                 │                 │
│     │ - Normalized    │  Sync   │ - Denormalized  │                 │
│     │ - Consistency   │◄───────►│ - Optimized     │                 │
│     │ - Domain Entity │         │ - Query-ready   │                 │
│     └─────────────────┘         └─────────────────┘                 │
│              │                           │                           │
│              ▼                           ▼                           │
│     ┌─────────────────┐         ┌─────────────────┐                 │
│     │   Write DB      │         │   Read DB       │                 │
│     │ (PostgreSQL)    │         │ (Elasticsearch, │                 │
│     │                 │         │  Redis, etc.)   │                 │
│     └─────────────────┘         └─────────────────┘                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### CQRS 구현 레벨

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CQRS 구현 복잡도 수준                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Level 1: 단일 DB + 코드 분리                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Command Service    Query Service                            │    │
│  │        │                   │                                 │    │
│  │        └───────┬───────────┘                                 │    │
│  │                │                                              │    │
│  │         [ Single DB ]                                        │    │
│  │                                                               │    │
│  │  장점: 구현 단순, 강한 일관성                                  │    │
│  │  단점: 스케일링 제한                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Level 2: 단일 DB + Read Replica                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Command Service          Query Service                      │    │
│  │        │                        │                            │    │
│  │        ▼                        ▼                            │    │
│  │   [ Write DB ] ──replication──► [ Read Replica ]             │    │
│  │                                                               │    │
│  │  장점: 읽기 확장성 향상, 구현 간단                             │    │
│  │  단점: 복제 지연 (Replication Lag)                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Level 3: 분리된 DB + 동기화                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Command Service              Query Service                  │    │
│  │        │                            │                        │    │
│  │        ▼                            ▼                        │    │
│  │   [ Write DB ]    Events     [ Read DB ]                     │    │
│  │   (PostgreSQL) ──────────►  (Elasticsearch)                  │    │
│  │                                                               │    │
│  │  장점: 각각 최적화, 독립 스케일링                              │    │
│  │  단점: 최종 일관성, 동기화 복잡                                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Level 4: Event Sourcing + 다중 Read Model                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Command Handler                                             │    │
│  │        │                                                     │    │
│  │        ▼                                                     │    │
│  │  [ Event Store ]                                             │    │
│  │        │                                                     │    │
│  │        ├──► Projection A ──► [ Read DB 1 ] (상세 조회용)      │    │
│  │        ├──► Projection B ──► [ Read DB 2 ] (검색용)          │    │
│  │        └──► Projection C ──► [ Read DB 3 ] (통계용)          │    │
│  │                                                               │    │
│  │  장점: 완전한 유연성, 이벤트 재생                              │    │
│  │  단점: 높은 복잡도, 러닝 커브                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Command와 Query 분리

### Command 구현

```java
// Command 정의 (불변, 의도를 나타내는 이름)
@Value
public class PlaceOrderCommand {
    @NotNull
    String customerId;

    @NotEmpty
    List<OrderItem> items;

    @NotNull
    ShippingAddress shippingAddress;

    @Value
    public static class OrderItem {
        String productId;
        int quantity;
    }
}

@Value
public class CancelOrderCommand {
    @NotNull
    String orderId;

    @NotBlank
    String reason;

    @NotNull
    String cancelledBy;
}

// Command Handler
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderCommandHandler {

    private final OrderRepository orderRepository;
    private final EventStore eventStore;
    private final DomainEventPublisher eventPublisher;

    @CommandHandler
    @Transactional
    public String handle(PlaceOrderCommand command) {
        log.info("Handling PlaceOrderCommand for customer: {}", command.getCustomerId());

        // 1. Command 검증
        validateCommand(command);

        // 2. Aggregate 생성 및 도메인 로직 실행
        Order order = Order.place(
            new CustomerId(command.getCustomerId()),
            toOrderLines(command.getItems()),
            command.getShippingAddress()
        );

        // 3. 이벤트 저장 (Event Sourcing 사용 시)
        List<DomainEvent> events = order.getDomainEvents();
        eventStore.appendEvents(order.getId(), events);

        // 4. 이벤트 발행 (Read Model 동기화용)
        events.forEach(eventPublisher::publish);

        return order.getId().getValue();
    }

    @CommandHandler
    @Transactional
    public void handle(CancelOrderCommand command) {
        log.info("Handling CancelOrderCommand for order: {}", command.getOrderId());

        // 1. Aggregate 로드
        Order order = loadAggregate(command.getOrderId());

        // 2. 도메인 로직 실행
        order.cancel(command.getReason());

        // 3. 이벤트 저장 및 발행
        eventStore.appendEvents(order.getId(), order.getDomainEvents());
        order.getDomainEvents().forEach(eventPublisher::publish);
    }

    private Order loadAggregate(String orderId) {
        // Event Sourcing: 이벤트로부터 Aggregate 재구성
        List<DomainEvent> events = eventStore.getEvents(new OrderId(orderId));
        return Order.rehydrate(events);
    }
}

// Command Bus (Mediator 패턴)
public interface CommandBus {
    <R> R dispatch(Object command);
}

@Component
@RequiredArgsConstructor
public class SimpleCommandBus implements CommandBus {

    private final ApplicationContext applicationContext;

    @Override
    @SuppressWarnings("unchecked")
    public <R> R dispatch(Object command) {
        // Handler 찾기
        Class<?> commandType = command.getClass();
        Object handler = findHandler(commandType);

        // Handler 메서드 실행
        Method method = findHandlerMethod(handler, commandType);
        try {
            return (R) method.invoke(handler, command);
        } catch (Exception e) {
            throw new CommandExecutionException(e);
        }
    }
}
```

### Query 구현

```java
// Query 정의
@Value
public class GetOrderQuery {
    String orderId;
}

@Value
public class GetOrdersByCustomerQuery {
    String customerId;
    int page;
    int size;
    OrderSortField sortBy;
    SortDirection direction;
}

@Value
public class SearchOrdersQuery {
    String searchTerm;
    LocalDate fromDate;
    LocalDate toDate;
    List<OrderStatus> statuses;
    int page;
    int size;
}

// Query 결과 (Read Model / DTO)
@Value
public class OrderView {
    String orderId;
    String customerId;
    String customerName;
    List<OrderLineView> orderLines;
    String status;
    BigDecimal totalAmount;
    LocalDateTime createdAt;
    LocalDateTime updatedAt;

    @Value
    public static class OrderLineView {
        String productId;
        String productName;
        int quantity;
        BigDecimal unitPrice;
        BigDecimal lineTotal;
    }
}

@Value
public class OrderListView {
    String orderId;
    String customerName;
    String status;
    BigDecimal totalAmount;
    LocalDateTime createdAt;
    int itemCount;
}

// Query Handler
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderQueryHandler {

    private final OrderReadRepository orderReadRepository;
    private final OrderSearchRepository orderSearchRepository;  // Elasticsearch

    @QueryHandler
    public Optional<OrderView> handle(GetOrderQuery query) {
        log.debug("Handling GetOrderQuery: {}", query.getOrderId());

        return orderReadRepository.findById(query.getOrderId());
    }

    @QueryHandler
    public Page<OrderListView> handle(GetOrdersByCustomerQuery query) {
        log.debug("Handling GetOrdersByCustomerQuery: {}", query.getCustomerId());

        Pageable pageable = PageRequest.of(
            query.getPage(),
            query.getSize(),
            Sort.by(query.getDirection(), query.getSortBy().getFieldName())
        );

        return orderReadRepository.findByCustomerId(query.getCustomerId(), pageable);
    }

    @QueryHandler
    public Page<OrderListView> handle(SearchOrdersQuery query) {
        log.debug("Handling SearchOrdersQuery: {}", query.getSearchTerm());

        // Elasticsearch에서 검색
        return orderSearchRepository.search(
            query.getSearchTerm(),
            query.getFromDate(),
            query.getToDate(),
            query.getStatuses(),
            PageRequest.of(query.getPage(), query.getSize())
        );
    }
}

// Query Bus
public interface QueryBus {
    <R> R query(Object query);
}

// Read Repository (Read Model 전용)
public interface OrderReadRepository {
    Optional<OrderView> findById(String orderId);
    Page<OrderListView> findByCustomerId(String customerId, Pageable pageable);
    List<OrderListView> findRecentOrders(String customerId, int limit);
}

@Repository
@RequiredArgsConstructor
public class OrderReadJpaRepository implements OrderReadRepository {

    private final OrderViewJpaRepository jpaRepository;

    @Override
    @Transactional(readOnly = true)  // 읽기 전용 트랜잭션
    public Optional<OrderView> findById(String orderId) {
        return jpaRepository.findById(orderId)
            .map(this::toView);
    }

    @Override
    @Transactional(readOnly = true)
    public Page<OrderListView> findByCustomerId(String customerId, Pageable pageable) {
        return jpaRepository.findByCustomerId(customerId, pageable)
            .map(this::toListView);
    }
}
```

### Controller에서 Command/Query 분리

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {

    private final CommandBus commandBus;
    private final QueryBus queryBus;

    // ===== Commands (상태 변경) =====

    @PostMapping
    public ResponseEntity<CreateOrderResponse> placeOrder(
            @Valid @RequestBody PlaceOrderRequest request) {

        PlaceOrderCommand command = request.toCommand();
        String orderId = commandBus.dispatch(command);

        return ResponseEntity
            .created(URI.create("/api/orders/" + orderId))
            .body(new CreateOrderResponse(orderId));
    }

    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<Void> cancelOrder(
            @PathVariable String orderId,
            @Valid @RequestBody CancelOrderRequest request) {

        CancelOrderCommand command = new CancelOrderCommand(
            orderId,
            request.getReason(),
            getCurrentUserId()
        );
        commandBus.dispatch(command);

        return ResponseEntity.noContent().build();
    }

    // ===== Queries (조회) =====

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderView> getOrder(@PathVariable String orderId) {
        return queryBus.query(new GetOrderQuery(orderId))
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    @GetMapping
    public ResponseEntity<Page<OrderListView>> getOrders(
            @RequestParam String customerId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "CREATED_AT") OrderSortField sortBy,
            @RequestParam(defaultValue = "DESC") SortDirection direction) {

        GetOrdersByCustomerQuery query = new GetOrdersByCustomerQuery(
            customerId, page, size, sortBy, direction
        );
        Page<OrderListView> result = queryBus.query(query);

        return ResponseEntity.ok(result);
    }

    @GetMapping("/search")
    public ResponseEntity<Page<OrderListView>> searchOrders(
            @RequestParam(required = false) String q,
            @RequestParam(required = false) LocalDate from,
            @RequestParam(required = false) LocalDate to,
            @RequestParam(required = false) List<OrderStatus> statuses,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {

        SearchOrdersQuery query = new SearchOrdersQuery(
            q, from, to, statuses, page, size
        );
        Page<OrderListView> result = queryBus.query(query);

        return ResponseEntity.ok(result);
    }
}
```

---

## 이벤트 소싱 (Event Sourcing)

### 핵심 개념

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Event Sourcing 개념                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  전통적인 상태 저장:                                                  │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  Order Table:                                                       │
│  ┌────────┬──────────┬────────┬──────────────┐                     │
│  │   ID   │  Status  │ Amount │  Updated_At  │                     │
│  ├────────┼──────────┼────────┼──────────────┤                     │
│  │  O001  │ SHIPPED  │ 50000  │ 2024-01-15   │  ← 현재 상태만 저장  │
│  └────────┴──────────┴────────┴──────────────┘                     │
│                                                                      │
│  문제: 어떻게 SHIPPED가 됐는지? 이전 상태는?                         │
│                                                                      │
│  Event Sourcing:                                                    │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  Events Table:                                                      │
│  ┌────────┬─────────────────┬──────────────┬──────────────────────┐│
│  │Aggr ID │   Event Type    │    Data      │     Occurred_At     ││
│  ├────────┼─────────────────┼──────────────┼──────────────────────┤│
│  │  O001  │ OrderPlaced     │ {items: ...} │ 2024-01-10 10:00    ││
│  │  O001  │ PaymentReceived │ {amount: ..} │ 2024-01-10 10:30    ││
│  │  O001  │ OrderConfirmed  │ {}           │ 2024-01-10 10:31    ││
│  │  O001  │ OrderShipped    │ {carrier:..} │ 2024-01-15 09:00    ││
│  └────────┴─────────────────┴──────────────┴──────────────────────┘│
│                                                                      │
│  현재 상태 = 모든 이벤트를 순서대로 적용한 결과                        │
│                                                                      │
│  ┌─────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │ 초기 │───►│OrderPlaced  │───►│OrderConfirm │───►│OrderShipped │  │
│  │ 상태 │    │  (이벤트1)  │    │  (이벤트2)  │    │  (이벤트3)  │  │
│  └─────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│                                                                      │
│              상태: PLACED        상태: CONFIRMED    상태: SHIPPED    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Event Sourced Aggregate 구현

```java
// Event 기본 인터페이스
public interface DomainEvent {
    String getAggregateId();
    String getEventType();
    Instant getOccurredAt();
    long getVersion();
}

@Getter
public abstract class BaseDomainEvent implements DomainEvent {
    private final String eventId;
    private final String aggregateId;
    private final Instant occurredAt;
    private long version;

    protected BaseDomainEvent(String aggregateId) {
        this.eventId = UUID.randomUUID().toString();
        this.aggregateId = aggregateId;
        this.occurredAt = Instant.now();
    }

    @Override
    public String getEventType() {
        return this.getClass().getSimpleName();
    }

    public void setVersion(long version) {
        this.version = version;
    }
}

// 구체적인 이벤트들
@Getter
public class OrderPlacedEvent extends BaseDomainEvent {
    private final String customerId;
    private final List<OrderLineData> orderLines;
    private final BigDecimal totalAmount;
    private final AddressData shippingAddress;

    public OrderPlacedEvent(String orderId, String customerId,
                           List<OrderLineData> orderLines,
                           BigDecimal totalAmount,
                           AddressData shippingAddress) {
        super(orderId);
        this.customerId = customerId;
        this.orderLines = orderLines;
        this.totalAmount = totalAmount;
        this.shippingAddress = shippingAddress;
    }
}

@Getter
public class OrderConfirmedEvent extends BaseDomainEvent {
    private final Instant confirmedAt;

    public OrderConfirmedEvent(String orderId) {
        super(orderId);
        this.confirmedAt = Instant.now();
    }
}

@Getter
public class OrderCancelledEvent extends BaseDomainEvent {
    private final String reason;
    private final String cancelledBy;

    public OrderCancelledEvent(String orderId, String reason, String cancelledBy) {
        super(orderId);
        this.reason = reason;
        this.cancelledBy = cancelledBy;
    }
}

// Event Sourced Aggregate
public class Order {

    private OrderId id;
    private CustomerId customerId;
    private List<OrderLine> orderLines;
    private OrderStatus status;
    private Money totalAmount;
    private ShippingAddress shippingAddress;
    private LocalDateTime createdAt;
    private LocalDateTime confirmedAt;
    private long version;

    // 변경 사항을 이벤트로 기록
    private final List<DomainEvent> uncommittedEvents = new ArrayList<>();

    // ===== Factory Method (새로운 Aggregate 생성) =====
    public static Order place(CustomerId customerId,
                             List<OrderLineRequest> lineRequests,
                             ShippingAddress shippingAddress) {

        Order order = new Order();

        // 이벤트 생성 및 적용
        OrderPlacedEvent event = new OrderPlacedEvent(
            OrderId.generate().getValue(),
            customerId.getValue(),
            toOrderLineData(lineRequests),
            calculateTotal(lineRequests),
            toAddressData(shippingAddress)
        );

        order.apply(event);
        order.uncommittedEvents.add(event);

        return order;
    }

    // ===== Command Methods (상태 변경 요청) =====
    public void confirm() {
        // 비즈니스 규칙 검증
        if (this.status != OrderStatus.PENDING_PAYMENT) {
            throw new InvalidOrderStateException(
                "결제 대기 상태에서만 확정할 수 있습니다"
            );
        }

        // 이벤트 생성 및 적용
        OrderConfirmedEvent event = new OrderConfirmedEvent(this.id.getValue());
        apply(event);
        uncommittedEvents.add(event);
    }

    public void cancel(String reason) {
        if (this.status.isNotCancellable()) {
            throw new OrderCannotBeCancelledException(
                "취소할 수 없는 상태입니다: " + this.status
            );
        }

        OrderCancelledEvent event = new OrderCancelledEvent(
            this.id.getValue(),
            reason,
            "SYSTEM"
        );
        apply(event);
        uncommittedEvents.add(event);
    }

    // ===== Event Application (이벤트 적용) =====
    private void apply(DomainEvent event) {
        // 이벤트 타입에 따라 상태 변경
        if (event instanceof OrderPlacedEvent e) {
            applyOrderPlaced(e);
        } else if (event instanceof OrderConfirmedEvent e) {
            applyOrderConfirmed(e);
        } else if (event instanceof OrderCancelledEvent e) {
            applyOrderCancelled(e);
        }
        // ... 다른 이벤트 타입들

        this.version++;
    }

    private void applyOrderPlaced(OrderPlacedEvent event) {
        this.id = new OrderId(event.getAggregateId());
        this.customerId = new CustomerId(event.getCustomerId());
        this.orderLines = toOrderLines(event.getOrderLines());
        this.totalAmount = new Money(event.getTotalAmount(), Currency.KRW);
        this.shippingAddress = toShippingAddress(event.getShippingAddress());
        this.status = OrderStatus.PLACED;
        this.createdAt = LocalDateTime.ofInstant(event.getOccurredAt(), ZoneId.systemDefault());
    }

    private void applyOrderConfirmed(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
        this.confirmedAt = LocalDateTime.ofInstant(event.getConfirmedAt(), ZoneId.systemDefault());
    }

    private void applyOrderCancelled(OrderCancelledEvent event) {
        this.status = OrderStatus.CANCELLED;
    }

    // ===== Rehydration (이벤트로부터 상태 복원) =====
    public static Order rehydrate(List<DomainEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }

    // ===== Uncommitted Events =====
    public List<DomainEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void clearUncommittedEvents() {
        uncommittedEvents.clear();
    }

    public long getVersion() {
        return version;
    }
}
```

### Snapshot 패턴

이벤트가 많이 쌓이면 Aggregate 복원이 느려질 수 있다. Snapshot으로 최적화한다.

```java
// Snapshot Entity
@Value
public class AggregateSnapshot {
    String aggregateId;
    String aggregateType;
    long version;
    byte[] state;  // 직렬화된 Aggregate 상태
    Instant createdAt;
}

// Snapshot 적용 Event Store
@Component
@RequiredArgsConstructor
public class SnapshottingEventStore implements EventStore {

    private final EventJpaRepository eventRepository;
    private final SnapshotJpaRepository snapshotRepository;
    private final ObjectMapper objectMapper;

    private static final int SNAPSHOT_THRESHOLD = 100;  // 100개 이벤트마다 스냅샷

    @Override
    public <T extends AggregateRoot> T load(String aggregateId, Class<T> aggregateType) {
        // 1. 가장 최근 스냅샷 조회
        Optional<AggregateSnapshot> snapshot = snapshotRepository
            .findLatestByAggregateId(aggregateId);

        // 2. 스냅샷 이후의 이벤트만 조회
        long fromVersion = snapshot.map(AggregateSnapshot::getVersion).orElse(0L);
        List<DomainEvent> events = eventRepository
            .findByAggregateIdAndVersionGreaterThan(aggregateId, fromVersion);

        // 3. Aggregate 복원
        T aggregate;
        if (snapshot.isPresent()) {
            aggregate = deserialize(snapshot.get(), aggregateType);
            events.forEach(aggregate::apply);
        } else {
            aggregate = reconstructFromEvents(aggregateId, aggregateType, events);
        }

        return aggregate;
    }

    @Override
    public void save(AggregateRoot aggregate) {
        List<DomainEvent> newEvents = aggregate.getUncommittedEvents();

        // 이벤트 저장
        eventRepository.saveAll(toEventEntities(newEvents));

        // 스냅샷 필요 여부 확인
        if (shouldTakeSnapshot(aggregate)) {
            takeSnapshot(aggregate);
        }

        aggregate.clearUncommittedEvents();
    }

    private boolean shouldTakeSnapshot(AggregateRoot aggregate) {
        return aggregate.getVersion() % SNAPSHOT_THRESHOLD == 0;
    }

    private void takeSnapshot(AggregateRoot aggregate) {
        AggregateSnapshot snapshot = new AggregateSnapshot(
            aggregate.getId(),
            aggregate.getClass().getName(),
            aggregate.getVersion(),
            serialize(aggregate),
            Instant.now()
        );
        snapshotRepository.save(toEntity(snapshot));
    }
}
```

---

## 이벤트 저장소

### Event Store 구현

```java
// Event Store 인터페이스
public interface EventStore {

    void appendEvents(String aggregateId, List<DomainEvent> events);

    void appendEvents(String aggregateId, List<DomainEvent> events, long expectedVersion);

    List<DomainEvent> getEvents(String aggregateId);

    List<DomainEvent> getEvents(String aggregateId, long fromVersion);

    Optional<Long> getCurrentVersion(String aggregateId);
}

// JPA 기반 Event Store 구현
@Component
@RequiredArgsConstructor
public class JpaEventStore implements EventStore {

    private final EventJpaRepository repository;
    private final EventSerializer serializer;

    @Override
    @Transactional
    public void appendEvents(String aggregateId, List<DomainEvent> events) {
        appendEvents(aggregateId, events, -1);  // 버전 체크 안 함
    }

    @Override
    @Transactional
    public void appendEvents(String aggregateId, List<DomainEvent> events,
                            long expectedVersion) {
        // Optimistic Concurrency Control
        if (expectedVersion >= 0) {
            Long currentVersion = repository.findMaxVersionByAggregateId(aggregateId)
                .orElse(0L);

            if (!currentVersion.equals(expectedVersion)) {
                throw new OptimisticLockingException(
                    "Aggregate가 수정되었습니다. Expected: " + expectedVersion +
                    ", Current: " + currentVersion
                );
            }
        }

        // 이벤트 저장
        long version = expectedVersion >= 0 ? expectedVersion : getNextVersion(aggregateId);

        for (DomainEvent event : events) {
            EventEntity entity = EventEntity.builder()
                .eventId(event.getEventId())
                .aggregateId(aggregateId)
                .aggregateType(event.getClass().getSimpleName().replace("Event", ""))
                .eventType(event.getEventType())
                .eventData(serializer.serialize(event))
                .version(++version)
                .occurredAt(event.getOccurredAt())
                .createdAt(Instant.now())
                .build();

            repository.save(entity);
        }
    }

    @Override
    @Transactional(readOnly = true)
    public List<DomainEvent> getEvents(String aggregateId) {
        return repository.findByAggregateIdOrderByVersionAsc(aggregateId)
            .stream()
            .map(entity -> serializer.deserialize(entity.getEventData(), entity.getEventType()))
            .collect(toList());
    }

    @Override
    @Transactional(readOnly = true)
    public List<DomainEvent> getEvents(String aggregateId, long fromVersion) {
        return repository.findByAggregateIdAndVersionGreaterThanOrderByVersionAsc(
                aggregateId, fromVersion)
            .stream()
            .map(entity -> serializer.deserialize(entity.getEventData(), entity.getEventType()))
            .collect(toList());
    }

    private long getNextVersion(String aggregateId) {
        return repository.findMaxVersionByAggregateId(aggregateId)
            .orElse(0L);
    }
}

// Event Entity
@Entity
@Table(name = "event_store",
       indexes = {
           @Index(name = "idx_aggregate_id", columnList = "aggregateId"),
           @Index(name = "idx_aggregate_version", columnList = "aggregateId, version", unique = true),
           @Index(name = "idx_occurred_at", columnList = "occurredAt")
       })
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
@Builder
public class EventEntity {

    @Id
    @Column(length = 36)
    private String eventId;

    @Column(nullable = false, length = 36)
    private String aggregateId;

    @Column(nullable = false, length = 100)
    private String aggregateType;

    @Column(nullable = false, length = 100)
    private String eventType;

    @Lob
    @Column(nullable = false)
    private String eventData;  // JSON

    @Column(nullable = false)
    private long version;

    @Column(nullable = false)
    private Instant occurredAt;

    @Column(nullable = false)
    private Instant createdAt;
}

// Event Serializer
@Component
@RequiredArgsConstructor
public class JsonEventSerializer implements EventSerializer {

    private final ObjectMapper objectMapper;
    private final Map<String, Class<? extends DomainEvent>> eventTypeRegistry;

    @Override
    public String serialize(DomainEvent event) {
        try {
            return objectMapper.writeValueAsString(event);
        } catch (JsonProcessingException e) {
            throw new EventSerializationException("Failed to serialize event", e);
        }
    }

    @Override
    public DomainEvent deserialize(String json, String eventType) {
        Class<? extends DomainEvent> eventClass = eventTypeRegistry.get(eventType);
        if (eventClass == null) {
            throw new UnknownEventTypeException("Unknown event type: " + eventType);
        }

        try {
            return objectMapper.readValue(json, eventClass);
        } catch (JsonProcessingException e) {
            throw new EventDeserializationException("Failed to deserialize event", e);
        }
    }
}
```

---

## 프로젝션과 Read Model

### Projection 구현

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Projection 개념                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  "이벤트를 특정 목적에 맞는 읽기 모델로 변환하는 과정"                  │
│                                                                      │
│  Event Store                                                        │
│       │                                                             │
│       │  OrderPlaced, OrderConfirmed, OrderShipped, ...             │
│       │                                                             │
│       ├──► Projection 1: Order Details                              │
│       │    └──► Read Model: order_details (PostgreSQL)              │
│       │                                                             │
│       ├──► Projection 2: Order Summary                              │
│       │    └──► Read Model: order_summaries (Redis)                 │
│       │                                                             │
│       ├──► Projection 3: Search Index                               │
│       │    └──► Read Model: orders_index (Elasticsearch)            │
│       │                                                             │
│       └──► Projection 4: Statistics                                 │
│            └──► Read Model: order_stats (ClickHouse)                │
│                                                                      │
│  각 Projection은 동일한 이벤트를 다른 방식으로 해석                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// Projection 인터페이스
public interface Projection<E extends DomainEvent> {
    void project(E event);
    boolean supports(Class<? extends DomainEvent> eventType);
}

// Order Details Projection
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderDetailsProjection implements Projection<DomainEvent> {

    private final OrderViewRepository orderViewRepository;
    private final CustomerQueryService customerQueryService;

    @Override
    public boolean supports(Class<? extends DomainEvent> eventType) {
        return OrderPlacedEvent.class.isAssignableFrom(eventType) ||
               OrderConfirmedEvent.class.isAssignableFrom(eventType) ||
               OrderCancelledEvent.class.isAssignableFrom(eventType) ||
               OrderShippedEvent.class.isAssignableFrom(eventType);
    }

    @Override
    @Transactional
    public void project(DomainEvent event) {
        if (event instanceof OrderPlacedEvent e) {
            handleOrderPlaced(e);
        } else if (event instanceof OrderConfirmedEvent e) {
            handleOrderConfirmed(e);
        } else if (event instanceof OrderCancelledEvent e) {
            handleOrderCancelled(e);
        } else if (event instanceof OrderShippedEvent e) {
            handleOrderShipped(e);
        }
    }

    private void handleOrderPlaced(OrderPlacedEvent event) {
        log.info("Projecting OrderPlacedEvent: {}", event.getAggregateId());

        // 고객 정보 조회 (다른 서비스에서)
        CustomerInfo customer = customerQueryService
            .getCustomerInfo(event.getCustomerId());

        // Read Model 생성
        OrderViewEntity view = OrderViewEntity.builder()
            .orderId(event.getAggregateId())
            .customerId(event.getCustomerId())
            .customerName(customer.getName())
            .customerEmail(customer.getEmail())
            .status(OrderStatus.PLACED.name())
            .totalAmount(event.getTotalAmount())
            .orderLines(toOrderLineViews(event.getOrderLines()))
            .shippingAddress(event.getShippingAddress())
            .createdAt(event.getOccurredAt())
            .updatedAt(event.getOccurredAt())
            .build();

        orderViewRepository.save(view);
    }

    private void handleOrderConfirmed(OrderConfirmedEvent event) {
        log.info("Projecting OrderConfirmedEvent: {}", event.getAggregateId());

        orderViewRepository.findById(event.getAggregateId())
            .ifPresent(view -> {
                view.setStatus(OrderStatus.CONFIRMED.name());
                view.setConfirmedAt(event.getConfirmedAt());
                view.setUpdatedAt(event.getOccurredAt());
                orderViewRepository.save(view);
            });
    }

    private void handleOrderCancelled(OrderCancelledEvent event) {
        log.info("Projecting OrderCancelledEvent: {}", event.getAggregateId());

        orderViewRepository.findById(event.getAggregateId())
            .ifPresent(view -> {
                view.setStatus(OrderStatus.CANCELLED.name());
                view.setCancelReason(event.getReason());
                view.setUpdatedAt(event.getOccurredAt());
                orderViewRepository.save(view);
            });
    }
}

// Elasticsearch Projection (검색용)
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderSearchProjection implements Projection<DomainEvent> {

    private final ElasticsearchClient esClient;
    private final String INDEX_NAME = "orders";

    @Override
    @Async  // 비동기 처리
    public void project(DomainEvent event) {
        if (event instanceof OrderPlacedEvent e) {
            indexOrder(e);
        } else if (event instanceof OrderConfirmedEvent e) {
            updateOrderStatus(e.getAggregateId(), "CONFIRMED");
        } else if (event instanceof OrderCancelledEvent e) {
            updateOrderStatus(e.getAggregateId(), "CANCELLED");
        }
    }

    private void indexOrder(OrderPlacedEvent event) {
        OrderSearchDocument doc = OrderSearchDocument.builder()
            .orderId(event.getAggregateId())
            .customerId(event.getCustomerId())
            .status("PLACED")
            .totalAmount(event.getTotalAmount())
            .productNames(extractProductNames(event.getOrderLines()))
            .createdAt(event.getOccurredAt())
            .build();

        esClient.index(i -> i
            .index(INDEX_NAME)
            .id(event.getAggregateId())
            .document(doc)
        );
    }

    private void updateOrderStatus(String orderId, String status) {
        esClient.update(u -> u
            .index(INDEX_NAME)
            .id(orderId)
            .doc(Map.of("status", status, "updatedAt", Instant.now()))
        );
    }
}

// Projection Manager (이벤트 구독 및 라우팅)
@Component
@RequiredArgsConstructor
@Slf4j
public class ProjectionManager {

    private final List<Projection<DomainEvent>> projections;

    @EventListener  // Spring Event 사용
    public void onDomainEvent(DomainEvent event) {
        log.debug("Routing event to projections: {}", event.getEventType());

        projections.stream()
            .filter(p -> p.supports(event.getClass()))
            .forEach(p -> {
                try {
                    p.project(event);
                } catch (Exception e) {
                    log.error("Projection failed for event {}: {}",
                        event.getEventType(), e.getMessage());
                    // 실패한 projection 재시도 로직...
                }
            });
    }

    // Kafka Consumer 사용 시
    @KafkaListener(topics = "order-events", groupId = "order-projection")
    public void onKafkaEvent(String eventJson) {
        DomainEvent event = deserialize(eventJson);
        onDomainEvent(event);
    }
}
```

### Live Projection vs Catch-up Projection

```java
// Live Projection: 실시간 이벤트 처리
@Component
public class LiveOrderProjection {

    @KafkaListener(topics = "order-events", groupId = "live-projection")
    public void handle(DomainEvent event) {
        // 실시간으로 Read Model 업데이트
        updateReadModel(event);
    }
}

// Catch-up Projection: 이벤트 재생
@Component
@RequiredArgsConstructor
@Slf4j
public class CatchUpProjectionRunner {

    private final EventStore eventStore;
    private final ProjectionCheckpointRepository checkpointRepository;
    private final List<Projection<DomainEvent>> projections;

    // 전체 이벤트 재생 (새로운 Read Model 생성 시)
    public void rebuildProjection(String projectionName) {
        log.info("Starting full rebuild for projection: {}", projectionName);

        Projection<DomainEvent> projection = findProjection(projectionName);

        // 체크포인트 초기화
        checkpointRepository.reset(projectionName);

        // 모든 이벤트 재생
        eventStore.streamAllEvents()
            .filter(event -> projection.supports(event.getClass()))
            .forEach(event -> {
                projection.project(event);
                updateCheckpoint(projectionName, event);
            });

        log.info("Rebuild completed for projection: {}", projectionName);
    }

    // 특정 시점부터 재생 (장애 복구 시)
    @Scheduled(fixedDelay = 60000)  // 1분마다 체크
    public void catchUp() {
        for (Projection<DomainEvent> projection : projections) {
            String projectionName = projection.getClass().getSimpleName();
            ProjectionCheckpoint checkpoint = checkpointRepository
                .findByProjectionName(projectionName)
                .orElse(new ProjectionCheckpoint(projectionName, 0L));

            List<DomainEvent> missedEvents = eventStore
                .getEventsAfter(checkpoint.getLastProcessedPosition());

            missedEvents.stream()
                .filter(event -> projection.supports(event.getClass()))
                .forEach(event -> {
                    projection.project(event);
                    updateCheckpoint(projectionName, event);
                });
        }
    }
}

// Projection Checkpoint
@Entity
@Table(name = "projection_checkpoints")
@Getter @Setter
public class ProjectionCheckpoint {

    @Id
    private String projectionName;

    private long lastProcessedPosition;

    private Instant lastProcessedAt;
}
```

---

## 최종 일관성

### Eventual Consistency 이해

```
┌─────────────────────────────────────────────────────────────────────┐
│                      최종 일관성 (Eventual Consistency)               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  "Write와 Read 사이에 시간 차이가 있을 수 있다"                        │
│                                                                      │
│  Timeline:                                                          │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                      │
│  t0: Client가 주문 생성 요청                                         │
│      │                                                              │
│  t1: │ Command Handler가 Event Store에 이벤트 저장 ✓                 │
│      │     │                                                        │
│  t2: │     │  Client에게 응답 반환 (orderId)                        │
│      │     │     │                                                  │
│  t3: │     │     │  Projection이 이벤트 수신                        │
│      │     │     │     │                                            │
│  t4: │     │     │     │  Read Model 업데이트 ✓                     │
│      │     │     │     │     │                                      │
│      ▼     ▼     ▼     ▼     ▼                                      │
│  ──────────────────────────────────────────────────►  time          │
│                                                                      │
│  t2~t4 사이: Client가 조회하면 새 주문이 안 보일 수 있음!              │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  이 시간 차이는 보통 밀리초~초 단위                           │    │
│  │  하지만 장애 상황에서는 더 길어질 수 있음                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 최종 일관성 처리 전략

```java
// 전략 1: Read Your Writes
// 방금 쓴 데이터를 바로 읽을 수 있도록 보장

@Service
@RequiredArgsConstructor
public class OrderService {

    private final CommandBus commandBus;
    private final QueryBus queryBus;
    private final EventStore eventStore;

    public OrderView placeOrderAndReturn(PlaceOrderCommand command) {
        // 1. Command 실행
        String orderId = commandBus.dispatch(command);

        // 2. 방금 생성한 주문은 Event Store에서 직접 읽기 (Inline Projection)
        List<DomainEvent> events = eventStore.getEvents(orderId);
        return projectToView(events);  // 이벤트로부터 직접 View 생성
    }

    private OrderView projectToView(List<DomainEvent> events) {
        // 이벤트를 순회하며 View 구성
        OrderView.OrderViewBuilder builder = OrderView.builder();

        for (DomainEvent event : events) {
            if (event instanceof OrderPlacedEvent e) {
                builder.orderId(e.getAggregateId())
                       .customerId(e.getCustomerId())
                       .status("PLACED")
                       .totalAmount(e.getTotalAmount());
            } else if (event instanceof OrderConfirmedEvent e) {
                builder.status("CONFIRMED");
            }
            // ...
        }

        return builder.build();
    }
}

// 전략 2: Polling with Version Check
// 버전 체크를 통해 일관성 확인

@Service
public class OrderQueryService {

    public OrderView waitForConsistency(String orderId, long expectedVersion,
                                       Duration timeout) {
        Instant deadline = Instant.now().plus(timeout);

        while (Instant.now().isBefore(deadline)) {
            OrderView view = queryBus.query(new GetOrderQuery(orderId));

            if (view != null && view.getVersion() >= expectedVersion) {
                return view;  // 일관성 확보
            }

            try {
                Thread.sleep(100);  // 100ms 대기 후 재시도
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new ServiceException("Interrupted while waiting for consistency");
            }
        }

        throw new ConsistencyTimeoutException(
            "Could not achieve consistency within " + timeout
        );
    }
}

// 전략 3: Subscription 기반 알림
// 클라이언트가 업데이트 완료를 구독

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<CreateOrderResponse> createOrder(
            @RequestBody PlaceOrderRequest request) {

        String orderId = commandBus.dispatch(request.toCommand());

        // 클라이언트에게 orderId와 함께 SSE/WebSocket 구독 정보 반환
        return ResponseEntity
            .accepted()
            .header("X-Subscribe-To", "/api/orders/" + orderId + "/events")
            .body(new CreateOrderResponse(orderId, "PROCESSING"));
    }

    @GetMapping("/{orderId}/events")
    public SseEmitter subscribeToOrderEvents(@PathVariable String orderId) {
        SseEmitter emitter = new SseEmitter(30000L);  // 30초 타임아웃

        // 이벤트 발생 시 클라이언트에게 푸시
        eventSubscriber.subscribe(orderId, event -> {
            try {
                emitter.send(SseEmitter.event()
                    .name(event.getEventType())
                    .data(event));

                if (event instanceof OrderConfirmedEvent) {
                    emitter.complete();  // 확정되면 구독 종료
                }
            } catch (IOException e) {
                emitter.completeWithError(e);
            }
        });

        return emitter;
    }
}
```

### 비즈니스와 최종 일관성 논의

```java
// UI/UX 레벨에서 최종 일관성 처리

// 1. Optimistic UI Update
// 프론트엔드에서 성공을 가정하고 UI 먼저 업데이트
/*
Frontend:
1. 주문 버튼 클릭
2. UI에 "주문 완료" 표시 (낙관적)
3. API 호출
4. 성공하면 유지, 실패하면 롤백 표시
*/

// 2. Loading State with Polling
@Component
public class OrderStatusPoller {

    @Async
    public CompletableFuture<OrderStatus> pollUntilReady(
            String orderId, Duration timeout) {

        return CompletableFuture.supplyAsync(() -> {
            Instant deadline = Instant.now().plus(timeout);

            while (Instant.now().isBefore(deadline)) {
                OrderView view = queryBus.query(new GetOrderQuery(orderId));
                if (view != null && view.getStatus() != "PROCESSING") {
                    return OrderStatus.valueOf(view.getStatus());
                }
                sleepQuietly(500);
            }

            return OrderStatus.TIMEOUT;
        });
    }
}

// 3. 비즈니스 관점의 질문
/*
 도메인 전문가와 논의할 질문:

 Q: "주문 후 바로 주문 목록에서 새 주문이 안 보이면 어떤 문제가 있을까요?"
 A: "사용자가 혼란스러울 수 있지만, 1-2초 내에 보인다면 괜찮아요"

 Q: "만약 5초 정도 걸린다면요?"
 A: "그건 좀 길어요. '처리 중' 메시지라도 보여줘야 해요"

 Q: "재고 감소도 같은 방식인데, 동시 주문 시 재고 초과 판매 가능성은요?"
 A: "그건 안 돼요! 재고는 실시간으로 정확해야 해요"

 → 최종 일관성이 허용되는 영역과 강한 일관성이 필요한 영역 구분
*/
```

---

## 실무 구현 예제

### 전체 프로젝트 구조

```
src/main/java/com/example/order/
├── command/                          # Command Side
│   ├── domain/
│   │   ├── Order.java               # Event Sourced Aggregate
│   │   ├── OrderId.java
│   │   └── events/
│   │       ├── OrderPlacedEvent.java
│   │       ├── OrderConfirmedEvent.java
│   │       └── OrderCancelledEvent.java
│   ├── handler/
│   │   └── OrderCommandHandler.java
│   └── port/
│       └── EventStore.java
│
├── query/                            # Query Side
│   ├── model/
│   │   ├── OrderView.java
│   │   ├── OrderListView.java
│   │   └── OrderSearchDocument.java
│   ├── handler/
│   │   └── OrderQueryHandler.java
│   ├── projection/
│   │   ├── OrderDetailsProjection.java
│   │   └── OrderSearchProjection.java
│   └── repository/
│       ├── OrderViewRepository.java
│       └── OrderSearchRepository.java
│
├── infrastructure/
│   ├── eventstore/
│   │   ├── JpaEventStore.java
│   │   ├── EventEntity.java
│   │   └── EventSerializer.java
│   ├── messaging/
│   │   ├── KafkaEventPublisher.java
│   │   └── KafkaEventConsumer.java
│   └── persistence/
│       ├── OrderViewJpaRepository.java
│       └── OrderViewEntity.java
│
└── api/
    ├── OrderCommandController.java
    └── OrderQueryController.java
```

### 통합 예제

```java
// ===== API Layer =====

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderCommandController {

    private final OrderCommandHandler commandHandler;

    @PostMapping
    public ResponseEntity<CreateOrderResponse> placeOrder(
            @Valid @RequestBody PlaceOrderRequest request) {

        PlaceOrderCommand command = new PlaceOrderCommand(
            request.getCustomerId(),
            request.getItems(),
            request.getShippingAddress()
        );

        String orderId = commandHandler.handle(command);

        return ResponseEntity
            .accepted()  // 202 Accepted (비동기 처리)
            .body(new CreateOrderResponse(orderId, "PROCESSING"));
    }

    @PostMapping("/{orderId}/confirm")
    public ResponseEntity<Void> confirmOrder(@PathVariable String orderId) {
        commandHandler.handle(new ConfirmOrderCommand(orderId));
        return ResponseEntity.accepted().build();
    }
}

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderQueryController {

    private final OrderQueryHandler queryHandler;

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderView> getOrder(@PathVariable String orderId) {
        return queryHandler.handle(new GetOrderQuery(orderId))
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/search")
    public ResponseEntity<Page<OrderListView>> searchOrders(
            @RequestParam(required = false) String q,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {

        Page<OrderListView> result = queryHandler.handle(
            new SearchOrdersQuery(q, page, size)
        );
        return ResponseEntity.ok(result);
    }
}

// ===== Command Handler =====

@Component
@RequiredArgsConstructor
@Transactional
public class OrderCommandHandler {

    private final EventStore eventStore;
    private final DomainEventPublisher eventPublisher;

    public String handle(PlaceOrderCommand command) {
        // Aggregate 생성 (이벤트 발생)
        Order order = Order.place(
            new CustomerId(command.getCustomerId()),
            command.getItems(),
            command.getShippingAddress()
        );

        // 이벤트 저장
        eventStore.appendEvents(
            order.getId().getValue(),
            order.getUncommittedEvents()
        );

        // 이벤트 발행 (비동기)
        order.getUncommittedEvents()
            .forEach(eventPublisher::publish);

        return order.getId().getValue();
    }

    public void handle(ConfirmOrderCommand command) {
        // 이벤트로부터 Aggregate 복원
        Order order = eventStore.load(command.getOrderId(), Order.class);

        // 도메인 로직 실행
        order.confirm();

        // 이벤트 저장 (낙관적 락)
        eventStore.appendEvents(
            order.getId().getValue(),
            order.getUncommittedEvents(),
            order.getVersion()
        );

        // 이벤트 발행
        order.getUncommittedEvents()
            .forEach(eventPublisher::publish);
    }
}

// ===== Projection =====

@Component
@RequiredArgsConstructor
@Slf4j
public class OrderDetailsProjection {

    private final OrderViewRepository repository;

    @EventHandler
    @Transactional
    public void on(OrderPlacedEvent event) {
        log.info("Projecting OrderPlacedEvent: {}", event.getAggregateId());

        OrderViewEntity view = OrderViewEntity.builder()
            .orderId(event.getAggregateId())
            .customerId(event.getCustomerId())
            .status("PLACED")
            .totalAmount(event.getTotalAmount())
            .createdAt(event.getOccurredAt())
            .version(event.getVersion())
            .build();

        repository.save(view);
    }

    @EventHandler
    @Transactional
    public void on(OrderConfirmedEvent event) {
        log.info("Projecting OrderConfirmedEvent: {}", event.getAggregateId());

        repository.findById(event.getAggregateId())
            .ifPresent(view -> {
                view.setStatus("CONFIRMED");
                view.setConfirmedAt(event.getConfirmedAt());
                view.setVersion(event.getVersion());
                repository.save(view);
            });
    }
}

// ===== Event Publishing (Kafka) =====

@Component
@RequiredArgsConstructor
public class KafkaEventPublisher implements DomainEventPublisher {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Override
    @Async
    public void publish(DomainEvent event) {
        try {
            String payload = objectMapper.writeValueAsString(event);

            kafkaTemplate.send(
                "order-events",
                event.getAggregateId(),
                payload
            ).addCallback(
                result -> log.debug("Event published: {}", event.getEventType()),
                ex -> log.error("Failed to publish event", ex)
            );
        } catch (JsonProcessingException e) {
            throw new EventPublishException("Failed to serialize event", e);
        }
    }
}

@Component
@RequiredArgsConstructor
public class KafkaEventConsumer {

    private final List<Object> eventHandlers;
    private final ObjectMapper objectMapper;

    @KafkaListener(topics = "order-events", groupId = "order-projection")
    public void consume(String message) {
        DomainEvent event = deserialize(message);

        eventHandlers.stream()
            .filter(handler -> hasMatchingHandler(handler, event.getClass()))
            .forEach(handler -> invokeHandler(handler, event));
    }
}
```

---

## 참고 자료

- [CQRS Pattern - Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Event Sourcing Pattern - Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Projections and Read Models - Event-Driven.io](https://event-driven.io/en/projections_and_read_models_in_event_driven_architecture/)
- [Live Projections - Kurrent.io](https://www.kurrent.io/blog/live-projections-for-read-models-with-event-sourcing-and-cqrs)
- [Eventual Consistency - CQRS.com](https://www.cqrs.com/event-driven-architecture/eventual-consistency/)
- [Dispelling Eventual Consistency FUD - Axoniq](https://www.axoniq.io/blog/dispelling-the-eventual-consistency-fud-when-using-event-sourcing)
- Greg Young, "CQRS and Event Sourcing"
- Vaughn Vernon, "Implementing Domain-Driven Design", Chapter 4

---

*마지막 업데이트: 2025년 1월*

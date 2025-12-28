# 이벤트 드리븐 아키텍처

## 목차
1. [이벤트 소싱 패턴](#1-이벤트-소싱-패턴)
2. [CQRS와 이벤트 소싱](#2-cqrs와-이벤트-소싱)
3. [Saga 패턴](#3-saga-패턴)
4. [Outbox 패턴](#4-outbox-패턴)
5. [이벤트 스토어](#5-이벤트-스토어)
---

## 1. 이벤트 소싱 패턴

### 1.1 이벤트 소싱이란?

이벤트 소싱(Event Sourcing)은 애플리케이션 상태를 현재 상태로 저장하는 대신, **상태 변경을 일으킨 모든 이벤트의 시퀀스**로 저장하는 패턴이다.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     전통적인 CRUD vs 이벤트 소싱                              │
├───────────────────────────────────┬─────────────────────────────────────────┤
│          전통적인 CRUD             │           이벤트 소싱                   │
├───────────────────────────────────┼─────────────────────────────────────────┤
│                                   │                                         │
│  Orders 테이블                     │  Events 테이블                          │
│  ┌────────────────────────┐       │  ┌────────────────────────────────┐    │
│  │ id: 1                  │       │  │ OrderCreated    {id:1, amt:100}│    │
│  │ status: CANCELLED      │       │  │ OrderConfirmed  {id:1}         │    │
│  │ amount: 80             │       │  │ OrderUpdated    {id:1, amt:80} │    │
│  │ updated_at: 2024-01-05 │       │  │ OrderShipped    {id:1}         │    │
│  └────────────────────────┘       │  │ OrderCancelled  {id:1}         │    │
│                                   │  └────────────────────────────────┘    │
│  ※ 현재 상태만 알 수 있음          │  ※ 전체 히스토리를 알 수 있음            │
│  ※ 왜 취소되었는지 알 수 없음       │  ※ 언제든 특정 시점 상태 복원 가능       │
└───────────────────────────────────┴─────────────────────────────────────────┘
```

### 1.2 핵심 개념

**Event (이벤트)**: 과거에 발생한 사실. 불변(Immutable)이며 절대 삭제하지 않음.

```java
// 이벤트 정의
public sealed interface OrderEvent {

    @Value
    class OrderCreated implements OrderEvent {
        String orderId;
        String customerId;
        List<OrderItem> items;
        BigDecimal totalAmount;
        Instant occurredAt;
    }

    @Value
    class OrderConfirmed implements OrderEvent {
        String orderId;
        Instant occurredAt;
    }

    @Value
    class OrderShipped implements OrderEvent {
        String orderId;
        String trackingNumber;
        Instant occurredAt;
    }

    @Value
    class OrderCancelled implements OrderEvent {
        String orderId;
        String reason;
        Instant occurredAt;
    }
}
```

**Aggregate (집합체)**: 이벤트를 발생시키고 적용하여 상태를 관리하는 도메인 객체.

```java
public class Order {
    private String orderId;
    private OrderStatus status;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private List<OrderEvent> uncommittedEvents = new ArrayList<>();

    // 이벤트로부터 상태 복원 (재생)
    public static Order fromEvents(List<OrderEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }

    // 커맨드 처리 - 이벤트 생성
    public void create(String orderId, String customerId, List<OrderItem> items) {
        if (this.orderId != null) {
            throw new IllegalStateException("Order already exists");
        }

        BigDecimal total = calculateTotal(items);
        OrderCreated event = new OrderCreated(orderId, customerId, items, total, Instant.now());

        apply(event);
        uncommittedEvents.add(event);
    }

    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Can only confirm pending orders");
        }

        OrderConfirmed event = new OrderConfirmed(orderId, Instant.now());
        apply(event);
        uncommittedEvents.add(event);
    }

    public void cancel(String reason) {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new IllegalStateException("Cannot cancel shipped or delivered orders");
        }

        OrderCancelled event = new OrderCancelled(orderId, reason, Instant.now());
        apply(event);
        uncommittedEvents.add(event);
    }

    // 이벤트 적용 - 상태 변경
    private void apply(OrderEvent event) {
        switch (event) {
            case OrderCreated e -> {
                this.orderId = e.getOrderId();
                this.items = e.getItems();
                this.totalAmount = e.getTotalAmount();
                this.status = OrderStatus.PENDING;
            }
            case OrderConfirmed e -> this.status = OrderStatus.CONFIRMED;
            case OrderShipped e -> this.status = OrderStatus.SHIPPED;
            case OrderCancelled e -> this.status = OrderStatus.CANCELLED;
        }
    }

    public List<OrderEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void markEventsAsCommitted() {
        uncommittedEvents.clear();
    }
}
```

### 1.3 이벤트 저장과 로드

```java
public interface EventStore {
    void saveEvents(String aggregateId, List<Event> events, int expectedVersion);
    List<Event> getEvents(String aggregateId);
    List<Event> getEvents(String aggregateId, int fromVersion);
}

@Repository
public class JpaEventStore implements EventStore {

    @PersistenceContext
    private EntityManager em;

    @Override
    @Transactional
    public void saveEvents(String aggregateId, List<Event> events, int expectedVersion) {
        // 낙관적 동시성 제어
        Integer currentVersion = getCurrentVersion(aggregateId);
        if (currentVersion != null && currentVersion != expectedVersion) {
            throw new ConcurrencyException("Aggregate has been modified");
        }

        int version = expectedVersion;
        for (Event event : events) {
            EventEntity entity = new EventEntity();
            entity.setAggregateId(aggregateId);
            entity.setVersion(++version);
            entity.setEventType(event.getClass().getName());
            entity.setPayload(serialize(event));
            entity.setOccurredAt(event.getOccurredAt());

            em.persist(entity);
        }
    }

    @Override
    public List<Event> getEvents(String aggregateId) {
        List<EventEntity> entities = em.createQuery(
            "SELECT e FROM EventEntity e WHERE e.aggregateId = :aggregateId ORDER BY e.version",
            EventEntity.class
        )
        .setParameter("aggregateId", aggregateId)
        .getResultList();

        return entities.stream()
            .map(e -> deserialize(e.getPayload(), e.getEventType()))
            .collect(Collectors.toList());
    }
}
```

### 1.4 스냅샷 (Snapshot)

이벤트가 많아지면 Aggregate 재구성 시간이 길어진다. 스냅샷은 특정 시점의 상태를 저장하여 성능을 최적화한다.

```java
public class SnapshotRepository {

    private static final int SNAPSHOT_THRESHOLD = 100;

    public Order loadAggregate(String orderId) {
        // 1. 스냅샷 조회
        Optional<Snapshot> snapshot = snapshotStore.getLatest(orderId);

        Order order;
        int fromVersion;

        if (snapshot.isPresent()) {
            // 스냅샷이 있으면 스냅샷에서 복원
            order = deserialize(snapshot.get().getData());
            fromVersion = snapshot.get().getVersion();
        } else {
            order = new Order();
            fromVersion = 0;
        }

        // 2. 스냅샷 이후의 이벤트만 조회하여 적용
        List<Event> events = eventStore.getEvents(orderId, fromVersion);
        events.forEach(order::apply);

        // 3. 필요시 새 스냅샷 생성
        if (events.size() >= SNAPSHOT_THRESHOLD) {
            saveSnapshot(order);
        }

        return order;
    }

    private void saveSnapshot(Order order) {
        Snapshot snapshot = new Snapshot();
        snapshot.setAggregateId(order.getOrderId());
        snapshot.setVersion(order.getVersion());
        snapshot.setData(serialize(order));
        snapshot.setCreatedAt(Instant.now());

        snapshotStore.save(snapshot);
    }
}
```

### 1.5 이벤트 소싱의 장단점

| 장점 | 단점 |
|-----|------|
| 완전한 감사 로그 (Audit Trail) | 학습 곡선이 높음 |
| 시간 여행 (특정 시점 상태 복원) | 이벤트 스키마 진화 관리 필요 |
| 디버깅 용이 (이벤트 재생) | 저장소 용량 증가 |
| 이벤트 기반 통합 용이 | 복잡한 쿼리 어려움 (CQRS 필요) |
| 장애 복구 용이 | 최종 일관성 모델 필요 |

---

## 2. CQRS와 이벤트 소싱

### 2.1 CQRS란?

CQRS(Command Query Responsibility Segregation)는 **데이터 수정(Command)과 조회(Query)를 분리**하는 패턴이다.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CQRS Architecture                              │
│                                                                             │
│  ┌─────────────┐                                        ┌─────────────┐    │
│  │   Client    │                                        │   Client    │    │
│  └──────┬──────┘                                        └──────┬──────┘    │
│         │ Command                                              │ Query     │
│         ▼                                                      ▼           │
│  ┌─────────────┐                                        ┌─────────────┐    │
│  │  Command    │                                        │   Query     │    │
│  │  Handler    │                                        │  Handler    │    │
│  └──────┬──────┘                                        └──────┬──────┘    │
│         │                                                      │           │
│         ▼                                                      ▼           │
│  ┌─────────────┐     Events      ┌─────────────┐       ┌─────────────┐    │
│  │   Write     │ ──────────────▶ │   Event     │ ────▶ │    Read     │    │
│  │   Model     │                 │   Handler   │       │    Model    │    │
│  │ (Aggregate) │                 │ (Projector) │       │  (View DB)  │    │
│  └─────────────┘                 └─────────────┘       └─────────────┘    │
│         │                                                                  │
│         ▼                                                                  │
│  ┌─────────────┐                                                          │
│  │   Event     │                                                          │
│  │   Store     │                                                          │
│  └─────────────┘                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Command 처리

```java
// Command 정의
public record CreateOrderCommand(
    String customerId,
    List<OrderItemDto> items
) {}

public record ConfirmOrderCommand(String orderId) {}

// Command Handler
@Service
public class OrderCommandHandler {

    private final EventStore eventStore;
    private final EventBus eventBus;

    @Transactional
    public String handle(CreateOrderCommand command) {
        String orderId = UUID.randomUUID().toString();

        Order order = new Order();
        order.create(orderId, command.customerId(), toOrderItems(command.items()));

        // 이벤트 저장
        eventStore.saveEvents(orderId, order.getUncommittedEvents(), 0);

        // 이벤트 발행
        order.getUncommittedEvents().forEach(eventBus::publish);
        order.markEventsAsCommitted();

        return orderId;
    }

    @Transactional
    public void handle(ConfirmOrderCommand command) {
        // Aggregate 로드
        List<Event> events = eventStore.getEvents(command.orderId());
        Order order = Order.fromEvents(events);

        // 비즈니스 로직 실행
        order.confirm();

        // 이벤트 저장 및 발행
        eventStore.saveEvents(command.orderId(), order.getUncommittedEvents(), events.size());
        order.getUncommittedEvents().forEach(eventBus::publish);
        order.markEventsAsCommitted();
    }
}
```

### 2.3 Query 처리 (Projection)

```java
// Read Model 엔티티
@Entity
@Table(name = "order_summary")
public class OrderSummaryView {
    @Id
    private String orderId;
    private String customerId;
    private String customerName;
    private BigDecimal totalAmount;
    private String status;
    private int itemCount;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Projector (Event Handler)
@Component
public class OrderSummaryProjector {

    private final OrderSummaryViewRepository repository;
    private final CustomerClient customerClient;

    @EventHandler
    @Transactional
    public void on(OrderCreated event) {
        CustomerInfo customer = customerClient.getCustomer(event.getCustomerId());

        OrderSummaryView view = new OrderSummaryView();
        view.setOrderId(event.getOrderId());
        view.setCustomerId(event.getCustomerId());
        view.setCustomerName(customer.getName());
        view.setTotalAmount(event.getTotalAmount());
        view.setStatus("PENDING");
        view.setItemCount(event.getItems().size());
        view.setCreatedAt(event.getOccurredAt());
        view.setUpdatedAt(event.getOccurredAt());

        repository.save(view);
    }

    @EventHandler
    @Transactional
    public void on(OrderConfirmed event) {
        OrderSummaryView view = repository.findById(event.getOrderId())
            .orElseThrow(() -> new NotFoundException("Order not found"));

        view.setStatus("CONFIRMED");
        view.setUpdatedAt(event.getOccurredAt());

        repository.save(view);
    }

    @EventHandler
    @Transactional
    public void on(OrderCancelled event) {
        OrderSummaryView view = repository.findById(event.getOrderId())
            .orElseThrow(() -> new NotFoundException("Order not found"));

        view.setStatus("CANCELLED");
        view.setUpdatedAt(event.getOccurredAt());

        repository.save(view);
    }
}

// Query Handler
@Service
public class OrderQueryHandler {

    private final OrderSummaryViewRepository repository;

    public OrderSummaryView getOrder(String orderId) {
        return repository.findById(orderId)
            .orElseThrow(() -> new NotFoundException("Order not found"));
    }

    public Page<OrderSummaryView> getOrdersByCustomer(String customerId, Pageable pageable) {
        return repository.findByCustomerId(customerId, pageable);
    }

    public List<OrderSummaryView> getPendingOrders() {
        return repository.findByStatus("PENDING");
    }
}
```

### 2.4 다중 Projection

하나의 이벤트 스트림에서 여러 Read Model을 생성할 수 있다.

```java
// 주문 상세 Projection
@Component
public class OrderDetailProjector {

    @EventHandler
    public void on(OrderCreated event) {
        // 주문 상세 정보 저장
    }
}

// 일별 매출 통계 Projection
@Component
public class DailySalesProjector {

    @EventHandler
    public void on(OrderCreated event) {
        String date = event.getOccurredAt().toLocalDate().toString();
        dailySalesRepository.incrementSales(date, event.getTotalAmount());
        dailySalesRepository.incrementOrderCount(date);
    }

    @EventHandler
    public void on(OrderCancelled event) {
        // 환불 금액 차감
    }
}

// 고객별 주문 통계 Projection
@Component
public class CustomerOrderStatsProjector {

    @EventHandler
    public void on(OrderCreated event) {
        customerStatsRepository.incrementOrderCount(event.getCustomerId());
        customerStatsRepository.addTotalSpent(event.getCustomerId(), event.getTotalAmount());
    }
}
```

### 2.5 Projection 재구축

Projection 로직 변경 시 이벤트를 재생하여 Read Model을 재구축할 수 있다.

```java
@Service
public class ProjectionRebuilder {

    private final EventStore eventStore;
    private final List<Projector> projectors;

    public void rebuildProjection(String projectionName) {
        // 1. 기존 Projection 데이터 삭제
        dropProjectionData(projectionName);

        // 2. 모든 이벤트 조회
        try (Stream<Event> eventStream = eventStore.streamAllEvents()) {
            // 3. 이벤트 재생
            eventStream.forEach(event -> {
                projectors.stream()
                    .filter(p -> p.getName().equals(projectionName))
                    .forEach(p -> p.handle(event));
            });
        }

        // 4. Projection 상태 업데이트
        updateProjectionStatus(projectionName, "COMPLETED");
    }
}
```

---

## 3. Saga 패턴

### 3.1 Saga 패턴이란?

Saga 패턴은 마이크로서비스 간 **분산 트랜잭션을 관리**하는 패턴이다. 각 서비스의 로컬 트랜잭션을 순차적으로 실행하고, 실패 시 보상 트랜잭션(Compensating Transaction)을 실행하여 일관성을 유지한다.

```
주문 생성 Saga:

성공 케이스:
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Order  │───▶│ Payment │───▶│  Stock  │───▶│Delivery │
│ Created │    │ Charged │    │Reserved │    │Scheduled│
└─────────┘    └─────────┘    └─────────┘    └─────────┘

실패 케이스 (재고 부족):
┌─────────┐    ┌─────────┐    ┌─────────┐
│  Order  │───▶│ Payment │───▶│  Stock  │ ✗ (재고 부족)
│ Created │    │ Charged │    │ Failed  │
└─────────┘    └─────────┘    └─────────┘
      │              │
      ▼              ▼
┌─────────┐    ┌─────────┐
│  Order  │◀───│ Payment │
│Cancelled│    │ Refunded│    ← 보상 트랜잭션
└─────────┘    └─────────┘
```

### 3.2 Choreography 방식

각 서비스가 이벤트를 발행하고 구독하여 **분산된 방식**으로 Saga를 진행한다.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Choreography-based Saga                                  │
│                                                                             │
│  ┌─────────┐                                                               │
│  │  Order  │───OrderCreated──────────────────────────────────────┐         │
│  │ Service │◀──PaymentFailed──┐                                  │         │
│  │         │◀──StockReservationFailed──┐                         │         │
│  └─────────┘                  │        │                         ▼         │
│                               │        │                   ┌─────────┐     │
│                               │        │                   │ Payment │     │
│                               │        │    ┌──────────────│ Service │     │
│                               │        │    │              └─────────┘     │
│                               │        │    │ PaymentCompleted             │
│                               │        │    ▼                              │
│                               │   ┌─────────┐                              │
│                               │   │  Stock  │                              │
│                               └───│ Service │                              │
│                                   └─────────┘                              │
│                                        │ StockReserved                     │
│                                        ▼                                   │
│                                   ┌─────────┐                              │
│                                   │Delivery │                              │
│                                   │ Service │                              │
│                                   └─────────┘                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```java
// Order Service
@Component
public class OrderSagaChoreography {

    @EventHandler
    public void on(PaymentCompleted event) {
        // 다음 단계는 Stock Service가 PaymentCompleted를 듣고 처리
        log.info("Payment completed for order: {}", event.getOrderId());
    }

    @EventHandler
    @Transactional
    public void on(PaymentFailed event) {
        // 보상: 주문 취소
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.cancel("Payment failed: " + event.getReason());
        orderRepository.save(order);
    }

    @EventHandler
    @Transactional
    public void on(StockReservationFailed event) {
        // 보상: 결제 환불 요청 이벤트 발행
        eventBus.publish(new RefundRequested(event.getOrderId(), event.getAmount()));
    }
}

// Payment Service
@Component
public class PaymentSagaHandler {

    @EventHandler
    @Transactional
    public void on(OrderCreated event) {
        try {
            Payment payment = paymentService.charge(
                event.getCustomerId(),
                event.getTotalAmount()
            );
            eventBus.publish(new PaymentCompleted(event.getOrderId(), payment.getId()));
        } catch (PaymentException e) {
            eventBus.publish(new PaymentFailed(event.getOrderId(), e.getMessage()));
        }
    }

    @EventHandler
    @Transactional
    public void on(RefundRequested event) {
        paymentService.refund(event.getOrderId());
        eventBus.publish(new PaymentRefunded(event.getOrderId()));
    }
}

// Stock Service
@Component
public class StockSagaHandler {

    @EventHandler
    @Transactional
    public void on(PaymentCompleted event) {
        try {
            stockService.reserve(event.getOrderId());
            eventBus.publish(new StockReserved(event.getOrderId()));
        } catch (InsufficientStockException e) {
            eventBus.publish(new StockReservationFailed(
                event.getOrderId(),
                event.getAmount(),
                e.getMessage()
            ));
        }
    }
}
```

### 3.3 Orchestration 방식

중앙의 **Orchestrator(조정자)**가 Saga의 흐름을 제어한다.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Orchestration-based Saga                                │
│                                                                             │
│                          ┌────────────────┐                                │
│                          │     Saga       │                                │
│               ┌─────────▶│  Orchestrator  │◀─────────┐                     │
│               │          └───────┬────────┘          │                     │
│               │                  │                   │                     │
│    Response   │     Command      │      Command      │   Response          │
│               │                  ▼                   │                     │
│    ┌──────────┴──┐         ┌─────────────┐    ┌─────┴──────┐              │
│    │   Order     │         │  Payment    │    │   Stock    │              │
│    │  Service    │         │  Service    │    │  Service   │              │
│    └─────────────┘         └─────────────┘    └────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```java
// Saga State
public enum OrderSagaState {
    STARTED,
    PAYMENT_PENDING,
    PAYMENT_COMPLETED,
    STOCK_PENDING,
    STOCK_RESERVED,
    DELIVERY_PENDING,
    COMPLETED,
    COMPENSATING,
    FAILED
}

// Saga 엔티티
@Entity
public class OrderSaga {
    @Id
    private String sagaId;
    private String orderId;

    @Enumerated(EnumType.STRING)
    private OrderSagaState state;

    private String paymentId;
    private String reservationId;
    private String deliveryId;
    private String failureReason;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Saga Orchestrator
@Component
public class OrderSagaOrchestrator {

    private final OrderSagaRepository sagaRepository;
    private final PaymentService paymentService;
    private final StockService stockService;
    private final DeliveryService deliveryService;
    private final OrderService orderService;

    @Transactional
    public void startSaga(String orderId) {
        OrderSaga saga = new OrderSaga();
        saga.setSagaId(UUID.randomUUID().toString());
        saga.setOrderId(orderId);
        saga.setState(OrderSagaState.STARTED);
        sagaRepository.save(saga);

        processPayment(saga);
    }

    private void processPayment(OrderSaga saga) {
        saga.setState(OrderSagaState.PAYMENT_PENDING);
        sagaRepository.save(saga);

        try {
            PaymentResult result = paymentService.charge(saga.getOrderId());
            saga.setPaymentId(result.getPaymentId());
            saga.setState(OrderSagaState.PAYMENT_COMPLETED);
            sagaRepository.save(saga);

            reserveStock(saga);

        } catch (PaymentException e) {
            handleFailure(saga, "Payment failed: " + e.getMessage());
        }
    }

    private void reserveStock(OrderSaga saga) {
        saga.setState(OrderSagaState.STOCK_PENDING);
        sagaRepository.save(saga);

        try {
            ReservationResult result = stockService.reserve(saga.getOrderId());
            saga.setReservationId(result.getReservationId());
            saga.setState(OrderSagaState.STOCK_RESERVED);
            sagaRepository.save(saga);

            scheduleDelivery(saga);

        } catch (StockException e) {
            handleFailure(saga, "Stock reservation failed: " + e.getMessage());
        }
    }

    private void scheduleDelivery(OrderSaga saga) {
        saga.setState(OrderSagaState.DELIVERY_PENDING);
        sagaRepository.save(saga);

        try {
            DeliveryResult result = deliveryService.schedule(saga.getOrderId());
            saga.setDeliveryId(result.getDeliveryId());
            saga.setState(OrderSagaState.COMPLETED);
            sagaRepository.save(saga);

            // 주문 상태 업데이트
            orderService.markAsProcessed(saga.getOrderId());

        } catch (DeliveryException e) {
            handleFailure(saga, "Delivery scheduling failed: " + e.getMessage());
        }
    }

    // 보상 트랜잭션
    private void handleFailure(OrderSaga saga, String reason) {
        saga.setFailureReason(reason);
        saga.setState(OrderSagaState.COMPENSATING);
        sagaRepository.save(saga);

        // 역순으로 보상 실행
        if (saga.getReservationId() != null) {
            stockService.cancelReservation(saga.getReservationId());
        }

        if (saga.getPaymentId() != null) {
            paymentService.refund(saga.getPaymentId());
        }

        orderService.cancel(saga.getOrderId(), reason);

        saga.setState(OrderSagaState.FAILED);
        sagaRepository.save(saga);
    }
}
```

### 3.4 Choreography vs Orchestration

| 구분 | Choreography | Orchestration |
|-----|-------------|---------------|
| **결합도** | 느슨함 (서비스 간 직접 의존 없음) | 중앙 집중 (Orchestrator에 의존) |
| **복잡도** | 이벤트 흐름 추적 어려움 | 흐름이 명확함 |
| **유연성** | 높음 (서비스 독립적 변경) | 중간 (Orchestrator 수정 필요) |
| **장애 지점** | 분산됨 | Orchestrator가 SPOF |
| **디버깅** | 어려움 | 상대적으로 쉬움 |
| **적합한 경우** | 단순한 워크플로우, 3-4개 이하 서비스 | 복잡한 워크플로우, 많은 서비스 |

### 3.5 Saga 구현 프레임워크

**Axon Framework 예제:**

```java
@Saga
public class OrderManagementSaga {

    @Autowired
    private transient CommandGateway commandGateway;

    private String orderId;
    private String paymentId;

    @StartSaga
    @SagaEventHandler(associationProperty = "orderId")
    public void on(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();

        // 결제 명령 전송
        commandGateway.send(new ProcessPaymentCommand(
            UUID.randomUUID().toString(),
            event.getOrderId(),
            event.getTotalAmount()
        ));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(PaymentProcessedEvent event) {
        this.paymentId = event.getPaymentId();

        // 재고 예약 명령 전송
        commandGateway.send(new ReserveStockCommand(
            event.getOrderId()
        ));
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(StockReservedEvent event) {
        // 배송 예약 명령 전송
        commandGateway.send(new ScheduleDeliveryCommand(
            event.getOrderId()
        ));
    }

    @SagaEventHandler(associationProperty = "orderId")
    @EndSaga
    public void on(DeliveryScheduledEvent event) {
        // Saga 완료
        commandGateway.send(new CompleteOrderCommand(orderId));
    }

    // 보상 처리
    @SagaEventHandler(associationProperty = "orderId")
    public void on(PaymentFailedEvent event) {
        commandGateway.send(new CancelOrderCommand(orderId, event.getReason()));
        SagaLifecycle.end();
    }

    @SagaEventHandler(associationProperty = "orderId")
    public void on(StockReservationFailedEvent event) {
        // 결제 환불
        commandGateway.send(new RefundPaymentCommand(paymentId));
        commandGateway.send(new CancelOrderCommand(orderId, event.getReason()));
        SagaLifecycle.end();
    }
}
```

---

## 4. Outbox 패턴

### 4.1 문제 상황

마이크로서비스에서 데이터베이스 업데이트와 메시지 발행을 원자적으로 처리해야 하는 상황에서, 둘 중 하나만 성공하면 데이터 불일치가 발생한다.

```
문제 시나리오:

1. DB 업데이트 성공, 메시지 발행 실패
   → 다른 서비스는 변경 사항을 모름

2. DB 업데이트 실패, 메시지 발행 성공
   → 다른 서비스는 존재하지 않는 데이터 참조

┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Service   │      │  Database   │      │   Message   │
│             │      │             │      │   Broker    │
└──────┬──────┘      └──────┬──────┘      └──────┬──────┘
       │                    │                    │
       │  1. UPDATE order   │                    │
       │───────────────────▶│                    │
       │                    │                    │
       │    OK (committed)  │                    │
       │◀───────────────────│                    │
       │                    │                    │
       │  2. Publish Event  │                    │
       │─────────────────────────────────────────┤ ✗ 네트워크 오류
       │                    │                    │
       │        ※ 데이터 불일치 발생!             │
       │                    │                    │
```

### 4.2 Outbox 패턴 해결책

로컬 데이터베이스의 Outbox 테이블에 메시지를 저장하고, 별도의 프로세스가 이를 메시지 브로커로 발행한다.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Outbox Pattern                                      │
│                                                                             │
│  ┌─────────────┐                               ┌─────────────────────────┐ │
│  │   Service   │                               │        Database         │ │
│  │             │                               │  ┌─────────────────┐    │ │
│  │             │──── 1. 트랜잭션 시작 ──────────▶│  │   orders        │    │ │
│  │             │                               │  │  ┌───────────┐  │    │ │
│  │             │──── 2. 비즈니스 데이터 저장 ────▶│  │  │ order row │  │    │ │
│  │             │                               │  │  └───────────┘  │    │ │
│  │             │                               │  └─────────────────┘    │ │
│  │             │                               │                         │ │
│  │             │──── 3. Outbox에 이벤트 저장 ───▶│  ┌─────────────────┐    │ │
│  │             │                               │  │   outbox        │    │ │
│  │             │                               │  │  ┌───────────┐  │    │ │
│  │             │──── 4. 트랜잭션 커밋 ──────────▶│  │  │event row  │  │    │ │
│  │             │                               │  │  └───────────┘  │    │ │
│  └─────────────┘                               │  └─────────────────┘    │ │
│                                                └───────────┬─────────────┘ │
│                                                            │               │
│  ┌─────────────────────────────────────────────────────────┘               │
│  │                                                                         │
│  │  ┌─────────────────┐                         ┌─────────────────┐       │
│  │  │ Message Relay   │                         │ Message Broker  │       │
│  │  │ (Polling/CDC)   │──── 5. 이벤트 발행 ────▶│  (Kafka, etc.)  │       │
│  │  │                 │                         │                 │       │
│  │  │                 │──── 6. Outbox 업데이트 ─▶│                 │       │
│  │  └─────────────────┘     (처리 완료 표시)     └─────────────────┘       │
│  │                                                                         │
└──┴─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 Outbox 테이블 구조

```sql
CREATE TABLE outbox (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_type  VARCHAR(255) NOT NULL,   -- 예: "Order"
    aggregate_id    VARCHAR(255) NOT NULL,   -- 예: "order-123"
    event_type      VARCHAR(255) NOT NULL,   -- 예: "OrderCreated"
    payload         JSONB NOT NULL,          -- 이벤트 데이터
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    processed_at    TIMESTAMP,               -- 발행 완료 시간
    INDEX idx_outbox_unprocessed (processed_at) WHERE processed_at IS NULL
);
```

### 4.4 Outbox 패턴 구현

```java
// Outbox 엔티티
@Entity
@Table(name = "outbox")
public class OutboxMessage {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String aggregateType;
    private String aggregateId;
    private String eventType;

    @Type(JsonBinaryType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> payload;

    private Instant createdAt;
    private Instant processedAt;
}

// 비즈니스 서비스
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;

    @Transactional  // 단일 트랜잭션!
    public Order createOrder(CreateOrderRequest request) {
        // 1. 비즈니스 로직 실행
        Order order = new Order(request);
        orderRepository.save(order);

        // 2. Outbox에 이벤트 저장 (같은 트랜잭션)
        OutboxMessage message = new OutboxMessage();
        message.setAggregateType("Order");
        message.setAggregateId(order.getId());
        message.setEventType("OrderCreated");
        message.setPayload(Map.of(
            "orderId", order.getId(),
            "customerId", order.getCustomerId(),
            "totalAmount", order.getTotalAmount(),
            "items", order.getItems()
        ));
        message.setCreatedAt(Instant.now());

        outboxRepository.save(message);

        return order;
    }
}
```

### 4.5 Polling Publisher (폴링 방식)

```java
@Component
public class OutboxPollingPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Scheduled(fixedDelay = 100)  // 100ms 간격
    @Transactional
    public void publishPendingMessages() {
        List<OutboxMessage> messages = outboxRepository
            .findTop100ByProcessedAtIsNullOrderByCreatedAt();

        for (OutboxMessage message : messages) {
            try {
                String topic = message.getAggregateType().toLowerCase() + "-events";
                String payload = objectMapper.writeValueAsString(message.getPayload());

                kafkaTemplate.send(topic, message.getAggregateId(), payload).get();

                // 처리 완료 표시
                message.setProcessedAt(Instant.now());
                outboxRepository.save(message);

            } catch (Exception e) {
                log.error("Failed to publish message: {}", message.getId(), e);
                // 재시도를 위해 processedAt을 null로 유지
            }
        }
    }

    // 오래된 처리 완료 메시지 정리
    @Scheduled(cron = "0 0 2 * * *")  // 매일 새벽 2시
    @Transactional
    public void cleanupProcessedMessages() {
        Instant threshold = Instant.now().minus(7, ChronoUnit.DAYS);
        outboxRepository.deleteByProcessedAtBefore(threshold);
    }
}
```

### 4.6 CDC (Change Data Capture) 방식 - Debezium

Debezium을 사용하여 Outbox 테이블의 변경을 감지하고 자동으로 Kafka로 발행한다.

```json
{
    "name": "outbox-connector",
    "config": {
        "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
        "database.hostname": "postgres",
        "database.port": "5432",
        "database.user": "postgres",
        "database.password": "postgres",
        "database.dbname": "orders",
        "database.server.name": "orders",
        "table.include.list": "public.outbox",

        "transforms": "outbox",
        "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
        "transforms.outbox.table.fields.additional.placement": "event_type:header:eventType",
        "transforms.outbox.route.by.field": "aggregate_type",
        "transforms.outbox.route.topic.replacement": "${routedByValue}-events",

        "tombstones.on.delete": "false"
    }
}
```

**Debezium Outbox Event Router:**
- `aggregate_type`: 토픽 이름 결정 (예: "Order" → "order-events")
- `aggregate_id`: Kafka 메시지 키
- `payload`: Kafka 메시지 값
- `event_type`: Kafka 헤더

### 4.7 Outbox 패턴 장단점

| 장점 | 단점 |
|-----|------|
| 데이터베이스와 메시지 발행의 원자성 보장 | 추가적인 테이블 관리 필요 |
| At-least-once 전달 보장 | 메시지 순서 보장이 복잡할 수 있음 |
| 기존 인프라 활용 (별도 2PC 불필요) | 폴링 방식은 지연 발생 가능 |
| 재시도 및 장애 복구 용이 | Consumer 측 멱등성 처리 필요 |

---

## 5. 이벤트 스토어

### 5.1 이벤트 스토어란?

이벤트 스토어는 이벤트 소싱 패턴에서 이벤트를 저장하고 조회하기 위한 특화된 데이터베이스이다.

```
이벤트 스토어 핵심 요구사항:
1. 이벤트 append-only 저장
2. 집합체(Aggregate) 단위로 이벤트 조회
3. 전체 이벤트 스트림 순회 (Projection 재구축용)
4. 낙관적 동시성 제어
5. 이벤트 구독 (Pub/Sub)
```

### 5.2 이벤트 스토어 옵션

| 옵션 | 특징 | 적합한 경우 |
|-----|------|-----------|
| **EventStoreDB** | Greg Young의 Event Sourcing 전용 DB | 이벤트 소싱 전문, 고성능 필요 |
| **Axon Server** | Axon Framework와 통합 | Axon 에코시스템 사용 시 |
| **PostgreSQL** | RDBMS 활용 | 간단한 구현, 기존 인프라 활용 |
| **Kafka** | 로그 기반 스토리지 | 대용량 이벤트, 스트리밍 |
| **MongoDB** | Document 기반 | 유연한 스키마 필요 시 |

### 5.3 PostgreSQL 기반 이벤트 스토어

```sql
-- 이벤트 테이블
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(255) NOT NULL,
    version         INT NOT NULL,
    event_type      VARCHAR(255) NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB,
    occurred_at     TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    UNIQUE (aggregate_id, version)
);

-- 인덱스
CREATE INDEX idx_events_aggregate ON events (aggregate_id, version);
CREATE INDEX idx_events_type ON events (event_type);
CREATE INDEX idx_events_occurred_at ON events (occurred_at);

-- 스냅샷 테이블
CREATE TABLE snapshots (
    aggregate_id    UUID PRIMARY KEY,
    aggregate_type  VARCHAR(255) NOT NULL,
    version         INT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

```java
@Repository
public class PostgresEventStore implements EventStore {

    private final JdbcTemplate jdbcTemplate;
    private final ObjectMapper objectMapper;

    @Override
    @Transactional
    public void saveEvents(String aggregateId, List<Event> events, int expectedVersion) {
        // 낙관적 동시성 제어
        Integer currentVersion = jdbcTemplate.queryForObject(
            "SELECT MAX(version) FROM events WHERE aggregate_id = ?",
            Integer.class,
            UUID.fromString(aggregateId)
        );

        if (currentVersion != null && currentVersion != expectedVersion) {
            throw new ConcurrencyException(
                String.format("Expected version %d but found %d", expectedVersion, currentVersion)
            );
        }

        int version = expectedVersion;
        for (Event event : events) {
            jdbcTemplate.update(
                """
                INSERT INTO events (aggregate_id, aggregate_type, version, event_type, payload, metadata, occurred_at)
                VALUES (?, ?, ?, ?, ?::jsonb, ?::jsonb, ?)
                """,
                UUID.fromString(aggregateId),
                event.getAggregateType(),
                ++version,
                event.getClass().getSimpleName(),
                objectMapper.writeValueAsString(event),
                objectMapper.writeValueAsString(event.getMetadata()),
                event.getOccurredAt()
            );
        }
    }

    @Override
    public List<Event> getEvents(String aggregateId) {
        return jdbcTemplate.query(
            "SELECT * FROM events WHERE aggregate_id = ? ORDER BY version",
            this::mapRowToEvent,
            UUID.fromString(aggregateId)
        );
    }

    @Override
    public List<Event> getEvents(String aggregateId, int fromVersion) {
        return jdbcTemplate.query(
            "SELECT * FROM events WHERE aggregate_id = ? AND version > ? ORDER BY version",
            this::mapRowToEvent,
            UUID.fromString(aggregateId),
            fromVersion
        );
    }

    // 전체 이벤트 스트림 (Projection 재구축용)
    public Stream<Event> streamAllEvents() {
        return jdbcTemplate.queryForStream(
            "SELECT * FROM events ORDER BY id",
            this::mapRowToEvent
        );
    }

    // 특정 이벤트 타입만 조회
    public List<Event> getEventsByType(String eventType, Instant from, Instant to) {
        return jdbcTemplate.query(
            "SELECT * FROM events WHERE event_type = ? AND occurred_at BETWEEN ? AND ? ORDER BY id",
            this::mapRowToEvent,
            eventType, from, to
        );
    }
}
```

### 5.4 EventStoreDB 사용

```java
// EventStoreDB 클라이언트 설정
EventStoreDBClient client = EventStoreDBClient.create(
    EventStoreDBConnectionString.parseOrThrow("esdb://localhost:2113?tls=false")
);

// 이벤트 저장
public void saveEvents(String streamName, List<Event> events, long expectedRevision) {
    List<EventData> eventDataList = events.stream()
        .map(event -> EventData.builderAsJson(
            event.getClass().getSimpleName(),
            objectMapper.writeValueAsBytes(event)
        ).build())
        .toList();

    AppendToStreamOptions options = AppendToStreamOptions.get()
        .expectedRevision(expectedRevision);

    client.appendToStream(streamName, options, eventDataList.iterator())
        .get();
}

// 이벤트 읽기
public List<Event> readEvents(String streamName) {
    ReadStreamOptions options = ReadStreamOptions.get()
        .forwards()
        .fromStart();

    ReadResult result = client.readStream(streamName, options).get();

    return result.getEvents().stream()
        .map(resolvedEvent -> {
            RecordedEvent recorded = resolvedEvent.getOriginalEvent();
            String eventType = recorded.getEventType();
            byte[] data = recorded.getEventData();
            return deserialize(eventType, data);
        })
        .toList();
}

// Catch-up Subscription (실시간 이벤트 구독)
public void subscribeToAll(Consumer<Event> handler) {
    SubscribeToAllOptions options = SubscribeToAllOptions.get()
        .fromStart();

    client.subscribeToAll(new SubscriptionListener() {
        @Override
        public void onEvent(Subscription subscription, ResolvedEvent event) {
            if (!event.getEvent().getEventType().startsWith("$")) {
                Event domainEvent = deserialize(
                    event.getEvent().getEventType(),
                    event.getEvent().getEventData()
                );
                handler.accept(domainEvent);
            }
        }

        @Override
        public void onError(Subscription subscription, Throwable throwable) {
            log.error("Subscription error", throwable);
        }
    }, options);
}
```

### 5.5 Axon Server 사용

```java
// Axon Framework 설정
@Configuration
public class AxonConfig {

    @Bean
    public EventStore eventStore(EventStorageEngine eventStorageEngine) {
        return EmbeddedEventStore.builder()
            .storageEngine(eventStorageEngine)
            .build();
    }

    // Axon Server 연결 시 자동 설정됨
    // 별도 설정 없이 axon-server-connector 의존성만 추가하면 됨
}

// Aggregate
@Aggregate
public class Order {

    @AggregateIdentifier
    private String orderId;
    private OrderStatus status;
    private List<OrderItem> items;

    @CommandHandler
    public Order(CreateOrderCommand command) {
        AggregateLifecycle.apply(new OrderCreatedEvent(
            command.getOrderId(),
            command.getCustomerId(),
            command.getItems()
        ));
    }

    @EventSourcingHandler
    public void on(OrderCreatedEvent event) {
        this.orderId = event.getOrderId();
        this.items = event.getItems();
        this.status = OrderStatus.PENDING;
    }

    @CommandHandler
    public void handle(ConfirmOrderCommand command) {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("Order cannot be confirmed");
        }
        AggregateLifecycle.apply(new OrderConfirmedEvent(this.orderId));
    }

    @EventSourcingHandler
    public void on(OrderConfirmedEvent event) {
        this.status = OrderStatus.CONFIRMED;
    }
}
```

### 5.6 이벤트 스키마 진화

이벤트는 불변이므로 스키마 변경 시 Upcaster를 사용한다.

```java
// 기존 이벤트 (v1)
public class OrderCreatedEventV1 {
    private String orderId;
    private String customerId;
    private BigDecimal amount;  // 단일 금액
}

// 새로운 이벤트 (v2)
public class OrderCreatedEvent {
    private String orderId;
    private String customerId;
    private List<OrderItem> items;  // 아이템 목록으로 변경
    private BigDecimal totalAmount;
}

// Upcaster 구현
public class OrderCreatedEventUpcaster extends SingleEventUpcaster {

    @Override
    protected boolean canUpcast(IntermediateEventRepresentation intermediateRepresentation) {
        return intermediateRepresentation.getType().getName().equals("OrderCreatedEvent")
            && intermediateRepresentation.getType().getRevision() == null;  // v1
    }

    @Override
    protected IntermediateEventRepresentation doUpcast(
            IntermediateEventRepresentation intermediateRepresentation) {

        return intermediateRepresentation.upcastPayload(
            new SimpleSerializedType("OrderCreatedEvent", "2.0"),
            JsonNode.class,
            oldEvent -> {
                ObjectNode newEvent = objectMapper.createObjectNode();
                newEvent.put("orderId", oldEvent.get("orderId").asText());
                newEvent.put("customerId", oldEvent.get("customerId").asText());

                // amount를 items로 변환
                ArrayNode items = objectMapper.createArrayNode();
                ObjectNode item = objectMapper.createObjectNode();
                item.put("name", "Legacy Item");
                item.put("quantity", 1);
                item.put("price", oldEvent.get("amount").asDouble());
                items.add(item);
                newEvent.set("items", items);

                newEvent.put("totalAmount", oldEvent.get("amount").asDouble());

                return newEvent;
            }
        );
    }
}

// Upcaster 등록
@Bean
public EventUpcasterChain eventUpcasterChain() {
    return new EventUpcasterChain(
        new OrderCreatedEventUpcaster()
    );
}
```

---

## 참고 자료

- [Microservices.io - Event Sourcing](https://microservices.io/patterns/data/event-sourcing.html)
- [Microservices.io - CQRS](https://microservices.io/patterns/data/cqrs.html)
- [Microservices.io - Saga](https://microservices.io/patterns/data/saga.html)
- [Microservices.io - Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [EventStoreDB Documentation](https://developers.eventstore.com/)
- [Axon Framework Reference](https://docs.axoniq.io/)
- [Debezium Outbox Event Router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
- [Saga Pattern - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)

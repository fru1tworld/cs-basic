# CQRS와 이벤트 소싱

읽기와 쓰기가 같은 모델을 사용하면 복잡한 변경 규칙과 조회 요구를 한 구조로 처리해야 한다. CQRS(Command Query Responsibility Segregation)는 두 역할을 분리하고, Event Sourcing은 상태가 바뀐 과정을 이벤트로 저장한다.

이 글의 결합 예제에서는 Command를 Aggregate가 처리해 이벤트를 만들고 Event Store에 저장한다. Projection은 그 이벤트로 조회용 Read Model을 만들며, Query Handler는 DB나 캐시에 준비된 이 모델을 읽는다.

## 이벤트 소싱 패턴

- Event (이벤트): 과거에 발생한 사실. 불변(Immutable)이며 절대 삭제하지 않음.

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

이 이벤트를 만들고 적용해 상태를 관리하는 도메인 객체가 Aggregate(집합체)다. 아래 `Order`는 명령을 처리할 때 규칙을 검사한 뒤 이벤트를 만들며, 저장된 이벤트를 다시 읽을 때도 같은 `apply` 메서드로 상태를 복원한다.

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

### 이벤트 소싱이란?

- 이벤트 소싱(Event Sourcing)은 애플리케이션 상태를 현재 상태로 저장하는 대신, 상태 변경을 일으킨 모든 이벤트의 시퀀스로 저장하는 패턴임.

- 전통적인 CRUD vs 이벤트 소싱
- 전통적인 CRUD, 이벤트 소싱
- Orders 테이블, Events 테이블
- id: 1, OrderCreated, {id:1, amt:100}
- status: CANCELLED, OrderConfirmed, {id:1}
- amount: 80, OrderUpdated, {id:1, amt:80}
- updated_at: 2024-01-05, OrderShipped, {id:1}
- OrderCancelled, {id:1}
- 현재 상태만 알 수 있음,  전체 히스토리를 알 수 있음
- 왜 취소되었는지 알 수 없음,  언제든 특정 시점 상태 복원 가능

### 이벤트 저장과 로드

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

### 스냅샷 (Snapshot)

이벤트가 쌓이면 Aggregate를 복원할 때 재생할 양도 늘어난다. 스냅샷에 특정 시점의 상태를 저장해 두면 그 이후 이벤트만 적용할 수 있다. 아래 예제는 최신 스냅샷을 먼저 읽고, 이후 이벤트가 100개 이상이면 새 스냅샷을 만든다.

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

### 이벤트 소싱의 장단점

- 장점: 완전한 감사 로그 (Audit Trail):
  - 단점: 학습 곡선이 높음
- 장점: 시간 여행 (특정 시점 상태 복원):
  - 단점: 이벤트 스키마 진화 관리 필요
- 장점: 디버깅 용이 (이벤트 재생):
  - 단점: 저장소 용량 증가
- 장점: 이벤트 기반 통합 용이:
  - 단점: 복잡한 쿼리 어려움 (CQRS 필요)
- 장점: 장애 복구 용이:
  - 단점: 최종 일관성 모델 필요

## CQRS와 이벤트 소싱

### CQRS란?

CQRS는 데이터 수정(Command)과 조회(Query)를 분리하는 패턴이다. 아래 예제의 Command Handler는 Aggregate를 통해 상태를 변경하고 이벤트를 저장한다. Projector가 이벤트를 받아 View DB를 갱신하면 Query Handler는 이 읽기 전용 모델을 조회한다.

### Command 처리

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

### Query 처리 (Projection)

조회할 때마다 이벤트 전체를 재생하지 않도록 주문 요약을 별도 모델로 만든다. 아래 Projector는 주문 생성, 확정, 취소 이벤트에 맞춰 `OrderSummaryView`를 갱신하고, Query Handler는 고객별 주문이나 결제 대기 주문을 이 모델에서 조회한다.

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

### 다중 Projection

같은 이벤트라도 조회 목적에 따라 다른 모델로 만들 수 있다. 아래 예제는 주문 생성 이벤트를 주문 상세, 일별 매출, 고객별 주문 통계에 각각 반영한다.

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

### Projection 재구축

조회 요구가 바뀌어 Projection 로직을 수정했다면, 저장된 이벤트를 새 로직으로 다시 처리해 Read Model을 재구축할 수 있다. 아래 코드는 기존 Projection 데이터를 비운 뒤 이벤트 스트림을 순회하고, 완료 상태를 기록하는 흐름이다.

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

## 이벤트 스토어

### 이벤트 스토어란?

- 이벤트 스토어는 이벤트 소싱 패턴에서 이벤트를 저장하고 조회하기 위한 특화된 데이터베이스임.

- 이벤트 스토어 핵심 요구사항:
- 이벤트 append-only 저장
- 집합체(Aggregate) 단위로 이벤트 조회
- 전체 이벤트 스트림 순회 (Projection 재구축용)
- 낙관적 동시성 제어
- 이벤트 구독 (Pub/Sub)

### 이벤트 스토어 옵션

- 옵션: EventStoreDB:
  - 특징: Greg Young의 Event Sourcing 전용 DB
  - 적합한 경우: 이벤트 소싱 전문, 고성능 필요
- 옵션: Axon Server:
  - 특징: Axon Framework와 통합
  - 적합한 경우: Axon 에코시스템 사용 시
- 옵션: PostgreSQL:
  - 특징: RDBMS 활용
  - 적합한 경우: 간단한 구현, 기존 인프라 활용
- 옵션: Kafka:
  - 특징: 로그 기반 스토리지
  - 적합한 경우: 대용량 이벤트, 스트리밍
- 옵션: MongoDB:
  - 특징: Document 기반
  - 적합한 경우: 유연한 스키마 필요 시

### PostgreSQL 기반 이벤트 스토어

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

### EventStoreDB 사용

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

### Axon Server 사용

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

### 이벤트 스키마 진화

저장된 이벤트의 스키마와 현재 코드가 기대하는 형식이 달라질 수 있다. Upcaster는 예전 이벤트를 읽을 때 새 형식으로 변환한다. 아래 예제는 v1의 단일 `amount`를 v2의 `items`와 `totalAmount`로 옮긴다.

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

## 왜 CQRS와 Event Sourcing을 사용하는가?

- 사용 이유
- CQRS의 장점:
- 읽기/쓰기 독립 확장 (읽기가 많은 시스템에서 Read DB만 스케일아웃)
- 각각에 최적화된 데이터 모델 사용 가능
- 복잡한 도메인 로직과 간단한 조회 로직 분리
- 보안: 읽기/쓰기에 다른 권한 적용 용이
- Event Sourcing의 장점:
- 완전한 감사 추적 (Audit Trail) - 모든 변경 기록
- 시간 여행 (Time Travel) - 과거 어느 시점 상태 재현
- 디버깅 - 버그 발생 경위 추적
- 이벤트 재생 (Replay) - 새로운 Read Model 생성
- 도메인 이벤트 자연스럽게 도출
- 적합한 사용 사례:
- 금융 시스템 (거래 내역 추적 필수)
- 예약 시스템 (변경 이력 중요)
- 협업 도구 (실시간 동기화)
- 쇼핑몰 장바구니/주문 (상태 변경 추적)
- 부적합한 사용 사례:
- 단순 CRUD 애플리케이션
- 실시간 일관성이 필수인 시스템
- 팀에 경험이 부족한 경우

## CQRS 패턴 상세

### 기본 구조

- CQRS 아키텍처
- Client
- Commands, Queries
- (POST, PUT,, (GET)
- DELETE)
- Command Handler, Query Handler
- Validation, - Simple Read
- Domain Logic, - No Side Effect
- State Change
- Write Model, Read Model
- Normalized, Sync, - Denormalized
- Consistency, ← →, - Optimized
- Domain Entity, - Query-ready
- Write DB, Read DB
- (PostgreSQL), (Elasticsearch
- Redis, etc.)

### CQRS 구현 레벨

- CQRS 구현 복잡도 수준
- Level 1: 단일 DB + 코드 분리
- Command Service, Query Service
- [ Single DB ]
- 장점: 구현 단순, 강한 일관성
- 단점: 스케일링 제한
- Level 2: 단일 DB + Read Replica
- Command Service, Query Service
- [ Write DB ], replication → [ Read Replica ]
- 장점: 읽기 확장성 향상, 구현 간단
- 단점: 복제 지연 (Replication Lag)
- Level 3: 분리된 DB + 동기화
- Command Service, Query Service
- [ Write DB ], Events, [ Read DB ]
- (PostgreSQL), →, (Elasticsearch)
- 장점: 각각 최적화, 독립 스케일링
- 단점: 최종 일관성, 동기화 복잡
- Level 4: Event Sourcing + 다중 Read Model
- Command Handler
- [ Event Store ]
- → Projection A, → [ Read DB 1 ] (상세 조회용)
- → Projection B, → [ Read DB 2 ] (검색용)
- → Projection C, → [ Read DB 3 ] (통계용)
- 장점: 완전한 유연성, 이벤트 재생
- 단점: 높은 복잡도, 러닝 커브

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

## 이벤트 소싱 (Event Sourcing)

- Event Sourcing 개념
- 전통적인 상태 저장:
- Order Table:
- ID, Status, Amount, Updated_At
- O001, SHIPPED, 50000, 2024-01-15, ← 현재 상태만 저장
- 문제: 어떻게 SHIPPED가 됐는지? 이전 상태는?
- Event Sourcing:
- Events Table:
- Aggr ID, Event Type, Data, Occurred_At
- O001, OrderPlaced, {items: ...}, 2024-01-10 10:00
- O001, PaymentReceived, {amount: ..}, 2024-01-10 10:30
- O001, OrderConfirmed, {}, 2024-01-10 10:31
- O001, OrderShipped, {carrier:..}, 2024-01-15 09:00
- 현재 상태 = 모든 이벤트를 순서대로 적용한 결과
- 초기, → OrderPlaced, → OrderConfirm, → OrderShipped
- 상태, (이벤트1), (이벤트2), (이벤트3)
- 상태: PLACED, 상태: CONFIRMED, 상태: SHIPPED

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

- 이벤트가 많이 쌓이면 Aggregate 복원이 느려질 수 있음. Snapshot으로 최적화함.

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

## 프로젝션과 Read Model

### Projection 구현

Projection은 이벤트를 조회 목적에 맞는 모델로 변환한다. 같은 `OrderPlaced`, `OrderConfirmed`, `OrderShipped` 이벤트도 상세 조회, 요약, 검색, 통계에 필요한 방식으로 각각 반영할 수 있다.

| 목적 | Read Model | 저장소 예시 |
| --- | --- | --- |
| 주문 상세 | `order_details` | PostgreSQL |
| 주문 요약 | `order_summaries` | Redis |
| 검색 | `orders_index` | Elasticsearch |
| 통계 | `order_stats` | ClickHouse |

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

## 최종 일관성

### Eventual Consistency 이해

이벤트 저장과 Read Model 갱신이 비동기로 진행되면 쓰기 성공 직후의 조회에 변경이 아직 보이지 않을 수 있다. 주문 생성 예제의 흐름은 다음과 같다.

1. `t0`: 클라이언트가 주문 생성을 요청한다.
2. `t1`: Command Handler가 Event Store에 이벤트를 저장한다.
3. `t2`: 클라이언트에 `orderId`를 반환한다.
4. `t3`: Projection이 이벤트를 받는다.
5. `t4`: Read Model을 갱신한다.

`t2`부터 `t4` 사이에 조회하면 새 주문이 보이지 않을 수 있다. 이 차이는 보통 밀리초에서 초 단위지만 장애가 나면 더 길어질 수 있으므로, 다음과 같이 조회 방식과 사용자에게 보여 줄 상태를 정해야 한다.

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

## 참고 자료

- [Microservices.io - Event Sourcing](https://microservices.io/patterns/data/event-sourcing.html)
- [Microservices.io - CQRS](https://microservices.io/patterns/data/cqrs.html)
- [Microservices.io - Saga](https://microservices.io/patterns/data/saga.html)
- [Microservices.io - Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [EventStoreDB Documentation](https://developers.eventstore.com/)
- [Axon Framework Reference](https://docs.axoniq.io/)
- [Debezium Outbox Event Router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
- [Saga Pattern - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)

- [CQRS Pattern - Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Event Sourcing Pattern - Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Projections and Read Models - Event-Driven.io](https://event-driven.io/en/projections_and_read_models_in_event_driven_architecture/)
- [Live Projections - Kurrent.io](https://www.kurrent.io/blog/live-projections-for-read-models-with-event-sourcing-and-cqrs)
- [Eventual Consistency - CQRS.com](https://www.cqrs.com/event-driven-architecture/eventual-consistency/)
- [Dispelling Eventual Consistency FUD - Axoniq](https://www.axoniq.io/blog/dispelling-the-eventual-consistency-fud-when-using-event-sourcing)
- Greg Young, "CQRS and Event Sourcing"
- Vaughn Vernon, "Implementing Domain-Driven Design", Chapter 4

- \*마지막 업데이트: 2025년 1월\*

## 관련 학습

- [Saga와 Outbox](02-Saga와-Outbox.md)
- [트랜잭션과 ACID](../../데이터베이스/트랜잭션/01-트랜잭션과-ACID.md)

# Saga와 Outbox

## Saga 패턴

### Saga 패턴이란?

Saga 패턴은 여러 서비스에 걸친 작업을 각 서비스의 로컬 트랜잭션으로 나누어 실행한다. 중간에 실패하면 이미 끝난 작업에 대한 보상 트랜잭션(Compensating Transaction)을 실행해 일관성을 유지한다.

주문 생성 과정을 예로 들면 주문 생성(Order Created), 결제(Payment Charged), 재고 예약(Stock Reserved), 배송 예약(Delivery Scheduled) 순서로 진행한다. 결제까지 끝났는데 재고가 부족하다면 결제를 환불(Payment Refunded)하고 주문을 취소(Order Cancelled)하는 보상 작업이 필요하다.

### Choreography 방식

Choreography 방식에서는 각 서비스가 이벤트를 발행하고 구독하면서 다음 작업으로 이어간다. 주문 서비스의 `OrderCreated`를 결제 서비스가 받아 처리하고, 결제가 끝나면 `PaymentCompleted`를 재고 서비스가 받는다. 이후 `StockReserved`가 배송 서비스로 전달된다.

실패도 이벤트로 전달한다. 아래 예제에서 주문 서비스는 `PaymentFailed`를 받으면 주문을 취소하고, `StockReservationFailed`를 받으면 환불을 요청한다.

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

### Orchestration 방식

Orchestration 방식에서는 중앙의 Orchestrator(조정자)가 각 서비스에 명령을 보내고 응답에 따라 다음 작업을 선택한다. 아래 예제는 Saga의 상태와 결제·재고·배송 식별자를 저장하고, 실패하면 이미 완료한 작업의 역순으로 보상한다.

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

### Choreography vs Orchestration

- 구분: 결합도:
  - Choreography: 느슨함 (서비스 간 직접 의존 없음)
  - Orchestration: 중앙 집중 (Orchestrator에 의존)
- 구분: 복잡도:
  - Choreography: 이벤트 흐름 추적 어려움
  - Orchestration: 흐름이 명확함
- 구분: 유연성:
  - Choreography: 높음 (서비스 독립적 변경)
  - Orchestration: 중간 (Orchestrator 수정 필요)
- 구분: 장애 지점:
  - Choreography: 분산됨
  - Orchestration: Orchestrator가 SPOF
- 구분: 디버깅:
  - Choreography: 어려움
  - Orchestration: 상대적으로 쉬움
- 구분: 적합한 경우:
  - Choreography: 단순한 워크플로우, 3-4개 이하 서비스
  - Orchestration: 복잡한 워크플로우, 많은 서비스

### Saga 구현 프레임워크

- Axon Framework 예제:

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

## Outbox 패턴

### 문제 상황

데이터베이스 변경과 메시지 발행을 별도로 수행하면 둘 중 하나만 성공할 수 있다. 주문을 DB에 커밋한 뒤 네트워크 오류로 이벤트 발행에 실패하면 다른 서비스는 주문이 생성된 사실을 알지 못한다. 반대로 메시지는 발행했는데 DB 변경에 실패하면 다른 서비스가 존재하지 않는 데이터를 참조하게 된다.

### Outbox 패턴 해결책

Outbox 패턴은 발행할 메시지를 비즈니스 데이터와 함께 로컬 데이터베이스에 저장한다. 별도의 Message Relay가 저장된 메시지를 읽어 브로커에 발행하므로, 다음 순서로 처리한다.

1. 트랜잭션을 시작하고 `orders`에 주문을 저장한다.
2. 같은 트랜잭션에서 `outbox`에 이벤트를 저장한 뒤 커밋한다.
3. Message Relay가 폴링이나 CDC로 이벤트를 읽어 Kafka 등의 브로커에 발행한다.
4. 폴링 방식에서는 발행한 메시지에 처리 완료 표시를 남긴다.

### Outbox 테이블 구조

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

### Outbox 패턴 구현

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

### Polling Publisher (폴링 방식)

아래 Publisher는 아직 발행하지 않은 메시지를 최대 100개씩 읽는다. 발행에 성공하면 `processedAt`을 기록하고, 실패하면 값을 비워 두어 다음 폴링에서 다시 처리한다.

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

### CDC (Change Data Capture) 방식 - Debezium

폴링 대신 Debezium으로 Outbox 테이블의 변경을 감지해 Kafka에 발행할 수도 있다. 다음 설정은 PostgreSQL의 `public.outbox`를 대상으로 Outbox Event Router를 적용한다.

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

- Debezium Outbox Event Router:
- `aggregate_type`: 토픽 이름 결정 (예: "Order" → "order-events")
- `aggregate_id`: Kafka 메시지 키
- `payload`: Kafka 메시지 값
- `event_type`: Kafka 헤더

### Outbox 패턴 장단점

- 장점: 데이터베이스와 메시지 발행의 원자성 보장:
  - 단점: 추가적인 테이블 관리 필요
- 장점: At-least-once 전달 보장:
  - 단점: 메시지 순서 보장이 복잡할 수 있음
- 장점: 기존 인프라 활용 (별도 2PC 불필요):
  - 단점: 폴링 방식은 지연 발생 가능
- 장점: 재시도 및 장애 복구 용이:
  - 단점: Consumer 측 멱등성 처리 필요

## 참고 자료

- [Microservices.io - Event Sourcing](https://microservices.io/patterns/data/event-sourcing.html)
- [Microservices.io - CQRS](https://microservices.io/patterns/data/cqrs.html)
- [Microservices.io - Saga](https://microservices.io/patterns/data/saga.html)
- [Microservices.io - Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [EventStoreDB Documentation](https://developers.eventstore.com/)
- [Axon Framework Reference](https://docs.axoniq.io/)
- [Debezium Outbox Event Router](https://debezium.io/documentation/reference/transformations/outbox-event-router.html)
- [Saga Pattern - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html)

## 관련 학습

- [CQRS와 이벤트 소싱](01-CQRS와-이벤트-소싱.md)
- [트랜잭션과 ACID](../../데이터베이스/트랜잭션/01-트랜잭션과-ACID.md)

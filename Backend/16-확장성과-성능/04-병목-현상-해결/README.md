# 병목 현상 해결

## 목차
1. [개요](#개요)
2. [데이터베이스 최적화](#데이터베이스-최적화)
3. [캐싱 전략](#캐싱-전략)
4. [비동기 처리](#비동기-처리)
5. [Connection Pooling](#connection-pooling)
6. [배치 처리](#배치-처리)
7. [실전 적용 가이드](#실전-적용-가이드)
---

## 개요

병목 현상(Bottleneck)은 시스템의 전체 성능을 제한하는 가장 느린 구성 요소입니다. Michael Nygard의 "Release It!"에서 강조하듯이, 병목은 시스템 어디에서나 발생할 수 있으며, 하나를 해결하면 다른 곳에서 나타날 수 있습니다.

> "시스템은 가장 느린 구성 요소만큼만 빠르다. 병목을 해결하면 다음 병목이 드러난다." - Release It!

```
┌─────────────────────────────────────────────────────────────┐
│                    Common Bottlenecks                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Request → [ App Server ] → [ Database ]                   │
│                   ↓              ↓                           │
│              CPU/Memory      Query/Lock                      │
│                   ↓              ↓                           │
│            [ External API ]  [ Cache ]                       │
│                   ↓              ↓                           │
│              Network I/O     Memory                          │
│                                                              │
│   각 구간에서 병목이 발생할 수 있음                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 데이터베이스 최적화

### 1. 쿼리 최적화

```sql
-- 문제: 풀 테이블 스캔
SELECT * FROM orders WHERE DATE(created_at) = '2024-01-01';

-- 해결: 인덱스 활용 가능한 형태로 변경
SELECT * FROM orders
WHERE created_at >= '2024-01-01 00:00:00'
  AND created_at < '2024-01-02 00:00:00';

-- 문제: SELECT *
SELECT * FROM users WHERE id = 1;

-- 해결: 필요한 컬럼만 조회
SELECT id, name, email FROM users WHERE id = 1;

-- 문제: 서브쿼리
SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');

-- 해결: JOIN으로 변경
SELECT o.* FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';
```

```java
/**
 * EXPLAIN 분석을 통한 쿼리 최적화
 */
public class QueryOptimization {

    /**
     * 실행 계획 분석 포인트
     */
    public void analyzeExplain() {
        // EXPLAIN ANALYZE SELECT ...

        /*
        주요 체크 포인트:
        1. type:
           - ALL: 풀 스캔 (나쁨)
           - index: 인덱스 풀 스캔
           - range: 인덱스 범위 스캔 (좋음)
           - ref: 인덱스 참조 (좋음)
           - eq_ref: 유니크 인덱스 참조 (매우 좋음)
           - const: 상수 (가장 좋음)

        2. key: 사용된 인덱스
           - NULL이면 인덱스 미사용

        3. rows: 예상 스캔 행 수
           - 작을수록 좋음

        4. Extra:
           - Using filesort: 정렬 필요 (인덱스로 해결 가능한지 검토)
           - Using temporary: 임시 테이블 사용
           - Using index: 커버링 인덱스 (좋음)
        */
    }
}
```

### 2. 인덱스 최적화

```sql
-- 복합 인덱스 설계
-- 쿼리 패턴: WHERE status = ? AND created_at > ? ORDER BY id

-- 효율적인 인덱스 (왼쪽부터 매칭)
CREATE INDEX idx_orders_status_created ON orders(status, created_at);

-- 커버링 인덱스 (조회 컬럼까지 포함)
CREATE INDEX idx_orders_covering ON orders(status, created_at, id, total_amount);

-- 인덱스 힌트 사용 (MySQL)
SELECT /*+ INDEX(orders idx_orders_status_created) */
    id, total_amount
FROM orders
WHERE status = 'COMPLETED'
  AND created_at > '2024-01-01';
```

```java
/**
 * JPA에서 인덱스 활용
 */
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_status_created", columnList = "status, createdAt"),
    @Index(name = "idx_user_id", columnList = "userId")
})
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @Column(nullable = false)
    private LocalDateTime createdAt;

    private Long userId;

    private BigDecimal totalAmount;
}

/**
 * N+1 문제 해결
 */
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // 문제: N+1 쿼리 발생
    List<Order> findByUserId(Long userId);

    // 해결 1: Fetch Join
    @Query("SELECT o FROM Order o JOIN FETCH o.orderItems WHERE o.userId = :userId")
    List<Order> findByUserIdWithItems(@Param("userId") Long userId);

    // 해결 2: EntityGraph
    @EntityGraph(attributePaths = {"orderItems", "user"})
    List<Order> findWithGraphByUserId(Long userId);

    // 해결 3: Batch Size 설정
    // application.yml: spring.jpa.properties.hibernate.default_batch_fetch_size=100
}

/**
 * QueryDSL을 활용한 최적화된 조회
 */
@Repository
@RequiredArgsConstructor
public class OrderQueryRepository {

    private final JPAQueryFactory queryFactory;

    /**
     * 페이징 + 필터링 최적화
     * - 커버링 인덱스 활용
     * - 서브쿼리로 ID만 먼저 조회
     */
    public Page<OrderDto> findOrdersOptimized(OrderSearchCondition condition, Pageable pageable) {
        // 1. 커버링 인덱스로 ID만 조회
        List<Long> ids = queryFactory
            .select(order.id)
            .from(order)
            .where(
                statusEq(condition.getStatus()),
                createdAtBetween(condition.getStartDate(), condition.getEndDate())
            )
            .orderBy(order.id.desc())
            .offset(pageable.getOffset())
            .limit(pageable.getPageSize())
            .fetch();

        if (ids.isEmpty()) {
            return Page.empty(pageable);
        }

        // 2. ID로 실제 데이터 조회 (IN 쿼리)
        List<OrderDto> content = queryFactory
            .select(new QOrderDto(
                order.id,
                order.status,
                order.totalAmount,
                order.createdAt
            ))
            .from(order)
            .where(order.id.in(ids))
            .orderBy(order.id.desc())
            .fetch();

        // 3. 전체 카운트 (별도 최적화)
        Long total = queryFactory
            .select(order.count())
            .from(order)
            .where(
                statusEq(condition.getStatus()),
                createdAtBetween(condition.getStartDate(), condition.getEndDate())
            )
            .fetchOne();

        return new PageImpl<>(content, pageable, total);
    }
}
```

### 3. Connection Pool 최적화

```java
/**
 * HikariCP 최적화 설정
 */
@Configuration
public class HikariCPConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    public HikariConfig hikariConfig() {
        HikariConfig config = new HikariConfig();

        // 풀 사이즈 공식: connections = (core_count * 2) + effective_spindle_count
        // 일반적으로 CPU 코어 수의 2~4배
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(10);

        // 커넥션 타임아웃 (대기 시간)
        config.setConnectionTimeout(30000);  // 30초

        // 유휴 커넥션 타임아웃
        config.setIdleTimeout(600000);  // 10분

        // 커넥션 최대 수명
        config.setMaxLifetime(1800000);  // 30분

        // 커넥션 유효성 검사
        config.setConnectionTestQuery("SELECT 1");
        config.setValidationTimeout(5000);  // 5초

        // 리크 감지
        config.setLeakDetectionThreshold(60000);  // 60초

        // 성능 최적화
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");

        return config;
    }

    @Bean
    public DataSource dataSource(HikariConfig config) {
        return new HikariDataSource(config);
    }
}

/**
 * 커넥션 풀 모니터링
 */
@Component
@RequiredArgsConstructor
public class ConnectionPoolMonitor {

    private final HikariDataSource dataSource;
    private final MeterRegistry meterRegistry;

    @Scheduled(fixedRate = 10000)
    public void logPoolStats() {
        HikariPoolMXBean pool = dataSource.getHikariPoolMXBean();

        log.info("HikariCP Stats - Active: {}, Idle: {}, Waiting: {}, Total: {}",
            pool.getActiveConnections(),
            pool.getIdleConnections(),
            pool.getThreadsAwaitingConnection(),
            pool.getTotalConnections());

        // 경고 조건
        if (pool.getThreadsAwaitingConnection() > 0) {
            log.warn("Threads waiting for connection: {}", pool.getThreadsAwaitingConnection());
        }

        if (pool.getActiveConnections() >= dataSource.getMaximumPoolSize() * 0.8) {
            log.warn("Connection pool utilization > 80%");
        }
    }
}
```

---

## 캐싱 전략

### 1. 캐시 레이어 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    Cache Layers                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Client                                                     │
│     │                                                        │
│     ▼                                                        │
│   ┌─────────────────────┐                                   │
│   │   CDN / Edge Cache  │  ← 정적 리소스, API 응답          │
│   └──────────┬──────────┘                                   │
│              │                                               │
│              ▼                                               │
│   ┌─────────────────────┐                                   │
│   │   Application Cache │  ← 세션, 인증 토큰                │
│   │   (Redis/Memcached) │                                   │
│   └──────────┬──────────┘                                   │
│              │                                               │
│              ▼                                               │
│   ┌─────────────────────┐                                   │
│   │   Local Cache       │  ← 설정, 자주 조회 데이터         │
│   │   (Caffeine/Guava)  │                                   │
│   └──────────┬──────────┘                                   │
│              │                                               │
│              ▼                                               │
│   ┌─────────────────────┐                                   │
│   │   Database          │                                   │
│   └─────────────────────┘                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. 캐시 패턴 구현

```java
/**
 * 다층 캐시 구현
 */
@Configuration
@EnableCaching
public class CacheConfig {

    /**
     * 로컬 캐시 (Caffeine)
     */
    @Bean
    public CaffeineCacheManager localCacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats());
        return cacheManager;
    }

    /**
     * 분산 캐시 (Redis)
     */
    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(
                new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()));

        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        cacheConfigs.put("products", defaultConfig.entryTtl(Duration.ofHours(1)));
        cacheConfigs.put("users", defaultConfig.entryTtl(Duration.ofMinutes(10)));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .build();
    }
}

/**
 * Cache-Aside 패턴 구현
 */
@Service
@RequiredArgsConstructor
public class ProductCacheService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Product> redisTemplate;
    private final Cache<Long, Product> localCache;

    private static final String CACHE_PREFIX = "product:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(30);

    /**
     * Cache-Aside (Lazy Loading)
     * 1. 캐시 확인
     * 2. 캐시 미스 시 DB 조회
     * 3. 결과를 캐시에 저장
     */
    public Product getProduct(Long productId) {
        // 1. 로컬 캐시 확인
        Product product = localCache.getIfPresent(productId);
        if (product != null) {
            return product;
        }

        // 2. Redis 캐시 확인
        String cacheKey = CACHE_PREFIX + productId;
        product = redisTemplate.opsForValue().get(cacheKey);
        if (product != null) {
            localCache.put(productId, product);  // 로컬 캐시에 저장
            return product;
        }

        // 3. DB 조회
        product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        // 4. 캐시 저장
        redisTemplate.opsForValue().set(cacheKey, product, CACHE_TTL);
        localCache.put(productId, product);

        return product;
    }

    /**
     * Write-Through 패턴
     * DB 쓰기와 캐시 갱신을 동시에
     */
    @Transactional
    public Product updateProduct(Long productId, ProductUpdateRequest request) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        product.update(request);
        productRepository.save(product);

        // 캐시 갱신
        String cacheKey = CACHE_PREFIX + productId;
        redisTemplate.opsForValue().set(cacheKey, product, CACHE_TTL);
        localCache.put(productId, product);

        return product;
    }

    /**
     * Cache Invalidation
     * 데이터 변경 시 캐시 무효화
     */
    @Transactional
    public void deleteProduct(Long productId) {
        productRepository.deleteById(productId);

        // 캐시 삭제
        String cacheKey = CACHE_PREFIX + productId;
        redisTemplate.delete(cacheKey);
        localCache.invalidate(productId);
    }
}

/**
 * 캐시 스탬피드 방지
 */
@Service
public class CacheStampedeProtection {

    private final RedisTemplate<String, Object> redisTemplate;
    private final RedissonClient redissonClient;

    /**
     * 분산 락을 사용한 캐시 스탬피드 방지
     */
    public Product getProductWithLock(Long productId) {
        String cacheKey = "product:" + productId;
        String lockKey = "lock:" + cacheKey;

        // 1. 캐시 확인
        Product product = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (product != null) {
            return product;
        }

        // 2. 분산 락 획득
        RLock lock = redissonClient.getLock(lockKey);
        try {
            if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                try {
                    // Double-check: 다른 스레드가 이미 캐싱했을 수 있음
                    product = (Product) redisTemplate.opsForValue().get(cacheKey);
                    if (product != null) {
                        return product;
                    }

                    // 3. DB 조회 및 캐싱
                    product = productRepository.findById(productId)
                        .orElseThrow(() -> new ProductNotFoundException(productId));
                    redisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(30));
                    return product;
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Lock interrupted", e);
        }

        // 락 획득 실패 시 직접 DB 조회
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    /**
     * 확률적 조기 만료 (Probabilistic Early Expiration)
     * 캐시 스탬피드 방지의 또 다른 방법
     */
    public Product getProductWithProbabilisticExpiry(Long productId) {
        String cacheKey = "product:" + productId;

        CachedProduct cached = (CachedProduct) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            // 확률적으로 조기 재계산
            long delta = cached.getComputeTime();  // 계산에 걸린 시간
            long expiry = cached.getExpiryTime();
            long now = System.currentTimeMillis();
            double beta = 1.0;

            // 만료 시간에 가까울수록 재계산 확률 증가
            double randomValue = Math.random();
            double expiryThreshold = delta * beta * Math.log(randomValue);

            if (now - expiryThreshold < expiry) {
                return cached.getProduct();
            }
        }

        // 재계산
        long start = System.currentTimeMillis();
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        long computeTime = System.currentTimeMillis() - start;

        CachedProduct newCached = new CachedProduct(
            product,
            System.currentTimeMillis() + Duration.ofMinutes(30).toMillis(),
            computeTime
        );
        redisTemplate.opsForValue().set(cacheKey, newCached);

        return product;
    }
}
```

### 3. 캐시 무효화 전략

```java
/**
 * 캐시 무효화 패턴
 */
@Service
@RequiredArgsConstructor
public class CacheInvalidationService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final ApplicationEventPublisher eventPublisher;

    /**
     * 1. 개별 키 무효화
     */
    public void invalidateKey(String key) {
        redisTemplate.delete(key);
    }

    /**
     * 2. 패턴 기반 무효화
     */
    public void invalidateByPattern(String pattern) {
        Set<String> keys = redisTemplate.keys(pattern);
        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }

    /**
     * 3. 태그 기반 무효화
     * 관련된 캐시를 그룹으로 관리
     */
    @Transactional
    public void invalidateByTag(String tag) {
        String tagKey = "cache_tag:" + tag;
        Set<Object> members = redisTemplate.opsForSet().members(tagKey);
        if (members != null && !members.isEmpty()) {
            List<String> keys = members.stream()
                .map(Object::toString)
                .collect(Collectors.toList());
            redisTemplate.delete(keys);
            redisTemplate.delete(tagKey);
        }
    }

    public void cacheWithTag(String key, Object value, String... tags) {
        redisTemplate.opsForValue().set(key, value);
        for (String tag : tags) {
            redisTemplate.opsForSet().add("cache_tag:" + tag, key);
        }
    }

    /**
     * 4. 이벤트 기반 무효화 (분산 환경)
     */
    public void publishInvalidationEvent(String cacheKey) {
        eventPublisher.publishEvent(new CacheInvalidationEvent(cacheKey));
        // Redis Pub/Sub으로 다른 인스턴스에도 전파
        redisTemplate.convertAndSend("cache:invalidation", cacheKey);
    }
}

/**
 * Redis Pub/Sub 리스너로 캐시 무효화 수신
 */
@Component
@RequiredArgsConstructor
public class CacheInvalidationListener implements MessageListener {

    private final Cache<String, Object> localCache;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String cacheKey = new String(message.getBody());
        localCache.invalidate(cacheKey);
        log.info("Local cache invalidated: {}", cacheKey);
    }
}
```

---

## 비동기 처리

### 1. CompletableFuture 활용

```java
/**
 * CompletableFuture를 활용한 비동기 처리
 */
@Service
@RequiredArgsConstructor
public class AsyncOrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final NotificationService notificationService;

    private final ExecutorService executorService;

    /**
     * 병렬 처리로 응답 시간 단축
     * 각 서비스 호출: 100ms
     * 순차 처리: 400ms
     * 병렬 처리: ~100ms
     */
    public OrderResponse getOrderDetails(Long orderId) {
        CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(
            () -> orderRepository.findById(orderId).orElseThrow(),
            executorService
        );

        CompletableFuture<PaymentInfo> paymentFuture = orderFuture.thenApplyAsync(
            order -> paymentService.getPaymentInfo(order.getPaymentId()),
            executorService
        );

        CompletableFuture<DeliveryInfo> deliveryFuture = orderFuture.thenApplyAsync(
            order -> deliveryService.getDeliveryInfo(order.getDeliveryId()),
            executorService
        );

        CompletableFuture<List<ProductInfo>> productsFuture = orderFuture.thenApplyAsync(
            order -> order.getItems().stream()
                .map(item -> productService.getProductInfo(item.getProductId()))
                .collect(Collectors.toList()),
            executorService
        );

        // 모든 결과 조합
        return orderFuture
            .thenCombine(paymentFuture, (order, payment) ->
                new OrderResponseBuilder().order(order).payment(payment))
            .thenCombine(deliveryFuture, (builder, delivery) ->
                builder.delivery(delivery))
            .thenCombine(productsFuture, (builder, products) ->
                builder.products(products).build())
            .exceptionally(ex -> {
                log.error("Error fetching order details", ex);
                throw new OrderFetchException(orderId, ex);
            })
            .join();
    }

    /**
     * 타임아웃 처리
     */
    public OrderResponse getOrderDetailsWithTimeout(Long orderId) {
        try {
            return getOrderDetails(orderId);
        } catch (CompletionException e) {
            if (e.getCause() instanceof TimeoutException) {
                throw new ServiceTimeoutException("Order details fetch timed out");
            }
            throw e;
        }
    }

    /**
     * Fire-and-Forget 패턴
     * 결과를 기다리지 않는 비동기 작업
     */
    @Async
    public void sendOrderNotification(Order order) {
        try {
            notificationService.sendEmail(order.getUserEmail(), "Order Confirmed", ...);
            notificationService.sendSms(order.getUserPhone(), ...);
            notificationService.sendPush(order.getUserId(), ...);
        } catch (Exception e) {
            // 알림 실패는 로깅만 (주문에 영향 없음)
            log.error("Failed to send notification for order: {}", order.getId(), e);
        }
    }
}
```

### 2. 메시지 큐 기반 비동기 처리

```java
/**
 * Kafka를 활용한 비동기 처리
 */
@Service
@RequiredArgsConstructor
public class AsyncOrderProcessingService {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    /**
     * 주문 생성 후 비동기 처리 위임
     */
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 1. 주문 저장 (동기)
        Order order = orderRepository.save(request.toEntity());

        // 2. 후속 작업은 Kafka로 비동기 처리
        OrderEvent event = new OrderEvent(order.getId(), OrderEventType.CREATED);
        kafkaTemplate.send("order-events", order.getId().toString(), event);

        // 3. 빠른 응답 반환
        return order;
    }
}

/**
 * Kafka Consumer - 비동기 작업 처리
 */
@Component
@RequiredArgsConstructor
public class OrderEventConsumer {

    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void handleOrderEvent(OrderEvent event) {
        log.info("Processing order event: {}", event);

        switch (event.getType()) {
            case CREATED:
                handleOrderCreated(event.getOrderId());
                break;
            case PAYMENT_COMPLETED:
                handlePaymentCompleted(event.getOrderId());
                break;
            case CANCELLED:
                handleOrderCancelled(event.getOrderId());
                break;
        }
    }

    private void handleOrderCreated(Long orderId) {
        // 재고 확인 및 차감
        inventoryService.reserveStock(orderId);

        // 결제 요청
        paymentService.requestPayment(orderId);
    }

    private void handlePaymentCompleted(Long orderId) {
        // 주문 상태 업데이트
        orderService.updateStatus(orderId, OrderStatus.PAID);

        // 알림 발송
        notificationService.sendOrderConfirmation(orderId);
    }

    private void handleOrderCancelled(Long orderId) {
        // 재고 복원
        inventoryService.releaseStock(orderId);

        // 환불 처리
        paymentService.refund(orderId);
    }
}

/**
 * 배압(Backpressure) 처리
 */
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent> kafkaListenerContainerFactory(
            ConsumerFactory<String, OrderEvent> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory);

        // 동시 컨슈머 수 제한
        factory.setConcurrency(3);

        // 배치 처리
        factory.setBatchListener(true);

        // 에러 핸들링
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new FixedBackOff(1000L, 3)  // 1초 간격, 3회 재시도
        ));

        return factory;
    }
}
```

### 3. @Async 활용

```java
/**
 * Spring @Async 설정 및 활용
 */
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    @Bean(name = "asyncExecutor")
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async method {} threw exception: {}", method.getName(), ex.getMessage(), ex);
            // 알림 또는 모니터링 시스템에 보고
        };
    }
}

@Service
@RequiredArgsConstructor
public class NotificationAsyncService {

    private final EmailSender emailSender;
    private final SmsSender smsSender;
    private final PushSender pushSender;

    /**
     * 여러 채널로 동시 알림 발송
     */
    @Async("asyncExecutor")
    public CompletableFuture<Void> sendNotifications(User user, NotificationContent content) {
        List<CompletableFuture<Void>> futures = new ArrayList<>();

        if (user.isEmailEnabled()) {
            futures.add(CompletableFuture.runAsync(() ->
                emailSender.send(user.getEmail(), content)));
        }

        if (user.isSmsEnabled()) {
            futures.add(CompletableFuture.runAsync(() ->
                smsSender.send(user.getPhone(), content)));
        }

        if (user.isPushEnabled()) {
            futures.add(CompletableFuture.runAsync(() ->
                pushSender.send(user.getPushToken(), content)));
        }

        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
    }
}
```

---

## Connection Pooling

### 1. HTTP Connection Pool

```java
/**
 * HTTP 클라이언트 Connection Pool 설정
 */
@Configuration
public class HttpClientConfig {

    /**
     * Apache HttpClient Connection Pool
     */
    @Bean
    public CloseableHttpClient httpClient() {
        // Connection Manager 설정
        PoolingHttpClientConnectionManager connectionManager =
            new PoolingHttpClientConnectionManager();

        // 전체 최대 커넥션
        connectionManager.setMaxTotal(200);

        // 호스트당 최대 커넥션
        connectionManager.setDefaultMaxPerRoute(50);

        // 특정 호스트에 대한 설정
        connectionManager.setMaxPerRoute(
            new HttpRoute(new HttpHost("api.example.com", 443)),
            100
        );

        // Keep-Alive 전략
        ConnectionKeepAliveStrategy keepAliveStrategy = (response, context) -> {
            HeaderElementIterator it = new BasicHeaderElementIterator(
                response.headerIterator(HTTP.CONN_KEEP_ALIVE)
            );
            while (it.hasNext()) {
                HeaderElement he = it.nextElement();
                String param = he.getName();
                String value = he.getValue();
                if (value != null && param.equalsIgnoreCase("timeout")) {
                    return Long.parseLong(value) * 1000;
                }
            }
            return 30000; // 기본 30초
        };

        // Request 설정
        RequestConfig requestConfig = RequestConfig.custom()
            .setConnectTimeout(5000)         // 연결 타임아웃
            .setSocketTimeout(30000)         // 소켓 타임아웃
            .setConnectionRequestTimeout(5000) // 풀에서 커넥션 대기 시간
            .build();

        return HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setKeepAliveStrategy(keepAliveStrategy)
            .setDefaultRequestConfig(requestConfig)
            .evictExpiredConnections()       // 만료된 커넥션 자동 제거
            .evictIdleConnections(60, TimeUnit.SECONDS) // 유휴 커넥션 제거
            .build();
    }

    /**
     * RestTemplate with Connection Pool
     */
    @Bean
    public RestTemplate restTemplate(CloseableHttpClient httpClient) {
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory(httpClient);
        return new RestTemplate(factory);
    }
}

/**
 * WebClient Connection Pool (Reactive)
 */
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        // Connection Provider 설정
        ConnectionProvider provider = ConnectionProvider.builder("custom")
            .maxConnections(200)
            .maxIdleTime(Duration.ofSeconds(60))
            .maxLifeTime(Duration.ofMinutes(5))
            .pendingAcquireTimeout(Duration.ofSeconds(30))
            .pendingAcquireMaxCount(500)
            .evictInBackground(Duration.ofSeconds(30))
            .build();

        // HTTP Client 설정
        HttpClient httpClient = HttpClient.create(provider)
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
            .responseTimeout(Duration.ofSeconds(30))
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(30))
                .addHandlerLast(new WriteTimeoutHandler(30)));

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

### 2. Redis Connection Pool

```java
/**
 * Redis Connection Pool 설정 (Lettuce)
 */
@Configuration
public class RedisConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        // 클러스터 설정
        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration(
            Arrays.asList("redis1:6379", "redis2:6379", "redis3:6379")
        );
        clusterConfig.setMaxRedirects(3);

        // Connection Pool 설정
        GenericObjectPoolConfig<Object> poolConfig = new GenericObjectPoolConfig<>();
        poolConfig.setMaxTotal(50);
        poolConfig.setMaxIdle(20);
        poolConfig.setMinIdle(10);
        poolConfig.setMaxWait(Duration.ofSeconds(5));
        poolConfig.setTestOnBorrow(true);
        poolConfig.setTestOnReturn(true);
        poolConfig.setTestWhileIdle(true);

        LettucePoolingClientConfiguration clientConfig = LettucePoolingClientConfiguration.builder()
            .poolConfig(poolConfig)
            .commandTimeout(Duration.ofSeconds(10))
            .shutdownTimeout(Duration.ofSeconds(5))
            .build();

        return new LettuceConnectionFactory(clusterConfig, clientConfig);
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(LettuceConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

---

## 배치 처리

### 1. 배치 쿼리 최적화

```java
/**
 * 대량 데이터 처리를 위한 배치 최적화
 */
@Service
@RequiredArgsConstructor
public class BatchProcessingService {

    private final JdbcTemplate jdbcTemplate;
    private final EntityManager entityManager;

    /**
     * JDBC Batch Insert
     */
    public void batchInsert(List<Order> orders) {
        String sql = "INSERT INTO orders (user_id, total_amount, status, created_at) VALUES (?, ?, ?, ?)";

        jdbcTemplate.batchUpdate(sql, new BatchPreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                Order order = orders.get(i);
                ps.setLong(1, order.getUserId());
                ps.setBigDecimal(2, order.getTotalAmount());
                ps.setString(3, order.getStatus().name());
                ps.setTimestamp(4, Timestamp.valueOf(order.getCreatedAt()));
            }

            @Override
            public int getBatchSize() {
                return orders.size();
            }
        });
    }

    /**
     * JPA Batch Insert (Hibernate 최적화)
     * application.yml:
     *   hibernate.jdbc.batch_size: 50
     *   hibernate.order_inserts: true
     *   hibernate.order_updates: true
     */
    @Transactional
    public void batchInsertWithJpa(List<Order> orders) {
        int batchSize = 50;

        for (int i = 0; i < orders.size(); i++) {
            entityManager.persist(orders.get(i));

            if (i > 0 && i % batchSize == 0) {
                entityManager.flush();
                entityManager.clear();  // 1차 캐시 정리로 메모리 절약
            }
        }

        entityManager.flush();
        entityManager.clear();
    }

    /**
     * Bulk Update
     */
    @Modifying
    @Transactional
    public int bulkUpdateStatus(List<Long> orderIds, OrderStatus newStatus) {
        return entityManager.createQuery(
            "UPDATE Order o SET o.status = :status, o.updatedAt = :now WHERE o.id IN :ids")
            .setParameter("status", newStatus)
            .setParameter("now", LocalDateTime.now())
            .setParameter("ids", orderIds)
            .executeUpdate();
    }
}
```

### 2. Spring Batch 활용

```java
/**
 * Spring Batch 설정
 */
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public Job orderProcessingJob(JobRepository jobRepository, Step processOrdersStep) {
        return new JobBuilder("orderProcessingJob", jobRepository)
            .start(processOrdersStep)
            .build();
    }

    @Bean
    public Step processOrdersStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            ItemReader<Order> reader,
            ItemProcessor<Order, ProcessedOrder> processor,
            ItemWriter<ProcessedOrder> writer) {

        return new StepBuilder("processOrdersStep", jobRepository)
            .<Order, ProcessedOrder>chunk(100, transactionManager)  // 청크 크기
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skipLimit(10)
            .skip(ProcessingException.class)  // 일부 실패 허용
            .retryLimit(3)
            .retry(TransientException.class)  // 재시도
            .build();
    }

    /**
     * 커서 기반 Reader (메모리 효율적)
     */
    @Bean
    public JdbcCursorItemReader<Order> orderReader(DataSource dataSource) {
        return new JdbcCursorItemReaderBuilder<Order>()
            .name("orderReader")
            .dataSource(dataSource)
            .sql("SELECT * FROM orders WHERE status = 'PENDING' ORDER BY id")
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .fetchSize(100)  // DB에서 한 번에 가져올 크기
            .build();
    }

    /**
     * 페이징 기반 Reader (대용량 처리)
     */
    @Bean
    public JdbcPagingItemReader<Order> pagingOrderReader(DataSource dataSource) {
        Map<String, Order> parameterValues = new HashMap<>();
        parameterValues.put("status", "PENDING");

        return new JdbcPagingItemReaderBuilder<Order>()
            .name("pagingOrderReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders")
            .whereClause("WHERE status = :status")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .parameterValues(parameterValues)
            .pageSize(100)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
    }

    /**
     * 병렬 처리 Step
     */
    @Bean
    public Step parallelProcessStep(
            JobRepository jobRepository,
            PlatformTransactionManager transactionManager,
            TaskExecutor taskExecutor) {

        return new StepBuilder("parallelStep", jobRepository)
            .<Order, ProcessedOrder>chunk(100, transactionManager)
            .reader(reader)
            .processor(processor)
            .writer(writer)
            .taskExecutor(taskExecutor)
            .throttleLimit(4)  // 동시 스레드 수 제한
            .build();
    }
}

/**
 * 파티셔닝을 통한 대용량 처리
 */
@Configuration
public class PartitionedBatchConfig {

    @Bean
    public Step masterStep(
            JobRepository jobRepository,
            Step workerStep,
            Partitioner partitioner) {

        return new StepBuilder("masterStep", jobRepository)
            .partitioner("workerStep", partitioner)
            .step(workerStep)
            .gridSize(10)  // 10개 파티션
            .taskExecutor(new SimpleAsyncTaskExecutor())
            .build();
    }

    @Bean
    public Partitioner orderPartitioner(DataSource dataSource) {
        ColumnRangePartitioner partitioner = new ColumnRangePartitioner();
        partitioner.setColumn("id");
        partitioner.setTable("orders");
        partitioner.setDataSource(dataSource);
        return partitioner;
    }
}
```

---

## 실전 적용 가이드

### 병목 해결 체크리스트

```yaml
# 병목 해결 우선순위 체크리스트
bottleneck_resolution:
  level_1_quick_wins:
    - "불필요한 로깅 제거"
    - "N+1 쿼리 수정"
    - "인덱스 추가"
    - "Connection Pool 크기 조정"

  level_2_caching:
    - "자주 조회되는 데이터 캐싱"
    - "세션 캐싱 (Redis)"
    - "HTTP 응답 캐싱"

  level_3_async:
    - "비동기 처리 가능한 작업 분리"
    - "메시지 큐 도입"
    - "이벤트 기반 아키텍처"

  level_4_architecture:
    - "읽기/쓰기 분리"
    - "마이크로서비스 분리"
    - "CQRS 패턴 적용"

  monitoring:
    - "APM 도구 적용"
    - "메트릭 대시보드 구성"
    - "알림 설정"
```

### 성능 개선 전/후 비교 템플릿

```markdown
## 성능 개선 보고서

### 문제 상황
- API 엔드포인트: POST /api/orders
- 평균 응답 시간: 3.5초
- P99 응답 시간: 8초
- 에러율: 2%

### 원인 분석
1. N+1 쿼리: 주문 조회 시 상품 정보 개별 조회
2. 동기 외부 API 호출: 결제 API, 재고 API 순차 호출
3. Connection Pool 부족: 최대 10개, 피크 시 대기 발생

### 개선 사항
1. Fetch Join으로 N+1 해결
2. CompletableFuture로 외부 API 병렬 호출
3. Connection Pool 50개로 증가

### 결과
- 평균 응답 시간: 3.5초 → 0.8초 (77% 개선)
- P99 응답 시간: 8초 → 1.5초 (81% 개선)
- 에러율: 2% → 0.1% (95% 개선)
```

---

## 참고 자료

- [Release It!: Design and Deploy Production-Ready Software](https://pragprog.com/titles/mnee2/release-it-second-edition/) - Michael Nygard
- [HikariCP GitHub - About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [Optimal Connection Pool Size - Vlad Mihalcea](https://vladmihalcea.com/optimal-connection-pool-size/)
- [Spring Batch Documentation](https://docs.spring.io/spring-batch/docs/current/reference/html/)
- [High Performance MySQL](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)
- [Redis Best Practices](https://redis.io/docs/management/optimization/)

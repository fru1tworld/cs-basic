# 서킷 브레이커 패턴과 복원력 패턴

## 목차
1. [개요](#개요)
2. [Circuit Breaker 패턴 상세](#circuit-breaker-패턴-상세)
3. [Resilience4j](#resilience4j)
4. [Hystrix (Deprecated)](#hystrix-deprecated)
5. [Bulkhead 패턴](#bulkhead-패턴)
6. [Retry 패턴](#retry-패턴)
7. [Timeout 패턴](#timeout-패턴)
8. [Rate Limiter 패턴](#rate-limiter-패턴)
9. [실전 적용 가이드](#실전-적용-가이드)
---

## 개요

분산 시스템에서 서비스 간 통신은 필연적으로 실패할 수 있습니다. Michael Nygard의 "Release It!"에서는 이러한 실패에 대응하기 위한 안정성 패턴들을 소개합니다.

> "모든 외부 호출은 잠재적인 장애 지점이다. 방어적으로 코딩하라." - Release It!

```
┌─────────────────────────────────────────────────────────────┐
│                  Stability Patterns                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐│
│   │    Timeout    │   │     Retry     │   │   Circuit     ││
│   │               │   │               │   │   Breaker     ││
│   └───────────────┘   └───────────────┘   └───────────────┘│
│                                                              │
│   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐│
│   │   Bulkhead    │   │  Rate Limiter │   │    Fallback   ││
│   │               │   │               │   │               ││
│   └───────────────┘   └───────────────┘   └───────────────┘│
│                                                              │
│   이 패턴들은 Cascading Failure를 방지하고                  │
│   시스템의 복원력(Resilience)을 높여줍니다.                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Circuit Breaker 패턴 상세

### 서킷 브레이커란?

서킷 브레이커는 전기 회로 차단기에서 영감을 받은 패턴입니다. 외부 서비스 호출 실패가 임계치를 초과하면 회로를 열어(차단) 더 이상의 호출을 막고, 일정 시간 후 다시 시도합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                 Circuit Breaker States                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                    ┌─────────────┐                          │
│                    │   CLOSED    │  ← 정상 상태             │
│                    │  (모든 호출 │     모든 요청 통과       │
│                    │   허용)     │                          │
│                    └──────┬──────┘                          │
│                           │                                  │
│                           │ 실패율 > 임계치                  │
│                           ▼                                  │
│                    ┌─────────────┐                          │
│      타임아웃      │    OPEN     │  ← 차단 상태             │
│      ┌───────────→ │  (모든 호출 │     요청 즉시 실패       │
│      │             │   거부)     │                          │
│      │             └──────┬──────┘                          │
│      │                    │                                  │
│      │                    │ wait_duration 경과               │
│      │                    ▼                                  │
│      │             ┌─────────────┐                          │
│      │             │  HALF_OPEN  │  ← 테스트 상태           │
│      │             │  (제한된    │     일부 요청만 허용     │
│      └─────────────│   호출)     │                          │
│        실패        └──────┬──────┘                          │
│                           │                                  │
│                           │ 성공                             │
│                           ▼                                  │
│                     CLOSED로 전환                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 서킷 브레이커가 필요한 이유

```java
/**
 * 서킷 브레이커 없이 외부 서비스 호출 시 문제
 */
public class WithoutCircuitBreaker {

    /**
     * 문제 1: Cascading Failure
     * 하나의 서비스 장애가 전체 시스템으로 전파
     */
    public OrderResponse createOrder(OrderRequest request) {
        // 결제 서비스 장애 발생
        PaymentResult payment = paymentService.process(request.getPayment());
        // ↑ 여기서 30초 타임아웃 대기

        // 그 동안 스레드 점유
        // 더 많은 요청 도착
        // 스레드 풀 고갈
        // 전체 서비스 다운!

        return new OrderResponse(order, payment);
    }

    /**
     * 문제 2: Resource Exhaustion
     * 실패하는 서비스에 계속 요청 → 리소스 낭비
     */
    public void retryForever() {
        while (true) {
            try {
                externalService.call();  // 계속 실패
                break;
            } catch (Exception e) {
                // 무한 재시도 → CPU, 네트워크 낭비
            }
        }
    }

    /**
     * 문제 3: Slow Recovery
     * 복구된 서비스에 과부하
     */
    public void thunderingHerd() {
        // 서비스 복구 시
        // 밀린 요청이 한꺼번에 몰림
        // 다시 장애 발생!
    }
}
```

### 서킷 브레이커 기본 구현

```java
/**
 * 서킷 브레이커 직접 구현 (개념 이해용)
 */
public class SimpleCircuitBreaker {

    private enum State {
        CLOSED, OPEN, HALF_OPEN
    }

    private State state = State.CLOSED;
    private int failureCount = 0;
    private int successCount = 0;
    private long lastFailureTime = 0;

    private final int failureThreshold;
    private final long resetTimeoutMs;
    private final int halfOpenMaxCalls;

    public SimpleCircuitBreaker(int failureThreshold, long resetTimeoutMs, int halfOpenMaxCalls) {
        this.failureThreshold = failureThreshold;
        this.resetTimeoutMs = resetTimeoutMs;
        this.halfOpenMaxCalls = halfOpenMaxCalls;
    }

    public <T> T execute(Supplier<T> action, Supplier<T> fallback) {
        if (!allowRequest()) {
            return fallback.get();
        }

        try {
            T result = action.get();
            onSuccess();
            return result;
        } catch (Exception e) {
            onFailure();
            return fallback.get();
        }
    }

    private synchronized boolean allowRequest() {
        switch (state) {
            case CLOSED:
                return true;

            case OPEN:
                if (System.currentTimeMillis() - lastFailureTime >= resetTimeoutMs) {
                    state = State.HALF_OPEN;
                    failureCount = 0;
                    successCount = 0;
                    return true;
                }
                return false;

            case HALF_OPEN:
                return successCount + failureCount < halfOpenMaxCalls;

            default:
                return false;
        }
    }

    private synchronized void onSuccess() {
        switch (state) {
            case HALF_OPEN:
                successCount++;
                if (successCount >= halfOpenMaxCalls) {
                    state = State.CLOSED;
                    failureCount = 0;
                }
                break;

            case CLOSED:
                failureCount = 0;  // 성공 시 실패 카운트 리셋
                break;
        }
    }

    private synchronized void onFailure() {
        lastFailureTime = System.currentTimeMillis();

        switch (state) {
            case CLOSED:
                failureCount++;
                if (failureCount >= failureThreshold) {
                    state = State.OPEN;
                }
                break;

            case HALF_OPEN:
                state = State.OPEN;
                break;
        }
    }
}
```

---

## Resilience4j

### Resilience4j 소개

Resilience4j는 Netflix Hystrix에서 영감을 받아 만들어진 경량 내결함성 라이브러리입니다. Java 8+ 함수형 프로그래밍을 기반으로 설계되었습니다.

```xml
<!-- Maven 의존성 -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
    <version>2.2.0</version>
</dependency>
```

### Circuit Breaker 설정

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        # 슬라이딩 윈도우 설정
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        minimum-number-of-calls: 5

        # 실패율 임계치
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 2s

        # 상태 전환 설정
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true

        # 예외 처리
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:
          - com.example.BusinessException

      inventoryService:
        sliding-window-type: TIME_BASED
        sliding-window-size: 60  # 60초
        failure-rate-threshold: 60
        wait-duration-in-open-state: 60s
```

### Resilience4j 사용 예시

```java
/**
 * Resilience4j 어노테이션 기반 사용
 */
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentClient paymentClient;

    /**
     * 서킷 브레이커 + 폴백
     */
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }

    /**
     * 폴백 메서드
     * - 원본 메서드와 같은 파라미터 + Exception
     * - 반환 타입 동일
     */
    private PaymentResult paymentFallback(PaymentRequest request, Exception e) {
        log.warn("Payment circuit breaker fallback triggered", e);

        // 비즈니스 로직에 따른 폴백 처리
        if (e instanceof TimeoutException) {
            // 타임아웃: 나중에 재시도하도록 큐에 저장
            paymentRetryQueue.add(request);
            return PaymentResult.pending(request.getOrderId());
        }

        // 기타 오류: 사용자에게 안내
        return PaymentResult.failed("Payment service temporarily unavailable");
    }

    /**
     * 여러 패턴 조합
     */
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    @Bulkhead(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResult> processPaymentWithResilience(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> paymentClient.process(request));
    }
}

/**
 * 프로그래밍 방식 사용
 */
@Service
public class ResilientPaymentService {

    private final CircuitBreaker circuitBreaker;
    private final Retry retry;
    private final Bulkhead bulkhead;
    private final TimeLimiter timeLimiter;
    private final PaymentClient paymentClient;

    public ResilientPaymentService(
            CircuitBreakerRegistry cbRegistry,
            RetryRegistry retryRegistry,
            BulkheadRegistry bulkheadRegistry,
            TimeLimiterRegistry tlRegistry,
            PaymentClient paymentClient) {

        this.circuitBreaker = cbRegistry.circuitBreaker("paymentService");
        this.retry = retryRegistry.retry("paymentService");
        this.bulkhead = bulkheadRegistry.bulkhead("paymentService");
        this.timeLimiter = tlRegistry.timeLimiter("paymentService");
        this.paymentClient = paymentClient;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        // 데코레이터 체이닝: Bulkhead → TimeLimiter → CircuitBreaker → Retry → 실제 호출
        Supplier<PaymentResult> supplier = () -> paymentClient.process(request);

        Supplier<PaymentResult> decoratedSupplier = Decorators.ofSupplier(supplier)
            .withBulkhead(bulkhead)
            .withCircuitBreaker(circuitBreaker)
            .withRetry(retry)
            .decorate();

        return Try.ofSupplier(decoratedSupplier)
            .recover(throwable -> {
                log.error("All retries failed", throwable);
                return PaymentResult.failed("Service unavailable");
            })
            .get();
    }

    /**
     * 비동기 처리
     */
    public CompletableFuture<PaymentResult> processPaymentAsync(PaymentRequest request) {
        Supplier<CompletableFuture<PaymentResult>> supplier =
            () -> CompletableFuture.supplyAsync(() -> paymentClient.process(request));

        Supplier<CompletableFuture<PaymentResult>> decoratedSupplier =
            CircuitBreaker.decorateSupplier(circuitBreaker, supplier);

        return decoratedSupplier.get()
            .exceptionally(throwable -> {
                log.error("Async payment failed", throwable);
                return PaymentResult.failed("Service unavailable");
            });
    }
}

/**
 * 서킷 브레이커 이벤트 모니터링
 */
@Component
@RequiredArgsConstructor
public class CircuitBreakerEventListener {

    private final CircuitBreakerRegistry circuitBreakerRegistry;
    private final MeterRegistry meterRegistry;

    @PostConstruct
    public void registerEventListeners() {
        circuitBreakerRegistry.getAllCircuitBreakers().forEach(cb -> {
            cb.getEventPublisher()
                .onStateTransition(event -> {
                    log.info("CircuitBreaker {} state changed from {} to {}",
                        event.getCircuitBreakerName(),
                        event.getStateTransition().getFromState(),
                        event.getStateTransition().getToState());

                    // 메트릭 기록
                    meterRegistry.counter("circuit_breaker.state.transition",
                        "name", event.getCircuitBreakerName(),
                        "from", event.getStateTransition().getFromState().name(),
                        "to", event.getStateTransition().getToState().name()
                    ).increment();
                })
                .onSuccess(event ->
                    log.debug("CircuitBreaker {} success", event.getCircuitBreakerName()))
                .onError(event ->
                    log.warn("CircuitBreaker {} error: {}",
                        event.getCircuitBreakerName(),
                        event.getThrowable().getMessage()))
                .onSlowCallRateExceeded(event ->
                    log.warn("CircuitBreaker {} slow call rate exceeded",
                        event.getCircuitBreakerName()));
        });
    }
}
```

---

## Hystrix (Deprecated)

### Hystrix 개요

Netflix Hystrix는 서킷 브레이커 패턴의 선구자였지만, 2018년 유지보수 모드에 들어갔습니다. 현재는 Resilience4j 사용을 권장합니다.

```java
/**
 * Hystrix 사용 예시 (레거시 참고용)
 *
 * @deprecated Resilience4j 사용 권장
 */
@Deprecated
public class HystrixExample {

    /**
     * HystrixCommand 패턴
     */
    public class PaymentCommand extends HystrixCommand<PaymentResult> {

        private final PaymentRequest request;
        private final PaymentClient client;

        public PaymentCommand(PaymentRequest request, PaymentClient client) {
            super(Setter
                .withGroupKey(HystrixCommandGroupKey.Factory.asKey("PaymentService"))
                .andCommandKey(HystrixCommandKey.Factory.asKey("ProcessPayment"))
                .andThreadPoolKey(HystrixThreadPoolKey.Factory.asKey("PaymentPool"))
                .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                    .withCircuitBreakerRequestVolumeThreshold(10)
                    .withCircuitBreakerErrorThresholdPercentage(50)
                    .withCircuitBreakerSleepWindowInMilliseconds(30000)
                    .withExecutionTimeoutInMilliseconds(5000))
                .andThreadPoolPropertiesDefaults(HystrixThreadPoolProperties.Setter()
                    .withCoreSize(10)
                    .withMaxQueueSize(100)));

            this.request = request;
            this.client = client;
        }

        @Override
        protected PaymentResult run() throws Exception {
            return client.process(request);
        }

        @Override
        protected PaymentResult getFallback() {
            return PaymentResult.failed("Payment service unavailable");
        }
    }

    /**
     * 어노테이션 방식 (Spring Cloud Netflix Hystrix)
     */
    @HystrixCommand(
        fallbackMethod = "fallback",
        commandProperties = {
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000")
        }
    )
    public PaymentResult process(PaymentRequest request) {
        return paymentClient.process(request);
    }

    public PaymentResult fallback(PaymentRequest request, Throwable t) {
        return PaymentResult.failed("Service unavailable");
    }
}
```

### Hystrix vs Resilience4j 비교

| 특성 | Hystrix | Resilience4j |
|------|---------|--------------|
| 상태 | Deprecated (2018~) | Active |
| 스레드 모델 | 스레드 풀 격리 기본 | 세마포어 기반 |
| 함수형 | 제한적 | 완전한 함수형 API |
| 의존성 | 무거움 (Archaius 등) | 경량 (Vavr만) |
| 모니터링 | Hystrix Dashboard | Micrometer 통합 |
| 설정 | 복잡 | YAML로 간단 |
| Spring 통합 | Spring Cloud Netflix | Spring Boot Starter |

---

## Bulkhead 패턴

### Bulkhead(격벽)란?

배의 격벽처럼 시스템을 격리된 구역으로 나누어, 한 부분의 문제가 전체로 퍼지지 않게 합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    Bulkhead Pattern                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   격벽 없이:                                                 │
│   ┌─────────────────────────────────────────┐               │
│   │     하나의 큰 스레드 풀 (100 threads)    │               │
│   │  Service A, B, C 모두 같은 풀 사용      │               │
│   │  → A 장애 시 모든 스레드 점유           │               │
│   │  → B, C도 영향 받음!                    │               │
│   └─────────────────────────────────────────┘               │
│                                                              │
│   격벽 적용:                                                 │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│   │ Service A │  │ Service B │  │ Service C │                │
│   │  (30)     │  │  (40)     │  │  (30)     │                │
│   └──────────┘  └──────────┘  └──────────┘                 │
│   A 장애 시 30개만 영향                                      │
│   B, C는 정상 동작!                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Bulkhead 구현

```yaml
# application.yml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 25           # 최대 동시 호출 수
        max-wait-duration: 500ms           # 대기 시간

  thread-pool-bulkhead:
    instances:
      inventoryService:
        max-thread-pool-size: 10           # 스레드 풀 크기
        core-thread-pool-size: 5
        queue-capacity: 100
        keep-alive-duration: 100ms
```

```java
/**
 * Bulkhead 패턴 구현
 */
@Service
@RequiredArgsConstructor
public class BulkheadService {

    /**
     * 세마포어 기반 Bulkhead
     * - 동일 스레드에서 실행
     * - 컨텍스트 전파 가능
     * - 낮은 오버헤드
     */
    @Bulkhead(name = "paymentService", fallbackMethod = "paymentBulkheadFallback")
    public PaymentResult processPaymentSemaphore(PaymentRequest request) {
        return paymentClient.process(request);
    }

    /**
     * 스레드 풀 기반 Bulkhead
     * - 별도 스레드에서 실행
     * - 완전한 격리
     * - 타임아웃 제어 용이
     */
    @Bulkhead(name = "inventoryService", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<InventoryResult> checkInventoryThreadPool(String productId) {
        return CompletableFuture.supplyAsync(() -> inventoryClient.check(productId));
    }

    private PaymentResult paymentBulkheadFallback(PaymentRequest request, BulkheadFullException e) {
        log.warn("Bulkhead full for payment service");
        return PaymentResult.failed("Too many concurrent requests");
    }
}

/**
 * 프로그래밍 방식 Bulkhead
 */
@Service
public class CustomBulkheadService {

    private final Semaphore semaphore;
    private final ExecutorService executorService;

    public CustomBulkheadService() {
        // 세마포어 기반
        this.semaphore = new Semaphore(25);

        // 스레드 풀 기반
        this.executorService = new ThreadPoolExecutor(
            5,                      // core pool size
            10,                     // max pool size
            60L, TimeUnit.SECONDS,  // keep alive
            new ArrayBlockingQueue<>(100),  // queue
            new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
        );
    }

    public <T> T executeWithSemaphore(Callable<T> task, T fallback) {
        if (!semaphore.tryAcquire()) {
            return fallback;
        }
        try {
            return task.call();
        } catch (Exception e) {
            log.error("Task execution failed", e);
            return fallback;
        } finally {
            semaphore.release();
        }
    }

    public <T> CompletableFuture<T> executeWithThreadPool(Supplier<T> task) {
        return CompletableFuture.supplyAsync(task, executorService);
    }
}
```

---

## Retry 패턴

### Retry 설정

```yaml
# application.yml
resilience4j:
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 1s
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.BusinessException
```

```java
/**
 * Retry 패턴 구현
 */
@Service
public class RetryService {

    /**
     * 어노테이션 기반 Retry
     */
    @Retry(name = "paymentService", fallbackMethod = "retryFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        log.info("Attempting payment processing...");
        return paymentClient.process(request);
    }

    private PaymentResult retryFallback(PaymentRequest request, Exception e) {
        log.error("All retry attempts failed", e);
        return PaymentResult.failed("Payment failed after retries");
    }

    /**
     * 프로그래밍 방식 Retry with Exponential Backoff
     */
    public PaymentResult processWithExponentialBackoff(PaymentRequest request) {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(5)
            .intervalFunction(IntervalFunction.ofExponentialBackoff(
                Duration.ofMillis(500),   // 초기 대기 시간
                2,                         // 지수
                Duration.ofSeconds(10)     // 최대 대기 시간
            ))
            .retryOnException(e -> e instanceof IOException ||
                                   e instanceof TimeoutException)
            .retryOnResult(result -> result == null)  // 결과 기반 재시도
            .build();

        Retry retry = Retry.of("payment", config);

        // 재시도 이벤트 로깅
        retry.getEventPublisher()
            .onRetry(event -> log.info("Retry attempt #{} due to: {}",
                event.getNumberOfRetryAttempts(),
                event.getLastThrowable().getMessage()));

        Supplier<PaymentResult> supplier = Retry.decorateSupplier(
            retry,
            () -> paymentClient.process(request)
        );

        return Try.ofSupplier(supplier)
            .recover(e -> PaymentResult.failed("Payment failed"))
            .get();
    }

    /**
     * Jitter를 추가한 Retry
     * 동시에 많은 클라이언트가 재시도할 때 부하 분산
     */
    public PaymentResult processWithJitter(PaymentRequest request) {
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(3)
            .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(500),
                2,
                0.5,  // randomization factor (50% jitter)
                Duration.ofSeconds(10)
            ))
            .build();

        Retry retry = Retry.of("paymentWithJitter", config);
        return Retry.decorateSupplier(retry, () -> paymentClient.process(request)).get();
    }
}
```

### Retry 안티패턴

```java
/**
 * Retry 안티패턴 (피해야 할 것들)
 */
public class RetryAntiPatterns {

    /**
     * 안티패턴 1: 무한 재시도
     */
    public void infiniteRetry() {
        while (true) {
            try {
                externalService.call();
                break;
            } catch (Exception e) {
                // 무한 루프 → CPU 낭비, 외부 서비스 과부하
            }
        }
    }

    /**
     * 안티패턴 2: 비멱등 작업 재시도
     */
    public void retryNonIdempotent() {
        // POST 요청 재시도 → 중복 생성!
        retry(() -> httpClient.post("/orders", orderData));
    }

    /**
     * 안티패턴 3: 고정 간격 재시도 (Thundering Herd)
     */
    public void fixedIntervalRetry() {
        // 모든 클라이언트가 1초 후 동시에 재시도
        // → 서버 다시 과부하
        Thread.sleep(1000);
        retry();
    }

    /**
     * 올바른 패턴: Exponential Backoff + Jitter
     */
    public void properRetry() {
        int attempt = 0;
        int maxAttempts = 5;
        long baseDelay = 1000;

        while (attempt < maxAttempts) {
            try {
                externalService.call();
                return;
            } catch (Exception e) {
                attempt++;
                // Exponential backoff with jitter
                long delay = (long) (baseDelay * Math.pow(2, attempt));
                long jitter = (long) (delay * 0.2 * Math.random());
                Thread.sleep(delay + jitter);
            }
        }
    }
}
```

---

## Timeout 패턴

### Timeout 설정

```yaml
# application.yml
resilience4j:
  timelimiter:
    instances:
      paymentService:
        timeout-duration: 5s
        cancel-running-future: true

      inventoryService:
        timeout-duration: 3s
```

```java
/**
 * Timeout 패턴 구현
 */
@Service
public class TimeoutService {

    /**
     * TimeLimiter 어노테이션
     * 비동기 호출에만 적용 가능
     */
    @TimeLimiter(name = "paymentService", fallbackMethod = "timeoutFallback")
    public CompletableFuture<PaymentResult> processPaymentAsync(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> paymentClient.process(request));
    }

    private CompletableFuture<PaymentResult> timeoutFallback(
            PaymentRequest request, TimeoutException e) {
        log.warn("Payment timed out for order: {}", request.getOrderId());
        return CompletableFuture.completedFuture(
            PaymentResult.pending("Payment processing timeout"));
    }

    /**
     * 동기 호출에 타임아웃 적용
     */
    public PaymentResult processWithTimeout(PaymentRequest request) {
        TimeLimiterConfig config = TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(5))
            .cancelRunningFuture(true)
            .build();

        TimeLimiter timeLimiter = TimeLimiter.of("payment", config);

        Callable<PaymentResult> callable = TimeLimiter.decorateFutureSupplier(
            timeLimiter,
            () -> CompletableFuture.supplyAsync(() -> paymentClient.process(request))
        );

        try {
            return callable.call();
        } catch (TimeoutException e) {
            log.warn("Payment timed out", e);
            return PaymentResult.pending("Timeout");
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 다양한 타임아웃 레이어
     */
    public class TimeoutLayers {
        // 1. HTTP 클라이언트 타임아웃
        public RestTemplate restTemplateWithTimeout() {
            HttpComponentsClientHttpRequestFactory factory =
                new HttpComponentsClientHttpRequestFactory();
            factory.setConnectTimeout(3000);    // 연결 타임아웃
            factory.setReadTimeout(10000);       // 읽기 타임아웃
            return new RestTemplate(factory);
        }

        // 2. 서비스 레벨 타임아웃 (Resilience4j)
        @TimeLimiter(name = "service")
        public CompletableFuture<Result> serviceCall() { ... }

        // 3. 글로벌 타임아웃 (Gateway)
        // application.yml에서 설정
    }
}
```

---

## Rate Limiter 패턴

### Rate Limiter 설정

```yaml
# application.yml
resilience4j:
  ratelimiter:
    instances:
      paymentApi:
        limit-for-period: 100           # 기간당 허용 요청 수
        limit-refresh-period: 1s        # 제한 갱신 주기
        timeout-duration: 500ms         # 대기 시간

      userApi:
        limit-for-period: 1000
        limit-refresh-period: 1s
        timeout-duration: 0             # 즉시 실패
```

```java
/**
 * Rate Limiter 패턴 구현
 */
@Service
public class RateLimiterService {

    /**
     * 어노테이션 기반 Rate Limiter
     */
    @RateLimiter(name = "paymentApi", fallbackMethod = "rateLimitFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }

    private PaymentResult rateLimitFallback(PaymentRequest request, RequestNotPermitted e) {
        log.warn("Rate limit exceeded for payment API");
        throw new TooManyRequestsException("Too many requests. Please try again later.");
    }

    /**
     * 사용자별 Rate Limiter
     */
    private final ConcurrentHashMap<String, RateLimiter> userRateLimiters = new ConcurrentHashMap<>();

    public PaymentResult processPaymentPerUser(String userId, PaymentRequest request) {
        RateLimiter limiter = userRateLimiters.computeIfAbsent(userId, id -> {
            RateLimiterConfig config = RateLimiterConfig.custom()
                .limitForPeriod(10)
                .limitRefreshPeriod(Duration.ofMinutes(1))
                .timeoutDuration(Duration.ZERO)
                .build();
            return RateLimiter.of("user-" + id, config);
        });

        return RateLimiter.decorateSupplier(limiter, () -> paymentClient.process(request))
            .get();
    }

    /**
     * Token Bucket 알고리즘 구현
     */
    public class TokenBucketRateLimiter {
        private final int maxTokens;
        private final int refillRate;  // tokens per second
        private double tokens;
        private long lastRefillTime;

        public TokenBucketRateLimiter(int maxTokens, int refillRate) {
            this.maxTokens = maxTokens;
            this.refillRate = refillRate;
            this.tokens = maxTokens;
            this.lastRefillTime = System.nanoTime();
        }

        public synchronized boolean tryAcquire() {
            refill();
            if (tokens >= 1) {
                tokens--;
                return true;
            }
            return false;
        }

        private void refill() {
            long now = System.nanoTime();
            double elapsedSeconds = (now - lastRefillTime) / 1_000_000_000.0;
            tokens = Math.min(maxTokens, tokens + elapsedSeconds * refillRate);
            lastRefillTime = now;
        }
    }

    /**
     * Sliding Window Log 알고리즘
     */
    public class SlidingWindowRateLimiter {
        private final int maxRequests;
        private final long windowSizeMs;
        private final Deque<Long> requestTimestamps = new LinkedList<>();

        public SlidingWindowRateLimiter(int maxRequests, long windowSizeMs) {
            this.maxRequests = maxRequests;
            this.windowSizeMs = windowSizeMs;
        }

        public synchronized boolean tryAcquire() {
            long now = System.currentTimeMillis();
            long windowStart = now - windowSizeMs;

            // 윈도우 밖의 요청 제거
            while (!requestTimestamps.isEmpty() &&
                   requestTimestamps.peekFirst() < windowStart) {
                requestTimestamps.pollFirst();
            }

            if (requestTimestamps.size() < maxRequests) {
                requestTimestamps.addLast(now);
                return true;
            }
            return false;
        }
    }
}

/**
 * Redis 기반 분산 Rate Limiter
 */
@Service
@RequiredArgsConstructor
public class DistributedRateLimiter {

    private final StringRedisTemplate redisTemplate;

    private static final String RATE_LIMIT_SCRIPT =
        "local key = KEYS[1] " +
        "local limit = tonumber(ARGV[1]) " +
        "local window = tonumber(ARGV[2]) " +
        "local current = redis.call('INCR', key) " +
        "if current == 1 then " +
        "    redis.call('EXPIRE', key, window) " +
        "end " +
        "if current > limit then " +
        "    return 0 " +
        "end " +
        "return 1";

    public boolean tryAcquire(String key, int limit, int windowSeconds) {
        Long result = redisTemplate.execute(
            new DefaultRedisScript<>(RATE_LIMIT_SCRIPT, Long.class),
            Collections.singletonList("rate_limit:" + key),
            String.valueOf(limit),
            String.valueOf(windowSeconds)
        );
        return result != null && result == 1;
    }
}
```

---

## 실전 적용 가이드

### 패턴 조합 전략

```java
/**
 * 패턴 조합 우선순위
 * 요청 → TimeLimiter → Bulkhead → RateLimiter → CircuitBreaker → Retry → 실제 호출
 */
@Configuration
public class ResiliencePatternOrder {

    /**
     * 전체 패턴 조합 예시
     */
    @Bean
    public PaymentService paymentService(
            CircuitBreakerRegistry cbRegistry,
            RetryRegistry retryRegistry,
            BulkheadRegistry bulkheadRegistry,
            TimeLimiterRegistry tlRegistry,
            RateLimiterRegistry rlRegistry) {

        return new ResilientPaymentService(
            cbRegistry.circuitBreaker("payment"),
            retryRegistry.retry("payment"),
            bulkheadRegistry.bulkhead("payment"),
            tlRegistry.timeLimiter("payment"),
            rlRegistry.rateLimiter("payment")
        );
    }
}

@Service
public class ResilientPaymentService {

    public PaymentResult process(PaymentRequest request) {
        Supplier<PaymentResult> supplier = () -> paymentClient.process(request);

        // 데코레이터 체이닝 (안에서 밖으로)
        // Retry가 가장 안쪽: 실패 시 재시도
        // CircuitBreaker: 연속 실패 시 빠른 실패
        // RateLimiter: 요청 수 제한
        // Bulkhead: 동시성 제한
        // TimeLimiter: 시간 제한
        Supplier<PaymentResult> decorated = Decorators.ofSupplier(supplier)
            .withRetry(retry)
            .withCircuitBreaker(circuitBreaker)
            .withRateLimiter(rateLimiter)
            .withBulkhead(bulkhead)
            .decorate();

        return Try.ofSupplier(Decorators.ofSupplier(decorated)
                .decorate())
            .recover(this::handleException)
            .get();
    }

    private PaymentResult handleException(Throwable t) {
        if (t instanceof CircuitBreakerOpenException) {
            return PaymentResult.serviceUnavailable();
        }
        if (t instanceof BulkheadFullException) {
            return PaymentResult.tooManyRequests();
        }
        if (t instanceof RequestNotPermitted) {
            return PaymentResult.rateLimited();
        }
        if (t instanceof TimeoutException) {
            return PaymentResult.timeout();
        }
        return PaymentResult.failed(t.getMessage());
    }
}
```

### 모니터링 및 메트릭

```java
/**
 * Resilience4j 메트릭 노출
 */
@Configuration
public class ResilienceMetricsConfig {

    @Bean
    public TaggedCircuitBreakerMetrics circuitBreakerMetrics(
            CircuitBreakerRegistry registry, MeterRegistry meterRegistry) {
        TaggedCircuitBreakerMetrics metrics = TaggedCircuitBreakerMetrics.ofCircuitBreakerRegistry(registry);
        metrics.bindTo(meterRegistry);
        return metrics;
    }

    @Bean
    public TaggedRetryMetrics retryMetrics(
            RetryRegistry registry, MeterRegistry meterRegistry) {
        TaggedRetryMetrics metrics = TaggedRetryMetrics.ofRetryRegistry(registry);
        metrics.bindTo(meterRegistry);
        return metrics;
    }

    @Bean
    public TaggedBulkheadMetrics bulkheadMetrics(
            BulkheadRegistry registry, MeterRegistry meterRegistry) {
        TaggedBulkheadMetrics metrics = TaggedBulkheadMetrics.ofBulkheadRegistry(registry);
        metrics.bindTo(meterRegistry);
        return metrics;
    }
}

/**
 * Actuator 엔드포인트
 */
// GET /actuator/circuitbreakers
// GET /actuator/circuitbreakerevents
// GET /actuator/retries
// GET /actuator/bulkheads
```

### 설정 가이드라인

```yaml
# 서비스 특성별 설정 가이드
resilience4j:
  circuitbreaker:
    instances:
      # 결제 서비스: 중요도 높음, 안정성 중시
      paymentService:
        failure-rate-threshold: 30    # 낮은 임계치
        wait-duration-in-open-state: 60s  # 긴 대기
        sliding-window-size: 20

      # 추천 서비스: 실패 허용 가능
      recommendationService:
        failure-rate-threshold: 70    # 높은 임계치
        wait-duration-in-open-state: 10s  # 짧은 대기
        sliding-window-size: 10

  retry:
    instances:
      # 결제: 신중한 재시도
      paymentService:
        max-attempts: 2
        wait-duration: 2s

      # 조회: 적극적 재시도
      catalogService:
        max-attempts: 5
        wait-duration: 500ms

  bulkhead:
    instances:
      # 핵심 서비스: 충분한 리소스
      paymentService:
        max-concurrent-calls: 50

      # 부가 서비스: 제한된 리소스
      analyticsService:
        max-concurrent-calls: 10
```

---

## 참고 자료

- [Release It!: Design and Deploy Production-Ready Software](https://pragprog.com/titles/mnee2/release-it-second-edition/) - Michael Nygard
- [Resilience4j Documentation](https://resilience4j.readme.io/)
- [CircuitBreaker - Resilience4j](https://resilience4j.readme.io/docs/circuitbreaker)
- [Spring Cloud Circuit Breaker](https://spring.io/projects/spring-cloud-circuitbreaker)
- [Guide to Resilience4j - Baeldung](https://www.baeldung.com/resilience4j)
- [Netflix Hystrix Wiki](https://github.com/Netflix/Hystrix/wiki) (Archived)
- [Microservices Patterns](https://microservices.io/patterns/reliability/circuit-breaker.html)

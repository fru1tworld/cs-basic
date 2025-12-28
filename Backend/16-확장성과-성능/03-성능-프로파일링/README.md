# 성능 프로파일링

## 목차
1. [개요](#개요)
2. [APM 도구](#apm-도구)
3. [프로파일링 기법](#프로파일링-기법)
4. [병목 식별 방법](#병목-식별-방법)
5. [Flame Graph 분석](#flame-graph-분석)
6. [실전 적용 가이드](#실전-적용-가이드)
---

## 개요

성능 프로파일링은 시스템의 성능 병목을 식별하고 최적화하기 위한 필수 과정입니다. Michael Nygard의 "Release It!"에서 강조하듯이, 운영 환경의 성능 문제는 프로덕션 데이터와 트래픽 패턴에서만 드러나는 경우가 많습니다.

> "프로덕션에서 발생하는 성능 문제는 테스트 환경에서 재현하기 어렵다. 그러므로 운영 환경의 관측 가능성(Observability)이 중요하다." - Release It!

---

## APM 도구

### APM(Application Performance Monitoring)이란?

APM은 애플리케이션의 성능과 가용성을 실시간으로 모니터링하고 분석하는 도구입니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    APM 핵심 구성 요소                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐      │
│   │   Metrics   │   │   Traces    │   │    Logs     │      │
│   │   (수치)    │   │   (추적)    │   │   (로그)    │      │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘      │
│          │                 │                 │              │
│          └─────────────────┼─────────────────┘              │
│                            │                                 │
│                            ▼                                 │
│                  ┌─────────────────┐                        │
│                  │   Correlation   │                        │
│                  │   (상관 분석)   │                        │
│                  └────────┬────────┘                        │
│                           │                                  │
│                           ▼                                  │
│                  ┌─────────────────┐                        │
│                  │  Dashboards &   │                        │
│                  │    Alerts       │                        │
│                  └─────────────────┘                        │
│                                                              │
│   Three Pillars of Observability                            │
└─────────────────────────────────────────────────────────────┘
```

### 1. New Relic

```java
/**
 * New Relic APM 설정
 * - 자동 계측(Auto-instrumentation) 지원
 * - Java Agent 기반
 */

// build.gradle
dependencies {
    implementation 'com.newrelic.agent.java:newrelic-api:8.7.0'
}

// newrelic.yml
common: &default_settings
  license_key: 'your_license_key'
  app_name: 'My Application'

  # 분산 추적 활성화
  distributed_tracing:
    enabled: true

  # 트랜잭션 추적
  transaction_tracer:
    enabled: true
    transaction_threshold: apdex_f
    record_sql: obfuscated
    explain_enabled: true
    explain_threshold: 0.5

  # 느린 SQL 추적
  slow_sql:
    enabled: true

  # 에러 분석
  error_collector:
    enabled: true
    ignore_errors: com.example.ExpectedException

/**
 * New Relic 커스텀 계측
 */
@Service
public class OrderService {

    @Trace(dispatcher = true)  // 새 트랜잭션 시작
    public Order createOrder(OrderRequest request) {
        NewRelic.addCustomParameter("userId", request.getUserId());
        NewRelic.addCustomParameter("orderAmount", request.getTotalAmount());

        try {
            Order order = processOrder(request);
            NewRelic.incrementCounter("Custom/Orders/Created");
            return order;
        } catch (Exception e) {
            NewRelic.noticeError(e);
            throw e;
        }
    }

    @Trace  // 기존 트랜잭션에 추가
    private Order processOrder(OrderRequest request) {
        // 처리 로직
    }

    /**
     * 커스텀 메트릭 기록
     */
    public void recordCustomMetrics() {
        // 응답 시간 기록
        NewRelic.recordResponseTimeMetric("Custom/ExternalApi/ResponseTime", 150);

        // 이벤트 기록
        Map<String, Object> eventAttributes = new HashMap<>();
        eventAttributes.put("orderId", "12345");
        eventAttributes.put("amount", 99.99);
        NewRelic.getAgent().getInsights().recordCustomEvent("OrderCompleted", eventAttributes);
    }
}
```

### 2. Datadog

```java
/**
 * Datadog APM 설정
 * - dd-java-agent 사용
 * - 자동/수동 계측 지원
 */

// Java Agent 옵션
// java -javaagent:/path/to/dd-java-agent.jar \
//      -Ddd.service=my-service \
//      -Ddd.env=production \
//      -Ddd.version=1.0.0 \
//      -Ddd.trace.sample.rate=1.0 \
//      -jar my-app.jar

// build.gradle
dependencies {
    implementation 'com.datadoghq:dd-trace-api:1.24.0'
}

/**
 * Datadog 커스텀 트레이싱
 */
@Service
public class PaymentService {

    private final Tracer tracer;

    public PaymentService() {
        this.tracer = GlobalTracer.get();
    }

    public PaymentResult processPayment(PaymentRequest request) {
        // 커스텀 스팬 생성
        Span span = tracer.buildSpan("payment.process")
            .withTag("payment.method", request.getMethod())
            .withTag("payment.amount", request.getAmount())
            .start();

        try (Scope scope = tracer.activateSpan(span)) {
            // 외부 PG 호출
            Span pgSpan = tracer.buildSpan("pg.request")
                .withTag("pg.provider", "nice")
                .start();

            try (Scope pgScope = tracer.activateSpan(pgSpan)) {
                PaymentResult result = callPaymentGateway(request);
                pgSpan.setTag("pg.transaction_id", result.getTransactionId());
                return result;
            } finally {
                pgSpan.finish();
            }
        } catch (Exception e) {
            span.setTag("error", true);
            span.log(Map.of("error.message", e.getMessage()));
            throw e;
        } finally {
            span.finish();
        }
    }
}

/**
 * Datadog 메트릭 전송
 */
@Component
public class DatadogMetricsReporter {

    private final StatsDClient statsd;

    public DatadogMetricsReporter() {
        this.statsd = new NonBlockingStatsDClientBuilder()
            .prefix("myapp")
            .hostname("localhost")
            .port(8125)
            .build();
    }

    public void recordOrderMetrics(Order order) {
        // 카운터
        statsd.incrementCounter("orders.created", "region:" + order.getRegion());

        // 게이지
        statsd.recordGaugeValue("orders.pending", pendingOrders.size());

        // 히스토그램
        statsd.recordHistogramValue("orders.processing_time",
            order.getProcessingTimeMs(),
            "tier:" + order.getTier());

        // 이벤트
        Event event = Event.builder()
            .withTitle("Large Order Received")
            .withText("Order ID: " + order.getId())
            .withAlertType(Event.AlertType.INFO)
            .build();
        statsd.recordEvent(event);
    }

    @PreDestroy
    public void close() {
        statsd.close();
    }
}
```

### 3. Pinpoint (NAVER)

```java
/**
 * Pinpoint APM 설정
 * - 한국에서 많이 사용되는 오픈소스 APM
 * - 분산 추적 특화
 * - 바이트코드 조작으로 무침투적 모니터링
 */

// pinpoint-agent/pinpoint.config
profiler.sampling.enable=true
profiler.sampling.rate=1

# Application 설정
profiler.applicationservertype=SPRING_BOOT

# HTTP 추적
profiler.apache.httpclient4.entity.enable=true
profiler.apache.httpclient4.entity.statuscode=true

# SQL 추적
profiler.jdbc.sqlcachesize=1024
profiler.jdbc.tracesqlbindvalue=true

# Redis 추적
profiler.redis.pipeline=true

/**
 * Pinpoint 사용 시 Agent 옵션
 */
// java -javaagent:/path/to/pinpoint-bootstrap.jar \
//      -Dpinpoint.agentId=my-agent-id \
//      -Dpinpoint.applicationName=my-application \
//      -Dpinpoint.profiler.profiles.active=release \
//      -jar my-app.jar

/**
 * Pinpoint 플러그인 개발 예시
 * 커스텀 라이브러리 추적이 필요한 경우
 */
public class CustomServicePlugin implements ProfilerPlugin, TransformCallback {

    @Override
    public void setup(ProfilerPluginSetupContext context) {
        // 추적할 클래스 설정
        context.addApplicationTypeDetector(new CustomServiceDetector());

        TransformTemplate template = context.getTransformTemplate();
        template.transform("com.example.CustomService", CustomServiceTransformer.class);
    }
}

public class CustomServiceTransformer implements TransformCallback {

    @Override
    public byte[] doInTransform(Instrumentor instrumentor,
                                 ClassLoader loader,
                                 String className,
                                 Class<?> classBeingRedefined,
                                 ProtectionDomain protectionDomain,
                                 byte[] classfileBuffer) throws InstrumentException {

        InstrumentClass target = instrumentor.getInstrumentClass(loader, className, classfileBuffer);

        // 메서드 인터셉터 추가
        InstrumentMethod method = target.getDeclaredMethod("execute", "java.lang.String");
        if (method != null) {
            method.addInterceptor(CustomServiceInterceptor.class);
        }

        return target.toBytecode();
    }
}
```

### APM 도구 비교

| 특성 | New Relic | Datadog | Pinpoint |
|------|-----------|---------|----------|
| 유형 | 상용 SaaS | 상용 SaaS | 오픈소스 |
| 가격 | 높음 | 높음 | 무료 |
| 설치 복잡도 | 낮음 | 낮음 | 높음 (HBase 필요) |
| 분산 추적 | 우수 | 우수 | 매우 우수 |
| 인프라 모니터링 | 우수 | 매우 우수 | 없음 |
| 커뮤니티 | 활발 | 활발 | 활발 (한국) |
| 학습 곡선 | 낮음 | 중간 | 중간 |

---

## 프로파일링 기법

### 1. CPU 프로파일링

```java
/**
 * CPU 프로파일링 기법
 */
public class CPUProfilingExample {

    /**
     * JFR (Java Flight Recorder) 사용
     * JDK 11+에서 무료 사용 가능
     */
    public void enableJFR() {
        // 커맨드라인 옵션
        // java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

        // 프로그래밍 방식
        Configuration config = Configuration.getConfiguration("profile");
        Recording recording = new Recording(config);
        recording.setMaxSize(100 * 1024 * 1024); // 100MB
        recording.setMaxAge(Duration.ofMinutes(10));
        recording.setDestination(Path.of("recording.jfr"));
        recording.start();
    }

    /**
     * async-profiler 사용 (리눅스/맥)
     * 낮은 오버헤드, 정확한 프로파일링
     */
    public void useAsyncProfiler() {
        // 커맨드라인:
        // ./profiler.sh -d 30 -f profile.html <pid>

        // 결과: Flame Graph HTML 생성
    }

    /**
     * VisualVM을 통한 샘플링
     */
    public void cpuIntensiveMethod() {
        // VisualVM에서 Sampler 탭 사용
        // CPU 사용률이 높은 메서드 식별

        // 예: 비효율적인 문자열 연결
        String result = "";
        for (int i = 0; i < 10000; i++) {
            result += "item" + i;  // 매번 새 String 객체 생성
        }

        // 개선된 버전
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 10000; i++) {
            sb.append("item").append(i);
        }
        String optimizedResult = sb.toString();
    }
}
```

### 2. 메모리 프로파일링

```java
/**
 * 메모리 프로파일링 기법
 */
public class MemoryProfilingExample {

    /**
     * 힙 덤프 분석
     */
    public void analyzeHeapDump() {
        // 힙 덤프 생성
        // jmap -dump:format=b,file=heap.hprof <pid>

        // OOM 발생 시 자동 덤프
        // -XX:+HeapDumpOnOutOfMemoryError
        // -XX:HeapDumpPath=/path/to/dumps

        // MAT(Memory Analyzer Tool)로 분석
        // - Dominator Tree: 메모리를 많이 차지하는 객체
        // - Leak Suspects: 메모리 누수 의심 객체
        // - Histogram: 클래스별 인스턴스 수와 크기
    }

    /**
     * JFR 메모리 이벤트 분석
     */
    public void analyzeMemoryWithJFR() {
        // JFR 설정에서 메모리 이벤트 활성화
        // jdk.ObjectAllocationInNewTLAB
        // jdk.ObjectAllocationOutsideTLAB

        // JMC(JDK Mission Control)로 분석
    }

    /**
     * 메모리 누수 패턴 예시
     */
    public class MemoryLeakPatterns {

        // 패턴 1: 정적 컬렉션에 계속 추가
        private static final List<Object> cache = new ArrayList<>();

        public void leakingMethod(Object data) {
            cache.add(data);  // 제거하지 않으면 누수
        }

        // 패턴 2: 리스너 미해제
        public void registerListener() {
            EventBus.register(new EventListener() {
                @Override
                public void onEvent(Event e) {
                    // 처리
                }
            });
            // 해제하지 않으면 누수
        }

        // 패턴 3: 커넥션/스트림 미반환
        public void leakingConnection() throws SQLException {
            Connection conn = dataSource.getConnection();
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT * FROM users");
            // conn.close() 호출 안 함 -> 커넥션 누수
        }

        // 해결: try-with-resources 사용
        public void properResourceHandling() throws SQLException {
            try (Connection conn = dataSource.getConnection();
                 Statement stmt = conn.createStatement();
                 ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {
                // 처리
            }  // 자동으로 close() 호출
        }
    }
}
```

### 3. I/O 및 대기 시간 프로파일링

```java
/**
 * I/O 프로파일링
 */
public class IOProfilingExample {

    /**
     * 네트워크 I/O 분석
     */
    @Trace
    public void analyzeNetworkIO() {
        // Wireshark, tcpdump로 패킷 분석
        // APM의 외부 호출 추적 활용

        RestTemplate restTemplate = new RestTemplate();

        // 타이밍 측정
        long start = System.nanoTime();
        ResponseEntity<String> response = restTemplate.getForEntity(
            "http://api.example.com/data",
            String.class
        );
        long duration = System.nanoTime() - start;

        log.info("External API call took {} ms", duration / 1_000_000);
    }

    /**
     * 디스크 I/O 분석
     */
    public void analyzeDiskIO() {
        // iostat -x 1 으로 디스크 사용률 확인
        // await: 평균 대기 시간
        // %util: 디스크 사용률

        // JFR 파일 I/O 이벤트
        // jdk.FileRead, jdk.FileWrite
    }

    /**
     * 락 경합 분석
     */
    public void analyzeLockContention() {
        // JFR 이벤트: jdk.JavaMonitorEnter, jdk.JavaMonitorWait

        // 예: 락 경합 발생 코드
        private final Object lock = new Object();

        public void contentionExample() {
            synchronized (lock) {
                // 긴 작업 -> 다른 스레드 대기
                heavyOperation();
            }
        }

        // 해결: 락 범위 최소화
        public void reducedContention() {
            Data data;
            synchronized (lock) {
                data = fetchData();  // 빠른 작업만 락 안에서
            }
            processData(data);  // 락 밖에서 처리
        }
    }
}
```

### 4. 분산 추적 (Distributed Tracing)

```java
/**
 * 분산 추적 구현
 * OpenTelemetry 기반
 */
@Configuration
public class OpenTelemetryConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "order-service",
                ResourceAttributes.SERVICE_VERSION, "1.0.0"
            )));

        // Jaeger Exporter 설정
        SpanExporter jaegerExporter = JaegerGrpcSpanExporter.builder()
            .setEndpoint("http://jaeger:14250")
            .build();

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .addSpanProcessor(BatchSpanProcessor.builder(jaegerExporter).build())
            .setResource(resource)
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(W3CTraceContextPropagator.getInstance()))
            .build();
    }
}

@Service
public class DistributedTracingExample {

    private final Tracer tracer;
    private final RestTemplate restTemplate;

    public DistributedTracingExample(OpenTelemetry openTelemetry) {
        this.tracer = openTelemetry.getTracer("order-service");
    }

    /**
     * 분산 트레이스 예시
     * Order Service -> Payment Service -> Notification Service
     */
    public OrderResult processOrder(OrderRequest request) {
        Span span = tracer.spanBuilder("processOrder")
            .setSpanKind(SpanKind.SERVER)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            span.setAttribute("order.id", request.getOrderId());
            span.setAttribute("order.amount", request.getAmount());

            // 1. 결제 서비스 호출
            PaymentResult payment = callPaymentService(request);

            // 2. 알림 서비스 호출
            callNotificationService(request, payment);

            span.setStatus(StatusCode.OK);
            return new OrderResult(request.getOrderId(), payment.getTransactionId());

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }

    private PaymentResult callPaymentService(OrderRequest request) {
        Span span = tracer.spanBuilder("callPaymentService")
            .setSpanKind(SpanKind.CLIENT)
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Context propagation: HTTP 헤더에 trace ID 전파
            HttpHeaders headers = new HttpHeaders();
            W3CTraceContextPropagator.getInstance().inject(
                Context.current(),
                headers,
                HttpHeaders::set
            );

            HttpEntity<PaymentRequest> entity = new HttpEntity<>(
                new PaymentRequest(request.getOrderId(), request.getAmount()),
                headers
            );

            ResponseEntity<PaymentResult> response = restTemplate.exchange(
                "http://payment-service/api/payments",
                HttpMethod.POST,
                entity,
                PaymentResult.class
            );

            return response.getBody();
        } finally {
            span.end();
        }
    }
}
```

---

## 병목 식별 방법

### 1. Amdahl의 법칙 활용

```
┌─────────────────────────────────────────────────────────────┐
│                     Amdahl's Law                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Speedup = 1 / ((1 - P) + P/S)                             │
│                                                              │
│   P = 병렬화 가능 비율                                       │
│   S = 병렬화 배수                                            │
│                                                              │
│   예: 90% 병렬화 가능, 10배 속도 향상                        │
│   Speedup = 1 / ((1 - 0.9) + 0.9/10)                        │
│          = 1 / (0.1 + 0.09)                                 │
│          = 5.26배                                            │
│                                                              │
│   → 10% 직렬 부분이 병목!                                    │
│   → 90% 개선해도 5.26배만 향상                               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. 병목 유형별 식별 방법

```java
/**
 * 병목 유형별 진단 방법
 */
public class BottleneckIdentification {

    /**
     * CPU 병목 진단
     */
    public void diagnoseCPUBottleneck() {
        // 1. top/htop으로 CPU 사용률 확인
        // us(user), sy(system) 모두 높으면 CPU 병목

        // 2. JFR/async-profiler로 핫스팟 메서드 식별

        // 3. 일반적인 CPU 병목 원인:
        //    - 비효율적인 알고리즘 (O(n^2) 등)
        //    - 과도한 직렬화/역직렬화
        //    - 정규식 백트래킹
        //    - 동기 암호화 작업
    }

    /**
     * 메모리 병목 진단
     */
    public void diagnoseMemoryBottleneck() {
        // 1. jstat -gc <pid> 1000 으로 GC 확인
        // Full GC가 자주 발생하면 메모리 병목

        // 2. GC 로그 분석
        // -Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10m

        // 3. 일반적인 메모리 병목 원인:
        //    - 힙 크기 부족
        //    - 메모리 누수
        //    - 큰 객체 빈번한 생성
        //    - 잘못된 GC 설정
    }

    /**
     * I/O 병목 진단
     */
    public void diagnoseIOBottleneck() {
        // 1. iostat -x 1 으로 디스크 사용률 확인
        // %util이 높으면 디스크 병목

        // 2. netstat, ss로 네트워크 연결 상태 확인
        // TIME_WAIT이 많으면 커넥션 문제

        // 3. APM에서 외부 호출 시간 확인

        // 4. 일반적인 I/O 병목 원인:
        //    - 느린 외부 API
        //    - 디스크 I/O 과다
        //    - 네트워크 대역폭 부족
        //    - 커넥션 풀 부족
    }

    /**
     * 락 경합 병목 진단
     */
    public void diagnoseLockContention() {
        // 1. jstack <pid> 으로 스레드 덤프
        // BLOCKED 상태 스레드 많으면 락 경합

        // 2. JFR의 Lock 관련 이벤트 분석

        // 3. 일반적인 락 경합 원인:
        //    - synchronized 블록 범위가 너무 큼
        //    - 단일 자원에 대한 과도한 경쟁
        //    - 데드락
    }

    /**
     * 데이터베이스 병목 진단
     */
    public void diagnoseDatabaseBottleneck() {
        // 1. 슬로우 쿼리 로그 분석
        // MySQL: slow_query_log
        // PostgreSQL: log_min_duration_statement

        // 2. EXPLAIN으로 실행 계획 확인

        // 3. 커넥션 풀 상태 확인
        // HikariCP: HikariDataSource.getHikariPoolMXBean()

        // 4. 일반적인 DB 병목 원인:
        //    - N+1 쿼리
        //    - 인덱스 미사용
        //    - 풀 스캔
        //    - 락 대기
    }
}
```

### 3. 병목 우선순위 결정

```java
/**
 * 병목 우선순위 분석 프레임워크
 */
public class BottleneckPrioritization {

    /**
     * USE 방법론 (Utilization, Saturation, Errors)
     * Brendan Gregg 제안
     */
    public BottleneckReport analyzeWithUSE() {
        BottleneckReport report = new BottleneckReport();

        // CPU 분석
        report.addResource("CPU", new USEMetrics(
            getCpuUtilization(),      // Utilization: CPU 사용률
            getCpuRunQueueLength(),   // Saturation: 실행 큐 길이
            getCpuErrors()            // Errors: 에러 수
        ));

        // Memory 분석
        report.addResource("Memory", new USEMetrics(
            getMemoryUtilization(),
            getSwapUsage(),           // Saturation: 스왑 사용
            getOOMKills()
        ));

        // Disk I/O 분석
        report.addResource("Disk", new USEMetrics(
            getDiskUtilization(),
            getDiskQueueLength(),     // Saturation: I/O 큐
            getDiskErrors()
        ));

        // Network 분석
        report.addResource("Network", new USEMetrics(
            getNetworkUtilization(),
            getNetworkDrops(),        // Saturation: 패킷 드롭
            getNetworkErrors()
        ));

        return report;
    }

    /**
     * RED 방법론 (Request Rate, Errors, Duration)
     * 마이크로서비스에 적합
     */
    public ServiceMetrics analyzeWithRED(String serviceName) {
        return ServiceMetrics.builder()
            .rate(getRequestRate(serviceName))         // 초당 요청 수
            .errors(getErrorRate(serviceName))         // 에러율
            .duration(getResponseTime(serviceName))    // 응답 시간 (p50, p95, p99)
            .build();
    }

    /**
     * 병목 우선순위 점수 계산
     */
    public int calculatePriority(BottleneckInfo bottleneck) {
        int score = 0;

        // 영향도 (사용자 경험에 미치는 영향)
        score += bottleneck.getImpactPercentage() * 3;

        // 빈도 (얼마나 자주 발생하는지)
        score += bottleneck.getFrequency() * 2;

        // 해결 용이성 (쉽게 해결 가능한지)
        score += (100 - bottleneck.getComplexity());

        return score;
    }
}
```

---

## Flame Graph 분석

### Flame Graph란?

```
┌─────────────────────────────────────────────────────────────┐
│                      Flame Graph                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                      main()                          │   │
│   ├─────────────────────────────────────────────────────┤   │
│   │         processRequest()              │  idle      │   │
│   ├─────────────────────────────────┬─────┼─────────────┤   │
│   │    handleOrder()    │ validateData()  │            │   │
│   ├──────────────┬──────┼─────────────────┤            │   │
│   │ createOrder()│query │    validate()   │            │   │
│   ├──────────────┴──────┴─────────────────┴─────────────┤   │
│                                                              │
│   X축: 스택 프레임 (폭 = 시간/샘플 비율)                     │
│   Y축: 콜 스택 깊이 (아래가 루트, 위가 리프)                 │
│   색상: 무작위 (가독성 목적)                                 │
│                                                              │
│   넓은 프레임 = 많은 시간 소비 = 최적화 대상!                │
└─────────────────────────────────────────────────────────────┘
```

### Flame Graph 생성

```bash
# async-profiler로 Flame Graph 생성 (가장 권장)
./profiler.sh -d 30 -f flamegraph.html <pid>

# JFR + 변환
# 1. JFR 녹화
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar

# 2. jfr2flame으로 변환
git clone https://github.com/jvm-profiling-tools/async-profiler
cd async-profiler
./jfr2flame recording.jfr flamegraph.html

# perf + FlameGraph (리눅스)
perf record -F 99 -p <pid> -g -- sleep 30
perf script | ./stackcollapse-perf.pl | ./flamegraph.pl > flamegraph.svg
```

### Flame Graph 분석 실습

```java
/**
 * Flame Graph 분석을 위한 예시 코드
 */
public class FlameGraphAnalysisExample {

    /**
     * 성능 문제가 있는 메서드
     * Flame Graph에서 넓게 표시됨
     */
    public List<OrderSummary> getOrderSummaries(List<Long> orderIds) {
        List<OrderSummary> summaries = new ArrayList<>();

        for (Long orderId : orderIds) {
            // N+1 문제: 각 주문마다 개별 쿼리
            Order order = orderRepository.findById(orderId);  // 여기가 넓게 표시

            // 또 다른 개별 쿼리
            User user = userRepository.findById(order.getUserId());

            // 비효율적인 문자열 처리
            String description = buildDescription(order, user);  // 여기도 넓게

            summaries.add(new OrderSummary(order, user, description));
        }

        return summaries;
    }

    private String buildDescription(Order order, User user) {
        // 비효율적인 문자열 연결
        String result = "";
        for (OrderItem item : order.getItems()) {
            result += item.getName() + ", ";  // 반복적인 String 생성
        }
        return result;
    }

    /**
     * 최적화된 버전
     * Flame Graph에서 좁게 표시됨
     */
    public List<OrderSummary> getOrderSummariesOptimized(List<Long> orderIds) {
        // 배치 조회로 N+1 해결
        List<Order> orders = orderRepository.findAllByIdIn(orderIds);

        // 사용자 ID 추출 및 배치 조회
        Set<Long> userIds = orders.stream()
            .map(Order::getUserId)
            .collect(Collectors.toSet());
        Map<Long, User> userMap = userRepository.findAllByIdIn(userIds)
            .stream()
            .collect(Collectors.toMap(User::getId, Function.identity()));

        // 병렬 스트림으로 처리
        return orders.parallelStream()
            .map(order -> {
                User user = userMap.get(order.getUserId());
                String description = buildDescriptionOptimized(order, user);
                return new OrderSummary(order, user, description);
            })
            .collect(Collectors.toList());
    }

    private String buildDescriptionOptimized(Order order, User user) {
        // StringBuilder 사용
        return order.getItems().stream()
            .map(OrderItem::getName)
            .collect(Collectors.joining(", "));
    }
}
```

### Flame Graph 해석 팁

```
┌─────────────────────────────────────────────────────────────┐
│                Flame Graph 해석 가이드                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   1. 플래토 (Plateau) 찾기                                   │
│      ┌────────────────────────────────┐                     │
│      │     넓고 평평한 영역 = 병목     │                     │
│      └────────────────────────────────┘                     │
│                                                              │
│   2. 타워 (Tower) 분석                                       │
│      ┌──┐                                                   │
│      │  │  높고 좁은 영역 = 깊은 호출 스택                   │
│      │  │  재귀 또는 프레임워크 오버헤드                     │
│      └──┘                                                   │
│                                                              │
│   3. 패턴별 해석:                                            │
│                                                              │
│      넓은 JDBC 프레임 → DB 쿼리 최적화 필요                  │
│      넓은 JSON 프레임 → 직렬화 최적화 필요                   │
│      넓은 GC 프레임   → 메모리 사용 최적화 필요              │
│      넓은 Lock 프레임 → 동시성 문제 해결 필요                │
│                                                              │
│   4. 비교 분석:                                              │
│      개선 전/후 Flame Graph를 비교하여                       │
│      최적화 효과 검증                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 실전 적용 가이드

### 성능 모니터링 체크리스트

```yaml
# 운영 환경 성능 모니터링 체크리스트
performance_monitoring:
  apm:
    - tool: "Datadog/New Relic/Pinpoint 중 택일"
    - coverage: "모든 프로덕션 서비스"
    - retention: "최소 30일"

  metrics:
    - response_time_p99: "< 500ms"
    - error_rate: "< 0.1%"
    - throughput: "목표 RPS 이상"
    - cpu_utilization: "< 70%"
    - memory_utilization: "< 80%"
    - gc_pause_time: "< 200ms"

  alerts:
    - response_time_degradation: "p99 > 1s for 5 min"
    - error_spike: "error rate > 1% for 2 min"
    - resource_exhaustion: "CPU/Memory > 90% for 10 min"
    - no_traffic: "0 requests for 5 min"

  profiling:
    - continuous_profiling: "enabled in production"
    - sample_rate: "1%"
    - flame_graph_review: "weekly"
```

### 성능 문제 대응 플로우

```
성능 문제 감지
      │
      ▼
APM 대시보드 확인
      │
      ├── 응답 시간 증가 ──→ 트레이스 분석 ──→ 느린 스팬 식별
      │
      ├── 에러율 증가 ──→ 에러 로그 분석 ──→ 근본 원인 파악
      │
      └── 리소스 사용률 증가 ──→ 메트릭 분석 ──→ 병목 유형 식별
                                                      │
                                                      ▼
                                              ┌───────────────┐
                                              │ 병목 유형별   │
                                              │ 상세 프로파일 │
                                              └───────┬───────┘
                                                      │
      ┌─────────────────────┬─────────────────────────┼───────────────────┐
      │                     │                         │                   │
      ▼                     ▼                         ▼                   ▼
   CPU 병목            메모리 병목               I/O 병목            DB 병목
      │                     │                         │                   │
   Flame Graph         Heap Dump               네트워크 분석        슬로우 쿼리
   async-profiler      MAT 분석                커넥션 풀 확인       EXPLAIN 분석
```

---

## 참고 자료

- [New Relic Documentation](https://docs.newrelic.com/)
- [Datadog APM Documentation](https://docs.datadoghq.com/tracing/)
- [Pinpoint APM GitHub](https://github.com/pinpoint-apm/pinpoint)
- [Flame Graphs - Brendan Gregg](https://www.brendangregg.com/flamegraphs.html)
- [async-profiler GitHub](https://github.com/async-profiler/async-profiler)
- [Java Flight Recorder Guide](https://docs.oracle.com/javacomponents/jmc-5-4/jfr-runtime-guide/)
- [Release It!: Design and Deploy Production-Ready Software](https://pragprog.com/titles/mnee2/release-it-second-edition/) - Michael Nygard

# 분산 추적 (Distributed Tracing)

## 목차
1. [개요](#개요)
2. [OpenTelemetry](#opentelemetry)
3. [Jaeger](#jaeger)
4. [Zipkin](#zipkin)
5. [Trace Context (W3C)](#trace-context-w3c)
6. [Span과 Trace](#span과-trace)
7. [샘플링 전략](#샘플링-전략)
8. [실무 Best Practices](#실무-best-practices)
---

## 개요

분산 추적(Distributed Tracing)은 마이크로서비스 아키텍처에서 단일 요청이 여러 서비스를 거치며 처리되는 과정을 추적하는 기술입니다. 로그가 개별 이벤트를, 메트릭이 집계된 수치를 제공한다면, 분산 추적은 요청의 전체 흐름과 각 단계의 지연 시간을 시각화합니다.

### 분산 추적의 필요성

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 분산 시스템에서의 요청 흐름 복잡성                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Client                                                                 │
│    │                                                                    │
│    ▼                                                                    │
│  ┌─────────────┐                                                       │
│  │ API Gateway │ ─────────────────────────────────────────────┐        │
│  └──────┬──────┘                                              │        │
│         │                                                      │        │
│    ┌────┴────┐                                                │        │
│    ▼         ▼                                                ▼        │
│  ┌─────┐  ┌─────┐                                        ┌─────┐      │
│  │Order│  │User │                                        │Auth │      │
│  │ Svc │  │ Svc │                                        │ Svc │      │
│  └──┬──┘  └─────┘                                        └─────┘      │
│     │                                                                  │
│  ┌──┴──────────────┐                                                  │
│  ▼                 ▼                                                  │
│  ┌─────┐       ┌─────────┐                                            │
│  │Inven│       │ Payment │                                            │
│  │tory │       │   Svc   │                                            │
│  └──┬──┘       └────┬────┘                                            │
│     │               │                                                  │
│     ▼               ▼                                                  │
│  ┌─────┐       ┌─────────┐                                            │
│  │Redis│       │  Stripe │                                            │
│  └─────┘       │   API   │                                            │
│                └─────────┘                                            │
│                                                                         │
│  문제: "주문 API가 느린데, 어디서 지연이 발생하는가?"                      │
│  해결: 분산 추적으로 전체 요청 흐름과 각 구간 소요 시간 확인               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 관찰성 세 기둥에서의 위치

```
┌─────────────────────────────────────────────────────────────────┐
│                    Three Pillars of Observability               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│     Logs                  Metrics                Traces         │
│  ┌──────────┐          ┌──────────┐          ┌──────────┐      │
│  │ 무엇이    │          │ 얼마나   │          │ 어디서    │      │
│  │ 발생했나? │          │ 발생했나?│          │ 발생했나? │      │
│  │          │          │          │          │          │      │
│  │ • 이벤트  │          │ • 카운터  │          │ • 요청흐름│      │
│  │ • 상세   │          │ • 게이지  │          │ • 지연시간│      │
│  │ • 디버깅  │          │ • 집계   │          │ • 의존성  │      │
│  └──────────┘          └──────────┘          └──────────┘      │
│       │                     │                     │            │
│       │                     │                     │            │
│       └─────────────────────┼─────────────────────┘            │
│                             │                                   │
│                             ▼                                   │
│                    ┌────────────────┐                          │
│                    │  Correlation   │                          │
│                    │  trace_id로    │                          │
│                    │  모두 연결     │                          │
│                    └────────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## OpenTelemetry

OpenTelemetry(OTel)는 CNCF에서 관리하는 관찰성 프레임워크로, Traces, Metrics, Logs를 수집하고 전송하기 위한 표준화된 API, SDK, 도구를 제공합니다. 2024년 기준 Kubernetes 다음으로 큰 CNCF 프로젝트입니다.

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      OpenTelemetry Architecture                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │                        Application                             │     │
│  │  ┌─────────────────────────────────────────────────────────┐  │     │
│  │  │                   OTel SDK                               │  │     │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │     │
│  │  │  │   Tracer    │  │   Meter     │  │   Logger    │      │  │     │
│  │  │  │  Provider   │  │  Provider   │  │  Provider   │      │  │     │
│  │  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘      │  │     │
│  │  │         │                │                │              │  │     │
│  │  │  ┌──────▼────────────────▼────────────────▼──────┐      │  │     │
│  │  │  │              Exporters                         │      │  │     │
│  │  │  │  • OTLP (OpenTelemetry Protocol)              │      │  │     │
│  │  │  │  • Jaeger, Zipkin                             │      │  │     │
│  │  │  │  • Prometheus                                  │      │  │     │
│  │  │  └───────────────────┬────────────────────────────┘      │  │     │
│  │  └──────────────────────┼────────────────────────────────────┘  │     │
│  └─────────────────────────┼────────────────────────────────────────┘    │
│                            │                                             │
│                            ▼                                             │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    OTel Collector                                 │   │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐                    │   │
│  │  │Receivers │ → │Processors│ → │Exporters │                    │   │
│  │  │          │    │          │    │          │                    │   │
│  │  │ • OTLP   │    │ • Batch  │    │ • Jaeger │                    │   │
│  │  │ • Jaeger │    │ • Filter │    │ • Tempo  │                    │   │
│  │  │ • Zipkin │    │ • Sample │    │ • Zipkin │                    │   │
│  │  │ • Prom   │    │ • Attrib │    │ • OTLP   │                    │   │
│  │  └──────────┘    └──────────┘    └──────────┘                    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### OpenTelemetry 구성요소

| 구성요소 | 설명 |
|---------|------|
| **API** | 계측을 위한 인터페이스 (벤더 중립) |
| **SDK** | API 구현체, 설정 및 내보내기 |
| **Collector** | 텔레메트리 데이터 수신, 처리, 내보내기 |
| **OTLP** | OpenTelemetry Protocol (표준 전송 프로토콜) |
| **Instrumentation** | 자동/수동 계측 라이브러리 |

### Java 구현 예시

**build.gradle**
```gradle
dependencies {
    // OpenTelemetry API
    implementation 'io.opentelemetry:opentelemetry-api:1.34.0'
    implementation 'io.opentelemetry:opentelemetry-sdk:1.34.0'

    // OTLP Exporter
    implementation 'io.opentelemetry:opentelemetry-exporter-otlp:1.34.0'

    // Auto-instrumentation agent (선택)
    // -javaagent:opentelemetry-javaagent.jar

    // Spring Boot integration
    implementation 'io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter:2.1.0-alpha'
}
```

**OpenTelemetry 설정**
```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.propagation.W3CTraceContextPropagator;
import io.opentelemetry.context.propagation.ContextPropagators;
import io.opentelemetry.exporter.otlp.trace.OtlpGrpcSpanExporter;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;

@Configuration
public class OpenTelemetryConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        // 리소스 정의 (서비스 정보)
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "order-service",
                ResourceAttributes.SERVICE_VERSION, "1.0.0",
                ResourceAttributes.DEPLOYMENT_ENVIRONMENT, "production"
            )));

        // OTLP Exporter 설정
        OtlpGrpcSpanExporter spanExporter = OtlpGrpcSpanExporter.builder()
            .setEndpoint("http://otel-collector:4317")
            .setTimeout(Duration.ofSeconds(10))
            .build();

        // TracerProvider 설정
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .addSpanProcessor(BatchSpanProcessor.builder(spanExporter)
                .setMaxQueueSize(2048)
                .setMaxExportBatchSize(512)
                .setScheduleDelay(Duration.ofMillis(5000))
                .build())
            .build();

        // OpenTelemetry 인스턴스 생성
        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .setPropagators(ContextPropagators.create(
                W3CTraceContextPropagator.getInstance()))
            .buildAndRegisterGlobal();
    }

    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("order-service", "1.0.0");
    }
}
```

**수동 계측 예시**
```java
@Service
public class OrderService {

    private final Tracer tracer;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;

    public Order createOrder(OrderRequest request) {
        // 부모 Span 생성
        Span span = tracer.spanBuilder("createOrder")
            .setSpanKind(SpanKind.INTERNAL)
            .setAttribute("order.customerId", request.getCustomerId())
            .setAttribute("order.itemCount", request.getItems().size())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // 재고 확인 (자식 Span)
            Span inventorySpan = tracer.spanBuilder("checkInventory")
                .setSpanKind(SpanKind.CLIENT)
                .startSpan();

            try (Scope inventoryScope = inventorySpan.makeCurrent()) {
                boolean available = inventoryClient.checkAvailability(request.getItems());
                inventorySpan.setAttribute("inventory.available", available);

                if (!available) {
                    inventorySpan.setStatus(StatusCode.ERROR, "Insufficient inventory");
                    throw new InsufficientInventoryException();
                }
            } finally {
                inventorySpan.end();
            }

            // 결제 처리 (자식 Span)
            Span paymentSpan = tracer.spanBuilder("processPayment")
                .setSpanKind(SpanKind.CLIENT)
                .startSpan();

            try (Scope paymentScope = paymentSpan.makeCurrent()) {
                PaymentResult result = paymentClient.process(request.getPayment());
                paymentSpan.setAttribute("payment.transactionId", result.getTransactionId());
                paymentSpan.setAttribute("payment.amount", result.getAmount());
            } catch (PaymentException e) {
                paymentSpan.recordException(e);
                paymentSpan.setStatus(StatusCode.ERROR, e.getMessage());
                throw e;
            } finally {
                paymentSpan.end();
            }

            Order order = saveOrder(request);
            span.setAttribute("order.id", order.getId());

            // 이벤트 추가
            span.addEvent("order_created", Attributes.of(
                AttributeKey.stringKey("order.id"), order.getId()
            ));

            return order;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Python 구현 예시

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# 리소스 설정
resource = Resource(attributes={
    SERVICE_NAME: "order-service",
    "service.version": "1.0.0",
    "deployment.environment": "production"
})

# TracerProvider 설정
provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(
    OTLPSpanExporter(endpoint="http://otel-collector:4317")
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Tracer 획득
tracer = trace.get_tracer(__name__)

# 자동 계측
FastAPIInstrumentor.instrument()
RequestsInstrumentor().instrument()

# 수동 계측 예시
@app.post("/orders")
async def create_order(request: OrderRequest):
    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("order.customer_id", request.customer_id)
        span.set_attribute("order.item_count", len(request.items))

        # 재고 확인
        with tracer.start_as_current_span("check_inventory") as inventory_span:
            inventory_span.set_attribute("inventory.item_ids",
                                         [item.id for item in request.items])
            available = await inventory_service.check(request.items)

            if not available:
                inventory_span.set_status(Status(StatusCode.ERROR, "Insufficient inventory"))
                raise HTTPException(status_code=400, detail="Insufficient inventory")

        # 결제 처리
        with tracer.start_as_current_span("process_payment") as payment_span:
            try:
                result = await payment_service.process(request.payment)
                payment_span.set_attribute("payment.transaction_id", result.transaction_id)
            except PaymentError as e:
                payment_span.record_exception(e)
                payment_span.set_status(Status(StatusCode.ERROR, str(e)))
                raise

        order = await order_repository.save(request)
        span.add_event("order_created", {"order.id": order.id})

        return order
```

### OTel Collector 설정

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

  # Jaeger 포맷도 수신 가능
  jaeger:
    protocols:
      grpc:
        endpoint: 0.0.0.0:14250
      thrift_http:
        endpoint: 0.0.0.0:14268

  # Zipkin 포맷도 수신 가능
  zipkin:
    endpoint: 0.0.0.0:9411

processors:
  # 배치 처리
  batch:
    timeout: 5s
    send_batch_size: 1000
    send_batch_max_size: 1500

  # 메모리 제한
  memory_limiter:
    check_interval: 1s
    limit_mib: 2000
    spike_limit_mib: 400

  # 속성 추가/수정
  attributes:
    actions:
      - key: environment
        value: production
        action: upsert
      - key: cluster
        value: ${CLUSTER_NAME}
        action: insert

  # 샘플링 (10%)
  probabilistic_sampler:
    sampling_percentage: 10

  # 필터링
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.target"] == "/health"'
        - 'attributes["http.target"] == "/ready"'

exporters:
  # Jaeger로 내보내기
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true

  # Tempo로 내보내기
  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  # 디버그 출력
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp, jaeger, zipkin]
      processors: [memory_limiter, batch, attributes]
      exporters: [jaeger, otlp/tempo]

    # 에러 트레이스는 별도 파이프라인
    traces/errors:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger, logging]

  telemetry:
    logs:
      level: info
    metrics:
      address: 0.0.0.0:8888
```

---

## Jaeger

Uber에서 개발하고 CNCF에 기증한 분산 추적 시스템입니다. 2024년 11월 Jaeger 2.0이 출시되어 OpenTelemetry를 핵심으로 채택했습니다.

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Jaeger Architecture                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      Applications                                │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │   │
│  │  │  App 1   │  │  App 2   │  │  App 3   │  │  App 4   │        │   │
│  │  │ (OTel)   │  │ (OTel)   │  │ (OTel)   │  │ (OTel)   │        │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘        │   │
│  └───────┼──────────────┼──────────────┼──────────────┼────────────┘   │
│          │              │              │              │                │
│          └──────────────┼──────────────┼──────────────┘                │
│                         │              │                               │
│                         ▼              ▼                               │
│              ┌────────────────────────────────────┐                    │
│              │         Jaeger Collector           │                    │
│              │                                    │                    │
│              │  • 스팬 수신 (OTLP, Jaeger, Zipkin)│                    │
│              │  • 유효성 검사                     │                    │
│              │  • 인덱싱                         │                    │
│              │  • 변환                           │                    │
│              └─────────────┬──────────────────────┘                    │
│                            │                                           │
│                   ┌────────┴────────┐                                  │
│                   ▼                 ▼                                  │
│          ┌──────────────┐  ┌──────────────┐                           │
│          │    Kafka     │  │   Storage    │                           │
│          │  (Optional)  │  │              │                           │
│          │   버퍼링     │  │ • Elasticsearch│                          │
│          └──────────────┘  │ • Cassandra  │                           │
│                            │ • Badger     │                           │
│                            │ • Memory     │                           │
│                            └──────┬───────┘                           │
│                                   │                                    │
│                                   ▼                                    │
│                         ┌──────────────────┐                          │
│                         │   Jaeger Query   │                          │
│                         │   (API Server)   │                          │
│                         └────────┬─────────┘                          │
│                                  │                                     │
│                                  ▼                                     │
│                         ┌──────────────────┐                          │
│                         │    Jaeger UI     │                          │
│                         │   (Web 인터페이스)│                          │
│                         └──────────────────┘                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jaeger 2.0 변경사항 (2024년 11월)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Jaeger 1.x vs Jaeger 2.0                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Jaeger 1.x                        Jaeger 2.0                   │
│  ┌─────────────────────┐          ┌─────────────────────┐      │
│  │ jaeger-agent        │          │                     │      │
│  │ jaeger-collector    │    →     │ jaeger (단일 바이너리)│      │
│  │ jaeger-query        │          │ + OTel Collector    │      │
│  │ jaeger-ingester     │          │                     │      │
│  └─────────────────────┘          └─────────────────────┘      │
│                                                                 │
│  주요 변경:                                                     │
│  • OpenTelemetry Collector 기반으로 재구축                       │
│  • 단일 바이너리로 통합 (all-in-one)                            │
│  • jaeger-agent 제거 (OTel Collector 사용)                     │
│  • OTLP가 기본 프로토콜                                         │
│  • 2026년 1월: Jaeger v1 지원 종료                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Kubernetes 배포

```yaml
# jaeger-deployment.yaml (All-in-One for development)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:2.0
          ports:
            - containerPort: 4317   # OTLP gRPC
            - containerPort: 4318   # OTLP HTTP
            - containerPort: 14250  # Jaeger gRPC
            - containerPort: 14268  # Jaeger HTTP
            - containerPort: 16686  # UI
            - containerPort: 9411   # Zipkin
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
            - name: SPAN_STORAGE_TYPE
              value: "memory"  # 프로덕션에서는 elasticsearch 또는 cassandra
          resources:
            limits:
              memory: 1Gi
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: observability
spec:
  ports:
    - name: otlp-grpc
      port: 4317
    - name: otlp-http
      port: 4318
    - name: ui
      port: 16686
    - name: zipkin
      port: 9411
  selector:
    app: jaeger
```

**프로덕션 배포 (Elasticsearch)**
```yaml
# jaeger-production.yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: observability
spec:
  strategy: production
  collector:
    replicas: 3
    maxReplicas: 10
    resources:
      limits:
        cpu: 1
        memory: 2Gi
  query:
    replicas: 2
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: https://elasticsearch:9200
        index-prefix: jaeger
        tls:
          ca: /es/certificates/ca.crt
    secretName: jaeger-es-secret
  ingress:
    enabled: true
    hosts:
      - jaeger.example.com
```

### Jaeger API 활용

```python
# Jaeger Query API 사용 예시
import requests
from datetime import datetime, timedelta

JAEGER_QUERY_URL = "http://jaeger-query:16686"

def find_traces(service: str, operation: str = None,
                start_time: datetime = None, limit: int = 20):
    """트레이스 검색"""
    params = {
        "service": service,
        "limit": limit,
        "lookback": "1h"
    }

    if operation:
        params["operation"] = operation

    if start_time:
        params["start"] = int(start_time.timestamp() * 1_000_000)
        params["end"] = int(datetime.now().timestamp() * 1_000_000)

    response = requests.get(
        f"{JAEGER_QUERY_URL}/api/traces",
        params=params
    )
    return response.json()["data"]

def get_trace(trace_id: str):
    """특정 트레이스 조회"""
    response = requests.get(
        f"{JAEGER_QUERY_URL}/api/traces/{trace_id}"
    )
    return response.json()["data"][0]

def get_services():
    """서비스 목록 조회"""
    response = requests.get(f"{JAEGER_QUERY_URL}/api/services")
    return response.json()["data"]

def get_operations(service: str):
    """서비스의 오퍼레이션 목록"""
    response = requests.get(
        f"{JAEGER_QUERY_URL}/api/services/{service}/operations"
    )
    return response.json()["data"]

def find_error_traces(service: str, min_duration_ms: int = 0):
    """에러가 있는 트레이스 검색"""
    params = {
        "service": service,
        "tags": '{"error":"true"}',
        "minDuration": f"{min_duration_ms}ms",
        "limit": 100
    }

    response = requests.get(
        f"{JAEGER_QUERY_URL}/api/traces",
        params=params
    )
    return response.json()["data"]
```

---

## Zipkin

Twitter에서 개발한 분산 추적 시스템으로, Google Dapper 논문을 기반으로 합니다. 가벼운 설정과 빠른 도입이 장점입니다.

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Zipkin Architecture                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                      ┌─────────────────────────┐                       │
│                      │    Zipkin Server        │                       │
│                      │    (All-in-One)         │                       │
│                      │                         │                       │
│                      │  ┌─────────────────┐   │                       │
│                      │  │    Collector     │   │                       │
│                      │  │  (HTTP/Kafka)    │   │                       │
│                      │  └────────┬────────┘   │                       │
│                      │           │            │                       │
│                      │  ┌────────▼────────┐   │                       │
│                      │  │    Storage      │   │                       │
│                      │  │ • In-Memory     │   │                       │
│                      │  │ • MySQL         │   │                       │
│                      │  │ • Cassandra     │   │                       │
│                      │  │ • Elasticsearch │   │                       │
│                      │  └────────┬────────┘   │                       │
│                      │           │            │                       │
│                      │  ┌────────▼────────┐   │                       │
│                      │  │   Query API     │   │                       │
│                      │  │   + Web UI      │   │                       │
│                      │  └─────────────────┘   │                       │
│                      │                         │                       │
│                      └────────────▲────────────┘                       │
│                                   │                                    │
│         ┌─────────────────────────┼─────────────────────────┐         │
│         │                         │                         │         │
│  ┌──────┴──────┐           ┌──────┴──────┐           ┌──────┴──────┐  │
│  │   App 1     │           │   App 2     │           │   App 3     │  │
│  │ (Reporter)  │           │ (Reporter)  │           │ (Reporter)  │  │
│  └─────────────┘           └─────────────┘           └─────────────┘  │
│                                                                         │
│  특징:                                                                  │
│  • 단순한 All-in-One 아키텍처                                          │
│  • 빠른 설정, 작은 학습 곡선                                           │
│  • Spring Cloud Sleuth와 긴밀한 통합                                   │
│  • OpenTelemetry Adapter로 OTel 데이터 수신 가능                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Jaeger vs Zipkin 비교

| 특성 | Jaeger | Zipkin |
|------|--------|--------|
| **개발** | Uber (CNCF) | Twitter |
| **아키텍처** | 분산 컴포넌트 | All-in-One |
| **확장성** | 대규모에 적합 | 중소규모에 적합 |
| **UI** | 현대적, 풍부한 기능 | 단순, 직관적 |
| **스토리지** | ES, Cassandra, Badger | ES, Cassandra, MySQL |
| **OpenTelemetry** | 네이티브 지원 (v2.0) | Adapter 필요 |
| **Spring 통합** | 가능 | Spring Cloud Sleuth 긴밀 통합 |
| **복잡도** | 높음 | 낮음 |

### Zipkin 설정 (Spring Boot)

```yaml
# application.yml
spring:
  application:
    name: order-service

  # Zipkin 설정 (Spring Boot 3.x with Micrometer Tracing)
  zipkin:
    base-url: http://zipkin:9411

management:
  tracing:
    sampling:
      probability: 1.0  # 100% 샘플링 (개발용)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans

# 또는 OpenTelemetry 사용 시
# otel.exporter.zipkin.endpoint=http://zipkin:9411/api/v2/spans
```

```xml
<!-- pom.xml (Spring Boot 3.x) -->
<dependencies>
    <!-- Micrometer Tracing with Zipkin -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-otel</artifactId>
    </dependency>
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-exporter-zipkin</artifactId>
    </dependency>
</dependencies>
```

### Kubernetes 배포

```yaml
# zipkin-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zipkin
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zipkin
  template:
    metadata:
      labels:
        app: zipkin
    spec:
      containers:
        - name: zipkin
          image: openzipkin/zipkin:latest
          ports:
            - containerPort: 9411
          env:
            - name: STORAGE_TYPE
              value: "elasticsearch"
            - name: ES_HOSTS
              value: "http://elasticsearch:9200"
          resources:
            limits:
              memory: 512Mi
              cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: zipkin
  namespace: observability
spec:
  ports:
    - port: 9411
      targetPort: 9411
  selector:
    app: zipkin
```

---

## Trace Context (W3C)

W3C Trace Context는 분산 추적에서 컨텍스트를 전파하기 위한 표준 HTTP 헤더 형식입니다.

### 헤더 형식

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      W3C Trace Context Headers                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  traceparent: 00-{trace-id}-{parent-span-id}-{trace-flags}             │
│                                                                         │
│  예시: traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  00                           version (현재 00 고정)            │    │
│  │  ──                                                            │    │
│  │  0af7651916cd43dd8448eb211c80319c   trace-id (32 hex, 16 bytes)│    │
│  │  ──                                                            │    │
│  │  b7ad6b7169203331           parent-span-id (16 hex, 8 bytes)   │    │
│  │  ──                                                            │    │
│  │  01                         trace-flags (sampled=01)           │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  tracestate: {vendor}={value},{vendor2}={value2}                       │
│                                                                         │
│  예시: tracestate: congo=t61rcWkgMzE,rojo=00f067aa0ba902b7              │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  벤더별 추가 정보 전달                                          │    │
│  │  • 각 벤더는 자신의 키로 값 저장                                 │    │
│  │  • 최대 32개 항목, 512자                                        │    │
│  │  • 순서는 최근 수정된 순                                        │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 컨텍스트 전파 흐름

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Context Propagation Flow                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Client                                                                 │
│    │ Request (no headers)                                               │
│    ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Service A (API Gateway)                                          │  │
│  │                                                                   │  │
│  │ 1. trace-id 생성: abc123                                         │  │
│  │ 2. span-id 생성: span-A                                          │  │
│  │                                                                   │  │
│  │ Outgoing headers:                                                │  │
│  │ traceparent: 00-abc123-span-A-01                                 │  │
│  └───────────────────────────┬──────────────────────────────────────┘  │
│                              │                                          │
│    ┌─────────────────────────┴─────────────────────────┐               │
│    ▼                                                   ▼               │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐     │
│  │ Service B                    │  │ Service C                    │     │
│  │                              │  │                              │     │
│  │ Incoming:                    │  │ Incoming:                    │     │
│  │ traceparent:00-abc123-span-A-01│  │ traceparent:00-abc123-span-A-01│ │
│  │                              │  │                              │     │
│  │ 새 span-id: span-B           │  │ 새 span-id: span-C           │     │
│  │ parent: span-A               │  │ parent: span-A               │     │
│  │                              │  │                              │     │
│  │ Outgoing:                    │  │ Outgoing:                    │     │
│  │ traceparent:00-abc123-span-B-01│ │ traceparent:00-abc123-span-C-01│ │
│  └──────────────┬───────────────┘  └─────────────────────────────┘     │
│                 │                                                       │
│                 ▼                                                       │
│  ┌─────────────────────────────┐                                       │
│  │ Service D (Database)        │                                       │
│  │                              │                                       │
│  │ trace-id: abc123 (동일)      │                                       │
│  │ span-id: span-D              │                                       │
│  │ parent: span-B               │                                       │
│  └─────────────────────────────┘                                       │
│                                                                         │
│  결과: 모든 스팬이 trace-id abc123으로 연결됨                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 구현 예시

**Java (Spring)**
```java
@Component
public class TracePropagationFilter extends OncePerRequestFilter {

    private final Tracer tracer;
    private final TextMapPropagator propagator;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // 들어오는 컨텍스트 추출
        Context extractedContext = propagator.extract(
            Context.current(),
            request,
            new TextMapGetter<HttpServletRequest>() {
                @Override
                public Iterable<String> keys(HttpServletRequest carrier) {
                    return Collections.list(carrier.getHeaderNames());
                }

                @Override
                public String get(HttpServletRequest carrier, String key) {
                    return carrier.getHeader(key);
                }
            }
        );

        try (Scope scope = extractedContext.makeCurrent()) {
            filterChain.doFilter(request, response);
        }
    }
}

// HTTP 클라이언트에서 컨텍스트 주입
@Component
public class TracingRestTemplateCustomizer implements RestTemplateCustomizer {

    private final TextMapPropagator propagator;

    @Override
    public void customize(RestTemplate restTemplate) {
        restTemplate.getInterceptors().add((request, body, execution) -> {
            propagator.inject(
                Context.current(),
                request.getHeaders(),
                (headers, key, value) -> headers.set(key, value)
            );
            return execution.execute(request, body);
        });
    }
}
```

**Node.js**
```javascript
const { trace, context, propagation } = require('@opentelemetry/api');
const { W3CTraceContextPropagator } = require('@opentelemetry/core');

// Propagator 설정
propagation.setGlobalPropagator(new W3CTraceContextPropagator());

// 들어오는 요청에서 컨텍스트 추출
app.use((req, res, next) => {
  const extractedContext = propagation.extract(
    context.active(),
    req.headers
  );

  context.with(extractedContext, () => {
    next();
  });
});

// 외부 요청 시 컨텍스트 주입
async function callExternalService(url, data) {
  const headers = {};

  // 현재 컨텍스트를 헤더에 주입
  propagation.inject(context.active(), headers);

  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      ...headers  // traceparent, tracestate 포함
    },
    body: JSON.stringify(data)
  });

  return response.json();
}
```

---

## Span과 Trace

### 기본 개념

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Trace and Span Concepts                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Trace: 하나의 요청이 시스템을 통과하는 전체 경로                         │
│         = 연관된 모든 Span의 집합                                        │
│         = 동일한 trace_id를 공유                                        │
│                                                                         │
│  Span: 작업의 단위                                                      │
│        = 이름, 시작/종료 시간, 속성, 이벤트, 상태 포함                    │
│        = parent_span_id로 계층 구조 형성                                 │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                        Trace (trace_id: abc123)                  │  │
│  │                                                                   │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │ Root Span: HTTP GET /orders (span_id: A)                   │  │  │
│  │  │ ├──────────────────────────────────────────────────────────│  │  │
│  │  │ │ Duration: 850ms                                          │  │  │
│  │  │ │ Service: api-gateway                                     │  │  │
│  │  │ └──────────────────────────────────────────────────────────│  │  │
│  │  │                                                            │  │  │
│  │  │   ┌──────────────────────────────────────────────────┐    │  │  │
│  │  │   │ Child Span: Validate Token (span_id: B)          │    │  │  │
│  │  │   │ parent: A | Duration: 50ms | Service: auth       │    │  │  │
│  │  │   └──────────────────────────────────────────────────┘    │  │  │
│  │  │                                                            │  │  │
│  │  │   ┌──────────────────────────────────────────────────────┐│  │  │
│  │  │   │ Child Span: Get Order (span_id: C)                   ││  │  │
│  │  │   │ parent: A | Duration: 400ms | Service: order-service ││  │  │
│  │  │   │                                                      ││  │  │
│  │  │   │   ┌──────────────────────────────────────────────┐  ││  │  │
│  │  │   │   │ Grandchild: DB Query (span_id: D)            │  ││  │  │
│  │  │   │   │ parent: C | Duration: 120ms                  │  ││  │  │
│  │  │   │   └──────────────────────────────────────────────┘  ││  │  │
│  │  │   │                                                      ││  │  │
│  │  │   │   ┌──────────────────────────────────────────────┐  ││  │  │
│  │  │   │   │ Grandchild: Cache Lookup (span_id: E)        │  ││  │  │
│  │  │   │   │ parent: C | Duration: 5ms                    │  ││  │  │
│  │  │   │   └──────────────────────────────────────────────┘  ││  │  │
│  │  │   └──────────────────────────────────────────────────────┘│  │  │
│  │  │                                                            │  │  │
│  │  │   ┌──────────────────────────────────────────────────┐    │  │  │
│  │  │   │ Child Span: Serialize Response (span_id: F)      │    │  │  │
│  │  │   │ parent: A | Duration: 10ms                       │    │  │  │
│  │  │   └──────────────────────────────────────────────────┘    │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Span 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Span Structure                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  {                                                                      │
│    // 식별자                                                            │
│    "traceId": "0af7651916cd43dd8448eb211c80319c",                      │
│    "spanId": "b7ad6b7169203331",                                       │
│    "parentSpanId": "a1a2a3a4a5a6a7a8",  // 루트 스팬은 없음             │
│                                                                         │
│    // 기본 정보                                                         │
│    "operationName": "HTTP GET /api/orders/{id}",                       │
│    "serviceName": "order-service",                                     │
│    "startTime": "2024-01-15T10:30:00.000Z",                           │
│    "duration": 125000,  // microseconds                                │
│    "status": {                                                         │
│      "code": "OK",  // UNSET, OK, ERROR                               │
│      "message": ""                                                     │
│    },                                                                   │
│                                                                         │
│    // Span 종류                                                         │
│    "kind": "SERVER",  // CLIENT, SERVER, PRODUCER, CONSUMER, INTERNAL │
│                                                                         │
│    // 속성 (Attributes)                                                 │
│    "attributes": {                                                      │
│      "http.method": "GET",                                             │
│      "http.url": "/api/orders/12345",                                  │
│      "http.status_code": 200,                                          │
│      "http.response_content_length": 1234,                             │
│      "user.id": "user-789",                                            │
│      "order.id": "12345",                                              │
│      "db.system": "postgresql",                                        │
│      "db.statement": "SELECT * FROM orders WHERE id = $1"             │
│    },                                                                   │
│                                                                         │
│    // 이벤트 (Events) - 특정 시점의 발생 사항                            │
│    "events": [                                                          │
│      {                                                                  │
│        "name": "cache_miss",                                           │
│        "timestamp": "2024-01-15T10:30:00.005Z",                        │
│        "attributes": { "cache.key": "order:12345" }                    │
│      },                                                                 │
│      {                                                                  │
│        "name": "exception",                                            │
│        "timestamp": "2024-01-15T10:30:00.100Z",                        │
│        "attributes": {                                                  │
│          "exception.type": "TimeoutException",                         │
│          "exception.message": "Database connection timeout"           │
│        }                                                                │
│      }                                                                  │
│    ],                                                                   │
│                                                                         │
│    // 링크 (Links) - 다른 트레이스와의 관계                              │
│    "links": [                                                           │
│      {                                                                  │
│        "traceId": "other-trace-id",                                    │
│        "spanId": "other-span-id",                                      │
│        "attributes": { "link.type": "caused_by" }                      │
│      }                                                                  │
│    ]                                                                    │
│  }                                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Span Kind

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            Span Kinds                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  CLIENT                              SERVER                             │
│  ┌─────────────────────┐            ┌─────────────────────┐            │
│  │ 외부 서비스 호출     │     →     │ 들어오는 요청 처리   │            │
│  │ 요청 측              │            │ 응답 측              │            │
│  │                      │            │                      │            │
│  │ 예:                  │            │ 예:                  │            │
│  │ • HTTP 클라이언트    │            │ • HTTP 서버          │            │
│  │ • gRPC 클라이언트    │            │ • gRPC 서버          │            │
│  │ • DB 쿼리            │            │                      │            │
│  └─────────────────────┘            └─────────────────────┘            │
│                                                                         │
│  PRODUCER                            CONSUMER                           │
│  ┌─────────────────────┐            ┌─────────────────────┐            │
│  │ 메시지 발행          │     →     │ 메시지 소비          │            │
│  │ 비동기 전송          │            │ 비동기 수신          │            │
│  │                      │            │                      │            │
│  │ 예:                  │            │ 예:                  │            │
│  │ • Kafka 프로듀서     │            │ • Kafka 컨슈머       │            │
│  │ • RabbitMQ 발행      │            │ • RabbitMQ 구독      │            │
│  └─────────────────────┘            └─────────────────────┘            │
│                                                                         │
│  INTERNAL                                                               │
│  ┌─────────────────────┐                                               │
│  │ 내부 작업            │                                               │
│  │ 네트워크 통신 없음   │                                               │
│  │                      │                                               │
│  │ 예:                  │                                               │
│  │ • 함수 호출          │                                               │
│  │ • 로컬 캐시 접근     │                                               │
│  │ • 비즈니스 로직      │                                               │
│  └─────────────────────┘                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 시맨틱 컨벤션 (Semantic Conventions)

OpenTelemetry에서 정의한 표준 속성 이름입니다.

```java
// HTTP 스팬 속성
span.setAttribute(SemanticAttributes.HTTP_METHOD, "GET");
span.setAttribute(SemanticAttributes.HTTP_URL, "/api/orders");
span.setAttribute(SemanticAttributes.HTTP_STATUS_CODE, 200);
span.setAttribute(SemanticAttributes.HTTP_REQUEST_CONTENT_LENGTH, 1024);
span.setAttribute(SemanticAttributes.HTTP_RESPONSE_CONTENT_LENGTH, 2048);
span.setAttribute(SemanticAttributes.HTTP_USER_AGENT, userAgent);

// 데이터베이스 스팬 속성
span.setAttribute(SemanticAttributes.DB_SYSTEM, "postgresql");
span.setAttribute(SemanticAttributes.DB_NAME, "orders");
span.setAttribute(SemanticAttributes.DB_OPERATION, "SELECT");
span.setAttribute(SemanticAttributes.DB_STATEMENT, "SELECT * FROM orders WHERE id = $1");

// 메시징 스팬 속성
span.setAttribute(SemanticAttributes.MESSAGING_SYSTEM, "kafka");
span.setAttribute(SemanticAttributes.MESSAGING_DESTINATION, "orders-topic");
span.setAttribute(SemanticAttributes.MESSAGING_MESSAGE_ID, messageId);
span.setAttribute(SemanticAttributes.MESSAGING_OPERATION, "publish");

// RPC 스팬 속성
span.setAttribute(SemanticAttributes.RPC_SYSTEM, "grpc");
span.setAttribute(SemanticAttributes.RPC_SERVICE, "OrderService");
span.setAttribute(SemanticAttributes.RPC_METHOD, "GetOrder");

// 예외 기록
span.recordException(exception);
span.setAttribute(SemanticAttributes.EXCEPTION_TYPE, exception.getClass().getName());
span.setAttribute(SemanticAttributes.EXCEPTION_MESSAGE, exception.getMessage());
```

---

## 샘플링 전략

### 샘플링의 필요성

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Why Sampling?                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  문제 상황:                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ • 서비스당 1000 RPS × 10개 서비스 × 평균 5개 스팬                   │ │
│  │ • = 50,000 스팬/초 = 1.3조 스팬/월                                 │ │
│  │ • 스팬당 2KB = 2.6PB/월 저장 필요                                  │ │
│  │ • 비용과 성능 문제 발생                                            │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  해결: 샘플링                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ • 전체 트레이스 중 일부만 저장                                     │ │
│  │ • 중요한 트레이스 (에러, 느린 요청)는 항상 저장                     │ │
│  │ • 비용 절감 (10% 샘플링 = 90% 비용 절감)                          │ │
│  │ • 성능 영향 최소화                                                 │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  주의: 샘플링 결정은 Trace 시작점에서!                                   │
│       (모든 스팬이 같은 결정을 따라야 함)                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 샘플링 유형

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Sampling Strategies                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. Head-based Sampling (선행 샘플링)                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Request → [결정] → Span A → Span B → Span C → ... → 저장/버림   │   │
│  │            ↑ 시작점에서 결정                                     │   │
│  │                                                                  │   │
│  │  장점: 단순함, 오버헤드 낮음                                      │   │
│  │  단점: 에러/지연 발생 여부를 미리 알 수 없음                       │   │
│  │                                                                  │   │
│  │  종류:                                                           │   │
│  │  • Probability: 10% 확률로 샘플링                                │   │
│  │  • Rate Limiting: 초당 100개 트레이스만                          │   │
│  │  • Parent-based: 부모 결정 따름                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  2. Tail-based Sampling (후행 샘플링)                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Span A → Span B → Span C → [Collector] → [결정] → 저장/버림     │   │
│  │                                           ↑ 완료 후 결정         │   │
│  │                                                                  │   │
│  │  장점: 에러/지연 트레이스 보장                                    │   │
│  │  단점: 복잡함, 메모리 필요, 지연                                  │   │
│  │                                                                  │   │
│  │  결정 기준:                                                      │   │
│  │  • 에러 포함 트레이스 → 무조건 저장                               │   │
│  │  • 지연 시간 > 임계값 → 저장                                     │   │
│  │  • 특정 속성 포함 → 저장                                         │   │
│  │  • 나머지 → 확률적 샘플링                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Head-based 샘플링 구현

**Java (OpenTelemetry SDK)**
```java
import io.opentelemetry.sdk.trace.samplers.*;

@Configuration
public class SamplerConfig {

    @Bean
    public Sampler sampler() {
        // 1. 확률 기반 샘플링 (10%)
        Sampler probabilistic = Sampler.traceIdRatioBased(0.1);

        // 2. 부모 기반 샘플링
        //    - 부모가 샘플링되면 자식도 샘플링
        //    - 부모가 없으면 확률 적용
        Sampler parentBased = Sampler.parentBased(probabilistic);

        // 3. 커스텀 샘플러
        Sampler custom = new CustomSampler();

        return parentBased;
    }
}

public class CustomSampler implements Sampler {

    private final Sampler defaultSampler = Sampler.traceIdRatioBased(0.1);

    @Override
    public SamplingResult shouldSample(
            Context parentContext,
            String traceId,
            String name,
            SpanKind spanKind,
            Attributes attributes,
            List<LinkData> parentLinks) {

        // 특정 조건에서는 항상 샘플링
        if (name.contains("error") || name.contains("critical")) {
            return SamplingResult.recordAndSample();
        }

        // health check는 샘플링하지 않음
        String path = attributes.get(SemanticAttributes.HTTP_URL);
        if (path != null && (path.contains("/health") || path.contains("/ready"))) {
            return SamplingResult.drop();
        }

        // 기본 샘플링 적용
        return defaultSampler.shouldSample(
            parentContext, traceId, name, spanKind, attributes, parentLinks
        );
    }

    @Override
    public String getDescription() {
        return "CustomSampler";
    }
}
```

### Tail-based 샘플링 (OTel Collector)

```yaml
# otel-collector-config.yaml
processors:
  # Tail-based 샘플링 설정
  tail_sampling:
    decision_wait: 10s  # 결정 대기 시간 (모든 스팬 수집 대기)
    num_traces: 100000  # 메모리에 유지할 트레이스 수
    expected_new_traces_per_sec: 1000
    policies:
      # 정책 1: 에러 트레이스는 항상 샘플링
      - name: errors
        type: status_code
        status_code:
          status_codes: [ERROR]

      # 정책 2: 느린 트레이스 샘플링 (1초 초과)
      - name: slow-traces
        type: latency
        latency:
          threshold_ms: 1000

      # 정책 3: 특정 속성 포함 시 샘플링
      - name: important-users
        type: string_attribute
        string_attribute:
          key: user.tier
          values: [premium, enterprise]

      # 정책 4: 특정 서비스 100% 샘플링
      - name: critical-service
        type: string_attribute
        string_attribute:
          key: service.name
          values: [payment-service]

      # 정책 5: 나머지는 10% 확률 샘플링
      - name: probabilistic
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

      # 정책 조합: AND 조건
      - name: slow-api-gateway
        type: and
        and:
          and_sub_policy:
            - name: service-filter
              type: string_attribute
              string_attribute:
                key: service.name
                values: [api-gateway]
            - name: latency-filter
              type: latency
              latency:
                threshold_ms: 500

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [tail_sampling, batch]
      exporters: [jaeger]
```

### 샘플링 전략 선택 가이드

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  Sampling Strategy Selection Guide                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  트래픽 볼륨                                                             │
│  ├── 낮음 (<100 RPS): 100% 샘플링 가능                                  │
│  ├── 중간 (100-1000 RPS): Head-based 10-50%                            │
│  └── 높음 (>1000 RPS): Tail-based + Head-based 조합                    │
│                                                                         │
│  요구사항                                                                │
│  ├── 단순함/낮은 오버헤드: Head-based (Probability)                     │
│  ├── 에러 트레이스 보장: Tail-based (status_code)                       │
│  ├── 느린 요청 분석: Tail-based (latency)                               │
│  └── 비용 최적화: Tail-based + 낮은 기본 비율                           │
│                                                                         │
│  권장 조합:                                                              │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ 1. Head-based (Parent-based + 10% 확률)                           │ │
│  │    - 일관된 trace 샘플링                                          │ │
│  │                                                                    │ │
│  │ 2. Tail-based (Collector에서):                                    │ │
│  │    - 에러 → 100%                                                  │ │
│  │    - 지연 > 1초 → 100%                                            │ │
│  │    - Premium 사용자 → 100%                                        │ │
│  │    - 나머지 → 10%                                                 │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 실무 Best Practices

### 1. 효과적인 스팬 생성

```java
// ❌ 너무 세분화된 스팬
public void processOrder(Order order) {
    Span span1 = tracer.spanBuilder("validate-order").startSpan();
    validateOrder(order);
    span1.end();

    Span span2 = tracer.spanBuilder("check-field-1").startSpan();  // 너무 세분화
    checkField1(order);
    span2.end();

    Span span3 = tracer.spanBuilder("check-field-2").startSpan();  // 너무 세분화
    checkField2(order);
    span3.end();
}

// ✅ 적절한 수준의 스팬
public void processOrder(Order order) {
    Span span = tracer.spanBuilder("process-order")
        .setAttribute("order.id", order.getId())
        .startSpan();

    try (Scope scope = span.makeCurrent()) {
        // 빠른 작업은 이벤트로 기록
        span.addEvent("validation-started");
        validateOrder(order);
        span.addEvent("validation-completed");

        // 외부 호출은 별도 스팬
        Span paymentSpan = tracer.spanBuilder("call-payment-service")
            .setSpanKind(SpanKind.CLIENT)
            .startSpan();
        try (Scope paymentScope = paymentSpan.makeCurrent()) {
            paymentService.process(order);
        } finally {
            paymentSpan.end();
        }
    } finally {
        span.end();
    }
}
```

### 2. 의미있는 속성 추가

```java
// ❌ 불필요하거나 중복된 속성
span.setAttribute("timestamp", System.currentTimeMillis());  // 이미 포함됨
span.setAttribute("user_data", user.toString());  // 너무 큰 데이터
span.setAttribute("password", password);  // 민감 정보

// ✅ 의미있는 속성
span.setAttribute("order.id", order.getId());
span.setAttribute("order.total_amount", order.getTotalAmount());
span.setAttribute("order.item_count", order.getItems().size());
span.setAttribute("customer.tier", customer.getTier());  // 분석에 유용
span.setAttribute("payment.method", payment.getMethod());

// 에러 컨텍스트
span.setAttribute("error.retry_count", retryCount);
span.setAttribute("error.recoverable", true);
```

### 3. 비동기 처리에서의 컨텍스트 전파

```java
// ❌ 컨텍스트가 전파되지 않음
CompletableFuture.supplyAsync(() -> {
    // 현재 스팬 컨텍스트 없음!
    return processOrder(order);
});

// ✅ 컨텍스트 전파
Context context = Context.current();
CompletableFuture.supplyAsync(() -> {
    try (Scope scope = context.makeCurrent()) {
        return processOrder(order);
    }
});

// ✅ 또는 Executor 래핑
ExecutorService tracedExecutor = Context.taskWrapping(executorService);
tracedExecutor.submit(() -> processOrder(order));
```

```python
# Python 비동기
import asyncio
from opentelemetry import context

async def process_order(order):
    # 현재 컨텍스트 캡처
    ctx = context.get_current()

    async def process_with_context():
        token = context.attach(ctx)
        try:
            # 컨텍스트 내에서 작업
            await do_work(order)
        finally:
            context.detach(token)

    await asyncio.create_task(process_with_context())
```

### 4. 에러 처리

```java
try {
    result = processOrder(order);
    span.setStatus(StatusCode.OK);
} catch (ValidationException e) {
    // 비즈니스 예외: ERROR 상태 + 상세 정보
    span.recordException(e);
    span.setStatus(StatusCode.ERROR, "Validation failed: " + e.getMessage());
    span.setAttribute("error.type", "validation");
    span.setAttribute("error.field", e.getFieldName());
    throw e;
} catch (Exception e) {
    // 시스템 예외
    span.recordException(e);
    span.setStatus(StatusCode.ERROR, "Unexpected error");
    span.setAttribute("error.type", "system");
    throw e;
} finally {
    span.end();
}
```

### 5. 배치 작업 추적

```java
@Scheduled(cron = "0 0 * * * *")
public void hourlyBatchJob() {
    Span batchSpan = tracer.spanBuilder("hourly-batch-job")
        .setSpanKind(SpanKind.INTERNAL)
        .setAttribute("batch.scheduled_time", Instant.now().toString())
        .startSpan();

    try (Scope scope = batchSpan.makeCurrent()) {
        List<Order> orders = orderRepository.findPendingOrders();
        batchSpan.setAttribute("batch.total_items", orders.size());

        int processed = 0;
        int failed = 0;

        for (Order order : orders) {
            Span itemSpan = tracer.spanBuilder("process-batch-item")
                .setAttribute("order.id", order.getId())
                .startSpan();

            try (Scope itemScope = itemSpan.makeCurrent()) {
                processOrder(order);
                processed++;
            } catch (Exception e) {
                failed++;
                itemSpan.recordException(e);
                itemSpan.setStatus(StatusCode.ERROR);
            } finally {
                itemSpan.end();
            }
        }

        batchSpan.setAttribute("batch.processed", processed);
        batchSpan.setAttribute("batch.failed", failed);
        batchSpan.addEvent("batch_completed");
    } finally {
        batchSpan.end();
    }
}
```

### 6. 성능 최적화

```java
// 스팬 이름에 변수 사용 금지 (High Cardinality)
// ❌
tracer.spanBuilder("get-order-" + orderId).startSpan();

// ✅
tracer.spanBuilder("get-order")
    .setAttribute("order.id", orderId)
    .startSpan();

// 불필요한 스팬 생성 방지
if (tracer.isEnabled()) {  // 샘플링 여부 확인
    Span span = tracer.spanBuilder("expensive-operation").startSpan();
    // ...
}

// 대량 속성 설정 시 빌더 사용
Attributes attributes = Attributes.builder()
    .put("key1", "value1")
    .put("key2", "value2")
    .put("key3", "value3")
    .build();
span.setAllAttributes(attributes);
```

---

## 참고 자료

### 공식 문서
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Zipkin Documentation](https://zipkin.io/pages/documentation.html)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)

### 추천 자료
- [Google Dapper Paper](https://research.google/pubs/pub36356/)
- [Distributed Systems Observability - Cindy Sridharan](https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/)
- [OpenTelemetry Best Practices](https://betterstack.com/community/guides/observability/opentelemetry-best-practices/)

### 관련 문서
- [01-로깅](../01-로깅/README.md)
- [02-메트릭](../02-메트릭/README.md)
- [04-Prometheus-Grafana](../04-Prometheus-Grafana/README.md)

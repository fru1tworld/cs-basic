# 로깅 (Logging)

## 목차
1. [개요](#개요)
2. [구조화된 로깅 (Structured Logging)](#구조화된-로깅-structured-logging)
3. [로그 레벨](#로그-레벨)
4. [ELK Stack](#elk-stack)
5. [Fluentd/Fluent Bit](#fluentdfluent-bit)
6. [로그 집계 패턴](#로그-집계-패턴)
7. [Correlation ID](#correlation-id)
8. [실무 Best Practices](#실무-best-practices)
---

## 개요

로깅은 관찰성(Observability)의 세 가지 핵심 요소(Logs, Metrics, Traces) 중 하나로, 시스템에서 발생하는 이벤트를 시간순으로 기록하는 것을 의미합니다. 효과적인 로깅 전략은 디버깅, 감사(audit), 성능 분석, 보안 모니터링의 기반이 됩니다.

### 관찰성의 세 기둥 (Three Pillars of Observability)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Observability                               │
├─────────────────┬─────────────────┬─────────────────────────────┤
│      Logs       │     Metrics     │          Traces             │
├─────────────────┼─────────────────┼─────────────────────────────┤
│ - 이벤트 기록    │ - 수치 데이터    │ - 요청 흐름 추적             │
│ - 디버깅/감사    │ - 시계열 분석    │ - 분산 시스템 연결           │
│ - 상세 컨텍스트  │ - 알림/대시보드  │ - 지연 시간 분석             │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

## 구조화된 로깅 (Structured Logging)

### 비구조화 vs 구조화 로깅

**비구조화 로깅 (전통적 방식)**
```
2024-01-15 10:23:45 ERROR User login failed for user john@example.com from IP 192.168.1.100
```

**구조화된 로깅 (권장)**
```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "logger": "auth-service",
  "message": "User login failed",
  "context": {
    "user_email": "john@example.com",
    "ip_address": "192.168.1.100",
    "failure_reason": "invalid_password",
    "attempt_count": 3
  },
  "trace_id": "abc123def456",
  "span_id": "span789",
  "service": "auth-service",
  "environment": "production",
  "host": "auth-pod-abc123"
}
```

### 구조화된 로깅의 장점

| 장점 | 설명 |
|------|------|
| **검색 용이성** | 필드 기반 쿼리로 빠른 필터링 가능 |
| **파싱 자동화** | 정규식 없이 자동 파싱 |
| **일관성** | 표준화된 스키마로 분석 용이 |
| **컨텍스트 보존** | 메타데이터와 함께 의미 전달 |
| **도구 호환성** | ELK, Loki 등과 원활한 연동 |

### 언어별 구현 예시

**Java (Logback + SLF4J)**
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import net.logstash.logback.marker.Markers;

@Service
public class OrderService {
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);

    public Order createOrder(OrderRequest request) {
        // 구조화된 로깅 with MDC (Mapped Diagnostic Context)
        MDC.put("orderId", request.getOrderId());
        MDC.put("userId", request.getUserId());
        MDC.put("traceId", request.getTraceId());

        try {
            logger.info(
                Markers.append("orderAmount", request.getAmount())
                       .and(Markers.append("itemCount", request.getItems().size())),
                "Order creation started"
            );

            Order order = processOrder(request);

            logger.info(
                Markers.append("processingTimeMs", order.getProcessingTime()),
                "Order created successfully"
            );

            return order;
        } catch (Exception e) {
            logger.error(
                Markers.append("errorType", e.getClass().getSimpleName()),
                "Order creation failed",
                e
            );
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

**logback-spring.xml 설정**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>orderId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <customFields>{"service":"order-service","environment":"${SPRING_PROFILES_ACTIVE}"}</customFields>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON_CONSOLE"/>
    </root>
</configuration>
```

**Python (structlog)**
```python
import structlog
from structlog.contextvars import bind_contextvars, clear_contextvars

# structlog 설정
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ],
    logger_factory=structlog.PrintLoggerFactory(),
)

logger = structlog.get_logger()

def process_payment(payment_request):
    # 컨텍스트 바인딩
    bind_contextvars(
        payment_id=payment_request.id,
        user_id=payment_request.user_id,
        amount=payment_request.amount
    )

    try:
        logger.info("payment_processing_started", gateway="stripe")

        result = gateway.process(payment_request)

        logger.info(
            "payment_completed",
            transaction_id=result.transaction_id,
            processing_time_ms=result.processing_time
        )
        return result

    except PaymentError as e:
        logger.error(
            "payment_failed",
            error_code=e.code,
            error_message=str(e)
        )
        raise
    finally:
        clear_contextvars()
```

**Node.js (Pino)**
```javascript
const pino = require('pino');

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => ({ level: label }),
  },
  base: {
    service: 'api-gateway',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION,
  },
  timestamp: pino.stdTimeFunctions.isoTime,
});

// Express 미들웨어
const requestLogger = (req, res, next) => {
  const startTime = Date.now();
  const requestId = req.headers['x-request-id'] || uuid();

  // 요청별 child logger 생성
  req.log = logger.child({
    requestId,
    method: req.method,
    path: req.path,
    userAgent: req.headers['user-agent'],
    ip: req.ip,
  });

  req.log.info('request_received');

  res.on('finish', () => {
    req.log.info({
      statusCode: res.statusCode,
      responseTimeMs: Date.now() - startTime,
    }, 'request_completed');
  });

  next();
};

// 사용 예시
app.post('/api/orders', async (req, res) => {
  req.log.info({ orderData: req.body }, 'creating_order');

  try {
    const order = await orderService.create(req.body);
    req.log.info({ orderId: order.id }, 'order_created');
    res.json(order);
  } catch (error) {
    req.log.error({ err: error }, 'order_creation_failed');
    res.status(500).json({ error: 'Order creation failed' });
  }
});
```

---

## 로그 레벨

### 로그 레벨 계층 구조

```
┌─────────────────────────────────────────────────────────────────┐
│ FATAL  │ 시스템 크래시, 즉시 종료 필요                           │
├────────┼────────────────────────────────────────────────────────┤
│ ERROR  │ 오류 발생, 기능 동작 불가, 즉시 조치 필요                 │
├────────┼────────────────────────────────────────────────────────┤
│ WARN   │ 잠재적 문제, 주의 필요, 기능은 동작                      │
├────────┼────────────────────────────────────────────────────────┤
│ INFO   │ 주요 비즈니스 이벤트, 정상 동작 확인                     │
├────────┼────────────────────────────────────────────────────────┤
│ DEBUG  │ 상세 디버깅 정보, 개발/스테이징 환경용                   │
├────────┼────────────────────────────────────────────────────────┤
│ TRACE  │ 가장 상세한 정보, 코드 흐름 추적                         │
└─────────────────────────────────────────────────────────────────┘
          ▲
          │ 상세도 증가 (Verbosity)
          │
          ▼ 심각도 증가 (Severity)
```

### 각 레벨별 상세 가이드

#### TRACE
```java
// 코드 흐름의 매우 상세한 추적
logger.trace("Entering method calculateDiscount with params: userId={}, cartTotal={}",
             userId, cartTotal);

// 반복문 내부 상태
for (Item item : items) {
    logger.trace("Processing item: id={}, price={}, quantity={}",
                 item.getId(), item.getPrice(), item.getQuantity());
}
```
- **용도**: 메서드 진입/종료, 반복문 상태, 변수 값 변화
- **환경**: 로컬 개발, 특정 문제 디버깅 시에만
- **주의**: 프로덕션에서는 절대 활성화하지 않음

#### DEBUG
```java
// 개발 중 유용한 상세 정보
logger.debug("Cache lookup for key={}, hit={}, ttl={}ms",
             cacheKey, cacheHit, remainingTtl);

// 알고리즘 중간 결과
logger.debug("Discount calculation: baseDiscount={}, memberBonus={}, finalDiscount={}",
             baseDiscount, memberBonus, finalDiscount);

// 외부 서비스 응답 상세
logger.debug("External API response: status={}, body={}",
             response.getStatus(), response.getBody());
```
- **용도**: 캐시 동작, 알고리즘 결과, API 상세 응답
- **환경**: 개발, 스테이징
- **주의**: 프로덕션에서는 동적으로 활성화 가능하도록 구성

#### INFO
```java
// 비즈니스 이벤트 (항상 기록)
logger.info("Order completed: orderId={}, userId={}, totalAmount={}, itemCount={}",
            orderId, userId, totalAmount, itemCount);

// 시스템 상태 변화
logger.info("Application started on port {} in {} mode", port, profile);

// 스케줄 작업 실행
logger.info("Daily batch job started: jobId={}, targetDate={}", jobId, targetDate);

// 중요 설정 로드
logger.info("Configuration loaded: cacheEnabled={}, maxConnections={}",
            cacheEnabled, maxConnections);
```
- **용도**: 주요 비즈니스 이벤트, 애플리케이션 상태 변화
- **환경**: 모든 환경
- **기준**: "운영팀이 정상 동작을 확인하는 데 필요한 정보"

#### WARN
```java
// 복구 가능한 문제
logger.warn("Connection pool running low: available={}, max={}",
            availableConnections, maxConnections);

// Deprecated API 사용
logger.warn("Deprecated endpoint called: endpoint={}, suggestAlternative={}",
            "/api/v1/users", "/api/v2/users");

// 재시도 성공
logger.warn("Database connection restored after {} retries, downtime={}ms",
            retryCount, downtime);

// 임계값 접근
logger.warn("Memory usage approaching limit: current={}%, threshold={}%",
            currentUsage, threshold);
```
- **용도**: 잠재적 문제, 성능 저하, 임계값 접근
- **환경**: 모든 환경
- **액션**: 조사 필요하지만 즉시 조치는 불필요

#### ERROR
```java
// 기능 실패 (복구 불가)
logger.error("Payment processing failed: orderId={}, errorCode={}, errorMessage={}",
             orderId, errorCode, errorMessage, exception);

// 외부 서비스 연결 실패
logger.error("Failed to connect to payment gateway after {} attempts: gateway={}",
             maxRetries, gatewayUrl, exception);

// 데이터 무결성 문제
logger.error("Data integrity violation: expected={}, actual={}, table={}",
             expectedCount, actualCount, tableName);
```
- **용도**: 기능 실패, 예외 발생, 복구 불가능한 상태
- **환경**: 모든 환경
- **액션**: 즉시 조사 및 조치 필요, 알림 트리거

#### FATAL
```java
// 시스템 시작 불가
logger.fatal("Failed to connect to database on startup: url={}", dbUrl, exception);

// 필수 설정 누락
logger.fatal("Required configuration missing: missingKeys={}", missingKeys);

// 복구 불가능한 상태
logger.fatal("Unrecoverable error in core component: component={}", componentName);
```
- **용도**: 애플리케이션 크래시, 시스템 시작 실패
- **환경**: 모든 환경
- **액션**: 즉시 인시던트 대응 필요, PagerDuty 알림

### 환경별 로그 레벨 설정

```yaml
# application-dev.yml
logging:
  level:
    root: DEBUG
    com.mycompany.app: TRACE
    org.springframework: INFO
    org.hibernate.SQL: DEBUG

# application-staging.yml
logging:
  level:
    root: INFO
    com.mycompany.app: DEBUG
    org.springframework: WARN

# application-prod.yml
logging:
  level:
    root: WARN
    com.mycompany.app: INFO
    org.springframework: WARN
    org.hibernate: ERROR
```

---

## ELK Stack

### 아키텍처 개요

ELK Stack은 Elasticsearch, Logstash, Kibana의 조합으로, 로그 수집, 저장, 시각화를 위한 가장 널리 사용되는 솔루션입니다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           ELK Stack Architecture                        │
└─────────────────────────────────────────────────────────────────────────┘

 ┌──────────┐   ┌──────────┐   ┌──────────┐
 │  App 1   │   │  App 2   │   │  App 3   │
 │(Filebeat)│   │(Filebeat)│   │(Filebeat)│
 └────┬─────┘   └────┬─────┘   └────┬─────┘
      │              │              │
      └──────────────┼──────────────┘
                     │
                     ▼
            ┌────────────────┐
            │    Logstash    │  ← 수집, 파싱, 변환
            │   (Pipeline)   │
            └───────┬────────┘
                    │
                    ▼
            ┌────────────────┐
            │ Elasticsearch  │  ← 인덱싱, 저장, 검색
            │   (Cluster)    │
            └───────┬────────┘
                    │
                    ▼
            ┌────────────────┐
            │    Kibana      │  ← 시각화, 대시보드
            │  (Dashboard)   │
            └────────────────┘
```

### 각 컴포넌트 상세

#### Elasticsearch

분산 검색 및 분석 엔진으로, 로그 데이터의 저장과 검색을 담당합니다.

**인덱스 설정 예시**
```json
PUT /logs-2024.01
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "index.lifecycle.name": "logs-policy",
    "index.lifecycle.rollover_alias": "logs"
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" },
      "service": { "type": "keyword" },
      "trace_id": { "type": "keyword" },
      "span_id": { "type": "keyword" },
      "host": { "type": "keyword" },
      "response_time_ms": { "type": "integer" },
      "user_id": { "type": "keyword" },
      "error": {
        "properties": {
          "type": { "type": "keyword" },
          "message": { "type": "text" },
          "stack_trace": { "type": "text" }
        }
      }
    }
  }
}
```

**Index Lifecycle Management (ILM) 정책**
```json
PUT _ilm/policy/logs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          },
          "set_priority": {
            "priority": 100
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
          "set_priority": {
            "priority": 0
          }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### Logstash

데이터 파이프라인으로, 다양한 소스에서 데이터를 수집하고 변환합니다.

**logstash.conf 설정 예시**
```ruby
input {
  beats {
    port => 5044
    ssl => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key => "/etc/logstash/certs/logstash.key"
  }

  kafka {
    bootstrap_servers => "kafka:9092"
    topics => ["application-logs"]
    group_id => "logstash-consumer"
    codec => json
  }
}

filter {
  # JSON 파싱
  if [message] =~ /^\{/ {
    json {
      source => "message"
      target => "parsed"
    }
    mutate {
      rename => { "[parsed][level]" => "level" }
      rename => { "[parsed][message]" => "log_message" }
      rename => { "[parsed][trace_id]" => "trace_id" }
    }
  }

  # 타임스탬프 파싱
  date {
    match => ["[parsed][@timestamp]", "ISO8601"]
    target => "@timestamp"
  }

  # GeoIP 추가 (IP 기반 위치 정보)
  if [client_ip] {
    geoip {
      source => "client_ip"
      target => "geoip"
    }
  }

  # 민감 정보 마스킹
  mutate {
    gsub => [
      "log_message", "\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b", "[EMAIL_MASKED]",
      "log_message", "\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b", "[CARD_MASKED]"
    ]
  }

  # 사용자 정의 필드 추가
  mutate {
    add_field => {
      "environment" => "${ENVIRONMENT:production}"
      "cluster" => "${CLUSTER_NAME:default}"
    }
  }

  # 불필요한 필드 제거
  mutate {
    remove_field => ["parsed", "beat", "prospector"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[service]}-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "${ELASTICSEARCH_PASSWORD}"

    # 벌크 설정
    bulk_max_size => 1000
    flush_size => 500
  }

  # 에러 로그는 별도 인덱스
  if [level] == "ERROR" or [level] == "FATAL" {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "errors-%{+YYYY.MM.dd}"
    }
  }
}
```

#### Filebeat

경량 로그 수집기로, 각 서버에서 로그 파일을 읽어 전송합니다.

**filebeat.yml 설정**
```yaml
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'
    processors:
      - add_kubernetes_metadata:
          host: ${NODE_NAME}
          matchers:
            - logs_path:
                logs_path: "/var/lib/docker/containers/"
      - decode_json_fields:
          fields: ["message"]
          target: ""
          overwrite_keys: true

  - type: log
    enabled: true
    paths:
      - /var/log/app/*.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      service: my-application
    fields_under_root: true

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - drop_event:
      when:
        regexp:
          message: "^\\s*$"

output.logstash:
  hosts: ["logstash:5044"]
  ssl.certificate_authorities: ["/etc/filebeat/certs/ca.crt"]
  loadbalance: true
  bulk_max_size: 2048

# 또는 직접 Elasticsearch로
# output.elasticsearch:
#   hosts: ["elasticsearch:9200"]
#   index: "filebeat-%{+yyyy.MM.dd}"

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
```

#### Kibana

시각화 및 대시보드 도구입니다.

**KQL (Kibana Query Language) 예시**
```
# 에러 로그 검색
level: ERROR AND service: "payment-service"

# 특정 시간대 검색
@timestamp >= "2024-01-15T00:00:00" AND @timestamp < "2024-01-16T00:00:00"

# 응답 시간이 느린 요청
response_time_ms > 1000 AND level: INFO

# 특정 사용자의 모든 활동
user_id: "user123" AND (level: INFO OR level: WARN OR level: ERROR)

# Trace ID로 요청 추적
trace_id: "abc123def456"

# 와일드카드 검색
message: *timeout* AND service: order*

# 복합 조건
(level: ERROR OR level: FATAL) AND NOT service: "health-check" AND environment: "production"
```

---

## Fluentd/Fluent Bit

### Fluentd vs Fluent Bit 비교

```
┌─────────────────────────────────────────────────────────────────┐
│                 Fluentd vs Fluent Bit                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Fluent Bit (경량)              Fluentd (고기능)                │
│  ┌─────────────────┐           ┌─────────────────┐             │
│  │  ~450KB Memory  │           │  ~40MB Memory   │             │
│  │  C 언어 구현     │           │  Ruby + C 구현   │             │
│  │  Edge 수집용     │           │  중앙 집계용     │             │
│  │  빠른 처리       │           │  풍부한 플러그인 │             │
│  └────────┬────────┘           └────────┬────────┘             │
│           │                             │                       │
│           │ Kubernetes DaemonSet        │ Aggregator            │
│           │ 각 노드에서 실행             │ 중앙에서 처리          │
│           │                             │                       │
└─────────────────────────────────────────────────────────────────┘
```

| 특성 | Fluent Bit | Fluentd |
|------|-----------|---------|
| **메모리 사용량** | ~450KB | ~40MB |
| **언어** | C | Ruby + C |
| **플러그인 수** | ~100개 | ~1000개 |
| **용도** | 엣지 수집, IoT | 중앙 집계, 복잡한 처리 |
| **쿠버네티스** | DaemonSet 권장 | Deployment 권장 |

### EFK Stack 아키텍처 (Kubernetes)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │                    Node 1                                    │       │
│  │  ┌─────────┐ ┌─────────┐                                    │       │
│  │  │  Pod A  │ │  Pod B  │ ← Application Pods                 │       │
│  │  └────┬────┘ └────┬────┘                                    │       │
│  │       └───────┬───┘                                         │       │
│  │               ▼                                              │       │
│  │  ┌───────────────────┐                                      │       │
│  │  │   Fluent Bit      │ ← DaemonSet (각 노드에 1개)           │       │
│  │  │   (DaemonSet)     │                                      │       │
│  │  └─────────┬─────────┘                                      │       │
│  └────────────┼──────────────────────────────────────────────────┘     │
│               │                                                         │
│  ┌────────────┼──────────────────────────────────────────────────┐     │
│  │            ▼                    Node 2                         │     │
│  │  ┌───────────────────┐                                        │     │
│  │  │   Fluentd         │ ← Aggregator (선택적)                   │     │
│  │  │   (Deployment)    │                                        │     │
│  │  └─────────┬─────────┘                                        │     │
│  └────────────┼──────────────────────────────────────────────────┘     │
│               │                                                         │
│               ▼                                                         │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │                    Elasticsearch Cluster                       │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                     │     │
│  │  │  Data 1  │  │  Data 2  │  │  Data 3  │                     │     │
│  │  └──────────┘  └──────────┘  └──────────┘                     │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Fluent Bit 설정 (Kubernetes)

**fluent-bit-configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
        Health_Check  On

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     50MB
        Skip_Long_Lines   On
        DB                /var/log/flb_kube.db

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On
        Labels              On
        Annotations         Off

    [FILTER]
        Name          nest
        Match         kube.*
        Operation     lift
        Nested_under  kubernetes
        Add_prefix    kubernetes_

    [FILTER]
        Name   modify
        Match  kube.*
        Remove kubernetes_pod_id
        Remove kubernetes_docker_id
        Remove stream
        Remove logtag

    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch.logging.svc.cluster.local
        Port            9200
        Index           kubernetes-logs
        Type            _doc
        Logstash_Format On
        Logstash_Prefix kubernetes
        Retry_Limit     False
        Time_Key        @timestamp
        Replace_Dots    On
        tls             Off
        tls.verify      Off

    [OUTPUT]
        Name          stdout
        Match         *
        Format        json_lines

  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On

    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S.%LZ

    [PARSER]
        Name        syslog
        Format      regex
        Regex       ^<(?<pri>[0-9]+)>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key    time
        Time_Format %b %d %H:%M:%S
```

**fluent-bit-daemonset.yaml**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          ports:
            - containerPort: 2020
          resources:
            limits:
              memory: 200Mi
              cpu: 200m
            requests:
              memory: 100Mi
              cpu: 100m
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluent-bit/etc/
          livenessProbe:
            httpGet:
              path: /
              port: 2020
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /api/v1/health
              port: 2020
            initialDelaySeconds: 10
            periodSeconds: 10
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config
```

### Fluentd 설정 (Aggregator)

**fluentd.conf**
```ruby
# 입력: Fluent Bit에서 전달받음
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

# 입력: HTTP로도 받을 수 있음
<source>
  @type http
  port 9880
  bind 0.0.0.0
  body_size_limit 32m
  keepalive_timeout 10s
</source>

# 필터: 레코드 변환
<filter **>
  @type record_transformer
  enable_ruby true
  <record>
    hostname "#{Socket.gethostname}"
    processed_at ${time.strftime('%Y-%m-%dT%H:%M:%S%z')}
    environment "#{ENV['ENVIRONMENT'] || 'unknown'}"
  </record>
</filter>

# 필터: 민감 정보 마스킹
<filter **>
  @type record_transformer
  enable_ruby true
  <record>
    message ${record["message"].gsub(/password=\S+/, 'password=***')}
  </record>
</filter>

# 버퍼 설정과 함께 Elasticsearch로 출력
<match kubernetes.**>
  @type elasticsearch
  host elasticsearch.logging.svc.cluster.local
  port 9200
  logstash_format true
  logstash_prefix kubernetes
  include_timestamp true

  <buffer>
    @type file
    path /var/log/fluentd-buffers/kubernetes.buffer
    flush_mode interval
    flush_interval 5s
    flush_thread_count 2
    retry_type exponential_backoff
    retry_forever true
    retry_max_interval 30
    chunk_limit_size 10M
    queue_limit_length 8
    overflow_action block
  </buffer>
</match>

# 에러 로그는 별도 처리
<match error.**>
  @type copy

  <store>
    @type elasticsearch
    host elasticsearch.logging.svc.cluster.local
    port 9200
    index_name errors
    include_timestamp true
  </store>

  <store>
    @type slack
    webhook_url "#{ENV['SLACK_WEBHOOK_URL']}"
    channel "#alerts"
    username fluentd
    icon_emoji :warning:
    flush_interval 10s
  </store>
</match>
```

---

## 로그 집계 패턴

### 1. Direct Push 패턴

```
┌──────────────────────────────────────────────────────────────────┐
│                     Direct Push Pattern                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                      │
│  │  App 1  │    │  App 2  │    │  App 3  │                      │
│  │         │    │         │    │         │                      │
│  └────┬────┘    └────┬────┘    └────┬────┘                      │
│       │              │              │                            │
│       │         Direct HTTP/TCP     │                            │
│       └──────────────┼──────────────┘                            │
│                      │                                           │
│                      ▼                                           │
│              ┌──────────────┐                                    │
│              │ Log Backend  │                                    │
│              │(ES/Loki/etc) │                                    │
│              └──────────────┘                                    │
│                                                                  │
│  장점: 단순함, 즉시 전달                                          │
│  단점: 애플리케이션 부하, 네트워크 의존성, 백프레셔 처리 어려움       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 2. Agent 기반 패턴 (권장)

```
┌──────────────────────────────────────────────────────────────────┐
│                    Agent-based Pattern                           │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────┐            │
│  │                    Host/Node                     │            │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │            │
│  │  │  App 1  │  │  App 2  │  │  App 3  │          │            │
│  │  │  (log)  │  │  (log)  │  │  (log)  │          │            │
│  │  └────┬────┘  └────┬────┘  └────┬────┘          │            │
│  │       │            │            │                │            │
│  │       └────────────┼────────────┘                │            │
│  │                    ▼                             │            │
│  │            ┌──────────────┐                      │            │
│  │            │ Log Agent    │ (Filebeat/Fluent Bit)│            │
│  │            │ (DaemonSet)  │                      │            │
│  │            └──────┬───────┘                      │            │
│  └───────────────────┼──────────────────────────────┘            │
│                      │                                           │
│                      ▼                                           │
│              ┌──────────────┐                                    │
│              │ Log Backend  │                                    │
│              └──────────────┘                                    │
│                                                                  │
│  장점: 애플리케이션 분리, 로컬 버퍼링, 재전송                       │
│  단점: 에이전트 관리 필요                                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 3. Aggregator 패턴 (대규모)

```
┌──────────────────────────────────────────────────────────────────┐
│                    Aggregator Pattern                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Node 1                    Node 2                    Node 3      │
│  ┌──────────────┐         ┌──────────────┐         ┌────────────┐│
│  │ App → Agent  │         │ App → Agent  │         │ App → Agent││
│  └──────┬───────┘         └──────┬───────┘         └──────┬─────┘│
│         │                        │                        │      │
│         └────────────────────────┼────────────────────────┘      │
│                                  │                               │
│                                  ▼                               │
│                    ┌─────────────────────────┐                   │
│                    │      Aggregator         │                   │
│                    │ (Fluentd/Logstash)      │                   │
│                    │ - 필터링                │                   │
│                    │ - 변환                  │                   │
│                    │ - 라우팅                │                   │
│                    │ - 버퍼링                │                   │
│                    └───────────┬─────────────┘                   │
│                                │                                 │
│              ┌─────────────────┼─────────────────┐               │
│              │                 │                 │               │
│              ▼                 ▼                 ▼               │
│      ┌─────────────┐   ┌─────────────┐   ┌─────────────┐        │
│      │ Elasticsearch│   │    Kafka    │   │    S3      │        │
│      │  (실시간)    │   │  (스트림)   │   │ (아카이브) │        │
│      └─────────────┘   └─────────────┘   └─────────────┘        │
│                                                                  │
│  장점: 중앙 처리, 다중 대상, 복잡한 변환                           │
│  단점: SPOF 위험, 지연 증가                                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 4. Sidecar 패턴 (Kubernetes)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging-sidecar
spec:
  containers:
    # 메인 애플리케이션
    - name: application
      image: myapp:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

    # 로깅 사이드카
    - name: log-shipper
      image: fluent/fluent-bit:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/
      resources:
        limits:
          memory: 100Mi
          cpu: 100m

  volumes:
    - name: logs
      emptyDir: {}
    - name: fluent-bit-config
      configMap:
        name: fluent-bit-sidecar-config
```

### 패턴 선택 가이드

| 패턴 | 사용 시점 | 복잡도 | 확장성 |
|------|----------|--------|--------|
| **Direct Push** | 소규모, 단순 환경 | 낮음 | 낮음 |
| **Agent** | 일반적인 프로덕션 | 중간 | 높음 |
| **Aggregator** | 대규모, 복잡한 변환 | 높음 | 매우 높음 |
| **Sidecar** | Kubernetes, 격리 필요 | 중간 | 높음 |

---

## Correlation ID

### 개념

Correlation ID(상관 ID)는 분산 시스템에서 단일 사용자 요청이 여러 서비스를 거치며 처리될 때, 모든 관련 로그를 연결하는 고유 식별자입니다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Request Flow with Correlation ID                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Client                                                                 │
│    │                                                                    │
│    │ Request (no correlation ID)                                        │
│    ▼                                                                    │
│  ┌─────────────────────┐                                               │
│  │    API Gateway      │ ← Correlation ID 생성: "req-abc123"           │
│  │ X-Correlation-ID:   │                                               │
│  │   req-abc123        │                                               │
│  └──────────┬──────────┘                                               │
│             │                                                           │
│    ┌────────┴────────┐                                                 │
│    │                 │                                                 │
│    ▼                 ▼                                                 │
│  ┌──────────┐    ┌──────────┐                                         │
│  │ Order    │    │ Inventory│  ← 동일한 Correlation ID 전파            │
│  │ Service  │    │ Service  │                                         │
│  │req-abc123│    │req-abc123│                                         │
│  └────┬─────┘    └────┬─────┘                                         │
│       │               │                                                │
│       ▼               ▼                                                │
│  ┌──────────┐    ┌──────────┐                                         │
│  │ Payment  │    │ Shipping │                                         │
│  │ Service  │    │ Service  │                                         │
│  │req-abc123│    │req-abc123│                                         │
│  └──────────┘    └──────────┘                                         │
│                                                                         │
│  모든 서비스의 로그에서 "req-abc123"으로 검색하면                        │
│  해당 요청의 전체 흐름을 추적할 수 있음                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 구현 예시

#### Spring Boot (Java)

**CorrelationIdFilter.java**
```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    public static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    public static final String CORRELATION_ID_LOG_KEY = "correlationId";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        try {
            String correlationId = extractOrGenerateCorrelationId(request);

            // MDC에 설정 (로그에 자동 포함)
            MDC.put(CORRELATION_ID_LOG_KEY, correlationId);

            // 응답 헤더에도 포함
            response.setHeader(CORRELATION_ID_HEADER, correlationId);

            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(CORRELATION_ID_LOG_KEY);
        }
    }

    private String extractOrGenerateCorrelationId(HttpServletRequest request) {
        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isEmpty()) {
            correlationId = UUID.randomUUID().toString();
        }
        return correlationId;
    }
}
```

**RestTemplate 인터셉터**
```java
@Component
public class CorrelationIdInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(
            HttpRequest request,
            byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        String correlationId = MDC.get(CorrelationIdFilter.CORRELATION_ID_LOG_KEY);
        if (correlationId != null) {
            request.getHeaders().add(
                CorrelationIdFilter.CORRELATION_ID_HEADER,
                correlationId
            );
        }

        return execution.execute(request, body);
    }
}

@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(CorrelationIdInterceptor interceptor) {
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setInterceptors(
            Collections.singletonList(interceptor)
        );
        return restTemplate;
    }
}
```

**WebClient 설정 (Spring WebFlux)**
```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .filter((request, next) -> {
                return Mono.deferContextual(contextView -> {
                    String correlationId = contextView.getOrDefault(
                        "correlationId",
                        UUID.randomUUID().toString()
                    );

                    ClientRequest newRequest = ClientRequest.from(request)
                        .header("X-Correlation-ID", correlationId)
                        .build();

                    return next.exchange(newRequest);
                });
            })
            .build();
    }
}
```

#### Node.js (Express)

```javascript
const { v4: uuidv4 } = require('uuid');
const asyncLocalStorage = require('async_hooks').AsyncLocalStorage;

// AsyncLocalStorage 인스턴스
const correlationContext = new asyncLocalStorage.AsyncLocalStorage();

// 미들웨어
const correlationIdMiddleware = (req, res, next) => {
  const correlationId = req.headers['x-correlation-id'] || uuidv4();

  // 응답 헤더에 포함
  res.setHeader('X-Correlation-ID', correlationId);

  // 요청 객체에 저장
  req.correlationId = correlationId;

  // AsyncLocalStorage에 저장 (비동기 컨텍스트 유지)
  correlationContext.run({ correlationId }, () => {
    next();
  });
};

// 로거 설정
const createLogger = () => {
  return {
    log: (level, message, meta = {}) => {
      const store = correlationContext.getStore();
      const correlationId = store?.correlationId || 'unknown';

      console.log(JSON.stringify({
        timestamp: new Date().toISOString(),
        level,
        correlationId,
        message,
        ...meta
      }));
    },
    info: (message, meta) => this.log('info', message, meta),
    error: (message, meta) => this.log('error', message, meta),
  };
};

// 외부 API 호출 시 전파
const axios = require('axios');

const createAxiosInstance = () => {
  const instance = axios.create();

  instance.interceptors.request.use((config) => {
    const store = correlationContext.getStore();
    if (store?.correlationId) {
      config.headers['X-Correlation-ID'] = store.correlationId;
    }
    return config;
  });

  return instance;
};

// 사용 예시
app.use(correlationIdMiddleware);

app.get('/orders/:id', async (req, res) => {
  const logger = createLogger();
  const httpClient = createAxiosInstance();

  logger.info('Fetching order', { orderId: req.params.id });

  try {
    // 다른 서비스 호출 (Correlation ID 자동 전파)
    const inventory = await httpClient.get(
      `http://inventory-service/check/${req.params.id}`
    );

    logger.info('Inventory checked', { available: inventory.data.available });

    res.json({ order: {}, inventory: inventory.data });
  } catch (error) {
    logger.error('Failed to fetch order', { error: error.message });
    res.status(500).json({ error: 'Internal error' });
  }
});
```

#### Python (FastAPI)

```python
import uuid
from contextvars import ContextVar
from fastapi import FastAPI, Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import httpx
import structlog

# Context Variable for Correlation ID
correlation_id_ctx: ContextVar[str] = ContextVar('correlation_id', default='')

class CorrelationIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        correlation_id = request.headers.get('X-Correlation-ID', str(uuid.uuid4()))
        correlation_id_ctx.set(correlation_id)

        response: Response = await call_next(request)
        response.headers['X-Correlation-ID'] = correlation_id

        return response

# Structlog 프로세서
def add_correlation_id(logger, method_name, event_dict):
    event_dict['correlation_id'] = correlation_id_ctx.get('')
    return event_dict

structlog.configure(
    processors=[
        add_correlation_id,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()

app = FastAPI()
app.add_middleware(CorrelationIdMiddleware)

# HTTP 클라이언트 with Correlation ID
class CorrelatedHttpClient:
    def __init__(self):
        self.client = httpx.AsyncClient()

    async def get(self, url: str, **kwargs):
        headers = kwargs.pop('headers', {})
        headers['X-Correlation-ID'] = correlation_id_ctx.get('')
        return await self.client.get(url, headers=headers, **kwargs)

    async def post(self, url: str, **kwargs):
        headers = kwargs.pop('headers', {})
        headers['X-Correlation-ID'] = correlation_id_ctx.get('')
        return await self.client.post(url, headers=headers, **kwargs)

http_client = CorrelatedHttpClient()

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    logger.info("fetching_order", order_id=order_id)

    # 다른 서비스 호출 (Correlation ID 자동 전파)
    inventory_response = await http_client.get(
        f"http://inventory-service/check/{order_id}"
    )

    logger.info("inventory_checked", available=inventory_response.json()["available"])

    return {"order_id": order_id, "inventory": inventory_response.json()}
```

### Correlation ID vs Trace ID

| 구분 | Correlation ID | Trace ID |
|------|---------------|----------|
| **목적** | 비즈니스 요청 추적 | 분산 추적 시스템용 |
| **생성** | 애플리케이션 레벨 | 추적 라이브러리 |
| **포맷** | UUID (자유) | W3C Trace Context 표준 |
| **범위** | 하나의 비즈니스 트랜잭션 | 기술적 요청 체인 |
| **사용** | 로그 검색, 디버깅 | Jaeger, Zipkin 시각화 |

---

## 실무 Best Practices

### 1. 민감 정보 처리

```java
// 잘못된 예시 - 민감 정보 노출
logger.info("User login: email={}, password={}", email, password);  // ❌

// 올바른 예시 - 민감 정보 마스킹
logger.info("User login: email={}", maskEmail(email));  // ✅

private String maskEmail(String email) {
    if (email == null) return null;
    int atIndex = email.indexOf('@');
    if (atIndex <= 2) return "***" + email.substring(atIndex);
    return email.substring(0, 2) + "***" + email.substring(atIndex);
}

// 패턴 기반 자동 마스킹 (Logback)
// logback.xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <fieldNames>
        <message>[ignore]</message>
    </fieldNames>
    <provider class="net.logstash.logback.composite.loggingevent.LoggingEventPatternJsonProvider">
        <pattern>
            {
                "message": "#asJson{%replace(%msg){'password=\\S+', 'password=***'}}"
            }
        </pattern>
    </provider>
</encoder>
```

### 2. 성능 고려사항

```java
// 잘못된 예시 - 항상 문자열 연결 수행
logger.debug("Complex object: " + expensiveToString());  // ❌

// 올바른 예시 - 조건부 실행
if (logger.isDebugEnabled()) {  // ✅
    logger.debug("Complex object: {}", expensiveToString());
}

// 또는 람다 사용 (Slf4j 2.0+)
logger.atDebug()
    .addArgument(() -> expensiveToString())  // 지연 평가
    .log("Complex object: {}");
```

### 3. 예외 로깅

```java
// 잘못된 예시 - 스택 트레이스 없음
logger.error("Error occurred: " + e.getMessage());  // ❌

// 올바른 예시 - 전체 예외 포함
logger.error("Order processing failed: orderId={}", orderId, e);  // ✅

// 예외 체인 보존
try {
    processPayment(order);
} catch (PaymentException e) {
    throw new OrderProcessingException("Payment failed for order: " + orderId, e);
}
```

### 4. 구조화된 로그 스키마

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Application Log Schema",
  "type": "object",
  "required": ["timestamp", "level", "service", "message"],
  "properties": {
    "timestamp": {
      "type": "string",
      "format": "date-time"
    },
    "level": {
      "type": "string",
      "enum": ["TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"]
    },
    "service": {
      "type": "string"
    },
    "message": {
      "type": "string"
    },
    "correlation_id": {
      "type": "string",
      "format": "uuid"
    },
    "trace_id": {
      "type": "string"
    },
    "span_id": {
      "type": "string"
    },
    "user_id": {
      "type": "string"
    },
    "duration_ms": {
      "type": "integer",
      "minimum": 0
    },
    "error": {
      "type": "object",
      "properties": {
        "type": {"type": "string"},
        "message": {"type": "string"},
        "stack_trace": {"type": "string"}
      }
    },
    "context": {
      "type": "object",
      "additionalProperties": true
    }
  }
}
```

### 5. 로그 보관 정책

| 환경 | Hot (실시간 검색) | Warm (분석용) | Cold (아카이브) | 삭제 |
|------|-------------------|--------------|----------------|------|
| **개발** | 3일 | - | - | 7일 |
| **스테이징** | 7일 | - | - | 14일 |
| **프로덕션** | 30일 | 90일 | 365일 | 3년 |
| **감사 로그** | 30일 | 365일 | 7년 | 10년 |

---

## 참고 자료

### 공식 문서
- [Elasticsearch Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Fluent Bit Documentation](https://docs.fluentbit.io/)
- [Logback Manual](https://logback.qos.ch/manual/)
- [OpenTelemetry Logging](https://opentelemetry.io/docs/specs/otel/logs/)

### 추천 도서
- "Distributed Systems Observability" - Cindy Sridharan
- "Site Reliability Engineering" - Google SRE Book

### 관련 문서
- [02-메트릭](../02-메트릭/README.md)
- [03-분산-추적](../03-분산-추적/README.md)
- [04-Prometheus-Grafana](../04-Prometheus-Grafana/README.md)

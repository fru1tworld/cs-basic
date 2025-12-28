# 메트릭 (Metrics)

## 목차
1. [개요](#개요)
2. [4가지 골든 시그널 (Four Golden Signals)](#4가지-골든-시그널-four-golden-signals)
3. [RED 방법론](#red-방법론)
4. [USE 방법론](#use-방법론)
5. [메트릭 타입](#메트릭-타입)
6. [메트릭 수집 패턴](#메트릭-수집-패턴)
7. [메트릭 네이밍 컨벤션](#메트릭-네이밍-컨벤션)
8. [실무 Best Practices](#실무-best-practices)
---

## 개요

메트릭은 시스템의 상태를 수치로 표현한 시계열 데이터입니다. 로그가 개별 이벤트를 기록한다면, 메트릭은 집계된 수치를 통해 시스템의 전반적인 건강 상태와 성능을 파악하는 데 사용됩니다.

### 메트릭의 특징

```
┌─────────────────────────────────────────────────────────────────┐
│                    Metrics Characteristics                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  수치 기반 (Numeric)         시계열 (Time-Series)               │
│  ┌─────────────────┐         ┌─────────────────┐                │
│  │ cpu_usage: 75%  │         │ 10:00 → 75%    │                │
│  │ memory_mb: 1024 │         │ 10:01 → 78%    │                │
│  │ requests: 1500  │         │ 10:02 → 72%    │                │
│  └─────────────────┘         └─────────────────┘                │
│                                                                 │
│  집계 가능 (Aggregatable)    저장 효율 (Storage Efficient)       │
│  ┌─────────────────┐         ┌─────────────────┐                │
│  │ avg, sum, max   │         │ 로그: 1TB/day  │                │
│  │ percentile      │         │ 메트릭: 10GB/day│                │
│  │ rate, increase  │         │ (동일 시스템)   │                │
│  └─────────────────┘         └─────────────────┘                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Logs vs Metrics vs Traces

| 특성 | Logs | Metrics | Traces |
|------|------|---------|--------|
| **데이터 형태** | 텍스트/JSON | 숫자 | 구조화된 스팬 |
| **용도** | 디버깅, 감사 | 모니터링, 알림 | 요청 흐름 분석 |
| **저장 비용** | 높음 | 낮음 | 중간 |
| **쿼리 속도** | 느림 | 빠름 | 중간 |
| **상세도** | 높음 | 낮음 | 중간 |
| **집계** | 어려움 | 용이 | 중간 |

---

## 4가지 골든 시그널 (Four Golden Signals)

Google SRE Book에서 제안한 서비스 모니터링의 4가지 핵심 지표입니다.

### 개념도

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Four Golden Signals                                 │
│                  (Google SRE Book 권장)                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐    ┌─────────────────┐                            │
│  │   1. Latency    │    │   2. Traffic    │                            │
│  │   (지연 시간)    │    │   (트래픽)       │                            │
│  │                 │    │                 │                            │
│  │ 요청 처리 시간   │    │ 시스템 수요량    │                            │
│  │ - p50, p95, p99 │    │ - RPS           │                            │
│  │ - 성공 vs 실패   │    │ - Concurrent    │                            │
│  └─────────────────┘    └─────────────────┘                            │
│                                                                         │
│  ┌─────────────────┐    ┌─────────────────┐                            │
│  │   3. Errors     │    │  4. Saturation  │                            │
│  │   (에러율)       │    │   (포화도)       │                            │
│  │                 │    │                 │                            │
│  │ 실패 요청 비율   │    │ 리소스 활용도    │                            │
│  │ - 5xx 에러      │    │ - CPU, Memory   │                            │
│  │ - 타임아웃      │    │ - Disk, Network │                            │
│  └─────────────────┘    └─────────────────┘                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. Latency (지연 시간)

요청을 처리하는 데 걸리는 시간입니다. 성공한 요청과 실패한 요청의 지연 시간을 구분하는 것이 중요합니다.

**Prometheus 메트릭 예시**
```yaml
# Histogram 타입으로 지연 시간 측정
- name: http_request_duration_seconds
  type: histogram
  labels:
    - method     # GET, POST, PUT, DELETE
    - path       # /api/users, /api/orders
    - status     # 2xx, 4xx, 5xx

# 버킷 설정 (초 단위)
buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
```

**PromQL 쿼리**
```promql
# p50 (중앙값)
histogram_quantile(0.5,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
)

# p95 지연 시간
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, path)
)

# p99 지연 시간 (서비스별)
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{service="api-gateway"}[5m])) by (le)
)

# 성공 요청만의 지연 시간
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{status=~"2.."}[5m])) by (le)
)

# 에러 요청의 지연 시간 (보통 더 빠름 - 조기 실패)
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket{status=~"5.."}[5m])) by (le)
)
```

**Java (Micrometer) 구현**
```java
@Component
public class LatencyMetrics {

    private final Timer httpRequestTimer;

    public LatencyMetrics(MeterRegistry registry) {
        this.httpRequestTimer = Timer.builder("http.request.duration")
            .description("HTTP request latency")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .serviceLevelObjectives(
                Duration.ofMillis(100),
                Duration.ofMillis(500),
                Duration.ofSeconds(1)
            )
            .register(registry);
    }

    public void recordRequest(String method, String path, int status, long durationMs) {
        Timer.builder("http.request.duration")
            .tag("method", method)
            .tag("path", normalizePath(path))
            .tag("status", String.valueOf(status / 100) + "xx")
            .register(registry)
            .record(durationMs, TimeUnit.MILLISECONDS);
    }

    // 경로 정규화 (cardinality 제어)
    private String normalizePath(String path) {
        // /users/123 -> /users/{id}
        return path.replaceAll("/\\d+", "/{id}");
    }
}
```

### 2. Traffic (트래픽)

시스템에 대한 수요량을 측정합니다. 서비스 유형에 따라 다른 메트릭을 사용합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Traffic Metrics by Service Type              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Web Server          API Gateway          Database              │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐       │
│  │ HTTP RPS    │     │ API calls/s │     │ Queries/s   │       │
│  │ Active      │     │ Concurrent  │     │ Connections │       │
│  │ connections │     │ requests    │     │ Transactions│       │
│  └─────────────┘     └─────────────┘     └─────────────┘       │
│                                                                 │
│  Message Queue       Storage            Streaming               │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐       │
│  │ Messages/s  │     │ IO ops/s    │     │ Events/s    │       │
│  │ Queue depth │     │ Throughput  │     │ Bytes/s     │       │
│  │ Consumers   │     │ (MB/s)      │     │ Lag         │       │
│  └─────────────┘     └─────────────┘     └─────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**PromQL 쿼리**
```promql
# 초당 요청 수 (RPS)
sum(rate(http_requests_total[5m])) by (service)

# 분당 요청 수 (RPM)
sum(rate(http_requests_total[5m])) by (service) * 60

# 엔드포인트별 RPS
sum(rate(http_requests_total[5m])) by (path)

# 피크 트래픽 (5분 최대값)
max_over_time(sum(rate(http_requests_total[1m]))[5m:])

# 동시 요청 수
sum(http_requests_in_progress) by (service)

# 트래픽 증가율 (전일 대비)
sum(rate(http_requests_total[1h]))
/
sum(rate(http_requests_total[1h] offset 1d)) - 1
```

### 3. Errors (에러율)

실패한 요청의 비율을 측정합니다. 명시적 에러(HTTP 5xx)와 암묵적 에러(잘못된 응답, 타임아웃)를 구분해야 합니다.

**에러 유형 분류**
```
┌─────────────────────────────────────────────────────────────────┐
│                         Error Types                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Explicit Errors (명시적)        Implicit Errors (암묵적)        │
│  ┌─────────────────────┐        ┌─────────────────────┐        │
│  │ • HTTP 5xx 응답     │        │ • 잘못된 응답 내용    │        │
│  │ • Exception 발생    │        │ • SLA 위반 지연시간   │        │
│  │ • Circuit breaker  │        │ • 재시도 후 성공      │        │
│  │   open             │        │ • Partial failure    │        │
│  └─────────────────────┘        └─────────────────────┘        │
│                                                                 │
│  Client Errors (4xx)             Infrastructure Errors          │
│  ┌─────────────────────┐        ┌─────────────────────┐        │
│  │ • 400 Bad Request   │        │ • Database 연결 실패 │        │
│  │ • 401 Unauthorized  │        │ • Network timeout   │        │
│  │ • 404 Not Found     │        │ • Disk full         │        │
│  │ • 429 Rate Limited  │        │ • OOM               │        │
│  └─────────────────────┘        └─────────────────────┘        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**PromQL 쿼리**
```promql
# 전체 에러율 (5xx만)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# 서비스별 에러율
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)

# 엔드포인트별 에러율 (상위 10개)
topk(10,
  sum(rate(http_requests_total{status=~"5.."}[5m])) by (path)
  /
  sum(rate(http_requests_total[5m])) by (path)
)

# 에러율이 1% 초과인 서비스
(
  sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  /
  sum(rate(http_requests_total[5m])) by (service)
) > 0.01

# 에러 유형별 분류
sum(rate(http_requests_total{status=~"5.."}[5m])) by (status)

# 가용성 (Availability)
1 - (
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)
```

**알림 규칙 예시**
```yaml
groups:
  - name: error-alerts
    rules:
      # 에러율 1% 초과 시 경고
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate for {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # 에러율 5% 초과 시 긴급
      - alert: CriticalErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical error rate for {{ $labels.service }}"
          runbook_url: "https://wiki/runbook/high-error-rate"
```

### 4. Saturation (포화도)

시스템 리소스의 활용도를 측정합니다. 100%에 가까울수록 성능 저하와 장애 위험이 높아집니다.

**리소스별 포화도 지표**
```
┌─────────────────────────────────────────────────────────────────┐
│                     Saturation Metrics                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CPU                    Memory                 Disk             │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │ Usage %      │      │ Usage %      │      │ Usage %      │  │
│  │ Load Average │      │ Available    │      │ IO Wait      │  │
│  │ Throttling   │      │ Swap Usage   │      │ Queue Length │  │
│  │ Queue        │      │ OOM Events   │      │ Latency      │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                                 │
│  Network                Thread Pool           Connection Pool   │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │ Bandwidth    │      │ Active/Max   │      │ Active/Max   │  │
│  │ Packet Drop  │      │ Queue Size   │      │ Wait Time    │  │
│  │ Connection   │      │ Rejected     │      │ Timeout      │  │
│  │ Limit        │      │ Tasks        │      │ Events       │  │
│  └──────────────┘      └──────────────┘      └──────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**PromQL 쿼리**
```promql
# CPU 사용률
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# 메모리 사용률
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 디스크 사용률
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# 네트워크 대역폭 사용률 (1Gbps 기준)
rate(node_network_transmit_bytes_total[5m]) * 8 / 1000000000 * 100

# Thread Pool 포화도
hikaricp_connections_active / hikaricp_connections_max * 100

# Connection Pool 대기 시간
rate(hikaricp_connections_acquire_seconds_sum[5m])
/
rate(hikaricp_connections_acquire_seconds_count[5m])

# Kafka Consumer Lag (메시지 큐 포화도)
sum(kafka_consumer_records_lag) by (topic, consumer_group)

# JVM 힙 메모리 사용률
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100
```

---

## RED 방법론

Weave Works에서 제안한 마이크로서비스 모니터링 방법론입니다. 서비스 관점에서 모니터링합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                       RED Method                                 │
│              (Request-focused Monitoring)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      R - Rate                            │   │
│  │                                                          │   │
│  │  "초당 얼마나 많은 요청을 처리하는가?"                     │   │
│  │                                                          │   │
│  │  sum(rate(http_requests_total[5m])) by (service)         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      E - Errors                          │   │
│  │                                                          │   │
│  │  "얼마나 많은 요청이 실패하는가?"                          │   │
│  │                                                          │   │
│  │  sum(rate(http_requests_total{status=~"5.."}[5m]))       │   │
│  │  / sum(rate(http_requests_total[5m]))                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      D - Duration                        │   │
│  │                                                          │   │
│  │  "요청 처리에 얼마나 시간이 걸리는가?"                     │   │
│  │                                                          │   │
│  │  histogram_quantile(0.99,                                │   │
│  │    rate(http_request_duration_seconds_bucket[5m]))       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### RED vs Four Golden Signals

| 구분 | RED | Four Golden Signals |
|------|-----|---------------------|
| **관점** | 요청 중심 | 시스템 중심 |
| **대상** | 마이크로서비스 | 모든 시스템 |
| **포화도** | 포함 안 함 | 포함 |
| **사용 시점** | 서비스 SLA 모니터링 | 인프라 포함 전체 모니터링 |

### RED 대시보드 구성 예시

```yaml
# Grafana Dashboard JSON (간략화)
panels:
  - title: "Request Rate"
    type: graph
    targets:
      - expr: sum(rate(http_requests_total[5m])) by (service)
        legendFormat: "{{service}}"

  - title: "Error Rate"
    type: gauge
    targets:
      - expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service)
    thresholds:
      - value: 0.01
        color: green
      - value: 0.05
        color: yellow
      - value: 0.1
        color: red

  - title: "Request Duration (p99)"
    type: graph
    targets:
      - expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          )
```

---

## USE 방법론

Brendan Gregg가 제안한 인프라 모니터링 방법론입니다. 리소스 관점에서 모니터링합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                       USE Method                                 │
│              (Resource-focused Monitoring)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   U - Utilization                        │   │
│  │                                                          │   │
│  │  "리소스가 얼마나 바쁜가?"                                 │   │
│  │  (시간 기준 사용률)                                       │   │
│  │                                                          │   │
│  │  • CPU: 100 - idle%                                      │   │
│  │  • Memory: used / total                                  │   │
│  │  • Disk: IO time / elapsed time                          │   │
│  │  • Network: bytes / bandwidth                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   S - Saturation                         │   │
│  │                                                          │   │
│  │  "리소스가 과부하 상태인가?"                               │   │
│  │  (대기열 길이, 지연)                                      │   │
│  │                                                          │   │
│  │  • CPU: Run queue length, load average                   │   │
│  │  • Memory: Swap usage, OOM events                        │   │
│  │  • Disk: IO queue depth, wait time                       │   │
│  │  • Network: Dropped packets, retransmits                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     E - Errors                           │   │
│  │                                                          │   │
│  │  "리소스에서 에러가 발생하는가?"                           │   │
│  │                                                          │   │
│  │  • CPU: Hardware errors                                  │   │
│  │  • Memory: ECC errors, allocation failures               │   │
│  │  • Disk: Read/write errors, SMART warnings               │   │
│  │  • Network: CRC errors, packet errors                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 리소스별 USE 메트릭

**CPU**
```promql
# Utilization
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# Saturation (Load Average > CPU cores = 포화)
node_load1 / count(node_cpu_seconds_total{mode="idle"}) by (instance)

# Errors (CPU throttling - Kubernetes)
sum(rate(container_cpu_cfs_throttled_periods_total[5m])) by (pod)
/
sum(rate(container_cpu_cfs_periods_total[5m])) by (pod)
```

**Memory**
```promql
# Utilization
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Saturation (Swap 사용량)
node_memory_SwapTotal_bytes - node_memory_SwapFree_bytes

# OOM 이벤트
increase(node_vmstat_oom_kill[1h])

# Errors (Memory allocation failures)
rate(node_memory_numa_pages_migrated_total[5m])
```

**Disk**
```promql
# Utilization (IO busy time)
rate(node_disk_io_time_seconds_total[5m]) * 100

# Saturation (IO queue)
rate(node_disk_io_time_weighted_seconds_total[5m])

# Errors
rate(node_disk_read_errors_total[5m]) + rate(node_disk_write_errors_total[5m])

# Disk space utilization
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100
```

**Network**
```promql
# Utilization (1Gbps 기준)
(rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m]))
* 8 / 1000000000 * 100

# Saturation (dropped packets)
rate(node_network_receive_drop_total[5m]) + rate(node_network_transmit_drop_total[5m])

# Errors
rate(node_network_receive_errs_total[5m]) + rate(node_network_transmit_errs_total[5m])
```

### RED vs USE 사용 시점

```
┌─────────────────────────────────────────────────────────────────┐
│                    When to Use RED vs USE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│               ┌──────────────────────┐                          │
│               │    Application       │ ← RED Method             │
│               │  (Microservices)     │   - Rate                 │
│               │                      │   - Errors               │
│               │  • API Gateway       │   - Duration             │
│               │  • Order Service     │                          │
│               │  • Payment Service   │                          │
│               └──────────┬───────────┘                          │
│                          │                                      │
│               ┌──────────▼───────────┐                          │
│               │   Infrastructure     │ ← USE Method             │
│               │                      │   - Utilization          │
│               │  • CPU               │   - Saturation           │
│               │  • Memory            │   - Errors               │
│               │  • Disk              │                          │
│               │  • Network           │                          │
│               │  • Database          │                          │
│               └──────────────────────┘                          │
│                                                                 │
│  TIP: 서비스 문제 → RED로 증상 확인 → USE로 원인 분석            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 메트릭 타입

### Prometheus 메트릭 타입

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Prometheus Metric Types                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Counter                              Gauge                             │
│  ┌─────────────────────────┐         ┌─────────────────────────┐       │
│  │      ↗                  │         │    ↗    ↘               │       │
│  │    ↗                    │         │  ↗        ↘   ↗         │       │
│  │  ↗                      │         │↗            ↘↗          │       │
│  │↗                        │         │                         │       │
│  │ 단조 증가만 가능         │         │ 증가/감소 가능           │       │
│  │ rate() 함수 사용        │         │ 현재 값 직접 사용         │       │
│  └─────────────────────────┘         └─────────────────────────┘       │
│  예: 총 요청 수, 에러 수     예: CPU 사용률, 메모리, 동시 접속 수       │
│                                                                         │
│  Histogram                            Summary                           │
│  ┌─────────────────────────┐         ┌─────────────────────────┐       │
│  │ bucket_le_0.1: 100     │         │ quantile_0.5: 0.05      │       │
│  │ bucket_le_0.5: 450     │         │ quantile_0.9: 0.12      │       │
│  │ bucket_le_1.0: 890     │         │ quantile_0.99: 0.35     │       │
│  │ bucket_le_inf: 1000    │         │ sum: 500                │       │
│  │ sum: 500, count: 1000  │         │ count: 1000             │       │
│  │                         │         │                         │       │
│  │ 서버에서 집계 가능      │         │ 클라이언트에서 계산      │       │
│  │ 버킷 경계 사전 정의     │         │ 정확한 quantile         │       │
│  └─────────────────────────┘         └─────────────────────────┘       │
│  예: 응답 시간 분포                   예: 정밀한 백분위 수치             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1. Counter

단조 증가하는 값으로, 재시작 시에만 0으로 리셋됩니다.

```java
// Java (Micrometer)
@Component
public class RequestCounter {

    private final Counter requestCounter;
    private final Counter errorCounter;

    public RequestCounter(MeterRegistry registry) {
        this.requestCounter = Counter.builder("http_requests_total")
            .description("Total HTTP requests")
            .tag("service", "order-service")
            .register(registry);

        this.errorCounter = Counter.builder("http_errors_total")
            .description("Total HTTP errors")
            .tag("service", "order-service")
            .register(registry);
    }

    public void recordRequest() {
        requestCounter.increment();
    }

    public void recordError() {
        errorCounter.increment();
    }
}
```

```python
# Python (prometheus_client)
from prometheus_client import Counter

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# 사용
REQUEST_COUNT.labels(method='GET', endpoint='/api/users', status='200').inc()
```

**Counter PromQL 패턴**
```promql
# 절대 금지: Counter를 직접 사용
http_requests_total  # ❌ 의미없음, 계속 증가하는 값

# 올바른 사용: rate() 또는 increase()
rate(http_requests_total[5m])      # ✅ 초당 증가율
increase(http_requests_total[1h])  # ✅ 1시간 동안 증가량

# 재시작 시 값이 리셋되어도 자동 처리
rate(http_requests_total[5m])  # rate()가 리셋을 감지하고 보정
```

### 2. Gauge

현재 상태를 나타내는 값으로, 증가하거나 감소할 수 있습니다.

```java
// Java (Micrometer)
@Component
public class SystemGauges {

    public SystemGauges(MeterRegistry registry) {
        // 현재 활성 요청 수
        Gauge.builder("http_requests_in_progress",
            this::getActiveRequests)
            .description("Current active HTTP requests")
            .register(registry);

        // 큐 크기
        Gauge.builder("queue_size",
            () -> messageQueue.size())
            .description("Current queue size")
            .register(registry);

        // 마지막 처리 시간
        AtomicLong lastProcessedTime = new AtomicLong();
        Gauge.builder("last_processed_timestamp",
            lastProcessedTime::get)
            .register(registry);
    }
}
```

```python
# Python
from prometheus_client import Gauge
import psutil

# 시스템 메트릭
CPU_USAGE = Gauge('cpu_usage_percent', 'Current CPU usage')
MEMORY_USAGE = Gauge('memory_usage_bytes', 'Current memory usage')
ACTIVE_CONNECTIONS = Gauge('active_connections', 'Active database connections')

# 업데이트
def update_metrics():
    CPU_USAGE.set(psutil.cpu_percent())
    MEMORY_USAGE.set(psutil.virtual_memory().used)

# in_progress 컨텍스트 매니저
IN_PROGRESS = Gauge('requests_in_progress', 'Requests in progress')

@IN_PROGRESS.track_inprogress()
def process_request():
    # 함수 실행 중 자동으로 +1, 완료 시 -1
    pass
```

**Gauge PromQL 패턴**
```promql
# 현재 값 직접 사용
cpu_usage_percent
memory_usage_bytes

# 시간에 따른 변화
delta(temperature_celsius[1h])       # 1시간 동안 변화량
deriv(temperature_celsius[1h])       # 변화율 (미분)

# 집계
avg(cpu_usage_percent) by (instance)
max(memory_usage_bytes) by (pod)
```

### 3. Histogram

값의 분포를 측정하며, 미리 정의된 버킷에 관측값을 분류합니다.

```java
// Java (Micrometer)
@Component
public class LatencyHistogram {

    private final Timer requestTimer;

    public LatencyHistogram(MeterRegistry registry) {
        this.requestTimer = Timer.builder("http_request_duration_seconds")
            .description("HTTP request latency")
            // SLO 기준 버킷 설정
            .serviceLevelObjectives(
                Duration.ofMillis(5),
                Duration.ofMillis(10),
                Duration.ofMillis(25),
                Duration.ofMillis(50),
                Duration.ofMillis(100),
                Duration.ofMillis(250),
                Duration.ofMillis(500),
                Duration.ofSeconds(1),
                Duration.ofSeconds(2),
                Duration.ofSeconds(5)
            )
            .publishPercentileHistogram()
            .register(registry);
    }

    public void recordLatency(String method, String path, Duration duration) {
        Timer.builder("http_request_duration_seconds")
            .tag("method", method)
            .tag("path", path)
            .register(registry)
            .record(duration);
    }
}
```

```python
# Python
from prometheus_client import Histogram

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

# 사용
with REQUEST_LATENCY.labels(method='GET', endpoint='/api/users').time():
    process_request()

# 또는 수동
start = time.time()
process_request()
REQUEST_LATENCY.labels(method='GET', endpoint='/api/users').observe(time.time() - start)
```

**Histogram 데이터 구조**
```
# Prometheus가 수집하는 메트릭
http_request_duration_seconds_bucket{le="0.005"} 24
http_request_duration_seconds_bucket{le="0.01"}  42
http_request_duration_seconds_bucket{le="0.025"} 58
http_request_duration_seconds_bucket{le="0.05"}  65
http_request_duration_seconds_bucket{le="0.1"}   78
http_request_duration_seconds_bucket{le="0.25"}  89
http_request_duration_seconds_bucket{le="0.5"}   94
http_request_duration_seconds_bucket{le="1"}     98
http_request_duration_seconds_bucket{le="+Inf"}  100

http_request_duration_seconds_sum    45.2  # 모든 관측값의 합
http_request_duration_seconds_count  100   # 총 관측 횟수
```

**Histogram PromQL 패턴**
```promql
# 백분위수 계산
histogram_quantile(0.5, rate(http_request_duration_seconds_bucket[5m]))   # p50
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))  # p95
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))  # p99

# 평균 계산
rate(http_request_duration_seconds_sum[5m])
/
rate(http_request_duration_seconds_count[5m])

# Apdex Score (목표: 0.5초 이하)
(
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="2.0"}[5m])) / 2
)
/
sum(rate(http_request_duration_seconds_count[5m]))
```

### 4. Summary

클라이언트에서 직접 백분위수를 계산합니다. 정확하지만 집계가 불가능합니다.

```java
// Java (Micrometer)
@Component
public class LatencySummary {

    private final DistributionSummary summary;

    public LatencySummary(MeterRegistry registry) {
        this.summary = DistributionSummary.builder("http_request_duration")
            .description("Request duration")
            .publishPercentiles(0.5, 0.9, 0.95, 0.99)  // 클라이언트 계산
            .register(registry);
    }
}
```

```python
# Python
from prometheus_client import Summary

REQUEST_LATENCY = Summary(
    'http_request_duration_seconds',
    'Request latency',
    ['method', 'endpoint']
)

# 사용
REQUEST_LATENCY.labels(method='GET', endpoint='/users').observe(0.152)
```

### Histogram vs Summary

| 특성 | Histogram | Summary |
|------|-----------|---------|
| **계산 위치** | 서버 (PromQL) | 클라이언트 (앱) |
| **집계 가능** | 가능 (여러 인스턴스) | 불가능 |
| **정확도** | 버킷 경계에 의존 | 정확 |
| **버킷 설정** | 필요 (미리 정의) | 불필요 |
| **메모리 사용** | 버킷 수에 비례 | 슬라이딩 윈도우 크기 |
| **권장 사용** | 일반적인 경우 | 단일 인스턴스, 정밀도 필요 시 |

**권장 선택 기준**
```
다수의 인스턴스 집계 필요?
├── Yes → Histogram ✅
└── No
    └── 매우 정확한 백분위수 필요?
        ├── Yes → Summary ✅
        └── No → Histogram ✅ (기본 선택)
```

---

## 메트릭 수집 패턴

### 1. Pull 방식 (Prometheus 기본)

```
┌─────────────────────────────────────────────────────────────────┐
│                       Pull-based Collection                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    ┌────────────────┐                           │
│                    │   Prometheus   │                           │
│                    │    Server      │                           │
│                    └───────┬────────┘                           │
│                            │                                    │
│           ┌────────────────┼────────────────┐                  │
│           │ scrape         │ scrape         │ scrape           │
│           ▼                ▼                ▼                  │
│    ┌────────────┐   ┌────────────┐   ┌────────────┐           │
│    │   App 1    │   │   App 2    │   │   App 3    │           │
│    │ :8080/     │   │ :8080/     │   │ :8080/     │           │
│    │ metrics    │   │ metrics    │   │ metrics    │           │
│    └────────────┘   └────────────┘   └────────────┘           │
│                                                                 │
│  장점:                                                          │
│  • 중앙 제어 (수집 간격, 타겟 관리)                               │
│  • 서비스 디스커버리 연동                                        │
│  • 타겟 상태 모니터링 (up/down)                                  │
│                                                                 │
│  단점:                                                          │
│  • 방화벽 이슈 (Prometheus → 앱 접근 필요)                       │
│  • 짧은 수명 작업 측정 어려움                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Spring Boot 설정**
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,info
  endpoint:
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
```

**Prometheus 설정**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['app1:8080', 'app2:8080', 'app3:8080']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):\d+'
        replacement: '${1}'
```

### 2. Push 방식 (Pushgateway)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Push-based Collection                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐              │
│  │ Batch Job  │   │ Short-lived│   │ Serverless │              │
│  │            │   │   Task     │   │  Function  │              │
│  └─────┬──────┘   └─────┬──────┘   └─────┬──────┘              │
│        │                │                │                      │
│        │ push           │ push           │ push                 │
│        └────────────────┼────────────────┘                      │
│                         ▼                                       │
│                 ┌───────────────┐                               │
│                 │  Pushgateway  │                               │
│                 │  (중간 저장소) │                               │
│                 └───────┬───────┘                               │
│                         │ scrape                                │
│                         ▼                                       │
│                 ┌───────────────┐                               │
│                 │  Prometheus   │                               │
│                 └───────────────┘                               │
│                                                                 │
│  사용 시점:                                                      │
│  • 배치 작업 (cron job)                                         │
│  • 서버리스 함수                                                 │
│  • 짧은 수명 컨테이너                                            │
│                                                                 │
│  주의:                                                          │
│  • 단일 장애점 (SPOF)                                           │
│  • 마지막 값만 유지 (히스토리 X)                                  │
│  • 일반 서비스에는 권장하지 않음                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Pushgateway 사용 예시**
```python
# Python 배치 작업
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway
import time

registry = CollectorRegistry()

# 메트릭 정의
duration = Gauge('batch_job_duration_seconds',
                 'Duration of batch job',
                 registry=registry)
records_processed = Gauge('batch_job_records_processed',
                          'Number of records processed',
                          registry=registry)
job_success = Gauge('batch_job_success',
                    'Whether the job succeeded (1) or failed (0)',
                    registry=registry)

def run_batch_job():
    start_time = time.time()
    try:
        # 배치 작업 실행
        count = process_records()

        # 메트릭 기록
        duration.set(time.time() - start_time)
        records_processed.set(count)
        job_success.set(1)
    except Exception as e:
        job_success.set(0)
        raise
    finally:
        # Pushgateway로 전송
        push_to_gateway(
            'pushgateway:9091',
            job='daily_etl',
            registry=registry
        )
```

### 3. Service Discovery (Kubernetes)

```
┌─────────────────────────────────────────────────────────────────┐
│                   Kubernetes Service Discovery                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐     │
│  │                  Kubernetes Cluster                    │     │
│  │                                                        │     │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │     │
│  │  │   Pod 1     │  │   Pod 2     │  │   Pod 3     │   │     │
│  │  │ app: order  │  │ app: order  │  │ app: payment│   │     │
│  │  │ port: 8080  │  │ port: 8080  │  │ port: 8080  │   │     │
│  │  └─────────────┘  └─────────────┘  └─────────────┘   │     │
│  │                                                        │     │
│  │       ▲               ▲               ▲               │     │
│  │       │               │               │               │     │
│  │       └───────────────┼───────────────┘               │     │
│  │                       │                                │     │
│  │           ┌───────────┴───────────┐                   │     │
│  │           │     Prometheus        │                   │     │
│  │           │  (kubernetes_sd)      │                   │     │
│  │           │                       │                   │     │
│  │           │ 1. K8s API 조회       │                   │     │
│  │           │ 2. Pod 목록 확인      │                   │     │
│  │           │ 3. 자동 scrape        │                   │     │
│  │           └───────────────────────┘                   │     │
│  └───────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Prometheus Kubernetes SD 설정**
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod

    relabel_configs:
      # prometheus.io/scrape: "true" 어노테이션이 있는 Pod만 수집
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      # 커스텀 메트릭 경로 사용
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # 커스텀 포트 사용
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # 레이블 추가
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace

      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod

      - source_labels: [__meta_kubernetes_pod_label_app]
        action: replace
        target_label: app
```

**Pod 어노테이션**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: order-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

### 4. Federation (멀티 클러스터)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Prometheus Federation                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Cluster A                 Cluster B                 Cluster C          │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐ │
│  │  ┌───────────┐  │      │  ┌───────────┐  │      │  ┌───────────┐  │ │
│  │  │Prometheus │  │      │  │Prometheus │  │      │  │Prometheus │  │ │
│  │  │ (Local)   │  │      │  │ (Local)   │  │      │  │ (Local)   │  │ │
│  │  └─────┬─────┘  │      │  └─────┬─────┘  │      │  └─────┬─────┘  │ │
│  │        │        │      │        │        │      │        │        │ │
│  │  Apps  │  Apps  │      │  Apps  │  Apps  │      │  Apps  │  Apps  │ │
│  └────────┼────────┘      └────────┼────────┘      └────────┼────────┘ │
│           │                        │                        │          │
│           └────────────────────────┼────────────────────────┘          │
│                                    │                                   │
│                                    ▼                                   │
│                         ┌─────────────────────┐                        │
│                         │    Global/Central   │                        │
│                         │     Prometheus      │                        │
│                         │                     │                        │
│                         │ • 집계된 메트릭     │                        │
│                         │ • 크로스 클러스터   │                        │
│                         │ • 장기 저장         │                        │
│                         └─────────────────────┘                        │
│                                    │                                   │
│                                    ▼                                   │
│                         ┌─────────────────────┐                        │
│                         │      Grafana        │                        │
│                         │  (Global Dashboard) │                        │
│                         └─────────────────────┘                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Global Prometheus 설정**
```yaml
# global-prometheus.yml
scrape_configs:
  - job_name: 'federate-cluster-a'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job=~".+"}'  # 모든 메트릭
        # 또는 특정 메트릭만
        # - 'http_requests_total'
        # - 'up'
    static_configs:
      - targets: ['prometheus-cluster-a.example.com:9090']
        labels:
          cluster: 'cluster-a'

  - job_name: 'federate-cluster-b'
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{__name__=~"job:.*"}'  # recording rules만
    static_configs:
      - targets: ['prometheus-cluster-b.example.com:9090']
        labels:
          cluster: 'cluster-b'
```

---

## 메트릭 네이밍 컨벤션

### Prometheus 공식 가이드라인

```
┌─────────────────────────────────────────────────────────────────┐
│                   Metric Naming Convention                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  기본 형식:                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  {namespace}_{subsystem}_{name}_{unit}_{suffix}         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  예시:                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  http_server_requests_seconds_total                      │   │
│  │  ├── http_server     : namespace (HTTP 서버)            │   │
│  │  ├── requests        : 측정 대상                        │   │
│  │  ├── seconds         : 단위                             │   │
│  │  └── total           : suffix (Counter)                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Suffix 규칙:                                                   │
│  • _total     : Counter                                        │
│  • _count     : Counter (Histogram/Summary의 관측 횟수)         │
│  • _sum       : Counter (Histogram/Summary의 관측값 합)         │
│  • _bucket    : Histogram 버킷                                 │
│  • _info      : Info 메트릭 (값은 항상 1)                       │
│  • _created   : 생성 타임스탬프                                 │
│                                                                 │
│  단위 규칙 (기본 단위 사용):                                     │
│  • 시간: seconds (not milliseconds)                             │
│  • 크기: bytes (not kilobytes)                                  │
│  • 온도: celsius                                                │
│  • 비율: ratio (0-1) or percent (0-100)                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 좋은 예시 vs 나쁜 예시

```yaml
# ❌ 잘못된 네이밍
request_latency_ms_total        # ms 대신 seconds, Counter인데 latency는 Histogram
api_errors                      # 단위/suffix 없음
http-requests-count             # 하이픈 사용 금지
totalHttpRequests               # camelCase 사용 금지
myapp.requests.total            # 점 사용 금지

# ✅ 올바른 네이밍
http_request_duration_seconds   # Histogram (자동으로 _bucket, _sum, _count 생성)
http_requests_total             # Counter
http_request_size_bytes         # Histogram
process_cpu_seconds_total       # Counter
node_memory_bytes               # Gauge
```

### 레이블 사용 가이드

```yaml
# ❌ 잘못된 레이블 사용
http_requests_total{url="/api/users/123/orders/456"}  # High cardinality

# ✅ 올바른 레이블 사용
http_requests_total{
  method="GET",
  path="/api/users/{user_id}/orders/{order_id}",  # 템플릿화
  status="200",
  service="order-service"
}

# Cardinality 추정
# 레이블 조합 수 = method(5) × path(20) × status(10) × service(10) = 10,000
# 시계열 수 = 메트릭 수 × Cardinality = 50 × 10,000 = 500,000 (관리 가능)
```

---

## 실무 Best Practices

### 1. Cardinality 관리

```java
// ❌ High Cardinality - 피해야 할 패턴
Counter.builder("http_requests_total")
    .tag("user_id", userId)           // 수백만 개의 고유값
    .tag("session_id", sessionId)     // 무한대의 고유값
    .tag("url", request.getUri())     // 경로 변수 포함
    .register(registry);

// ✅ 적절한 Cardinality
Counter.builder("http_requests_total")
    .tag("method", request.getMethod())           // ~5개
    .tag("path", normalizePath(request.getUri())) // ~50개
    .tag("status", String.valueOf(statusCode))    // ~50개
    .tag("service", serviceName)                  // ~20개
    .register(registry);

private String normalizePath(String uri) {
    return uri
        .replaceAll("/users/\\d+", "/users/{id}")
        .replaceAll("/orders/[a-f0-9-]+", "/orders/{id}")
        .replaceAll("\\?.*", "");  // 쿼리 파라미터 제거
}
```

### 2. 성능 고려사항

```java
// ❌ 매번 레지스트리에서 조회
public void recordRequest() {
    meterRegistry.counter("requests", "method", method).increment();  // 매번 조회
}

// ✅ 캐시 사용
private final Map<String, Counter> counterCache = new ConcurrentHashMap<>();

public void recordRequest(String method) {
    counterCache.computeIfAbsent(method, m ->
        Counter.builder("requests")
            .tag("method", m)
            .register(meterRegistry)
    ).increment();
}

// ✅ 또는 메트릭 미리 등록
@PostConstruct
public void initMetrics() {
    for (String method : Arrays.asList("GET", "POST", "PUT", "DELETE")) {
        Counter.builder("requests")
            .tag("method", method)
            .register(meterRegistry);
    }
}
```

### 3. SLO 기반 Histogram 버킷 설정

```java
// SLO: 95%의 요청이 500ms 이내, 99%가 1초 이내
Timer.builder("http_request_duration")
    .serviceLevelObjectives(
        Duration.ofMillis(50),   // 매우 빠름
        Duration.ofMillis(100),  // 빠름
        Duration.ofMillis(250),  // 정상
        Duration.ofMillis(500),  // SLO 목표 (p95)
        Duration.ofSeconds(1),   // SLO 목표 (p99)
        Duration.ofSeconds(2),   // 느림
        Duration.ofSeconds(5)    // 매우 느림
    )
    .register(registry);
```

### 4. Recording Rules 활용

```yaml
# prometheus-rules.yml
groups:
  - name: http-metrics
    interval: 30s
    rules:
      # 미리 계산된 에러율
      - record: job:http_requests_error_rate:5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      # 미리 계산된 p99
      - record: job:http_request_duration_seconds:p99:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le)
          )

      # RPS
      - record: job:http_requests:rate5m
        expr: |
          sum(rate(http_requests_total[5m])) by (job)

# 복잡한 쿼리 대신 recording rule 사용
# Before: histogram_quantile(0.99, sum(rate(...)) by (...))
# After:  job:http_request_duration_seconds:p99:5m
```

### 5. 메트릭 구조화

```
┌─────────────────────────────────────────────────────────────────┐
│                  Recommended Metric Structure                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Business Metrics (비즈니스)                                     │
│  ├── orders_created_total                                       │
│  ├── payments_processed_total                                   │
│  ├── users_registered_total                                     │
│  └── revenue_dollars_total                                      │
│                                                                 │
│  Application Metrics (애플리케이션)                              │
│  ├── http_request_duration_seconds                              │
│  ├── http_requests_total                                        │
│  ├── db_query_duration_seconds                                  │
│  └── cache_hit_total / cache_miss_total                        │
│                                                                 │
│  Runtime Metrics (런타임)                                        │
│  ├── jvm_memory_used_bytes                                      │
│  ├── jvm_gc_pause_seconds                                       │
│  ├── jvm_threads_current                                        │
│  └── process_cpu_seconds_total                                  │
│                                                                 │
│  Infrastructure Metrics (인프라)                                 │
│  ├── node_cpu_seconds_total                                     │
│  ├── node_memory_MemAvailable_bytes                             │
│  ├── node_disk_io_time_seconds_total                            │
│  └── container_cpu_usage_seconds_total                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

### 공식 문서
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
- [OpenTelemetry Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)

### 방법론 원문
- [The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)
- [The RED Method](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [The USE Method](https://www.brendangregg.com/usemethod.html)

### 관련 문서
- [01-로깅](../01-로깅/README.md)
- [03-분산-추적](../03-분산-추적/README.md)
- [04-Prometheus-Grafana](../04-Prometheus-Grafana/README.md)

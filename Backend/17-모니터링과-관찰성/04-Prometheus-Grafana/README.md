# Prometheus & Grafana

## 목차
1. [개요](#개요)
2. [Prometheus 아키텍처](#prometheus-아키텍처)
3. [PromQL 기초와 심화](#promql-기초와-심화)
4. [Grafana 대시보드 설계](#grafana-대시보드-설계)
5. [알림 설정 (Alertmanager)](#알림-설정-alertmanager)
6. [Service Discovery](#service-discovery)
7. [실무 Best Practices](#실무-best-practices)
---

## 개요

Prometheus는 SoundCloud에서 개발한 오픈소스 모니터링 시스템으로, CNCF의 졸업 프로젝트입니다. 시계열 데이터베이스와 강력한 쿼리 언어(PromQL)를 제공하며, Grafana와 함께 사용하여 시각화와 알림 기능을 완성합니다.

### Prometheus + Grafana 스택

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Prometheus + Grafana Stack                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   App 1     │  │   App 2     │  │   App 3     │  │   Node      │   │
│  │  /metrics   │  │  /metrics   │  │  /metrics   │  │  Exporter   │   │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │
│         │                │                │                │          │
│         └────────────────┼────────────────┼────────────────┘          │
│                          │                │                            │
│                   ┌──────▼────────────────▼──────┐                    │
│                   │         Prometheus           │                    │
│                   │                              │                    │
│                   │  • Scrape (Pull)             │                    │
│                   │  • Store (TSDB)              │                    │
│                   │  • Query (PromQL)            │                    │
│                   │  • Alert (Rules)             │                    │
│                   └──────┬───────────┬───────────┘                    │
│                          │           │                                 │
│              ┌───────────▼───┐   ┌───▼───────────────┐                │
│              │ Alertmanager  │   │     Grafana       │                │
│              │               │   │                   │                │
│              │ • Routing     │   │ • Visualization   │                │
│              │ • Grouping    │   │ • Dashboards      │                │
│              │ • Silencing   │   │ • Alerting        │                │
│              │ • Notification│   │ • Explore         │                │
│              └───────┬───────┘   └───────────────────┘                │
│                      │                                                 │
│         ┌────────────┼────────────┐                                   │
│         ▼            ▼            ▼                                   │
│     ┌───────┐   ┌───────┐   ┌───────┐                                │
│     │ Slack │   │ Email │   │PagerDuty│                               │
│     └───────┘   └───────┘   └───────┘                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2025년 Observability 트렌드

2025년 Observability 조사에 따르면, 70%의 기업이 Prometheus와 OpenTelemetry를 함께 사용하고 있으며, Grafana LGTM 스택(Loki, Grafana, Tempo, Mimir)이 벤더 종속성 없는 통합 관찰성 플랫폼으로 주목받고 있습니다.

---

## Prometheus 아키텍처

### 핵심 컴포넌트

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Prometheus Architecture                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                      Prometheus Server                          │    │
│  │                                                                 │    │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │    │
│  │  │  Retrieval  │    │    TSDB     │    │  HTTP       │        │    │
│  │  │             │    │             │    │  Server     │        │    │
│  │  │ • Scrape    │───▶│ • Storage   │◀───│             │        │    │
│  │  │ • SD        │    │ • Index     │    │ • API       │        │    │
│  │  │             │    │ • Compact   │    │ • PromQL    │        │    │
│  │  └──────┬──────┘    └─────────────┘    └──────┬──────┘        │    │
│  │         │                                      │               │    │
│  │         │           ┌─────────────┐           │               │    │
│  │         │           │    Rules    │           │               │    │
│  │         └──────────▶│   Engine    │◀──────────┘               │    │
│  │                     │             │                            │    │
│  │                     │ • Recording │                            │    │
│  │                     │ • Alerting  │                            │    │
│  │                     └──────┬──────┘                            │    │
│  └─────────────────────────────┼──────────────────────────────────┘    │
│                                │                                        │
│                                ▼                                        │
│                     ┌─────────────────────┐                            │
│                     │    Alertmanager     │                            │
│                     │                     │                            │
│                     │ • Deduplication     │                            │
│                     │ • Grouping          │                            │
│                     │ • Routing           │                            │
│                     │ • Silencing         │                            │
│                     └─────────────────────┘                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### TSDB (Time Series Database)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Prometheus TSDB                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  시계열 데이터 구조:                                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ http_requests_total{method="GET", path="/api", status="200"}    │   │
│  │ ├── (t1, 100)                                                   │   │
│  │ ├── (t2, 150)                                                   │   │
│  │ ├── (t3, 200)                                                   │   │
│  │ └── ...                                                         │   │
│  │                                                                  │   │
│  │ [메트릭명]{[레이블]} = (타임스탬프, 값) 쌍의 시퀀스               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  저장 구조:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ./data/                                                        │   │
│  │  ├── 01ABCDEF.../ (Block - 2시간 단위)                          │   │
│  │  │   ├── chunks/           # 압축된 시계열 데이터               │   │
│  │  │   ├── index             # 레이블 인덱스                      │   │
│  │  │   ├── meta.json         # 블록 메타데이터                    │   │
│  │  │   └── tombstones        # 삭제 마커                          │   │
│  │  ├── 01GHIJKL.../                                               │   │
│  │  ├── wal/                   # Write-Ahead Log                   │   │
│  │  │   ├── 00000001                                               │   │
│  │  │   └── 00000002                                               │   │
│  │  └── chunks_head/           # 현재 수집 중인 데이터             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  특징:                                                                   │
│  • 2시간마다 블록 생성 (Compaction)                                     │
│  • WAL로 데이터 손실 방지                                               │
│  • 압축률: 평균 1.3 bytes/sample                                        │
│  • 로컬 저장 (원격 스토리지 연동 가능)                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Prometheus 설정

**prometheus.yml**
```yaml
# 전역 설정
global:
  scrape_interval: 15s          # 기본 수집 주기
  evaluation_interval: 15s      # 규칙 평가 주기
  scrape_timeout: 10s           # 수집 타임아웃

  # 외부 레이블 (Federation, Remote Write 시 사용)
  external_labels:
    cluster: 'production'
    region: 'ap-northeast-2'

# 알림 규칙 파일
rule_files:
  - '/etc/prometheus/rules/*.yml'

# Alertmanager 설정
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
      timeout: 10s

# 수집 대상 설정
scrape_configs:
  # Prometheus 자체 메트릭
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # 애플리케이션 메트릭
  - job_name: 'spring-boot'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets: ['app1:8080', 'app2:8080', 'app3:8080']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '([^:]+):\d+'
        replacement: '${1}'

  # Node Exporter (시스템 메트릭)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']

  # Kubernetes Service Discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

# 원격 저장소 설정 (장기 보관)
remote_write:
  - url: 'http://mimir:9009/api/v1/push'
    queue_config:
      max_samples_per_send: 1000
      batch_send_deadline: 5s
      capacity: 5000

remote_read:
  - url: 'http://mimir:9009/prometheus/api/v1/read'
    read_recent: true
```

### 멀티 클러스터 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Multi-Cluster Architecture                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Cluster A                 Cluster B                 Cluster C          │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐ │
│  │ Prometheus      │      │ Prometheus      │      │ Prometheus      │ │
│  │ (Local)         │      │ (Local)         │      │ (Local)         │ │
│  │                 │      │                 │      │                 │ │
│  │ • 상세 메트릭   │      │ • 상세 메트릭   │      │ • 상세 메트릭   │ │
│  │ • 15일 보관    │      │ • 15일 보관    │      │ • 15일 보관    │ │
│  └────────┬────────┘      └────────┬────────┘      └────────┬────────┘ │
│           │                        │                        │          │
│           │     Remote Write       │                        │          │
│           └────────────────────────┼────────────────────────┘          │
│                                    │                                    │
│                                    ▼                                    │
│           ┌──────────────────────────────────────────────┐             │
│           │              Grafana Mimir / Thanos          │             │
│           │              (Long-term Storage)             │             │
│           │                                              │             │
│           │  • 집계된 메트릭                             │             │
│           │  • 1년 이상 보관                             │             │
│           │  • 크로스 클러스터 쿼리                      │             │
│           │  • Object Storage (S3/GCS)                  │             │
│           └─────────────────────┬────────────────────────┘             │
│                                 │                                       │
│                                 ▼                                       │
│           ┌──────────────────────────────────────────────┐             │
│           │                  Grafana                     │             │
│           │                                              │             │
│           │  • Global Dashboard                          │             │
│           │  • Cross-cluster Alerting                    │             │
│           └──────────────────────────────────────────────┘             │
│                                                                         │
│  대안 스택:                                                             │
│  • Thanos: Sidecar 방식, 기존 Prometheus 활용                          │
│  • Mimir: Grafana 개발, 더 나은 확장성                                 │
│  • VictoriaMetrics: 높은 성능, 낮은 리소스                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## PromQL 기초와 심화

### 기본 문법

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PromQL Fundamentals                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  데이터 타입:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. Instant Vector (순간 벡터)                                    │   │
│  │    - 특정 시점의 시계열 집합                                     │   │
│  │    - 예: http_requests_total                                     │   │
│  │                                                                  │   │
│  │ 2. Range Vector (범위 벡터)                                      │   │
│  │    - 시간 범위의 시계열 집합                                     │   │
│  │    - 예: http_requests_total[5m]                                 │   │
│  │                                                                  │   │
│  │ 3. Scalar (스칼라)                                               │   │
│  │    - 단일 숫자 값                                                │   │
│  │    - 예: 3.14, count(...)                                        │   │
│  │                                                                  │   │
│  │ 4. String (문자열)                                               │   │
│  │    - 문자열 값 (거의 사용 안 함)                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  레이블 셀렉터:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ =    : 정확히 일치          http_requests_total{method="GET"}   │   │
│  │ !=   : 일치하지 않음        http_requests_total{status!="200"}  │   │
│  │ =~   : 정규식 일치          http_requests_total{path=~"/api.*"} │   │
│  │ !~   : 정규식 불일치        http_requests_total{path!~"/health"}│   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  시간 범위:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ s  : 초        [5s]                                              │   │
│  │ m  : 분        [5m]                                              │   │
│  │ h  : 시간      [1h]                                              │   │
│  │ d  : 일        [7d]                                              │   │
│  │ w  : 주        [2w]                                              │   │
│  │ y  : 년        [1y]                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 핵심 함수

#### rate() vs irate()

```promql
# rate(): 지정 시간 범위의 평균 변화율
# - 스파이크 완화, 안정적인 그래프
# - 알림 규칙에 권장
rate(http_requests_total[5m])

# irate(): 마지막 두 데이터 포인트의 순간 변화율
# - 스파이크 감지, 실시간 모니터링
# - 변동이 심함
irate(http_requests_total[5m])

# 사용 권장:
# - 대시보드 그래프: rate() (5분 이상)
# - 알림: rate() (안정적)
# - 실시간 스파이크 감지: irate()
```

#### increase() vs delta()

```promql
# increase(): Counter의 증가량 (재시작 처리)
increase(http_requests_total[1h])  # 1시간 동안 요청 수

# delta(): Gauge의 변화량
delta(temperature_celsius[1h])  # 1시간 동안 온도 변화

# 주의: Counter에 delta() 사용 금지 (재시작 시 음수 가능)
```

#### 집계 함수

```promql
# sum: 합계
sum(rate(http_requests_total[5m])) by (service)

# avg: 평균
avg(cpu_usage_percent) by (instance)

# max/min: 최대/최소
max(memory_usage_bytes) by (pod)

# count: 시계열 개수
count(up == 1) by (job)

# topk/bottomk: 상위/하위 N개
topk(10, sum(rate(http_requests_total[5m])) by (path))

# quantile: 백분위수
quantile(0.95, http_request_duration_seconds)

# stddev/stdvar: 표준편차/분산
stddev(response_time_seconds) by (service)
```

### 실전 PromQL 쿼리

#### 에러율 계산

```promql
# 기본 에러율
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# 서비스별 에러율
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/
sum(rate(http_requests_total[5m])) by (service)

# 에러율이 1% 초과인 서비스만
(
  sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
  /
  sum(rate(http_requests_total[5m])) by (service)
) > 0.01

# 가용성 (Availability)
1 - (
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
)
```

#### 지연 시간 (Latency)

```promql
# p50, p95, p99 지연 시간 (Histogram)
histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# 서비스별 p99
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# 평균 지연 시간
sum(rate(http_request_duration_seconds_sum[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))

# Apdex Score (목표: 500ms 이하)
(
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="2.0"}[5m])) / 2
)
/
sum(rate(http_request_duration_seconds_count[5m]))
```

#### 리소스 사용량

```promql
# CPU 사용률 (%)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

# 메모리 사용률 (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 디스크 사용률 (%)
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Pod CPU 사용률 (Kubernetes)
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
/
sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod) * 100

# Pod 메모리 사용률
sum(container_memory_usage_bytes{container!=""}) by (pod)
/
sum(kube_pod_container_resource_limits{resource="memory"}) by (pod) * 100
```

#### 트래픽 분석

```promql
# RPS (Requests Per Second)
sum(rate(http_requests_total[5m])) by (service)

# RPM (Requests Per Minute)
sum(rate(http_requests_total[5m])) by (service) * 60

# 피크 트래픽 (지난 1시간 최대 RPS)
max_over_time(sum(rate(http_requests_total[1m]))[1h:])

# 전일 대비 트래픽 변화
(
  sum(rate(http_requests_total[1h]))
  -
  sum(rate(http_requests_total[1h] offset 1d))
)
/
sum(rate(http_requests_total[1h] offset 1d))

# 동시 요청 수
sum(http_requests_in_progress) by (service)
```

#### 시간 기반 분석

```promql
# offset: 과거 데이터 비교
sum(rate(http_requests_total[1h])) - sum(rate(http_requests_total[1h] offset 1d))

# 1주일 전 대비 증가율
(sum(rate(http_requests_total[1h])) - sum(rate(http_requests_total[1h] offset 1w)))
/ sum(rate(http_requests_total[1h] offset 1w)) * 100

# 시간대별 패턴 (Recording Rule 권장)
avg_over_time(sum(rate(http_requests_total[5m]))[1h:5m])

# 특정 시간대 필터링 (Grafana에서 더 쉬움)
sum(rate(http_requests_total[5m])) and on() hour() >= 9 < 18
```

### Recording Rules

자주 사용하는 복잡한 쿼리를 미리 계산하여 저장합니다.

```yaml
# recording-rules.yml
groups:
  - name: http-metrics
    interval: 30s
    rules:
      # 에러율
      - record: job:http_requests_error_rate:5m
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          /
          sum(rate(http_requests_total[5m])) by (job)

      # p99 지연 시간
      - record: job:http_request_duration_seconds:p99:5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le)
          )

      # RPS
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)

      # 가용성
      - record: job:http_availability:5m
        expr: |
          1 - (
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
            /
            sum(rate(http_requests_total[5m])) by (job)
          )

  - name: node-metrics
    interval: 1m
    rules:
      # CPU 사용률
      - record: instance:node_cpu_utilization:rate5m
        expr: |
          100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100)

      # 메모리 사용률
      - record: instance:node_memory_utilization:percent
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

Recording Rules 사용:
```promql
# 복잡한 원본 쿼리 대신
# histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (job, le))

# 간단하게 사용
job:http_request_duration_seconds:p99:5m
```

---

## Grafana 대시보드 설계

### 대시보드 구조 원칙

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Dashboard Design Principles                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  3-3-3 규칙 (권장):                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ • 3 Rows (행)                                                    │   │
│  │ • 3 Panels per Row (행당 3개 패널)                               │   │
│  │ • 3 Key Metrics per Panel (패널당 3개 핵심 메트릭)               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  계층 구조:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Level 1: Overview (개요)                                        │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │ • 전체 시스템 상태                                         │ │   │
│  │  │ • SLI/SLO 대시보드                                         │ │   │
│  │  │ • 핵심 비즈니스 메트릭                                     │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  │           │                                                      │   │
│  │           ▼                                                      │   │
│  │  Level 2: Service (서비스별)                                     │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │ • RED 메트릭 (Rate, Errors, Duration)                      │ │   │
│  │  │ • 서비스 의존성                                            │ │   │
│  │  │ • 주요 엔드포인트 상태                                     │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  │           │                                                      │   │
│  │           ▼                                                      │   │
│  │  Level 3: Detail (상세)                                          │   │
│  │  ┌────────────────────────────────────────────────────────────┐ │   │
│  │  │ • Pod/Instance 레벨 메트릭                                 │ │   │
│  │  │ • JVM/Runtime 상세                                         │ │   │
│  │  │ • 데이터베이스 상세                                        │ │   │
│  │  └────────────────────────────────────────────────────────────┘ │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### SLI/SLO 대시보드 예시

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SLI/SLO Dashboard                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Row 1: SLO Summary                                                     │
│  ┌──────────────────┬──────────────────┬──────────────────┐            │
│  │   Availability   │     Latency      │   Error Budget   │            │
│  │    SLO: 99.9%    │  SLO: p99<500ms  │   Remaining      │            │
│  │  ┌────────────┐  │  ┌────────────┐  │  ┌────────────┐  │            │
│  │  │   99.95%   │  │  │   320ms    │  │  │   45.2%    │  │            │
│  │  │    ✓ OK    │  │  │    ✓ OK    │  │  │            │  │            │
│  │  └────────────┘  │  └────────────┘  │  └────────────┘  │            │
│  └──────────────────┴──────────────────┴──────────────────┘            │
│                                                                         │
│  Row 2: Trends                                                          │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │  Availability over Time                                   │          │
│  │  ┌──────────────────────────────────────────────────────┐│          │
│  │  │ 100%|     ____    ___________    ___                 ││          │
│  │  │ 99.9|----/----\__/           \__/   \______ SLO     ││          │
│  │  │ 99% |                                                ││          │
│  │  │     └──────────────────────────────────────────────  ││          │
│  │  │        Mon   Tue   Wed   Thu   Fri   Sat   Sun      ││          │
│  │  └──────────────────────────────────────────────────────┘│          │
│  └──────────────────────────────────────────────────────────┘          │
│                                                                         │
│  Row 3: Error Budget Burn Rate                                          │
│  ┌──────────────────┬──────────────────┬──────────────────┐            │
│  │  1h Burn Rate    │  6h Burn Rate    │  24h Burn Rate   │            │
│  │  ┌────────────┐  │  ┌────────────┐  │  ┌────────────┐  │            │
│  │  │    2.1x    │  │  │    1.5x    │  │  │    0.8x    │  │            │
│  │  │  ⚠ Warning │  │  │    ✓ OK    │  │  │    ✓ OK    │  │            │
│  │  └────────────┘  │  └────────────┘  │  └────────────┘  │            │
│  └──────────────────┴──────────────────┴──────────────────┘            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Grafana 패널 설정 (JSON)

```json
{
  "dashboard": {
    "title": "Service Overview",
    "tags": ["sre", "overview"],
    "timezone": "browser",
    "panels": [
      {
        "title": "Request Rate (RPS)",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 8, "x": 0, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "color": {"mode": "palette-classic"},
            "custom": {
              "lineWidth": 2,
              "fillOpacity": 10
            }
          }
        }
      },
      {
        "title": "Error Rate (%)",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 8, "x": 8, "y": 0},
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service) * 100",
            "legendFormat": "{{service}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        }
      },
      {
        "title": "p99 Latency",
        "type": "timeseries",
        "gridPos": {"h": 8, "w": 8, "x": 16, "y": 0},
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))",
            "legendFormat": "{{service}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "custom": {
              "thresholdsStyle": {"mode": "line"}
            },
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "red", "value": 0.5}
              ]
            }
          }
        }
      },
      {
        "title": "Current Availability",
        "type": "stat",
        "gridPos": {"h": 4, "w": 4, "x": 0, "y": 8},
        "targets": [
          {
            "expr": "(1 - sum(rate(http_requests_total{status=~\"5..\"}[1h])) / sum(rate(http_requests_total[1h]))) * 100",
            "instant": true
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "decimals": 3,
            "thresholds": {
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 99.5},
                {"color": "green", "value": 99.9}
              ]
            }
          }
        },
        "options": {
          "reduceOptions": {"calcs": ["lastNotNull"]}
        }
      }
    ],
    "templating": {
      "list": [
        {
          "name": "service",
          "type": "query",
          "query": "label_values(http_requests_total, service)",
          "multi": true,
          "includeAll": true
        },
        {
          "name": "environment",
          "type": "custom",
          "options": [
            {"text": "production", "value": "production"},
            {"text": "staging", "value": "staging"}
          ]
        }
      ]
    },
    "annotations": {
      "list": [
        {
          "name": "Deployments",
          "datasource": "Prometheus",
          "expr": "changes(deployment_timestamp[5m]) > 0",
          "tagKeys": "service"
        }
      ]
    }
  }
}
```

### 대시보드 프로비저닝

```yaml
# /etc/grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: 'SRE'
    folderUid: 'sre-dashboards'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /var/lib/grafana/dashboards/sre

  - name: 'kubernetes'
    orgId: 1
    folder: 'Kubernetes'
    type: file
    options:
      path: /var/lib/grafana/dashboards/kubernetes
```

---

## 알림 설정 (Alertmanager)

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Alertmanager Architecture                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Prometheus                                                             │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Alert Rules Evaluation                                         │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │ ALERT HighErrorRate                                      │  │    │
│  │  │   expr: error_rate > 0.01                                │  │    │
│  │  │   for: 5m                                                │  │    │
│  │  │   → PENDING → FIRING                                     │  │    │
│  │  └──────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────┬──────────────────────────────────┘    │
│                                │                                        │
│                                ▼                                        │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                       Alertmanager                              │    │
│  │                                                                 │    │
│  │  1. Deduplication (중복 제거)                                   │    │
│  │     ┌────────────────────────────────────────────────────────┐ │    │
│  │     │ 동일한 알림 = 하나로 처리                               │ │    │
│  │     └────────────────────────────────────────────────────────┘ │    │
│  │                                                                 │    │
│  │  2. Grouping (그룹화)                                           │    │
│  │     ┌────────────────────────────────────────────────────────┐ │    │
│  │     │ 관련 알림 묶음 (예: 같은 서비스의 여러 인스턴스)        │ │    │
│  │     └────────────────────────────────────────────────────────┘ │    │
│  │                                                                 │    │
│  │  3. Routing (라우팅)                                            │    │
│  │     ┌────────────────────────────────────────────────────────┐ │    │
│  │     │ 레이블 기반으로 수신자 결정                             │ │    │
│  │     │ severity=critical → PagerDuty                          │ │    │
│  │     │ severity=warning → Slack                               │ │    │
│  │     └────────────────────────────────────────────────────────┘ │    │
│  │                                                                 │    │
│  │  4. Silencing (억제)                                            │    │
│  │     ┌────────────────────────────────────────────────────────┐ │    │
│  │     │ 유지보수 중 알림 억제                                   │ │    │
│  │     └────────────────────────────────────────────────────────┘ │    │
│  │                                                                 │    │
│  │  5. Inhibition (억제 규칙)                                      │    │
│  │     ┌────────────────────────────────────────────────────────┐ │    │
│  │     │ 상위 알림 발생 시 하위 알림 억제                        │ │    │
│  │     │ (클러스터 다운 → 개별 Pod 알림 억제)                    │ │    │
│  │     └────────────────────────────────────────────────────────┘ │    │
│  │                                                                 │    │
│  └─────────────────────────────┬──────────────────────────────────┘    │
│                                │                                        │
│         ┌──────────────────────┼──────────────────────────┐            │
│         ▼                      ▼                          ▼            │
│     ┌───────┐            ┌───────────┐            ┌───────────┐       │
│     │ Slack │            │ PagerDuty │            │   Email   │       │
│     └───────┘            └───────────┘            └───────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 알림 규칙 (Prometheus)

```yaml
# alerting-rules.yml
groups:
  - name: slo-alerts
    rules:
      # SLO 기반 알림 (Error Budget Burn Rate)
      - alert: ErrorBudgetBurnRateCritical
        expr: |
          (
            # 1시간 에러율
            sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
            / sum(rate(http_requests_total[1h])) by (service)
          ) > (14.4 * 0.001)  # 14.4배 burn rate, SLO 99.9%
        for: 2m
        labels:
          severity: critical
          category: slo
        annotations:
          summary: "High error budget burn rate for {{ $labels.service }}"
          description: "Service {{ $labels.service }} is burning error budget 14.4x faster than expected"
          runbook_url: "https://wiki/runbook/error-budget-burn"
          dashboard_url: "https://grafana/d/slo-dashboard?var-service={{ $labels.service }}"

      - alert: ErrorBudgetBurnRateWarning
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) by (service)
            / sum(rate(http_requests_total[6h])) by (service)
          ) > (6 * 0.001)
        for: 5m
        labels:
          severity: warning
          category: slo
        annotations:
          summary: "Elevated error budget burn rate for {{ $labels.service }}"

  - name: latency-alerts
    rules:
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency for {{ $labels.service }}"
          description: "p99 latency is {{ $value | humanizeDuration }}"

      - alert: HighLatencyP99Critical
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
          ) > 2
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical p99 latency for {{ $labels.service }}"

  - name: infrastructure-alerts
    rules:
      - alert: HighCPUUsage
        expr: |
          100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance) * 100) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | printf \"%.1f\" }}%"

      - alert: HighMemoryUsage
        expr: |
          (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"

      - alert: DiskSpaceLow
        expr: |
          (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"

      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

  - name: availability-alerts
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.instance }} is down"
          description: "Prometheus cannot scrape {{ $labels.job }}/{{ $labels.instance }}"
```

### Alertmanager 설정

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/xxx'
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# 라우팅 규칙
route:
  # 기본 수신자
  receiver: 'slack-notifications'
  # 그룹화 키
  group_by: ['alertname', 'service', 'severity']
  # 첫 알림 대기 시간
  group_wait: 30s
  # 같은 그룹 알림 간격
  group_interval: 5m
  # 동일 알림 재전송 간격
  repeat_interval: 4h

  # 하위 라우팅
  routes:
    # Critical 알림 → PagerDuty
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      continue: true  # 다음 라우트도 확인

    # SLO 관련 알림 → 전용 채널
    - match:
        category: slo
      receiver: 'slack-slo'
      group_by: ['service']

    # 인프라 알림
    - match_re:
        alertname: ^(HighCPU|HighMemory|DiskSpace).*
      receiver: 'slack-infra'
      group_wait: 10m

    # 특정 서비스 알림
    - match:
        service: payment-service
      receiver: 'pagerduty-payments'

# 수신자 정의
receivers:
  - name: 'slack-notifications'
    slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ .Status | toUpper }}: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Service:* {{ .Labels.service }}
          {{ if .Annotations.runbook_url }}*Runbook:* {{ .Annotations.runbook_url }}{{ end }}
          {{ end }}
        actions:
          - type: button
            text: 'View Dashboard'
            url: '{{ (index .Alerts 0).Annotations.dashboard_url }}'
          - type: button
            text: 'Silence'
            url: '{{ template "__alertmanagerURL__" . }}/#/silences/new?filter=%7Balertname%3D%22{{ .GroupLabels.alertname }}%22%7D'

  - name: 'slack-slo'
    slack_configs:
      - channel: '#slo-alerts'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

  - name: 'slack-infra'
    slack_configs:
      - channel: '#infra-alerts'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-routing-key'
        severity: critical
        description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
        details:
          firing: '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
          resolved: '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'

  - name: 'pagerduty-payments'
    pagerduty_configs:
      - routing_key: 'payments-team-routing-key'
        severity: '{{ .Labels.severity }}'

# 억제 규칙
inhibit_rules:
  # 클러스터 다운 시 개별 노드 알림 억제
  - source_match:
      alertname: 'ClusterDown'
    target_match_re:
      alertname: 'Node.*'
    equal: ['cluster']

  # Critical 알림 시 동일 서비스의 Warning 억제
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'service']

# 템플릿
templates:
  - '/etc/alertmanager/templates/*.tmpl'
```

### Grafana Alerting (통합 알림)

```yaml
# Grafana 8.0+ Unified Alerting
apiVersion: 1

groups:
  - orgId: 1
    name: service-alerts
    folder: SRE
    interval: 1m
    rules:
      - uid: high-error-rate
        title: High Error Rate
        condition: C
        data:
          - refId: A
            relativeTimeRange:
              from: 600
              to: 0
            datasourceUid: prometheus
            model:
              expr: |
                sum(rate(http_requests_total{status=~"5.."}[$__rate_interval])) by (service)
                / sum(rate(http_requests_total[$__rate_interval])) by (service)
              intervalMs: 1000
              maxDataPoints: 43200
          - refId: B
            relativeTimeRange:
              from: 600
              to: 0
            datasourceUid: __expr__
            model:
              conditions:
                - evaluator:
                    params: [0.01]
                    type: gt
                  operator:
                    type: and
                  query:
                    params: [A]
                  reducer:
                    type: last
              type: classic_conditions
          - refId: C
            datasourceUid: __expr__
            model:
              expression: B
              type: threshold
        noDataState: NoData
        execErrState: Error
        for: 5m
        annotations:
          summary: "Error rate is {{ $values.A }}% for {{ $labels.service }}"
          dashboard_uid: service-overview
        labels:
          severity: warning
```

---

## Service Discovery

### Kubernetes Service Discovery

```yaml
# prometheus.yml - Kubernetes SD
scrape_configs:
  # Pod 자동 발견
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - default
            - production
    relabel_configs:
      # prometheus.io/scrape: "true" 어노테이션 필터
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

      # 커스텀 메트릭 경로
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # 커스텀 포트
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # 네임스페이스 레이블 추가
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace

      # Pod 이름 레이블 추가
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod

      # 서비스 이름 (앱 레이블에서)
      - source_labels: [__meta_kubernetes_pod_label_app]
        action: replace
        target_label: app

      # 사이드카 컨테이너 제외
      - source_labels: [__meta_kubernetes_pod_container_name]
        action: drop
        regex: (istio-proxy|envoy)

  # Service 자동 발견
  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

  # Endpoints 자동 발견 (더 정밀한 제어)
  - job_name: 'kubernetes-endpoints'
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_endpoints_name]
        action: replace
        target_label: endpoint

  # Node 자동 발견 (Node Exporter)
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics
```

### 애플리케이션 어노테이션

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
      labels:
        app: order-service
        version: v1.2.3
    spec:
      containers:
        - name: order-service
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 8081  # 별도 메트릭 포트인 경우
```

### Consul Service Discovery

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'consul.service.consul:8500'
        services: []  # 모든 서비스
        tags:
          - 'prometheus'  # 특정 태그가 있는 서비스만
    relabel_configs:
      - source_labels: [__meta_consul_service]
        target_label: service
      - source_labels: [__meta_consul_tags]
        regex: .*,env=([^,]+),.*
        replacement: '${1}'
        target_label: environment
```

### File-based Service Discovery

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'file-based'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
          - '/etc/prometheus/targets/*.yml'
        refresh_interval: 30s

# /etc/prometheus/targets/apps.json
[
  {
    "targets": ["app1:8080", "app2:8080"],
    "labels": {
      "job": "spring-boot",
      "environment": "production",
      "team": "platform"
    }
  },
  {
    "targets": ["app3:3000", "app4:3000"],
    "labels": {
      "job": "node-apps",
      "environment": "production"
    }
  }
]
```

---

## 실무 Best Practices

### 1. 성능 최적화

```yaml
# prometheus.yml 성능 튜닝
global:
  scrape_interval: 15s
  scrape_timeout: 10s

storage:
  tsdb:
    # 메모리 사용량 제한
    out_of_order_time_window: 10m
    # 블록 보관 기간
    retention.time: 15d
    # 디스크 사용량 기반 보관
    retention.size: 100GB

# 리소스 요청/제한 (Kubernetes)
resources:
  requests:
    memory: "2Gi"
    cpu: "500m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

```promql
# 비효율적인 쿼리 예시
# ❌ 모든 시계열 반환
http_requests_total

# ❌ 범위가 너무 넓음
rate(http_requests_total[30d])

# ❌ High Cardinality 레이블
sum(rate(http_requests_total[5m])) by (user_id)

# ✅ 효율적인 쿼리
sum(rate(http_requests_total{job="api"}[5m])) by (service)

# ✅ Recording Rule 활용
job:http_requests:rate5m
```

### 2. 메트릭 표준화

```yaml
# 팀 전체 메트릭 가이드라인
metric_guidelines:
  naming:
    format: "{namespace}_{subsystem}_{name}_{unit}_{suffix}"
    examples:
      - http_server_requests_seconds_total
      - db_query_duration_seconds
      - cache_hits_total

  labels:
    required:
      - service
      - environment
    optional:
      - version
      - instance
    forbidden:
      - user_id
      - session_id
      - request_id

  cardinality:
    max_label_values: 100
    max_timeseries_per_metric: 10000
```

### 3. 알림 설계 원칙

```yaml
# 알림 피라미드
alerting_strategy:
  page_worthy:
    description: "즉시 대응 필요, 담당자 호출"
    examples:
      - 서비스 완전 다운
      - 에러율 > 10%
      - SLO 위반
    notification: PagerDuty

  urgent:
    description: "업무 시간 내 확인 필요"
    examples:
      - 에러율 상승 (1-10%)
      - 지연 시간 증가
      - 리소스 80% 이상
    notification: Slack #alerts

  informational:
    description: "인지만 필요, 조치 불필요할 수 있음"
    examples:
      - 배포 완료
      - 스케일링 이벤트
    notification: Slack #info
```

### 4. 대시보드 관리

```yaml
# 대시보드 관리 정책
dashboard_governance:
  ownership:
    - 각 대시보드에 담당 팀 명시
    - 정기적 리뷰 (분기 1회)

  structure:
    - L1: 전체 시스템 Overview (SRE 관리)
    - L2: 서비스별 (서비스 팀 관리)
    - L3: 디버깅용 (개발 팀 관리)

  versioning:
    - Git으로 대시보드 JSON 관리
    - 변경 시 PR 필수

  performance:
    - 패널 수 20개 이하
    - 시간 범위 기본 1시간
    - 자동 새로고침 30초 이상
```

### 5. 확장성 고려

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Scalability Considerations                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  시계열 수 증가 대응:                                                    │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ <100K: 단일 Prometheus                                          │   │
│  │ 100K-1M: Prometheus + Remote Storage (Mimir/Thanos)             │   │
│  │ >1M: 샤딩 + Federation                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  권장 아키텍처 (대규모):                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │   │
│  │  Prometheus (per cluster)                                        │   │
│  │  ├── 상세 메트릭 (15일 보관)                                     │   │
│  │  ├── Recording Rules (집계)                                      │   │
│  │  └── Remote Write → Mimir                                        │   │
│  │                                                                  │   │
│  │  Grafana Mimir                                                   │   │
│  │  ├── 장기 저장 (1년+)                                            │   │
│  │  ├── 크로스 클러스터 쿼리                                        │   │
│  │  └── Object Storage (S3/GCS)                                     │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

### 공식 문서
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Alertmanager Documentation](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [PromQL Reference](https://prometheus.io/docs/prometheus/latest/querying/basics/)

### 추천 자료
- [Google SRE Book - Monitoring](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Prometheus: Up & Running](https://www.oreilly.com/library/view/prometheus-up/9781492034131/)
- [Grafana Fundamentals](https://grafana.com/tutorials/grafana-fundamentals/)
- [Awesome Prometheus Alerts](https://awesome-prometheus-alerts.grep.to/)

### 관련 문서
- [01-로깅](../01-로깅/README.md)
- [02-메트릭](../02-메트릭/README.md)
- [03-분산-추적](../03-분산-추적/README.md)

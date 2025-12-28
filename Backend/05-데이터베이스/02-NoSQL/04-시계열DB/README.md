# 시계열 데이터베이스

## 목차
1. [시계열 데이터베이스 개요](#시계열-데이터베이스-개요)
2. [InfluxDB](#influxdb)
3. [TimescaleDB](#timescaledb)
4. [비교 분석](#비교-분석)
5. [설계 패턴](#설계-패턴)
---

## 시계열 데이터베이스 개요

### 시계열 데이터란?

시계열 데이터는 시간 순서로 기록된 데이터로, 주로 타임스탬프와 함께 측정값을 저장합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    시계열 데이터 특성                             │
│                                                                  │
│  Time ──────────────────────────────────────────────────────►   │
│                                                                  │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐               │
│  │ P1  │ │ P2  │ │ P3  │ │ P4  │ │ P5  │ │ P6  │ ...           │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘               │
│   t=0    t=1     t=2     t=3     t=4     t=5                    │
│                                                                  │
│  특성:                                                          │
│  - 쓰기 집중 (Write-Heavy)                                      │
│  - 시간 기반 조회                                                │
│  - 최근 데이터 자주 접근                                         │
│  - 데이터 불변 (Immutable)                                       │
│  - 시간순 정렬                                                   │
│  - 대량의 데이터                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 주요 사용 사례

| 분야 | 예시 |
|------|------|
| IoT | 센서 데이터, 스마트 홈, 공장 모니터링 |
| DevOps | 서버 메트릭, 애플리케이션 로그, APM |
| 금융 | 주가, 거래량, 시세 데이터 |
| 에너지 | 전력 사용량, 스마트 미터링 |
| 물류 | 차량 위치 추적, 배송 상태 |

### TSDB vs RDBMS

```sql
-- RDBMS에서 시계열 데이터 (비효율적)
CREATE TABLE metrics (
    id SERIAL PRIMARY KEY,
    sensor_id VARCHAR(50),
    timestamp TIMESTAMP,
    value DOUBLE PRECISION
);

-- 문제점:
-- 1. INSERT 성능 저하 (인덱스 갱신)
-- 2. 대용량 데이터 관리 어려움
-- 3. 시간 기반 집계 최적화 없음
-- 4. 데이터 압축 미지원
-- 5. 자동 데이터 만료 없음
```

---

## InfluxDB

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                      InfluxDB 3.0                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   Query Engine                           │    │
│  │              (SQL + InfluxQL 지원)                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Storage Engine                          │    │
│  │              (Apache Arrow + Parquet)                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Object Storage                          │    │
│  │            (S3, GCS, Azure Blob, Local)                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  InfluxDB 1.x/2.x: TSM (Time-Structured Merge Tree)            │
│  InfluxDB 3.0: Apache Arrow columnar format                     │
└─────────────────────────────────────────────────────────────────┘
```

### 데이터 모델

```
┌─────────────────────────────────────────────────────────────────┐
│                     InfluxDB Line Protocol                       │
│                                                                  │
│  <measurement>,<tag_set> <field_set> <timestamp>                │
│                                                                  │
│  예시:                                                          │
│  cpu,host=server01,region=us-west usage=75.5,temp=42 1704067200 │
│      │              │                  │              │          │
│      │              │                  │              └ 타임스탬프│
│      │              │                  └ 필드 (측정값)           │
│      │              └ 태그 (인덱싱됨, 메타데이터)                │
│      └ Measurement (테이블과 유사)                               │
│                                                                  │
│  - 태그: 인덱싱됨, 카디널리티 낮은 메타데이터                    │
│  - 필드: 실제 측정값, 인덱싱 안됨                                │
│  - 시리즈: measurement + tag set 조합                           │
└─────────────────────────────────────────────────────────────────┘
```

### InfluxQL / Flux 쿼리

```sql
-- InfluxQL (SQL-like)
-- 데이터 조회
SELECT mean("usage") FROM "cpu"
WHERE "host" = 'server01'
AND time >= now() - 1h
GROUP BY time(5m), "region"

-- 연속 쿼리 (Continuous Query)
CREATE CONTINUOUS QUERY "cq_hourly" ON "mydb"
BEGIN
    SELECT mean("usage") AS "mean_usage"
    INTO "cpu_hourly"
    FROM "cpu"
    GROUP BY time(1h), *
END
```

```javascript
// Flux (InfluxDB 2.x)
from(bucket: "metrics")
    |> range(start: -1h)
    |> filter(fn: (r) => r["_measurement"] == "cpu")
    |> filter(fn: (r) => r["host"] == "server01")
    |> aggregateWindow(every: 5m, fn: mean, createEmpty: false)
    |> yield(name: "mean")

// 여러 소스 조인
cpu = from(bucket: "metrics")
    |> range(start: -1h)
    |> filter(fn: (r) => r["_measurement"] == "cpu")

mem = from(bucket: "metrics")
    |> range(start: -1h)
    |> filter(fn: (r) => r["_measurement"] == "memory")

join(tables: {cpu: cpu, mem: mem}, on: ["_time", "host"])
```

### 데이터 관리

```sql
-- Retention Policy (데이터 보존 정책)
CREATE RETENTION POLICY "one_week" ON "mydb"
DURATION 7d REPLICATION 1 DEFAULT

-- Shard Duration
-- 데이터를 시간 단위로 분리
CREATE RETENTION POLICY "one_year" ON "mydb"
DURATION 365d REPLICATION 1 SHARD DURATION 7d

-- Downsampling (다운샘플링)
-- 오래된 데이터는 해상도를 낮춰 저장
-- 1분 데이터 → 1시간 평균 → 1일 평균

-- 데이터 삭제
DELETE FROM "cpu" WHERE time < '2024-01-01'
DROP MEASUREMENT "cpu"
```

---

## TimescaleDB

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                      TimescaleDB                                 │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    PostgreSQL                            │    │
│  │              (완전한 SQL 지원)                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Timescale Extension                      │    │
│  │  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐  │    │
│  │  │  Hypertables  │  │ Continuous    │  │ Compression │  │    │
│  │  │  (자동 파티션) │  │ Aggregates   │  │             │  │    │
│  │  └───────────────┘  └───────────────┘  └─────────────┘  │    │
│  │  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐  │    │
│  │  │ Data Retention│  │ Distributed   │  │ Real-time   │  │    │
│  │  │ Policies      │  │ Hypertables   │  │ Analytics   │  │    │
│  │  └───────────────┘  └───────────────┘  └─────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  장점: PostgreSQL 생태계 + 시계열 최적화                        │
└─────────────────────────────────────────────────────────────────┘
```

### Hypertable 생성

```sql
-- 확장 활성화
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- 일반 테이블 생성
CREATE TABLE sensor_data (
    time        TIMESTAMPTZ NOT NULL,
    sensor_id   INTEGER NOT NULL,
    temperature DOUBLE PRECISION,
    humidity    DOUBLE PRECISION
);

-- Hypertable로 변환 (자동 파티셔닝)
SELECT create_hypertable('sensor_data', 'time');

-- 공간 파티셔닝 추가 (분산 환경)
SELECT create_hypertable('sensor_data', 'time',
    partitioning_column => 'sensor_id',
    number_partitions => 4
);

-- 청크 간격 설정 (기본 7일)
SELECT set_chunk_time_interval('sensor_data', INTERVAL '1 day');

-- Hypertable 정보 확인
SELECT * FROM timescaledb_information.hypertables;
SELECT * FROM timescaledb_information.chunks
WHERE hypertable_name = 'sensor_data';
```

### 시계열 쿼리

```sql
-- 시간 기반 집계
SELECT
    time_bucket('5 minutes', time) AS bucket,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp
FROM sensor_data
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket, sensor_id
ORDER BY bucket DESC;

-- 시간 가중 평균
SELECT
    time_bucket('1 hour', time) AS hour,
    sensor_id,
    time_weight('average', time, temperature) AS time_weighted_avg
FROM sensor_data
GROUP BY hour, sensor_id;

-- 최근 값 (Last Value)
SELECT DISTINCT ON (sensor_id)
    sensor_id,
    time,
    temperature
FROM sensor_data
ORDER BY sensor_id, time DESC;

-- 또는 TimescaleDB 함수 사용
SELECT
    sensor_id,
    last(temperature, time) AS last_temp,
    first(temperature, time) AS first_temp
FROM sensor_data
WHERE time > NOW() - INTERVAL '1 day'
GROUP BY sensor_id;

-- Gap Filling (누락 데이터 보간)
SELECT
    time_bucket_gapfill('1 hour', time) AS bucket,
    sensor_id,
    COALESCE(AVG(temperature), locf(AVG(temperature))) AS temp
FROM sensor_data
WHERE time BETWEEN '2024-01-01' AND '2024-01-02'
GROUP BY bucket, sensor_id;
-- locf: Last Observation Carried Forward
-- interpolate: 선형 보간
```

### Continuous Aggregates

```sql
-- 실시간 Materialized View (자동 갱신)
CREATE MATERIALIZED VIEW hourly_metrics
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    sensor_id,
    AVG(temperature) AS avg_temp,
    MAX(temperature) AS max_temp,
    MIN(temperature) AS min_temp,
    COUNT(*) AS sample_count
FROM sensor_data
GROUP BY bucket, sensor_id;

-- 갱신 정책 설정
SELECT add_continuous_aggregate_policy('hourly_metrics',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- 계층적 집계 (hourly → daily)
CREATE MATERIALIZED VIEW daily_metrics
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS day,
    sensor_id,
    AVG(avg_temp) AS avg_temp,
    MAX(max_temp) AS max_temp,
    MIN(min_temp) AS min_temp,
    SUM(sample_count) AS total_samples
FROM hourly_metrics
GROUP BY day, sensor_id;
```

### 압축과 보존

```sql
-- 압축 활성화
ALTER TABLE sensor_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id'
);

-- 7일 이상 된 청크 자동 압축
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');

-- 압축 상태 확인
SELECT
    chunk_name,
    before_compression_total_bytes,
    after_compression_total_bytes
FROM chunk_compression_stats('sensor_data');

-- 데이터 보존 정책 (90일 후 자동 삭제)
SELECT add_retention_policy('sensor_data', INTERVAL '90 days');

-- 수동 압축
SELECT compress_chunk(chunk) FROM show_chunks('sensor_data')
WHERE chunk_name = '_hyper_1_1_chunk';
```

---

## 비교 분석

### InfluxDB vs TimescaleDB

| 특성 | InfluxDB | TimescaleDB |
|------|----------|-------------|
| 기반 | Custom Storage | PostgreSQL Extension |
| 쿼리 언어 | InfluxQL / Flux | 표준 SQL |
| 생태계 | 독자적 | PostgreSQL 생태계 |
| JOIN | 제한적 | 완전 지원 |
| 트랜잭션 | 미지원 | ACID 지원 |
| 스키마 | Schemaless | Schema 필요 |
| 압축률 | 매우 높음 | 높음 |
| 복잡 쿼리 | 제한적 | 완전 지원 |
| 운영 복잡도 | 낮음 | 중간 |

### 성능 비교

```
┌─────────────────────────────────────────────────────────────────┐
│                      성능 특성 비교                              │
│                                                                  │
│  Cardinality (태그/컬럼 다양성)                                  │
│  ─────────────────────────────────────────────────────────────  │
│  낮음                                                      높음  │
│  │                                                          │    │
│  │   InfluxDB ████████████████                             │    │
│  │   TimescaleDB    ████████████████████████████████████   │    │
│  │                                                          │    │
│  InfluxDB: 낮은 카디널리티에서 우수                              │
│  TimescaleDB: 높은 카디널리티에서도 안정적                       │
│                                                                  │
│  복잡 쿼리 (JOIN, 윈도우 함수)                                   │
│  ─────────────────────────────────────────────────────────────  │
│  TimescaleDB: 완전한 SQL 지원으로 압도적                         │
│  InfluxDB: 제한적 (외부 조인 어려움)                             │
│                                                                  │
│  INSERT 성능                                                     │
│  ─────────────────────────────────────────────────────────────  │
│  InfluxDB: 간단한 데이터 모델로 약간 빠름                        │
│  TimescaleDB: PostgreSQL 오버헤드로 약간 느림                    │
│                                                                  │
│  디스크 압축                                                     │
│  ─────────────────────────────────────────────────────────────  │
│  InfluxDB: 매우 높은 압축률 (10:1 이상)                          │
│  TimescaleDB: 높은 압축률 (5:1 ~ 10:1)                          │
└─────────────────────────────────────────────────────────────────┘
```

### 선택 기준

**InfluxDB 선택:**
- 간단한 시계열 데이터
- 낮은 카디널리티
- Telegraf/Grafana 스택
- 빠른 설정과 운영

**TimescaleDB 선택:**
- 복잡한 분석 쿼리 필요
- 기존 PostgreSQL 활용
- JOIN, 트랜잭션 필요
- 높은 카디널리티 데이터
- 관계형 데이터와 혼합

---

## 설계 패턴

### 다운샘플링 전략

```sql
-- TimescaleDB: 계층적 저장
-- Raw: 1초 데이터 → 7일 보관
-- Hourly: 1시간 집계 → 1년 보관
-- Daily: 1일 집계 → 5년 보관

-- 1. Raw 데이터 테이블
CREATE TABLE sensor_data_raw (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INTEGER,
    value DOUBLE PRECISION
);
SELECT create_hypertable('sensor_data_raw', 'time');
SELECT add_retention_policy('sensor_data_raw', INTERVAL '7 days');

-- 2. Hourly 집계
CREATE MATERIALIZED VIEW sensor_data_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    sensor_id,
    AVG(value) AS avg_value,
    MAX(value) AS max_value
FROM sensor_data_raw
GROUP BY bucket, sensor_id;

-- 3. Daily 집계
CREATE MATERIALIZED VIEW sensor_data_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', bucket) AS day,
    sensor_id,
    AVG(avg_value) AS avg_value
FROM sensor_data_hourly
GROUP BY day, sensor_id;
```

### 고가용성 구성

```yaml
# InfluxDB Cluster (Enterprise)
# 또는 InfluxDB OSS + Kafka

# TimescaleDB 복제 설정 (PostgreSQL 기반)
# Primary
wal_level = replica
max_wal_senders = 3
synchronous_commit = on

# Standby
primary_conninfo = 'host=primary port=5432 user=replicator'
hot_standby = on
```

### 인덱싱 전략

```sql
-- TimescaleDB 인덱스
-- 기본: 시간 기반 인덱스 자동 생성

-- 추가 인덱스 (자주 조회하는 태그)
CREATE INDEX ON sensor_data (sensor_id, time DESC);

-- 복합 조건 쿼리용
CREATE INDEX ON sensor_data (sensor_id, time DESC)
WHERE value > 100;  -- 부분 인덱스

-- InfluxDB
-- 태그는 자동 인덱싱
-- 필드는 인덱스되지 않음 → 태그로 이동 고려
```

---

## 참고 자료

- [InfluxDB Documentation](https://docs.influxdata.com/)
- [TimescaleDB Documentation](https://docs.timescale.com/)
- [Time Series Benchmark Suite (TSBS)](https://github.com/timescale/tsbs)
- [Comparing InfluxDB, TimescaleDB, and QuestDB](https://questdb.io/blog/comparing-influxdb-timescaledb-questdb-time-series-databases/)
- Designing Data-Intensive Applications by Martin Kleppmann

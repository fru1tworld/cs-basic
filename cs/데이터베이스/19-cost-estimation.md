# 비용 추정 (Cost Estimation)

## 개요

비용 추정은 쿼리 옵티마이저가 **여러 실행 계획 중 최적의 계획을 선택**하기 위해 각 계획의 예상 비용을 계산하는 과정이다. 정확한 비용 추정은 좋은 실행 계획 선택의 핵심이며, 부정확한 추정은 성능 저하의 주요 원인이 된다.

## 핵심 개념

### 1. 비용 모델 구성 요소

```
┌─────────────────────────────────────────────────────────────┐
│              비용 모델 (Cost Model)                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Total Cost = I/O Cost + CPU Cost + Memory Cost + Network Cost│
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ I/O Cost (가장 중요)                                     │ │
│ │ - Sequential Read: 순차적 페이지 읽기                    │ │
│ │ - Random Read: 랜덤 페이지 읽기                          │ │
│ │ - Write: 페이지 쓰기                                     │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ CPU Cost                                                 │ │
│ │ - Tuple Processing: 각 행 처리 비용                      │ │
│ │ - Operator Evaluation: 조건식 평가                       │ │
│ │ - Function Call: 함수 호출 비용                          │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Memory Cost                                              │ │
│ │ - Hash Table 구축                                        │ │
│ │ - Sorting (in-memory vs disk)                            │ │
│ │ - Materialization                                        │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. PostgreSQL 비용 파라미터

```sql
-- 현재 비용 파라미터 확인
SHOW seq_page_cost;        -- 1.0 (기준)
SHOW random_page_cost;     -- 4.0 (SSD: 1.1~1.5 권장)
SHOW cpu_tuple_cost;       -- 0.01
SHOW cpu_index_tuple_cost; -- 0.005
SHOW cpu_operator_cost;    -- 0.0025
SHOW effective_cache_size; -- 버퍼 캐시 크기 추정

-- 비용 계산 예시
/*
Seq Scan Cost =
    (페이지 수 × seq_page_cost) +
    (행 수 × cpu_tuple_cost)

Index Scan Cost =
    (인덱스 페이지 수 × random_page_cost) +
    (테이블 페이지 수 × random_page_cost) +
    (인덱스 튜플 수 × cpu_index_tuple_cost) +
    (테이블 행 수 × cpu_tuple_cost)
*/
```

### 3. 카디널리티 추정 (Cardinality Estimation)

```
┌─────────────────────────────────────────────────────────────┐
│              카디널리티 추정                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 카디널리티 = 연산 결과로 예상되는 행 수                      │
│                                                             │
│ 1. 기본 테이블 카디널리티                                   │
│    - pg_class.reltuples에서 획득                           │
│    - ANALYZE 명령으로 갱신                                  │
│                                                             │
│ 2. 선택도 (Selectivity) 추정                               │
│    - 조건을 만족하는 행의 비율 (0.0 ~ 1.0)                  │
│    - 결과 카디널리티 = 입력 카디널리티 × 선택도             │
│                                                             │
│ 3. 선택도 공식                                              │
│    ┌─────────────────────────────────────────────────────┐ │
│    │ 조건식             │ 선택도 공식                     │ │
│    ├─────────────────────────────────────────────────────┤ │
│    │ col = value        │ 1/n_distinct 또는 MCV lookup    │ │
│    │ col > value        │ (max - value)/(max - min)      │ │
│    │ col < value        │ (value - min)/(max - min)      │ │
│    │ col BETWEEN a AND b│ (b - a)/(max - min)            │ │
│    │ col LIKE 'abc%'    │ 히스토그램 기반                 │ │
│    │ col IS NULL        │ null_frac                      │ │
│    │ A AND B            │ sel(A) × sel(B)  *독립 가정    │ │
│    │ A OR B             │ sel(A) + sel(B) - sel(A)×sel(B)│ │
│    │ NOT A              │ 1 - sel(A)                     │ │
│    └─────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4. 통계 정보

```sql
-- 테이블 통계 확인
SELECT
    relname,
    reltuples,      -- 추정 행 수
    relpages,       -- 페이지 수
    relallvisible   -- 모든 행이 visible한 페이지 수
FROM pg_class
WHERE relname = 'orders';

-- 컬럼 통계 확인
SELECT
    attname,
    n_distinct,     -- 고유값 수 (-1 = 모두 고유)
    null_frac,      -- NULL 비율
    avg_width,      -- 평균 바이트 크기
    correlation     -- 물리적 정렬 상관관계
FROM pg_stats
WHERE tablename = 'orders';

-- 히스토그램 확인
SELECT
    attname,
    histogram_bounds,  -- 구간 경계값 배열
    most_common_vals,  -- 최빈값 배열
    most_common_freqs  -- 최빈값 빈도 배열
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

### 5. 히스토그램 기반 추정

```
┌─────────────────────────────────────────────────────────────┐
│              히스토그램 기반 선택도 추정                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Equi-Width Histogram (동일 너비)                            │
│                                                             │
│ 행 수│                                                      │
│      │    ████                                               │
│  100 │    ████ ████                                         │
│      │    ████ ████ ████                                    │
│   50 │████████████████████████████                          │
│      │████████████████████████████████                      │
│      └───────────────────────────────→ 값                   │
│        0-20 20-40 40-60 60-80 80-100                        │
│                                                             │
│ Equi-Depth Histogram (동일 깊이) - PostgreSQL 사용          │
│                                                             │
│ 행 수│                                                      │
│      │████ ████ ████ ████ ████  ← 각 버킷 동일 행 수       │
│  100 │████ ████ ████ ████ ████                              │
│      │████ ████ ████ ████ ████                              │
│      └───────────────────────────────→ 값                   │
│        0-5 5-15 15-35 35-80 80-100   ← 버킷 너비 가변      │
│                                                             │
│ PostgreSQL default_statistics_target = 100                  │
│ → 100개 버킷의 히스토그램 생성                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 통계 상세 수준 조정
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 200;
ANALYZE orders;

-- 전역 통계 목표값 조정
SET default_statistics_target = 200;
```

### 6. Most Common Values (MCV)

```sql
-- MCV 통계 확인
SELECT
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';

-- 결과 예시:
-- most_common_vals: {pending, shipped, delivered, cancelled}
-- most_common_freqs: {0.35, 0.30, 0.25, 0.10}

-- 선택도 계산
-- SELECT * FROM orders WHERE status = 'pending'
-- 선택도 = 0.35 (MCV에서 직접 조회)

-- SELECT * FROM orders WHERE status = 'returned'
-- 'returned'가 MCV에 없으면:
-- 선택도 = (1 - sum(mcv_freqs)) / (n_distinct - mcv_count)
```

```
┌─────────────────────────────────────────────────────────────┐
│              MCV와 히스토그램의 역할                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 값 분포:                                                     │
│ ┌──────────────────────────────────────────┐               │
│ │  ████                                      │ ← MCV 영역  │
│ │  ████ ███                                  │   (빈번한 값)│
│ │  ████ ███ ██                               │              │
│ │  ████ ███ ██  █ █ █ █ █ █ █ █ █          │ ← 히스토그램│
│ │  ████ ███ ██  █ █ █ █ █ █ █ █ █          │   (나머지)   │
│ └──────────────────────────────────────────┘               │
│   A    B   C    (나머지 값들)                                │
│                                                             │
│ 조회 순서:                                                   │
│ 1. 값이 MCV에 있는지 확인 → 빈도 직접 사용                  │
│ 2. MCV에 없으면 → 히스토그램에서 추정                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7. 조인 카디널리티 추정

```
┌─────────────────────────────────────────────────────────────┐
│              조인 카디널리티 추정                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ R ⋈ S on R.a = S.b                                          │
│                                                             │
│ 기본 공식 (Uniform Distribution 가정):                      │
│                                                             │
│ |R ⋈ S| = |R| × |S| / max(V(R.a), V(S.b))                  │
│                                                             │
│ 여기서:                                                      │
│ - |R|, |S|: 테이블의 행 수                                  │
│ - V(R.a): R.a 컬럼의 고유값 수 (n_distinct)                 │
│                                                             │
│ 예시:                                                        │
│ - orders: 100,000 행, customer_id 고유값 10,000개           │
│ - customers: 10,000 행, id 고유값 10,000개                  │
│                                                             │
│ 조인 결과 = 100,000 × 10,000 / max(10,000, 10,000)          │
│          = 1,000,000,000 / 10,000                           │
│          = 100,000 행                                        │
│                                                             │
│ Foreign Key 관계:                                            │
│ - customer_id가 외래 키면 → 정확히 100,000행 (1:1 매칭)    │
│ - PostgreSQL은 FK 정보를 활용하지 않음 (수동 추정)         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8. 비용 추정 오류 원인

```
┌─────────────────────────────────────────────────────────────┐
│              비용 추정 오류의 원인                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. 상관관계 무시 (Correlation)                              │
│    - city='Seoul' AND country='Korea'                       │
│    - 독립 가정: sel = 0.01 × 0.1 = 0.001                    │
│    - 실제: sel ≈ 0.01 (강한 상관관계)                       │
│                                                             │
│ 2. 오래된 통계                                               │
│    - 대량 INSERT/DELETE 후 ANALYZE 미실행                   │
│    - 추정 행 100, 실제 행 100,000                           │
│                                                             │
│ 3. 히스토그램 부족                                           │
│    - default_statistics_target이 낮음                        │
│    - 편향된 데이터 분포 캡처 실패                           │
│                                                             │
│ 4. 함수/표현식                                               │
│    - WHERE UPPER(name) = 'JOHN'                             │
│    - 함수 적용 결과의 분포를 알 수 없음                     │
│    - 기본 선택도 0.5% 가정 (PostgreSQL)                     │
│                                                             │
│ 5. 파라미터화된 쿼리                                        │
│    - WHERE id = $1                                          │
│    - 실제 값 모름 → 일반적 추정치 사용                      │
│                                                             │
│ 6. 복잡한 조건                                               │
│    - OR, IN, BETWEEN 복합 조건                              │
│    - 조건 간 의존성 무시                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 추정 오류 확인
EXPLAIN (ANALYZE)
SELECT * FROM orders
WHERE status = 'pending' AND amount > 1000;

-- 출력:
-- Seq Scan on orders (rows=100) (actual rows=5000)
--                    ^^^^^               ^^^^^
--                    예상                실제

-- rows 예상치와 actual rows의 큰 차이 → 통계 갱신 필요
ANALYZE orders;
```

### 9. 확장 통계 (Extended Statistics)

```sql
-- PostgreSQL 10+ : 다중 컬럼 통계

-- 상관관계가 있는 컬럼에 대한 통계 생성
CREATE STATISTICS orders_city_country (dependencies)
    ON city, country FROM orders;

-- 함수적 종속성 통계 (A → B)
CREATE STATISTICS order_status_deps (dependencies)
    ON order_status, payment_status FROM orders;

-- N-distinct 통계 (컬럼 조합의 고유값 수)
CREATE STATISTICS order_ndistinct (ndistinct)
    ON customer_id, product_id FROM orders;

-- 복합 통계 (MCV for multiple columns)
CREATE STATISTICS order_mcv (mcv)
    ON status, region FROM orders;

-- 통계 갱신
ANALYZE orders;

-- 확장 통계 확인
SELECT * FROM pg_statistic_ext;
SELECT * FROM pg_statistic_ext_data;
```

### 10. 스캔 연산 비용

```
┌─────────────────────────────────────────────────────────────┐
│              스캔 연산별 비용 계산                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Sequential Scan                                             │
│ ───────────────                                             │
│ startup_cost = 0                                            │
│ total_cost = (relpages × seq_page_cost) +                  │
│              (reltuples × cpu_tuple_cost) +                │
│              (reltuples × cpu_operator_cost × num_quals)   │
│                                                             │
│ Index Scan                                                  │
│ ──────────                                                  │
│ startup_cost = index_startup_cost                          │
│ total_cost =                                                │
│   (index_pages × random_page_cost) +                       │
│   (index_tuples × cpu_index_tuple_cost) +                  │
│   (table_pages × random_page_cost) +  ← 랜덤 액세스       │
│   (table_tuples × cpu_tuple_cost)                          │
│                                                             │
│ Index Only Scan                                             │
│ ───────────────                                             │
│ total_cost =                                                │
│   (index_pages × random_page_cost) +                       │
│   (index_tuples × cpu_index_tuple_cost)                    │
│   ← 테이블 액세스 비용 없음 (Visibility Map 조건부)         │
│                                                             │
│ Bitmap Heap Scan                                            │
│ ────────────────                                            │
│ total_cost =                                                │
│   (bitmap_index_scan_cost) +                               │
│   (heap_pages × mix_of_seq_and_random) +  ← 순차성 고려   │
│   (heap_tuples × cpu_tuple_cost)                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 실무 적용

### 비용 추정 디버깅

```sql
-- 1. 예상 vs 실제 비교
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 12345;

-- 2. 통계 정확성 확인
SELECT
    n_distinct,
    null_frac,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'customer_id';

-- 3. 통계 갱신
ANALYZE orders;

-- 4. 특정 컬럼 통계 정밀도 향상
ALTER TABLE orders
    ALTER COLUMN customer_id SET STATISTICS 500;
ANALYZE orders;

-- 5. SSD 사용 시 비용 파라미터 조정
SET random_page_cost = 1.1;  -- HDD: 4.0, SSD: 1.1~1.5
SET effective_cache_size = '4GB';  -- 실제 캐시 가능 메모리

-- 6. 쿼리별 힌트 (pg_hint_plan 확장)
/*+ SeqScan(orders) */
SELECT * FROM orders WHERE id = 1;
```

### 통계 관리 베스트 프랙티스

```sql
-- 자동 VACUUM/ANALYZE 설정 확인
SELECT name, setting
FROM pg_settings
WHERE name LIKE 'autovacuum%';

-- 테이블별 자동 분석 임계값 설정
ALTER TABLE high_update_table
    SET (autovacuum_analyze_threshold = 1000,
         autovacuum_analyze_scale_factor = 0.05);

-- 수동 통계 갱신 (대량 변경 후)
ANALYZE VERBOSE orders;

-- 전체 데이터베이스 분석
ANALYZE;
```

## 참고 자료

- PostgreSQL Documentation: Planner Cost Constants
- "PostgreSQL Query Optimization" - Henrietta Dombrovskaya
- CMU 15-445: Query Optimization (Cost Estimation)
- "Database System Concepts" - Chapter 16

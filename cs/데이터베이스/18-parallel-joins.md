# 병렬 조인 (Parallel Joins)

## 개요

병렬 조인은 대용량 데이터 처리를 위해 **조인 연산을 여러 프로세서/스레드에 분산**하여 실행하는 기법이다. 단일 스레드로는 처리하기 힘든 대규모 조인을 효율적으로 수행할 수 있다. 현대 데이터베이스는 멀티코어 CPU를 활용한 Intra-Query Parallelism을 통해 쿼리 응답 시간을 크게 단축한다.

## 핵심 개념

### 1. 병렬 처리 유형

```
┌─────────────────────────────────────────────────────────────┐
│              데이터베이스 병렬 처리 유형                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. Inter-Query Parallelism                                  │
│    ┌───────┐ ┌───────┐ ┌───────┐                           │
│    │Query1 │ │Query2 │ │Query3 │  ← 서로 다른 쿼리 병렬     │
│    └───────┘ └───────┘ └───────┘                           │
│                                                             │
│ 2. Intra-Query Parallelism                                  │
│    ┌─────────────────────────────────┐                     │
│    │           Query 1               │                     │
│    │  ┌─────┐ ┌─────┐ ┌─────┐       │  ← 단일 쿼리 내부   │
│    │  │Work1│ │Work2│ │Work3│       │    여러 워커 병렬    │
│    │  └─────┘ └─────┘ └─────┘       │                     │
│    └─────────────────────────────────┘                     │
│                                                             │
│ 3. Intra-Operator vs Inter-Operator                        │
│    - Intra-Operator: 하나의 연산자를 병렬화                 │
│    - Inter-Operator: 여러 연산자를 파이프라인 병렬화        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. 데이터 분배 전략

```
┌─────────────────────────────────────────────────────────────┐
│              데이터 파티셔닝 전략                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. Range Partitioning                                       │
│    ┌────────────────────────────────┐                      │
│    │ A-F   │ G-L   │ M-R   │ S-Z   │                       │
│    │Worker1│Worker2│Worker3│Worker4│                       │
│    └────────────────────────────────┘                      │
│    장점: 범위 쿼리에 유리                                   │
│    단점: 데이터 편향 가능                                   │
│                                                             │
│ 2. Hash Partitioning                                        │
│    ┌────────────────────────────────┐                      │
│    │hash%4=0│hash%4=1│hash%4=2│hash%4=3│                   │
│    │Worker1 │Worker2 │Worker3 │Worker4 │                   │
│    └────────────────────────────────┘                      │
│    장점: 균등 분배, 등가 조인에 최적                        │
│    단점: 범위 쿼리 비효율                                   │
│                                                             │
│ 3. Round-Robin Partitioning                                 │
│    Row 1 → Worker1                                          │
│    Row 2 → Worker2                                          │
│    Row 3 → Worker3                                          │
│    Row 4 → Worker1 ...                                      │
│    장점: 완벽한 균등 분배                                   │
│    단점: 조인에 비효율적 (전체 브로드캐스트 필요)           │
│                                                             │
│ 4. Broadcast                                                │
│    작은 테이블을 모든 워커에 복제                           │
│    ┌────────────────────────────────┐                      │
│    │ Full  │ Full  │ Full  │ Full  │  ← 작은 테이블        │
│    │Copy   │Copy   │Copy   │Copy   │                       │
│    │Worker1│Worker2│Worker3│Worker4│                       │
│    └────────────────────────────────┘                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. 병렬 Hash Join

```
┌─────────────────────────────────────────────────────────────┐
│              Parallel Hash Join                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Phase 1: Parallel Build                                     │
│                                                             │
│   Build Table (Partitioned)                                 │
│   ┌─────────────────────────────────┐                      │
│   │ Part1 │ Part2 │ Part3 │ Part4 │                        │
│   └───┬───┴───┬───┴───┬───┴───┬───┘                        │
│       │       │       │       │                             │
│       ▼       ▼       ▼       ▼                             │
│   ┌──────┐┌──────┐┌──────┐┌──────┐                         │
│   │Hash  ││Hash  ││Hash  ││Hash  │  Build Workers          │
│   │Table1││Table2││Table3││Table4│                         │
│   └──────┘└──────┘└──────┘└──────┘                         │
│                                                             │
│ Phase 2: Parallel Probe                                     │
│                                                             │
│   Probe Table (Partitioned by same key)                     │
│   ┌─────────────────────────────────┐                      │
│   │ Part1 │ Part2 │ Part3 │ Part4 │                        │
│   └───┬───┴───┬───┴───┬───┴───┬───┘                        │
│       │       │       │       │                             │
│       ▼       ▼       ▼       ▼                             │
│   ┌──────┐┌──────┐┌──────┐┌──────┐                         │
│   │Probe ││Probe ││Probe ││Probe │  Probe Workers          │
│   │Work1 ││Work2 ││Work3 ││Work4 │                         │
│   └──┬───┘└──┬───┘└──┬───┘└──┬───┘                         │
│      │       │       │       │                              │
│      └───────┴───────┴───────┘                              │
│                    │                                        │
│                    ▼                                        │
│              ┌──────────┐                                   │
│              │  Gather  │                                   │
│              └──────────┘                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL 병렬 Hash Join 예시
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0.01;
SET parallel_setup_cost = 100;

EXPLAIN (ANALYZE, VERBOSE)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_date > '2024-01-01';

/*
Gather  (actual time=50.0..150.0 rows=100000)
  Workers Planned: 4
  Workers Launched: 4
  ->  Parallel Hash Join  (actual time=40.0..100.0 rows=25000 loops=5)
        Hash Cond: (o.customer_id = c.id)
        ->  Parallel Seq Scan on orders o  (actual time=0.1..30.0 rows=50000 loops=5)
              Filter: (order_date > '2024-01-01')
        ->  Parallel Hash  (actual time=20.0..20.0 rows=10000 loops=5)
              ->  Parallel Seq Scan on customers c  (actual time=0.1..10.0 rows=10000 loops=5)
*/
```

### 4. 병렬 Merge Join

```
┌─────────────────────────────────────────────────────────────┐
│              Parallel Merge Join                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 방법 1: Parallel Sort + Sequential Merge                    │
│                                                             │
│   Table R              Table S                              │
│   ┌─────┐              ┌─────┐                              │
│   │Part1│              │Part1│                              │
│   │Part2│              │Part2│                              │
│   │Part3│              │Part3│                              │
│   │Part4│              │Part4│                              │
│   └──┬──┘              └──┬──┘                              │
│      │                    │                                 │
│      ▼                    ▼                                 │
│   Parallel Sort       Parallel Sort                         │
│      │                    │                                 │
│      ▼                    ▼                                 │
│   ┌─────┐              ┌─────┐                              │
│   │Sorted│             │Sorted│                              │
│   │  R  │──────────────│  S  │                              │
│   └─────┘   Merge      └─────┘                              │
│                                                             │
│ 방법 2: Range Partition + Parallel Merge                    │
│                                                             │
│   R (Range Part)          S (Range Part)                    │
│   ┌──────────┐           ┌──────────┐                       │
│   │ 1-100    │──Merge──→│ 1-100    │ → Worker1             │
│   │101-200   │──Merge──→│101-200   │ → Worker2             │
│   │201-300   │──Merge──→│201-300   │ → Worker3             │
│   │301-400   │──Merge──→│301-400   │ → Worker4             │
│   └──────────┘           └──────────┘                       │
│                                                             │
│   ※ 각 파티션 내에서 독립적으로 Merge 수행                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. 병렬 Nested Loop Join

```
┌─────────────────────────────────────────────────────────────┐
│              Parallel Nested Loop Join                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Outer Table을 파티셔닝, Inner Table은 모든 워커가 접근      │
│                                                             │
│   Outer Table (orders)                                      │
│   ┌─────────────────────────────────┐                      │
│   │ Part1 │ Part2 │ Part3 │ Part4 │                        │
│   └───┬───┴───┬───┴───┬───┴───┬───┘                        │
│       │       │       │       │                             │
│       ▼       ▼       ▼       ▼                             │
│   ┌──────┐┌──────┐┌──────┐┌──────┐                         │
│   │Worker││Worker││Worker││Worker│                         │
│   │  1   ││  2   ││  3   ││  4   │                         │
│   └──┬───┘└──┬───┘└──┬───┘└──┬───┘                         │
│      │       │       │       │                              │
│      └───────┼───────┼───────┘                              │
│              │       │                                      │
│              ▼       ▼                                      │
│         ┌─────────────────┐                                 │
│         │  Inner Table    │  ← 인덱스 사용                  │
│         │  (customers)    │    모든 워커가 공유 접근        │
│         └─────────────────┘                                 │
│                                                             │
│ 적합한 경우:                                                 │
│ - Outer가 크고, Inner에 인덱스가 있는 경우                  │
│ - 각 Outer 행당 소수의 Inner 행만 매칭                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL Parallel Nested Loop
EXPLAIN (ANALYZE)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.amount > 1000;

/*
Gather  (actual time=1.0..50.0 rows=5000)
  Workers Planned: 2
  Workers Launched: 2
  ->  Nested Loop  (actual time=0.5..40.0 rows=1667 loops=3)
        ->  Parallel Seq Scan on orders o  (actual time=0.1..10.0 rows=1667 loops=3)
              Filter: (amount > 1000)
        ->  Index Scan using customers_pkey on customers c  (actual time=0.01..0.01 rows=1 loops=5000)
              Index Cond: (id = o.customer_id)
*/
```

### 6. PostgreSQL 병렬 처리 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL Parallel Query                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Client                                                    │
│     │                                                       │
│     ▼                                                       │
│   ┌────────────────────────────────────┐                   │
│   │         Leader Process             │                   │
│   │  (Backend that received query)     │                   │
│   └────────────────┬───────────────────┘                   │
│                    │                                        │
│         ┌──────────┼──────────┐                            │
│         │          │          │                             │
│         ▼          ▼          ▼                             │
│   ┌──────────┐┌──────────┐┌──────────┐                     │
│   │ Worker 1 ││ Worker 2 ││ Worker 3 │                     │
│   │(bgworker)││(bgworker)││(bgworker)│                     │
│   └────┬─────┘└────┬─────┘└────┬─────┘                     │
│        │           │           │                            │
│        └───────────┼───────────┘                            │
│                    │                                        │
│                    ▼                                        │
│   ┌────────────────────────────────────┐                   │
│   │            Gather Node             │                   │
│   │  - 워커 결과 수집                   │                   │
│   │  - tuple queue 기반 통신           │                   │
│   └────────────────────────────────────┘                   │
│                                                             │
│ 관련 파라미터:                                               │
│ - max_parallel_workers = 8                                  │
│ - max_parallel_workers_per_gather = 2                       │
│ - min_parallel_table_scan_size = 8MB                        │
│ - min_parallel_index_scan_size = 512kB                      │
│ - parallel_tuple_cost = 0.1                                 │
│ - parallel_setup_cost = 1000                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7. 병렬 처리 제약사항

```
┌─────────────────────────────────────────────────────────────┐
│              병렬 쿼리 제약사항                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ PostgreSQL에서 병렬 처리 불가능한 경우:                     │
│                                                             │
│ 1. 쿼리 유형                                                │
│    - INSERT/UPDATE/DELETE (단, SELECT INTO는 가능)         │
│    - CTE (WITH) 쿼리                                        │
│    - CURSOR 사용                                            │
│    - 트랜잭션이 아닌 함수 호출                              │
│                                                             │
│ 2. 함수 특성                                                │
│    - PARALLEL UNSAFE 함수 사용                              │
│    - PARALLEL RESTRICTED 함수 (Leader에서만 실행)           │
│                                                             │
│ 3. 테이블 특성                                              │
│    - 임시 테이블                                            │
│    - 너무 작은 테이블 (min_parallel_table_scan_size 미만)   │
│                                                             │
│ 4. 잠금                                                     │
│    - FOR UPDATE/SHARE 절 사용                               │
│    - 직렬화 가능 격리 수준                                  │
│                                                             │
│ -- 함수 병렬 안전성 확인                                    │
│ SELECT proname, proparallel                                 │
│ FROM pg_proc                                                │
│ WHERE proname = 'my_function';                              │
│ -- 's' = safe, 'r' = restricted, 'u' = unsafe              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8. 병렬 집계 (Parallel Aggregation)

```
┌─────────────────────────────────────────────────────────────┐
│              Parallel Aggregation                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 2단계 집계 방식:                                             │
│                                                             │
│ Step 1: Partial Aggregate (각 워커에서)                     │
│                                                             │
│   Worker1: SUM(x) = 100, COUNT(*) = 10                     │
│   Worker2: SUM(x) = 150, COUNT(*) = 15                     │
│   Worker3: SUM(x) = 120, COUNT(*) = 12                     │
│                                                             │
│ Step 2: Finalize Aggregate (Leader에서)                     │
│                                                             │
│   Final SUM = 100 + 150 + 120 = 370                        │
│   Final COUNT = 10 + 15 + 12 = 37                          │
│   AVG = 370 / 37 = 10                                       │
│                                                             │
│ -- EXPLAIN 출력 예시                                        │
│ Finalize Aggregate                                          │
│   ->  Gather                                                │
│         Workers Planned: 3                                  │
│         ->  Partial Aggregate                               │
│               ->  Parallel Seq Scan                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 병렬 GROUP BY 예시
EXPLAIN (ANALYZE)
SELECT dept_id, AVG(salary)
FROM employees
GROUP BY dept_id;

/*
Finalize GroupAggregate  (actual time=100.0..110.0 rows=50)
  Group Key: dept_id
  ->  Gather Merge  (actual time=90.0..100.0 rows=150)
        Workers Planned: 3
        Workers Launched: 3
        ->  Sort  (actual time=80.0..82.0 rows=50 loops=4)
              Sort Key: dept_id
              ->  Partial HashAggregate  (actual time=70.0..75.0 rows=50 loops=4)
                    Group Key: dept_id
                    ->  Parallel Seq Scan on employees  (actual time=0.1..50.0 rows=25000 loops=4)
*/
```

## 실무 적용

### 병렬 쿼리 튜닝

```sql
-- 1. 현재 병렬 설정 확인
SHOW max_parallel_workers;
SHOW max_parallel_workers_per_gather;
SHOW parallel_tuple_cost;
SHOW parallel_setup_cost;

-- 2. 병렬 처리 강제 활성화 (테스트용)
SET force_parallel_mode = on;
SET parallel_tuple_cost = 0;
SET parallel_setup_cost = 0;

-- 3. 테이블별 병렬 워커 수 설정
ALTER TABLE large_table SET (parallel_workers = 4);

-- 4. 병렬 처리 상태 모니터링
SELECT * FROM pg_stat_activity
WHERE backend_type = 'parallel worker';

-- 5. 병렬 처리 비활성화 (디버깅용)
SET max_parallel_workers_per_gather = 0;
```

### 최적의 병렬 워커 수 결정

```
┌─────────────────────────────────────────────────────────────┐
│              병렬 워커 수 결정 기준                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 고려 요소:                                                   │
│                                                             │
│ 1. CPU 코어 수                                               │
│    - max_parallel_workers ≤ CPU cores                       │
│    - 다른 워크로드 고려하여 여유 두기                        │
│                                                             │
│ 2. 테이블 크기                                               │
│    - 작은 테이블: 병렬 오버헤드 > 이득                       │
│    - 임계값: min_parallel_table_scan_size (default 8MB)     │
│                                                             │
│ 3. 쿼리 특성                                                 │
│    - I/O bound: 워커 수 늘려도 효과 제한적                   │
│    - CPU bound: 워커 수에 비례하여 성능 향상                 │
│                                                             │
│ 4. 동시 쿼리 수                                              │
│    - 동시에 10개 쿼리 × 워커 4개 = 40 프로세스              │
│    - 컨텍스트 스위칭 오버헤드 고려                           │
│                                                             │
│ 권장 설정:                                                   │
│ - OLAP: max_parallel_workers_per_gather = 4~8               │
│ - OLTP: max_parallel_workers_per_gather = 2~4               │
│ - 혼합: 동적 조정 또는 낮게 설정                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 참고 자료

- PostgreSQL Documentation: Parallel Query
- "Database System Concepts" - Silberschatz, Chapter 18
- CMU 15-445: Parallel Query Execution
- "Designing Data-Intensive Applications" - Martin Kleppmann

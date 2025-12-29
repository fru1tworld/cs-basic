# 실행 계획 분석 (EXPLAIN Analysis)

## 개요

EXPLAIN은 쿼리 옵티마이저가 선택한 **실행 계획을 시각화**하는 도구다. 실행 계획을 읽고 해석하는 능력은 쿼리 튜닝의 핵심이다. 예상 비용과 실제 성능의 차이를 분석하고, 병목 지점을 식별하여 최적화 방향을 결정할 수 있다.

## 핵심 개념

### 1. EXPLAIN 기본 구조

```sql
-- PostgreSQL
EXPLAIN SELECT * FROM employees WHERE dept_id = 10;

-- 결과:
-- Seq Scan on employees  (cost=0.00..12.50 rows=100 width=64)
--   Filter: (dept_id = 10)

-- 구성 요소:
-- 1. 연산자: Seq Scan
-- 2. 대상: employees
-- 3. 비용: (startup_cost..total_cost)
-- 4. 예상 행 수: rows=100
-- 5. 행 폭: width=64 (bytes)
```

### 2. 비용 모델 이해

```
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL 비용 모델                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ cost = (startup_cost)..(total_cost)                        │
│                                                             │
│ startup_cost: 첫 행 반환까지의 비용                        │
│ total_cost: 모든 행 반환까지의 비용                        │
│                                                             │
│ 비용 = I/O 비용 + CPU 비용                                  │
│                                                             │
│ 주요 비용 파라미터 (postgresql.conf):                       │
│ - seq_page_cost = 1.0        (순차 페이지 읽기)            │
│ - random_page_cost = 4.0     (랜덤 페이지 읽기)            │
│ - cpu_tuple_cost = 0.01      (튜플 처리)                   │
│ - cpu_index_tuple_cost = 0.005 (인덱스 튜플 처리)          │
│ - cpu_operator_cost = 0.0025 (연산자 처리)                 │
│                                                             │
│ 예: Seq Scan 비용                                           │
│ cost = pages × seq_page_cost + tuples × cpu_tuple_cost     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. EXPLAIN 옵션

```sql
-- PostgreSQL
EXPLAIN (ANALYZE)     -- 실제 실행 및 시간 측정
EXPLAIN (BUFFERS)     -- 버퍼 사용량 표시
EXPLAIN (VERBOSE)     -- 상세 정보
EXPLAIN (COSTS)       -- 비용 표시 (기본값)
EXPLAIN (FORMAT JSON) -- JSON 형식 출력
EXPLAIN (FORMAT YAML) -- YAML 형식 출력

-- 조합
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM employees WHERE dept_id = 10;
```

### 4. 주요 연산자 노드

```
┌─────────────────────────────────────────────────────────────┐
│              실행 계획 연산자                                 │
├─────────────────────────────────────────────────────────────┤
│ 스캔 연산자:                                                 │
│ - Seq Scan: 순차 테이블 스캔                                │
│ - Index Scan: 인덱스 사용, 테이블 접근                      │
│ - Index Only Scan: 인덱스만으로 결과 반환                   │
│ - Bitmap Heap Scan: 비트맵으로 페이지 수집 후 스캔          │
│ - TID Scan: 물리적 위치로 직접 접근                        │
│                                                             │
│ 조인 연산자:                                                 │
│ - Nested Loop: 중첩 루프 조인                               │
│ - Hash Join: 해시 테이블 기반 조인                          │
│ - Merge Join: 정렬된 데이터 병합 조인                       │
│                                                             │
│ 집계 연산자:                                                 │
│ - Aggregate: 단순 집계                                      │
│ - HashAggregate: 해시 기반 그룹화                           │
│ - GroupAggregate: 정렬 기반 그룹화                          │
│                                                             │
│ 기타:                                                        │
│ - Sort: 정렬                                                │
│ - Limit: 결과 제한                                          │
│ - Materialize: 중간 결과 물리화                             │
│ - Append: UNION 등 결과 연결                                │
│ - Subquery Scan: 서브쿼리 스캔                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. 실행 계획 읽기

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT e.name, d.dept_name, SUM(s.amount)
FROM employees e
JOIN departments d ON e.dept_id = d.id
JOIN sales s ON s.emp_id = e.id
WHERE d.region = 'Seoul'
GROUP BY e.name, d.dept_name
ORDER BY SUM(s.amount) DESC
LIMIT 10;

-- 결과 예시 (들여쓰기가 계층 구조를 나타냄):
/*
Limit  (cost=1234.56..1234.58 rows=10 width=48) (actual time=15.2..15.3 rows=10 loops=1)
  ->  Sort  (cost=1234.56..1236.06 rows=600 width=48) (actual time=15.2..15.2 rows=10 loops=1)
        Sort Key: (sum(s.amount)) DESC
        Sort Method: top-N heapsort  Memory: 25kB
        ->  HashAggregate  (cost=1200.00..1212.00 rows=600 width=48) (actual time=14.5..14.8 rows=500 loops=1)
              Group Key: e.name, d.dept_name
              Batches: 1  Memory Usage: 200kB
              ->  Hash Join  (cost=100.00..1000.00 rows=5000 width=40) (actual time=5.0..12.0 rows=5000 loops=1)
                    Hash Cond: (s.emp_id = e.id)
                    ->  Seq Scan on sales s  (cost=0.00..500.00 rows=20000 width=12) (actual time=0.01..2.0 rows=20000 loops=1)
                          Buffers: shared hit=100
                    ->  Hash  (cost=80.00..80.00 rows=800 width=32) (actual time=4.5..4.5 rows=800 loops=1)
                          Buckets: 1024  Batches: 1  Memory Usage: 50kB
                          ->  Hash Join  (cost=20.00..80.00 rows=800 width=32) (actual time=1.0..4.0 rows=800 loops=1)
                                Hash Cond: (e.dept_id = d.id)
                                ->  Seq Scan on employees e  (cost=0.00..40.00 rows=1000 width=20) (actual time=0.01..1.0 rows=1000 loops=1)
                                      Buffers: shared hit=20
                                ->  Hash  (cost=15.00..15.00 rows=20 width=16) (actual time=0.5..0.5 rows=20 loops=1)
                                      Buckets: 1024  Batches: 1  Memory Usage: 10kB
                                      ->  Seq Scan on departments d  (cost=0.00..15.00 rows=20 width=16) (actual time=0.01..0.4 rows=20 loops=1)
                                            Filter: (region = 'Seoul'::text)
                                            Rows Removed by Filter: 80
                                            Buffers: shared hit=5
Planning Time: 0.5 ms
Execution Time: 16.0 ms
*/
```

### 6. 핵심 분석 포인트

```
┌─────────────────────────────────────────────────────────────┐
│              분석 체크리스트                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. 예상 vs 실제 행 수 (rows= vs actual rows=)              │
│    - 큰 차이 → 통계 정보 부정확                             │
│    - ANALYZE 테이블 실행 필요                               │
│                                                             │
│ 2. Seq Scan vs Index Scan                                   │
│    - 작은 결과인데 Seq Scan → 인덱스 필요?                  │
│    - 큰 테이블 Seq Scan → 정상일 수 있음                    │
│                                                             │
│ 3. 조인 방법과 순서                                          │
│    - Nested Loop on large table → 문제                      │
│    - 작은 테이블이 Build 측인지 확인                         │
│                                                             │
│ 4. 정렬 (Sort)                                               │
│    - Sort Method: external merge → 메모리 부족              │
│    - work_mem 증가 검토                                      │
│                                                             │
│ 5. 버퍼 (Buffers)                                            │
│    - shared hit: 캐시 히트                                   │
│    - shared read: 디스크 읽기                                │
│    - hit/(hit+read) > 99% 목표                              │
│                                                             │
│ 6. 시간 분포                                                 │
│    - 어느 노드에서 시간이 많이 걸리는지                      │
│    - actual time의 첫 번째 값 = startup time                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7. 문제 패턴 식별

```sql
-- 문제 1: 예상 행 수 부정확
-- rows=1 (expected) vs actual rows=10000
-- → ANALYZE 실행
ANALYZE table_name;

-- 문제 2: Seq Scan on large table
-- → 인덱스 추가 검토
CREATE INDEX idx_col ON table_name(col);

-- 문제 3: Nested Loop with large inner
-- → 조인 방식 변경 또는 인덱스 추가
SET enable_nestloop = off;  -- 테스트용

-- 문제 4: Sort external merge
-- → work_mem 증가
SET work_mem = '256MB';

-- 문제 5: HashAggregate Batches > 1
-- → hash_mem_multiplier 증가 또는 work_mem 증가

-- 문제 6: Bitmap Heap Scan with many recheck
-- → 인덱스 선택도 문제
```

### 8. MySQL EXPLAIN

```sql
-- MySQL
EXPLAIN SELECT * FROM employees WHERE dept_id = 10;

-- 출력 컬럼:
-- id: SELECT 식별자
-- select_type: SIMPLE, PRIMARY, SUBQUERY 등
-- table: 테이블명
-- type: 접근 방식 (ALL, index, range, ref, eq_ref, const)
-- possible_keys: 사용 가능한 인덱스
-- key: 실제 사용된 인덱스
-- key_len: 인덱스 키 길이
-- ref: 비교 대상
-- rows: 예상 행 수
-- filtered: 필터 후 남는 비율
-- Extra: 추가 정보

-- 중요한 type 값 (좋음 → 나쁨):
-- const: PK/unique로 하나의 행
-- eq_ref: 조인에서 PK/unique 사용
-- ref: 인덱스로 여러 행
-- range: 인덱스 범위 스캔
-- index: 인덱스 전체 스캔
-- ALL: 테이블 전체 스캔

-- MySQL 8.0+
EXPLAIN ANALYZE SELECT * FROM employees WHERE dept_id = 10;
-- 실제 실행 시간 포함
```

## 실무 적용

### 느린 쿼리 분석 절차

```sql
-- 1. 실행 계획 확인
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;

-- 2. 문제점 식별
-- - 예상/실제 행 차이
-- - 비효율적 스캔
-- - 부적절한 조인

-- 3. 통계 갱신
ANALYZE table_name;

-- 4. 인덱스 추가/변경
CREATE INDEX ...;

-- 5. 쿼리 재작성
-- - 조인 순서 힌트
-- - 서브쿼리 → 조인 변환
-- - CTE 사용

-- 6. 파라미터 조정
SET work_mem = '...';
SET random_page_cost = ...;

-- 7. 재확인
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

### 유용한 분석 쿼리

```sql
-- PostgreSQL: 느린 쿼리 찾기
SELECT query, calls, total_time / calls as avg_time, rows / calls as avg_rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;

-- 테이블 통계 확인
SELECT attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'employees';

-- 인덱스 사용률
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0;  -- 미사용 인덱스
```

## 참고 자료

- PostgreSQL Documentation: Using EXPLAIN
- "High Performance MySQL" - Chapter 6
- Use The Index, Luke: https://use-the-index-luke.com/
- pgMustard: EXPLAIN ANALYZE visualization

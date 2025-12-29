# 접근 방법 (Access Methods)

## 개요

접근 방법(Access Method)은 쿼리 실행 시 테이블 데이터에 물리적으로 접근하는 방식을 결정한다. **Sequential Scan, Index Scan, Index-Only Scan, Bitmap Scan** 등 각 방법은 서로 다른 I/O 패턴과 성능 특성을 가진다. 옵티마이저는 비용 추정을 통해 최적의 접근 방법을 선택한다.

## 핵심 개념

### 1. Sequential Scan (Seq Scan)

테이블의 모든 페이지를 순차적으로 읽음:

```
┌─────────────────────────────────────────────────────────────┐
│                Sequential Scan                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 동작:                                                        │
│ [Page 0] → [Page 1] → [Page 2] → ... → [Page N]            │
│                                                             │
│ 각 페이지에서:                                               │
│ - 모든 튜플 읽기                                             │
│ - 조건(WHERE) 체크                                          │
│ - 조건 만족 시 결과에 포함                                   │
│                                                             │
│ 비용:                                                        │
│ I/O: N (전체 페이지)                                        │
│ CPU: T × (condition check) (전체 튜플)                      │
│                                                             │
│ 장점:                                                        │
│ - Sequential I/O (디스크에서 빠름)                          │
│ - 인덱스 유지 오버헤드 없음                                  │
│ - 선택도가 높을 때(많은 행 반환) 효율적                      │
│                                                             │
│ 단점:                                                        │
│ - 적은 행만 필요해도 전체 스캔                               │
│ - 큰 테이블에서 느림                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Sequential Scan 예시
EXPLAIN SELECT * FROM employees;
-- Seq Scan on employees

-- 조건이 있어도 인덱스 없으면 Seq Scan
EXPLAIN SELECT * FROM employees WHERE name = 'Kim';
-- Seq Scan on employees
--   Filter: (name = 'Kim'::text)
```

### 2. Index Scan

인덱스를 사용하여 조건에 맞는 행의 위치를 찾고 테이블에서 데이터를 가져옴:

```
┌─────────────────────────────────────────────────────────────┐
│                    Index Scan                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 동작:                                                        │
│                                                             │
│  1. 인덱스 검색        2. 테이블 접근                        │
│  ┌─────────────┐      ┌─────────────┐                      │
│  │   B+Tree    │      │   Table     │                      │
│  │    Index    │      │   (Heap)    │                      │
│  │      │      │      │             │                      │
│  │      ▼      │      │             │                      │
│  │   Leaf:     │─TID→ │   Tuple     │                      │
│  │   [Kim→TID] │      │             │                      │
│  └─────────────┘      └─────────────┘                      │
│                                                             │
│ 비용:                                                        │
│ I/O: 인덱스 높이 + 선택된 튜플 수 (Random I/O)              │
│                                                             │
│ 선택도에 따른 효율:                                          │
│ - 선택도 낮음 (적은 행): Index Scan 유리                    │
│ - 선택도 높음 (많은 행): Seq Scan 유리                      │
│ - 임계점: 보통 5-10%                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Index Scan 예시
CREATE INDEX idx_emp_id ON employees(id);

EXPLAIN SELECT * FROM employees WHERE id = 123;
-- Index Scan using idx_emp_id on employees
--   Index Cond: (id = 123)

-- 범위 검색
EXPLAIN SELECT * FROM employees WHERE id BETWEEN 100 AND 200;
-- Index Scan using idx_emp_id on employees
--   Index Cond: ((id >= 100) AND (id <= 200))
```

### 3. Index-Only Scan

인덱스만으로 결과를 반환, 테이블 접근 없음:

```
┌─────────────────────────────────────────────────────────────┐
│                Index-Only Scan                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 조건:                                                        │
│ 1. SELECT 컬럼이 모두 인덱스에 포함                         │
│ 2. PostgreSQL: Visibility Map에서 all-visible               │
│                                                             │
│ 동작:                                                        │
│  ┌─────────────┐                                            │
│  │   B+Tree    │                                            │
│  │   Index     │                                            │
│  │      │      │                                            │
│  │      ▼      │                                            │
│  │   Leaf:     │─────→ 결과 반환 (테이블 접근 없음)        │
│  │[Kim,50000]  │                                            │
│  └─────────────┘                                            │
│                                                             │
│ 비용:                                                        │
│ I/O: 인덱스 높이 + 리프 페이지 (Sequential 가능)            │
│ 테이블 Random I/O 없음 → 대폭 성능 향상                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Covering Index 생성
CREATE INDEX idx_emp_name_salary ON employees(name, salary);

-- Index-Only Scan 가능
EXPLAIN SELECT name, salary FROM employees WHERE name = 'Kim';
-- Index Only Scan using idx_emp_name_salary on employees
--   Index Cond: (name = 'Kim'::text)
--   Heap Fetches: 0  ← 테이블 접근 없음

-- Include 절 활용 (PostgreSQL 11+)
CREATE INDEX idx_name_include ON employees(name) INCLUDE (email, dept_id);

-- Index-Only Scan
SELECT name, email, dept_id FROM employees WHERE name = 'Kim';
```

### 4. Bitmap Scan

여러 인덱스를 조합하거나 많은 행을 효율적으로 가져올 때:

```
┌─────────────────────────────────────────────────────────────┐
│                    Bitmap Scan                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 동작:                                                        │
│                                                             │
│ 1. Bitmap Index Scan: 인덱스에서 TID 수집 → 비트맵 생성    │
│                                                             │
│    인덱스 A (dept_id=10)     인덱스 B (status='active')     │
│    ┌───────────────────┐    ┌───────────────────┐          │
│    │ Page 0: 1 0 1 1 0 │    │ Page 0: 1 1 1 0 0 │          │
│    │ Page 1: 0 1 1 0 0 │    │ Page 1: 1 1 0 1 0 │          │
│    └───────────────────┘    └───────────────────┘          │
│                                                             │
│ 2. BitmapAnd / BitmapOr: 비트맵 결합                        │
│                                                             │
│    AND 결과:                                                 │
│    ┌───────────────────┐                                    │
│    │ Page 0: 1 0 1 0 0 │                                    │
│    │ Page 1: 0 1 0 0 0 │                                    │
│    └───────────────────┘                                    │
│                                                             │
│ 3. Bitmap Heap Scan: 비트맵 순서대로 테이블 페이지 접근     │
│    - 같은 페이지의 튜플들을 한 번에 처리                     │
│    - Random I/O를 Sequential에 가깝게 변환                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Bitmap Scan 예시
EXPLAIN SELECT * FROM employees
WHERE dept_id = 10 AND status = 'active';

-- Bitmap Heap Scan on employees
--   Recheck Cond: ((dept_id = 10) AND (status = 'active'))
--   -> BitmapAnd
--      -> Bitmap Index Scan on idx_dept_id
--         Index Cond: (dept_id = 10)
--      -> Bitmap Index Scan on idx_status
--         Index Cond: (status = 'active')
```

**Bitmap Scan 선택 이유:**
- Index Scan보다 많은 행을 가져올 때
- 여러 인덱스를 조합할 때 (AND/OR)
- Seq Scan보다 적은 행이지만 Random I/O 줄이고 싶을 때

### 5. TID Scan

튜플의 물리적 위치(TID)를 직접 지정:

```sql
-- TID = (page_number, tuple_offset)
SELECT * FROM employees WHERE ctid = '(0,1)';
-- TID Scan on employees
--   TID Cond: (ctid = '(0,1)'::tid)

-- 주로 내부적으로 사용 (예: UPDATE/DELETE 후 재검증)
-- 직접 사용은 드뭄 (MVCC로 TID 변경 가능)
```

### 6. 접근 방법 선택 기준

```
┌─────────────────────────────────────────────────────────────┐
│              접근 방법 선택 가이드                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 선택도(Selectivity) 기준:                                   │
│                                                             │
│ 0%        ~5%           ~20%                    100%        │
│ │          │             │                        │         │
│ │──────────│─────────────│────────────────────────│         │
│ │ Index    │ Bitmap      │    Sequential          │         │
│ │ Scan     │ Scan        │    Scan                │         │
│                                                             │
│ 결정 요소:                                                   │
│ 1. 선택도: 반환 행 / 전체 행                                │
│ 2. 클러스터링: 데이터가 인덱스 순서로 정렬되어 있는가       │
│ 3. 컬럼 커버리지: Index-Only Scan 가능한가                  │
│ 4. 인덱스 유형: B+Tree, Hash, GIN 등                        │
│ 5. 테이블 크기: 작은 테이블은 Seq Scan이 더 빠를 수 있음   │
│                                                             │
│ 비용 공식 (개념적):                                          │
│ Seq Scan: seq_page_cost × pages + cpu_tuple_cost × tuples  │
│ Index Scan: random_page_cost × index_pages +               │
│             random_page_cost × heap_pages +                │
│             cpu_index_tuple_cost × index_tuples            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7. 실제 DBMS별 접근 방법

```sql
-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM employees WHERE dept_id = 10;

-- 가능한 스캔 방법:
-- Seq Scan, Index Scan, Index Only Scan, Bitmap Heap Scan, TID Scan

-- MySQL InnoDB
EXPLAIN SELECT * FROM employees WHERE dept_id = 10;

-- type 컬럼:
-- ALL (full table scan), index, range, ref, eq_ref, const, system
```

### 8. 힌트를 통한 접근 방법 강제

```sql
-- PostgreSQL (비공식, enable_* 설정)
SET enable_seqscan = off;
SET enable_indexscan = off;
SET enable_bitmapscan = off;

EXPLAIN SELECT * FROM employees WHERE dept_id = 10;
-- 옵티마이저가 남은 옵션 중 선택

-- MySQL
SELECT /*+ INDEX(employees idx_dept_id) */ *
FROM employees WHERE dept_id = 10;

SELECT /*+ NO_INDEX(employees idx_dept_id) */ *
FROM employees WHERE dept_id = 10;

-- Oracle
SELECT /*+ FULL(employees) */ * FROM employees WHERE dept_id = 10;
SELECT /*+ INDEX(employees idx_dept_id) */ * FROM employees WHERE dept_id = 10;
```

## 실무 적용

### 접근 방법 분석

```sql
-- PostgreSQL: EXPLAIN ANALYZE로 실제 접근 방법 확인
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 123 AND status = 'active';

-- 확인 사항:
-- 1. 사용된 접근 방법
-- 2. 예상 vs 실제 행 수
-- 3. 버퍼 hit/read 비율
-- 4. 인덱스 사용 여부

-- Shared hit: 버퍼 캐시 히트
-- Shared read: 디스크 읽기
```

### Index-Only Scan 최적화

```sql
-- Visibility Map 갱신
VACUUM orders;

-- 확인
SELECT * FROM pg_visibility_map_summary('orders');

-- Index-Only Scan 후 Heap Fetches 확인
EXPLAIN (ANALYZE)
SELECT order_id, customer_id FROM orders WHERE customer_id = 123;
-- Heap Fetches: 0 → 완전한 Index-Only Scan
```

## 참고 자료

- PostgreSQL Documentation: Using EXPLAIN
- "High Performance MySQL" - Chapter 6
- CMU 15-445: Query Execution
- PostgreSQL Wiki: Index Scanning

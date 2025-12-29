# 조인 알고리즘 심화 (Join Algorithms)

## 개요

조인은 두 테이블의 관련된 행을 결합하는 연산으로, 쿼리 성능에 가장 큰 영향을 미치는 요소 중 하나다. **Nested Loop Join, Sort-Merge Join, Hash Join** 세 가지 기본 알고리즘이 있으며, 각각 다른 조건에서 최적의 성능을 보인다. 이 문서에서는 각 알고리즘의 내부 동작과 I/O 비용 분석을 다룬다.

## 핵심 개념

### 1. 조인 알고리즘 개요

```
┌─────────────────────────────────────────────────────────────┐
│              조인 알고리즘 비교                               │
├───────────────┬─────────────┬────────────────┬──────────────┤
│               │ Nested Loop │  Sort-Merge   │ Hash Join    │
├───────────────┼─────────────┼────────────────┼──────────────┤
│ 조인 조건     │ 모든 조건   │ 등호(=) + 정렬│ 등호(=) only │
├───────────────┼─────────────┼────────────────┼──────────────┤
│ 메모리 요구   │ 낮음        │ 정렬 공간     │ 해시 테이블  │
├───────────────┼─────────────┼────────────────┼──────────────┤
│ 최적 시나리오 │ 작은 inner, │ 정렬된 데이터 │ 큰 테이블,   │
│               │ 인덱스 있음 │ 또는 정렬 필요│ 등호 조인    │
├───────────────┼─────────────┼────────────────┼──────────────┤
│ 비용 복잡도   │O(M×N) ~ O(M+N)│ O(M log M +  │ O(M + N)     │
│               │              │   N log N + M+N)             │
└───────────────┴─────────────┴────────────────┴──────────────┘
```

### 2. Nested Loop Join

가장 단순한 조인 알고리즘:

```
┌─────────────────────────────────────────────────────────────┐
│              Simple Nested Loop Join                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ for each tuple r in R (outer):                              │
│     for each tuple s in S (inner):                          │
│         if r.key == s.key:                                  │
│             emit (r, s)                                     │
│                                                             │
│ 비용:                                                        │
│ - I/O: M + M × N (M = outer 페이지, N = inner 페이지)       │
│ - inner를 outer 행 수만큼 반복 스캔                         │
│                                                             │
│ 예: M=1000, N=500                                           │
│ I/O = 1000 + 1000 × 500 = 501,000 페이지                   │
│                                                             │
│ 최적화: 작은 테이블을 inner로                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Block Nested Loop Join (개선):**

```
┌─────────────────────────────────────────────────────────────┐
│              Block Nested Loop Join                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 버퍼 B개를 outer에 할당                                      │
│                                                             │
│ for each block of B pages in R:                             │
│     for each page in S:                                     │
│         for each tuple r in block:                          │
│             for each tuple s in page:                       │
│                 if r.key == s.key:                          │
│                     emit (r, s)                             │
│                                                             │
│ 비용:                                                        │
│ I/O: M + ⌈M/B⌉ × N                                          │
│                                                             │
│ 예: M=1000, N=500, B=100                                    │
│ I/O = 1000 + ⌈1000/100⌉ × 500 = 1000 + 5000 = 6,000        │
│                                                             │
│ vs Simple: 501,000 → 6,000 (83배 감소)                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Index Nested Loop Join:**

```
┌─────────────────────────────────────────────────────────────┐
│              Index Nested Loop Join                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 조건: inner 테이블의 조인 컬럼에 인덱스 존재                │
│                                                             │
│ for each tuple r in R:                                      │
│     matching_tuples = index_lookup(S.index, r.key)          │
│     for each s in matching_tuples:                          │
│         emit (r, s)                                         │
│                                                             │
│ 비용:                                                        │
│ I/O: M + |R| × (인덱스 탐색 비용)                           │
│    ≈ M + |R| × (log N + 평균 매칭 수)                       │
│                                                             │
│ 예: |R|=10,000, 인덱스 높이=3, 평균 매칭=1                  │
│ I/O ≈ M + 10,000 × 4 = M + 40,000                          │
│                                                             │
│ 작은 outer + 인덱스된 inner에서 매우 효율적                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. Sort-Merge Join

정렬된 데이터를 병합하면서 조인:

```
┌─────────────────────────────────────────────────────────────┐
│              Sort-Merge Join                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Phase 1: Sort (필요시)                                      │
│   sort R by join key                                        │
│   sort S by join key                                        │
│                                                             │
│ Phase 2: Merge                                              │
│   r = first(R), s = first(S)                               │
│   while r != EOF and s != EOF:                              │
│       if r.key < s.key: r = next(R)                        │
│       elif r.key > s.key: s = next(S)                      │
│       else:  # r.key == s.key                              │
│           # 같은 키를 가진 모든 쌍 출력                      │
│           emit all (r, s) pairs with same key              │
│           advance both                                      │
│                                                             │
│ 비용:                                                        │
│ Sort: 2M × (⌈log_{B-1}(M/B)⌉ + 1) + 2N × ...              │
│ Merge: M + N                                                │
│                                                             │
│ 이미 정렬되어 있으면: O(M + N)                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```
예시: Sort-Merge Join

R (정렬 후):          S (정렬 후):
[1, a]                [1, x]
[2, b]                [2, y]
[2, c]                [2, z]
[4, d]                [3, w]

Merge:
r=[1,a], s=[1,x] → emit (1,a,x), advance both
r=[2,b], s=[2,y] → emit (2,b,y), (2,b,z), (2,c,y), (2,c,z)
r=[4,d], s=[3,w] → 3 < 4, advance S
s=EOF → done

결과:
(1, a, x)
(2, b, y)
(2, b, z)
(2, c, y)
(2, c, z)
```

**Sort-Merge Join 적합 조건:**
- 입력이 이미 정렬됨 (인덱스 스캔 결과)
- 결과도 정렬 필요 (ORDER BY)
- 비등호 조인 (<, <=, >=, >)

### 4. Hash Join

해시 테이블을 사용한 등호 조인:

```
┌─────────────────────────────────────────────────────────────┐
│              Simple Hash Join (In-Memory)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Phase 1: Build (작은 테이블 R)                              │
│   hash_table = {}                                           │
│   for each tuple r in R:                                    │
│       bucket = hash(r.key)                                  │
│       hash_table[bucket].append(r)                          │
│                                                             │
│ Phase 2: Probe (큰 테이블 S)                                │
│   for each tuple s in S:                                    │
│       bucket = hash(s.key)                                  │
│       for each r in hash_table[bucket]:                     │
│           if r.key == s.key:                                │
│               emit (r, s)                                   │
│                                                             │
│ 비용:                                                        │
│ I/O: M + N (각 테이블 한 번씩 스캔)                         │
│ 메모리: Build 테이블이 메모리에 들어가야 함                  │
│                                                             │
│ 조건: 해시 테이블이 메모리에 fit                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Grace Hash Join (디스크 기반):**

```
┌─────────────────────────────────────────────────────────────┐
│              Grace Hash Join                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 메모리에 안 들어가는 큰 테이블 처리                         │
│                                                             │
│ Phase 1: Partition (해시 함수 h1으로 분할)                  │
│                                                             │
│   R: [r1 r2 r3 r4 ...]  →  R0: [r1 r4]                     │
│                            R1: [r2]                         │
│   S: [s1 s2 s3 s4 ...]  →  S0: [s1 s3]                     │
│                            S1: [s2 s4]                      │
│                                                             │
│   같은 파티션끼리만 조인 가능 (키가 다른 파티션이면 매칭 불가)│
│                                                             │
│ Phase 2: Build & Probe (각 파티션에서 h2로 해시 조인)       │
│                                                             │
│   for i = 0 to num_partitions:                              │
│       build hash table from Ri                              │
│       probe with Si                                         │
│                                                             │
│ 비용:                                                        │
│ I/O: 3(M + N) - 읽기 + 파티션 쓰기 + 조인 읽기             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Hybrid Hash Join:**

```
┌─────────────────────────────────────────────────────────────┐
│              Hybrid Hash Join                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Grace Hash Join + 첫 파티션 메모리 유지                     │
│                                                             │
│ - 파티션 0은 디스크에 쓰지 않고 메모리에 유지               │
│ - Probe 시 파티션 0은 바로 처리                             │
│ - I/O 절약                                                   │
│                                                             │
│ 비용:                                                        │
│ I/O: M + N + 2(M-Mf + N-Nf)                                 │
│      Mf, Nf = 메모리에 유지된 첫 파티션 크기                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. 비용 비교 분석

```
┌─────────────────────────────────────────────────────────────┐
│              I/O 비용 비교 예시                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 가정: R = 1000 pages, S = 500 pages, Buffer = 100 pages    │
│                                                             │
│ Simple Nested Loop:                                         │
│   1000 + 1000 × 500 = 501,000                              │
│                                                             │
│ Block Nested Loop:                                          │
│   1000 + ⌈1000/100⌉ × 500 = 6,000                          │
│                                                             │
│ Sort-Merge (이미 정렬된 경우):                              │
│   1000 + 500 = 1,500                                        │
│                                                             │
│ Sort-Merge (정렬 필요):                                     │
│   Sort R: ~3000, Sort S: ~1500, Merge: 1500                │
│   Total: ~6,000                                             │
│                                                             │
│ Hash Join (R이 메모리에 fit하지 않음):                      │
│   Grace: 3 × (1000 + 500) = 4,500                          │
│                                                             │
│ 최적: 상황에 따라 다름                                       │
│ - 정렬 불필요 + 메모리 충분: Hash Join                      │
│ - 이미 정렬됨: Sort-Merge                                   │
│ - 작은 테이블 + 인덱스: Index Nested Loop                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6. 실제 DBMS의 조인 알고리즘

```sql
-- PostgreSQL EXPLAIN으로 조인 알고리즘 확인
EXPLAIN (ANALYZE)
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.region = 'Seoul';

-- 가능한 조인 방법:
-- Nested Loop
-- Hash Join
-- Merge Join

-- 힌트로 강제 (비공식)
SET enable_hashjoin = off;
SET enable_mergejoin = off;
SET enable_nestloop = off;
```

```sql
-- MySQL 조인 알고리즘
EXPLAIN SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- type: eq_ref, ref, ALL 등으로 확인
-- Extra: Using join buffer (Block Nested Loop)
--        Using where; Using join buffer (hash join) (8.0.18+)
```

### 7. 조인 순서 최적화

```
N개 테이블 조인 시 가능한 순서: N!

예: 4개 테이블
(((A ⋈ B) ⋈ C) ⋈ D)
(((A ⋈ B) ⋈ D) ⋈ C)
((A ⋈ (B ⋈ C)) ⋈ D)
... (총 4! × Catalan(4) = 수백 가지)

옵티마이저 전략:
1. Dynamic Programming (System R)
2. Genetic Algorithm (PostgreSQL GEQO)
3. Greedy Heuristics
```

## 실무 적용

### 조인 성능 분석

```sql
-- PostgreSQL: 조인 비용 분석
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT *
FROM large_orders o
JOIN large_customers c ON o.customer_id = c.id;

-- 확인 사항:
-- 1. 조인 알고리즘 (Hash, Nested Loop, Merge)
-- 2. Build vs Probe 테이블
-- 3. 버퍼 사용량
-- 4. 실제 vs 예상 행 수
```

### 조인 최적화 팁

```sql
-- 1. 작은 테이블을 Build 측으로
-- Hash Join에서 작은 테이블로 해시 테이블 생성

-- 2. 조인 전 필터링
SELECT *
FROM (SELECT * FROM orders WHERE status = 'active') o
JOIN customers c ON o.customer_id = c.id;

-- 3. 적절한 인덱스
CREATE INDEX idx_customer_id ON orders(customer_id);
-- Index Nested Loop Join 활성화

-- 4. work_mem 조정 (PostgreSQL)
SET work_mem = '256MB';  -- Hash Join에 더 많은 메모리
```

## 참고 자료

- "Database System Concepts" (Silberschatz) - Chapter 15
- CMU 15-445: Join Algorithms
- PostgreSQL Documentation: Query Planning
- "Architecture of a Database System" (Hellerstein) - Section 4

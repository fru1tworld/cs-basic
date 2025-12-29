# 실행 모델 (Execution Models)

## 개요

쿼리 실행 엔진이 연산자 트리를 어떻게 실행하는지를 결정하는 것이 실행 모델이다. **Volcano (Iterator) Model**, **Materialization Model**, **Vectorized Execution** 등 각 모델은 서로 다른 트레이드오프를 가진다. 현대 분석 시스템에서는 벡터화 실행과 JIT 컴파일이 주류가 되고 있다.

## 핵심 개념

### 1. Volcano (Iterator) Model

가장 널리 사용되는 전통적 실행 모델:

```
┌─────────────────────────────────────────────────────────────┐
│              Volcano (Iterator) Model                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 원칙: Pull-based, Tuple-at-a-time                          │
│                                                             │
│ 각 연산자가 3개의 함수 구현:                                │
│ - open(): 초기화                                            │
│ - next(): 다음 튜플 반환 (또는 EOF)                         │
│ - close(): 정리                                             │
│                                                             │
│ 예: SELECT name FROM emp WHERE dept_id = 10                 │
│                                                             │
│         ┌─────────────┐                                     │
│         │  Project    │ ← next() 호출받음                   │
│         │  (name)     │                                     │
│         └─────────────┘                                     │
│               │ next()                                      │
│               ▼                                             │
│         ┌─────────────┐                                     │
│         │   Filter    │                                     │
│         │(dept_id=10) │                                     │
│         └─────────────┘                                     │
│               │ next() (조건 만족할 때까지 반복)             │
│               ▼                                             │
│         ┌─────────────┐                                     │
│         │  SeqScan    │                                     │
│         │   (emp)     │                                     │
│         └─────────────┘                                     │
│                                                             │
│ 실행 흐름:                                                   │
│ 1. Root(Project).next() 호출                                │
│ 2. Project → Filter.next() 호출                             │
│ 3. Filter → SeqScan.next() 반복 (조건 만족까지)            │
│ 4. SeqScan이 튜플 반환                                      │
│ 5. Filter가 조건 검사 후 통과시키거나 다시 next()           │
│ 6. Project가 name만 추출하여 반환                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**장점:**
- 단순한 구현
- 파이프라이닝 (중간 결과 저장 불필요)
- 메모리 효율적

**단점:**
- 함수 호출 오버헤드 (튜플당 여러 번)
- CPU 캐시 비효율
- 가상 함수 호출 오버헤드

### 2. Materialization Model

```
┌─────────────────────────────────────────────────────────────┐
│              Materialization Model                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 원칙: 각 연산자가 전체 결과를 물리화                        │
│                                                             │
│ 각 연산자가 1개의 함수 구현:                                │
│ - output(): 전체 결과 집합 반환                             │
│                                                             │
│ 실행 흐름:                                                   │
│                                                             │
│         ┌─────────────┐                                     │
│         │  SeqScan    │ output() → 모든 튜플               │
│         └─────────────┘                                     │
│               │ 전체 결과                                   │
│               ▼                                             │
│         ┌─────────────┐                                     │
│         │   Filter    │ output() → 필터링된 전체 결과      │
│         └─────────────┘                                     │
│               │ 전체 결과                                   │
│               ▼                                             │
│         ┌─────────────┐                                     │
│         │  Project    │ output() → 최종 전체 결과          │
│         └─────────────┘                                     │
│                                                             │
│ 장점:                                                        │
│ - 구현 단순                                                  │
│ - 함수 호출 횟수 감소                                        │
│                                                             │
│ 단점:                                                        │
│ - 큰 중간 결과 저장 필요                                    │
│ - 메모리 사용량 높음                                         │
│ - 첫 결과까지 시간 오래 걸림                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. Vectorized Execution

```
┌─────────────────────────────────────────────────────────────┐
│              Vectorized Execution                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 원칙: Batch-at-a-time (예: 1000개 튜플 단위)               │
│                                                             │
│ Volcano와 Materialization의 절충:                           │
│ - 튜플 단위 → 함수 호출 오버헤드                           │
│ - 전체 결과 → 메모리 낭비                                   │
│ - 배치 단위 → 균형점                                        │
│                                                             │
│ 구조: 컬럼 단위 배치 (Column Batch)                         │
│                                                             │
│ ┌────────────────────────────────────────────┐             │
│ │ Column Batch (1024 rows)                    │             │
│ │ ┌──────────────────────────────────────┐   │             │
│ │ │ id:    [1, 2, 3, ..., 1024]          │   │             │
│ │ │ name:  ["Kim", "Lee", "Park", ...]   │   │             │
│ │ │ salary:[50000, 60000, 55000, ...]    │   │             │
│ │ │ valid: [1, 1, 0, 1, ...]  (selection)│   │             │
│ │ └──────────────────────────────────────┘   │             │
│ └────────────────────────────────────────────┘             │
│                                                             │
│ 장점:                                                        │
│ - 함수 호출 오버헤드 감소 (1024배)                          │
│ - CPU 캐시 친화적                                            │
│ - SIMD 활용 가능                                             │
│ - 컬럼 압축과 잘 어울림                                      │
│                                                             │
│ 사용: MonetDB, ClickHouse, DuckDB, Snowflake               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**벡터화 연산 예시:**

```cpp
// 튜플 단위 (Volcano)
for each tuple t:
    if t.salary > 50000:
        output(t)

// 벡터 단위
for i = 0 to batch_size:
    selection[i] = salary[i] > 50000  // SIMD 가능

// SIMD 활용
__m256i threshold = _mm256_set1_epi32(50000);
__m256i salaries = _mm256_loadu_si256(...);
__m256i result = _mm256_cmpgt_epi32(salaries, threshold);
```

### 4. Push vs Pull

```
┌─────────────────────────────────────────────────────────────┐
│              Push vs Pull Execution                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Pull-based (Volcano):                                       │
│                                                             │
│      Consumer                                               │
│          │ next()                                           │
│          ▼                                                  │
│      Producer                                               │
│                                                             │
│ - 상위 연산자가 하위에게 "다음 데이터 줘" 요청              │
│ - 데이터가 위로 끌어올려짐                                   │
│                                                             │
│ Push-based:                                                 │
│                                                             │
│      Producer                                               │
│          │ produce(tuple)                                   │
│          ▼                                                  │
│      Consumer                                               │
│                                                             │
│ - 하위 연산자가 상위에게 데이터를 밀어넣음                  │
│ - 이벤트 기반, 비동기 처리에 적합                           │
│ - 병렬화, 파이프라이닝에 유리                               │
│                                                             │
│ Morsel-Driven Parallelism (HyPer):                         │
│ - Push-based + 작업 단위(Morsel) 분배                      │
│ - 동적 로드 밸런싱                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. Pipeline Breakers

특정 연산자는 모든 입력을 받아야 결과를 출력할 수 있음:

```
┌─────────────────────────────────────────────────────────────┐
│              Pipeline Breakers                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Non-Blocking (파이프라이닝 가능):                           │
│ - Filter: 조건 체크 후 바로 출력                            │
│ - Project: 컬럼 선택 후 바로 출력                           │
│ - Nested Loop Join (inner가 작을 때)                        │
│                                                             │
│ Blocking (파이프라인 끊김):                                 │
│ - Sort: 전체 정렬 후 출력                                   │
│ - Hash Join (Build): 해시 테이블 완성까지 대기              │
│ - Aggregation: 전체 그룹화 후 출력                          │
│ - DISTINCT: 전체 확인 후 출력                               │
│                                                             │
│ 예시:                                                        │
│                                                             │
│     Project ──────┐                                         │
│        │          │ Pipeline 1                              │
│      Sort  ◀──────┘ (Blocking)                             │
│        │                                                    │
│      Filter ──────┐                                         │
│        │          │ Pipeline 2                              │
│     SeqScan ◀─────┘                                         │
│                                                             │
│ Sort가 Pipeline Breaker: 위아래가 다른 파이프라인           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6. JIT Compilation (PostgreSQL)

```
┌─────────────────────────────────────────────────────────────┐
│              JIT Compilation                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 일반 실행:                                                   │
│ - 범용 인터프리터로 연산자 트리 순회                        │
│ - 각 표현식마다 함수 호출                                    │
│                                                             │
│ JIT 실행:                                                    │
│ - 쿼리별 특화 코드 생성                                      │
│ - LLVM을 사용하여 기계어로 컴파일                           │
│                                                             │
│ JIT 대상 (PostgreSQL):                                       │
│ 1. Expression Evaluation                                    │
│    WHERE a + b > c * 2 → 특화 코드                         │
│                                                             │
│ 2. Tuple Deforming                                          │
│    튜플 → 컬럼 값 추출 코드                                 │
│                                                             │
│ 3. Aggregation                                              │
│    SUM, COUNT 등 집계 루프                                  │
│                                                             │
│ 효과:                                                        │
│ - 함수 호출 오버헤드 제거                                    │
│ - 인라인화, 상수 폴딩                                        │
│ - OLAP 쿼리에서 2-5배 성능 향상                             │
│                                                             │
│ 비용:                                                        │
│ - 컴파일 시간 (수십~수백 ms)                                │
│ - 작은 쿼리에서는 오히려 느림                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL JIT 설정
SET jit = on;
SET jit_above_cost = 100000;  -- 이 비용 이상에서 JIT
SET jit_inline_above_cost = 500000;
SET jit_optimize_above_cost = 500000;

-- JIT 사용 확인
EXPLAIN (ANALYZE) SELECT SUM(amount) FROM large_table;
-- JIT:
--   Functions: 4
--   Generation Time: 1.234 ms
--   Inlining Time: 5.678 ms
--   Optimization Time: 10.123 ms
--   Emission Time: 15.456 ms
```

### 7. 실행 모델 비교

```
┌─────────────────────────────────────────────────────────────┐
│              실행 모델 비교                                   │
├───────────────┬─────────────┬──────────────┬───────────────┤
│               │  Volcano    │Materialization│  Vectorized  │
├───────────────┼─────────────┼──────────────┼───────────────┤
│ 단위          │ Tuple       │ Entire Set   │ Vector/Batch │
├───────────────┼─────────────┼──────────────┼───────────────┤
│ 메모리        │ 낮음        │ 높음         │ 중간         │
├───────────────┼─────────────┼──────────────┼───────────────┤
│ 함수 호출     │ 많음        │ 적음         │ 적음         │
├───────────────┼─────────────┼──────────────┼───────────────┤
│ 캐시 효율     │ 낮음        │ 중간         │ 높음         │
├───────────────┼─────────────┼──────────────┼───────────────┤
│ SIMD          │ 어려움      │ 가능         │ 최적         │
├───────────────┼─────────────┼──────────────┼───────────────┤
│ 적합 워크로드 │ OLTP        │ 작은 쿼리    │ OLAP         │
├───────────────┼─────────────┼──────────────┼───────────────┤
│ 사용 시스템   │ PostgreSQL, │ SQLite,      │ MonetDB,     │
│               │ MySQL       │ 일부 임베디드│ ClickHouse,  │
│               │             │              │ DuckDB       │
└───────────────┴─────────────┴──────────────┴───────────────┘
```

## 실무 적용

### 실행 모델 확인

```sql
-- PostgreSQL: 파이프라인 확인
EXPLAIN (ANALYZE, BUFFERS)
SELECT dept_id, SUM(salary)
FROM employees
WHERE status = 'active'
GROUP BY dept_id
ORDER BY SUM(salary) DESC;

-- Sort가 Pipeline Breaker
-- HashAggregate도 Blocking
```

### JIT 효과 측정

```sql
-- JIT 없이 실행
SET jit = off;
EXPLAIN (ANALYZE) SELECT ... ;

-- JIT 켜고 실행
SET jit = on;
EXPLAIN (ANALYZE) SELECT ... ;

-- 비교: Execution Time 차이
```

## 참고 자료

- Graefe, "Volcano - An Extensible and Parallel Query Evaluation System" (1994)
- Boncz et al., "MonetDB/X100: Hyper-Pipelining Query Execution" (2005)
- CMU 15-445: Query Execution
- PostgreSQL Documentation: JIT Compilation

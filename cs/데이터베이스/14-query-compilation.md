# 쿼리 컴파일 (Query Compilation)

## 개요

SQL 쿼리가 실행되기까지는 여러 단계의 컴파일 과정을 거친다. **Parsing → Binding → Optimization → Code Generation**의 파이프라인을 통해 선언적 SQL이 효율적인 실행 계획으로 변환된다. 이 과정을 이해하면 쿼리 오류의 원인을 파악하고, 옵티마이저의 동작을 예측할 수 있다.

## 핵심 개념

### 1. 쿼리 처리 파이프라인

```
┌─────────────────────────────────────────────────────────────┐
│                Query Processing Pipeline                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   SQL Query                                                 │
│       │                                                     │
│       ▼                                                     │
│   ┌───────────────┐                                        │
│   │    Parser     │  구문 분석 → AST                        │
│   └───────────────┘                                        │
│       │                                                     │
│       ▼                                                     │
│   ┌───────────────┐                                        │
│   │    Binder     │  의미 분석, 이름 해석                   │
│   │   (Analyzer)  │  → Logical Plan                        │
│   └───────────────┘                                        │
│       │                                                     │
│       ▼                                                     │
│   ┌───────────────┐                                        │
│   │   Rewriter    │  쿼리 재작성, 뷰 확장                   │
│   └───────────────┘                                        │
│       │                                                     │
│       ▼                                                     │
│   ┌───────────────┐                                        │
│   │  Optimizer    │  비용 기반 최적화                       │
│   │               │  → Physical Plan                       │
│   └───────────────┘                                        │
│       │                                                     │
│       ▼                                                     │
│   ┌───────────────┐                                        │
│   │   Executor    │  실행                                   │
│   └───────────────┘                                        │
│       │                                                     │
│       ▼                                                     │
│   Results                                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. Parsing (구문 분석)

SQL 문자열을 AST(Abstract Syntax Tree)로 변환:

```
┌─────────────────────────────────────────────────────────────┐
│                    Parsing                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 입력: SELECT name, salary FROM employees WHERE dept_id = 10 │
│                                                             │
│ 1. Lexical Analysis (어휘 분석)                             │
│    토큰화: [SELECT] [name] [,] [salary] [FROM] ...          │
│                                                             │
│ 2. Syntax Analysis (구문 분석)                              │
│    문법 검사, AST 생성                                       │
│                                                             │
│ 결과 AST:                                                    │
│                                                             │
│           SelectStatement                                   │
│          /       |        \                                 │
│    SelectList  FromClause  WhereClause                      │
│      /    \        |            |                           │
│  Column Column  TableRef   Comparison                       │
│  "name" "salary" "employees"  /    \                        │
│                          Column    Literal                  │
│                         "dept_id"    10                     │
│                                                             │
│ 오류 감지:                                                   │
│ - 문법 오류: "SELEC * FROM t" → syntax error               │
│ - 괄호 불일치, 키워드 오타 등                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. Binding / Semantic Analysis (의미 분석)

AST의 이름을 실제 데이터베이스 객체에 연결:

```
┌─────────────────────────────────────────────────────────────┐
│                    Binding                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 수행 작업:                                                   │
│                                                             │
│ 1. 이름 해석 (Name Resolution)                              │
│    - "employees" → schema.employees (OID: 12345)            │
│    - "name" → employees.name (column index: 1, type: text)  │
│    - "salary" → employees.salary (column index: 3, type: int)│
│                                                             │
│ 2. 타입 검사 (Type Checking)                                │
│    - dept_id = 10 → 정수 비교 OK                            │
│    - dept_id = 'abc' → 타입 오류 또는 암시적 변환           │
│                                                             │
│ 3. 권한 검사 (Permission Check)                             │
│    - 현재 사용자가 employees 테이블 SELECT 권한 있는지      │
│                                                             │
│ 4. 스코프 해석                                               │
│    - 서브쿼리의 컬럼 참조 범위 결정                          │
│    - 상관 서브쿼리의 외부 참조 식별                          │
│                                                             │
│ 결과: Bound AST (모든 참조가 해석됨)                        │
│                                                             │
│ 오류 감지:                                                   │
│ - "테이블이 존재하지 않습니다"                               │
│ - "컬럼 'nam'을(를) 찾을 수 없습니다"                        │
│ - "권한이 없습니다"                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4. Query Rewriter (쿼리 재작성)

```
┌─────────────────────────────────────────────────────────────┐
│                Query Rewriting                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. 뷰 확장 (View Expansion)                                 │
│                                                             │
│ CREATE VIEW active_emp AS                                   │
│   SELECT * FROM employees WHERE status = 'active';          │
│                                                             │
│ SELECT name FROM active_emp WHERE dept_id = 10;            │
│ →                                                           │
│ SELECT name FROM employees                                  │
│ WHERE status = 'active' AND dept_id = 10;                  │
│                                                             │
│ 2. 서브쿼리 평탄화 (Subquery Flattening)                    │
│                                                             │
│ SELECT * FROM a WHERE id IN (SELECT a_id FROM b);          │
│ →                                                           │
│ SELECT DISTINCT a.* FROM a JOIN b ON a.id = b.a_id;        │
│                                                             │
│ 3. 상수 폴딩 (Constant Folding)                             │
│                                                             │
│ WHERE created_at > '2024-01-01' + INTERVAL '30 days'       │
│ →                                                           │
│ WHERE created_at > '2024-01-31'                            │
│                                                             │
│ 4. 불필요한 조건 제거                                        │
│                                                             │
│ WHERE 1 = 1 AND name = 'Kim'                               │
│ →                                                           │
│ WHERE name = 'Kim'                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. Logical Plan vs Physical Plan

```
┌─────────────────────────────────────────────────────────────┐
│           Logical Plan vs Physical Plan                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Logical Plan (논리적 계획):                                 │
│ - WHAT: 무엇을 계산할지                                     │
│ - 관계 대수 연산자 트리                                      │
│ - 물리적 구현 방법 미지정                                    │
│                                                             │
│      Project(name, salary)                                  │
│            │                                                │
│      Select(dept_id = 10)                                   │
│            │                                                │
│       Scan(employees)                                       │
│                                                             │
│                                                             │
│ Physical Plan (물리적 계획):                                │
│ - HOW: 어떻게 계산할지                                      │
│ - 구체적인 알고리즘 선택                                     │
│ - 인덱스 사용 여부, 조인 방법 등                            │
│                                                             │
│      Project(name, salary)                                  │
│            │                                                │
│      IndexScan(employees, idx_dept_id, dept_id = 10)       │
│                                                             │
│                                                             │
│ 옵티마이저가 Logical → Physical 변환 시                     │
│ 여러 Physical Plan 중 최적 선택                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6. 옵티마이저 개요

```
┌─────────────────────────────────────────────────────────────┐
│                    Optimizer                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Rule-Based Optimization (규칙 기반):                        │
│ - 항상 적용되는 변환 규칙                                    │
│ - Predicate Pushdown                                        │
│ - Projection Pushdown                                       │
│ - 불필요한 연산 제거                                         │
│                                                             │
│ Cost-Based Optimization (비용 기반):                        │
│ - 여러 대안 중 비용이 가장 낮은 선택                        │
│ - 통계 정보 필요 (행 수, 카디널리티, 분포)                  │
│                                                             │
│ 결정 사항:                                                   │
│ 1. 접근 방법: Seq Scan vs Index Scan                        │
│ 2. 조인 방법: Nested Loop vs Hash vs Merge                  │
│ 3. 조인 순서: (A ⋈ B) ⋈ C vs A ⋈ (B ⋈ C)                  │
│ 4. 정렬 방법: 인덱스 활용 vs Sort 연산                       │
│                                                             │
│ 탐색 공간:                                                   │
│ - N개 테이블 조인: N! 가지 순서                              │
│ - 각각에 여러 알고리즘 적용 가능                             │
│ - 탐색 공간 폭발 → 휴리스틱 필요                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7. Prepared Statement와 Plan Caching

```
┌─────────────────────────────────────────────────────────────┐
│              Prepared Statement                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 일반 쿼리:                                                   │
│ Parse → Bind → Optimize → Execute (매번 반복)               │
│                                                             │
│ Prepared Statement:                                         │
│ 1. PREPARE stmt AS SELECT * FROM t WHERE id = $1;          │
│    Parse → Bind (파라미터 제외) → 저장                       │
│                                                             │
│ 2. EXECUTE stmt(123);                                       │
│    파라미터 바인딩 → (필요시 Optimize) → Execute            │
│                                                             │
│ 장점:                                                        │
│ - Parse 오버헤드 제거                                        │
│ - SQL Injection 방지                                        │
│ - 일부 DBMS: 실행 계획 캐싱                                 │
│                                                             │
│ PostgreSQL Plan Caching:                                    │
│ - 처음 5회: 파라미터별 Custom Plan                          │
│ - 이후: Generic Plan 사용 여부 결정                         │
│ - plan_cache_mode 설정으로 제어                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL Prepared Statement
PREPARE get_user(int) AS
SELECT * FROM users WHERE id = $1;

EXECUTE get_user(123);
EXECUTE get_user(456);

DEALLOCATE get_user;

-- 실행 계획 확인
EXPLAIN EXECUTE get_user(123);
```

### 8. JIT Compilation

일부 DBMS에서 쿼리를 기계어로 컴파일하여 실행:

```
┌─────────────────────────────────────────────────────────────┐
│              JIT Compilation (PostgreSQL 11+)                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 일반 실행:                                                   │
│ Executor → 일반적인 Tuple-at-a-time 인터프리터             │
│                                                             │
│ JIT 실행:                                                    │
│ 1. 표현식을 LLVM IR로 컴파일                                │
│ 2. 기계어로 변환                                             │
│ 3. 컴파일된 코드 실행                                        │
│                                                             │
│ JIT 대상:                                                    │
│ - WHERE 조건 평가                                            │
│ - Projection 계산                                            │
│ - 집계 함수                                                  │
│ - 튜플 역직렬화                                              │
│                                                             │
│ 장점:                                                        │
│ - 함수 호출 오버헤드 제거                                    │
│ - CPU 최적화 (SIMD 등)                                      │
│ - 대규모 스캔에서 효과적                                     │
│                                                             │
│ 단점:                                                        │
│ - 컴파일 시간 오버헤드                                       │
│ - 작은 쿼리에서는 비효율                                     │
│                                                             │
│ 설정:                                                        │
│ jit = on                                                    │
│ jit_above_cost = 100000  -- 이 비용 이상일 때 JIT          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 실무 적용

### 쿼리 컴파일 시간 확인

```sql
-- PostgreSQL: 쿼리 실행 시간 분해
EXPLAIN (ANALYZE, TIMING)
SELECT * FROM large_table WHERE complex_condition;

-- Planning Time: 컴파일 시간
-- Execution Time: 실행 시간

-- 복잡한 쿼리에서 Planning Time이 길면
-- Prepared Statement 고려
```

### 플랜 캐시 효과 확인

```sql
-- PostgreSQL: pg_stat_statements
CREATE EXTENSION pg_stat_statements;

SELECT
    query,
    calls,
    total_time / calls as avg_time,
    rows / calls as avg_rows
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```

## 참고 자료

- "Architecture of a Database System" (Hellerstein) - Section 4
- CMU 15-445: Query Processing
- PostgreSQL Documentation: Query Planning
- "Database System Concepts" (Silberschatz) - Chapter 15

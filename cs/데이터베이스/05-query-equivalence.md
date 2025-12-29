# 쿼리 동등성 (Query Equivalence)

## 개요

쿼리 동등성은 **서로 다른 형태의 쿼리가 동일한 결과를 반환**하는지를 판단하는 이론이다. 쿼리 옵티마이저는 이 원리를 활용하여 비효율적인 쿼리를 동등하지만 더 효율적인 형태로 변환한다. 대수적 최적화 규칙을 이해하면 옵티마이저의 동작을 예측하고, 더 나은 쿼리를 작성할 수 있다.

## 핵심 개념

### 1. 쿼리 동등성의 정의

두 쿼리 Q₁과 Q₂가 **동등(equivalent)**하다는 것:

```
모든 가능한 데이터베이스 인스턴스 D에 대해:
Q₁(D) = Q₂(D)

즉, 어떤 데이터가 들어있든 동일한 결과를 반환
```

**주의**: SQL에서 결과는 **멀티셋(multiset)**이므로, 행의 순서와 중복을 고려해야 한다.

### 2. 관계 대수의 동등성 규칙

#### 선택(σ) 연산 규칙

```
┌─────────────────────────────────────────────────────────────┐
│ 선택 연산 동등성                                             │
├─────────────────────────────────────────────────────────────┤
│ 1. σ₁∧σ₂(R) = σ₁(σ₂(R)) = σ₂(σ₁(R))                        │
│    연쇄(Cascade): 복합 조건을 분리 가능                      │
│                                                             │
│ 2. σ₁(σ₂(R)) = σ₂(σ₁(R))                                    │
│    교환(Commute): 선택 순서 교환 가능                        │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 동등한 쿼리들
SELECT * FROM orders WHERE status = 'active' AND amount > 100;
SELECT * FROM orders WHERE amount > 100 AND status = 'active';
-- σ_status='active' ∧ amount>100(orders)
-- = σ_status='active'(σ_amount>100(orders))
-- = σ_amount>100(σ_status='active'(orders))
```

#### 투영(π) 연산 규칙

```
┌─────────────────────────────────────────────────────────────┐
│ 투영 연산 동등성                                             │
├─────────────────────────────────────────────────────────────┤
│ 1. π_A(π_B(R)) = π_A(R)  (단, A ⊆ B)                        │
│    연쇄: 더 좁은 투영이 결과                                 │
│                                                             │
│ 2. π_A(σ_c(R)) = σ_c(π_A(R))                                │
│    (단, 조건 c가 A의 속성만 사용할 때)                       │
│    선택과 투영 교환 가능                                     │
└─────────────────────────────────────────────────────────────┘
```

#### 조인(⋈) 연산 규칙

```
┌─────────────────────────────────────────────────────────────┐
│ 조인 연산 동등성                                             │
├─────────────────────────────────────────────────────────────┤
│ 1. R ⋈ S = S ⋈ R                                            │
│    교환(Commutative): 조인 순서 교환 가능                    │
│                                                             │
│ 2. (R ⋈ S) ⋈ T = R ⋈ (S ⋈ T)                                │
│    결합(Associative): 조인 순서 재배치 가능                  │
│                                                             │
│ 3. σ_c(R ⋈ S) = σ_c(R) ⋈ S                                  │
│    (단, c가 R의 속성만 사용할 때)                            │
│    선택 푸시다운(Predicate Pushdown)                         │
│                                                             │
│ 4. π_A(R ⋈ S) = π_A(π_B(R) ⋈ π_C(S))                        │
│    (단, B, C는 조인 속성과 A에 필요한 속성)                  │
│    투영 푸시다운                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3. 선택 푸시다운 (Predicate Pushdown)

가장 중요한 최적화 규칙 중 하나: **선택 조건을 가능한 일찍 적용**

```sql
-- 최적화 전
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE e.salary > 50000 AND d.region = 'Seoul';

-- 최적화 후 (개념적)
-- σ_salary>50000(employees) ⋈ σ_region='Seoul'(departments)

-- 이유: 조인 전에 필터링하면 처리할 행 수가 줄어듦
```

```
최적화 전:
employees (1,000,000 rows) ⋈ departments (100 rows)
→ 조인 비용: 1,000,000 × 100
→ 필터링 후 결과: 1,000 rows

최적화 후:
σ_salary>50000(employees) (50,000 rows) ⋈ σ_region='Seoul'(departments) (10 rows)
→ 조인 비용: 50,000 × 10
→ 500배 효율적
```

### 4. 조인 재배열 (Join Reordering)

조인은 교환 및 결합 법칙이 성립하므로 순서를 재배열할 수 있다.

```sql
-- 세 테이블 조인: 여러 순서 가능
SELECT *
FROM A
JOIN B ON A.b_id = B.id
JOIN C ON B.c_id = C.id;

-- 가능한 조인 순서
-- 1. (A ⋈ B) ⋈ C
-- 2. (A ⋈ C) ⋈ B  (직접 조인 조건 없으면 비효율)
-- 3. (B ⋈ C) ⋈ A
-- 4. A ⋈ (B ⋈ C)
-- ... (더 많은 조합)
```

**조인 순서 선택 기준:**
- **선택도(Selectivity)**: 결과 행이 적은 조인을 먼저
- **인덱스 활용**: 인덱스 스캔이 가능한 조인을 먼저
- **메모리 제약**: 작은 테이블을 내부 테이블로

### 5. 서브쿼리 평탄화 (Subquery Flattening)

중첩 서브쿼리를 조인으로 변환하여 최적화 가능성을 높인다.

```sql
-- 서브쿼리 형태
SELECT * FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE region = 'Seoul');

-- 평탄화 후 (조인으로 변환)
SELECT DISTINCT e.*
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE d.region = 'Seoul';

-- EXISTS도 변환 가능
SELECT * FROM employees e
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.emp_id = e.id);

-- 세미조인으로 변환
SELECT DISTINCT e.*
FROM employees e
SEMI JOIN orders o ON o.emp_id = e.id;
```

**평탄화가 불가능한 경우:**
- 상관 서브쿼리의 복잡한 상관관계
- LIMIT이 있는 서브쿼리
- 집계 결과에 따른 분기

### 6. 뷰 업데이트 문제 (View Update Problem)

모든 뷰가 갱신 가능한 것은 아니다.

```sql
-- 갱신 가능한 뷰
CREATE VIEW active_employees AS
SELECT * FROM employees WHERE status = 'active';

INSERT INTO active_employees (id, name, status)
VALUES (1, 'Kim', 'active');  -- OK

-- 갱신 불가능한 뷰
CREATE VIEW dept_summary AS
SELECT dept_id, COUNT(*) as cnt, AVG(salary) as avg_salary
FROM employees
GROUP BY dept_id;

INSERT INTO dept_summary VALUES (1, 10, 50000);  -- 불가능!
-- COUNT, AVG는 어떻게 역으로 변환?
```

**갱신 가능 조건 (SQL 표준):**
1. DISTINCT 없음
2. 집계 함수 없음
3. GROUP BY 없음
4. UNION, INTERSECT, EXCEPT 없음
5. FROM에 하나의 갱신 가능한 테이블/뷰
6. WHERE에 서브쿼리 없음

### 7. 공통 부분식 제거 (Common Subexpression Elimination)

동일한 표현식이 여러 번 나타나면 한 번만 계산한다.

```sql
-- CSE 적용 전
SELECT
    amount * tax_rate AS tax,
    amount * tax_rate * 0.1 AS fee
FROM orders;

-- CSE 적용 후 (개념적)
SELECT
    t.tax,
    t.tax * 0.1 AS fee
FROM (
    SELECT amount * tax_rate AS tax FROM orders
) t;
```

### 8. 대수적 최적화 vs 비용 기반 최적화

```
┌─────────────────────────────────────────────────────────────┐
│ 대수적 최적화 (Rule-based)                                   │
├─────────────────────────────────────────────────────────────┤
│ - 항상 적용되는 변환 규칙                                    │
│ - 선택 푸시다운, 불필요한 조인 제거 등                       │
│ - 비용 추정 없이 적용                                        │
│ - 실행 순서 초기에 적용                                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 비용 기반 최적화 (Cost-based)                                │
├─────────────────────────────────────────────────────────────┤
│ - 여러 대안 중 비용이 가장 낮은 것 선택                      │
│ - 조인 순서, 인덱스 선택, 조인 방법 등                       │
│ - 통계 정보(행 수, 카디널리티 등) 필요                       │
│ - 대수적 최적화 후 적용                                      │
└─────────────────────────────────────────────────────────────┘
```

## 실무 적용

### 동등 변환을 이용한 쿼리 튜닝

```sql
-- 비효율적: 상관 서브쿼리
SELECT *
FROM orders o
WHERE o.amount > (
    SELECT AVG(amount)
    FROM orders o2
    WHERE o2.customer_id = o.customer_id
);
-- 각 행마다 서브쿼리 실행 (개념적)

-- 효율적: 조인으로 변환
SELECT o.*
FROM orders o
JOIN (
    SELECT customer_id, AVG(amount) as avg_amount
    FROM orders
    GROUP BY customer_id
) avg_by_customer ON o.customer_id = avg_by_customer.customer_id
WHERE o.amount > avg_by_customer.avg_amount;
-- 서브쿼리 한 번 실행 후 조인
```

### 조인 순서 힌트

옵티마이저가 잘못된 조인 순서를 선택할 때:

```sql
-- PostgreSQL
SET join_collapse_limit = 1;  -- 조인 재배열 비활성화
SELECT /*+ Leading((a b) c) */ ...;  -- 힌트 (확장)

-- MySQL
SELECT /*+ JOIN_ORDER(a, b, c) */ ...;
SELECT STRAIGHT_JOIN ...;  -- FROM 순서대로 조인

-- Oracle
SELECT /*+ LEADING(a b c) */ ...;
SELECT /*+ ORDERED */ ...;
```

### EXPLAIN으로 최적화 확인

```sql
-- PostgreSQL
EXPLAIN (ANALYZE, VERBOSE)
SELECT e.name, d.dept_name
FROM employees e
JOIN departments d ON e.dept_id = d.id
WHERE e.salary > 50000;

-- 실행 계획에서 확인:
-- 1. 선택 조건이 푸시다운 되었는지
-- 2. 조인 순서와 방법
-- 3. 인덱스 사용 여부
```

## 참고 자료

- "Database System Concepts" (Silberschatz) - Chapter 16: Query Optimization
- "Query Processing and Optimization" - CMU 15-445
- PostgreSQL Documentation: Controlling the Planner
- MySQL Reference Manual: Query Optimization

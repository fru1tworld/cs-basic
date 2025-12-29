# SQL 의미론 (SQL Semantics)

## 개요

SQL은 관계형 데이터베이스의 표준 쿼리 언어로, 선언적(declarative) 특성을 가진다. **무엇을** 원하는지 명시하면 DBMS가 **어떻게** 실행할지 결정한다. SQL의 정확한 의미론을 이해하면 예상치 못한 쿼리 결과를 방지하고, NULL 처리나 집계 함수의 동작을 정확히 예측할 수 있다.

## 핵심 개념

### 1. SQL의 논리적 처리 순서

SQL 쿼리는 작성 순서와 다른 **논리적 처리 순서**를 따른다:

```
┌─────────────────────────────────────────────────────────────┐
│           SQL 논리적 처리 순서                               │
├─────────────────────────────────────────────────────────────┤
│ 1. FROM       - 테이블 지정 및 조인                         │
│ 2. WHERE      - 행 필터링 (그룹화 전)                       │
│ 3. GROUP BY   - 그룹화                                      │
│ 4. HAVING     - 그룹 필터링                                 │
│ 5. SELECT     - 컬럼 선택 및 표현식 계산                    │
│ 6. DISTINCT   - 중복 제거                                   │
│ 7. ORDER BY   - 정렬                                        │
│ 8. LIMIT      - 결과 제한                                   │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 논리적 처리 순서의 영향
SELECT dept_id, COUNT(*) as cnt     -- (5) SELECT에서 정의한 별칭
FROM employees                       -- (1) FROM
WHERE status = 'active'             -- (2) WHERE
GROUP BY dept_id                    -- (3) GROUP BY
HAVING COUNT(*) > 5                 -- (4) HAVING - cnt 사용 불가 (SELECT 이전)
ORDER BY cnt DESC;                  -- (7) ORDER BY - cnt 사용 가능 (SELECT 이후)

-- HAVING에서 별칭 사용 불가한 이유:
-- HAVING은 SELECT 전에 처리되므로 cnt가 아직 정의되지 않음
-- (MySQL은 확장으로 허용하지만, 표준은 아님)
```

### 2. NULL 처리와 3-Valued Logic

SQL은 **3값 논리(true, false, unknown)**를 사용한다.

```
┌─────────────────────────────────────────────────────────────┐
│           3-Valued Logic Truth Tables                        │
├─────────────────────────────────────────────────────────────┤
│   AND    │ TRUE   │ FALSE  │ UNKNOWN │                      │
│   TRUE   │ TRUE   │ FALSE  │ UNKNOWN │                      │
│   FALSE  │ FALSE  │ FALSE  │ FALSE   │                      │
│   UNKNOWN│ UNKNOWN│ FALSE  │ UNKNOWN │                      │
├─────────────────────────────────────────────────────────────┤
│   OR     │ TRUE   │ FALSE  │ UNKNOWN │                      │
│   TRUE   │ TRUE   │ TRUE   │ TRUE    │                      │
│   FALSE  │ TRUE   │ FALSE  │ UNKNOWN │                      │
│   UNKNOWN│ TRUE   │ UNKNOWN│ UNKNOWN │                      │
├─────────────────────────────────────────────────────────────┤
│   NOT    │                                                   │
│   TRUE   │ FALSE                                             │
│   FALSE  │ TRUE                                              │
│   UNKNOWN│ UNKNOWN                                           │
└─────────────────────────────────────────────────────────────┘
```

**NULL 비교의 함정:**

```sql
-- NULL과의 비교는 항상 UNKNOWN
SELECT * FROM employees WHERE dept_id = NULL;    -- 결과 없음 (항상)
SELECT * FROM employees WHERE dept_id <> NULL;   -- 결과 없음 (항상)

-- NULL 비교에는 IS NULL / IS NOT NULL 사용
SELECT * FROM employees WHERE dept_id IS NULL;
SELECT * FROM employees WHERE dept_id IS NOT NULL;

-- NOT IN과 NULL의 문제
SELECT * FROM employees
WHERE dept_id NOT IN (SELECT dept_id FROM departments);
-- departments에 NULL이 있으면 결과가 항상 빈 집합

-- 이유:
-- NOT IN (1, 2, NULL) = (x <> 1) AND (x <> 2) AND (x <> NULL)
-- x <> NULL = UNKNOWN
-- 어떤 값이든 UNKNOWN과 AND하면 TRUE가 될 수 없음

-- 해결책: NOT EXISTS 또는 NULL 제외
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM departments d WHERE d.dept_id = e.dept_id
);
-- 또는
SELECT * FROM employees
WHERE dept_id NOT IN (
    SELECT dept_id FROM departments WHERE dept_id IS NOT NULL
);
```

### 3. 집계 함수의 정의

```sql
-- 집계 함수와 NULL
COUNT(*)      -- 모든 행 수 (NULL 포함)
COUNT(column) -- NULL이 아닌 값의 수
SUM(column)   -- NULL 제외하고 합계, 모두 NULL이면 NULL
AVG(column)   -- NULL 제외하고 평균, 모두 NULL이면 NULL
MAX/MIN       -- NULL 제외, 모두 NULL이면 NULL

-- 예시
| value |
|-------|
| 10    |
| NULL  |
| 30    |

COUNT(*)      = 3
COUNT(value)  = 2
SUM(value)    = 40
AVG(value)    = 20    -- (10 + 30) / 2, NULL 제외
```

**DISTINCT와 집계:**

```sql
-- COUNT(DISTINCT column)
SELECT COUNT(DISTINCT dept_id) FROM employees;
-- NULL은 하나로 취급되어 계산에서 제외

-- GROUP BY에서 NULL
SELECT dept_id, COUNT(*)
FROM employees
GROUP BY dept_id;
-- NULL 값들은 하나의 그룹으로 묶임
```

### 4. 서브쿼리 유형과 상관관계

```sql
-- 1. 스칼라 서브쿼리: 단일 값 반환
SELECT name,
       (SELECT dept_name FROM departments d
        WHERE d.id = e.dept_id) as dept_name
FROM employees e;

-- 2. 행 서브쿼리: 단일 행 반환
SELECT * FROM employees
WHERE (dept_id, level) = (SELECT dept_id, level FROM managers WHERE id = 1);

-- 3. 테이블 서브쿼리: 여러 행/열 반환
SELECT * FROM employees
WHERE dept_id IN (SELECT id FROM departments WHERE region = 'Seoul');

-- 4. 상관 서브쿼리 (Correlated Subquery)
SELECT * FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees e2
                WHERE e2.dept_id = e.dept_id);
-- 외부 쿼리의 각 행에 대해 서브쿼리 실행 (개념적으로)
```

**EXISTS vs IN:**

```sql
-- EXISTS: 결과 존재 여부만 확인
SELECT * FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.id);
-- 서브쿼리가 한 행이라도 반환하면 TRUE

-- IN: 값 목록과 비교
SELECT * FROM departments
WHERE id IN (SELECT dept_id FROM employees);

-- 차이점:
-- 1. EXISTS는 NULL에 안전 (결과 유무만 판단)
-- 2. EXISTS는 보통 상관 서브쿼리와 사용
-- 3. 옵티마이저가 같은 실행 계획으로 변환하기도 함
```

### 5. 조인의 의미론

```sql
-- INNER JOIN: 양쪽 테이블에서 조건 만족하는 행만
SELECT * FROM A INNER JOIN B ON A.id = B.a_id;
-- NULL = NULL은 UNKNOWN이므로, NULL 키는 조인되지 않음

-- LEFT (OUTER) JOIN: 왼쪽 테이블 전체 + 매칭되는 오른쪽
SELECT * FROM A LEFT JOIN B ON A.id = B.a_id;
-- A의 모든 행, 매칭 없으면 B 컬럼은 NULL

-- FULL OUTER JOIN: 양쪽 전체
SELECT * FROM A FULL OUTER JOIN B ON A.id = B.a_id;

-- CROSS JOIN: 카티션 곱
SELECT * FROM A CROSS JOIN B;
-- A의 행 수 × B의 행 수
```

**조인 조건 위치의 의미:**

```sql
-- INNER JOIN: ON과 WHERE 동일한 결과
SELECT * FROM A JOIN B ON A.id = B.a_id AND B.status = 'active';
SELECT * FROM A JOIN B ON A.id = B.a_id WHERE B.status = 'active';
-- 동일 결과 (조인 순서에 따라 성능은 다를 수 있음)

-- LEFT JOIN: ON과 WHERE 다른 결과
SELECT * FROM A LEFT JOIN B ON A.id = B.a_id AND B.status = 'active';
-- A의 모든 행 유지, B는 조건 만족할 때만 매칭

SELECT * FROM A LEFT JOIN B ON A.id = B.a_id WHERE B.status = 'active';
-- LEFT JOIN 결과에서 B.status = 'active'만 필터
-- B 매칭 없는 A 행도 제거됨 (B.status가 NULL이므로)
```

### 6. UNION과 집합 연산

```sql
-- UNION: 중복 제거된 합집합
SELECT name FROM employees
UNION
SELECT name FROM contractors;

-- UNION ALL: 중복 포함 합집합
SELECT name FROM employees
UNION ALL
SELECT name FROM contractors;
-- 중복 제거 안 함, 더 빠름

-- INTERSECT: 교집합
SELECT name FROM employees
INTERSECT
SELECT name FROM contractors;

-- EXCEPT (MINUS): 차집합
SELECT name FROM employees
EXCEPT
SELECT name FROM contractors;
```

**집합 연산과 NULL:**

```sql
-- UNION에서 NULL 처리
-- NULL은 같은 값으로 취급 (중복 제거 시)
SELECT NULL UNION SELECT NULL;  -- 결과: 1개의 NULL 행

-- INTERSECT에서 NULL
SELECT NULL INTERSECT SELECT NULL;  -- 결과: NULL (NULL = NULL로 취급)

-- 이는 WHERE에서 NULL = NULL이 UNKNOWN인 것과 다름!
-- 집합 연산은 "같은 값인지" 비교에서 NULL을 동일하게 취급
```

### 7. 윈도우 함수

```sql
-- 기본 구문
SELECT
    column,
    window_function() OVER (
        PARTITION BY partition_column
        ORDER BY order_column
        frame_clause
    )
FROM table;

-- 프레임 절 (Frame Clause)
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- 기본값 (ORDER BY 있을 때)
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- 기본값 (ORDER BY 없을 때)

RANGE BETWEEN ... -- 값 기준
ROWS BETWEEN ...  -- 행 기준
```

```sql
-- 예제: 누적 합계
SELECT
    date,
    amount,
    SUM(amount) OVER (ORDER BY date) as running_total
FROM sales;

-- 예제: 이동 평균
SELECT
    date,
    amount,
    AVG(amount) OVER (
        ORDER BY date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg_3
FROM sales;

-- 예제: 순위 함수
SELECT
    name,
    score,
    RANK() OVER (ORDER BY score DESC) as rank,
    DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank,
    ROW_NUMBER() OVER (ORDER BY score DESC) as row_num
FROM students;

-- RANK: 동점 시 같은 순위, 다음 순위 건너뜀 (1, 1, 3)
-- DENSE_RANK: 동점 시 같은 순위, 다음 순위 연속 (1, 1, 2)
-- ROW_NUMBER: 항상 고유한 번호 (1, 2, 3)
```

### 8. SQL 표준 (SQL:2016, SQL:2023)

```
SQL:2016 주요 추가 사항:
├── JSON 지원 (JSON_OBJECT, JSON_ARRAY, JSON_TABLE 등)
├── Row pattern recognition (MATCH_RECOGNIZE)
├── 다형성 테이블 함수 (Polymorphic Table Functions)
└── LISTAGG 함수

SQL:2023 주요 추가 사항:
├── GRAPH 쿼리 (SQL/PGQ - Property Graph Queries)
├── JSON 스키마 지원
├── ANY_VALUE 집계 함수
└── 향상된 윈도우 함수
```

## 실무 적용

### NULL-safe 쿼리 작성

```sql
-- Anti-pattern: NULL 비교
SELECT * FROM orders WHERE status = NULL;  -- 항상 빈 결과

-- 올바른 방법
SELECT * FROM orders WHERE status IS NULL;

-- Anti-pattern: NOT IN with NULL
SELECT * FROM products
WHERE category_id NOT IN (SELECT id FROM categories);

-- 올바른 방법
SELECT * FROM products p
WHERE NOT EXISTS (SELECT 1 FROM categories c WHERE c.id = p.category_id);

-- 또는 COALESCE 활용
SELECT * FROM products
WHERE COALESCE(category_id, -1) NOT IN (
    SELECT COALESCE(id, -1) FROM categories
);
```

### 집계 함수의 정확한 사용

```sql
-- 비율 계산 시 NULL 주의
SELECT
    COUNT(*) as total,
    COUNT(status) as with_status,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed,
    -- 완료 비율 (status가 있는 것 중에서)
    100.0 * COUNT(CASE WHEN status = 'completed' THEN 1 END)
        / NULLIF(COUNT(status), 0) as completion_rate
FROM orders;

-- 평균 계산: NULL 제외됨을 인지
SELECT
    AVG(rating) as avg_rating,  -- NULL 제외
    SUM(rating) / COUNT(*) as avg_with_null_as_zero  -- 다른 결과
FROM reviews;
```

### 조인 조건 최적화

```sql
-- LEFT JOIN 후 필터링 시 주의
-- 의도: 모든 고객 + 활성 주문 (있는 경우)
SELECT c.*, o.order_id
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id AND o.status = 'active';
-- 모든 고객 포함, 활성 주문만 매칭

-- 실수하기 쉬운 패턴
SELECT c.*, o.order_id
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE o.status = 'active';  -- 매칭 없는 고객은 제외됨!
-- LEFT JOIN의 의미가 사라짐
```

## 참고 자료

- SQL:2016 Standard (ISO/IEC 9075)
- "SQL and Relational Theory" - C.J. Date
- PostgreSQL Documentation: SQL Syntax
- MySQL Reference Manual: SQL Statements
- CMU 15-445: Query Processing

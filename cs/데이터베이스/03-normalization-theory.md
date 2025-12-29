# 정규화 이론 (Normalization Theory)

## 개요

정규화는 관계형 데이터베이스 설계에서 **데이터 중복을 최소화**하고 **갱신 이상(Anomaly)**을 방지하기 위한 체계적인 방법론이다. 함수적 종속성과 다치 종속성을 기반으로 릴레이션을 분해하여 높은 정규형으로 변환한다. 정규화의 핵심은 **하나의 사실은 하나의 장소에만 저장**하는 원칙이다.

## 핵심 개념

### 1. 갱신 이상 (Update Anomalies)

정규화가 필요한 이유를 이해하기 위해 갱신 이상을 먼저 살펴본다.

```sql
-- 비정규화된 테이블
CREATE TABLE StudentCourse (
    student_id INT,
    student_name VARCHAR(100),
    dept_id INT,
    dept_name VARCHAR(100),
    course_id INT,
    grade CHAR(2)
);

-- 데이터 예시
| student_id | student_name | dept_id | dept_name | course_id | grade |
|------------|--------------|---------|-----------|-----------|-------|
| 1          | Kim          | 10      | CS        | 101       | A     |
| 1          | Kim          | 10      | CS        | 102       | B     |
| 2          | Lee          | 10      | CS        | 101       | A     |
| 3          | Park         | 20      | EE        | 201       | B     |
```

```
┌─────────────────────────────────────────────────────────────┐
│ 삽입 이상 (Insertion Anomaly)                                │
├─────────────────────────────────────────────────────────────┤
│ 새 학과(dept_id=30, Math)를 추가하려면?                      │
│ → 학생 없이는 학과만 추가 불가능 (student_id가 키이므로)      │
│ → NULL 허용 시 데이터 무결성 문제                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 삭제 이상 (Deletion Anomaly)                                 │
├─────────────────────────────────────────────────────────────┤
│ Park(student_id=3)의 수강 정보를 삭제하면?                   │
│ → EE 학과 정보도 함께 삭제됨                                 │
│ → 의도치 않은 정보 손실                                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 갱신 이상 (Update Anomaly)                                   │
├─────────────────────────────────────────────────────────────┤
│ CS 학과명을 "Computer Science"로 변경하려면?                 │
│ → 3개 행 모두 수정 필요                                      │
│ → 일부만 수정 시 불일치 발생                                 │
└─────────────────────────────────────────────────────────────┘
```

### 2. 정규형의 계층 구조

```
1NF ⊃ 2NF ⊃ 3NF ⊃ BCNF ⊃ 4NF ⊃ 5NF ⊃ 6NF

더 높은 정규형 = 더 적은 중복 = 더 많은 테이블 분해
```

### 3. 제1정규형 (1NF)

**정의**: 모든 속성값이 **원자값(atomic value)**이어야 한다.

```sql
-- 1NF 위반
| student_id | courses          |
|------------|------------------|
| 1          | CS101, CS102     |  -- 다중값
| 2          | CS101            |

-- 1NF 준수
| student_id | course_id |
|------------|-----------|
| 1          | CS101     |
| 1          | CS102     |
| 2          | CS101     |
```

**실무 주의사항**:
- JSON 컬럼은 1NF 위반으로 볼 수 있으나, 현대 DB에서 자주 사용
- 반복 그룹(repeating groups) 제거 필요
- "원자값"의 정의는 도메인에 따라 다름 (주소를 하나로 볼지, 분리할지)

### 4. 제2정규형 (2NF)

**정의**: 1NF이면서, 모든 비주요 속성이 기본키에 **완전 함수적 종속**이어야 한다.

```sql
-- 2NF 위반: 부분 함수적 종속 존재
-- 기본키: {student_id, course_id}
| student_id | course_id | student_name | grade |
|------------|-----------|--------------|-------|
-- student_id → student_name (부분 종속: 키의 일부에만 종속)

-- 2NF 준수: 분해
-- Students(student_id, student_name)
-- Enrollments(student_id, course_id, grade)
```

**판별 방법**:
1. 기본키가 복합키인지 확인
2. 복합키의 일부분에만 종속되는 속성이 있는지 확인
3. 단일 속성 기본키면 자동으로 2NF

### 5. 제3정규형 (3NF)

**정의**: 2NF이면서, 모든 비주요 속성이 기본키에 **이행적으로 종속되지 않아야** 한다.

```
X → Y가 3NF를 만족하려면 다음 중 하나:
1. X → Y가 자명한 FD (Y ⊆ X)
2. X가 슈퍼키
3. Y - X의 모든 속성이 후보키의 일부
```

```sql
-- 3NF 위반: 이행적 종속 존재
-- student_id → dept_id → dept_name
| student_id | student_name | dept_id | dept_name |
|------------|--------------|---------|-----------|

-- 3NF 준수: 분해
-- Students(student_id, student_name, dept_id)
-- Departments(dept_id, dept_name)
```

**이행적 종속 탐지**:
```
A → B → C (A가 키, B는 비키 속성, C도 비키 속성)
→ A → C는 이행적 종속
→ 3NF 위반
```

### 6. BCNF (Boyce-Codd Normal Form)

**정의**: 모든 비자명한 FD X → Y에서, X가 **슈퍼키**여야 한다.

3NF와의 차이:
```
3NF: X가 슈퍼키 OR Y가 후보키의 일부
BCNF: X가 반드시 슈퍼키

BCNF가 더 엄격함
```

```sql
-- BCNF 위반, 3NF 만족 예시
-- R(student, course, instructor)
-- FD: {student, course} → instructor
--     instructor → course (강사는 한 과목만 담당)

-- 후보키: {student, course}, {student, instructor}
-- instructor → course에서 instructor는 슈퍼키가 아님 → BCNF 위반
-- 하지만 course는 후보키의 일부 → 3NF 만족

-- BCNF 분해
-- R1(instructor, course)  -- instructor → course
-- R2(student, instructor) -- 조인 속성
```

**BCNF 분해의 한계**: 종속성 보존이 보장되지 않음

### 7. 제4정규형 (4NF)

**다치 종속성(Multi-valued Dependency)**: X →→ Y

```
X →→ Y: X값이 같은 튜플들에서 Y값들의 집합이 X에만 의존하고,
        나머지 속성(Z)과 독립적
```

**정의**: BCNF이면서, 모든 비자명한 다치 종속성 X →→ Y에서 X가 슈퍼키여야 한다.

```sql
-- 4NF 위반 예시
-- Employee(emp_id, skill, language)
-- 직원은 여러 기술과 여러 언어를 독립적으로 가질 수 있음
| emp_id | skill | language |
|--------|-------|----------|
| 1      | Java  | Korean   |
| 1      | Java  | English  |
| 1      | SQL   | Korean   |
| 1      | SQL   | English  |

-- emp_id →→ skill (skill과 language는 독립)
-- emp_id →→ language

-- 4NF 준수: 분해
-- EmpSkills(emp_id, skill)
-- EmpLanguages(emp_id, language)
```

### 8. 제5정규형 (5NF / PJNF)

**조인 종속성(Join Dependency)**: *(R₁, R₂, ..., Rₙ)

릴레이션이 R₁, R₂, ..., Rₙ의 조인으로 무손실 분해 가능

**정의**: 4NF이면서, 모든 조인 종속성이 후보키에 의해 암시되어야 한다.

```sql
-- 5NF 예시: 3자 관계
-- SPJ(supplier, part, project)
-- "공급자가 부품을 공급하고, 부품이 프로젝트에 사용되고,
--  공급자가 프로젝트에 참여하면" → 공급자가 프로젝트에 그 부품을 공급

-- 조인 종속성: *(SP, PJ, SJ)
-- 세 개의 이진 관계로 분해 가능하고, 조인하면 원본 복원
```

### 9. 정규화 vs 반정규화

```
┌─────────────────────────────────────────────────────────────┐
│                    정규화                                    │
├─────────────────────────────────────────────────────────────┤
│ 장점:                                                       │
│ - 데이터 중복 제거 → 저장 공간 절약                          │
│ - 갱신 이상 방지 → 데이터 무결성                            │
│ - 유연한 스키마 변경                                        │
│                                                             │
│ 단점:                                                       │
│ - 조인 증가 → 읽기 성능 저하 가능                           │
│ - 쿼리 복잡도 증가                                          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   반정규화                                   │
├─────────────────────────────────────────────────────────────┤
│ 장점:                                                       │
│ - 조인 제거 → 읽기 성능 향상                                │
│ - 쿼리 단순화                                               │
│                                                             │
│ 단점:                                                       │
│ - 데이터 중복 → 저장 공간 증가                              │
│ - 갱신 이상 위험                                            │
│ - 데이터 불일치 가능성                                      │
└─────────────────────────────────────────────────────────────┘
```

## 실무 적용

### 정규화 수준 결정

```sql
-- 1. OLTP 시스템: 3NF 또는 BCNF 권장
-- 쓰기가 많고, 데이터 무결성이 중요
CREATE TABLE Orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES Customers,
    order_date DATE
);

CREATE TABLE OrderItems (
    order_id INT REFERENCES Orders,
    product_id INT REFERENCES Products,
    quantity INT,
    unit_price DECIMAL(10,2),  -- 주문 시점 가격 (반정규화)
    PRIMARY KEY (order_id, product_id)
);
-- unit_price는 의도적 반정규화: 상품 가격이 변해도 주문 기록 유지

-- 2. OLAP/분석 시스템: 반정규화된 스타/스노우플레이크 스키마
CREATE TABLE FactSales (
    sale_id SERIAL PRIMARY KEY,
    date_key INT,
    product_key INT,
    customer_key INT,
    quantity INT,
    amount DECIMAL(12,2),
    -- 차원 속성 반정규화 (읽기 최적화)
    product_name VARCHAR(100),
    category_name VARCHAR(50),
    customer_region VARCHAR(50)
);
```

### 반정규화 전략

```sql
-- 전략 1: 계산 컬럼 추가
ALTER TABLE Orders ADD COLUMN total_amount DECIMAL(12,2);
-- 트리거로 자동 갱신
CREATE TRIGGER update_total
AFTER INSERT OR UPDATE ON OrderItems
FOR EACH ROW EXECUTE FUNCTION calculate_order_total();

-- 전략 2: 중복 컬럼 추가
-- 자주 조회되는 외래 테이블 속성 복사
ALTER TABLE Orders ADD COLUMN customer_name VARCHAR(100);
-- 트리거 또는 애플리케이션에서 동기화

-- 전략 3: 파생 테이블/물리화 뷰
CREATE MATERIALIZED VIEW OrderSummary AS
SELECT
    o.order_id,
    o.order_date,
    c.customer_name,
    SUM(oi.quantity * oi.unit_price) as total
FROM Orders o
JOIN Customers c ON o.customer_id = c.customer_id
JOIN OrderItems oi ON o.order_id = oi.order_id
GROUP BY o.order_id, o.order_date, c.customer_name;

-- 주기적 갱신
REFRESH MATERIALIZED VIEW OrderSummary;
```

### 분해 시 고려사항

```sql
-- 무손실 분해 확인
-- 분해 R1, R2에서 R1 ∩ R2가 R1 또는 R2의 키여야 함

-- 예: R(A, B, C), F = {A → B}
-- 분해: R1(A, B), R2(A, C)
-- R1 ∩ R2 = {A}, A는 R1의 키 → 무손실

-- 종속성 보존 확인
-- 원본 FD가 분해된 테이블에서 검증 가능한지

-- 예: R(A, B, C), F = {A → B, B → C}
-- 분해: R1(A, B), R2(B, C)
-- A → B는 R1에서 검증 가능
-- B → C는 R2에서 검증 가능
-- → 종속성 보존됨
```

## 참고 자료

- Codd, E.F. "Further Normalization of the Data Base Relational Model" (1972)
- Kent, W. "A Simple Guide to Five Normal Forms" (1983)
- Database System Concepts (Silberschatz) - Chapter 7
- "Database Management Systems" (Ramakrishnan) - Chapter 19

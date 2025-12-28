# 정규화 (Normalization)

## 목차
1. [정규화 개요](#정규화-개요)
2. [제1정규형 (1NF)](#제1정규형-1nf)
3. [제2정규형 (2NF)](#제2정규형-2nf)
4. [제3정규형 (3NF)](#제3정규형-3nf)
5. [보이스-코드 정규형 (BCNF)](#보이스-코드-정규형-bcnf)
6. [반정규화 (Denormalization)](#반정규화-denormalization)
---

## 정규화 개요

### 정규화란?

정규화는 데이터베이스 설계 시 데이터 중복을 최소화하고 데이터 무결성을 보장하기 위해 테이블을 분리하는 과정입니다.

```
정규화 수준:
┌─────────────────────────────────────────────────────────────┐
│  비정규화        →  1NF  →  2NF  →  3NF  →  BCNF  →  4NF   │
│                                                              │
│  ◄──────── 중복 감소, 무결성 증가 ────────►                 │
│  ◄──────── 조인 증가, 복잡도 증가 ────────►                 │
│                                                              │
│  실무에서는 대부분 3NF 또는 BCNF까지 적용                    │
└─────────────────────────────────────────────────────────────┘
```

### 정규화의 목적

| 목적 | 설명 |
|------|------|
| 데이터 중복 제거 | 동일 데이터가 여러 곳에 저장되는 것 방지 |
| 갱신 이상 방지 | 수정 시 일부만 변경되는 문제 방지 |
| 삽입 이상 방지 | 불필요한 데이터 없이 삽입 불가 문제 방지 |
| 삭제 이상 방지 | 필요한 데이터가 함께 삭제되는 문제 방지 |

### 이상 현상 (Anomaly) 예시

```sql
-- 비정규화된 테이블
CREATE TABLE orders_denorm (
    order_id INT,
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    customer_name VARCHAR(100),
    customer_address VARCHAR(200)
);

-- 데이터 예시:
-- | order_id | product_name | product_price | customer_name | customer_address |
-- |----------|-------------|---------------|---------------|------------------|
-- | 1        | iPhone      | 1000          | 김철수         | 서울시 강남구     |
-- | 2        | iPhone      | 1000          | 이영희         | 부산시 해운대구   |
-- | 3        | MacBook     | 2000          | 김철수         | 서울시 강남구     |

-- 갱신 이상: iPhone 가격을 1100으로 변경하려면 모든 행 수정 필요
UPDATE orders_denorm SET product_price = 1100 WHERE product_name = 'iPhone';
-- 일부만 수정되면 데이터 불일치!

-- 삽입 이상: 새 상품을 등록하려면 주문이 있어야만 가능
-- INSERT INTO orders_denorm (product_name, product_price) VALUES ('AirPods', 200);
-- order_id가 NULL이 되는 문제

-- 삭제 이상: 주문 삭제 시 고객 정보도 함께 삭제됨
DELETE FROM orders_denorm WHERE order_id = 3;
-- 김철수의 주소 정보가 사라질 수 있음
```

---

## 제1정규형 (1NF)

### 1NF 정의

**모든 속성 값이 원자값(Atomic Value)이어야 함**

- 각 컬럼은 하나의 값만 포함
- 반복되는 그룹이 없어야 함
- 각 행은 유일하게 식별 가능해야 함

### 1NF 적용 예시

```sql
-- 1NF 위반: 다중값 속성
CREATE TABLE students_bad (
    student_id INT,
    name VARCHAR(50),
    phone_numbers VARCHAR(200)  -- "010-1234-5678, 02-9876-5432"
);

-- 1NF 적용 방법 1: 별도 테이블 분리
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE student_phones (
    student_id INT,
    phone_number VARCHAR(20),
    phone_type VARCHAR(10),  -- 'mobile', 'home', 'office'
    PRIMARY KEY (student_id, phone_number),
    FOREIGN KEY (student_id) REFERENCES students(student_id)
);

-- 1NF 적용 방법 2: 컬럼 분리 (비권장, 확장성 문제)
CREATE TABLE students_fixed (
    student_id INT PRIMARY KEY,
    name VARCHAR(50),
    phone_mobile VARCHAR(20),
    phone_home VARCHAR(20)
);
```

```sql
-- 1NF 위반: 반복 그룹
CREATE TABLE orders_bad (
    order_id INT,
    product1_name VARCHAR(50),
    product1_qty INT,
    product2_name VARCHAR(50),
    product2_qty INT,
    product3_name VARCHAR(50),
    product3_qty INT
);

-- 1NF 적용: 행으로 분리
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    order_date DATE,
    customer_id INT
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

---

## 제2정규형 (2NF)

### 2NF 정의

**1NF를 만족하고, 모든 비주요 속성이 기본키 전체에 완전 함수 종속**

- 부분 함수 종속 제거
- 복합 기본키의 일부에만 종속되는 속성 분리

### 2NF 적용 예시

```sql
-- 2NF 위반: 부분 함수 종속 존재
CREATE TABLE order_details_bad (
    order_id INT,
    product_id INT,
    quantity INT,
    product_name VARCHAR(100),    -- product_id에만 종속
    product_category VARCHAR(50), -- product_id에만 종속
    PRIMARY KEY (order_id, product_id)
);

-- 함수 종속 관계:
-- {order_id, product_id} → quantity (완전 함수 종속) ✓
-- {product_id} → product_name (부분 함수 종속) ✗
-- {product_id} → product_category (부분 함수 종속) ✗
```

```sql
-- 2NF 적용: 부분 종속 속성 분리
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    product_category VARCHAR(50)
);

CREATE TABLE order_details (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**함수 종속 다이어그램:**

```
Before (2NF 위반):
┌─────────────────────────────────┐
│     {order_id, product_id}      │
│              ↓                  │
│           quantity              │
└─────────────────────────────────┘
              ↓
┌─────────────────────────────────┐
│         {product_id}            │
│              ↓                  │
│  product_name, product_category │
└─────────────────────────────────┘

After (2NF 만족):
┌──────────────┐    ┌──────────────────────────┐
│   products   │    │     order_details        │
│──────────────│    │──────────────────────────│
│ product_id   │←───│ order_id, product_id (PK)│
│ product_name │    │ quantity                 │
│ category     │    └──────────────────────────┘
└──────────────┘
```

---

## 제3정규형 (3NF)

### 3NF 정의

**2NF를 만족하고, 모든 비주요 속성이 기본키에 이행적 함수 종속이 아님**

- 이행적 종속 제거: A → B, B → C 이면 A → C (이행적 종속)
- 비주요 속성이 다른 비주요 속성에 종속되면 안 됨

### 3NF 적용 예시

```sql
-- 3NF 위반: 이행적 함수 종속 존재
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),  -- department_id → department_name
    department_location VARCHAR(100) -- department_id → department_location
);

-- 함수 종속 관계:
-- employee_id → department_id (직접 종속) ✓
-- department_id → department_name (직접 종속) ✓
-- employee_id → department_name (이행적 종속) ✗
```

```sql
-- 3NF 적용: 이행적 종속 분리
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    department_location VARCHAR(100)
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);
```

**이행적 종속 다이어그램:**

```
Before (3NF 위반):
┌─────────────────────────────────────────────────┐
│                 employees_bad                    │
│─────────────────────────────────────────────────│
│ employee_id (PK)                                │
│      ↓                                          │
│ employee_name, department_id                    │
│                    ↓                            │
│           department_name, location  (이행적)    │
└─────────────────────────────────────────────────┘

After (3NF 만족):
┌──────────────────┐      ┌────────────────────────┐
│   departments    │      │      employees         │
│──────────────────│      │────────────────────────│
│ department_id    │←─────│ employee_id (PK)       │
│ department_name  │      │ employee_name          │
│ location         │      │ department_id (FK)     │
└──────────────────┘      └────────────────────────┘
```

### 3NF 실무 예시

```sql
-- 3NF 위반: 주문 테이블에 고객 정보 포함
CREATE TABLE orders_bad (
    order_id INT PRIMARY KEY,
    order_date DATE,
    customer_id INT,
    customer_name VARCHAR(100),    -- customer_id에 종속
    customer_email VARCHAR(100),   -- customer_id에 종속
    total_amount DECIMAL(10,2)
);

-- 3NF 적용
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    order_date DATE,
    customer_id INT,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

---

## 보이스-코드 정규형 (BCNF)

### BCNF 정의

**모든 결정자가 후보키(Candidate Key)여야 함**

- 3NF보다 엄격한 조건
- 3NF는 만족하지만 BCNF는 위반하는 경우가 있음
- 비주요 속성이 아닌 속성이 결정자일 때 발생

### BCNF 위반 예시

```sql
-- 상황: 학생이 과목을 수강하고, 각 과목은 한 교수만 담당
-- 제약조건:
-- 1. 한 학생은 한 과목을 한 교수에게만 수강
-- 2. 한 교수는 한 과목만 담당

CREATE TABLE course_enrollment_bad (
    student_id INT,
    course VARCHAR(50),
    professor VARCHAR(50),
    PRIMARY KEY (student_id, course)
);

-- 함수 종속:
-- {student_id, course} → professor  (기본키 → 속성) ✓
-- {professor} → course              (비기본키 → 속성) ✗ BCNF 위반!

-- 문제: professor가 결정자이지만 후보키가 아님
```

```sql
-- BCNF 적용
CREATE TABLE professor_courses (
    professor_id INT PRIMARY KEY,
    professor_name VARCHAR(100),
    course VARCHAR(50) UNIQUE  -- 교수당 하나의 과목
);

CREATE TABLE enrollments (
    student_id INT,
    professor_id INT,
    PRIMARY KEY (student_id, professor_id),
    FOREIGN KEY (professor_id) REFERENCES professor_courses(professor_id)
);
```

### 3NF vs BCNF

```
3NF 조건:
- 모든 비주요 속성이 후보키에 완전 함수 종속
- 모든 비주요 속성이 후보키에 이행적 종속이 아님

BCNF 조건:
- 모든 결정자가 후보키

차이점:
- 3NF는 비주요 속성에 대해서만 규정
- BCNF는 모든 속성(주요/비주요)의 결정자 규정

실무에서:
- 대부분 3NF와 BCNF가 동일
- 복합키가 있고, 비키 속성이 키의 일부를 결정할 때 차이 발생
```

---

## 반정규화 (Denormalization)

### 반정규화란?

정규화된 테이블을 의도적으로 중복을 허용하여 성능을 향상시키는 기법

```
정규화 ←──────────────────────────────────────────────► 반정규화
      데이터 무결성 ↑                     조회 성능 ↑
      저장 공간 효율 ↑                    복잡한 조인 감소
      갱신 성능 ↑                         중복 데이터 관리 필요
```

### 반정규화 기법

#### 1. 테이블 통합

```sql
-- 정규화된 상태
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT
);

CREATE TABLE order_details (
    order_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10,2)
);

-- 반정규화: 자주 함께 조회되는 경우 통합
CREATE TABLE orders_denorm (
    order_id INT,
    customer_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)
);
```

#### 2. 컬럼 중복 추가

```sql
-- 정규화된 상태
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2)
);

-- 반정규화: 합계를 미리 계산하여 저장
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    total_amount DECIMAL(10,2),  -- 중복 저장 (계산된 값)
    item_count INT               -- 중복 저장 (계산된 값)
);

-- 동기화 필요 (트리거 또는 애플리케이션 레벨)
CREATE OR REPLACE FUNCTION update_order_totals()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders
    SET total_amount = (
        SELECT SUM(quantity * unit_price) FROM order_items WHERE order_id = NEW.order_id
    ),
    item_count = (
        SELECT COUNT(*) FROM order_items WHERE order_id = NEW.order_id
    )
    WHERE order_id = NEW.order_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

#### 3. 파생 테이블 생성

```sql
-- 집계 테이블 생성
CREATE TABLE daily_sales_summary (
    date DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(15,2),
    avg_order_value DECIMAL(10,2)
);

-- 정기적으로 집계 (배치 작업)
INSERT INTO daily_sales_summary
SELECT
    DATE(order_date),
    COUNT(*),
    SUM(total_amount),
    AVG(total_amount)
FROM orders
WHERE DATE(order_date) = CURRENT_DATE - 1
GROUP BY DATE(order_date)
ON CONFLICT (date) DO UPDATE SET
    total_orders = EXCLUDED.total_orders,
    total_revenue = EXCLUDED.total_revenue,
    avg_order_value = EXCLUDED.avg_order_value;
```

#### 4. 이력 테이블 분리

```sql
-- 변경이 잦은 현재 데이터와 이력 분리
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    current_tier VARCHAR(20),
    current_points INT,
    updated_at TIMESTAMP
);

CREATE TABLE user_tier_history (
    id SERIAL PRIMARY KEY,
    user_id INT,
    tier VARCHAR(20),
    points INT,
    changed_at TIMESTAMP
);
```

### 반정규화 시 고려사항

```sql
-- 반정규화 결정 체크리스트:

-- 1. 읽기/쓰기 비율 확인
SELECT
    relname,
    seq_tup_read,      -- 읽기
    n_tup_ins,         -- 삽입
    n_tup_upd,         -- 수정
    n_tup_del          -- 삭제
FROM pg_stat_user_tables;

-- 2. 조인 빈도 및 비용 확인
EXPLAIN ANALYZE
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- 3. 데이터 변경 빈도 확인
-- 자주 변경되는 데이터는 중복 시 동기화 비용 높음

-- 4. 무결성 관리 방안
-- 트리거, 저장 프로시저, 애플리케이션 레벨 관리 방법 결정
```

---

## 정규화 단계 요약

| 정규형 | 조건 | 제거 대상 |
|--------|------|-----------|
| 1NF | 원자값 | 다중값, 반복그룹 |
| 2NF | 1NF + 완전 함수 종속 | 부분 함수 종속 |
| 3NF | 2NF + 직접 종속 | 이행적 함수 종속 |
| BCNF | 모든 결정자가 후보키 | 비후보키 결정자 |

---

## 참고 자료

- Database System Concepts by Silberschatz, Korth, Sudarshan
- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/)
- Designing Data-Intensive Applications by Martin Kleppmann
- SQL Antipatterns by Bill Karwin

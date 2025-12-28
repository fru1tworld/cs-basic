# SQL 심화

## 목차
1. [SQL 명령어 분류](#sql-명령어-분류)
2. [DDL (Data Definition Language)](#ddl-data-definition-language)
3. [DML (Data Manipulation Language)](#dml-data-manipulation-language)
4. [DCL (Data Control Language)](#dcl-data-control-language)
5. [TCL (Transaction Control Language)](#tcl-transaction-control-language)
6. [조인(JOIN) 종류](#조인join-종류)
7. [서브쿼리(Subquery)](#서브쿼리subquery)
---

## SQL 명령어 분류

SQL 명령어는 기능에 따라 크게 4가지로 분류됩니다:

| 분류 | 설명 | 주요 명령어 |
|------|------|-------------|
| DDL | 데이터 구조 정의 | CREATE, ALTER, DROP, TRUNCATE |
| DML | 데이터 조작 | SELECT, INSERT, UPDATE, DELETE |
| DCL | 권한 제어 | GRANT, REVOKE |
| TCL | 트랜잭션 제어 | COMMIT, ROLLBACK, SAVEPOINT |

---

## DDL (Data Definition Language)

DDL은 데이터베이스 스키마를 정의하거나 수정하는 명령어입니다.

### CREATE - 객체 생성

```sql
-- 테이블 생성 (고급 옵션 포함)
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    username VARCHAR(50) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'banned')),

    -- 복합 인덱스
    CONSTRAINT idx_email_status UNIQUE (email, status)
);

-- 파티션 테이블 생성 (PostgreSQL)
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(15, 2),
    PRIMARY KEY (id, order_date)
) PARTITION BY RANGE (order_date);

-- 파티션 생성
CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- 인덱스 생성
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- Partial Index (조건부 인덱스)
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

### ALTER - 객체 수정

```sql
-- 컬럼 추가
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- 컬럼 타입 변경 (주의: 대용량 테이블에서 락 발생 가능)
ALTER TABLE users ALTER COLUMN phone TYPE VARCHAR(30);

-- 컬럼에 NOT NULL 제약조건 추가
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- 기본값 설정
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'pending';

-- 외래키 추가
ALTER TABLE orders
ADD CONSTRAINT fk_orders_user
FOREIGN KEY (user_id) REFERENCES users(id)
ON DELETE CASCADE ON UPDATE CASCADE;

-- 컬럼명 변경
ALTER TABLE users RENAME COLUMN phone TO phone_number;
```

### DROP vs TRUNCATE

```sql
-- DROP: 테이블 구조까지 완전 삭제
DROP TABLE IF EXISTS users CASCADE;

-- TRUNCATE: 데이터만 삭제 (DDL이므로 롤백 불가, 더 빠름)
TRUNCATE TABLE orders RESTART IDENTITY CASCADE;
```

> **TRUNCATE vs DELETE 차이점**
> - TRUNCATE: DDL, 롤백 불가(일부 DBMS), Where 절 불가, 매우 빠름
> - DELETE: DML, 롤백 가능, Where 절 가능, 상대적으로 느림

---

## DML (Data Manipulation Language)

### SELECT 심화

```sql
-- 윈도우 함수 활용
SELECT
    user_id,
    order_date,
    total_amount,
    SUM(total_amount) OVER (
        PARTITION BY user_id
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as running_total,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY order_date DESC) as order_rank,
    LAG(total_amount) OVER (PARTITION BY user_id ORDER BY order_date) as prev_amount,
    LEAD(total_amount) OVER (PARTITION BY user_id ORDER BY order_date) as next_amount
FROM orders;

-- CTE (Common Table Expression)
WITH monthly_sales AS (
    SELECT
        DATE_TRUNC('month', order_date) as month,
        SUM(total_amount) as total_sales,
        COUNT(*) as order_count
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY DATE_TRUNC('month', order_date)
),
sales_growth AS (
    SELECT
        month,
        total_sales,
        LAG(total_sales) OVER (ORDER BY month) as prev_month_sales,
        (total_sales - LAG(total_sales) OVER (ORDER BY month)) /
            NULLIF(LAG(total_sales) OVER (ORDER BY month), 0) * 100 as growth_rate
    FROM monthly_sales
)
SELECT * FROM sales_growth WHERE growth_rate > 10;

-- 재귀 CTE (조직도, 카테고리 트리 등)
WITH RECURSIVE category_tree AS (
    -- 기본 케이스
    SELECT id, name, parent_id, 1 as depth, ARRAY[name] as path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- 재귀 케이스
    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY path;
```

### INSERT 심화

```sql
-- UPSERT (INSERT ... ON CONFLICT)
INSERT INTO users (email, username, status)
VALUES ('user@example.com', 'newuser', 'active')
ON CONFLICT (email)
DO UPDATE SET
    username = EXCLUDED.username,
    updated_at = CURRENT_TIMESTAMP;

-- 다른 테이블에서 INSERT
INSERT INTO user_archive (id, email, archived_at)
SELECT id, email, CURRENT_TIMESTAMP
FROM users
WHERE status = 'inactive' AND updated_at < NOW() - INTERVAL '1 year';

-- RETURNING 절 활용
INSERT INTO orders (user_id, total_amount)
VALUES (1, 100.00)
RETURNING id, created_at;

-- Bulk INSERT with VALUES
INSERT INTO products (name, price, category_id) VALUES
    ('Product A', 10.00, 1),
    ('Product B', 20.00, 1),
    ('Product C', 30.00, 2);
```

### UPDATE 심화

```sql
-- 다른 테이블 조인하여 UPDATE
UPDATE orders o
SET status = 'shipped',
    shipped_at = CURRENT_TIMESTAMP
FROM users u
WHERE o.user_id = u.id
    AND u.status = 'premium'
    AND o.status = 'pending';

-- 서브쿼리를 이용한 UPDATE
UPDATE products
SET price = price * 1.1
WHERE category_id IN (
    SELECT id FROM categories WHERE name = 'Electronics'
);

-- CASE를 이용한 조건부 UPDATE
UPDATE users
SET tier = CASE
    WHEN total_purchases >= 10000 THEN 'platinum'
    WHEN total_purchases >= 5000 THEN 'gold'
    WHEN total_purchases >= 1000 THEN 'silver'
    ELSE 'bronze'
END;
```

### DELETE 심화

```sql
-- 서브쿼리를 이용한 DELETE
DELETE FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE status = 'banned'
);

-- JOIN을 이용한 DELETE (PostgreSQL)
DELETE FROM order_items oi
USING orders o
WHERE oi.order_id = o.id AND o.status = 'cancelled';

-- RETURNING과 함께 삭제된 행 반환
DELETE FROM users
WHERE last_login < NOW() - INTERVAL '2 years'
RETURNING id, email;
```

---

## DCL (Data Control Language)

### GRANT - 권한 부여

```sql
-- 특정 테이블에 대한 권한 부여
GRANT SELECT, INSERT, UPDATE ON users TO app_user;

-- 모든 테이블에 대한 읽기 권한
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

-- 시퀀스 사용 권한
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- 실행 권한 (함수, 프로시저)
GRANT EXECUTE ON FUNCTION calculate_discount(numeric) TO app_user;

-- 스키마 사용 권한
GRANT USAGE ON SCHEMA analytics TO data_analyst;

-- 역할(Role) 생성 및 권한 부여
CREATE ROLE api_service LOGIN PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE myapp TO api_service;
GRANT app_user TO api_service;
```

### REVOKE - 권한 회수

```sql
-- 특정 권한 회수
REVOKE INSERT, UPDATE ON users FROM app_user;

-- CASCADE로 연관 권한까지 회수
REVOKE ALL PRIVILEGES ON users FROM app_user CASCADE;

-- 스키마 권한 회수
REVOKE USAGE ON SCHEMA analytics FROM data_analyst;
```

### Row Level Security (PostgreSQL)

```sql
-- RLS 활성화
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- 정책 생성: 사용자는 자신의 주문만 볼 수 있음
CREATE POLICY user_orders_policy ON orders
    FOR ALL
    TO app_user
    USING (user_id = current_setting('app.current_user_id')::bigint);

-- 관리자는 모든 데이터 접근 가능
CREATE POLICY admin_all_access ON orders
    FOR ALL
    TO admin_role
    USING (true);
```

---

## TCL (Transaction Control Language)

### COMMIT과 ROLLBACK

```sql
-- 기본 트랜잭션
BEGIN;
    UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
    UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
COMMIT;

-- 에러 발생 시 롤백
BEGIN;
    UPDATE inventory SET quantity = quantity - 5 WHERE product_id = 100;
    INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 100, 5);
    -- 에러 발생
ROLLBACK;
```

### SAVEPOINT

```sql
BEGIN;
    INSERT INTO orders (user_id, total_amount) VALUES (1, 100);
    SAVEPOINT order_created;

    INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 100, 5);
    SAVEPOINT items_added;

    -- 결제 처리 중 에러 발생
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;

    -- items_added 시점으로 롤백 (주문과 주문항목은 유지)
    ROLLBACK TO SAVEPOINT items_added;

    -- 다른 결제 방식 시도
    INSERT INTO pending_payments (order_id, amount) VALUES (1, 100);
COMMIT;
```

### 트랜잭션 격리 수준 설정

```sql
-- 세션 레벨 설정
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 트랜잭션별 설정
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT * FROM accounts WHERE id = 1;
    -- 다른 트랜잭션의 커밋된 변경사항이 보이지 않음
COMMIT;

-- READ ONLY 트랜잭션 (최적화)
BEGIN TRANSACTION READ ONLY;
    SELECT * FROM large_report_view;
COMMIT;
```

---

## 조인(JOIN) 종류

### JOIN 종류 비교

```
┌─────────────┬─────────────┬─────────────┐
│   Table A   │    JOIN     │   Table B   │
├─────────────┼─────────────┼─────────────┤
│     A1      │   INNER     │     B1      │  → A1, B1 (매칭되는 것만)
│     A2      │   LEFT      │     B2      │  → A 전체 + 매칭 B
│     A3      │   RIGHT     │     B3      │  → 매칭 A + B 전체
│             │   FULL      │             │  → A 전체 + B 전체
│             │   CROSS     │             │  → A × B (카티션 곱)
└─────────────┴─────────────┴─────────────┘
```

### INNER JOIN

```sql
-- 양쪽 테이블에 모두 존재하는 데이터만 반환
SELECT u.id, u.username, o.id as order_id, o.total_amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id
WHERE o.order_date >= '2024-01-01';

-- 다중 테이블 INNER JOIN
SELECT
    u.username,
    o.id as order_id,
    p.name as product_name,
    oi.quantity,
    oi.price
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id;
```

### LEFT JOIN (LEFT OUTER JOIN)

```sql
-- 왼쪽 테이블의 모든 행 + 매칭되는 오른쪽 테이블 행
SELECT u.id, u.username, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;

-- 주문이 없는 사용자 찾기 (Anti-Join 패턴)
SELECT u.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
```

### RIGHT JOIN (RIGHT OUTER JOIN)

```sql
-- 오른쪽 테이블의 모든 행 + 매칭되는 왼쪽 테이블 행
SELECT p.name, c.name as category_name
FROM products p
RIGHT JOIN categories c ON p.category_id = c.id;
```

### FULL OUTER JOIN

```sql
-- 양쪽 테이블의 모든 행 (PostgreSQL)
SELECT
    COALESCE(e.name, 'No Employee') as employee,
    COALESCE(d.name, 'No Department') as department
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;
```

### CROSS JOIN

```sql
-- 카티션 곱 (모든 조합)
SELECT p.name as product, c.code as color
FROM products p
CROSS JOIN colors c;

-- 날짜 범위 생성에 활용
SELECT d.date, COALESCE(o.order_count, 0) as orders
FROM (
    SELECT generate_series('2024-01-01'::date, '2024-12-31'::date, '1 day'::interval)::date as date
) d
LEFT JOIN (
    SELECT order_date, COUNT(*) as order_count
    FROM orders
    GROUP BY order_date
) o ON d.date = o.order_date;
```

### SELF JOIN

```sql
-- 같은 테이블 간의 조인 (조직도, 상위-하위 관계)
SELECT
    e.name as employee,
    m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- 연속된 이벤트 분석
SELECT
    e1.user_id,
    e1.event_type as first_event,
    e2.event_type as second_event,
    e2.created_at - e1.created_at as time_diff
FROM events e1
JOIN events e2 ON e1.user_id = e2.user_id
    AND e2.created_at > e1.created_at
    AND e2.created_at < e1.created_at + INTERVAL '1 hour';
```

### LATERAL JOIN (PostgreSQL)

```sql
-- 각 사용자의 최근 3개 주문
SELECT u.id, u.username, recent_orders.*
FROM users u
CROSS JOIN LATERAL (
    SELECT o.id, o.total_amount, o.order_date
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.order_date DESC
    LIMIT 3
) recent_orders;
```

---

## 서브쿼리(Subquery)

### 스칼라 서브쿼리 (단일 값 반환)

```sql
-- SELECT 절에서 사용
SELECT
    p.name,
    p.price,
    (SELECT AVG(price) FROM products WHERE category_id = p.category_id) as category_avg_price,
    p.price - (SELECT AVG(price) FROM products WHERE category_id = p.category_id) as diff_from_avg
FROM products p;

-- WHERE 절에서 사용
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

### 테이블 서브쿼리 (여러 행 반환)

```sql
-- IN 연산자와 함께
SELECT * FROM users
WHERE id IN (
    SELECT DISTINCT user_id
    FROM orders
    WHERE total_amount > 1000
);

-- ANY/ALL 연산자
SELECT * FROM products
WHERE price > ALL (
    SELECT AVG(price)
    FROM products
    GROUP BY category_id
);

-- EXISTS 연산자 (존재 여부 확인, 성능 우수)
SELECT u.*
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
    AND o.total_amount > 500
);

-- NOT EXISTS (Anti-Join 대체)
SELECT u.*
FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
);
```

### 상관 서브쿼리 (Correlated Subquery)

```sql
-- 외부 쿼리의 값을 참조
SELECT
    o1.id,
    o1.total_amount,
    (SELECT COUNT(*)
     FROM orders o2
     WHERE o2.user_id = o1.user_id
     AND o2.order_date < o1.order_date) as previous_order_count
FROM orders o1;

-- 각 카테고리에서 가장 비싼 상품
SELECT p1.*
FROM products p1
WHERE p1.price = (
    SELECT MAX(p2.price)
    FROM products p2
    WHERE p2.category_id = p1.category_id
);
```

### FROM 절 서브쿼리 (인라인 뷰)

```sql
-- 집계 결과를 다시 집계
SELECT
    tier,
    COUNT(*) as user_count,
    AVG(total_orders) as avg_orders
FROM (
    SELECT
        user_id,
        COUNT(*) as total_orders,
        CASE
            WHEN COUNT(*) >= 10 THEN 'heavy'
            WHEN COUNT(*) >= 5 THEN 'medium'
            ELSE 'light'
        END as tier
    FROM orders
    GROUP BY user_id
) user_stats
GROUP BY tier;
```

### 서브쿼리 vs JOIN 성능 비교

```sql
-- 서브쿼리 방식 (상황에 따라 느릴 수 있음)
SELECT * FROM products
WHERE category_id IN (
    SELECT id FROM categories WHERE name LIKE 'Electronics%'
);

-- JOIN 방식 (일반적으로 더 효율적)
SELECT p.*
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE c.name LIKE 'Electronics%';

-- EXISTS 방식 (대용량에서 효율적)
SELECT p.*
FROM products p
WHERE EXISTS (
    SELECT 1 FROM categories c
    WHERE c.id = p.category_id
    AND c.name LIKE 'Electronics%'
);
```

---

## 참고 자료

- [PostgreSQL 공식 문서](https://www.postgresql.org/docs/current/)
- [MySQL 공식 문서](https://dev.mysql.com/doc/)
- SQL Performance Explained by Markus Winand
- Designing Data-Intensive Applications by Martin Kleppmann

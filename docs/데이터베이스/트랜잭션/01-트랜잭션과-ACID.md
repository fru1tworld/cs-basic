# 트랜잭션과 ACID

트랜잭션은 여러 SQL 문을 하나의 논리적 작업으로 묶는다. 계좌 이체를 예로 들면 A에서 돈을 빼는 작업과 B에 넣는 작업이 함께 성공하거나 함께 실패해야 한다.

```sql
-- A 계좌에서 B 계좌로 10만원 이체
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100000 WHERE id = 'B';
COMMIT;
```

두 UPDATE 중 하나만 반영되면 10만원이 사라지거나 늘어난다. 트랜잭션은 이 두 변경을 하나로 다뤄 부분적으로만 반영되는 일을 막는다.

## 트랜잭션이란?

- 트랜잭션은 데이터베이스에서 하나의 논리적 작업 단위를 구성하는 연산들의 집합임. "All or Nothing" - 모든 연산이 성공하거나, 모두 실패해야 함.

```sql
-- 트랜잭션 예시: 계좌 이체
BEGIN;
    -- 1. 출금 계좌에서 차감
    UPDATE accounts SET balance = balance - 10000 WHERE account_id = 'A';

    -- 2. 입금 계좌에 추가
    UPDATE accounts SET balance = balance + 10000 WHERE account_id = 'B';

    -- 모든 연산 성공 시
COMMIT;

-- 또는 문제 발생 시
ROLLBACK;
```

## 트랜잭션 상태

BEGIN으로 시작한 트랜잭션은 Active 상태에서 연산을 수행한다. 마지막 연산까지 마치면 Partially Committed 상태에서 커밋을 기다리고, 커밋이 성공하면 Committed가 된다.

반대로 오류가 발생해 더 진행할 수 없으면 Failed 상태로 들어간다. 이후 변경을 롤백하면 Aborted가 된다. 각 상태를 정리하면 다음과 같다.

| 상태 | 의미 |
| --- | --- |
| Active | 연산 실행 중 |
| Partially Committed | 마지막 연산을 마치고 커밋 대기 |
| Committed | 성공적으로 완료 |
| Failed | 오류로 진행 불가 |
| Aborted | 롤백 완료 |

```sql
-- 트랜잭션 예시
BEGIN;                           -- Active 상태
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;                          -- Committed 상태

-- 오류 시
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- 제약조건 위반 또는 오류 발생
ROLLBACK;                        -- Aborted 상태
```

## Atomicity (원자성)

- 트랜잭션의 모든 연산이 완전히 수행되거나, 전혀 수행되지 않아야 함.

```sql
-- 원자성 보장 예시
BEGIN;
    INSERT INTO orders (customer_id, total) VALUES (1, 50000);
    INSERT INTO order_items (order_id, product_id, qty) VALUES (LASTVAL(), 100, 2);
    INSERT INTO order_items (order_id, product_id, qty) VALUES (LASTVAL(), 101, 1);
    -- 재고 차감
    UPDATE products SET stock = stock - 2 WHERE product_id = 100;
    UPDATE products SET stock = stock - 1 WHERE product_id = 101;
COMMIT;  -- 모두 성공하거나 모두 취소

-- 원자성이 보장되지 않으면?
-- 주문은 생성되었는데 재고만 차감되지 않는 불일치 발생!
```

- 구현 메커니즘:
- Undo Log: 롤백을 위해 변경 전 데이터 저장
- Write-Ahead Logging (WAL): 변경 사항을 먼저 로그에 기록

- 트랜잭션의 모든 연산이 전부 실행되거나 전부 실행되지 않음
- 부분 실행 불가
- 실패 시 Rollback으로 원상복구

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100000 WHERE id = 'A';
-- 여기서 에러 발생 시, 위 UPDATE도 취소됨
UPDATE accounts SET balance = balance + 100000 WHERE id = 'B';
COMMIT;
```

## Consistency (일관성)

- 트랜잭션 실행 전후로 데이터베이스가 일관된 상태를 유지해야 함.

```sql
-- 일관성 제약조건 예시
CREATE TABLE accounts (
    account_id VARCHAR(20) PRIMARY KEY,
    balance DECIMAL(15, 2) CHECK (balance >= 0)  -- 잔액은 0 이상
);

-- 일관성 위반 시도
BEGIN;
    UPDATE accounts SET balance = balance - 100000 WHERE account_id = 'A';
    -- 잔액이 음수가 되면 CHECK 제약조건 위반
    -- 트랜잭션 자동 롤백
COMMIT;

-- 외래키 제약조건도 일관성의 일부
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

- 트랜잭션 전후로 데이터베이스의 무결성 제약조건 유지
- 예: 잔액은 0 이상이어야 함, 외래 키 참조 무결성

```sql
-- 잔액이 5만원인데 10만원 출금 시도
BEGIN;
UPDATE accounts SET balance = balance - 100000 WHERE id = 'A';
-- CHECK 제약 조건 위반 → 트랜잭션 실패
COMMIT;
```

## Isolation (격리성)

- 동시에 실행되는 트랜잭션들이 서로 영향을 주지 않아야 함.

```sql
-- 격리성 문제 예시
-- 트랜잭션 A: 계좌 잔액 조회 후 출금
-- 트랜잭션 B: 동시에 같은 계좌에서 출금

-- 격리성이 없다면:
-- A: SELECT balance FROM accounts WHERE id = 1;  -- 1000원
-- B: SELECT balance FROM accounts WHERE id = 1;  -- 1000원
-- A: UPDATE accounts SET balance = 500;          -- 500원 차감
-- B: UPDATE accounts SET balance = 200;          -- 800원 차감
-- 결과: 1300원이 차감되어야 하는데 800원만 차감됨!

-- 격리성 보장
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
    SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- 락 획득
    UPDATE accounts SET balance = balance - 500;
COMMIT;
```

- 동시에 실행되는 트랜잭션이 서로 영향을 주지 않음
- 트랜잭션은 마치 혼자 실행되는 것처럼 보여야 함
- 격리 수준(Isolation Level)으로 제어

## Durability (지속성)

- 커밋된 트랜잭션의 결과는 시스템 장애가 발생해도 영구적으로 유지됨.

```sql
-- 지속성 보장 메커니즘
-- 1. Write-Ahead Logging (WAL)
SHOW wal_level;  -- PostgreSQL

-- 2. Redo Log
-- MySQL InnoDB의 경우
SHOW VARIABLES LIKE 'innodb_flush_log_at_trx_commit';
-- 1: 매 트랜잭션마다 flush (가장 안전)
-- 2: 초당 1회 flush
-- 0: OS에 위임

-- 3. Checkpoint
-- 주기적으로 메모리의 변경사항을 디스크에 기록
```

- 성공한 트랜잭션의 결과는 영구적으로 보장
- 시스템 장애가 발생해도 데이터 유지
- 로그 파일(WAL)로 보장

## ACID 속성

- 트랜잭션이 보장해야 하는 4가지 속성임.

- ACID Properties
- A - Atomicity (원자성)
- "All or Nothing"
- 트랜잭션의 모든 연산이 완료되거나, 하나도 반영 안 됨
- 구현: Undo 로그, WAL
- C - Consistency (일관성)
- 트랜잭션 전후로 데이터베이스가 일관된 상태 유지
- 제약조건 (PK, FK, CHECK 등) 만족
- 구현: 제약조건 검사, 애플리케이션 로직
- I - Isolation (격리성)
- 동시 실행 트랜잭션이 서로 간섭하지 않음
- 각 트랜잭션이 혼자 실행되는 것처럼 보임
- 구현: 잠금, MVCC, 타임스탬프
- D - Durability (지속성)
- 커밋된 트랜잭션은 영구 저장
- 시스템 장애 후에도 복구 가능
- 구현: WAL, Force at Commit

## 참고 자료

- [PostgreSQL 공식 문서 - MVCC](https://www.postgresql.org/docs/current/mvcc-intro.html)
- [MySQL 공식 문서 - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [Vlad Mihalcea - MVCC](https://vladmihalcea.com/how-does-mvcc-multi-version-concurrency-control-work/)
- Designing Data-Intensive Applications by Martin Kleppmann
- High Performance MySQL, 3rd Edition

- [MySQL 공식 문서 - InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)
- [PostgreSQL 공식 문서 - Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)

- "Database System Concepts" (Silberschatz) - Chapter 17, 18
- "Transaction Processing: Concepts and Techniques" (Gray, Reuter)
- CMU 15-445: Concurrency Control
- PostgreSQL Documentation: Transaction Isolation

## 관련 학습

- [직렬화 가능성과 복구](02-직렬화-가능성과-복구.md)
- [격리 수준과 이상 현상](03-격리-수준과-이상-현상.md)
- [MVCC](04-MVCC.md)
- [잠금과 교착 상태](05-잠금과-교착-상태.md)

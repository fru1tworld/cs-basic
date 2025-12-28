# 트랜잭션과 ACID

## 목차
1. [트랜잭션 기초](#트랜잭션-기초)
2. [ACID 속성](#acid-속성)
3. [격리 수준 (Isolation Levels)](#격리-수준-isolation-levels)
4. [동시성 문제](#동시성-문제)
5. [MVCC (Multi-Version Concurrency Control)](#mvcc-multi-version-concurrency-control)
6. [락(Lock)과 동시성 제어](#락lock과-동시성-제어)
---

## 트랜잭션 기초

### 트랜잭션이란?

트랜잭션은 데이터베이스에서 하나의 논리적 작업 단위를 구성하는 연산들의 집합입니다. "All or Nothing" - 모든 연산이 성공하거나, 모두 실패해야 합니다.

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

### 트랜잭션 상태

```
┌─────────────────────────────────────────────────────────────────┐
│                     Transaction States                           │
│                                                                  │
│   ┌────────┐      ┌─────────┐      ┌───────────┐                │
│   │ Active │ ───► │Partially│ ───► │ Committed │                │
│   │        │      │Committed│      │           │                │
│   └────────┘      └─────────┘      └───────────┘                │
│        │                                                         │
│        │          ┌─────────┐      ┌───────────┐                │
│        └────────► │ Failed  │ ───► │  Aborted  │                │
│                   │         │      │           │                │
│                   └─────────┘      └───────────┘                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Active: 트랜잭션 실행 중
Partially Committed: 마지막 연산 완료, 커밋 대기
Committed: 성공적으로 완료
Failed: 오류 발생
Aborted: 롤백 완료
```

---

## ACID 속성

### Atomicity (원자성)

트랜잭션의 모든 연산이 완전히 수행되거나, 전혀 수행되지 않아야 합니다.

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

**구현 메커니즘:**
- **Undo Log**: 롤백을 위해 변경 전 데이터 저장
- **Write-Ahead Logging (WAL)**: 변경 사항을 먼저 로그에 기록

### Consistency (일관성)

트랜잭션 실행 전후로 데이터베이스가 일관된 상태를 유지해야 합니다.

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

### Isolation (격리성)

동시에 실행되는 트랜잭션들이 서로 영향을 주지 않아야 합니다.

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

### Durability (지속성)

커밋된 트랜잭션의 결과는 시스템 장애가 발생해도 영구적으로 유지됩니다.

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

---

## 격리 수준 (Isolation Levels)

### 네 가지 격리 수준

```
┌────────────────────────────────────────────────────────────────────┐
│           Isolation Levels (낮은 격리 → 높은 격리)                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  READ UNCOMMITTED  →  READ COMMITTED  →  REPEATABLE READ  →  SERIALIZABLE
│                                                                     │
│  [Dirty Read 가능]    [기본값: Oracle,    [기본값: MySQL    [완전 격리]
│                        PostgreSQL]        InnoDB]                   │
│                                                                     │
│  성능 ↑               균형               안정성 ↑          성능 ↓   │
│  일관성 ↓                                일관성 ↑          일관성 ↑  │
└────────────────────────────────────────────────────────────────────┘
```

### READ UNCOMMITTED

```sql
-- 다른 트랜잭션의 커밋되지 않은 데이터도 읽을 수 있음
-- Dirty Read 발생 가능

SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- Session 1:
BEGIN;
UPDATE products SET price = 1000 WHERE id = 1;  -- 아직 COMMIT 안 함

-- Session 2:
BEGIN;
SELECT price FROM products WHERE id = 1;  -- 1000 읽음 (Dirty Read)
-- Session 1이 ROLLBACK하면? → 잘못된 데이터를 읽은 것!
```

### READ COMMITTED

```sql
-- 커밋된 데이터만 읽을 수 있음
-- Dirty Read 방지, Non-Repeatable Read 발생 가능

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Session 1:
BEGIN;
SELECT price FROM products WHERE id = 1;  -- 500

-- Session 2:
UPDATE products SET price = 1000 WHERE id = 1;
COMMIT;

-- Session 1 (계속):
SELECT price FROM products WHERE id = 1;  -- 1000 (값이 변함!)
-- Non-Repeatable Read 발생
```

### REPEATABLE READ

```sql
-- 트랜잭션 내에서 같은 쿼리는 같은 결과를 보장
-- Non-Repeatable Read 방지, Phantom Read 발생 가능

SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Session 1:
BEGIN;
SELECT COUNT(*) FROM products WHERE price > 500;  -- 10개

-- Session 2:
INSERT INTO products (name, price) VALUES ('New', 1000);
COMMIT;

-- Session 1 (계속):
SELECT COUNT(*) FROM products WHERE price > 500;  -- 여전히 10개
-- 하지만 INSERT된 새 행은 못 볼 수도 있음 (DBMS에 따라 다름)
```

### SERIALIZABLE

```sql
-- 완전한 격리, 트랜잭션이 순차적으로 실행되는 것처럼 동작
-- 모든 이상현상 방지, 성능 가장 낮음

SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- PostgreSQL에서 직렬화 실패 시 재시도 필요
BEGIN;
    SELECT SUM(balance) FROM accounts;
    UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;
-- 다른 트랜잭션과 충돌 시:
-- ERROR: could not serialize access due to concurrent update
-- 애플리케이션에서 재시도 로직 필요
```

### 격리 수준별 문제 발생 여부

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|------------|---------------------|--------------|
| READ UNCOMMITTED | O | O | O |
| READ COMMITTED | X | O | O |
| REPEATABLE READ | X | X | O (DB에 따라 X) |
| SERIALIZABLE | X | X | X |

---

## 동시성 문제

### Dirty Read (더티 리드)

커밋되지 않은 데이터를 읽는 문제

```sql
-- T1: 아직 커밋 안 됨
UPDATE accounts SET balance = 0 WHERE id = 1;

-- T2: T1의 변경사항을 읽음
SELECT balance FROM accounts WHERE id = 1;  -- 0

-- T1: 롤백
ROLLBACK;

-- T2가 읽은 0은 실제로 존재하지 않는 데이터!
```

### Non-Repeatable Read (반복 불가능 읽기)

같은 쿼리가 다른 결과를 반환하는 문제

```sql
-- T1:
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 1000

-- T2:
UPDATE accounts SET balance = 500 WHERE id = 1;
COMMIT;

-- T1 (계속):
SELECT balance FROM accounts WHERE id = 1;  -- 500 (다른 결과!)
COMMIT;
```

### Phantom Read (팬텀 리드)

범위 조회 시 새로운 행이 나타나는 문제

```sql
-- T1:
BEGIN;
SELECT * FROM products WHERE price BETWEEN 100 AND 200;
-- 결과: 5행

-- T2:
INSERT INTO products (name, price) VALUES ('New', 150);
COMMIT;

-- T1 (계속):
SELECT * FROM products WHERE price BETWEEN 100 AND 200;
-- 결과: 6행 (팬텀 행 발생!)
COMMIT;
```

### Lost Update (갱신 분실)

두 트랜잭션이 동시에 같은 데이터를 수정할 때 발생

```sql
-- 초기 balance = 1000

-- T1과 T2가 거의 동시에:
-- T1: SELECT balance FROM accounts WHERE id = 1;  -- 1000
-- T2: SELECT balance FROM accounts WHERE id = 1;  -- 1000

-- T1: UPDATE accounts SET balance = 1000 + 100;   -- 1100
-- T2: UPDATE accounts SET balance = 1000 + 200;   -- 1200

-- 최종 결과: 1200 (T1의 갱신이 손실됨, 1300이어야 함)

-- 해결: SELECT FOR UPDATE
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- 락 획득
UPDATE accounts SET balance = balance + 100;
COMMIT;
```

---

## MVCC (Multi-Version Concurrency Control)

### MVCC 개념

MVCC는 데이터의 여러 버전을 유지하여 읽기와 쓰기가 서로 블로킹하지 않도록 합니다.

```
┌────────────────────────────────────────────────────────────────────┐
│                    MVCC 동작 방식                                   │
│                                                                     │
│   Time →                                                           │
│                                                                     │
│   T1(xid=100): ─────[UPDATE row]─────[COMMIT]────────              │
│                          │                                          │
│                          ▼                                          │
│              ┌─────────────────────────────────┐                   │
│              │ Row Versions:                    │                   │
│              │  v1: {data: 'old', xmin:50, xmax:100}               │
│              │  v2: {data: 'new', xmin:100, xmax:∞}                │
│              └─────────────────────────────────┘                   │
│                          │                                          │
│   T2(xid=101): ─────────[SELECT]─────────────────                  │
│                    (T1 커밋 전: v1 읽음)                            │
│                    (T1 커밋 후: v2 읽음) - 격리수준에 따라 다름      │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### PostgreSQL MVCC

```sql
-- 숨겨진 시스템 컬럼 조회
SELECT xmin, xmax, ctid, * FROM users LIMIT 5;

-- xmin: 이 버전을 생성한 트랜잭션 ID
-- xmax: 이 버전을 삭제/수정한 트랜잭션 ID (0이면 활성)
-- ctid: 물리적 위치 (page, offset)

-- 가시성 판단:
-- 1. xmin이 커밋되었는가?
-- 2. xmin이 현재 스냅샷보다 이전인가?
-- 3. xmax가 없거나 아직 커밋되지 않았는가?
```

### MySQL InnoDB MVCC

```sql
-- InnoDB는 Undo Log를 사용하여 MVCC 구현

-- Undo Log 조회
SELECT * FROM information_schema.INNODB_TRX;

-- Read View (스냅샷) 생성 시점:
-- READ COMMITTED: 각 쿼리마다
-- REPEATABLE READ: 트랜잭션 시작 시

-- Undo Log 정리
-- Purge Thread가 더 이상 필요 없는 버전 정리
```

### MVCC 장단점

**장점:**
- 읽기가 쓰기를 블로킹하지 않음
- 쓰기가 읽기를 블로킹하지 않음
- 높은 동시성 처리 가능

**단점:**
- 여러 버전 저장으로 저장공간 증가
- VACUUM (PostgreSQL) 또는 Purge (MySQL) 필요
- 트랜잭션 ID Wraparound 관리 필요

---

## 락(Lock)과 동시성 제어

### 락의 종류

```sql
-- 1. Shared Lock (공유 락, 읽기 락)
SELECT * FROM products WHERE id = 1 FOR SHARE;
-- 다른 트랜잭션도 읽기 가능, 쓰기 불가

-- 2. Exclusive Lock (배타 락, 쓰기 락)
SELECT * FROM products WHERE id = 1 FOR UPDATE;
-- 다른 트랜잭션 읽기/쓰기 모두 불가 (MVCC에서는 읽기 가능)

-- 3. Row-Level Lock vs Table-Level Lock
-- Row: 특정 행만 락
-- Table: 전체 테이블 락

LOCK TABLE products IN ACCESS EXCLUSIVE MODE;  -- 테이블 락
```

### 락 호환성 매트릭스

| 요청 \ 보유 | None | Shared | Exclusive |
|-------------|------|--------|-----------|
| Shared | O | O | X |
| Exclusive | O | X | X |

### 데드락 (Deadlock)

```sql
-- 데드락 시나리오
-- T1:
BEGIN;
UPDATE accounts SET balance = 100 WHERE id = 1;  -- A에 락 획득
-- 대기 중...

-- T2:
BEGIN;
UPDATE accounts SET balance = 200 WHERE id = 2;  -- B에 락 획득
UPDATE accounts SET balance = 300 WHERE id = 1;  -- A 대기 (T1이 보유)

-- T1 (계속):
UPDATE accounts SET balance = 400 WHERE id = 2;  -- B 대기 (T2가 보유)
-- 데드락 발생!
```

```sql
-- 데드락 방지 전략
-- 1. 일관된 락 순서
-- 항상 id 오름차순으로 락 획득
BEGIN;
UPDATE accounts SET balance = 100 WHERE id = 1;  -- 먼저 작은 id
UPDATE accounts SET balance = 200 WHERE id = 2;  -- 그 다음 큰 id
COMMIT;

-- 2. 타임아웃 설정
SET lock_timeout = '5s';

-- 3. 데드락 감지
-- PostgreSQL/MySQL은 자동으로 감지하고 하나의 트랜잭션을 롤백
```

### 락 모니터링

```sql
-- PostgreSQL 락 확인
SELECT
    l.locktype,
    l.relation::regclass,
    l.mode,
    l.granted,
    a.usename,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL;

-- MySQL 락 확인
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 블로킹 쿼리 찾기
SELECT
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query
FROM pg_stat_activity blocking
JOIN pg_locks bl ON blocking.pid = bl.pid
JOIN pg_locks wl ON bl.locktype = wl.locktype
    AND bl.relation = wl.relation
    AND bl.pid != wl.pid
JOIN pg_stat_activity blocked ON wl.pid = blocked.pid
WHERE NOT wl.granted;
```

---

## 참고 자료

- [PostgreSQL 공식 문서 - MVCC](https://www.postgresql.org/docs/current/mvcc-intro.html)
- [MySQL 공식 문서 - InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- [Vlad Mihalcea - MVCC](https://vladmihalcea.com/how-does-mvcc-multi-version-concurrency-control-work/)
- Designing Data-Intensive Applications by Martin Kleppmann
- High Performance MySQL, 3rd Edition

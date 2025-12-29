# 트랜잭션 (Transaction)

## 개요

트랜잭션은 데이터베이스의 **논리적 작업 단위**입니다. 하나의 트랜잭션은 여러 SQL 문으로 구성될 수 있으며, **모두 성공하거나 모두 실패**해야 합니다.

**핵심 예시**: 계좌 이체
```sql
-- A 계좌에서 B 계좌로 10만원 이체
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 100000 WHERE id = 'B';
COMMIT;
```

두 UPDATE 중 하나만 실행되면 데이터 불일치가 발생합니다. 트랜잭션은 이를 방지합니다.

## 핵심 개념

### 1. ACID 속성

트랜잭션이 보장해야 하는 4가지 속성입니다.

#### Atomicity (원자성)
- 트랜잭션의 모든 연산이 **전부 실행되거나 전부 실행되지 않음**
- 부분 실행 불가
- 실패 시 **Rollback**으로 원상복구

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100000 WHERE id = 'A';
-- 여기서 에러 발생 시, 위 UPDATE도 취소됨
UPDATE accounts SET balance = balance + 100000 WHERE id = 'B';
COMMIT;
```

#### Consistency (일관성)
- 트랜잭션 전후로 데이터베이스의 **무결성 제약조건** 유지
- 예: 잔액은 0 이상이어야 함, 외래 키 참조 무결성

```sql
-- 잔액이 5만원인데 10만원 출금 시도
BEGIN;
UPDATE accounts SET balance = balance - 100000 WHERE id = 'A';
-- CHECK 제약 조건 위반 → 트랜잭션 실패
COMMIT;
```

#### Isolation (격리성)
- 동시에 실행되는 트랜잭션이 **서로 영향을 주지 않음**
- 트랜잭션은 마치 혼자 실행되는 것처럼 보여야 함
- 격리 수준(Isolation Level)으로 제어

#### Durability (지속성)
- 성공한 트랜잭션의 결과는 **영구적으로 보장**
- 시스템 장애가 발생해도 데이터 유지
- 로그 파일(WAL)로 보장

### 2. 트랜잭션 상태

```
Active → Partially Committed → Committed
   ↓              ↓
Failed ← ← ← ← ←
   ↓
Aborted
```

| 상태 | 설명 |
|------|------|
| Active | 트랜잭션 실행 중 |
| Partially Committed | 마지막 연산 실행, COMMIT 대기 |
| Committed | 성공적으로 완료 |
| Failed | 오류 발생, 진행 불가 |
| Aborted | Rollback 완료 |

### 3. 격리 수준 (Isolation Level)

격리성을 얼마나 보장할지 결정합니다. 높을수록 안전하지만 성능 저하.

#### Read Uncommitted (레벨 0)
- 다른 트랜잭션의 **커밋되지 않은 데이터** 읽기 가능
- **Dirty Read** 발생
- 거의 사용하지 않음

```
T1: UPDATE accounts SET balance = 1000;  -- 아직 COMMIT 안 함
T2: SELECT balance FROM accounts;  -- 1000 읽음 (Dirty Read)
T1: ROLLBACK;  -- T2가 읽은 데이터는 실제로 존재하지 않음
```

#### Read Committed (레벨 1)
- **커밋된 데이터만** 읽기 가능
- Dirty Read 방지
- **Non-Repeatable Read** 발생 가능
- Oracle, PostgreSQL 기본값

```
T1: SELECT balance FROM accounts;  -- 1000
T2: UPDATE accounts SET balance = 2000; COMMIT;
T1: SELECT balance FROM accounts;  -- 2000 (같은 쿼리, 다른 결과)
```

#### Repeatable Read (레벨 2)
- 트랜잭션 내에서 **같은 쿼리는 같은 결과**
- Non-Repeatable Read 방지
- **Phantom Read** 발생 가능
- MySQL InnoDB 기본값

```
T1: SELECT * FROM accounts WHERE balance > 500;  -- 3건
T2: INSERT INTO accounts VALUES(...); COMMIT;
T1: SELECT * FROM accounts WHERE balance > 500;  -- 4건? (Phantom)
```

> MySQL InnoDB는 Gap Lock으로 Phantom Read도 방지

#### Serializable (레벨 3)
- 가장 높은 격리 수준
- 트랜잭션이 **순차적으로 실행되는 것처럼** 동작
- 모든 이상 현상 방지
- 동시성 매우 낮음

### 4. 이상 현상 정리

| 현상 | 설명 | 방지 레벨 |
|------|------|-----------|
| Dirty Read | 커밋 안 된 데이터 읽음 | Read Committed |
| Non-Repeatable Read | 같은 쿼리가 다른 결과 | Repeatable Read |
| Phantom Read | 없던 행이 새로 보임 | Serializable |

### 5. 동시성 제어

#### 낙관적 동시성 제어 (Optimistic)
- 충돌이 적다고 **가정**
- 커밋 시점에 충돌 검사
- 충돌 시 롤백 후 재시도
- 읽기 위주 워크로드에 적합

```sql
-- 버전 번호로 충돌 감지
SELECT id, balance, version FROM accounts WHERE id = 'A';
-- version = 1

UPDATE accounts
SET balance = 2000, version = version + 1
WHERE id = 'A' AND version = 1;
-- 영향받은 행이 0이면 다른 트랜잭션이 먼저 수정함
```

#### 비관적 동시성 제어 (Pessimistic)
- 충돌이 많다고 **가정**
- 데이터 접근 전에 **Lock** 획득
- 쓰기 위주 워크로드에 적합

```sql
-- 비관적 락 (SELECT FOR UPDATE)
SELECT * FROM accounts WHERE id = 'A' FOR UPDATE;
-- 다른 트랜잭션은 이 행에 접근 대기
UPDATE accounts SET balance = 2000 WHERE id = 'A';
COMMIT;
```

### 6. Lock 종류

#### Shared Lock (공유 락, S Lock)
- 읽기 락
- 여러 트랜잭션이 동시에 획득 가능
- 쓰기 불가

#### Exclusive Lock (배타 락, X Lock)
- 쓰기 락
- 한 트랜잭션만 획득 가능
- 다른 트랜잭션은 읽기/쓰기 모두 불가

| | S Lock 요청 | X Lock 요청 |
|---|---|---|
| **S Lock 보유** | 허용 | 대기 |
| **X Lock 보유** | 대기 | 대기 |

### 7. MVCC (Multi-Version Concurrency Control)

락 없이 동시성을 보장하는 방식입니다.

**원리**:
- 데이터를 수정할 때 이전 버전을 유지
- 읽기 요청은 **자신의 시작 시점 버전**을 읽음
- 쓰기만 락 필요

```
시간 →
T1 시작 (시점: 100)
                    T2 시작 (시점: 101)
                    T2: UPDATE row (버전 101 생성)
T1: SELECT row      T2: COMMIT
(버전 100 읽음)
```

**장점**: 읽기와 쓰기가 서로 블로킹하지 않음

## 실무 적용

### 트랜잭션 범위 최소화

```java
// Bad: 트랜잭션 범위가 너무 넓음
@Transactional
public void processOrder() {
    Order order = createOrder();
    sendEmail();  // 외부 API 호출 - 오래 걸림
    updateInventory();
}

// Good: 트랜잭션 범위 최소화
public void processOrder() {
    Order order = createOrderWithTx();  // 트랜잭션 O
    sendEmail();  // 트랜잭션 X
    updateInventoryWithTx();  // 별도 트랜잭션
}
```

### 적절한 격리 수준 선택

| 상황 | 권장 격리 수준 |
|------|----------------|
| 조회 위주 | Read Committed |
| 정합성 중요 | Repeatable Read |
| 금융 거래 | Serializable (또는 비관적 락) |

## 참고 자료

- [MySQL 공식 문서 - InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)
- [PostgreSQL 공식 문서 - Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)

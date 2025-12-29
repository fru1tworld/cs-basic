# 격리 수준 이상현상 (Isolation Anomalies)

## 개요

격리 수준이 낮을수록 다양한 **이상현상(Anomaly)**이 발생할 수 있다. Dirty Read, Non-Repeatable Read, Phantom Read는 SQL 표준에서 정의한 대표적인 이상현상이며, 실제로는 Write Skew, Read Skew, Lost Update 등 더 다양한 현상이 존재한다. 각 이상현상의 정확한 정의와 발생 조건을 이해해야 적절한 격리 수준을 선택할 수 있다.

## 핵심 개념

### 1. 이상현상 개요

```
┌─────────────────────────────────────────────────────────────┐
│              격리 수준과 이상현상                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 격리 수준      │ Dirty │ Non-Rep │ Phantom │ Write │ Lost  │
│                │ Read  │ Read    │ Read    │ Skew  │ Update│
│ ───────────────┼───────┼─────────┼─────────┼───────┼───────│
│ Read Uncommit  │  ○    │    ○    │    ○    │  ○    │   ○   │
│ Read Committed │  ✗    │    ○    │    ○    │  ○    │   ○   │
│ Repeatable Read│  ✗    │    ✗    │    ○    │  △    │   △   │
│ Serializable   │  ✗    │    ✗    │    ✗    │  ✗    │   ✗   │
│                                                             │
│ ○: 발생 가능, ✗: 방지, △: DBMS에 따라 다름                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. Dirty Read (오손 읽기)

```
┌─────────────────────────────────────────────────────────────┐
│                    Dirty Read                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 정의: 아직 커밋되지 않은 데이터를 읽는 현상                  │
│                                                             │
│ 시나리오:                                                    │
│                                                             │
│ T1                          T2                              │
│ ─────────────────────       ─────────────────────           │
│ BEGIN                                                       │
│ UPDATE accounts                                             │
│ SET balance = 900          BEGIN                           │
│ WHERE id = 1               SELECT balance                   │
│ (원래 1000)                FROM accounts                    │
│                            WHERE id = 1                     │
│                            → 900 읽음 (커밋 안 된 값)       │
│ ROLLBACK                                                    │
│ (balance = 1000 복구)      -- T2는 900으로 처리 중          │
│                            COMMIT                           │
│                                                             │
│ 문제: T2가 존재한 적 없는 데이터(900)를 기반으로 작업       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Dirty Read 발생 조건
-- T1: 변경 후 롤백
-- T2: READ UNCOMMITTED에서 변경된 값 읽음

-- 발생하는 격리 수준: READ UNCOMMITTED
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
```

### 3. Non-Repeatable Read (반복 불가능 읽기)

```
┌─────────────────────────────────────────────────────────────┐
│              Non-Repeatable Read (Fuzzy Read)                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 정의: 같은 쿼리를 두 번 실행했을 때 다른 결과가 나오는 현상 │
│       (다른 트랜잭션이 해당 행을 수정/삭제)                  │
│                                                             │
│ 시나리오:                                                    │
│                                                             │
│ T1                          T2                              │
│ ─────────────────────       ─────────────────────           │
│ BEGIN                       BEGIN                           │
│ SELECT balance                                              │
│ FROM accounts                                               │
│ WHERE id = 1                                                │
│ → 1000                      UPDATE accounts                 │
│                             SET balance = 900               │
│                             WHERE id = 1                    │
│                             COMMIT                          │
│ SELECT balance                                              │
│ FROM accounts                                               │
│ WHERE id = 1                                                │
│ → 900 (다른 결과!)                                          │
│ COMMIT                                                      │
│                                                             │
│ 문제: 같은 트랜잭션 내 같은 쿼리가 다른 결과                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Non-Repeatable Read 방지
-- REPEATABLE READ 이상에서 방지됨
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### 4. Phantom Read (유령 읽기)

```
┌─────────────────────────────────────────────────────────────┐
│                    Phantom Read                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 정의: 같은 범위 쿼리를 두 번 실행했을 때                    │
│       새로운 행이 나타나거나 사라지는 현상                   │
│                                                             │
│ 시나리오:                                                    │
│                                                             │
│ T1                          T2                              │
│ ─────────────────────       ─────────────────────           │
│ BEGIN                       BEGIN                           │
│ SELECT COUNT(*)                                             │
│ FROM accounts                                               │
│ WHERE balance > 500                                         │
│ → 10개                      INSERT INTO accounts            │
│                             VALUES (11, 700)                │
│                             COMMIT                          │
│ SELECT COUNT(*)                                             │
│ FROM accounts                                               │
│ WHERE balance > 500                                         │
│ → 11개 (유령 행 출현!)                                      │
│ COMMIT                                                      │
│                                                             │
│ 차이점: Non-Repeatable Read는 기존 행 변경                  │
│        Phantom Read는 새 행 삽입/삭제                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Phantom Read 방지
-- SERIALIZABLE에서만 완전 방지
-- MySQL InnoDB: REPEATABLE READ에서도 Next-Key Lock으로 방지
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

### 5. Lost Update (갱신 손실)

```
┌─────────────────────────────────────────────────────────────┐
│                    Lost Update                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 정의: 두 트랜잭션이 같은 데이터를 동시에 수정하여           │
│       하나의 수정이 덮어씌워지는 현상                        │
│                                                             │
│ 시나리오 (Read-Modify-Write):                               │
│                                                             │
│ T1                          T2                              │
│ ─────────────────────       ─────────────────────           │
│ BEGIN                       BEGIN                           │
│ SELECT balance                                              │
│ FROM accounts                                               │
│ WHERE id = 1                                                │
│ → 1000                      SELECT balance                  │
│                             FROM accounts                   │
│                             WHERE id = 1                    │
│                             → 1000                          │
│ -- balance - 100                                            │
│ UPDATE accounts                                             │
│ SET balance = 900                                           │
│ WHERE id = 1                -- balance + 200                │
│                             UPDATE accounts                 │
│                             SET balance = 1200              │
│                             WHERE id = 1                    │
│ COMMIT                      COMMIT                          │
│                                                             │
│ 결과: balance = 1200 (T1의 수정 손실!)                      │
│ 기대: 1000 - 100 + 200 = 1100                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Lost Update 방지 방법

-- 방법 1: 명시적 잠금
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- 방법 2: 원자적 갱신
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- 방법 3: 낙관적 잠금 (버전 번호)
UPDATE accounts
SET balance = 900, version = version + 1
WHERE id = 1 AND version = 5;
```

### 6. Write Skew

```
┌─────────────────────────────────────────────────────────────┐
│                    Write Skew                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 정의: 두 트랜잭션이 각각 다른 데이터를 읽고 수정하지만      │
│       전체 제약조건을 위반하게 되는 현상                     │
│                                                             │
│ 시나리오 (온콜 의사, 최소 1명은 대기해야 함):               │
│                                                             │
│ T1 (의사 Alice)              T2 (의사 Bob)                  │
│ ─────────────────────        ─────────────────────          │
│ BEGIN                        BEGIN                          │
│ SELECT COUNT(*)              SELECT COUNT(*)                │
│ FROM on_call                 FROM on_call                   │
│ WHERE on_duty = true         WHERE on_duty = true           │
│ → 2 (Alice, Bob 대기 중)     → 2 (Alice, Bob 대기 중)       │
│                                                             │
│ -- 2 > 1이니 휴가 가능       -- 2 > 1이니 휴가 가능         │
│ UPDATE doctors               UPDATE doctors                  │
│ SET on_duty = false          SET on_duty = false            │
│ WHERE name = 'Alice'         WHERE name = 'Bob'             │
│                                                             │
│ COMMIT                       COMMIT                          │
│                                                             │
│ 결과: 대기 의사 0명! (제약조건 위반)                        │
│                                                             │
│ 특징: 각 트랜잭션은 자신이 쓰지 않은 데이터를 읽음          │
│       → REPEATABLE READ로도 방지 안 됨 (읽은 행은 변경 없음)│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Write Skew 방지

-- 방법 1: SERIALIZABLE 격리 수준
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- 방법 2: 명시적 잠금 (읽는 행도 잠금)
SELECT COUNT(*) FROM on_call WHERE on_duty = true FOR UPDATE;

-- 방법 3: 물리화된 충돌 (별도 잠금 테이블)
SELECT * FROM on_call_lock FOR UPDATE;
-- 공유 자원에 대한 잠금
```

### 7. Read Skew

```
┌─────────────────────────────────────────────────────────────┐
│                    Read Skew                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 정의: 여러 데이터를 읽을 때 일관되지 않은 상태를 보는 현상  │
│                                                             │
│ 시나리오 (A→B 100원 이체):                                  │
│                                                             │
│ T1 (이체)                    T2 (잔액 합계 조회)            │
│ ─────────────────────        ─────────────────────          │
│                              BEGIN                          │
│                              SELECT balance FROM A          │
│                              → 500                          │
│ BEGIN                                                       │
│ UPDATE A SET balance = 400                                  │
│ UPDATE B SET balance = 600                                  │
│ COMMIT                                                      │
│                              SELECT balance FROM B          │
│                              → 600                          │
│                              -- 합계: 500 + 600 = 1100      │
│                              COMMIT                          │
│                                                             │
│ 문제: 실제 총액은 1000인데 1100으로 보임                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- Read Skew 방지: REPEATABLE READ 이상
-- 트랜잭션 시작 시점의 스냅샷 사용
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

### 8. 이상현상 요약 비교

```
┌─────────────────────────────────────────────────────────────┐
│              이상현상 발생 조건 비교                          │
├─────────────────┬───────────────────────────────────────────┤
│ 이상현상        │ 발생 조건                                  │
├─────────────────┼───────────────────────────────────────────┤
│ Dirty Read      │ 커밋 안 된 데이터 읽기                     │
├─────────────────┼───────────────────────────────────────────┤
│ Non-Repeatable  │ 다른 트랜잭션이 읽은 행을 수정/삭제        │
│ Read            │                                           │
├─────────────────┼───────────────────────────────────────────┤
│ Phantom Read    │ 다른 트랜잭션이 범위에 해당하는 행 삽입    │
├─────────────────┼───────────────────────────────────────────┤
│ Lost Update     │ Read-Modify-Write 패턴에서 덮어쓰기       │
├─────────────────┼───────────────────────────────────────────┤
│ Write Skew      │ 읽은 조건이 다른 트랜잭션 쓰기로 변경     │
├─────────────────┼───────────────────────────────────────────┤
│ Read Skew       │ 여러 데이터 읽는 중 일부가 변경됨         │
└─────────────────┴───────────────────────────────────────────┘
```

## 실무 적용

### 이상현상 탐지

```sql
-- 잠재적 Lost Update 패턴 탐지 (코드 리뷰)
-- 위험: SELECT 후 UPDATE
SELECT balance FROM accounts WHERE id = ?;
-- 애플리케이션에서 계산
UPDATE accounts SET balance = ? WHERE id = ?;

-- 안전: 원자적 갱신
UPDATE accounts SET balance = balance - 100 WHERE id = ?;
```

### 격리 수준 선택 가이드

```sql
-- 대부분의 경우: READ COMMITTED (기본값)
-- 장점: 좋은 동시성, Dirty Read 방지
-- 단점: Non-Repeatable/Phantom Read 가능

-- 일관된 읽기 필요: REPEATABLE READ
-- 보고서, 백업, 데이터 검증

-- 엄격한 일관성 필요: SERIALIZABLE
-- 금융 거래, 재고 관리 (성능 트레이드오프)
```

## 참고 자료

- Berenson et al., "A Critique of ANSI SQL Isolation Levels" (1995)
- "Designing Data-Intensive Applications" (Kleppmann) - Chapter 7
- PostgreSQL Documentation: Transaction Isolation
- MySQL Documentation: InnoDB Locking

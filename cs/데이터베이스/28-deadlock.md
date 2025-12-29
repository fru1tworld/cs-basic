# 데드락 (Deadlock)

## 개요

데드락은 두 개 이상의 트랜잭션이 **서로가 보유한 자원을 기다리며 영원히 대기**하는 상태다. 잠금 기반 동시성 제어에서 필연적으로 발생할 수 있으며, DBMS는 데드락을 탐지하고 해결하는 메커니즘을 제공한다. 데드락 발생 조건을 이해하고 예방 전략을 적용하는 것이 중요하다.

## 핵심 개념

### 1. 데드락의 조건

```
┌─────────────────────────────────────────────────────────────┐
│              데드락 필요충분조건 (Coffman Conditions)        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 네 가지 조건이 모두 만족되어야 데드락 발생:                 │
│                                                             │
│ 1. Mutual Exclusion (상호 배제)                             │
│    자원은 한 번에 하나의 트랜잭션만 사용 가능               │
│                                                             │
│ 2. Hold and Wait (점유 대기)                                │
│    자원을 보유한 채 다른 자원을 대기                        │
│                                                             │
│ 3. No Preemption (비선점)                                   │
│    보유 중인 자원을 강제로 빼앗을 수 없음                   │
│                                                             │
│ 4. Circular Wait (순환 대기)                                │
│    자원 대기 관계가 순환 형성                               │
│                                                             │
│ 하나라도 깨면 데드락 방지 가능                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2. 데드락 시나리오

```
┌─────────────────────────────────────────────────────────────┐
│              기본 데드락 시나리오                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ T1                          T2                              │
│ ─────────────────────       ─────────────────────           │
│ BEGIN                       BEGIN                           │
│ LOCK(A)                                                     │
│ -- A 잠금 획득              LOCK(B)                         │
│                             -- B 잠금 획득                   │
│ LOCK(B)                                                     │
│ -- B 대기 (T2 보유)         LOCK(A)                         │
│                             -- A 대기 (T1 보유)              │
│                                                             │
│      T1 ───대기───→ B                                       │
│       ↑              │                                      │
│       │              ↓                                      │
│      A ←───보유─── T2                                       │
│                                                             │
│ 결과: T1은 B를, T2는 A를 영원히 대기 → 데드락               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- 데드락 발생 예시

-- Session 1 (T1)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- id=1에 X-lock 획득

-- Session 2 (T2)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
-- id=2에 X-lock 획득
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
-- id=1 대기 (T1 보유)

-- Session 1 (T1) - 계속
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- id=2 대기 (T2 보유) → 데드락!
```

### 3. Wait-For Graph (대기 그래프)

```
┌─────────────────────────────────────────────────────────────┐
│              Wait-For Graph                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 노드: 트랜잭션                                               │
│ 간선: Ti → Tj (Ti가 Tj가 보유한 자원을 대기)               │
│                                                             │
│ 정상 상태:                     데드락 상태:                 │
│                                                             │
│ T1 → T2 → T3                   T1 → T2                      │
│ (사이클 없음)                  ↑     ↓                      │
│                                T4 ← T3                      │
│                                (사이클 있음!)               │
│                                                             │
│ 탐지: 주기적으로 그래프에서 사이클 검사                     │
│       DFS로 O(V + E) 복잡도                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4. 데드락 처리 전략

```
┌─────────────────────────────────────────────────────────────┐
│              데드락 처리 방법                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 1. 예방 (Prevention)                                        │
│    데드락 조건 중 하나를 원천적으로 제거                    │
│    - 잠금 순서 강제                                          │
│    - 모든 잠금을 한 번에 획득                                │
│    - 타임스탬프 기반 우선순위                                │
│                                                             │
│ 2. 회피 (Avoidance)                                         │
│    데드락 가능성을 동적으로 검사하고 회피                   │
│    - 은행원 알고리즘 (실용성 낮음)                          │
│    - Wait-Die, Wound-Wait                                   │
│                                                             │
│ 3. 탐지 및 복구 (Detection & Recovery)                      │
│    데드락 발생 후 탐지하고 희생자 선택                      │
│    - Wait-For Graph 사이클 탐지                             │
│    - 희생자 롤백                                             │
│    → 대부분의 DBMS가 사용하는 방법                          │
│                                                             │
│ 4. 타임아웃 (Timeout)                                       │
│    일정 시간 대기 후 자동 롤백                              │
│    - 간단하지만 정확도 낮음                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. Wait-Die / Wound-Wait

타임스탬프 기반 데드락 회피:

```
┌─────────────────────────────────────────────────────────────┐
│              Wait-Die / Wound-Wait                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 각 트랜잭션에 타임스탬프 부여 (작을수록 오래됨)             │
│                                                             │
│ Wait-Die (Non-preemptive):                                  │
│ Ti가 Tj가 보유한 자원을 요청할 때:                          │
│ - Ti < Tj (Ti가 오래됨): Ti가 대기 (wait)                   │
│ - Ti > Tj (Ti가 젊음): Ti가 죽음 (die, 롤백)               │
│ → 오래된 트랜잭션이 우선권                                  │
│                                                             │
│ Wound-Wait (Preemptive):                                    │
│ Ti가 Tj가 보유한 자원을 요청할 때:                          │
│ - Ti < Tj (Ti가 오래됨): Tj를 상처 (wound, 롤백)           │
│ - Ti > Tj (Ti가 젊음): Ti가 대기 (wait)                    │
│ → 오래된 트랜잭션이 젊은 트랜잭션을 밀어냄                  │
│                                                             │
│ 두 방식 모두 사이클 방지 → 데드락 없음                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6. 데드락 탐지 (PostgreSQL)

```
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL 데드락 탐지                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ 메커니즘:                                                    │
│ 1. 잠금 대기 시 deadlock_timeout(기본 1초) 대기            │
│ 2. 타임아웃 후 데드락 탐지 실행                             │
│ 3. Wait-For Graph에서 사이클 검사                           │
│ 4. 사이클 발견 시 하나의 트랜잭션 롤백                      │
│                                                             │
│ 희생자 선택:                                                 │
│ - 보통 탐지를 유발한 트랜잭션                               │
│ - 또는 가장 적은 작업을 한 트랜잭션                         │
│                                                             │
│ 설정:                                                        │
│ deadlock_timeout = 1s  -- 데드락 검사 전 대기 시간         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```sql
-- PostgreSQL: 데드락 발생 시 에러 메시지
-- ERROR:  deadlock detected
-- DETAIL:  Process 12345 waits for ShareLock on transaction 67890;
--          blocked by process 67890.
--          Process 67890 waits for ShareLock on transaction 12345;
--          blocked by process 12345.
-- HINT:  See server log for query details.

-- 현재 잠금 상태 확인
SELECT * FROM pg_locks WHERE NOT granted;

-- 대기 중인 쿼리 확인
SELECT
    blocked_activity.pid AS blocked_pid,
    blocked_activity.query AS blocked_query,
    blocking_activity.pid AS blocking_pid,
    blocking_activity.query AS blocking_query
FROM pg_stat_activity blocked_activity
JOIN pg_locks blocked_locks ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.database IS NOT DISTINCT FROM blocking_locks.database
    AND blocked_locks.relation IS NOT DISTINCT FROM blocking_locks.relation
JOIN pg_stat_activity blocking_activity ON blocking_locks.pid = blocking_activity.pid
WHERE NOT blocked_locks.granted AND blocking_locks.granted;
```

### 7. 데드락 탐지 (MySQL InnoDB)

```sql
-- MySQL: 최근 데드락 정보
SHOW ENGINE INNODB STATUS;

/*
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-01-15 10:30:45
*** (1) TRANSACTION:
TRANSACTION 12345, ACTIVE 5 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 100, OS thread handle 140...

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 10 page no 5 n bits 72 index PRIMARY

*** (2) TRANSACTION:
TRANSACTION 12346, ACTIVE 3 sec starting index read
...

*** WE ROLL BACK TRANSACTION (1)
*/

-- 데드락 관련 설정
-- innodb_lock_wait_timeout = 50  -- 잠금 대기 타임아웃 (초)
-- innodb_deadlock_detect = ON    -- 데드락 탐지 활성화
```

### 8. 데드락 예방 전략

```sql
-- 전략 1: 일관된 잠금 순서
-- 항상 id가 작은 것부터 잠금
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
SELECT * FROM accounts WHERE id = 2 FOR UPDATE;
-- ...
COMMIT;

-- 전략 2: 미리 모든 잠금 획득
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- 모든 필요한 자원을 한 번에 잠금
-- ...
COMMIT;

-- 전략 3: 낮은 격리 수준 사용
-- 가능하면 READ COMMITTED 사용
-- 필요한 경우에만 FOR UPDATE

-- 전략 4: 짧은 트랜잭션
-- 잠금 보유 시간 최소화
BEGIN;
-- 빠른 처리
COMMIT;

-- 전략 5: 낙관적 잠금 (잠금 회피)
UPDATE accounts
SET balance = 900, version = version + 1
WHERE id = 1 AND version = 5;
-- 충돌 시 재시도
```

## 실무 적용

### 데드락 모니터링

```sql
-- PostgreSQL: pg_stat_activity로 대기 확인
SELECT
    pid,
    now() - query_start AS duration,
    query,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';

-- MySQL: performance_schema
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

### 데드락 발생 시 애플리케이션 처리

```python
# Python 예시: 데드락 재시도 로직
import psycopg2
import time

def execute_with_retry(conn, sql, max_retries=3):
    for attempt in range(max_retries):
        try:
            with conn.cursor() as cur:
                cur.execute(sql)
                conn.commit()
                return
        except psycopg2.errors.DeadlockDetected:
            conn.rollback()
            if attempt < max_retries - 1:
                time.sleep(0.1 * (2 ** attempt))  # 지수 백오프
            else:
                raise
```

## 참고 자료

- "Database System Concepts" (Silberschatz) - Chapter 18
- PostgreSQL Documentation: Deadlocks
- MySQL Documentation: Deadlock Detection
- "Transaction Processing" (Gray, Reuter)

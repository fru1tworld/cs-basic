# 데드락 (Deadlock)

## 개요

데드락(교착 상태)은 두 개 이상의 프로세스가 서로 상대방이 가진 자원을 기다리며 **무한히 대기**하는 상태입니다.

```
Process A: Resource 1 보유 → Resource 2 대기
Process B: Resource 2 보유 → Resource 1 대기
          → 서로 영원히 대기
```

## 핵심 개념

### 1. 데드락 발생 조건 (Coffman Conditions)

데드락이 발생하려면 다음 **4가지 조건이 모두** 만족해야 합니다.

#### 1) 상호 배제 (Mutual Exclusion)
- 자원은 한 번에 하나의 프로세스만 사용 가능
- 다른 프로세스가 사용 중이면 대기

```
Resource R은 한 번에 하나의 프로세스만 사용 가능
P1이 R 사용 중 → P2는 R 대기
```

#### 2) 점유 대기 (Hold and Wait)
- 자원을 보유한 상태에서 다른 자원을 대기
- 이미 가진 자원을 놓지 않음

```
P1: R1 보유 + R2 대기
P1은 R1을 놓지 않고 R2를 기다림
```

#### 3) 비선점 (No Preemption)
- 다른 프로세스의 자원을 강제로 뺏을 수 없음
- 자원은 보유 프로세스가 자발적으로 해제

```
P2가 P1의 R1을 강제로 뺏을 수 없음
P1이 R1을 스스로 놓아야 함
```

#### 4) 순환 대기 (Circular Wait)
- 프로세스들이 순환 형태로 서로의 자원을 대기

```
P1 → R2 대기 (P2가 보유)
P2 → R3 대기 (P3가 보유)
P3 → R1 대기 (P1이 보유)
     순환 형성
```

### 2. 데드락 예시

#### 예시 1: 두 프로세스의 데드락

```
시간순서:
t1: P1이 R1 획득
t2: P2가 R2 획득
t3: P1이 R2 요청 → 대기 (P2가 보유)
t4: P2가 R1 요청 → 대기 (P1이 보유)
    → 데드락!
```

```
       P1                P2
       ↓                 ↓
    [R1 보유]         [R2 보유]
       ↓                 ↓
    R2 대기  ←───────→  R1 대기
              데드락
```

#### 예시 2: 코드로 보는 데드락

```java
// Thread 1
synchronized(lockA) {
    // lockA 획득
    Thread.sleep(100);
    synchronized(lockB) {  // lockB 대기
        // 작업
    }
}

// Thread 2
synchronized(lockB) {
    // lockB 획득
    Thread.sleep(100);
    synchronized(lockA) {  // lockA 대기 → 데드락!
        // 작업
    }
}
```

### 3. 자원 할당 그래프 (Resource Allocation Graph)

데드락 상태를 시각화하는 방법입니다.

```
● : 프로세스 (P)
■ : 자원 (R)
→ : 요청 (P → R)
← : 할당 (R → P)

데드락 상태:
    ┌─────────────┐
    ↓             │
  [P1] ← [R1]     │
    │             │
    │             │
    ↓             │
  [R2] → [P2] ────┘
```

**사이클 존재 = 데드락 가능성**
- 자원당 인스턴스 1개: 사이클 = 데드락
- 자원당 인스턴스 여러 개: 사이클이 있어도 데드락 아닐 수 있음

### 4. 데드락 처리 방법

#### 4.1 예방 (Prevention)

4가지 조건 중 **하나를 부정**하여 데드락을 원천 차단합니다.

| 조건 | 부정 방법 | 단점 |
|------|----------|------|
| 상호 배제 | 공유 가능하게 | 대부분 불가능 |
| 점유 대기 | 시작 전 모든 자원 요청 | 자원 낭비, 기아 |
| 비선점 | 선점 허용 | 롤백 비용, 일부 자원만 가능 |
| 순환 대기 | 자원에 순서 부여 | 구현 복잡 |

**순환 대기 부정 예시**:
```java
// 자원 순서: lockA < lockB
// 항상 낮은 번호 먼저 획득

// Thread 1
synchronized(lockA) {
    synchronized(lockB) {
        // 작업
    }
}

// Thread 2
synchronized(lockA) {  // lockB 대신 lockA 먼저
    synchronized(lockB) {
        // 작업
    }
}
```

#### 4.2 회피 (Avoidance)

데드락이 발생할 **가능성**이 있으면 자원 할당 거부합니다.

**은행원 알고리즘 (Banker's Algorithm)**:

자원 할당 시 **안전 상태**인지 검사합니다.

```
안전 상태: 모든 프로세스가 종료 가능한 순서가 존재
불안전 상태: 데드락 발생 가능

현재 상태:
- Available: [3, 3, 2]
- Max, Allocation, Need 매트릭스...

P1이 [1, 0, 2] 요청
→ 할당 후 안전 상태인지 검사
→ 안전하면 할당, 아니면 대기
```

**단점**:
- 최대 자원 요구량을 미리 알아야 함
- 오버헤드 큼
- 실제로 잘 사용 안 함

#### 4.3 탐지 (Detection)

데드락을 허용하고, 발생 시 **탐지**합니다.

**탐지 알고리즘**:
- 자원 할당 그래프에서 사이클 탐지
- 주기적으로 실행

```
대기 그래프 (Wait-for Graph):
P1 → P2 → P3 → P1
      사이클 발견 → 데드락!
```

#### 4.4 회복 (Recovery)

데드락 탐지 후 **복구**합니다.

**1. 프로세스 종료**
```
방법 1: 데드락 프로세스 모두 종료
방법 2: 사이클이 없어질 때까지 하나씩 종료

종료 순서 기준:
- 우선순위
- 실행 시간
- 사용 자원 양
```

**2. 자원 선점**
```
프로세스에서 자원을 강제로 뺏음
→ 해당 프로세스는 롤백
→ 기아 상태 주의 (같은 프로세스만 계속 희생)
```

### 5. 실무에서의 데드락

#### 5.1 데이터베이스 데드락

```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- T1: id=1 락
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- T1: id=2 대기 (T2가 보유)

-- Transaction 2
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;
-- T2: id=2 락
UPDATE accounts SET balance = balance + 50 WHERE id = 1;
-- T2: id=1 대기 (T1이 보유)
-- 데드락!
```

**해결**: 항상 같은 순서로 락 획득
```sql
-- 항상 id 오름차순으로 락
-- Transaction 1 & 2 모두
BEGIN;
UPDATE accounts SET balance = ... WHERE id = 1;  -- 먼저
UPDATE accounts SET balance = ... WHERE id = 2;  -- 나중
COMMIT;
```

#### 5.2 멀티스레드 프로그래밍

**Lock Ordering**:
```java
// 모든 스레드가 같은 순서로 락 획득
Object lock1 = new Object();
Object lock2 = new Object();

void method() {
    synchronized(lock1) {  // 항상 lock1 먼저
        synchronized(lock2) {
            // 작업
        }
    }
}
```

**Lock Timeout**:
```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // 작업
    } finally {
        lock.unlock();
    }
} else {
    // 타임아웃 - 데드락 회피
}
```

### 6. Livelock과 Starvation

#### Livelock
- 프로세스가 상태를 계속 변경하지만 진전이 없음
- 데드락과 달리 "활동"은 하지만 유용한 작업 없음

```
복도에서 두 사람이 서로 비키려다 같은 방향으로 계속 움직임
A: 왼쪽 → 오른쪽 → 왼쪽 → ...
B: 오른쪽 → 왼쪽 → 오른쪽 → ...
```

#### Starvation (기아)
- 특정 프로세스가 자원을 영원히 얻지 못함
- 우선순위가 낮은 프로세스가 계속 밀림

```
P1(높은 우선순위) → 자원 획득
P2(높은 우선순위) → 자원 획득
P3(낮은 우선순위) → 무한 대기...
```

## 참고 자료

- Operating System Concepts (공룡책) - Chapter 8: Deadlocks
- [MySQL 데드락 처리](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)

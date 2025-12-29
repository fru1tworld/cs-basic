# 데드락 처리 (Deadlock Handling)

## 개요

데드락을 처리하는 방법은 크게 4가지로 분류됩니다:

1. **예방 (Prevention)**: 데드락 발생 조건 중 하나를 원천적으로 차단
2. **회피 (Avoidance)**: 데드락이 발생할 가능성이 있으면 자원 할당을 거부
3. **탐지 및 복구 (Detection & Recovery)**: 데드락 발생 후 탐지하여 복구
4. **무시 (Ignorance)**: 데드락 발생 확률이 낮으면 무시 (Ostrich Algorithm)

```
데드락 처리 전략
┌──────────────────────────────────────────────────┐
│  Prevention  │  Avoidance  │  Detection  │  Ignore │
│  (예방)      │  (회피)     │  (탐지/복구)│  (무시) │
├──────────────┼─────────────┼─────────────┼─────────┤
│  사전 차단   │  안전 상태  │  사후 대응  │  방치   │
│  조건 무효화 │  유지       │  복구 필요  │         │
└──────────────┴─────────────┴─────────────┴─────────┘
      자원 활용도 ───────────────────────────→ 높음
      구현 복잡도 ←─────────────────────────── 높음
```

## 핵심 개념

### 1. 데드락 예방 (Deadlock Prevention)

데드락 4가지 필요조건 중 **최소 하나를 무효화**하여 데드락 발생을 원천 차단합니다.

#### 1.1 상호 배제 (Mutual Exclusion) 조건 제거

```
┌─────────────────────────────────────────┐
│ 방법: 자원을 공유 가능하게 만든다      │
│                                         │
│ 예: 읽기 전용 파일 → 여러 프로세스 공유 │
│     스풀러(Spooler) 사용               │
└─────────────────────────────────────────┘
```

**한계**: 대부분의 자원은 본질적으로 공유 불가 (프린터, 뮤텍스 등)

#### 1.2 점유 대기 (Hold and Wait) 조건 제거

**방법 1: 모든 자원을 한 번에 요청**

```java
// 나쁜 예: 점유 대기 발생
lock(A);
// ... 작업 ...
lock(B);  // A를 점유한 채로 B 대기

// 좋은 예: 모든 자원을 한 번에 요청
if (tryLockAll(A, B)) {
    // ... 작업 ...
    unlockAll(A, B);
}
```

**방법 2: 자원 요청 전 모든 자원 반환**

```java
void process() {
    lock(A);
    // A 사용
    unlock(A);  // 먼저 반환

    lock(B);    // 그 후에 요청
    // B 사용
    unlock(B);
}
```

**단점**:
- 자원 활용도 저하 (필요 없는 시점에도 점유)
- 기아(Starvation) 가능성
- 미래에 필요한 자원을 미리 알아야 함

#### 1.3 비선점 (No Preemption) 조건 제거

**방법: 자원을 선점 가능하게 만든다**

```
프로세스 P가 자원 R을 점유 중
           │
           ▼
프로세스 Q가 자원 R 요청
           │
           ▼
┌─────────────────────────────────┐
│ 옵션 1: P의 자원을 강제 해제   │
│ 옵션 2: P를 대기 상태로 전환   │
└─────────────────────────────────┘
```

```java
// 선점 가능한 락 구현
boolean acquired = lock.tryLock(timeout);
if (!acquired) {
    // 타임아웃 - 점유 자원 반환 후 재시도
    releaseAllResources();
    retry();
}
```

**적용 가능한 자원**:
- CPU (Context Switch로 선점)
- 메모리 (페이지 스왑)

**적용 어려운 자원**:
- 프린터, 테이프 드라이브 (중간 상태 저장 불가)

#### 1.4 순환 대기 (Circular Wait) 조건 제거

**방법: 자원에 순서를 부여하고, 정해진 순서대로만 요청**

```
자원 순서: R1 < R2 < R3 < R4

규칙: 항상 낮은 번호 → 높은 번호 순으로 요청

프로세스 A: R1 → R3 ✓
프로세스 B: R2 → R4 ✓
프로세스 C: R1 → R2 → R4 ✓

금지: R3 → R1 ✗ (역순 불가)
```

```java
// 락 순서 강제
class OrderedLocking {
    private static final Object lock1 = new Object();  // 순서 1
    private static final Object lock2 = new Object();  // 순서 2

    public void method1() {
        synchronized (lock1) {          // 항상 lock1 먼저
            synchronized (lock2) {
                // 작업
            }
        }
    }

    public void method2() {
        synchronized (lock1) {          // 여기서도 lock1 먼저
            synchronized (lock2) {
                // 작업
            }
        }
    }
}
```

**실무에서 많이 사용**: 구현이 간단하고 효과적

### 2. 데드락 회피 (Deadlock Avoidance)

시스템이 **안전 상태(Safe State)**를 유지하도록 자원 할당을 결정합니다.

#### 2.1 안전 상태 (Safe State)

```
안전 상태: 모든 프로세스가 정상 종료할 수 있는
          실행 순서(Safe Sequence)가 존재하는 상태

┌──────────────────────────────────────────┐
│          전체 상태 집합                  │
│  ┌────────────────────────────────────┐ │
│  │       안전 상태 (Safe State)       │ │
│  │  ┌──────────────────────────────┐ │ │
│  │  │                              │ │ │
│  │  └──────────────────────────────┘ │ │
│  └────────────────────────────────────┘ │
│         ↑                               │
│         │                               │
│  불안전 상태 (Unsafe State)             │
│  = 데드락 가능성 있음 (반드시는 아님)   │
└──────────────────────────────────────────┘
```

**안전 순서 (Safe Sequence) 예시**:

```
프로세스: P1, P2, P3
가용 자원: 12개

       최대 요구량    현재 할당    추가 필요
P1        10           5           5
P2         4           2           2
P3         9           2           7

가용 자원 = 12 - 5 - 2 - 2 = 3

안전 순서 찾기:
1. P2 실행 (필요: 2, 가용: 3) → 완료 후 가용: 3 + 2 = 5
2. P1 실행 (필요: 5, 가용: 5) → 완료 후 가용: 5 + 5 = 10
3. P3 실행 (필요: 7, 가용: 10) → 완료 후 가용: 10 + 2 = 12

안전 순서: <P2, P1, P3>
```

#### 2.2 은행원 알고리즘 (Banker's Algorithm)

Dijkstra가 제안한 데드락 회피 알고리즘입니다.

**자료 구조**:

```
n = 프로세스 개수
m = 자원 유형 개수

Available[m]     : 현재 사용 가능한 각 자원의 개수
Max[n][m]        : 각 프로세스가 최대로 필요한 자원 수
Allocation[n][m] : 각 프로세스에 현재 할당된 자원 수
Need[n][m]       : 각 프로세스가 추가로 필요한 자원 수
                   Need[i][j] = Max[i][j] - Allocation[i][j]
```

**안전성 검사 알고리즘**:

```python
def is_safe_state():
    work = Available.copy()
    finish = [False] * n

    while True:
        # 실행 가능한 프로세스 찾기
        found = False
        for i in range(n):
            if not finish[i] and all(Need[i][j] <= work[j] for j in range(m)):
                # 프로세스 i 실행 가능
                for j in range(m):
                    work[j] += Allocation[i][j]  # 자원 반환
                finish[i] = True
                found = True
                break

        if not found:
            break

    return all(finish)  # 모든 프로세스 완료 가능 여부
```

**자원 요청 알고리즘**:

```python
def request_resources(process_id, request):
    # 1. 요청이 Need보다 크면 오류
    if any(request[j] > Need[process_id][j] for j in range(m)):
        raise Exception("최대 요구량 초과")

    # 2. 요청이 Available보다 크면 대기
    if any(request[j] > Available[j] for j in range(m)):
        return False  # 대기

    # 3. 가상으로 할당해보기
    for j in range(m):
        Available[j] -= request[j]
        Allocation[process_id][j] += request[j]
        Need[process_id][j] -= request[j]

    # 4. 안전 상태인지 검사
    if is_safe_state():
        return True  # 할당 승인
    else:
        # 롤백
        for j in range(m):
            Available[j] += request[j]
            Allocation[process_id][j] -= request[j]
            Need[process_id][j] += request[j]
        return False  # 대기
```

**예제**:

```
5개 프로세스, 3개 자원 유형 (A, B, C)
Available: [3, 3, 2]

         Allocation    Max        Need
         A  B  C      A  B  C    A  B  C
P0       0  1  0      7  5  3    7  4  3
P1       2  0  0      3  2  2    1  2  2
P2       3  0  2      9  0  2    6  0  0
P3       2  1  1      2  2  2    0  1  1
P4       0  0  2      4  3  3    4  3  1

안전 순서: <P1, P3, P4, P2, P0>
```

**단점**:
- 최대 자원 요구량을 미리 알아야 함
- 프로세스 수와 자원 수가 고정되어야 함
- 오버헤드가 큼 (매 요청마다 안전성 검사)

### 3. 데드락 탐지 (Deadlock Detection)

데드락 발생을 허용하고, 발생 시 탐지하여 복구합니다.

#### 3.1 Wait-for 그래프

자원 유형당 인스턴스가 1개인 경우 사용합니다.

```
자원 할당 그래프           Wait-for 그래프
                          (자원 노드 제거)

P1 → R1 → P2              P1 → P2
↑         ↓               ↑     ↓
R2 ← P3 ← R3              └─ P3 ←┘

사이클 탐지 = 데드락 탐지
```

**알고리즘**: DFS로 사이클 탐지 - O(n²)

```python
def detect_deadlock_wait_for():
    visited = set()
    rec_stack = set()

    def dfs(node):
        visited.add(node)
        rec_stack.add(node)

        for neighbor in wait_for_graph[node]:
            if neighbor not in visited:
                if dfs(neighbor):
                    return True
            elif neighbor in rec_stack:
                return True  # 사이클 발견

        rec_stack.remove(node)
        return False

    for process in all_processes:
        if process not in visited:
            if dfs(process):
                return True  # 데드락 존재
    return False
```

#### 3.2 여러 인스턴스가 있는 경우

은행원 알고리즘과 유사한 방식을 사용합니다.

```python
def detect_deadlock():
    work = Available.copy()
    finish = [False] * n

    # 할당된 자원이 없는 프로세스는 종료 가능으로 표시
    for i in range(n):
        if all(Allocation[i][j] == 0 for j in range(m)):
            finish[i] = True

    while True:
        found = False
        for i in range(n):
            if not finish[i] and all(Request[i][j] <= work[j] for j in range(m)):
                for j in range(m):
                    work[j] += Allocation[i][j]
                finish[i] = True
                found = True

        if not found:
            break

    # finish[i]가 False인 프로세스들이 데드락 상태
    deadlocked = [i for i in range(n) if not finish[i]]
    return deadlocked
```

#### 3.3 탐지 주기

| 방식 | 장점 | 단점 |
|------|------|------|
| 자원 요청마다 | 즉시 탐지 | 오버헤드 큼 |
| 주기적 | 오버헤드 적음 | 탐지 지연 |
| CPU 사용률 감소 시 | 효율적 | 데드락 아닌 경우에도 실행 |

### 4. 데드락 복구 (Deadlock Recovery)

#### 4.1 프로세스 종료

**방법 1: 모든 데드락 프로세스 종료**
```
간단하지만 비용이 큼
진행 중인 작업 손실
```

**방법 2: 하나씩 종료**
```
종료 순서 결정 기준:
1. 프로세스 우선순위
2. 실행 시간 (오래 실행된 것 보존)
3. 남은 작업량
4. 사용 중인 자원 수
5. 종료에 필요한 프로세스 수
```

#### 4.2 자원 선점

```
1. 희생자 선택 (Victim Selection)
   - 비용이 최소인 프로세스 선택

2. 롤백 (Rollback)
   - 안전 상태로 롤백
   - 전체 롤백 또는 부분 롤백 (체크포인트)

3. 기아 방지
   - 같은 프로세스가 계속 희생자가 되지 않도록
   - 롤백 횟수를 비용에 포함
```

### 5. 데드락 무시 (Ostrich Algorithm)

```
"데드락? 그냥 무시하자" 🙈

이유:
- 데드락 발생 확률이 매우 낮음
- 예방/회피/탐지 비용이 큼
- 시스템 재시작으로 해결 가능

사용하는 시스템:
- Unix, Linux, Windows (대부분의 범용 OS)
- 데드락 발생 시 사용자가 프로세스 강제 종료
```

## 실무 적용

### 데드락 예방 실무 패턴

#### 1. 락 순서 통일 (가장 중요!)

```java
// 항상 id가 작은 계좌부터 락
void transfer(Account from, Account to, int amount) {
    Account first = from.id < to.id ? from : to;
    Account second = from.id < to.id ? to : from;

    synchronized (first) {
        synchronized (second) {
            from.withdraw(amount);
            to.deposit(amount);
        }
    }
}
```

#### 2. 타임아웃 설정

```java
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // 임계 구역
    } finally {
        lock.unlock();
    }
} else {
    // 타임아웃 - 재시도 또는 실패 처리
}
```

#### 3. 데이터베이스에서의 데드락 처리

```sql
-- MySQL: 데드락 탐지 자동 활성화
-- 한 트랜잭션을 롤백하여 해결

-- 데드락 정보 확인
SHOW ENGINE INNODB STATUS;

-- 락 타임아웃 설정
SET innodb_lock_wait_timeout = 50;
```

### 데드락 진단 도구

```bash
# Java 애플리케이션
jstack <pid> | grep -A 30 "deadlock"

# Linux 프로세스
cat /proc/<pid>/stack

# MySQL
SHOW PROCESSLIST;
SHOW ENGINE INNODB STATUS\G
```

## 참고 자료

- Operating System Concepts (Silberschatz) - Chapter 8: Deadlocks
- [OSTEP - Concurrency: Deadlock](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-deadlock.pdf)
- Banker's Algorithm 원논문 (Dijkstra, 1965)
- [MySQL Deadlock Detection](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlock-detection.html)

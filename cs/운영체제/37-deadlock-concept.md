# 데드락 (Deadlock) 개념

## 개요

데드락(교착 상태)은 둘 이상의 프로세스가 서로 상대방이 점유한 자원을 기다리면서 **무한히 대기**하는 상태입니다. 멀티프로세스/멀티스레드 환경에서 반드시 이해해야 할 핵심 개념입니다.

```
Process A          Process B
    │                  │
    ▼                  ▼
[Lock X 획득]      [Lock Y 획득]
    │                  │
    ▼                  ▼
[Lock Y 대기] ←────→ [Lock X 대기]
    │                  │
    └─── 무한 대기 ────┘
```

## 핵심 개념

### 1. 데드락의 4가지 필요조건 (Coffman Conditions)

데드락이 발생하려면 아래 **4가지 조건이 모두 동시에** 성립해야 합니다. 하나라도 성립하지 않으면 데드락은 발생하지 않습니다.

| 조건 | 영문명 | 설명 |
|------|--------|------|
| 상호 배제 | Mutual Exclusion | 자원은 한 번에 한 프로세스만 사용 가능 |
| 점유 대기 | Hold and Wait | 자원을 점유한 채로 다른 자원을 대기 |
| 비선점 | No Preemption | 다른 프로세스의 자원을 강제로 빼앗을 수 없음 |
| 순환 대기 | Circular Wait | 프로세스들이 순환 형태로 서로의 자원을 대기 |

#### 1.1 상호 배제 (Mutual Exclusion)

```
자원 R
┌─────────┐
│  Lock   │ ← 한 번에 한 프로세스만 획득 가능
└─────────┘
    │
    ▼
 Process A가 점유 → Process B는 대기
```

- 자원이 비공유 모드로만 사용 가능
- 예: 프린터, 뮤텍스, 파일 쓰기 락

#### 1.2 점유 대기 (Hold and Wait)

```java
// 데드락 가능한 코드
synchronized(lockA) {     // lockA 점유
    // 다른 작업 수행
    synchronized(lockB) { // lockB 대기 (lockA 점유한 채로)
        // ...
    }
}
```

- 프로세스가 자원을 점유한 상태에서 추가 자원을 요청
- 점유한 자원을 놓지 않고 계속 보유

#### 1.3 비선점 (No Preemption)

```
Process A: [자원 R 점유 중]
                │
                ▼
Process B: "자원 R 필요해!"
                │
                ▼
           강제로 빼앗을 수 없음 → 대기
```

- 이미 할당된 자원을 강제로 빼앗을 수 없음
- 자원 반환은 점유한 프로세스가 자발적으로만 가능

#### 1.4 순환 대기 (Circular Wait)

```
P0 → R1 → P1 → R2 → P2 → R0 → P0
│                              │
└──────────── 순환 ────────────┘

P0이 R1을 기다림 (P1이 점유)
P1이 R2를 기다림 (P2가 점유)
P2가 R0을 기다림 (P0이 점유)
```

- 프로세스 집합 {P0, P1, ..., Pn}에서
- P0은 P1이 점유한 자원을 대기
- P1은 P2가 점유한 자원을 대기
- ...
- Pn은 P0이 점유한 자원을 대기

### 2. 자원 할당 그래프 (Resource Allocation Graph)

데드락을 시각적으로 표현하고 탐지하는 도구입니다.

#### 그래프 구성 요소

```
○ : 프로세스 (Process)
□ : 자원 유형 (Resource Type)
● : 자원 인스턴스 (Resource Instance)

P → R : 요청 간선 (Request Edge) - P가 R을 요청
R → P : 할당 간선 (Assignment Edge) - R이 P에 할당됨
```

#### 데드락 상태 예시

```
        요청
    P1 ───→ R1
    ↑       │
할당 │       │ 할당
    │       ↓
    R2 ←─── P2
        요청

P1: R2 점유, R1 요청
P2: R1 점유, R2 요청
→ 순환(사이클) 발생 → 데드락
```

#### 사이클과 데드락 관계

| 상황 | 데드락 여부 |
|------|------------|
| 사이클 없음 | 데드락 없음 (반드시) |
| 사이클 있음 + 자원당 인스턴스 1개 | 데드락 (반드시) |
| 사이클 있음 + 자원당 인스턴스 여러 개 | 데드락 가능성 (반드시는 아님) |

#### 인스턴스가 여러 개인 경우

```
       R1 (인스턴스 2개)
      ┌──┬──┐
      │●│●│
      └──┴──┘
      ↙    ↘
    P1      P2
      ↘    ↙
      ┌──┬──┐
      │●│●│
      └──┴──┘
       R2 (인스턴스 2개)

사이클이 있어도 다른 인스턴스로 해결 가능
→ 데드락 아닐 수 있음
```

### 3. 데드락 예제 코드

#### Java에서의 데드락

```java
public class DeadlockExample {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void method1() {
        synchronized (lockA) {
            System.out.println("Thread 1: Holding lockA");
            try { Thread.sleep(100); } catch (InterruptedException e) {}

            System.out.println("Thread 1: Waiting for lockB");
            synchronized (lockB) {
                System.out.println("Thread 1: Holding lockA & lockB");
            }
        }
    }

    public void method2() {
        synchronized (lockB) {  // 순서가 다름!
            System.out.println("Thread 2: Holding lockB");
            try { Thread.sleep(100); } catch (InterruptedException e) {}

            System.out.println("Thread 2: Waiting for lockA");
            synchronized (lockA) {
                System.out.println("Thread 2: Holding lockB & lockA");
            }
        }
    }
}
```

#### 실행 흐름

```
Time    Thread 1              Thread 2
────────────────────────────────────────
t0      lockA 획득            lockB 획득
t1      lockB 대기            lockA 대기
t2      ...대기...            ...대기...
        └───── 데드락 발생 ─────┘
```

#### C/pthreads에서의 데드락

```c
pthread_mutex_t mutex1 = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex2 = PTHREAD_MUTEX_INITIALIZER;

void* thread1_func(void* arg) {
    pthread_mutex_lock(&mutex1);
    sleep(1);  // 데드락 발생 확률 높임
    pthread_mutex_lock(&mutex2);

    // 임계 구역

    pthread_mutex_unlock(&mutex2);
    pthread_mutex_unlock(&mutex1);
    return NULL;
}

void* thread2_func(void* arg) {
    pthread_mutex_lock(&mutex2);  // 순서가 다름!
    sleep(1);
    pthread_mutex_lock(&mutex1);

    // 임계 구역

    pthread_mutex_unlock(&mutex1);
    pthread_mutex_unlock(&mutex2);
    return NULL;
}
```

### 4. 데드락 vs 라이브락 vs 기아

| 상태 | 설명 | 특징 |
|------|------|------|
| 데드락 (Deadlock) | 서로 상대방의 자원을 기다리며 무한 대기 | 프로세스가 **정지** |
| 라이브락 (Livelock) | 서로 양보하며 무한 반복 | 프로세스가 **동작하지만** 진행 안 됨 |
| 기아 (Starvation) | 자원을 영원히 할당받지 못함 | 특정 프로세스만 **불이익** |

#### 라이브락 예시

```
복도에서 두 사람이 마주침

A: 오른쪽으로 비킴 ←→ B: 왼쪽으로 비킴 (여전히 마주침)
A: 왼쪽으로 비킴  ←→ B: 오른쪽으로 비킴 (여전히 마주침)
... 무한 반복 ...
```

```java
// 라이브락 예시 코드
while (resourceInUse) {
    // 양보
    Thread.yield();
    // 다시 시도 → 상대방도 같은 행동 → 무한 반복
}
```

#### 기아 예시

```
우선순위 스케줄링에서:

높은 우선순위 프로세스들이 계속 도착
    │
    ▼
낮은 우선순위 프로세스는 영원히 CPU 할당 못 받음
    │
    ▼
   기아 상태
```

### 5. 실제 데드락 사례

#### 5.1 데이터베이스 트랜잭션

```sql
-- Transaction A
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Row 1 락
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Row 2 대기

-- Transaction B (동시 실행)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- Row 2 락
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- Row 1 대기

-- 데드락 발생!
```

#### 5.2 파일 시스템

```
Process A: file1.txt 락 → file2.txt 요청
Process B: file2.txt 락 → file1.txt 요청
```

#### 5.3 분산 시스템

```
Node A: Resource X 락 → Node B의 Resource Y 요청
Node B: Resource Y 락 → Node A의 Resource X 요청
```

## 실무 적용

### 데드락이 발생하기 쉬운 상황

1. **다중 락 획득**: 여러 개의 락을 순서 없이 획득
2. **중첩 트랜잭션**: DB에서 여러 테이블을 다른 순서로 접근
3. **스레드 풀**: 스레드가 다른 작업 완료를 대기
4. **분산 잠금**: 여러 노드 간 자원 경쟁

### 데드락 탐지 방법

```java
// Java: ThreadMXBean으로 데드락 탐지
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] threadIds = bean.findDeadlockedThreads();

if (threadIds != null) {
    ThreadInfo[] infos = bean.getThreadInfo(threadIds);
    for (ThreadInfo info : infos) {
        System.out.println("Deadlocked thread: " + info.getThreadName());
    }
}
```

```bash
# Linux: 프로세스 상태 확인
ps aux | grep "D"  # D: uninterruptible sleep (IO 대기)

# Java: jstack으로 스레드 덤프
jstack <pid>
```

## 참고 자료

- Operating System Concepts (Silberschatz) - Chapter 8: Deadlocks
- [OSTEP - Concurrency: Deadlock](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-deadlock.pdf)
- Linux Kernel Development (Love) - Locking 관련 챕터

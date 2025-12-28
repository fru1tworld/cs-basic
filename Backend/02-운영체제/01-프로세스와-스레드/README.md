# 프로세스와 스레드 (Process and Thread)

> **OSTEP(Operating Systems: Three Easy Pieces)** 기반

---

## 목차

1. [프로세스 생명주기와 상태 전이](#1-프로세스-생명주기와-상태-전이)
2. [PCB (Process Control Block)](#2-pcb-process-control-block)
3. [스레드 모델](#3-스레드-모델)
4. [컨텍스트 스위칭 비용](#4-컨텍스트-스위칭-비용)
5. [스케줄링 알고리즘](#5-스케줄링-알고리즘)
7. [참고 자료](#7-참고-자료)

---

## 1. 프로세스 생명주기와 상태 전이

### 1.1 프로세스란?

프로세스는 **실행 중인 프로그램의 인스턴스**입니다. 단순한 프로그램 코드(정적)와 달리, 프로세스는 프로그램 카운터, 레지스터, 메모리 등 동적인 실행 상태를 포함합니다.

```
프로그램 (Program) → 디스크에 저장된 정적 코드
프로세스 (Process) → 메모리에 적재되어 실행 중인 동적 실체
```

### 1.2 프로세스 메모리 구조

```
+------------------+ 높은 주소
|      Stack       | ← 지역 변수, 함수 호출 정보
|        ↓         |
|                  |
|        ↑         |
|       Heap       | ← 동적 할당 메모리 (malloc, new)
+------------------+
|       BSS        | ← 초기화되지 않은 전역/정적 변수
+------------------+
|       Data       | ← 초기화된 전역/정적 변수
+------------------+
|       Text       | ← 실행 코드 (읽기 전용)
+------------------+ 낮은 주소
```

### 1.3 프로세스 상태 (Process States)

OSTEP에서 정의하는 프로세스의 5가지 상태:

```
                    +----------+
           fork()   |          |   exit()
     ┌──────────────►  Created  ├──────────────┐
     │              |          |              │
     │              +----┬-----+              │
     │                   │                    │
     │                   │ admitted           │
     │                   ▼                    │
     │              +----------+              │
     │              |          |              │
     │         ┌────►   Ready   ◄────┐        │
     │         │    |          |    │        │
     │         │    +----┬-----+    │        │
     │         │         │          │        ▼
     │    I/O  │         │ dispatch │   +-----------+
     │  완료   │         ▼          │   |           |
     │         │    +----------+    │   | Terminated|
     │         │    |          |    │   |           |
     │         └────+ Running  +────┘   +-----------+
     │              |          |  preempt
     │              +----┬-----+
     │                   │
     │                   │ I/O 또는
     │                   │ event wait
     │              +----▼-----+
     │              |          |
     └──────────────+ Waiting  |
                    | (Blocked)|
                    +----------+
```

#### 상태 설명

| 상태 | 설명 |
|------|------|
| **Created (New)** | 프로세스가 생성되었으나 아직 Ready 큐에 들어가지 않은 상태 |
| **Ready** | 실행 준비가 완료되어 CPU 할당을 기다리는 상태 |
| **Running** | CPU를 할당받아 실제로 실행 중인 상태 |
| **Waiting (Blocked)** | I/O 완료 또는 특정 이벤트를 기다리는 상태 |
| **Terminated** | 실행이 종료된 상태 |

### 1.4 상태 전이 트리거

```c
// C 예제: 프로세스 상태 전이 관찰
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();  // Created → Ready

    if (pid == 0) {
        // 자식 프로세스
        printf("Child: Running (PID=%d)\n", getpid());
        sleep(2);  // Running → Waiting (Blocked)
        printf("Child: Back to Running\n");
        exit(0);   // Running → Terminated
    } else {
        // 부모 프로세스
        printf("Parent: Child created with PID=%d\n", pid);
        wait(NULL);  // 부모도 Waiting 상태로 전이
        printf("Parent: Child terminated\n");
    }

    return 0;
}
```

---

## 2. PCB (Process Control Block)

### 2.1 PCB 개요

PCB는 **프로세스의 모든 정보를 저장하는 커널 데이터 구조**입니다. 운영체제가 프로세스를 관리하기 위한 핵심 자료구조입니다.

### 2.2 PCB 구조 (Linux task_struct 기반)

```c
// Linux 커널의 task_struct 간략화 버전
struct task_struct {
    // 프로세스 식별 정보
    pid_t pid;                    // 프로세스 ID
    pid_t tgid;                   // 스레드 그룹 ID

    // 프로세스 상태
    volatile long state;          // TASK_RUNNING, TASK_INTERRUPTIBLE 등

    // 스케줄링 정보
    int prio;                     // 동적 우선순위
    int static_prio;              // 정적 우선순위 (nice 값 기반)
    unsigned int policy;          // 스케줄링 정책 (SCHED_NORMAL, SCHED_FIFO 등)

    // CPU 컨텍스트
    struct thread_struct thread;  // CPU 레지스터 상태

    // 메모리 관리
    struct mm_struct *mm;         // 가상 메모리 공간 정보

    // 파일 시스템 정보
    struct files_struct *files;   // 열린 파일 디스크립터 테이블
    struct fs_struct *fs;         // 현재 작업 디렉토리, 루트 디렉토리

    // 신호 처리
    struct signal_struct *signal; // 시그널 핸들러 정보
    sigset_t blocked;             // 블록된 시그널 마스크

    // 프로세스 관계
    struct task_struct *parent;   // 부모 프로세스
    struct list_head children;    // 자식 프로세스 리스트

    // 시간 정보
    u64 utime, stime;            // 사용자/커널 모드 CPU 시간
    u64 start_time;              // 프로세스 시작 시간
};
```

### 2.3 PCB 정보 조회 (Python 예제)

```python
import os
import psutil

def get_process_info(pid=None):
    """프로세스의 PCB 정보 조회 (psutil 라이브러리 활용)"""
    if pid is None:
        pid = os.getpid()

    try:
        proc = psutil.Process(pid)

        info = {
            # 프로세스 식별
            'pid': proc.pid,
            'ppid': proc.ppid(),
            'name': proc.name(),

            # 상태 정보
            'status': proc.status(),

            # 스케줄링 정보
            'nice': proc.nice(),
            'cpu_affinity': proc.cpu_affinity() if hasattr(proc, 'cpu_affinity') else 'N/A',

            # 메모리 정보
            'memory_info': proc.memory_info()._asdict(),
            'memory_percent': proc.memory_percent(),

            # 파일 디스크립터
            'num_fds': proc.num_fds() if hasattr(proc, 'num_fds') else 'N/A',
            'open_files': len(proc.open_files()),

            # CPU 시간
            'cpu_times': proc.cpu_times()._asdict(),

            # 스레드 정보
            'num_threads': proc.num_threads(),

            # 시작 시간
            'create_time': proc.create_time(),
        }

        return info

    except psutil.NoSuchProcess:
        return None

if __name__ == "__main__":
    import json
    info = get_process_info()
    print(json.dumps(info, indent=2, default=str))
```

### 2.4 /proc 파일시스템을 통한 PCB 접근

```bash
# 프로세스 상태 확인
cat /proc/[PID]/status

# 메모리 맵 확인
cat /proc/[PID]/maps

# 열린 파일 디스크립터 확인
ls -la /proc/[PID]/fd

# CPU 사용 정보
cat /proc/[PID]/stat
```

---

## 3. 스레드 모델

### 3.1 스레드란?

스레드는 **프로세스 내에서 실행되는 흐름의 단위**입니다. 같은 프로세스의 스레드들은 코드, 데이터, 힙 영역을 공유하면서 각자의 스택과 레지스터를 가집니다.

```
프로세스 A
+------------------------------------------+
|  Code  |  Data  |  Heap  |              |
+------------------------------------------+
|  Thread 1  |  Thread 2  |  Thread 3    |
|  +-------+ |  +-------+ |  +-------+   |
|  | Stack | |  | Stack | |  | Stack |   |
|  | Regs  | |  | Regs  | |  | Regs  |   |
|  | PC    | |  | PC    | |  | PC    |   |
|  +-------+ |  +-------+ |  +-------+   |
+------------------------------------------+
```

### 3.2 User-Level Thread (ULT)

커널의 개입 없이 사용자 공간에서 관리되는 스레드입니다.

#### 특징

| 장점 | 단점 |
|------|------|
| 스레드 전환이 빠름 (커널 모드 전환 불필요) | 한 스레드가 블로킹되면 전체 프로세스 블록 |
| 커널 수정 없이 구현 가능 | 멀티코어 활용 불가 |
| 스케줄링 정책 커스터마이징 가능 | 시스템 콜 시 전체 프로세스 블록 |

#### 구현 예제: Fiber (경량 스레드)

```c
// User-level thread 구현 개념 (컨텍스트 전환)
#include <ucontext.h>
#include <stdio.h>
#include <stdlib.h>

#define STACK_SIZE 16384

ucontext_t main_ctx, thread1_ctx, thread2_ctx;
char stack1[STACK_SIZE], stack2[STACK_SIZE];

void thread1_func() {
    for (int i = 0; i < 3; i++) {
        printf("Thread 1: iteration %d\n", i);
        swapcontext(&thread1_ctx, &thread2_ctx);  // 수동 컨텍스트 스위칭
    }
}

void thread2_func() {
    for (int i = 0; i < 3; i++) {
        printf("Thread 2: iteration %d\n", i);
        swapcontext(&thread2_ctx, &thread1_ctx);
    }
}

int main() {
    // Thread 1 컨텍스트 설정
    getcontext(&thread1_ctx);
    thread1_ctx.uc_stack.ss_sp = stack1;
    thread1_ctx.uc_stack.ss_size = STACK_SIZE;
    thread1_ctx.uc_link = &main_ctx;
    makecontext(&thread1_ctx, thread1_func, 0);

    // Thread 2 컨텍스트 설정
    getcontext(&thread2_ctx);
    thread2_ctx.uc_stack.ss_sp = stack2;
    thread2_ctx.uc_stack.ss_size = STACK_SIZE;
    thread2_ctx.uc_link = &main_ctx;
    makecontext(&thread2_ctx, thread2_func, 0);

    // Thread 1 시작
    swapcontext(&main_ctx, &thread1_ctx);

    printf("Main: All threads completed\n");
    return 0;
}
```

### 3.3 Kernel-Level Thread (KLT)

커널이 직접 관리하는 스레드입니다.

#### 특징

| 장점 | 단점 |
|------|------|
| 멀티코어 병렬 실행 가능 | 스레드 전환 비용이 높음 (커널 모드 전환) |
| 한 스레드 블로킹 시 다른 스레드 실행 가능 | 커널 자원 소모 |
| 시스템 콜 효율적 처리 | 스케줄링 정책 제한적 |

#### POSIX 스레드 예제 (pthread)

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

#define NUM_THREADS 4

typedef struct {
    int thread_id;
    int iterations;
} thread_data_t;

void* worker(void* arg) {
    thread_data_t* data = (thread_data_t*)arg;

    for (int i = 0; i < data->iterations; i++) {
        printf("Thread %d: iteration %d (LWP: %lu)\n",
               data->thread_id, i, (unsigned long)pthread_self());
        usleep(100000);  // 100ms
    }

    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];
    thread_data_t thread_data[NUM_THREADS];

    // 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        thread_data[i].thread_id = i;
        thread_data[i].iterations = 3;

        int rc = pthread_create(&threads[i], NULL, worker, &thread_data[i]);
        if (rc) {
            fprintf(stderr, "Error: pthread_create returned %d\n", rc);
            exit(EXIT_FAILURE);
        }
    }

    // 스레드 종료 대기
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("All threads completed\n");
    return 0;
}
```

### 3.4 스레딩 모델 비교

| 모델 | 설명 | 예시 |
|------|------|------|
| **N:1 (Many-to-One)** | N개의 사용자 스레드가 1개의 커널 스레드에 매핑 | Green Threads (초기 Java) |
| **1:1 (One-to-One)** | 1개의 사용자 스레드가 1개의 커널 스레드에 매핑 | POSIX pthread, Windows Thread |
| **M:N (Many-to-Many)** | M개의 사용자 스레드가 N개의 커널 스레드에 매핑 | Go goroutine, Erlang process |

### 3.5 Python threading과 GIL

```python
import threading
import time
import os

def cpu_bound_task(name, count):
    """CPU 집약적 작업 (GIL 영향 받음)"""
    result = 0
    for i in range(count):
        result += i * i
    print(f"{name}: completed on thread {threading.current_thread().name}")
    return result

def io_bound_task(name, delay):
    """I/O 집약적 작업 (GIL 해제됨)"""
    print(f"{name}: starting I/O operation")
    time.sleep(delay)  # GIL 해제됨
    print(f"{name}: I/O completed")

# CPU-bound 작업은 GIL로 인해 병렬 실행 안됨
threads = []
start = time.time()

for i in range(4):
    t = threading.Thread(target=cpu_bound_task, args=(f"Task-{i}", 10000000))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

print(f"CPU-bound with threads: {time.time() - start:.2f}s")

# I/O-bound 작업은 병렬 실행 가능
threads = []
start = time.time()

for i in range(4):
    t = threading.Thread(target=io_bound_task, args=(f"IO-{i}", 1))
    threads.append(t)
    t.start()

for t in threads:
    t.join()

print(f"I/O-bound with threads: {time.time() - start:.2f}s")
```

---

## 4. 컨텍스트 스위칭 비용

### 4.1 컨텍스트 스위칭이란?

CPU가 한 프로세스/스레드에서 다른 프로세스/스레드로 전환하는 과정입니다.

```
Process A (Running)          Process B (Ready)
     │                            │
     │  1. 인터럽트 발생           │
     ▼                            │
[A의 상태 저장 (PCB)]             │
     │                            │
     │  2. 스케줄러 실행           │
     ▼                            │
[B 선택]                          │
     │                            │
     │  3. B의 상태 복원           │
     ▼                            ▼
Process A (Ready)            Process B (Running)
```

### 4.2 컨텍스트 스위칭 비용 분석

#### 직접 비용 (Direct Cost)

```
1. CPU 레지스터 저장/복원: ~수십 나노초
2. PCB 갱신: ~수십 나노초
3. 메모리 맵 전환: ~수백 나노초
4. TLB 플러시 (프로세스 전환 시): ~수 마이크로초
```

#### 간접 비용 (Indirect Cost)

```
1. CPU 캐시 미스: 수십~수백 마이크로초
2. TLB 미스: 수백 사이클
3. 파이프라인 플러시: 수십 사이클
4. 분기 예측 실패: 수십 사이클
```

### 4.3 벤치마크 결과 (2024 기준)

최신 Linux 커널 벤치마크 결과:

| 측정 조건 | 컨텍스트 스위칭 시간 |
|-----------|---------------------|
| 같은 코어, 핀드 상태 | 1.2 ~ 1.5 마이크로초 |
| 다른 코어로 마이그레이션 | ~2.2 마이크로초 |
| 큰 working set 포함 | 39 ~ 203 마이크로초 |
| Go goroutine 전환 | ~170 나노초 |

> **2024년 11월 Linux 커널 최적화**: Meta의 Rik van Riel이 `switch_mm_irqs_off` 함수 최적화 패치를 제출하여 CPU 시간의 17%를 차지하던 캐시 라인 경합 문제를 해결했습니다.

### 4.4 컨텍스트 스위칭 측정 코드

```c
// 컨텍스트 스위칭 시간 측정 (pipe 기반)
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sched.h>
#include <time.h>
#include <sys/wait.h>

#define ITERATIONS 100000

static inline long long get_nanos() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000LL + ts.tv_nsec;
}

int main() {
    int pipe1[2], pipe2[2];
    char buf;

    pipe(pipe1);
    pipe(pipe2);

    // CPU 0에 고정
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(0, &cpuset);
    sched_setaffinity(0, sizeof(cpuset), &cpuset);

    pid_t pid = fork();

    if (pid == 0) {
        // 자식 프로세스
        sched_setaffinity(0, sizeof(cpuset), &cpuset);

        for (int i = 0; i < ITERATIONS; i++) {
            read(pipe1[0], &buf, 1);
            write(pipe2[1], "x", 1);
        }
        exit(0);
    }

    // 부모 프로세스
    long long start = get_nanos();

    for (int i = 0; i < ITERATIONS; i++) {
        write(pipe1[1], "x", 1);
        read(pipe2[0], &buf, 1);
    }

    long long end = get_nanos();
    wait(NULL);

    // 2번의 컨텍스트 스위칭 (부모→자식, 자식→부모)
    double ns_per_switch = (double)(end - start) / (ITERATIONS * 2);

    printf("Context switch time: %.2f ns (%.2f us)\n",
           ns_per_switch, ns_per_switch / 1000);

    return 0;
}
```

### 4.5 컨텍스트 스위칭 최소화 전략

1. **스레드 풀 사용**: 스레드 생성/삭제 오버헤드 감소
2. **비동기 I/O 활용**: 블로킹으로 인한 스위칭 감소
3. **CPU 친화성 설정**: 캐시 효율성 향상
4. **경량 스레드 사용**: goroutine, 코루틴 활용

---

## 5. 스케줄링 알고리즘

### 5.1 스케줄링 기본 개념

```
스케줄링 목표:
├── CPU 활용률 (Utilization): CPU를 최대한 사용
├── 처리량 (Throughput): 단위 시간당 완료 작업 수
├── 반환 시간 (Turnaround Time): 작업 제출 ~ 완료
├── 대기 시간 (Waiting Time): Ready 큐에서 대기한 시간
└── 응답 시간 (Response Time): 작업 제출 ~ 첫 응답
```

### 5.2 FCFS (First-Come, First-Served)

가장 먼저 도착한 프로세스를 먼저 실행합니다.

```
예시: 프로세스 도착 (A: 24ms, B: 3ms, C: 3ms)

도착 순서: A → B → C

간트 차트:
|------A (24ms)------|--B (3ms)--|--C (3ms)--|
0                    24          27          30

대기 시간: A=0, B=24, C=27
평균 대기 시간: (0 + 24 + 27) / 3 = 17ms
```

#### 특징

- **장점**: 구현 간단, 기아(Starvation) 없음
- **단점**: Convoy Effect (긴 작업이 짧은 작업을 블록)
- **사용 사례**: 배치 처리 시스템

### 5.3 SJF (Shortest Job First)

실행 시간이 가장 짧은 프로세스를 먼저 실행합니다.

```
예시: 프로세스 도착 (A: 24ms, B: 3ms, C: 3ms)

SJF 순서: B → C → A

간트 차트:
|--B (3ms)--|--C (3ms)--|------A (24ms)------|
0           3           6                    30

대기 시간: A=6, B=0, C=3
평균 대기 시간: (6 + 0 + 3) / 3 = 3ms
```

#### SRTF (Shortest Remaining Time First) - 선점형 SJF

```python
import heapq
from dataclasses import dataclass, field
from typing import List

@dataclass(order=True)
class Process:
    remaining_time: int
    arrival_time: int = field(compare=False)
    pid: str = field(compare=False)
    original_burst: int = field(compare=False)

def srtf_scheduler(processes: List[tuple]) -> dict:
    """
    SRTF 스케줄링 시뮬레이션
    processes: [(pid, arrival_time, burst_time), ...]
    """
    # 우선순위 큐 (remaining_time 기준)
    ready_queue = []
    time = 0
    completed = []
    waiting_times = {}

    # 프로세스 복사 및 정렬
    remaining = [Process(bt, at, pid, bt)
                 for pid, at, bt in sorted(processes, key=lambda x: x[1])]

    while remaining or ready_queue:
        # 도착한 프로세스 큐에 추가
        while remaining and remaining[0].arrival_time <= time:
            heapq.heappush(ready_queue, remaining.pop(0))

        if not ready_queue:
            time = remaining[0].arrival_time if remaining else time
            continue

        # 가장 짧은 remaining time 프로세스 실행
        current = heapq.heappop(ready_queue)

        # 다음 이벤트까지 실행 (새 프로세스 도착 또는 완료)
        next_arrival = remaining[0].arrival_time if remaining else float('inf')
        run_time = min(current.remaining_time, next_arrival - time)

        time += run_time
        current.remaining_time -= run_time

        if current.remaining_time == 0:
            # 프로세스 완료
            turnaround = time - current.arrival_time
            waiting = turnaround - current.original_burst
            waiting_times[current.pid] = waiting
            completed.append(current.pid)
        else:
            heapq.heappush(ready_queue, current)

    return waiting_times

# 테스트
processes = [
    ("P1", 0, 8),
    ("P2", 1, 4),
    ("P3", 2, 9),
    ("P4", 3, 5),
]

result = srtf_scheduler(processes)
print("SRTF 결과:")
for pid, wt in result.items():
    print(f"  {pid}: 대기 시간 = {wt}ms")
print(f"  평균 대기 시간: {sum(result.values())/len(result):.2f}ms")
```

#### 특징

- **장점**: 평균 대기 시간 최소화 (최적)
- **단점**: 실행 시간 예측 어려움, 긴 작업 기아 가능
- **사용 사례**: 실행 시간 예측 가능한 배치 작업

### 5.4 RR (Round Robin)

각 프로세스에 고정된 시간 할당량(Time Quantum)을 부여합니다.

```
예시: 프로세스 (A: 24ms, B: 3ms, C: 3ms), Time Quantum = 4ms

간트 차트:
|A|B|C|A|A|A|A|A|
0 4 7 10 14 18 22 26 30

A: 0-4, 10-14, 14-18, 18-22, 22-26, 26-30 (총 24ms)
B: 4-7 (3ms, quantum 소진 전 완료)
C: 7-10 (3ms, quantum 소진 전 완료)
```

```python
from collections import deque
from dataclasses import dataclass

@dataclass
class RRProcess:
    pid: str
    arrival_time: int
    burst_time: int
    remaining_time: int = 0

    def __post_init__(self):
        self.remaining_time = self.burst_time

def round_robin(processes: list, quantum: int) -> dict:
    """
    Round Robin 스케줄링 시뮬레이션
    """
    queue = deque()
    time = 0
    results = {}
    gantt = []

    # 도착 시간순 정렬
    waiting = sorted([RRProcess(*p) for p in processes],
                     key=lambda x: x.arrival_time)

    while waiting or queue:
        # 도착한 프로세스 큐에 추가
        while waiting and waiting[0].arrival_time <= time:
            queue.append(waiting.pop(0))

        if not queue:
            time = waiting[0].arrival_time
            continue

        current = queue.popleft()

        # Time Quantum 또는 remaining time 중 작은 값만큼 실행
        run_time = min(quantum, current.remaining_time)
        gantt.append((current.pid, time, time + run_time))
        time += run_time
        current.remaining_time -= run_time

        # 실행 중 도착한 프로세스 추가
        while waiting and waiting[0].arrival_time <= time:
            queue.append(waiting.pop(0))

        if current.remaining_time > 0:
            queue.append(current)
        else:
            # 완료
            turnaround = time - current.arrival_time
            waiting_time = turnaround - current.burst_time
            results[current.pid] = {
                'turnaround': turnaround,
                'waiting': waiting_time,
                'completion': time
            }

    return results, gantt

# 테스트
processes = [("P1", 0, 24), ("P2", 0, 3), ("P3", 0, 3)]
results, gantt = round_robin(processes, quantum=4)

print("Round Robin (Quantum=4ms) 결과:")
print("\nGantt Chart:")
for pid, start, end in gantt:
    print(f"  [{start:2d}-{end:2d}] {pid}")

print("\n프로세스별 결과:")
for pid, data in results.items():
    print(f"  {pid}: 반환시간={data['turnaround']}ms, 대기시간={data['waiting']}ms")
```

#### Time Quantum 선택

- **너무 작으면**: 컨텍스트 스위칭 오버헤드 증가
- **너무 크면**: FCFS와 유사해짐
- **권장**: 80%의 CPU burst가 quantum 내에 완료되도록 설정

### 5.5 Priority Scheduling

우선순위가 높은 프로세스를 먼저 실행합니다.

```c
// Linux에서 프로세스 우선순위 설정
#include <sys/resource.h>
#include <unistd.h>

int main() {
    // nice 값 설정 (-20 ~ 19, 낮을수록 높은 우선순위)
    int ret = nice(-5);  // 우선순위 높임 (root 권한 필요)

    // 또는 setpriority 사용
    setpriority(PRIO_PROCESS, getpid(), -10);

    // 현재 우선순위 확인
    int prio = getpriority(PRIO_PROCESS, 0);
    printf("Current nice value: %d\n", prio);

    return 0;
}
```

#### Aging 기법 (기아 방지)

```python
class AgingPriorityScheduler:
    def __init__(self, aging_rate=1, aging_interval=10):
        self.aging_rate = aging_rate
        self.aging_interval = aging_interval

    def schedule(self, processes):
        """
        Aging이 적용된 우선순위 스케줄링
        processes: [(pid, priority, burst_time), ...]
        """
        queue = [{'pid': p[0], 'priority': p[1], 'burst': p[2], 'wait': 0}
                 for p in processes]
        time = 0
        results = []

        while queue:
            # 우선순위 기준 정렬 (낮은 값 = 높은 우선순위)
            queue.sort(key=lambda x: x['priority'])

            current = queue.pop(0)
            results.append((time, current['pid'], current['priority']))
            time += current['burst']

            # 대기 중인 프로세스 aging 적용
            for proc in queue:
                proc['wait'] += current['burst']
                # aging_interval마다 우선순위 증가 (값 감소)
                if proc['wait'] >= self.aging_interval:
                    proc['priority'] = max(0, proc['priority'] - self.aging_rate)
                    proc['wait'] = 0

        return results

# 테스트
scheduler = AgingPriorityScheduler(aging_rate=1, aging_interval=5)
processes = [
    ("P1", 3, 10),  # 낮은 우선순위
    ("P2", 1, 5),   # 높은 우선순위
    ("P3", 2, 8),
]
result = scheduler.schedule(processes)
print("Aging Priority Scheduling:")
for time, pid, prio in result:
    print(f"  Time {time}: {pid} (priority={prio})")
```

### 5.6 MLFQ (Multi-Level Feedback Queue)

**OSTEP에서 강조하는 가장 중요한 스케줄링 알고리즘**

여러 개의 Ready 큐를 우선순위별로 관리하며, 프로세스의 행동에 따라 동적으로 우선순위를 조정합니다.

```
+------------------+
| Queue 0 (최고)   | ← Time Quantum: 8ms
| RR 스케줄링      |
+------------------+
        ↓
+------------------+
| Queue 1          | ← Time Quantum: 16ms
| RR 스케줄링      |
+------------------+
        ↓
+------------------+
| Queue 2          | ← Time Quantum: 32ms
| RR 스케줄링      |
+------------------+
        ↓
+------------------+
| Queue 3 (최저)   | ← FCFS
| FCFS 스케줄링    |
+------------------+
```

#### MLFQ 규칙 (OSTEP)

1. **Rule 1**: Priority(A) > Priority(B) → A 실행
2. **Rule 2**: Priority(A) = Priority(B) → RR로 실행
3. **Rule 3**: 새 작업은 최고 우선순위 큐에 배치
4. **Rule 4**: Time Quantum 소진 시 우선순위 강등
5. **Rule 5**: 주기적으로 모든 작업을 최고 우선순위로 승격 (Boost)

```python
from collections import deque
from dataclasses import dataclass, field
from typing import List, Optional
import heapq

@dataclass
class MLFQProcess:
    pid: str
    arrival_time: int
    burst_time: int
    remaining_time: int = field(init=False)
    queue_level: int = 0
    time_allotment: int = 0

    def __post_init__(self):
        self.remaining_time = self.burst_time

class MLFQ:
    def __init__(self, num_queues: int = 4, base_quantum: int = 8, boost_interval: int = 100):
        self.num_queues = num_queues
        self.base_quantum = base_quantum
        self.boost_interval = boost_interval
        self.queues: List[deque] = [deque() for _ in range(num_queues)]
        self.time = 0
        self.last_boost = 0
        self.results = []

    def get_quantum(self, level: int) -> int:
        """레벨별 Time Quantum (2^level * base_quantum)"""
        return self.base_quantum * (2 ** level)

    def boost(self):
        """모든 프로세스를 최고 우선순위로 승격"""
        for level in range(1, self.num_queues):
            while self.queues[level]:
                proc = self.queues[level].popleft()
                proc.queue_level = 0
                proc.time_allotment = 0
                self.queues[0].append(proc)
        self.last_boost = self.time

    def get_next_process(self) -> Optional[MLFQProcess]:
        """가장 높은 우선순위 큐에서 프로세스 선택"""
        for level in range(self.num_queues):
            if self.queues[level]:
                return self.queues[level].popleft()
        return None

    def schedule(self, processes: List[tuple]) -> List[dict]:
        """
        MLFQ 스케줄링 실행
        processes: [(pid, arrival_time, burst_time), ...]
        """
        # 도착 시간순 정렬
        waiting = sorted([MLFQProcess(*p) for p in processes],
                        key=lambda x: x.arrival_time)

        gantt = []

        while waiting or any(q for q in self.queues):
            # Priority Boost 확인
            if self.time - self.last_boost >= self.boost_interval:
                self.boost()

            # 도착한 프로세스 추가
            while waiting and waiting[0].arrival_time <= self.time:
                new_proc = waiting.pop(0)
                self.queues[0].append(new_proc)

            current = self.get_next_process()

            if current is None:
                self.time = waiting[0].arrival_time if waiting else self.time
                continue

            # 현재 레벨의 quantum
            quantum = self.get_quantum(current.queue_level)

            # 실행 시간 계산
            run_time = min(quantum - current.time_allotment, current.remaining_time)

            gantt.append({
                'pid': current.pid,
                'start': self.time,
                'end': self.time + run_time,
                'queue': current.queue_level
            })

            self.time += run_time
            current.remaining_time -= run_time
            current.time_allotment += run_time

            # 도착한 프로세스 추가 (실행 중)
            while waiting and waiting[0].arrival_time <= self.time:
                new_proc = waiting.pop(0)
                self.queues[0].append(new_proc)

            if current.remaining_time == 0:
                # 완료
                self.results.append({
                    'pid': current.pid,
                    'completion': self.time,
                    'turnaround': self.time - processes[[p[0] for p in processes].index(current.pid)][1]
                })
            elif current.time_allotment >= quantum:
                # Time Quantum 소진 → 우선순위 강등
                if current.queue_level < self.num_queues - 1:
                    current.queue_level += 1
                current.time_allotment = 0
                self.queues[current.queue_level].append(current)
            else:
                # I/O 등으로 중단된 경우 같은 큐에 유지
                self.queues[current.queue_level].append(current)

        return gantt, self.results

# 테스트
print("=== MLFQ Scheduling Simulation ===\n")

mlfq = MLFQ(num_queues=3, base_quantum=8, boost_interval=50)
processes = [
    ("Interactive", 0, 6),   # 짧은 작업 (높은 우선순위 유지)
    ("CPU-bound", 0, 40),    # 긴 작업 (우선순위 강등)
    ("Mixed", 5, 20),        # 중간 작업
]

gantt, results = mlfq.schedule(processes)

print("Gantt Chart:")
for entry in gantt:
    print(f"  [{entry['start']:3d}-{entry['end']:3d}] {entry['pid']:12s} (Queue {entry['queue']})")

print("\nResults:")
for r in results:
    print(f"  {r['pid']}: 완료시간={r['completion']}ms, 반환시간={r['turnaround']}ms")
```

### 5.7 Linux CFS (Completely Fair Scheduler)

Linux 2.6.23부터 기본 스케줄러로 사용됩니다.

```
핵심 개념:
├── vruntime: 가상 실행 시간 (실제 실행 시간 / 가중치)
├── Red-Black Tree: O(log n) 삽입/삭제
├── nice 값 → 가중치 변환
└── Target Latency: 모든 태스크가 한 번씩 실행되는 주기
```

```c
// CFS의 핵심 로직 (개념적 구현)
struct cfs_rq {
    struct rb_root tasks_timeline;  // Red-Black Tree
    u64 min_vruntime;
    unsigned int nr_running;
};

// vruntime 계산
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se) {
    // delta_exec * NICE_0_WEIGHT / weight
    // nice 값이 높으면 (낮은 우선순위) vruntime이 빠르게 증가
    return delta * NICE_0_LOAD / se->load.weight;
}

// 다음 실행할 태스크 선택 (가장 작은 vruntime)
static struct task_struct *pick_next_task_fair(struct rq *rq) {
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);

    if (!left)
        return NULL;

    return rb_entry(left, struct sched_entity, run_node)->task;
}
```

---

## 7. 참고 자료

### 공식 문서 및 교재

- [OSTEP (Operating Systems: Three Easy Pieces)](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)
- [POSIX.1-2017 Specification](https://pubs.opengroup.org/onlinepubs/9699919799/)

### 관련 논문 및 블로그

- [Measuring Context Switching and Memory Overheads](https://eli.thegreenplace.net/2018/measuring-context-switching-and-memory-overheads-for-linux-threads/)
- [Context Switch Benchmark by tsuna](https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)
- [Linux 2024 Context Switch Optimizations](https://www.phoronix.com/news/Linux-2024-Optimize-Ctx-Switch)

### 도구

- [tsuna/contextswitch - GitHub](https://github.com/tsuna/contextswitch): 컨텍스트 스위칭 벤치마크 도구

---

> **문서 작성일**: 2024년 12월
>
> **참고**: 이 문서는 OSTEP, Linux 커널 문서, POSIX 표준을 기반으로 작성되었습니다.

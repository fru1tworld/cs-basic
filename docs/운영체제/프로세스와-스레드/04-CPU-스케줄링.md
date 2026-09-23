# CPU 스케줄링

- Linux CFS 설명은 기존 공정 스케줄러의 원리를 다룸. Linux 6.6부터 공정 스케줄링에 EEVDF를 도입하는 전환이 시작되었으므로 실제 커널 버전의 구현을 함께 확인해야 함. [Linux EEVDF 문서](https://docs.kernel.org/scheduler/sched-eevdf.html)

## 스케줄링 알고리즘

### 스케줄링 기본 개념

- 스케줄링 목표:
- CPU 활용률 (Utilization): CPU를 최대한 사용
- 처리량 (Throughput): 단위 시간당 완료 작업 수
- 반환 시간 (Turnaround Time): 작업 제출 ~ 완료
- 대기 시간 (Waiting Time): Ready 큐에서 대기한 시간
- 응답 시간 (Response Time): 작업 제출 ~ 첫 응답

### FCFS (First-Come, First-Served)

FCFS는 먼저 도착한 프로세스를 먼저 실행한다. 실행 시간이 A 24ms, B 3ms, C 3ms이고 도착 순서가 A → B → C인 예를 보자. A가 0~24ms, B가 24~27ms, C가 27~30ms에 실행되므로 대기 시간은 각각 0, 24, 27ms다. 평균 대기 시간은 `(0 + 24 + 27) / 3 = 17ms`가 된다.

#### 특징

- 장점: 구현 간단, 기아(Starvation) 없음
- 단점: Convoy Effect (긴 작업이 짧은 작업을 블록)
- 사용 사례: 배치 처리 시스템

### SJF (Shortest Job First)

- 실행 시간이 가장 짧은 프로세스를 먼저 실행함.

- 예시: 프로세스 도착 (A: 24ms, B: 3ms, C: 3ms)
- SJF 순서: B → C → A
- 간트 차트:
- --B (3ms)-- --C (3ms)--, A (24ms)
- 0, 3, 6, 30
- 대기 시간: A=6, B=0, C=3
- 평균 대기 시간: (6 + 0 + 3) / 3 = 3ms

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

- 장점: 평균 대기 시간 최소화 (최적)
- 단점: 실행 시간 예측 어려움, 긴 작업 기아 가능
- 사용 사례: 실행 시간 예측 가능한 배치 작업

### RR (Round Robin)

- 각 프로세스에 고정된 시간 할당량(Time Quantum)을 부여함.

- 예시: 프로세스 (A: 24ms, B: 3ms, C: 3ms), Time Quantum = 4ms
- 간트 차트:
- A B C A A A A A
- 0 4 7 10 14 18 22 26 30
- A: 0-4, 10-14, 14-18, 18-22, 22-26, 26-30 (총 24ms)
- B: 4-7 (3ms, quantum 소진 전 완료)
- C: 7-10 (3ms, quantum 소진 전 완료)

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

- 너무 작으면: 문맥 교환 오버헤드 증가
- 너무 크면: FCFS와 유사해짐
- 권장: 80%의 CPU burst가 quantum 내에 완료되도록 설정

### Priority Scheduling

- 우선순위가 높은 프로세스를 먼저 실행함.

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

### MLFQ (Multi-Level Feedback Queue)

- OSTEP에서 강조하는 가장 중요한 스케줄링 알고리즘

MLFQ는 우선순위별로 여러 Ready 큐를 두고 프로세스의 행동에 따라 큐 사이를 이동시킨다. 다음 예에서는 높은 우선순위에 짧은 Time Quantum을 주고, 낮은 큐에서는 더 오래 실행하도록 구성한다.

| 큐 | 우선순위 | 스케줄링 | Time Quantum |
| --- | --- | --- | --- |
| Queue 0 | 최고 | RR | 8ms |
| Queue 1 | 중간 | RR | 16ms |
| Queue 2 | 중간 | RR | 32ms |
| Queue 3 | 최저 | FCFS | FCFS로 실행 |

#### MLFQ 규칙 (OSTEP)

- Rule 1: Priority(A) > Priority(B) → A 실행
- Rule 2: Priority(A) = Priority(B) → RR로 실행
- Rule 3: 새 작업은 최고 우선순위 큐에 배치
- Rule 4: Time Quantum 소진 시 우선순위 강등
- Rule 5: 주기적으로 모든 작업을 최고 우선순위로 승격 (Boost)

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

### Linux CFS (Completely Fair Scheduler)

- Linux 2.6.23부터 기본 스케줄러로 사용됨.

- 핵심 개념:
- vruntime: 가상 실행 시간 (실제 실행 시간 / 가중치)
- Red-Black Tree: O(log n) 삽입/삭제
- nice 값 → 가중치 변환
- Target Latency: 모든 태스크가 한 번씩 실행되는 주기

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

## 참고 자료

- [OSTEP (Operating Systems: Three Easy Pieces)](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)
- [POSIX.1-2017 Specification](https://pubs.opengroup.org/onlinepubs/9699919799/)

- [Measuring Context Switching and Memory Overheads](https://eli.thegreenplace.net/2018/measuring-context-switching-and-memory-overheads-for-linux-threads/)
- [Context Switch Benchmark by tsuna](https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)
- [Linux 2024 Context Switch Optimizations](https://www.phoronix.com/news/Linux-2024-Optimize-Ctx-Switch)

- [tsuna/contextswitch - GitHub](https://github.com/tsuna/contextswitch): 문맥 교환 벤치마크 도구

- 문서 작성일: 2024년 12월
- 참고: 이 문서는 OSTEP, Linux 커널 문서, POSIX 표준을 기반으로 작성되었음.

## 관련 학습

- [문맥 교환](03-문맥-교환.md)
- [교착 상태의 조건과 사례](../동기화/02-교착-상태의-조건과-사례.md)
- [프로세스](01-프로세스.md)
- [스레드와 실행 모델](02-스레드와-실행-모델.md)

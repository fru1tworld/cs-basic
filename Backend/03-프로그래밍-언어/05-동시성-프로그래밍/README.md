# 동시성 프로그래밍 (Concurrent Programming)

## 목차
1. [동시성 vs 병렬성](#1-동시성-vs-병렬성)
2. [Race Condition과 Critical Section](#2-race-condition과-critical-section)
3. [락 기반 동기화](#3-락-기반-동기화)
4. [Lock-Free 알고리즘](#4-lock-free-알고리즘)
5. [데드락](#5-데드락)
6. [고급 동시성 모델](#6-고급-동시성-모델)
---

## 1. 동시성 vs 병렬성

### 1.1 개념 비교

```
동시성 (Concurrency)
────────────────────
여러 작업이 "동시에 진행되는 것처럼" 보이게 하는 것
→ 시간 분할 (Time Slicing)

  CPU Core 1
  ┌─────┬─────┬─────┬─────┬─────┬─────┐
  │Task1│Task2│Task1│Task3│Task2│Task1│ → 시간
  └─────┴─────┴─────┴─────┴─────┴─────┘
        컨텍스트 스위칭으로 번갈아 실행


병렬성 (Parallelism)
────────────────────
여러 작업이 "실제로 동시에" 실행되는 것
→ 물리적 병렬 실행

  CPU Core 1  ┌───────────────────────────┐
              │         Task 1            │ → 시간
              └───────────────────────────┘

  CPU Core 2  ┌───────────────────────────┐
              │         Task 2            │ → 시간
              └───────────────────────────┘

  CPU Core 3  ┌───────────────────────────┐
              │         Task 3            │ → 시간
              └───────────────────────────┘
```

### 1.2 관계

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  ┌──────────────┐          ┌──────────────┐        │
│  │   동시성     │          │    병렬성     │        │
│  │ Concurrency │          │  Parallelism  │        │
│  │             │          │               │        │
│  │  구조에 관한 │  ◀────▶  │  실행에 관한  │        │
│  │    개념     │          │     개념      │        │
│  └──────────────┘          └──────────────┘        │
│                                                     │
│  "동시에 많은 일을    vs   "많은 일을 동시에       │
│   다루는 것"              하는 것"                 │
│                                                     │
└─────────────────────────────────────────────────────┘

예시:
- 싱글 코어 + 멀티 스레드: 동시성 O, 병렬성 X
- 멀티 코어 + 멀티 스레드: 동시성 O, 병렬성 O
- 멀티 코어 + 싱글 스레드: 동시성 X, 병렬성 X
```

### 1.3 언어별 동시성 지원

```java
// Java - Thread, ExecutorService, CompletableFuture
public class JavaConcurrency {
    public static void main(String[] args) throws Exception {
        // 1. Thread 직접 사용
        Thread thread = new Thread(() -> {
            System.out.println("Running in thread: " + Thread.currentThread().getName());
        });
        thread.start();

        // 2. ExecutorService (Thread Pool)
        ExecutorService executor = Executors.newFixedThreadPool(4);
        Future<String> future = executor.submit(() -> {
            Thread.sleep(1000);
            return "Result";
        });
        String result = future.get();  // 블로킹

        // 3. CompletableFuture (비동기)
        CompletableFuture<String> cf = CompletableFuture
            .supplyAsync(() -> fetchData())
            .thenApply(data -> process(data))
            .thenApply(result -> format(result));

        // 4. Virtual Threads (Java 21+)
        Thread.startVirtualThread(() -> {
            System.out.println("Virtual thread!");
        });

        executor.shutdown();
    }
}
```

```go
// Go - Goroutines, Channels
package main

import (
    "fmt"
    "sync"
    "time"
)

func main() {
    // 1. Goroutine - 경량 스레드
    go func() {
        fmt.Println("Running in goroutine")
    }()

    // 2. Channel - 고루틴 간 통신
    ch := make(chan string)
    go func() {
        ch <- "Hello from goroutine"
    }()
    msg := <-ch
    fmt.Println(msg)

    // 3. WaitGroup - 동기화
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            fmt.Printf("Worker %d\n", n)
        }(i)
    }
    wg.Wait()

    // 4. Select - 다중 채널 처리
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() { ch1 <- "from ch1" }()
    go func() { ch2 <- "from ch2" }()

    select {
    case msg1 := <-ch1:
        fmt.Println(msg1)
    case msg2 := <-ch2:
        fmt.Println(msg2)
    case <-time.After(time.Second):
        fmt.Println("timeout")
    }
}
```

```python
# Python - threading, asyncio, multiprocessing
import asyncio
import threading
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# 1. threading (GIL 제한)
def worker():
    print(f"Thread: {threading.current_thread().name}")

threads = [threading.Thread(target=worker) for _ in range(5)]
for t in threads:
    t.start()
for t in threads:
    t.join()

# 2. asyncio (비동기 I/O)
async def fetch_data(url):
    await asyncio.sleep(1)  # I/O 작업 시뮬레이션
    return f"Data from {url}"

async def main():
    tasks = [fetch_data(f"url{i}") for i in range(5)]
    results = await asyncio.gather(*tasks)
    print(results)

asyncio.run(main())

# 3. ProcessPoolExecutor (CPU 집약적 작업)
def cpu_bound_task(n):
    return sum(i * i for i in range(n))

with ProcessPoolExecutor() as executor:
    futures = [executor.submit(cpu_bound_task, 10**6) for _ in range(4)]
    results = [f.result() for f in futures]
```

---

## 2. Race Condition과 Critical Section

### 2.1 Race Condition (경쟁 상태)

```
Race Condition: 여러 스레드가 공유 자원에 동시에 접근하여
                실행 순서에 따라 결과가 달라지는 상황

예: counter++ 연산 (원자적이지 않음)
──────────────────────────────────────
  Thread 1          Thread 2           counter
  ─────────         ─────────          ───────
  read counter                           0
                    read counter         0
  increment (0+1)
                    increment (0+1)
  write (1)                              1
                    write (1)            1  ← 예상: 2, 실제: 1
```

```java
// Race Condition 예제
public class RaceConditionExample {
    private int counter = 0;

    public void increment() {
        counter++;  // read-modify-write (3단계 연산)
    }

    public static void main(String[] args) throws InterruptedException {
        RaceConditionExample example = new RaceConditionExample();

        Runnable task = () -> {
            for (int i = 0; i < 10000; i++) {
                example.increment();
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("Counter: " + example.counter);
        // 예상: 20000, 실제: 매번 다름 (< 20000)
    }
}
```

### 2.2 Critical Section (임계 영역)

```
Critical Section: 공유 자원에 접근하는 코드 영역
                  한 번에 하나의 스레드만 실행해야 함

  ┌─────────────────────────────────────────┐
  │            Non-Critical Section          │
  │  (여러 스레드가 동시 실행 가능)            │
  └─────────────────────────────────────────┘
                      │
                      ▼
  ╔═════════════════════════════════════════╗
  ║          ⚠ Critical Section ⚠           ║
  ║  (한 번에 하나의 스레드만 실행)            ║
  ║                                          ║
  ║    shared_resource.modify();            ║
  ║                                          ║
  ╚═════════════════════════════════════════╝
                      │
                      ▼
  ┌─────────────────────────────────────────┐
  │            Non-Critical Section          │
  └─────────────────────────────────────────┘
```

### 2.3 데이터 레이스 vs Race Condition

```
데이터 레이스 (Data Race):
- 정의: 두 스레드가 동시에 같은 메모리에 접근하고,
        적어도 하나가 쓰기인 경우
- 해결: 동기화 (synchronized, lock, atomic)

Race Condition:
- 정의: 실행 순서에 따라 결과가 달라지는 상황
- 데이터 레이스 없이도 발생 가능

예: Check-then-act 패턴
```

```java
// Race Condition (데이터 레이스 없이)
public class CheckThenAct {
    private Map<String, String> cache = new ConcurrentHashMap<>();

    // Race Condition 존재 (synchronized 해도 문제)
    public String getOrCreate(String key) {
        if (!cache.containsKey(key)) {      // check
            cache.put(key, compute(key));    // act
        }
        return cache.get(key);
    }

    // 해결: computeIfAbsent (원자적 연산)
    public String getOrCreateSafe(String key) {
        return cache.computeIfAbsent(key, this::compute);
    }
}
```

---

## 3. 락 기반 동기화

### 3.1 Mutex (Mutual Exclusion)

```java
// Java - synchronized (내장 락/모니터 락)
public class Counter {
    private int count = 0;

    // 메서드 전체 동기화
    public synchronized void increment() {
        count++;
    }

    // 블록 동기화 (더 세밀한 제어)
    public void incrementWithBlock() {
        // non-critical section
        synchronized (this) {
            count++;  // critical section
        }
        // non-critical section
    }

    // static 메서드 동기화 (클래스 락)
    public static synchronized void staticMethod() {
        // Class 객체를 락으로 사용
    }
}
```

```java
// Java - ReentrantLock (명시적 락)
import java.util.concurrent.locks.ReentrantLock;

public class CounterWithLock {
    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();  // 반드시 finally에서 해제
        }
    }

    // tryLock - 논블로킹 락 획득 시도
    public boolean tryIncrement() {
        if (lock.tryLock()) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }

    // tryLock with timeout
    public boolean tryIncrementWithTimeout() throws InterruptedException {
        if (lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                count++;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false;
    }
}
```

```go
// Go - sync.Mutex
package main

import (
    "sync"
)

type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}
```

### 3.2 Semaphore (세마포어)

```
Semaphore: 동시에 리소스에 접근할 수 있는 스레드 수 제한

  ┌─────────────────────────────────────────┐
  │           Semaphore (permits=3)          │
  │  ┌───┐ ┌───┐ ┌───┐                      │
  │  │ 1 │ │ 2 │ │ 3 │  ← 사용 가능 permit   │
  │  └───┘ └───┘ └───┘                      │
  │                                          │
  │  acquire(): permit 획득 (없으면 대기)      │
  │  release(): permit 반환                   │
  └─────────────────────────────────────────┘

  Mutex = Semaphore(1)  (Binary Semaphore)
```

```java
import java.util.concurrent.Semaphore;

public class ConnectionPool {
    private static final int MAX_CONNECTIONS = 10;
    private final Semaphore semaphore = new Semaphore(MAX_CONNECTIONS);
    private final Queue<Connection> pool = new LinkedList<>();

    public Connection acquire() throws InterruptedException {
        semaphore.acquire();  // permit 획득 (없으면 블로킹)
        synchronized (pool) {
            return pool.poll();
        }
    }

    public void release(Connection conn) {
        synchronized (pool) {
            pool.offer(conn);
        }
        semaphore.release();  // permit 반환
    }

    // tryAcquire - 논블로킹
    public Connection tryAcquire() {
        if (semaphore.tryAcquire()) {
            synchronized (pool) {
                return pool.poll();
            }
        }
        return null;
    }
}
```

### 3.3 Read-Write Lock

```
Read-Write Lock: 읽기는 여러 스레드, 쓰기는 단독
                 읽기가 많은 상황에서 성능 향상

  ┌─────────────────────────────────────────┐
  │          Read-Write Lock                 │
  │                                          │
  │  Read Lock:   여러 스레드가 동시 획득     │
  │               (쓰기 락 없을 때)           │
  │                                          │
  │  Write Lock:  단독 획득                  │
  │               (읽기/쓰기 락 모두 없을 때)  │
  └─────────────────────────────────────────┘
```

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();

    public V get(K key) {
        rwLock.readLock().lock();  // 여러 스레드 동시 읽기
        try {
            return cache.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }

    public void put(K key, V value) {
        rwLock.writeLock().lock();  // 단독 쓰기
        try {
            cache.put(key, value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public V getOrCompute(K key, Function<K, V> compute) {
        // 읽기 락으로 먼저 확인
        rwLock.readLock().lock();
        try {
            V value = cache.get(key);
            if (value != null) return value;
        } finally {
            rwLock.readLock().unlock();
        }

        // 쓰기 락으로 계산 및 저장
        rwLock.writeLock().lock();
        try {
            // Double-check
            V value = cache.get(key);
            if (value == null) {
                value = compute.apply(key);
                cache.put(key, value);
            }
            return value;
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

```go
// Go - sync.RWMutex
type SafeCache struct {
    mu    sync.RWMutex
    cache map[string]interface{}
}

func (c *SafeCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()  // 읽기 락
    defer c.mu.RUnlock()
    val, ok := c.cache[key]
    return val, ok
}

func (c *SafeCache) Set(key string, value interface{}) {
    c.mu.Lock()  // 쓰기 락
    defer c.mu.Unlock()
    c.cache[key] = value
}
```

### 3.4 Condition Variable

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedBuffer<T> {
    private final T[] buffer;
    private int count, putIdx, takeIdx;

    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    @SuppressWarnings("unchecked")
    public BoundedBuffer(int capacity) {
        buffer = (T[]) new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (count == buffer.length) {
                notFull.await();  // 버퍼가 가득 차면 대기
            }
            buffer[putIdx] = item;
            if (++putIdx == buffer.length) putIdx = 0;
            count++;
            notEmpty.signal();  // 소비자에게 알림
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 버퍼가 비어있으면 대기
            }
            T item = buffer[takeIdx];
            if (++takeIdx == buffer.length) takeIdx = 0;
            count--;
            notFull.signal();  // 생산자에게 알림
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 4. Lock-Free 알고리즘

### 4.1 Atomic 연산

```java
import java.util.concurrent.atomic.*;

public class AtomicExample {
    // Atomic 변수 - CAS (Compare-And-Swap) 기반
    private AtomicInteger counter = new AtomicInteger(0);
    private AtomicReference<Node> head = new AtomicReference<>();
    private AtomicLong sequence = new AtomicLong(0);

    public void increment() {
        counter.incrementAndGet();  // 원자적 증가
    }

    // CAS 직접 사용
    public void incrementWithCAS() {
        int oldValue, newValue;
        do {
            oldValue = counter.get();
            newValue = oldValue + 1;
        } while (!counter.compareAndSet(oldValue, newValue));
    }

    // updateAndGet (Java 8+)
    public int incrementAndDouble() {
        return counter.updateAndGet(x -> x * 2 + 1);
    }

    // accumulateAndGet
    public int addWithMax(int delta) {
        return counter.accumulateAndGet(delta, Math::max);
    }
}
```

### 4.2 CAS (Compare-And-Swap)

```
CAS 연산:
─────────
compare_and_swap(memory_location, expected_value, new_value)

1. memory_location의 현재 값을 읽음
2. 현재 값 == expected_value 이면
   → memory_location = new_value (원자적)
   → return true
3. 아니면
   → return false (다시 시도)

┌─────────────────────────────────────────────────────────┐
│  Thread 1                    Thread 2                   │
│  ─────────                   ─────────                  │
│  read: value = 5                                        │
│                              read: value = 5            │
│  CAS(5, 6) → success                                    │
│  value = 6                                              │
│                              CAS(5, 6) → fail!          │
│                              retry...                    │
│                              read: value = 6            │
│                              CAS(6, 7) → success        │
│                              value = 7                  │
└─────────────────────────────────────────────────────────┘
```

### 4.3 Lock-Free Stack

```java
import java.util.concurrent.atomic.AtomicReference;

public class LockFreeStack<T> {
    private static class Node<T> {
        final T value;
        Node<T> next;

        Node(T value) {
            this.value = value;
        }
    }

    private final AtomicReference<Node<T>> top = new AtomicReference<>();

    public void push(T value) {
        Node<T> newNode = new Node<>(value);
        Node<T> oldTop;
        do {
            oldTop = top.get();
            newNode.next = oldTop;
        } while (!top.compareAndSet(oldTop, newNode));
    }

    public T pop() {
        Node<T> oldTop;
        Node<T> newTop;
        do {
            oldTop = top.get();
            if (oldTop == null) {
                return null;  // 스택 비어있음
            }
            newTop = oldTop.next;
        } while (!top.compareAndSet(oldTop, newTop));
        return oldTop.value;
    }
}
```

### 4.4 ABA 문제

```
ABA Problem:
────────────
CAS가 값이 변경되었다가 다시 원래 값으로 돌아온 경우를 감지 못함

Thread 1: read A
Thread 2: A → B → A (값이 변경되었다가 복원)
Thread 1: CAS(A, C) → success! (하지만 의미상 잘못됨)

해결: AtomicStampedReference (버전 번호 추가)
```

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABAPreventionExample {
    private AtomicStampedReference<Integer> value =
        new AtomicStampedReference<>(0, 0);

    public boolean update(int expected, int newValue) {
        int[] stampHolder = new int[1];
        Integer current = value.get(stampHolder);
        int stamp = stampHolder[0];

        if (current.equals(expected)) {
            // 값과 스탬프 모두 비교
            return value.compareAndSet(expected, newValue, stamp, stamp + 1);
        }
        return false;
    }
}
```

### 4.5 Lock-Free vs Wait-Free

```
Lock-Free:
- 최소 하나의 스레드는 항상 진행
- 다른 스레드는 기아(starvation) 가능

Wait-Free:
- 모든 스레드가 유한 시간 내에 완료 보장
- 구현이 더 어려움

락 기반 < Lock-Free < Wait-Free
(성능)     (진행 보장)   (공정성)
```

---

## 5. 데드락

### 5.1 데드락 조건 (Coffman Conditions)

```
데드락 발생의 4가지 필요조건:
───────────────────────────
1. Mutual Exclusion (상호 배제)
   - 자원은 한 번에 하나의 스레드만 사용

2. Hold and Wait (점유와 대기)
   - 자원을 보유한 채로 다른 자원 대기

3. No Preemption (비선점)
   - 자원을 강제로 빼앗을 수 없음

4. Circular Wait (순환 대기)
   - 스레드들이 원형으로 서로의 자원 대기

  Thread 1 ────▶ Resource A ────▶ Thread 2
      ▲                              │
      │                              │
      │                              ▼
  Resource B ◀────────────────── waiting
```

### 5.2 데드락 예제

```java
public class DeadlockExample {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    public void method1() {
        synchronized (lockA) {
            System.out.println("Thread 1: Holding lock A...");
            try { Thread.sleep(100); } catch (InterruptedException e) {}

            System.out.println("Thread 1: Waiting for lock B...");
            synchronized (lockB) {
                System.out.println("Thread 1: Holding lock A & B");
            }
        }
    }

    public void method2() {
        synchronized (lockB) {  // 순서가 다름!
            System.out.println("Thread 2: Holding lock B...");
            try { Thread.sleep(100); } catch (InterruptedException e) {}

            System.out.println("Thread 2: Waiting for lock A...");
            synchronized (lockA) {
                System.out.println("Thread 2: Holding lock A & B");
            }
        }
    }

    public static void main(String[] args) {
        DeadlockExample example = new DeadlockExample();

        new Thread(example::method1).start();
        new Thread(example::method2).start();
        // 데드락 발생!
    }
}
```

### 5.3 데드락 예방

```java
// 1. 락 순서 고정 (Lock Ordering)
public class LockOrderingExample {
    private final Object lockA = new Object();
    private final Object lockB = new Object();

    // 항상 lockA를 먼저, lockB를 나중에
    public void method1() {
        synchronized (lockA) {
            synchronized (lockB) {
                // work
            }
        }
    }

    public void method2() {
        synchronized (lockA) {  // 순서 통일!
            synchronized (lockB) {
                // work
            }
        }
    }
}

// 2. tryLock with timeout
public class TryLockExample {
    private final ReentrantLock lockA = new ReentrantLock();
    private final ReentrantLock lockB = new ReentrantLock();

    public void safeMethod() {
        while (true) {
            if (lockA.tryLock()) {
                try {
                    if (lockB.tryLock()) {
                        try {
                            // 두 락 모두 획득
                            doWork();
                            return;
                        } finally {
                            lockB.unlock();
                        }
                    }
                } finally {
                    lockA.unlock();
                }
            }
            // 잠시 대기 후 재시도
            try { Thread.sleep(50); } catch (InterruptedException e) {}
        }
    }
}

// 3. 락 계층화 (Lock Hierarchy)
public class LockHierarchy {
    private static final int LOCK_A_ORDER = 1;
    private static final int LOCK_B_ORDER = 2;

    private final Object lockA = new Object();
    private final Object lockB = new Object();

    // 더 낮은 순서의 락을 먼저 획득해야 함
    private void checkLockOrder(int current, int required) {
        if (current >= required) {
            throw new IllegalStateException("Lock order violation");
        }
    }
}
```

### 5.4 데드락 탐지

```java
// ThreadMXBean으로 데드락 탐지
import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;

public class DeadlockDetector {
    private final ThreadMXBean threadMXBean =
        ManagementFactory.getThreadMXBean();

    public void detectDeadlock() {
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();

        if (deadlockedThreads != null) {
            System.out.println("Deadlock detected!");
            for (long threadId : deadlockedThreads) {
                ThreadInfo info = threadMXBean.getThreadInfo(threadId);
                System.out.println("Thread: " + info.getThreadName());
                System.out.println("Lock: " + info.getLockName());
                System.out.println("Lock Owner: " + info.getLockOwnerName());
            }
        }
    }

    // 주기적 실행
    public void startMonitoring() {
        ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();
        executor.scheduleAtFixedRate(this::detectDeadlock, 0, 10, TimeUnit.SECONDS);
    }
}
```

---

## 6. 고급 동시성 모델

### 6.1 Actor 모델

```
Actor 모델:
───────────
- 각 Actor는 독립적인 상태와 메일박스를 가짐
- Actor 간 통신은 오직 메시지 전달로만
- 공유 상태 없음 → 락 불필요

  ┌────────────────┐     message     ┌────────────────┐
  │    Actor A     │ ──────────────▶ │    Actor B     │
  │  ┌──────────┐  │                 │  ┌──────────┐  │
  │  │  State   │  │                 │  │  State   │  │
  │  └──────────┘  │                 │  └──────────┘  │
  │  ┌──────────┐  │                 │  ┌──────────┐  │
  │  │ Mailbox  │  │                 │  │ Mailbox  │  │
  │  │ [msg1]   │  │                 │  │ [msg2]   │  │
  │  │ [msg2]   │  │                 │  │ [msg3]   │  │
  │  └──────────┘  │                 │  └──────────┘  │
  └────────────────┘                 └────────────────┘
```

```java
// Java - Akka 스타일 Actor (의사 코드)
public abstract class Actor {
    private final Queue<Message> mailbox = new ConcurrentLinkedQueue<>();
    private final ExecutorService executor;

    protected abstract void onReceive(Message message);

    public void tell(Message message) {
        mailbox.offer(message);
        executor.submit(this::processMailbox);
    }

    private void processMailbox() {
        Message message = mailbox.poll();
        if (message != null) {
            onReceive(message);
        }
    }
}

// 사용 예
public class CounterActor extends Actor {
    private int count = 0;  // 상태는 Actor 내부에만 존재

    @Override
    protected void onReceive(Message message) {
        switch (message.type()) {
            case "INCREMENT" -> count++;
            case "DECREMENT" -> count--;
            case "GET" -> sender().tell(new Message("RESULT", count));
        }
    }
}
```

```go
// Go - Actor 패턴 구현
type Actor struct {
    mailbox chan Message
    state   interface{}
    handler func(interface{}, Message) interface{}
}

func NewActor(initialState interface{}, handler func(interface{}, Message) interface{}) *Actor {
    a := &Actor{
        mailbox: make(chan Message, 100),
        state:   initialState,
        handler: handler,
    }
    go a.run()
    return a
}

func (a *Actor) run() {
    for msg := range a.mailbox {
        a.state = a.handler(a.state, msg)
    }
}

func (a *Actor) Send(msg Message) {
    a.mailbox <- msg
}

// 사용
counterActor := NewActor(0, func(state interface{}, msg Message) interface{} {
    count := state.(int)
    switch msg.Type {
    case "INCREMENT":
        return count + 1
    case "DECREMENT":
        return count - 1
    default:
        return count
    }
})

counterActor.Send(Message{Type: "INCREMENT"})
```

### 6.2 CSP (Communicating Sequential Processes)

```
CSP: 채널을 통한 프로세스 간 통신

  Process 1          Channel           Process 2
  ┌─────────┐       ┌───────┐         ┌─────────┐
  │  send   │ ────▶ │ data  │ ─────▶  │ receive │
  └─────────┘       └───────┘         └─────────┘

  "공유 메모리로 통신하지 말고, 통신으로 메모리를 공유하라"
  - Go 철학
```

```go
// Go - CSP 패턴
package main

// Worker Pool 패턴
func workerPool(numWorkers int, jobs <-chan int, results chan<- int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }(i)
    }
    wg.Wait()
    close(results)
}

// Fan-Out, Fan-In 패턴
func fanOut(input <-chan int, n int) []<-chan int {
    outputs := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        outputs[i] = worker(input)
    }
    return outputs
}

func fanIn(inputs ...<-chan int) <-chan int {
    output := make(chan int)
    var wg sync.WaitGroup

    for _, input := range inputs {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for v := range ch {
                output <- v
            }
        }(input)
    }

    go func() {
        wg.Wait()
        close(output)
    }()

    return output
}

// Pipeline 패턴
func pipeline() {
    // Stage 1: 생성
    gen := func(nums ...int) <-chan int {
        out := make(chan int)
        go func() {
            for _, n := range nums {
                out <- n
            }
            close(out)
        }()
        return out
    }

    // Stage 2: 제곱
    sq := func(in <-chan int) <-chan int {
        out := make(chan int)
        go func() {
            for n := range in {
                out <- n * n
            }
            close(out)
        }()
        return out
    }

    // Stage 3: 출력
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n)  // 16, 81
    }
}
```

### 6.3 Software Transactional Memory (STM)

```
STM: 트랜잭션처럼 메모리 접근을 관리

  atomic {
      // 이 블록 내의 모든 메모리 접근은 원자적
      x = x + 1
      y = y - 1
  }

특징:
- Optimistic Concurrency Control
- 충돌 시 자동 재시도
- 락보다 사용하기 쉬움
- 데드락 없음
```

```java
// Java - Multiverse STM (의사 코드)
import org.multiverse.api.StmUtils;
import org.multiverse.api.references.TxnInteger;

public class STMExample {
    private final TxnInteger balance1 = StmUtils.newTxnInteger(100);
    private final TxnInteger balance2 = StmUtils.newTxnInteger(100);

    public void transfer(int amount) {
        StmUtils.atomic(() -> {
            if (balance1.get() >= amount) {
                balance1.decrement(amount);
                balance2.increment(amount);
            }
        });
    }

    public int getTotal() {
        return StmUtils.atomic(() ->
            balance1.get() + balance2.get()
        );
    }
}
```

### 6.4 Reactive Streams

```java
// Java - Project Reactor
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public class ReactiveExample {
    public Flux<Integer> processNumbers() {
        return Flux.range(1, 100)
            .filter(n -> n % 2 == 0)
            .map(n -> n * 2)
            .subscribeOn(Schedulers.parallel())
            .publishOn(Schedulers.boundedElastic());
    }

    public Mono<User> getUserWithOrders(Long userId) {
        return userRepository.findById(userId)
            .flatMap(user ->
                orderRepository.findByUserId(userId)
                    .collectList()
                    .map(orders -> {
                        user.setOrders(orders);
                        return user;
                    })
            );
    }

    // Backpressure 처리
    public Flux<Data> streamData() {
        return Flux.generate(sink -> sink.next(getData()))
            .onBackpressureBuffer(100)
            .onBackpressureDrop(data ->
                log.warn("Dropped: {}", data)
            );
    }
}
```

---

## 참고 자료

- "Java Concurrency in Practice" - Brian Goetz
- "The Art of Multiprocessor Programming" - Maurice Herlihy, Nir Shavit
- "Concurrency in Go" - Katherine Cox-Buday
- "Seven Concurrency Models in Seven Weeks" - Paul Butcher
- Go Documentation - Effective Go (Concurrency)
- Java Documentation - java.util.concurrent

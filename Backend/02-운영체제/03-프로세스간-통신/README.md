# 프로세스간 통신 (Inter-Process Communication, IPC)

> **OSTEP(Operating Systems: Three Easy Pieces)** 및 **POSIX.1-2017** 기반

---

## 목차

1. [IPC 개요](#1-ipc-개요)
2. [Pipe와 Named Pipe](#2-pipe와-named-pipe)
3. [Message Queue](#3-message-queue)
4. [Shared Memory](#4-shared-memory)
5. [Semaphore와 Mutex](#5-semaphore와-mutex)
6. [Socket](#6-socket)
8. [참고 자료](#8-참고-자료)

---

## 1. IPC 개요

### 1.1 IPC가 필요한 이유

프로세스는 독립된 주소 공간을 가지므로, 다른 프로세스와 데이터를 공유하려면 특별한 메커니즘이 필요합니다.

```
Process A                    Process B
┌─────────────────┐         ┌─────────────────┐
│  Virtual Memory │         │  Virtual Memory │
│  ┌───────────┐  │   ???   │  ┌───────────┐  │
│  │   Data    │  │ ◄─────► │  │   Data    │  │
│  └───────────┘  │         │  └───────────┘  │
│  Physical: 0x100│         │  Physical: 0x200│
└─────────────────┘         └─────────────────┘

→ 커널을 통한 IPC 메커니즘 필요
```

### 1.2 IPC 메커니즘 비교

| 메커니즘 | 동기성 | 관계 | 데이터 형식 | 성능 | 사용 사례 |
|----------|--------|------|-------------|------|-----------|
| **Pipe** | 동기 | 부모-자식 | 바이트 스트림 | 중간 | 셸 파이프라인 |
| **Named Pipe** | 동기 | 임의 | 바이트 스트림 | 중간 | 독립 프로세스 통신 |
| **Message Queue** | 비동기 | 임의 | 구조화된 메시지 | 중간 | 생산자-소비자 |
| **Shared Memory** | N/A | 임의 | 원시 바이트 | 최고 | 대용량 데이터 공유 |
| **Semaphore** | 동기화 | 임의 | N/A | 높음 | 자원 접근 제어 |
| **Socket** | 동기/비동기 | 로컬/원격 | 바이트 스트림/데이터그램 | 낮음 | 네트워크 통신 |

### 1.3 IPC 선택 기준

```
데이터 크기
├── 작음 (< 1KB): Pipe, Message Queue
├── 중간 (1KB ~ 1MB): Named Pipe, Socket
└── 큼 (> 1MB): Shared Memory

통신 패턴
├── 단방향: Pipe
├── 양방향: Socket, 2개의 Pipe
└── 1:N, N:M: Message Queue, Shared Memory

프로세스 관계
├── 부모-자식: Pipe (익명)
└── 무관한 프로세스: Named Pipe, Message Queue, Socket

네트워크 여부
├── 로컬만: Pipe, Shared Memory (더 빠름)
└── 원격 가능: Socket
```

---

## 2. Pipe와 Named Pipe

### 2.1 익명 파이프 (Anonymous Pipe)

부모-자식 프로세스 간의 단방향 통신 채널입니다.

```
                    Kernel Buffer
Parent              ┌──────────────┐              Child
┌───────┐  write()  │              │  read()   ┌───────┐
│ fd[1] │ ────────► │  ░░░░░░░░░░  │ ────────► │ fd[0] │
└───────┘           │  Pipe Buffer │           └───────┘
                    └──────────────┘
                    (기본 65536 bytes)
```

#### C 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

#define BUFFER_SIZE 256

int main() {
    int pipefd[2];  // pipefd[0]: 읽기, pipefd[1]: 쓰기
    pid_t pid;
    char buffer[BUFFER_SIZE];

    // 파이프 생성
    if (pipe(pipefd) == -1) {
        perror("pipe");
        exit(EXIT_FAILURE);
    }

    pid = fork();

    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);
    }

    if (pid == 0) {
        // 자식 프로세스: 읽기만 수행
        close(pipefd[1]);  // 쓰기 끝 닫기

        ssize_t bytes_read;
        while ((bytes_read = read(pipefd[0], buffer, BUFFER_SIZE)) > 0) {
            printf("Child received: %.*s", (int)bytes_read, buffer);
        }

        close(pipefd[0]);
        exit(EXIT_SUCCESS);

    } else {
        // 부모 프로세스: 쓰기만 수행
        close(pipefd[0]);  // 읽기 끝 닫기

        const char *messages[] = {
            "Hello from parent!\n",
            "This is IPC via pipe.\n",
            "Goodbye!\n"
        };

        for (int i = 0; i < 3; i++) {
            write(pipefd[1], messages[i], strlen(messages[i]));
            usleep(100000);  // 100ms 대기
        }

        close(pipefd[1]);  // EOF 전송
        wait(NULL);
    }

    return 0;
}
```

#### 양방향 파이프 통신

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

int main() {
    int parent_to_child[2];
    int child_to_parent[2];
    pid_t pid;

    // 두 개의 파이프 생성
    pipe(parent_to_child);
    pipe(child_to_parent);

    pid = fork();

    if (pid == 0) {
        // 자식 프로세스
        close(parent_to_child[1]);  // P→C 쓰기 닫기
        close(child_to_parent[0]);  // C→P 읽기 닫기

        char buffer[256];
        int num;

        // 부모로부터 숫자 받기
        read(parent_to_child[0], buffer, sizeof(buffer));
        num = atoi(buffer);
        printf("Child: Received %d, sending back %d\n", num, num * 2);

        // 결과 전송
        sprintf(buffer, "%d", num * 2);
        write(child_to_parent[1], buffer, strlen(buffer) + 1);

        close(parent_to_child[0]);
        close(child_to_parent[1]);
        exit(0);

    } else {
        // 부모 프로세스
        close(parent_to_child[0]);  // P→C 읽기 닫기
        close(child_to_parent[1]);  // C→P 쓰기 닫기

        char buffer[256];

        // 자식에게 숫자 전송
        sprintf(buffer, "%d", 42);
        write(parent_to_child[1], buffer, strlen(buffer) + 1);

        // 결과 받기
        read(child_to_parent[0], buffer, sizeof(buffer));
        printf("Parent: Received result: %s\n", buffer);

        close(parent_to_child[1]);
        close(child_to_parent[0]);
        wait(NULL);
    }

    return 0;
}
```

#### Python 구현

```python
import os
import sys

def pipe_example():
    """Python에서의 파이프 사용 예제"""

    # 파이프 생성
    read_fd, write_fd = os.pipe()

    pid = os.fork()

    if pid == 0:
        # 자식 프로세스
        os.close(write_fd)  # 쓰기 끝 닫기

        # 파일 객체로 변환
        with os.fdopen(read_fd, 'r') as reader:
            for line in reader:
                print(f"Child received: {line.strip()}")

        sys.exit(0)

    else:
        # 부모 프로세스
        os.close(read_fd)  # 읽기 끝 닫기

        with os.fdopen(write_fd, 'w') as writer:
            messages = ["Hello", "World", "From Parent"]
            for msg in messages:
                writer.write(f"{msg}\n")
                writer.flush()

        os.wait()

if __name__ == "__main__":
    pipe_example()
```

### 2.2 Named Pipe (FIFO)

파일 시스템에 존재하는 특수 파일로, 관련 없는 프로세스 간 통신이 가능합니다.

```
$ ls -la /tmp/myfifo
prw-r--r-- 1 user user 0 Dec 15 10:00 /tmp/myfifo
 ^
 p = FIFO (named pipe)
```

#### C 구현: 서버 (읽기)

```c
// fifo_server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <errno.h>

#define FIFO_PATH "/tmp/my_fifo"
#define BUFFER_SIZE 1024

int main() {
    int fd;
    char buffer[BUFFER_SIZE];

    // FIFO 생성 (이미 존재하면 무시)
    if (mkfifo(FIFO_PATH, 0666) == -1) {
        if (errno != EEXIST) {
            perror("mkfifo");
            exit(EXIT_FAILURE);
        }
    }

    printf("Server: Waiting for client connection...\n");

    // 읽기 전용으로 열기 (클라이언트가 쓰기로 열 때까지 블록)
    fd = open(FIFO_PATH, O_RDONLY);
    if (fd == -1) {
        perror("open");
        exit(EXIT_FAILURE);
    }

    printf("Server: Client connected!\n");

    ssize_t bytes_read;
    while ((bytes_read = read(fd, buffer, BUFFER_SIZE - 1)) > 0) {
        buffer[bytes_read] = '\0';
        printf("Server received: %s", buffer);

        if (strncmp(buffer, "quit", 4) == 0) {
            break;
        }
    }

    close(fd);
    unlink(FIFO_PATH);
    printf("Server: Shutting down.\n");

    return 0;
}
```

#### C 구현: 클라이언트 (쓰기)

```c
// fifo_client.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>

#define FIFO_PATH "/tmp/my_fifo"
#define BUFFER_SIZE 1024

int main() {
    int fd;
    char buffer[BUFFER_SIZE];

    // 쓰기 전용으로 열기
    fd = open(FIFO_PATH, O_WRONLY);
    if (fd == -1) {
        perror("open");
        printf("Is the server running?\n");
        exit(EXIT_FAILURE);
    }

    printf("Client: Connected to server!\n");
    printf("Enter messages (type 'quit' to exit):\n");

    while (fgets(buffer, BUFFER_SIZE, stdin) != NULL) {
        write(fd, buffer, strlen(buffer));

        if (strncmp(buffer, "quit", 4) == 0) {
            break;
        }
    }

    close(fd);
    printf("Client: Disconnected.\n");

    return 0;
}
```

#### Python FIFO 예제

```python
import os
import stat
import errno

FIFO_PATH = "/tmp/python_fifo"

def create_fifo():
    """FIFO 생성"""
    try:
        os.mkfifo(FIFO_PATH)
    except OSError as e:
        if e.errno != errno.EEXIST:
            raise

def server():
    """FIFO 서버 (읽기)"""
    create_fifo()
    print("Server: Waiting for connection...")

    with open(FIFO_PATH, 'r') as fifo:
        print("Server: Client connected!")
        for line in fifo:
            print(f"Received: {line.strip()}")
            if line.strip() == "quit":
                break

    os.unlink(FIFO_PATH)
    print("Server: Shutdown")

def client():
    """FIFO 클라이언트 (쓰기)"""
    with open(FIFO_PATH, 'w') as fifo:
        print("Client: Connected!")
        while True:
            message = input("Enter message: ")
            fifo.write(message + "\n")
            fifo.flush()
            if message == "quit":
                break

    print("Client: Disconnected")

if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1 and sys.argv[1] == "client":
        client()
    else:
        server()
```

### 2.3 Pipe vs Named Pipe

| 특징 | Anonymous Pipe | Named Pipe (FIFO) |
|------|----------------|-------------------|
| **파일 시스템** | 없음 | 존재 (inode) |
| **프로세스 관계** | 부모-자식 필수 | 무관한 프로세스 가능 |
| **생성** | `pipe()` | `mkfifo()` |
| **지속성** | 프로세스 종료 시 삭제 | 명시적 삭제 필요 |
| **식별** | 파일 디스크립터 | 경로명 |

---

## 3. Message Queue

### 3.1 POSIX Message Queue

구조화된 메시지를 큐를 통해 비동기적으로 전달합니다.

```
Producer Process                Consumer Process
┌───────────────┐              ┌───────────────┐
│   mq_send()   │              │   mq_receive()│
└───────┬───────┘              └───────┬───────┘
        │                              │
        ▼                              ▼
   ┌─────────────────────────────────────┐
   │          Message Queue              │
   │  ┌─────┬─────┬─────┬─────┬─────┐   │
   │  │ Msg1│ Msg2│ Msg3│ ... │ MsgN│   │
   │  └─────┴─────┴─────┴─────┴─────┘   │
   │     Priority-based ordering         │
   └─────────────────────────────────────┘
                   Kernel Space
```

#### C 구현: 생산자

```c
// mq_producer.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <mqueue.h>
#include <errno.h>

#define QUEUE_NAME "/my_message_queue"
#define MAX_MSG_SIZE 256
#define MAX_MESSAGES 10

typedef struct {
    long type;
    char data[MAX_MSG_SIZE];
    int priority;
} message_t;

int main() {
    mqd_t mq;
    struct mq_attr attr;

    // 큐 속성 설정
    attr.mq_flags = 0;
    attr.mq_maxmsg = MAX_MESSAGES;
    attr.mq_msgsize = sizeof(message_t);
    attr.mq_curmsgs = 0;

    // 메시지 큐 생성/열기
    mq = mq_open(QUEUE_NAME, O_CREAT | O_WRONLY, 0644, &attr);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        exit(EXIT_FAILURE);
    }

    printf("Producer: Message queue opened.\n");

    // 메시지 전송
    for (int i = 0; i < 5; i++) {
        message_t msg;
        msg.type = 1;
        snprintf(msg.data, sizeof(msg.data), "Message #%d", i + 1);
        msg.priority = i % 3;  // 우선순위 0-2

        if (mq_send(mq, (char*)&msg, sizeof(msg), msg.priority) == -1) {
            perror("mq_send");
            exit(EXIT_FAILURE);
        }

        printf("Producer: Sent '%s' with priority %d\n", msg.data, msg.priority);
    }

    // 종료 메시지
    message_t end_msg = {.type = 0, .data = "END", .priority = 0};
    mq_send(mq, (char*)&end_msg, sizeof(end_msg), 9);  // 최고 우선순위

    mq_close(mq);
    printf("Producer: Done.\n");

    return 0;
}
```

#### C 구현: 소비자

```c
// mq_consumer.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <mqueue.h>

#define QUEUE_NAME "/my_message_queue"
#define MAX_MSG_SIZE 256

typedef struct {
    long type;
    char data[MAX_MSG_SIZE];
    int priority;
} message_t;

int main() {
    mqd_t mq;
    struct mq_attr attr;
    message_t msg;
    unsigned int priority;

    // 메시지 큐 열기
    mq = mq_open(QUEUE_NAME, O_RDONLY);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        exit(EXIT_FAILURE);
    }

    // 큐 속성 확인
    mq_getattr(mq, &attr);
    printf("Consumer: Queue has %ld messages\n", attr.mq_curmsgs);

    // 메시지 수신 (우선순위 높은 것부터)
    while (1) {
        ssize_t bytes_read = mq_receive(mq, (char*)&msg, sizeof(msg), &priority);

        if (bytes_read == -1) {
            perror("mq_receive");
            break;
        }

        printf("Consumer: Received '%s' (priority: %u)\n", msg.data, priority);

        if (strcmp(msg.data, "END") == 0) {
            break;
        }
    }

    mq_close(mq);
    mq_unlink(QUEUE_NAME);
    printf("Consumer: Done.\n");

    return 0;
}
```

#### 컴파일 및 실행

```bash
# 컴파일 (POSIX RT 라이브러리 링크)
gcc -o producer mq_producer.c -lrt
gcc -o consumer mq_consumer.c -lrt

# 실행
./producer &
./consumer
```

### 3.2 Python multiprocessing.Queue

```python
from multiprocessing import Process, Queue
import time
import random

def producer(queue: Queue, producer_id: int, num_items: int):
    """생산자 프로세스"""
    for i in range(num_items):
        item = {
            'id': f"{producer_id}-{i}",
            'value': random.randint(1, 100),
            'timestamp': time.time()
        }
        queue.put(item)
        print(f"Producer {producer_id}: Produced {item['id']}")
        time.sleep(random.uniform(0.1, 0.5))

    # 종료 신호
    queue.put(None)
    print(f"Producer {producer_id}: Done")

def consumer(queue: Queue, consumer_id: int, num_producers: int):
    """소비자 프로세스"""
    done_count = 0

    while done_count < num_producers:
        item = queue.get()

        if item is None:
            done_count += 1
            print(f"Consumer {consumer_id}: Producer finished ({done_count}/{num_producers})")
            continue

        print(f"Consumer {consumer_id}: Consumed {item['id']} = {item['value']}")
        time.sleep(random.uniform(0.05, 0.2))

    print(f"Consumer {consumer_id}: Done")

def main():
    """메시지 큐 데모"""
    queue = Queue(maxsize=10)

    # 2개의 생산자, 1개의 소비자
    producers = [
        Process(target=producer, args=(queue, i, 5))
        for i in range(2)
    ]
    consumer_proc = Process(target=consumer, args=(queue, 0, 2))

    # 시작
    for p in producers:
        p.start()
    consumer_proc.start()

    # 종료 대기
    for p in producers:
        p.join()
    consumer_proc.join()

    print("All processes completed")

if __name__ == "__main__":
    main()
```

### 3.3 Message Queue 속성

```c
// 큐 속성 조회 및 설정
#include <mqueue.h>

void inspect_queue(mqd_t mq) {
    struct mq_attr attr;
    mq_getattr(mq, &attr);

    printf("Queue Attributes:\n");
    printf("  mq_flags:   %ld (0=blocking, O_NONBLOCK)\n", attr.mq_flags);
    printf("  mq_maxmsg:  %ld (max messages in queue)\n", attr.mq_maxmsg);
    printf("  mq_msgsize: %ld (max message size)\n", attr.mq_msgsize);
    printf("  mq_curmsgs: %ld (current messages)\n", attr.mq_curmsgs);
}

// Non-blocking 설정
void set_nonblocking(mqd_t mq) {
    struct mq_attr attr;
    mq_getattr(mq, &attr);
    attr.mq_flags = O_NONBLOCK;
    mq_setattr(mq, &attr, NULL);
}
```

---

## 4. Shared Memory

### 4.1 공유 메모리 개요

가장 빠른 IPC 메커니즘으로, 프로세스들이 같은 물리 메모리 영역을 공유합니다.

```
Process A              Physical Memory            Process B
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│Virtual Memory│       │              │       │Virtual Memory│
│              │       │              │       │              │
│  0x7F000000 ─┼───────┼──► Frame N  ◄┼───────┼─ 0x7F100000  │
│  (shared)    │       │   (shared)   │       │  (shared)    │
│              │       │              │       │              │
└──────────────┘       └──────────────┘       └──────────────┘

장점: 커널 버퍼 복사 없음 → O(1) 데이터 접근
단점: 동기화 직접 구현 필요
```

### 4.2 POSIX Shared Memory

```c
// shm_writer.c - 공유 메모리에 데이터 쓰기
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

#define SHM_NAME "/my_shared_memory"
#define SHM_SIZE 4096

typedef struct {
    int counter;
    char message[256];
    int ready;
} shared_data_t;

int main() {
    int fd;
    shared_data_t *shared;

    // 공유 메모리 객체 생성
    fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    if (fd == -1) {
        perror("shm_open");
        exit(EXIT_FAILURE);
    }

    // 크기 설정
    if (ftruncate(fd, sizeof(shared_data_t)) == -1) {
        perror("ftruncate");
        exit(EXIT_FAILURE);
    }

    // 메모리 매핑
    shared = (shared_data_t*)mmap(NULL, sizeof(shared_data_t),
                                   PROT_READ | PROT_WRITE,
                                   MAP_SHARED, fd, 0);
    if (shared == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    // 데이터 초기화
    shared->counter = 0;
    shared->ready = 0;

    printf("Writer: Shared memory created at %p\n", shared);

    // 데이터 쓰기
    for (int i = 1; i <= 5; i++) {
        shared->counter = i;
        snprintf(shared->message, sizeof(shared->message),
                 "Message #%d from writer", i);
        shared->ready = 1;

        printf("Writer: Wrote counter=%d, message='%s'\n",
               shared->counter, shared->message);

        // 독자가 읽을 때까지 대기 (간단한 스핀락)
        while (shared->ready == 1) {
            usleep(1000);
        }
    }

    // 종료 신호
    shared->counter = -1;
    shared->ready = 1;

    sleep(1);  // 독자가 읽을 시간

    // 정리
    munmap(shared, sizeof(shared_data_t));
    close(fd);
    shm_unlink(SHM_NAME);

    printf("Writer: Done.\n");
    return 0;
}
```

```c
// shm_reader.c - 공유 메모리에서 데이터 읽기
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

#define SHM_NAME "/my_shared_memory"

typedef struct {
    int counter;
    char message[256];
    int ready;
} shared_data_t;

int main() {
    int fd;
    shared_data_t *shared;

    // 공유 메모리 열기
    fd = shm_open(SHM_NAME, O_RDWR, 0666);
    if (fd == -1) {
        perror("shm_open");
        printf("Is the writer running?\n");
        exit(EXIT_FAILURE);
    }

    // 메모리 매핑
    shared = (shared_data_t*)mmap(NULL, sizeof(shared_data_t),
                                   PROT_READ | PROT_WRITE,
                                   MAP_SHARED, fd, 0);
    if (shared == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    printf("Reader: Connected to shared memory at %p\n", shared);

    // 데이터 읽기
    while (1) {
        // 데이터 준비될 때까지 대기
        while (shared->ready == 0) {
            usleep(1000);
        }

        if (shared->counter == -1) {
            printf("Reader: Received termination signal.\n");
            break;
        }

        printf("Reader: Read counter=%d, message='%s'\n",
               shared->counter, shared->message);

        // 읽음 표시
        shared->ready = 0;
    }

    munmap(shared, sizeof(shared_data_t));
    close(fd);

    printf("Reader: Done.\n");
    return 0;
}
```

### 4.3 Python 공유 메모리

```python
from multiprocessing import shared_memory, Process
import numpy as np
import time

def writer_process(shm_name: str, shape: tuple):
    """공유 메모리에 데이터 쓰기"""
    # 기존 공유 메모리에 연결
    existing_shm = shared_memory.SharedMemory(name=shm_name)

    # NumPy 배열로 뷰 생성
    array = np.ndarray(shape, dtype=np.float64, buffer=existing_shm.buf)

    for i in range(5):
        array[:] = np.random.rand(*shape) * 100
        print(f"Writer: Updated array, sum = {array.sum():.2f}")
        time.sleep(1)

    # -1로 종료 신호
    array[:] = -1

    existing_shm.close()
    print("Writer: Done")

def reader_process(shm_name: str, shape: tuple):
    """공유 메모리에서 데이터 읽기"""
    time.sleep(0.5)  # Writer가 먼저 시작하도록

    existing_shm = shared_memory.SharedMemory(name=shm_name)
    array = np.ndarray(shape, dtype=np.float64, buffer=existing_shm.buf)

    while True:
        if np.all(array == -1):
            print("Reader: Received termination signal")
            break

        print(f"Reader: Array sum = {array.sum():.2f}")
        time.sleep(0.5)

    existing_shm.close()
    print("Reader: Done")

def main():
    """공유 메모리 데모"""
    shape = (10, 10)
    size = np.prod(shape) * np.float64().itemsize

    # 공유 메모리 생성
    shm = shared_memory.SharedMemory(create=True, size=size)
    print(f"Created shared memory: {shm.name}")

    # 초기화
    array = np.ndarray(shape, dtype=np.float64, buffer=shm.buf)
    array[:] = 0

    # 프로세스 생성 및 시작
    writer = Process(target=writer_process, args=(shm.name, shape))
    reader = Process(target=reader_process, args=(shm.name, shape))

    writer.start()
    reader.start()

    writer.join()
    reader.join()

    # 정리
    shm.close()
    shm.unlink()
    print("Shared memory cleaned up")

if __name__ == "__main__":
    main()
```

### 4.4 mmap을 이용한 파일 매핑

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>

int main() {
    const char *filepath = "/tmp/mmap_example.dat";
    const size_t filesize = 4096;

    // 파일 생성
    int fd = open(filepath, O_RDWR | O_CREAT | O_TRUNC, 0666);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // 파일 크기 설정
    ftruncate(fd, filesize);

    // 메모리 매핑
    char *mapped = (char*)mmap(NULL, filesize,
                                PROT_READ | PROT_WRITE,
                                MAP_SHARED, fd, 0);
    if (mapped == MAP_FAILED) {
        perror("mmap");
        return 1;
    }

    // 매핑된 메모리에 직접 쓰기
    strcpy(mapped, "Hello, Memory-Mapped File!");
    printf("Written: %s\n", mapped);

    // 변경 사항 동기화
    msync(mapped, filesize, MS_SYNC);

    // 정리
    munmap(mapped, filesize);
    close(fd);

    // 파일 내용 확인
    FILE *f = fopen(filepath, "r");
    char buffer[256];
    fgets(buffer, sizeof(buffer), f);
    printf("File content: %s\n", buffer);
    fclose(f);

    unlink(filepath);
    return 0;
}
```

---

## 5. Semaphore와 Mutex

### 5.1 동기화의 필요성

공유 자원에 대한 경쟁 조건(Race Condition)을 방지합니다.

```
Without Synchronization:

Thread A          Shared Counter         Thread B
   │                   │                    │
   ├── read(10) ◄──────┤                    │
   │                   │                    │
   │                   ├───────► read(10) ──┤
   │                   │                    │
   ├── write(11) ─────►│                    │
   │                   │                    │
   │                   │◄────── write(11) ──┤
   │                   │                    │

Result: 11 (Expected: 12) → RACE CONDITION!
```

### 5.2 Mutex (Mutual Exclusion)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define NUM_THREADS 4
#define ITERATIONS 1000000

// 공유 자원
int counter = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* increment_without_mutex(void* arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        counter++;  // Race condition!
    }
    return NULL;
}

void* increment_with_mutex(void* arg) {
    for (int i = 0; i < ITERATIONS; i++) {
        pthread_mutex_lock(&mutex);
        counter++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];

    // Without Mutex
    counter = 0;
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, increment_without_mutex, NULL);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    printf("Without mutex: counter = %d (expected: %d)\n",
           counter, NUM_THREADS * ITERATIONS);

    // With Mutex
    counter = 0;
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, increment_with_mutex, NULL);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    printf("With mutex:    counter = %d (expected: %d)\n",
           counter, NUM_THREADS * ITERATIONS);

    pthread_mutex_destroy(&mutex);
    return 0;
}
```

### 5.3 Semaphore

```
Counting Semaphore:
- 값이 N인 세마포어 = N개의 자원 사용 가능
- wait() (P 연산): 값 감소, 0이면 블록
- post() (V 연산): 값 증가, 대기 스레드 깨움

Binary Semaphore (Mutex와 유사):
- 값이 0 또는 1
- wait(): 1→0 (잠금), 0이면 블록
- post(): 0→1 (해제)
```

#### POSIX Named Semaphore

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>
#include <fcntl.h>
#include <unistd.h>

#define NUM_THREADS 5
#define MAX_CONCURRENT 2  // 동시 접근 가능한 스레드 수

sem_t *semaphore;

void* worker(void* arg) {
    int id = *(int*)arg;

    printf("Thread %d: Waiting for semaphore...\n", id);

    sem_wait(semaphore);

    printf("Thread %d: Entered critical section\n", id);
    sleep(2);  // 작업 시뮬레이션
    printf("Thread %d: Leaving critical section\n", id);

    sem_post(semaphore);

    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];
    int ids[NUM_THREADS];

    // Named semaphore 생성
    semaphore = sem_open("/my_semaphore", O_CREAT, 0644, MAX_CONCURRENT);
    if (semaphore == SEM_FAILED) {
        perror("sem_open");
        exit(EXIT_FAILURE);
    }

    // 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, worker, &ids[i]);
    }

    // 스레드 종료 대기
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    // 정리
    sem_close(semaphore);
    sem_unlink("/my_semaphore");

    printf("All threads completed.\n");
    return 0;
}
```

### 5.4 생산자-소비자 문제

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define BUFFER_SIZE 5
#define NUM_ITEMS 10

int buffer[BUFFER_SIZE];
int in = 0, out = 0;

sem_t empty;  // 빈 슬롯 수
sem_t full;   // 채워진 슬롯 수
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* producer(void* arg) {
    int id = *(int*)arg;

    for (int i = 0; i < NUM_ITEMS; i++) {
        int item = id * 100 + i;

        sem_wait(&empty);  // 빈 슬롯 대기
        pthread_mutex_lock(&mutex);

        buffer[in] = item;
        printf("Producer %d: Produced %d at index %d\n", id, item, in);
        in = (in + 1) % BUFFER_SIZE;

        pthread_mutex_unlock(&mutex);
        sem_post(&full);  // 채워진 슬롯 증가

        usleep(rand() % 500000);
    }

    return NULL;
}

void* consumer(void* arg) {
    int id = *(int*)arg;

    for (int i = 0; i < NUM_ITEMS; i++) {
        sem_wait(&full);  // 채워진 슬롯 대기
        pthread_mutex_lock(&mutex);

        int item = buffer[out];
        printf("Consumer %d: Consumed %d from index %d\n", id, item, out);
        out = (out + 1) % BUFFER_SIZE;

        pthread_mutex_unlock(&mutex);
        sem_post(&empty);  // 빈 슬롯 증가

        usleep(rand() % 300000);
    }

    return NULL;
}

int main() {
    pthread_t prod_threads[2], cons_threads[2];
    int prod_ids[] = {1, 2};
    int cons_ids[] = {1, 2};

    sem_init(&empty, 0, BUFFER_SIZE);
    sem_init(&full, 0, 0);

    // 생산자, 소비자 스레드 생성
    for (int i = 0; i < 2; i++) {
        pthread_create(&prod_threads[i], NULL, producer, &prod_ids[i]);
        pthread_create(&cons_threads[i], NULL, consumer, &cons_ids[i]);
    }

    // 종료 대기
    for (int i = 0; i < 2; i++) {
        pthread_join(prod_threads[i], NULL);
        pthread_join(cons_threads[i], NULL);
    }

    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&mutex);

    printf("All done.\n");
    return 0;
}
```

### 5.5 Python threading 동기화

```python
import threading
import time
import random
from queue import Queue

class BoundedBuffer:
    """세마포어를 이용한 Bounded Buffer"""

    def __init__(self, size: int):
        self.buffer = []
        self.size = size
        self.mutex = threading.Lock()
        self.empty = threading.Semaphore(size)  # 빈 슬롯
        self.full = threading.Semaphore(0)      # 채워진 슬롯

    def produce(self, item):
        self.empty.acquire()  # 빈 슬롯 대기
        with self.mutex:
            self.buffer.append(item)
            print(f"Produced: {item}, Buffer: {self.buffer}")
        self.full.release()   # 채워진 슬롯 증가

    def consume(self):
        self.full.acquire()   # 채워진 슬롯 대기
        with self.mutex:
            item = self.buffer.pop(0)
            print(f"Consumed: {item}, Buffer: {self.buffer}")
        self.empty.release()  # 빈 슬롯 증가
        return item

def producer(buffer: BoundedBuffer, producer_id: int, num_items: int):
    for i in range(num_items):
        item = f"P{producer_id}-{i}"
        buffer.produce(item)
        time.sleep(random.uniform(0.1, 0.5))

def consumer(buffer: BoundedBuffer, consumer_id: int, num_items: int):
    for i in range(num_items):
        item = buffer.consume()
        time.sleep(random.uniform(0.05, 0.3))

def main():
    buffer = BoundedBuffer(5)

    producers = [
        threading.Thread(target=producer, args=(buffer, i, 5))
        for i in range(2)
    ]
    consumers = [
        threading.Thread(target=consumer, args=(buffer, i, 5))
        for i in range(2)
    ]

    for t in producers + consumers:
        t.start()

    for t in producers + consumers:
        t.join()

    print("All threads completed")

if __name__ == "__main__":
    main()
```

### 5.6 Mutex vs Semaphore vs Spinlock

| 특징 | Mutex | Semaphore | Spinlock |
|------|-------|-----------|----------|
| **소유권** | 있음 (잠근 스레드만 해제) | 없음 | 없음 |
| **값 범위** | 0 또는 1 | 0 ~ N | 0 또는 1 |
| **대기 방식** | 블로킹 (sleep) | 블로킹 (sleep) | 바쁜 대기 (busy-wait) |
| **용도** | 상호 배제 | 자원 카운팅, 시그널링 | 짧은 임계 구역 |
| **오버헤드** | 중간 | 중간 | 낮음 (짧을 때) |
| **컨텍스트 스위치** | 발생 가능 | 발생 가능 | 없음 |

---

## 6. Socket

### 6.1 소켓 개요

네트워크 또는 로컬에서 프로세스 간 양방향 통신을 제공합니다.

```
                    ┌─────────────┐
                    │   Socket    │
                    │   Domain    │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
     ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
     │ AF_INET   │  │ AF_INET6  │  │ AF_UNIX   │
     │ (IPv4)    │  │ (IPv6)    │  │ (Local)   │
     └───────────┘  └───────────┘  └───────────┘

                    ┌─────────────┐
                    │   Socket    │
                    │    Type     │
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
     ┌─────▼─────┐  ┌─────▼─────┐  ┌─────▼─────┐
     │SOCK_STREAM│  │SOCK_DGRAM │  │ SOCK_RAW  │
     │  (TCP)    │  │  (UDP)    │  │ (Raw IP)  │
     └───────────┘  └───────────┘  └───────────┘
```

### 6.2 TCP 서버/클라이언트 (C)

#### TCP 서버

```c
// tcp_server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8080
#define BUFFER_SIZE 1024
#define MAX_CLIENTS 5

int main() {
    int server_fd, client_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    char buffer[BUFFER_SIZE];

    // 소켓 생성
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // SO_REUSEADDR 설정 (포트 재사용)
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 주소 설정
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    // 바인드
    if (bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    // 리슨
    if (listen(server_fd, MAX_CLIENTS) == -1) {
        perror("listen");
        exit(EXIT_FAILURE);
    }

    printf("Server listening on port %d...\n", PORT);

    // 연결 수락 및 처리
    while (1) {
        client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_len);
        if (client_fd == -1) {
            perror("accept");
            continue;
        }

        printf("Client connected: %s:%d\n",
               inet_ntoa(client_addr.sin_addr),
               ntohs(client_addr.sin_port));

        // Echo 서버
        ssize_t bytes_read;
        while ((bytes_read = recv(client_fd, buffer, BUFFER_SIZE - 1, 0)) > 0) {
            buffer[bytes_read] = '\0';
            printf("Received: %s", buffer);

            // 에코 응답
            send(client_fd, buffer, bytes_read, 0);

            if (strncmp(buffer, "quit", 4) == 0) {
                break;
            }
        }

        close(client_fd);
        printf("Client disconnected.\n");
    }

    close(server_fd);
    return 0;
}
```

#### TCP 클라이언트

```c
// tcp_client.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8080
#define BUFFER_SIZE 1024

int main() {
    int sock_fd;
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];

    // 소켓 생성
    sock_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (sock_fd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // 서버 주소 설정
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

    // 연결
    if (connect(sock_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("connect");
        exit(EXIT_FAILURE);
    }

    printf("Connected to server.\n");

    // 메시지 전송 및 수신
    while (fgets(buffer, BUFFER_SIZE, stdin) != NULL) {
        send(sock_fd, buffer, strlen(buffer), 0);

        ssize_t bytes_read = recv(sock_fd, buffer, BUFFER_SIZE - 1, 0);
        if (bytes_read > 0) {
            buffer[bytes_read] = '\0';
            printf("Echo: %s", buffer);
        }

        if (strncmp(buffer, "quit", 4) == 0) {
            break;
        }
    }

    close(sock_fd);
    printf("Disconnected.\n");
    return 0;
}
```

### 6.3 Unix Domain Socket

로컬 프로세스 간 통신에 최적화된 소켓입니다.

```c
// unix_socket_server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define SOCKET_PATH "/tmp/my_unix_socket"
#define BUFFER_SIZE 256

int main() {
    int server_fd, client_fd;
    struct sockaddr_un server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    char buffer[BUFFER_SIZE];

    // 이전 소켓 파일 삭제
    unlink(SOCKET_PATH);

    // 소켓 생성
    server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // 주소 설정
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sun_family = AF_UNIX;
    strncpy(server_addr.sun_path, SOCKET_PATH, sizeof(server_addr.sun_path) - 1);

    // 바인드
    if (bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) == -1) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    // 리슨
    listen(server_fd, 5);
    printf("Unix socket server listening at %s\n", SOCKET_PATH);

    // 연결 수락
    client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_len);
    if (client_fd == -1) {
        perror("accept");
        exit(EXIT_FAILURE);
    }

    printf("Client connected.\n");

    // 통신
    ssize_t bytes_read;
    while ((bytes_read = recv(client_fd, buffer, BUFFER_SIZE - 1, 0)) > 0) {
        buffer[bytes_read] = '\0';
        printf("Received: %s", buffer);
        send(client_fd, buffer, bytes_read, 0);
    }

    close(client_fd);
    close(server_fd);
    unlink(SOCKET_PATH);

    return 0;
}
```

### 6.4 Python 소켓 서버/클라이언트

```python
import socket
import threading

def handle_client(client_socket: socket.socket, address: tuple):
    """클라이언트 처리 핸들러"""
    print(f"Connected: {address}")

    try:
        while True:
            data = client_socket.recv(1024)
            if not data:
                break

            message = data.decode('utf-8')
            print(f"Received from {address}: {message}")

            # 에코 응답
            client_socket.send(data)

            if message.strip() == 'quit':
                break

    except Exception as e:
        print(f"Error handling {address}: {e}")

    finally:
        client_socket.close()
        print(f"Disconnected: {address}")

def tcp_server(host: str = '127.0.0.1', port: int = 8080):
    """멀티스레드 TCP 서버"""
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    server.bind((host, port))
    server.listen(5)
    print(f"Server listening on {host}:{port}")

    try:
        while True:
            client_socket, address = server.accept()
            thread = threading.Thread(
                target=handle_client,
                args=(client_socket, address)
            )
            thread.start()

    except KeyboardInterrupt:
        print("\nShutting down server...")

    finally:
        server.close()

def tcp_client(host: str = '127.0.0.1', port: int = 8080):
    """TCP 클라이언트"""
    client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    client.connect((host, port))
    print(f"Connected to {host}:{port}")

    try:
        while True:
            message = input("Enter message: ")
            client.send(message.encode('utf-8'))

            response = client.recv(1024).decode('utf-8')
            print(f"Echo: {response}")

            if message == 'quit':
                break

    except KeyboardInterrupt:
        pass

    finally:
        client.close()
        print("Disconnected")

if __name__ == "__main__":
    import sys

    if len(sys.argv) > 1 and sys.argv[1] == 'client':
        tcp_client()
    else:
        tcp_server()
```

### 6.5 소켓 옵션

```c
#include <sys/socket.h>

// 주요 소켓 옵션

// 1. SO_REUSEADDR: TIME_WAIT 상태의 포트 재사용
int opt = 1;
setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

// 2. SO_REUSEPORT: 여러 프로세스가 같은 포트 바인드 (로드밸런싱)
setsockopt(sock, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));

// 3. SO_KEEPALIVE: TCP Keepalive 활성화
setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));

// 4. SO_RCVBUF / SO_SNDBUF: 버퍼 크기 설정
int bufsize = 65536;
setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &bufsize, sizeof(bufsize));
setsockopt(sock, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));

// 5. TCP_NODELAY: Nagle 알고리즘 비활성화 (저지연)
int flag = 1;
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));

// 6. SO_LINGER: close() 동작 제어
struct linger ling = {.l_onoff = 1, .l_linger = 30};  // 30초 대기
setsockopt(sock, SOL_SOCKET, SO_LINGER, &ling, sizeof(ling));
```

---

## 8. 참고 자료

### 공식 문서

- [POSIX.1-2017 (The Open Group)](https://pubs.opengroup.org/onlinepubs/9699919799/)
- [Linux Programmer's Manual](https://man7.org/linux/man-pages/)
- [OSTEP - Concurrency](https://pages.cs.wisc.edu/~remzi/OSTEP/)

### 추가 학습 자료

- "Advanced Programming in the UNIX Environment" - W. Richard Stevens
- "The Linux Programming Interface" - Michael Kerrisk
- [Beej's Guide to Unix IPC](https://beej.us/guide/bgipc/)

### 시스템 명령어

```bash
# IPC 자원 확인
ipcs          # 모든 IPC 자원
ipcs -m       # 공유 메모리
ipcs -q       # 메시지 큐
ipcs -s       # 세마포어

# IPC 자원 삭제
ipcrm -m <shmid>  # 공유 메모리 삭제
ipcrm -q <msqid>  # 메시지 큐 삭제
ipcrm -s <semid>  # 세마포어 삭제

# 열린 파일/소켓 확인
lsof -p <pid>
lsof -i :8080

# 소켓 상태 확인
ss -tuln
netstat -tuln
```

---

> **문서 작성일**: 2024년 12월
>
> **참고**: 이 문서는 POSIX.1-2017, Linux 커널 문서, OSTEP을 기반으로 작성되었습니다.

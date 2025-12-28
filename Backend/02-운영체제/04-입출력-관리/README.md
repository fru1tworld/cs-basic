# 입출력 관리 (I/O Management)

> **OSTEP(Operating Systems: Three Easy Pieces)** 및 **POSIX.1-2017** 기반

---

## 목차

1. [I/O 개요](#1-io-개요)
2. [Blocking vs Non-Blocking I/O](#2-blocking-vs-non-blocking-io)
3. [Synchronous vs Asynchronous I/O](#3-synchronous-vs-asynchronous-io)
4. [I/O Multiplexing](#4-io-multiplexing)
5. [DMA (Direct Memory Access)](#5-dma-direct-memory-access)
7. [참고 자료](#7-참고-자료)

---

## 1. I/O 개요

### 1.1 I/O란?

I/O(Input/Output)는 컴퓨터와 외부 장치(디스크, 네트워크, 터미널 등) 간의 데이터 전송을 의미합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                     Application                              │
└─────────────────────────────────────────────────────────────┘
                              │
                         System Call
                              │
┌─────────────────────────────────────────────────────────────┐
│                     Kernel (VFS)                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │   ext4   │  │   NFS    │  │  socket  │  │  device  │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                        Device Driver
                              │
┌─────────────────────────────────────────────────────────────┐
│                     Hardware                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │   SSD    │  │   NIC    │  │  UART    │  │   GPU    │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 I/O의 특성

| 특성 | 설명 |
|------|------|
| **속도 격차** | CPU: ~ns, RAM: ~ns, SSD: ~us, HDD: ~ms, Network: ~ms |
| **예측 불가능** | 네트워크 지연, 디스크 seek 시간 가변 |
| **비용** | I/O 대기 동안 CPU 유휴 상태 |

### 1.3 I/O 처리 방식 분류

```
                    I/O 처리 방식
                          │
         ┌────────────────┼────────────────┐
         │                │                │
    ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
    │  동기   │     │  비동기  │     │   I/O   │
    │ (Sync)  │     │ (Async) │     │Multiplex│
    └────┬────┘     └────┬────┘     └────┬────┘
         │               │               │
    ┌────┴────┐     ┌────┴────┐     ┌────┴────────────┐
    │         │     │         │     │                 │
┌───▼───┐ ┌───▼───┐ │         │  ┌──▼──┐ ┌──▼──┐ ┌───▼───┐
│Block- │ │Non-   │ │  AIO    │  │select│ │poll │ │epoll/ │
│ ing   │ │Block  │ │io_uring │  │      │ │     │ │kqueue │
└───────┘ └───────┘ └─────────┘  └──────┘ └─────┘ └───────┘
```

---

## 2. Blocking vs Non-Blocking I/O

### 2.1 Blocking I/O

호출한 함수가 완료될 때까지 스레드가 대기(블록)합니다.

```
Application         Kernel          Hardware
    │                 │                │
    │   read()        │                │
    ├────────────────►│                │
    │                 │   DMA Read     │
    │   [BLOCKED]     ├───────────────►│
    │                 │                │
    │                 │◄───────────────┤
    │                 │   Data Ready   │
    │◄────────────────┤                │
    │   Data          │                │
    │                 │                │
    ▼ (계속 실행)      │                │
```

#### C 예제: Blocking I/O

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    char buffer[1024];
    ssize_t bytes_read;

    // Blocking read (기본값)
    printf("Waiting for input (blocking)...\n");
    bytes_read = read(STDIN_FILENO, buffer, sizeof(buffer));

    if (bytes_read > 0) {
        buffer[bytes_read] = '\0';
        printf("Read %zd bytes: %s", bytes_read, buffer);
    }

    return 0;
}
```

### 2.2 Non-Blocking I/O

호출한 함수가 즉시 반환하며, 데이터가 없으면 에러(EAGAIN/EWOULDBLOCK)를 반환합니다.

```
Application         Kernel          Hardware
    │                 │                │
    │   read()        │                │
    ├────────────────►│                │
    │◄────────────────┤ EAGAIN         │
    │   (즉시 반환)    │                │
    │                 │   DMA Read     │
    │   read()        ├───────────────►│
    ├────────────────►│                │
    │◄────────────────┤ EAGAIN         │
    │                 │◄───────────────┤
    │   read()        │   Data Ready   │
    ├────────────────►│                │
    │◄────────────────┤                │
    │   Data          │                │
    ▼                 │                │
```

#### C 예제: Non-Blocking I/O

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

int main() {
    char buffer[1024];
    ssize_t bytes_read;

    // stdin을 Non-blocking으로 설정
    int flags = fcntl(STDIN_FILENO, F_GETFL, 0);
    fcntl(STDIN_FILENO, F_SETFL, flags | O_NONBLOCK);

    printf("Non-blocking read (polling)...\n");

    int attempts = 0;
    while (1) {
        bytes_read = read(STDIN_FILENO, buffer, sizeof(buffer));

        if (bytes_read > 0) {
            buffer[bytes_read] = '\0';
            printf("Read %zd bytes: %s", bytes_read, buffer);
            break;
        } else if (bytes_read == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 데이터 없음 - 다른 작업 수행 가능
                printf("Attempt %d: No data available\n", ++attempts);
                usleep(500000);  // 0.5초 대기 (busy waiting 방지)
                continue;
            } else {
                perror("read");
                break;
            }
        } else {
            // EOF
            printf("EOF\n");
            break;
        }
    }

    // 원래 플래그로 복원
    fcntl(STDIN_FILENO, F_SETFL, flags);

    return 0;
}
```

### 2.3 Python 예제: Non-Blocking Socket

```python
import socket
import errno
import time

def non_blocking_client():
    """Non-blocking 소켓 클라이언트 예제"""

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setblocking(False)  # Non-blocking 모드

    try:
        sock.connect(('127.0.0.1', 8080))
    except BlockingIOError:
        # Non-blocking connect는 즉시 반환
        pass

    # 연결 완료 대기 (polling)
    while True:
        try:
            sock.send(b"Hello")
            print("Connected and sent data!")
            break
        except BlockingIOError:
            print("Connection in progress...")
            time.sleep(0.1)
        except Exception as e:
            print(f"Error: {e}")
            break

    # 응답 수신 (non-blocking)
    while True:
        try:
            data = sock.recv(1024)
            if data:
                print(f"Received: {data.decode()}")
            break
        except BlockingIOError:
            print("No data yet...")
            time.sleep(0.1)

    sock.close()

def non_blocking_server():
    """Non-blocking 소켓 서버 예제"""

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.setblocking(False)

    server.bind(('127.0.0.1', 8080))
    server.listen(5)
    print("Server listening (non-blocking)...")

    clients = []

    while True:
        # 새 연결 수락 시도
        try:
            client, addr = server.accept()
            client.setblocking(False)
            clients.append(client)
            print(f"New connection from {addr}")
        except BlockingIOError:
            pass  # 새 연결 없음

        # 기존 클라이언트 처리
        for client in clients[:]:
            try:
                data = client.recv(1024)
                if data:
                    print(f"Received: {data.decode()}")
                    client.send(data)  # Echo
                else:
                    # 연결 종료
                    clients.remove(client)
                    client.close()
            except BlockingIOError:
                pass  # 데이터 없음

        time.sleep(0.01)  # CPU 사용률 감소

if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1 and sys.argv[1] == 'client':
        non_blocking_client()
    else:
        non_blocking_server()
```

### 2.4 Blocking vs Non-Blocking 비교

| 특성 | Blocking | Non-Blocking |
|------|----------|--------------|
| **호출 반환** | 완료 후 | 즉시 |
| **CPU 사용** | 대기 중 유휴 | Polling 시 CPU 사용 |
| **프로그래밍 복잡도** | 간단 | 복잡 |
| **동시성** | 멀티스레드 필요 | 단일 스레드 가능 |
| **적합한 경우** | 간단한 I/O | 다중 I/O 처리 |

---

## 3. Synchronous vs Asynchronous I/O

### 3.1 개념 정리

```
Blocking/Non-Blocking: 호출자의 대기 여부
Synchronous/Asynchronous: 완료 알림 방식

┌─────────────────┬────────────────────────────────────────────┐
│                 │           완료 알림 방식                    │
│                 ├───────────────────┬────────────────────────┤
│                 │    Synchronous    │    Asynchronous        │
├─────────────────┼───────────────────┼────────────────────────┤
│                 │                   │                        │
│    Blocking     │   전통적 I/O      │   (일반적이지 않음)     │
│                 │   read(), write() │                        │
│                 │                   │                        │
├─────────────────┼───────────────────┼────────────────────────┤
│                 │                   │                        │
│   Non-Blocking  │   O_NONBLOCK      │   AIO, io_uring        │
│                 │   + polling       │   + 콜백/시그널         │
│                 │                   │                        │
└─────────────────┴───────────────────┴────────────────────────┘
```

### 3.2 동기 I/O (Synchronous I/O)

I/O 완료를 애플리케이션이 직접 확인합니다.

```c
// 동기 + 블로킹 (가장 일반적)
ssize_t n = read(fd, buffer, size);  // 완료까지 대기
if (n > 0) {
    // 데이터 처리
}

// 동기 + 논블로킹
fcntl(fd, F_SETFL, O_NONBLOCK);
while (1) {
    ssize_t n = read(fd, buffer, size);
    if (n > 0) {
        // 데이터 처리
        break;
    } else if (errno == EAGAIN) {
        // 다른 작업 수행
        do_something_else();
    }
}
```

### 3.3 비동기 I/O (Asynchronous I/O)

I/O 완료를 커널이 알려줍니다.

```
Application         Kernel          Hardware
    │                 │                │
    │   aio_read()    │                │
    ├────────────────►│                │
    │◄────────────────┤ (즉시 반환)    │
    │                 │   DMA Read     │
    │   다른 작업 수행 ├───────────────►│
    │                 │                │
    │                 │◄───────────────┤
    │   시그널/콜백    │   Data Ready   │
    │◄════════════════╡                │
    │   데이터 사용    │                │
    ▼                 │                │
```

#### POSIX AIO 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <aio.h>
#include <errno.h>
#include <signal.h>

#define BUFFER_SIZE 1024

volatile sig_atomic_t aio_done = 0;

void aio_completion_handler(int sig, siginfo_t *si, void *context) {
    struct aiocb *req = (struct aiocb *)si->si_value.sival_ptr;

    // 에러 확인
    if (aio_error(req) == 0) {
        ssize_t ret = aio_return(req);
        printf("AIO completed: %zd bytes read\n", ret);
        printf("Data: %.*s\n", (int)ret, (char*)req->aio_buf);
    } else {
        printf("AIO error: %s\n", strerror(aio_error(req)));
    }

    aio_done = 1;
}

int main() {
    struct aiocb cb;
    char buffer[BUFFER_SIZE];

    // 시그널 핸들러 설정
    struct sigaction sa;
    sa.sa_flags = SA_SIGINFO;
    sa.sa_sigaction = aio_completion_handler;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGIO, &sa, NULL);

    // 파일 열기
    int fd = open("/etc/passwd", O_RDONLY);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    // AIO 요청 설정
    memset(&cb, 0, sizeof(cb));
    cb.aio_fildes = fd;
    cb.aio_buf = buffer;
    cb.aio_nbytes = BUFFER_SIZE - 1;
    cb.aio_offset = 0;

    // 시그널 알림 설정
    cb.aio_sigevent.sigev_notify = SIGEV_SIGNAL;
    cb.aio_sigevent.sigev_signo = SIGIO;
    cb.aio_sigevent.sigev_value.sival_ptr = &cb;

    // 비동기 읽기 시작
    if (aio_read(&cb) == -1) {
        perror("aio_read");
        close(fd);
        return 1;
    }

    printf("AIO request submitted. Doing other work...\n");

    // 다른 작업 수행
    while (!aio_done) {
        printf("Working...\n");
        usleep(100000);  // 100ms
    }

    close(fd);
    return 0;
}
```

### 3.4 io_uring (Linux 5.1+)

Linux의 최신 비동기 I/O 인터페이스입니다.

```
io_uring 구조:

User Space          Shared Memory          Kernel Space
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ Application  │   │              │   │   Kernel     │
│              │   │   Submission │   │              │
│ io_uring_    │   │     Queue    │   │  io_uring    │
│  prep_*()   ─┼──►│   (SQ Ring)  │──►│  worker      │
│              │   │              │   │              │
│ io_uring_    │   │  Completion  │   │              │
│  wait_cqe() ◄┼───│     Queue    │◄──│              │
│              │   │   (CQ Ring)  │   │              │
└──────────────┘   └──────────────┘   └──────────────┘

특징:
- Ring Buffer 기반 → System Call 최소화
- Batch 처리 가능
- Polling 모드 지원 (SQPOLL)
- Zero-copy 지원
```

#### io_uring 예제 (liburing)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <liburing.h>

#define BUFFER_SIZE 4096
#define QUEUE_DEPTH 8

int main() {
    struct io_uring ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    char buffer[BUFFER_SIZE];
    int fd;

    // io_uring 초기화
    if (io_uring_queue_init(QUEUE_DEPTH, &ring, 0) < 0) {
        perror("io_uring_queue_init");
        return 1;
    }

    // 파일 열기
    fd = open("/etc/passwd", O_RDONLY);
    if (fd < 0) {
        perror("open");
        io_uring_queue_exit(&ring);
        return 1;
    }

    // Submission Queue Entry 가져오기
    sqe = io_uring_get_sqe(&ring);
    if (!sqe) {
        fprintf(stderr, "Could not get SQE\n");
        close(fd);
        io_uring_queue_exit(&ring);
        return 1;
    }

    // 읽기 요청 준비
    io_uring_prep_read(sqe, fd, buffer, BUFFER_SIZE - 1, 0);
    sqe->user_data = 1;  // 요청 식별자

    // 요청 제출
    io_uring_submit(&ring);

    printf("Request submitted. Doing other work...\n");

    // 완료 대기
    if (io_uring_wait_cqe(&ring, &cqe) < 0) {
        perror("io_uring_wait_cqe");
    } else {
        if (cqe->res < 0) {
            fprintf(stderr, "Read error: %s\n", strerror(-cqe->res));
        } else {
            buffer[cqe->res] = '\0';
            printf("Read %d bytes:\n%s\n", cqe->res, buffer);
        }
        io_uring_cqe_seen(&ring, cqe);
    }

    close(fd);
    io_uring_queue_exit(&ring);
    return 0;
}
```

### 3.5 Python asyncio

```python
import asyncio

async def read_file_async(filepath: str) -> str:
    """비동기 파일 읽기 (실제로는 스레드 풀 사용)"""
    loop = asyncio.get_event_loop()
    with open(filepath, 'r') as f:
        # run_in_executor로 블로킹 I/O를 비동기로 실행
        content = await loop.run_in_executor(None, f.read)
    return content

async def fetch_url(url: str) -> str:
    """비동기 HTTP 요청"""
    import aiohttp

    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

async def main():
    """비동기 I/O 데모"""

    # 여러 작업을 동시에 실행
    tasks = [
        read_file_async('/etc/passwd'),
        asyncio.sleep(1),  # 비동기 대기
    ]

    results = await asyncio.gather(*tasks)

    print(f"File content (first 100 chars): {results[0][:100]}...")
    print("Sleep completed")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 4. I/O Multiplexing

### 4.1 I/O Multiplexing 개념

하나의 스레드로 여러 I/O 작업을 동시에 처리합니다.

```
전통적 방식 (Thread per Connection):
┌──────────────┐
│   Thread 1   │ ─── Client 1
├──────────────┤
│   Thread 2   │ ─── Client 2
├──────────────┤
│   Thread 3   │ ─── Client 3
└──────────────┘
문제: 스레드 생성 비용, 컨텍스트 스위칭, 메모리 사용

I/O Multiplexing:
┌──────────────┐
│              │ ─── Client 1
│  Single      │ ─── Client 2
│  Thread      │ ─── Client 3
│              │ ─── Client 4
└──────────────┘
     │
     ▼
┌──────────────┐
│ select/poll  │
│ epoll/kqueue │
└──────────────┘
```

### 4.2 select()

가장 오래된 I/O Multiplexing 메커니즘입니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/select.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <errno.h>

#define PORT 8080
#define MAX_CLIENTS 10
#define BUFFER_SIZE 1024

int main() {
    int server_fd, client_fds[MAX_CLIENTS];
    fd_set read_fds, master_fds;
    int max_fd;
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];

    // 클라이언트 배열 초기화
    for (int i = 0; i < MAX_CLIENTS; i++) {
        client_fds[i] = -1;
    }

    // 서버 소켓 생성
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
    listen(server_fd, MAX_CLIENTS);

    printf("Server listening on port %d (using select)\n", PORT);

    // fd_set 초기화
    FD_ZERO(&master_fds);
    FD_SET(server_fd, &master_fds);
    max_fd = server_fd;

    while (1) {
        read_fds = master_fds;  // select가 수정하므로 복사

        // I/O 이벤트 대기
        int activity = select(max_fd + 1, &read_fds, NULL, NULL, NULL);

        if (activity < 0 && errno != EINTR) {
            perror("select");
            break;
        }

        // 새 연결 확인
        if (FD_ISSET(server_fd, &read_fds)) {
            struct sockaddr_in client_addr;
            socklen_t client_len = sizeof(client_addr);
            int new_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_len);

            if (new_fd >= 0) {
                printf("New connection: fd=%d\n", new_fd);

                // 클라이언트 배열에 추가
                for (int i = 0; i < MAX_CLIENTS; i++) {
                    if (client_fds[i] == -1) {
                        client_fds[i] = new_fd;
                        FD_SET(new_fd, &master_fds);
                        if (new_fd > max_fd) max_fd = new_fd;
                        break;
                    }
                }
            }
        }

        // 클라이언트 이벤트 처리
        for (int i = 0; i < MAX_CLIENTS; i++) {
            int fd = client_fds[i];
            if (fd == -1) continue;

            if (FD_ISSET(fd, &read_fds)) {
                ssize_t bytes = read(fd, buffer, BUFFER_SIZE - 1);

                if (bytes <= 0) {
                    // 연결 종료 또는 에러
                    printf("Client disconnected: fd=%d\n", fd);
                    close(fd);
                    FD_CLR(fd, &master_fds);
                    client_fds[i] = -1;
                } else {
                    buffer[bytes] = '\0';
                    printf("Received from fd=%d: %s", fd, buffer);

                    // Echo
                    write(fd, buffer, bytes);
                }
            }
        }
    }

    close(server_fd);
    return 0;
}
```

#### select() 한계

- **FD 개수 제한**: FD_SETSIZE (보통 1024)
- **매번 fd_set 복사**: O(n) 복잡도
- **선형 검색**: 어떤 fd가 ready인지 확인하려면 전체 순회

### 4.3 poll()

select의 FD 개수 제한을 해결합니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <poll.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_CLIENTS 1000
#define BUFFER_SIZE 1024

int main() {
    int server_fd;
    struct pollfd fds[MAX_CLIENTS + 1];
    int nfds = 1;
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];

    // 서버 소켓 생성 및 설정
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
    listen(server_fd, MAX_CLIENTS);

    printf("Server listening on port %d (using poll)\n", PORT);

    // pollfd 배열 초기화
    fds[0].fd = server_fd;
    fds[0].events = POLLIN;

    while (1) {
        int activity = poll(fds, nfds, -1);  // 무한 대기

        if (activity < 0) {
            perror("poll");
            break;
        }

        // 새 연결 확인
        if (fds[0].revents & POLLIN) {
            struct sockaddr_in client_addr;
            socklen_t client_len = sizeof(client_addr);
            int new_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_len);

            if (new_fd >= 0 && nfds < MAX_CLIENTS + 1) {
                printf("New connection: fd=%d\n", new_fd);
                fds[nfds].fd = new_fd;
                fds[nfds].events = POLLIN;
                nfds++;
            }
        }

        // 클라이언트 이벤트 처리
        for (int i = 1; i < nfds; i++) {
            if (fds[i].revents & POLLIN) {
                ssize_t bytes = read(fds[i].fd, buffer, BUFFER_SIZE - 1);

                if (bytes <= 0) {
                    printf("Client disconnected: fd=%d\n", fds[i].fd);
                    close(fds[i].fd);

                    // 배열 압축
                    fds[i] = fds[nfds - 1];
                    nfds--;
                    i--;  // 현재 위치 다시 확인
                } else {
                    buffer[bytes] = '\0';
                    printf("Received from fd=%d: %s", fds[i].fd, buffer);
                    write(fds[i].fd, buffer, bytes);
                }
            }
        }
    }

    close(server_fd);
    return 0;
}
```

### 4.4 epoll (Linux)

Linux의 고성능 I/O Multiplexing 메커니즘입니다.

```
epoll 구조:

User Space                    Kernel Space
┌──────────────┐             ┌──────────────────────────┐
│              │             │                          │
│ epoll_create ├────────────►│   epoll instance         │
│              │             │   ┌──────────────────┐   │
│ epoll_ctl    ├────────────►│   │  Red-Black Tree  │   │
│ (ADD/MOD/DEL)│             │   │  (관심 FD 목록)   │   │
│              │             │   └──────────────────┘   │
│ epoll_wait   │◄────────────┤   ┌──────────────────┐   │
│              │             │   │  Ready List      │   │
│              │             │   │  (이벤트 발생 FD) │   │
│              │             │   └──────────────────┘   │
└──────────────┘             └──────────────────────────┘

특징:
- O(1) 이벤트 알림 (Ready List 반환)
- FD 개수 제한 없음
- Edge/Level Triggered 모드 지원
```

#### epoll 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_EVENTS 100
#define BUFFER_SIZE 1024

// Non-blocking 설정
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int server_fd, epoll_fd;
    struct epoll_event ev, events[MAX_EVENTS];
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];

    // 서버 소켓 생성
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    set_nonblocking(server_fd);

    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
    listen(server_fd, SOMAXCONN);

    printf("Server listening on port %d (using epoll)\n", PORT);

    // epoll 인스턴스 생성
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    // 서버 소켓 등록
    ev.events = EPOLLIN;
    ev.data.fd = server_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, server_fd, &ev);

    while (1) {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);

        if (nfds == -1) {
            if (errno == EINTR) continue;
            perror("epoll_wait");
            break;
        }

        for (int i = 0; i < nfds; i++) {
            int fd = events[i].data.fd;

            if (fd == server_fd) {
                // 새 연결
                while (1) {
                    struct sockaddr_in client_addr;
                    socklen_t client_len = sizeof(client_addr);
                    int client_fd = accept(server_fd,
                                           (struct sockaddr*)&client_addr,
                                           &client_len);

                    if (client_fd == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            break;  // 모든 연결 처리됨
                        }
                        perror("accept");
                        break;
                    }

                    printf("New connection: fd=%d\n", client_fd);
                    set_nonblocking(client_fd);

                    // Edge Triggered 모드로 등록
                    ev.events = EPOLLIN | EPOLLET;
                    ev.data.fd = client_fd;
                    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);
                }

            } else if (events[i].events & EPOLLIN) {
                // 데이터 수신 (Edge Triggered이므로 모두 읽어야 함)
                while (1) {
                    ssize_t bytes = read(fd, buffer, BUFFER_SIZE - 1);

                    if (bytes <= 0) {
                        if (bytes == 0 || (errno != EAGAIN && errno != EWOULDBLOCK)) {
                            printf("Client disconnected: fd=%d\n", fd);
                            epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, NULL);
                            close(fd);
                        }
                        break;
                    }

                    buffer[bytes] = '\0';
                    printf("Received from fd=%d: %s", fd, buffer);
                    write(fd, buffer, bytes);  // Echo
                }
            }
        }
    }

    close(epoll_fd);
    close(server_fd);
    return 0;
}
```

### 4.5 kqueue (BSD/macOS)

BSD 계열 시스템의 고성능 I/O Multiplexing입니다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/event.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define PORT 8080
#define MAX_EVENTS 100
#define BUFFER_SIZE 1024

void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int server_fd, kq;
    struct kevent change_event, events[MAX_EVENTS];
    struct sockaddr_in server_addr;
    char buffer[BUFFER_SIZE];

    // 서버 소켓 생성
    server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    set_nonblocking(server_fd);

    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    bind(server_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
    listen(server_fd, SOMAXCONN);

    printf("Server listening on port %d (using kqueue)\n", PORT);

    // kqueue 생성
    kq = kqueue();
    if (kq == -1) {
        perror("kqueue");
        exit(EXIT_FAILURE);
    }

    // 서버 소켓 등록
    EV_SET(&change_event, server_fd, EVFILT_READ, EV_ADD | EV_ENABLE, 0, 0, NULL);
    kevent(kq, &change_event, 1, NULL, 0, NULL);

    while (1) {
        int nev = kevent(kq, NULL, 0, events, MAX_EVENTS, NULL);

        if (nev == -1) {
            perror("kevent");
            break;
        }

        for (int i = 0; i < nev; i++) {
            int fd = (int)events[i].ident;

            if (events[i].flags & EV_EOF) {
                printf("Client disconnected: fd=%d\n", fd);
                close(fd);
                continue;
            }

            if (fd == server_fd) {
                // 새 연결
                struct sockaddr_in client_addr;
                socklen_t client_len = sizeof(client_addr);
                int client_fd = accept(server_fd,
                                       (struct sockaddr*)&client_addr,
                                       &client_len);

                if (client_fd >= 0) {
                    printf("New connection: fd=%d\n", client_fd);
                    set_nonblocking(client_fd);

                    EV_SET(&change_event, client_fd, EVFILT_READ,
                           EV_ADD | EV_ENABLE, 0, 0, NULL);
                    kevent(kq, &change_event, 1, NULL, 0, NULL);
                }

            } else if (events[i].filter == EVFILT_READ) {
                // 데이터 수신
                ssize_t bytes = read(fd, buffer, BUFFER_SIZE - 1);

                if (bytes > 0) {
                    buffer[bytes] = '\0';
                    printf("Received from fd=%d: %s", fd, buffer);
                    write(fd, buffer, bytes);
                } else {
                    close(fd);
                }
            }
        }
    }

    close(kq);
    close(server_fd);
    return 0;
}
```

### 4.6 I/O Multiplexing 비교

| 특성 | select | poll | epoll | kqueue |
|------|--------|------|-------|--------|
| **FD 제한** | FD_SETSIZE (1024) | 없음 | 없음 | 없음 |
| **이벤트 전달** | 전체 복사 | 전체 복사 | Ready List만 | Ready List만 |
| **복잡도** | O(n) | O(n) | O(1) | O(1) |
| **Edge Trigger** | 없음 | 없음 | 지원 | 지원 |
| **지원 OS** | 모든 POSIX | 모든 POSIX | Linux | BSD/macOS |

### 4.7 Python selectors 모듈

```python
import selectors
import socket

def create_server_socket(host: str, port: int) -> socket.socket:
    """서버 소켓 생성"""
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.setblocking(False)
    server.bind((host, port))
    server.listen(100)
    return server

def accept_connection(sel: selectors.BaseSelector, server: socket.socket):
    """새 연결 수락"""
    client, addr = server.accept()
    print(f"New connection from {addr}")
    client.setblocking(False)
    sel.register(client, selectors.EVENT_READ, handle_client)

def handle_client(sel: selectors.BaseSelector, client: socket.socket):
    """클라이언트 데이터 처리"""
    try:
        data = client.recv(1024)
        if data:
            print(f"Received: {data.decode()}")
            client.send(data)  # Echo
        else:
            print("Client disconnected")
            sel.unregister(client)
            client.close()
    except Exception as e:
        print(f"Error: {e}")
        sel.unregister(client)
        client.close()

def main():
    """I/O Multiplexing 서버 (selectors 모듈 사용)"""

    # 최적의 selector 자동 선택 (epoll, kqueue, select 중)
    sel = selectors.DefaultSelector()
    print(f"Using selector: {type(sel).__name__}")

    server = create_server_socket('127.0.0.1', 8080)
    sel.register(server, selectors.EVENT_READ, accept_connection)
    print("Server listening on port 8080")

    try:
        while True:
            events = sel.select(timeout=None)

            for key, mask in events:
                callback = key.data
                if callback == accept_connection:
                    accept_connection(sel, key.fileobj)
                else:
                    handle_client(sel, key.fileobj)

    except KeyboardInterrupt:
        print("\nShutting down...")

    finally:
        sel.close()
        server.close()

if __name__ == "__main__":
    main()
```

### 4.8 Edge Trigger vs Level Trigger

```
Level Triggered (기본):
- 조건이 만족되는 동안 계속 알림
- 읽을 데이터가 있으면 epoll_wait이 계속 반환
- 놓치기 어려움, 구현 간단

Edge Triggered (EPOLLET):
- 상태 변화 시에만 알림
- 새 데이터가 도착했을 때만 알림
- 반드시 모든 데이터를 읽어야 함 (EAGAIN까지)
- 성능 향상, 구현 복잡

예시:
  100 bytes 도착

Level Triggered:
  epoll_wait() → returns (50 bytes 읽음)
  epoll_wait() → returns (50 bytes 남음)
  epoll_wait() → returns (데이터 있으면)

Edge Triggered:
  epoll_wait() → returns (한 번만!)
  read() until EAGAIN
```

---

## 5. DMA (Direct Memory Access)

### 5.1 DMA 개요

CPU 개입 없이 장치가 직접 메모리에 접근하여 데이터를 전송합니다.

```
Without DMA (Programmed I/O):
┌───────┐      ┌───────┐      ┌───────┐
│  CPU  │◄────►│ Memory│◄────►│Device │
└───────┘      └───────┘      └───────┘
    │               ▲               │
    └───────────────┴───────────────┘
         모든 데이터가 CPU 통과

With DMA:
┌───────┐      ┌───────┐      ┌───────┐
│  CPU  │      │ Memory│◄────►│Device │
└───────┘      └───────┘      └───────┘
    │               ▲ DMA          ▲
    │               └──────────────┘
    │ 1. DMA 설정
    ▼ 4. 인터럽트
┌──────────────┐
│DMA Controller│
└──────────────┘
```

### 5.2 DMA 동작 과정

```
1. CPU가 DMA 컨트롤러에 전송 정보 설정
   - 소스/목적지 주소
   - 전송 크기
   - 방향 (읽기/쓰기)

2. DMA 컨트롤러가 버스 제어권 획득

3. 장치와 메모리 간 직접 데이터 전송

4. 전송 완료 시 CPU에 인터럽트

5. CPU는 그 동안 다른 작업 수행 가능
```

### 5.3 DMA 장점

| 장점 | 설명 |
|------|------|
| **CPU 부하 감소** | 대용량 전송에서 CPU 사이클 절약 |
| **높은 처리량** | 버스 대역폭을 효율적으로 활용 |
| **병렬 처리** | CPU가 다른 작업 수행 가능 |

### 5.4 Zero-Copy I/O

사용자 공간과 커널 공간 간 데이터 복사를 최소화합니다.

```
Traditional Copy (4 copies):
┌────────────┐
│ User Space │ 4. copy_to_user()
│   Buffer   │◄─────────────────┐
└────────────┘                  │
      │                         │
      │ 3. copy_from_user()     │
      ▼                         │
┌────────────┐             ┌────────────┐
│  Kernel    │  2. DMA     │  Kernel    │
│Read Buffer │◄────────────│Write Buffer│
└────────────┘             └────────────┘
      ▲                         │
      │ 1. DMA                  │ 4. DMA
      │                         ▼
┌────────────┐             ┌────────────┐
│   Disk     │             │    NIC     │
└────────────┘             └────────────┘


Zero-Copy (sendfile, 2 copies):
┌────────────┐
│ User Space │  (No copy!)
│   Buffer   │
└────────────┘

┌────────────┐             ┌────────────┐
│  Kernel    │  sendfile() │  Kernel    │
│Read Buffer ├────────────►│Socket Buf  │
└────────────┘             └────────────┘
      ▲                         │
      │ DMA                     │ DMA
      │                         ▼
┌────────────┐             ┌────────────┐
│   Disk     │             │    NIC     │
└────────────┘             └────────────┘
```

#### sendfile() 예제

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/sendfile.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <netinet/in.h>

void send_file_traditional(int client_fd, const char *filepath) {
    // 전통적 방식: read() + write()
    char buffer[4096];
    int file_fd = open(filepath, O_RDONLY);
    ssize_t bytes;

    while ((bytes = read(file_fd, buffer, sizeof(buffer))) > 0) {
        write(client_fd, buffer, bytes);  // 2번의 데이터 복사
    }

    close(file_fd);
}

void send_file_zerocopy(int client_fd, const char *filepath) {
    // Zero-copy: sendfile()
    int file_fd = open(filepath, O_RDONLY);
    struct stat stat_buf;
    fstat(file_fd, &stat_buf);

    off_t offset = 0;
    sendfile(client_fd, file_fd, &offset, stat_buf.st_size);
    // 사용자 공간 버퍼 복사 없음!

    close(file_fd);
}

int main() {
    // 서버 소켓 설정...
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port = htons(8080)
    };
    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 5);

    printf("Waiting for connection...\n");
    int client_fd = accept(server_fd, NULL, NULL);

    // Zero-copy로 파일 전송
    send_file_zerocopy(client_fd, "/etc/passwd");

    close(client_fd);
    close(server_fd);
    return 0;
}
```

### 5.5 Memory-Mapped I/O vs Port I/O

```
Memory-Mapped I/O:
- 장치 레지스터가 메모리 주소에 매핑
- 일반 메모리 명령어로 접근
- 현대 시스템에서 주로 사용

Port I/O (x86):
- 별도의 I/O 주소 공간
- in, out 명령어 사용
- 레거시 장치에서 사용
```

---

## 7. 참고 자료

### 공식 문서

- [OSTEP - I/O Devices](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [Linux man pages - epoll](https://man7.org/linux/man-pages/man7/epoll.7.html)
- [io_uring documentation](https://kernel.dk/io_uring.pdf)

### 관련 자료

- [The C10K Problem](http://www.kegel.com/c10k.html)
- [io_uring vs epoll Performance](https://github.com/axboe/liburing/issues/536)
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)

### 성능 비교 자료

- [io_uring vs. epoll Performance](https://www.alibabacloud.com/blog/io-uring-vs--epoll-which-is-better-in-network-programming_599544)
- [From epoll to io_uring](https://codemia.io/blog/path/From-epoll-to-iourings-Multishot-Receives--Why-2025-Is-the-Year-We-Finally-Kill-the-Event-Loop)

---

> **문서 작성일**: 2024년 12월
>
> **참고**: 이 문서는 OSTEP, Linux 커널 문서, POSIX 표준을 기반으로 작성되었습니다.

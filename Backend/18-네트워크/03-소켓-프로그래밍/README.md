# 소켓 프로그래밍

## 목차
1. [개요](#개요)
2. [소켓 API](#소켓-api)
3. [블로킹 vs 논블로킹 소켓](#블로킹-vs-논블로킹-소켓)
4. [이벤트 드리븐 서버](#이벤트-드리븐-서버)
5. [C10K 문제](#c10k-문제)
6. [WebSocket과 HTTP Keep-Alive](#websocket과-http-keep-alive)
7. [실무 적용](#실무-적용)
---

## 개요

소켓(Socket)은 네트워크 통신의 끝점(endpoint)으로, 프로세스 간 통신(IPC)을 위한 인터페이스입니다. BSD 소켓 API는 1983년 BSD Unix에서 처음 도입되었으며, 현재 거의 모든 운영체제에서 표준으로 사용됩니다.

### 소켓의 구성 요소

```
소켓 = (프로토콜, 로컬 IP, 로컬 포트, 원격 IP, 원격 포트)

┌─────────────────────────────────────────────────────────────┐
│                        소켓 5-tuple                          │
├─────────────────────────────────────────────────────────────┤
│  Protocol    │  TCP 또는 UDP                                │
├──────────────┼──────────────────────────────────────────────┤
│  Local IP    │  192.168.1.100                               │
├──────────────┼──────────────────────────────────────────────┤
│  Local Port  │  8080                                        │
├──────────────┼──────────────────────────────────────────────┤
│  Remote IP   │  10.0.0.1                                    │
├──────────────┼──────────────────────────────────────────────┤
│  Remote Port │  54321                                       │
└──────────────┴──────────────────────────────────────────────┘
```

### 소켓 유형

| 유형 | 상수 | 프로토콜 | 특징 |
|------|------|----------|------|
| **Stream Socket** | SOCK_STREAM | TCP | 연결 지향, 신뢰성, 순서 보장 |
| **Datagram Socket** | SOCK_DGRAM | UDP | 비연결, 메시지 경계 유지 |
| **Raw Socket** | SOCK_RAW | IP/ICMP | 직접 IP 패킷 처리 |

---

## 소켓 API

### 핵심 함수 개요

```
TCP 서버-클라이언트 소켓 API 흐름:

    Server                              Client
      │                                    │
  socket()                             socket()
      │                                    │
   bind()                                  │
      │                                    │
  listen()                                 │
      │                                    │
  accept() ◄─────── 연결 요청 ────────── connect()
      │                                    │
   read() ◄─────── 데이터 전송 ─────────  write()
      │                                    │
  write() ─────── 데이터 전송 ─────────►  read()
      │                                    │
  close()                               close()
      │                                    │
      ▼                                    ▼
```

### 1. socket() - 소켓 생성

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);

// domain: 주소 체계
//   AF_INET   - IPv4
//   AF_INET6  - IPv6
//   AF_UNIX   - Unix 도메인 소켓 (로컬 IPC)

// type: 소켓 유형
//   SOCK_STREAM - TCP
//   SOCK_DGRAM  - UDP
//   SOCK_RAW    - Raw socket

// protocol: 프로토콜 (보통 0으로 자동 선택)

// 반환값: 소켓 파일 디스크립터 (실패 시 -1)
```

```c
// C 예제: TCP 소켓 생성
#include <stdio.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    // TCP 소켓 생성
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket creation failed");
        return 1;
    }

    printf("Socket created: fd=%d\n", sockfd);

    close(sockfd);
    return 0;
}
```

```python
# Python 예제
import socket

# TCP 소켓
tcp_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# UDP 소켓
udp_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# IPv6 TCP 소켓
tcp6_socket = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
```

### 2. bind() - 주소 바인딩

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

// sockfd: 소켓 파일 디스크립터
// addr: 바인딩할 주소 구조체
// addrlen: 주소 구조체 크기

// 반환값: 성공 시 0, 실패 시 -1
```

```c
// C 예제: 주소 바인딩
#include <string.h>

int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    // 주소 구조체 설정
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;  // 모든 인터페이스
    server_addr.sin_port = htons(8080);         // 포트 8080

    // 소켓 옵션: 포트 재사용
    int opt = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 바인딩
    if (bind(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        return 1;
    }

    printf("Socket bound to port 8080\n");

    close(sockfd);
    return 0;
}
```

```python
# Python 예제
import socket

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 포트 재사용 옵션
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# 모든 인터페이스의 8080 포트에 바인딩
server_socket.bind(('0.0.0.0', 8080))
# 또는 server_socket.bind(('', 8080))
```

### 3. listen() - 연결 대기

```c
int listen(int sockfd, int backlog);

// sockfd: 소켓 파일 디스크립터
// backlog: 대기 큐 크기 (연결 대기 중인 클라이언트 수)

// 반환값: 성공 시 0, 실패 시 -1
```

```
Listen Backlog 구조:

┌─────────────────────────────────────────────────────────────┐
│                     Listen Backlog                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  SYN Queue (SYN_RCVD 상태)                                  │
│  ┌───┬───┬───┬───┬───┐                                     │
│  │ C1│ C2│ C3│...│   │  SYN 받고 SYN+ACK 보낸 연결들       │
│  └───┴───┴───┴───┴───┘                                     │
│         │                                                   │
│         ▼ (ACK 수신 시)                                     │
│  Accept Queue (ESTABLISHED 상태)                            │
│  ┌───┬───┬───┬───┬───┐                                     │
│  │ C1│ C2│   │   │   │  accept() 대기 중인 연결들          │
│  └───┴───┴───┴───┴───┘                                     │
│         │                                                   │
│         ▼                                                   │
│     accept()                                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘

※ backlog는 Accept Queue 크기를 의미
※ Linux에서 실제 큐 크기: min(backlog, /proc/sys/net/core/somaxconn)
```

```c
// C 예제
if (listen(sockfd, 128) < 0) {  // backlog = 128
    perror("listen failed");
    return 1;
}
printf("Server listening...\n");
```

```python
# Python 예제
server_socket.listen(128)  # backlog = 128
print("Server listening...")
```

### 4. accept() - 연결 수락

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// sockfd: 리스닝 소켓
// addr: 클라이언트 주소 정보를 저장할 구조체 (출력)
// addrlen: addr 크기 (입출력)

// 반환값: 새로운 연결 소켓 fd (실패 시 -1)
```

```c
// C 예제: 연결 수락
int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(8080);

    bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
    listen(listen_fd, 128);

    printf("Waiting for connections...\n");

    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        // 연결 수락 (블로킹)
        int client_fd = accept(listen_fd,
                               (struct sockaddr*)&client_addr,
                               &client_len);

        if (client_fd < 0) {
            perror("accept failed");
            continue;
        }

        printf("Client connected: %s:%d\n",
               inet_ntoa(client_addr.sin_addr),
               ntohs(client_addr.sin_port));

        // 클라이언트 처리...

        close(client_fd);
    }

    close(listen_fd);
    return 0;
}
```

```python
# Python 예제
while True:
    # 연결 수락 (블로킹)
    client_socket, client_addr = server_socket.accept()
    print(f"Client connected: {client_addr}")

    # 클라이언트 처리
    data = client_socket.recv(1024)
    client_socket.send(b"Hello, Client!")

    client_socket.close()
```

### 5. connect() - 서버 연결

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);

// sockfd: 소켓 파일 디스크립터
// addr: 서버 주소
// addrlen: 주소 크기

// 반환값: 성공 시 0, 실패 시 -1
```

```c
// C 예제: 서버 연결
int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8080);
    inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);

    // 서버에 연결 (3-way handshake 수행)
    if (connect(sockfd, (struct sockaddr*)&server_addr,
                sizeof(server_addr)) < 0) {
        perror("connect failed");
        return 1;
    }

    printf("Connected to server\n");

    // 데이터 송수신...

    close(sockfd);
    return 0;
}
```

```python
# Python 예제
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 연결 시도 (타임아웃 설정 가능)
client_socket.settimeout(10)  # 10초 타임아웃

try:
    client_socket.connect(('127.0.0.1', 8080))
    print("Connected to server")
except socket.timeout:
    print("Connection timeout")
except ConnectionRefusedError:
    print("Connection refused")
```

### 6. send()/recv() - 데이터 송수신

```c
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);

// 주요 flags:
//   MSG_DONTWAIT - 논블로킹
//   MSG_PEEK     - 데이터 읽지만 큐에서 제거하지 않음
//   MSG_WAITALL  - 요청한 크기만큼 데이터 수신까지 대기

// 반환값: 실제 송수신 바이트 수, 0은 연결 종료, -1은 오류
```

```c
// C 예제: 데이터 송수신
void handle_client(int client_fd) {
    char buffer[1024];

    while (1) {
        // 데이터 수신
        ssize_t bytes_read = recv(client_fd, buffer, sizeof(buffer) - 1, 0);

        if (bytes_read < 0) {
            perror("recv error");
            break;
        }

        if (bytes_read == 0) {
            printf("Client disconnected\n");
            break;
        }

        buffer[bytes_read] = '\0';
        printf("Received: %s\n", buffer);

        // 에코 응답
        ssize_t bytes_sent = send(client_fd, buffer, bytes_read, 0);
        if (bytes_sent < 0) {
            perror("send error");
            break;
        }
    }
}
```

```python
# Python 예제
def handle_client(client_socket):
    while True:
        # 데이터 수신
        data = client_socket.recv(1024)

        if not data:  # 연결 종료
            print("Client disconnected")
            break

        print(f"Received: {data.decode()}")

        # 에코 응답
        client_socket.send(data)

    client_socket.close()
```

### 완전한 TCP 서버/클라이언트 예제

```c
// tcp_server.c - 멀티 클라이언트 Echo 서버
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <pthread.h>

#define PORT 8080
#define BUFFER_SIZE 1024

void* handle_client(void* arg) {
    int client_fd = *(int*)arg;
    free(arg);

    char buffer[BUFFER_SIZE];

    while (1) {
        ssize_t bytes = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
        if (bytes <= 0) break;

        buffer[bytes] = '\0';
        printf("Received: %s\n", buffer);

        send(client_fd, buffer, bytes, 0);
    }

    close(client_fd);
    return NULL;
}

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_addr.s_addr = INADDR_ANY,
        .sin_port = htons(PORT)
    };

    bind(server_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(server_fd, 128);

    printf("Server listening on port %d\n", PORT);

    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);

        int* client_fd = malloc(sizeof(int));
        *client_fd = accept(server_fd, (struct sockaddr*)&client_addr, &client_len);

        printf("Client connected: %s:%d\n",
               inet_ntoa(client_addr.sin_addr),
               ntohs(client_addr.sin_port));

        pthread_t thread;
        pthread_create(&thread, NULL, handle_client, client_fd);
        pthread_detach(thread);
    }

    close(server_fd);
    return 0;
}
```

```python
# tcp_server.py - 멀티스레드 Echo 서버
import socket
import threading

def handle_client(client_socket, addr):
    print(f"Client connected: {addr}")

    try:
        while True:
            data = client_socket.recv(1024)
            if not data:
                break

            print(f"Received from {addr}: {data.decode()}")
            client_socket.send(data)
    except Exception as e:
        print(f"Error: {e}")
    finally:
        client_socket.close()
        print(f"Client disconnected: {addr}")

def main():
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

    server_socket.bind(('0.0.0.0', 8080))
    server_socket.listen(128)

    print("Server listening on port 8080")

    try:
        while True:
            client_socket, addr = server_socket.accept()

            thread = threading.Thread(
                target=handle_client,
                args=(client_socket, addr)
            )
            thread.daemon = True
            thread.start()
    except KeyboardInterrupt:
        print("\nShutting down...")
    finally:
        server_socket.close()

if __name__ == "__main__":
    main()
```

---

## 블로킹 vs 논블로킹 소켓

### 블로킹 소켓

```
블로킹 소켓의 동작:

Thread                                   Socket
   │                                        │
   │ ──────── recv() 호출 ─────────────────>│
   │                                        │
   │    ┌─────────────────────────────┐     │
   │    │  스레드 블로킹 (대기)        │     │
   │    │  (CPU 사용 없음)            │     │
   │    │         ...                 │     │
   │    │  데이터 도착!               │     │
   │    └─────────────────────────────┘     │
   │                                        │
   │ <──────── 데이터 반환 ─────────────────│
   │                                        │
   ▼                                        ▼

문제점:
- 하나의 스레드가 하나의 연결만 처리 가능
- 많은 연결 = 많은 스레드 필요
- 스레드 생성/컨텍스트 스위칭 오버헤드
```

### 논블로킹 소켓

```c
// 논블로킹 모드 설정 방법

// 방법 1: fcntl 사용 (POSIX)
#include <fcntl.h>

int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

// 방법 2: ioctl 사용
#include <sys/ioctl.h>

int on = 1;
ioctl(sockfd, FIONBIO, &on);

// 방법 3: socket 생성 시 (Linux 2.6.27+)
int sockfd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
```

```python
# Python에서 논블로킹 설정
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setblocking(False)  # 논블로킹 모드

# 또는 타임아웃 설정 (0 = 논블로킹)
sock.settimeout(0)
```

### 논블로킹 소켓의 동작

```
논블로킹 소켓의 동작:

Thread                                   Socket
   │                                        │
   │ ──────── recv() 호출 ─────────────────>│
   │ <──────── EAGAIN/EWOULDBLOCK ─────────│  (데이터 없음)
   │                                        │
   │    다른 작업 수행                       │
   │                                        │
   │ ──────── recv() 호출 ─────────────────>│
   │ <──────── EAGAIN/EWOULDBLOCK ─────────│  (데이터 없음)
   │                                        │
   │    다른 작업 수행                       │
   │                                        │
   │ ──────── recv() 호출 ─────────────────>│
   │ <──────── 데이터 반환 ─────────────────│  (데이터 있음!)
   │                                        │
   ▼                                        ▼

문제점:
- Busy waiting (폴링) - CPU 낭비
- 효율적인 이벤트 알림 메커니즘 필요
```

```c
// 논블로킹 소켓 사용 예제
#include <errno.h>

void non_blocking_recv(int sockfd) {
    char buffer[1024];

    while (1) {
        ssize_t bytes = recv(sockfd, buffer, sizeof(buffer), 0);

        if (bytes > 0) {
            // 데이터 수신 성공
            buffer[bytes] = '\0';
            printf("Received: %s\n", buffer);
        }
        else if (bytes == 0) {
            // 연결 종료
            printf("Connection closed\n");
            break;
        }
        else {
            // 오류 또는 데이터 없음
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 데이터 없음 - 다른 작업 수행
                do_other_work();
                continue;
            }
            else {
                // 실제 오류
                perror("recv error");
                break;
            }
        }
    }
}
```

---

## 이벤트 드리븐 서버

### I/O 멀티플렉싱 개요

```
I/O 멀티플렉싱: 단일 스레드로 다수의 I/O 이벤트 처리

┌─────────────────────────────────────────────────────────────┐
│                    이벤트 기반 서버                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐          │
│   │Socket 1│  │Socket 2│  │Socket 3│  │Socket N│          │
│   └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘          │
│       │           │           │           │                │
│       └───────────┴─────┬─────┴───────────┘                │
│                         │                                   │
│                         ▼                                   │
│           ┌─────────────────────────┐                      │
│           │  I/O Multiplexer       │                      │
│           │  (select/poll/epoll)    │                      │
│           └───────────┬─────────────┘                      │
│                       │                                     │
│                       ▼                                     │
│           ┌─────────────────────────┐                      │
│           │    Single Thread        │                      │
│           │    Event Loop           │                      │
│           └─────────────────────────┘                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### select()

```c
#include <sys/select.h>

int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);

// nfds: 감시할 fd 중 가장 큰 값 + 1
// readfds: 읽기 가능 여부 감시할 fd 집합
// writefds: 쓰기 가능 여부 감시할 fd 집합
// exceptfds: 예외 상황 감시할 fd 집합
// timeout: 타임아웃 (NULL이면 무한 대기)

// fd_set 매크로:
// FD_ZERO(&set)     - 집합 초기화
// FD_SET(fd, &set)  - fd 추가
// FD_CLR(fd, &set)  - fd 제거
// FD_ISSET(fd, &set) - fd 포함 여부 확인
```

```c
// select() 사용 예제
#include <sys/select.h>

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    // ... bind, listen ...

    fd_set master_set, read_set;
    FD_ZERO(&master_set);
    FD_SET(server_fd, &master_set);
    int max_fd = server_fd;

    while (1) {
        read_set = master_set;  // select()가 수정하므로 복사

        struct timeval timeout = {.tv_sec = 5, .tv_usec = 0};
        int ready = select(max_fd + 1, &read_set, NULL, NULL, &timeout);

        if (ready < 0) {
            perror("select error");
            break;
        }

        if (ready == 0) {
            printf("Timeout\n");
            continue;
        }

        for (int fd = 0; fd <= max_fd; fd++) {
            if (!FD_ISSET(fd, &read_set)) continue;

            if (fd == server_fd) {
                // 새 연결 수락
                int client_fd = accept(server_fd, NULL, NULL);
                FD_SET(client_fd, &master_set);
                if (client_fd > max_fd) max_fd = client_fd;
                printf("New connection: fd=%d\n", client_fd);
            }
            else {
                // 클라이언트 데이터 처리
                char buffer[1024];
                ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

                if (bytes <= 0) {
                    close(fd);
                    FD_CLR(fd, &master_set);
                    printf("Connection closed: fd=%d\n", fd);
                }
                else {
                    buffer[bytes] = '\0';
                    send(fd, buffer, bytes, 0);  // Echo
                }
            }
        }
    }

    return 0;
}
```

**select()의 한계**:
- fd 수 제한 (FD_SETSIZE, 보통 1024)
- 매번 fd_set 복사 필요
- O(n) 순회 필요

### poll()

```c
#include <poll.h>

int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;         // 파일 디스크립터
    short events;     // 감시할 이벤트
    short revents;    // 발생한 이벤트 (출력)
};

// 주요 이벤트:
// POLLIN   - 읽기 가능
// POLLOUT  - 쓰기 가능
// POLLERR  - 오류 발생
// POLLHUP  - 연결 종료
// POLLNVAL - 잘못된 fd
```

```c
// poll() 사용 예제
#include <poll.h>

#define MAX_CLIENTS 1000

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    // ... bind, listen ...

    struct pollfd fds[MAX_CLIENTS];
    int nfds = 1;

    fds[0].fd = server_fd;
    fds[0].events = POLLIN;

    while (1) {
        int ready = poll(fds, nfds, 5000);  // 5초 타임아웃

        if (ready < 0) {
            perror("poll error");
            break;
        }

        for (int i = 0; i < nfds; i++) {
            if (fds[i].revents == 0) continue;

            if (fds[i].fd == server_fd) {
                // 새 연결
                int client_fd = accept(server_fd, NULL, NULL);
                fds[nfds].fd = client_fd;
                fds[nfds].events = POLLIN;
                nfds++;
            }
            else {
                // 클라이언트 데이터
                char buffer[1024];
                ssize_t bytes = recv(fds[i].fd, buffer, sizeof(buffer), 0);

                if (bytes <= 0) {
                    close(fds[i].fd);
                    // 배열에서 제거
                    fds[i] = fds[nfds - 1];
                    nfds--;
                    i--;
                }
                else {
                    send(fds[i].fd, buffer, bytes, 0);
                }
            }
        }
    }

    return 0;
}
```

### epoll (Linux)

```c
#include <sys/epoll.h>

// epoll 인스턴스 생성
int epoll_create(int size);      // size는 힌트 (무시됨)
int epoll_create1(int flags);    // flags: EPOLL_CLOEXEC

// fd 추가/수정/삭제
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL

// 이벤트 대기
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);

struct epoll_event {
    uint32_t events;    // 이벤트 마스크
    epoll_data_t data;  // 사용자 데이터
};

// 주요 이벤트:
// EPOLLIN      - 읽기 가능
// EPOLLOUT     - 쓰기 가능
// EPOLLERR     - 오류
// EPOLLHUP     - 연결 종료
// EPOLLET      - Edge Triggered 모드
// EPOLLONESHOT - 한 번만 알림
```

```c
// epoll 사용 예제
#include <sys/epoll.h>
#include <fcntl.h>

#define MAX_EVENTS 1024

void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
    // ... bind, listen ...

    // epoll 인스턴스 생성
    int epfd = epoll_create1(0);

    // 서버 소켓 등록
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = server_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, server_fd, &ev);

    struct epoll_event events[MAX_EVENTS];

    while (1) {
        int nready = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < nready; i++) {
            int fd = events[i].data.fd;

            if (fd == server_fd) {
                // 새 연결
                while (1) {
                    int client_fd = accept(server_fd, NULL, NULL);
                    if (client_fd < 0) {
                        if (errno == EAGAIN) break;  // 모든 연결 처리 완료
                        perror("accept");
                        break;
                    }

                    set_nonblocking(client_fd);

                    ev.events = EPOLLIN | EPOLLET;  // Edge Triggered
                    ev.data.fd = client_fd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);

                    printf("New connection: fd=%d\n", client_fd);
                }
            }
            else {
                // 클라이언트 데이터
                if (events[i].events & (EPOLLERR | EPOLLHUP)) {
                    close(fd);
                    continue;
                }

                char buffer[1024];
                while (1) {
                    ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

                    if (bytes < 0) {
                        if (errno == EAGAIN) break;  // 모든 데이터 읽음
                        close(fd);
                        break;
                    }

                    if (bytes == 0) {
                        close(fd);
                        break;
                    }

                    send(fd, buffer, bytes, 0);
                }
            }
        }
    }

    close(epfd);
    close(server_fd);
    return 0;
}
```

### Level Triggered vs Edge Triggered

```
Level Triggered (LT, 기본):
- 조건이 만족되는 동안 계속 이벤트 발생
- 버퍼에 데이터가 있으면 계속 EPOLLIN 반환
- select/poll과 동일한 동작

Edge Triggered (ET):
- 상태가 변경될 때만 이벤트 발생
- 새 데이터가 도착했을 때만 EPOLLIN 반환
- 한 번에 모든 데이터를 읽어야 함 (EAGAIN까지)
- 더 효율적이지만 주의 필요

┌─────────────────────────────────────────────────────────────┐
│  버퍼 상태:  [████████░░░░] (8KB 데이터)                     │
│                                                             │
│  Level Triggered:                                           │
│    epoll_wait() → EPOLLIN                                   │
│    read(4KB)    → 버퍼에 4KB 남음                            │
│    epoll_wait() → EPOLLIN (아직 데이터 있음)                 │
│    read(4KB)    → 버퍼 비움                                  │
│    epoll_wait() → (대기)                                     │
│                                                             │
│  Edge Triggered:                                            │
│    epoll_wait() → EPOLLIN                                   │
│    read(4KB)    → 버퍼에 4KB 남음                            │
│    epoll_wait() → (대기! 새 데이터 올 때까지)                 │
│                   ⚠️ 남은 4KB는 영원히 읽히지 않음!           │
│                                                             │
│  Edge Triggered 올바른 사용:                                 │
│    epoll_wait() → EPOLLIN                                   │
│    while (1) {                                              │
│      read() → EAGAIN까지 반복                                │
│    }                                                        │
└─────────────────────────────────────────────────────────────┘
```

### kqueue (BSD/macOS)

```c
#include <sys/event.h>

// kqueue 인스턴스 생성
int kqueue(void);

// 이벤트 등록/대기
int kevent(int kq,
           const struct kevent *changelist, int nchanges,
           struct kevent *eventlist, int nevents,
           const struct timespec *timeout);

struct kevent {
    uintptr_t  ident;   // 식별자 (fd)
    int16_t    filter;  // 필터 타입
    uint16_t   flags;   // 동작 플래그
    uint32_t   fflags;  // 필터별 플래그
    intptr_t   data;    // 필터별 데이터
    void       *udata;  // 사용자 데이터
};

// 주요 필터:
// EVFILT_READ   - 읽기 가능
// EVFILT_WRITE  - 쓰기 가능
// EVFILT_TIMER  - 타이머

// 주요 플래그:
// EV_ADD        - 이벤트 추가
// EV_DELETE     - 이벤트 삭제
// EV_ENABLE     - 이벤트 활성화
// EV_DISABLE    - 이벤트 비활성화
// EV_CLEAR      - 이벤트 자동 클리어 (Edge Triggered 유사)
```

```c
// kqueue 사용 예제 (macOS/BSD)
#include <sys/event.h>

#define MAX_EVENTS 1024

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    // ... bind, listen ...

    int kq = kqueue();

    struct kevent ev;
    EV_SET(&ev, server_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
    kevent(kq, &ev, 1, NULL, 0, NULL);

    struct kevent events[MAX_EVENTS];

    while (1) {
        int nev = kevent(kq, NULL, 0, events, MAX_EVENTS, NULL);

        for (int i = 0; i < nev; i++) {
            int fd = events[i].ident;

            if (fd == server_fd) {
                int client_fd = accept(server_fd, NULL, NULL);

                EV_SET(&ev, client_fd, EVFILT_READ, EV_ADD, 0, 0, NULL);
                kevent(kq, &ev, 1, NULL, 0, NULL);
            }
            else {
                char buffer[1024];
                ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

                if (bytes <= 0) {
                    EV_SET(&ev, fd, EVFILT_READ, EV_DELETE, 0, 0, NULL);
                    kevent(kq, &ev, 1, NULL, 0, NULL);
                    close(fd);
                }
                else {
                    send(fd, buffer, bytes, 0);
                }
            }
        }
    }

    return 0;
}
```

### I/O 멀티플렉싱 비교

| 구분 | select | poll | epoll | kqueue |
|------|--------|------|-------|--------|
| **플랫폼** | 모든 플랫폼 | POSIX | Linux | BSD/macOS |
| **fd 제한** | FD_SETSIZE (1024) | 없음 | 없음 | 없음 |
| **시간 복잡도** | O(n) | O(n) | O(1) | O(1) |
| **메모리 복사** | 매번 전체 복사 | 매번 전체 복사 | 없음 | 없음 |
| **Edge Triggered** | 불가 | 불가 | 가능 | 가능 |
| **파일 지원** | 가능 | 가능 | 제한적 | 가능 |

---

## C10K 문제

### 개요

C10K(Concurrent 10,000) 문제는 1999년 Dan Kegel이 제기한 것으로, 단일 서버에서 10,000개의 동시 연결을 효율적으로 처리하는 것에 대한 도전 과제였습니다.

```
C10K 문제의 핵심:

전통적인 방식 (스레드 per 연결):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   10,000 연결 = 10,000 스레드                               │
│                                                             │
│   문제점:                                                    │
│   1. 메모리: 스레드당 ~1MB 스택 = 10GB 메모리               │
│   2. 컨텍스트 스위칭: 수만 번의 스위칭 오버헤드              │
│   3. 스케줄러 부하: O(n) 스케줄링                           │
│   4. 스레드 생성 비용: 연결당 pthread_create 호출           │
│                                                             │
└─────────────────────────────────────────────────────────────┘

해결책 (이벤트 기반):
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   10,000 연결 = 소수의 스레드 (1~CPU 코어 수)                │
│                                                             │
│   epoll/kqueue:                                             │
│   - O(1) 이벤트 감지                                         │
│   - 최소한의 메모리 사용                                     │
│   - 컨텍스트 스위칭 최소화                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### C10K 해결 기술

```
1. I/O Multiplexing (epoll/kqueue)
   - 2000년: kqueue (FreeBSD 4.3)
   - 2002년: epoll (Linux 2.5.44)

2. 논블로킹 I/O
   - 스레드 블로킹 없이 I/O 처리
   - 이벤트 루프 기반 프로그래밍

3. 이벤트 기반 아키텍처
   - Reactor 패턴
   - Proactor 패턴

4. 대표적인 구현:
   - Nginx (2004)
   - Node.js (2009)
   - Netty (Java)
   - libevent, libuv
```

### C10M 시대 (2010년대~)

```
C10M: 10,000,000 동시 연결

추가 기술:
1. io_uring (Linux 5.1, 2019)
   - 비동기 I/O 인터페이스
   - 시스템 콜 오버헤드 최소화

2. 커널 바이패스
   - DPDK (Data Plane Development Kit)
   - 네트워크 스택을 유저 스페이스에서 처리

3. SO_REUSEPORT
   - 여러 프로세스가 동일 포트 공유
   - 로드 밸런싱 효과
```

### Python 이벤트 기반 서버 예제

```python
# asyncio 기반 에코 서버 (Python 3.7+)
import asyncio

async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    print(f"Client connected: {addr}")

    try:
        while True:
            data = await reader.read(1024)
            if not data:
                break

            print(f"Received from {addr}: {data.decode()}")
            writer.write(data)
            await writer.drain()
    except Exception as e:
        print(f"Error: {e}")
    finally:
        writer.close()
        await writer.wait_closed()
        print(f"Client disconnected: {addr}")

async def main():
    server = await asyncio.start_server(
        handle_client, '0.0.0.0', 8080
    )

    addr = server.sockets[0].getsockname()
    print(f"Server listening on {addr}")

    async with server:
        await server.serve_forever()

if __name__ == "__main__":
    asyncio.run(main())
```

---

## WebSocket과 HTTP Keep-Alive

### HTTP Keep-Alive

```
HTTP/1.0 (Keep-Alive 없이):
┌────────┐                           ┌────────┐
│ Client │                           │ Server │
└───┬────┘                           └───┬────┘
    │                                    │
    │ ──── TCP 연결 ───────────────────> │
    │ ──── HTTP 요청 1 ────────────────> │
    │ <─── HTTP 응답 1 ─────────────── │
    │ ──── TCP 종료 ───────────────────> │
    │                                    │
    │ ──── TCP 연결 ───────────────────> │
    │ ──── HTTP 요청 2 ────────────────> │
    │ <─── HTTP 응답 2 ─────────────── │
    │ ──── TCP 종료 ───────────────────> │
    │                                    │
    ▼                                    ▼

HTTP/1.1 (Keep-Alive 기본 활성화):
┌────────┐                           ┌────────┐
│ Client │                           │ Server │
└───┬────┘                           └───┬────┘
    │                                    │
    │ ──── TCP 연결 ───────────────────> │
    │                                    │
    │ ──── HTTP 요청 1 ────────────────> │
    │ <─── HTTP 응답 1 ─────────────── │
    │                                    │
    │ ──── HTTP 요청 2 ────────────────> │
    │ <─── HTTP 응답 2 ─────────────── │
    │                                    │
    │ ──── HTTP 요청 3 ────────────────> │
    │ <─── HTTP 응답 3 ─────────────── │
    │                                    │
    │ ──── TCP 종료 (타임아웃 또는 명시적) ─> │
    ▼                                    ▼

HTTP 헤더:
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

### HTTP Keep-Alive 설정

```python
# Python requests 라이브러리 - Keep-Alive 자동 사용
import requests

# Session 사용으로 Keep-Alive 연결 재사용
session = requests.Session()

# 동일 호스트에 대한 요청은 연결 재사용
response1 = session.get('https://api.example.com/users')
response2 = session.get('https://api.example.com/posts')
response3 = session.get('https://api.example.com/comments')

# 세션 종료
session.close()
```

```nginx
# Nginx Keep-Alive 설정
http {
    # 클라이언트와의 Keep-Alive
    keepalive_timeout 65;          # 타임아웃 (초)
    keepalive_requests 100;        # 연결당 최대 요청 수

    # Upstream(백엔드)과의 Keep-Alive
    upstream backend {
        server 127.0.0.1:8080;
        keepalive 32;              # Keep-Alive 연결 풀 크기
    }

    server {
        location /api/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";  # Keep-Alive 유지
        }
    }
}
```

### WebSocket

WebSocket은 HTTP를 통해 연결을 시작한 후, 전이중(Full-Duplex) 양방향 통신을 제공하는 프로토콜입니다.

```
WebSocket 핸드셰이크:

Client → Server (HTTP Upgrade 요청):
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Server → Client (101 Switching Protocols):
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

이후 WebSocket 프레임으로 통신:
┌────────┐  ◄────────────────────────►  ┌────────┐
│ Client │         양방향 통신            │ Server │
└────────┘                               └────────┘
```

### WebSocket vs HTTP 비교

```
HTTP (Polling):
┌────────┐                           ┌────────┐
│ Client │                           │ Server │
└───┬────┘                           └───┬────┘
    │                                    │
    │ ──── 새 메시지 있나요? ───────────> │
    │ <─── 없음 ──────────────────────  │
    │                                    │
    │ ──── 새 메시지 있나요? ───────────> │
    │ <─── 없음 ──────────────────────  │
    │                                    │
    │ ──── 새 메시지 있나요? ───────────> │
    │ <─── 있음! [메시지] ─────────────  │
    │                                    │
    문제: 불필요한 요청, 지연 발생

WebSocket:
┌────────┐                           ┌────────┐
│ Client │                           │ Server │
└───┬────┘                           └───┬────┘
    │                                    │
    │ ════ 연결 수립 ══════════════════> │
    │                                    │
    │      (연결 유지, 대기)              │
    │                                    │
    │ <════ 메시지 즉시 전송 ═══════════  │
    │                                    │
    │ ════ 클라이언트 메시지 ═══════════> │
    │                                    │
    장점: 실시간, 오버헤드 최소
```

### Python WebSocket 서버

```python
# websockets 라이브러리 사용
import asyncio
import websockets
import json

connected_clients = set()

async def handler(websocket, path):
    """WebSocket 연결 핸들러"""
    connected_clients.add(websocket)
    client_id = id(websocket)
    print(f"Client connected: {client_id}")

    try:
        async for message in websocket:
            print(f"Received from {client_id}: {message}")

            # 브로드캐스트
            response = json.dumps({
                'from': client_id,
                'message': message
            })

            await asyncio.gather(*[
                client.send(response)
                for client in connected_clients
                if client.open
            ])
    except websockets.ConnectionClosed:
        pass
    finally:
        connected_clients.remove(websocket)
        print(f"Client disconnected: {client_id}")

async def main():
    async with websockets.serve(handler, "0.0.0.0", 8765):
        print("WebSocket server started on ws://0.0.0.0:8765")
        await asyncio.Future()  # 무한 대기

if __name__ == "__main__":
    asyncio.run(main())
```

```javascript
// JavaScript 클라이언트
const ws = new WebSocket('ws://localhost:8765');

ws.onopen = () => {
    console.log('Connected');
    ws.send('Hello, Server!');
};

ws.onmessage = (event) => {
    console.log('Received:', event.data);
};

ws.onclose = () => {
    console.log('Disconnected');
};

ws.onerror = (error) => {
    console.error('Error:', error);
};
```

---

## 실무 적용

### 소켓 옵션 튜닝

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 포트 재사용 (TIME_WAIT 상태에서도 바인딩 가능)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# 송수신 버퍼 크기
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 65536)
sock.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, 65536)

# TCP_NODELAY (Nagle 알고리즘 비활성화)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

# TCP Keep-Alive
sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
# Linux 전용 Keep-Alive 세부 설정
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 60)   # 유휴 시간
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)  # 프로브 간격
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 5)     # 프로브 횟수

# 타임아웃
sock.settimeout(30)  # 30초
```

### 시스템 튜닝 (Linux)

```bash
# /etc/sysctl.conf

# 파일 디스크립터 제한 확인 및 증가
# 현재 값 확인
cat /proc/sys/fs/file-max
ulimit -n

# 시스템 전체 제한
fs.file-max = 2097152

# 네트워크 버퍼
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576

# TCP 버퍼
net.ipv4.tcp_rmem = 4096 1048576 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Connection Backlog
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535

# TIME_WAIT
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30

# 포트 범위 (아웃바운드 연결용)
net.ipv4.ip_local_port_range = 1024 65535

# 적용
sudo sysctl -p
```

```bash
# /etc/security/limits.conf
# 사용자별 파일 디스크립터 제한

*    soft    nofile    65535
*    hard    nofile    65535
root soft    nofile    65535
root hard    nofile    65535
```

### Connection Pool 구현

```python
import socket
import threading
from queue import Queue, Empty
from contextlib import contextmanager

class ConnectionPool:
    """TCP 연결 풀 구현"""

    def __init__(self, host, port, pool_size=10, timeout=30):
        self.host = host
        self.port = port
        self.pool_size = pool_size
        self.timeout = timeout

        self.pool = Queue(maxsize=pool_size)
        self.lock = threading.Lock()
        self._created = 0

    def _create_connection(self):
        """새 연결 생성"""
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(self.timeout)
        sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)
        sock.connect((self.host, self.port))
        return sock

    def get_connection(self):
        """풀에서 연결 획득"""
        try:
            conn = self.pool.get_nowait()
            # 연결 유효성 검사
            try:
                conn.getpeername()
                return conn
            except:
                conn.close()
                return self._create_connection()
        except Empty:
            with self.lock:
                if self._created < self.pool_size:
                    self._created += 1
                    return self._create_connection()
            # 풀이 가득 찬 경우 대기
            return self.pool.get(timeout=self.timeout)

    def release_connection(self, conn):
        """연결을 풀에 반환"""
        try:
            conn.getpeername()  # 연결 유효성 확인
            self.pool.put_nowait(conn)
        except:
            conn.close()
            with self.lock:
                self._created -= 1

    @contextmanager
    def connection(self):
        """컨텍스트 매니저로 연결 사용"""
        conn = self.get_connection()
        try:
            yield conn
        finally:
            self.release_connection(conn)

    def close_all(self):
        """모든 연결 종료"""
        while not self.pool.empty():
            try:
                conn = self.pool.get_nowait()
                conn.close()
            except Empty:
                break

# 사용 예시
pool = ConnectionPool('127.0.0.1', 8080, pool_size=20)

with pool.connection() as conn:
    conn.send(b"Hello")
    response = conn.recv(1024)
```

---

## 참고 자료

- [The C10K problem - Dan Kegel](http://www.kegel.com/c10k.html)
- [epoll: The API that powers the modern internet](https://darkcoding.net/software/epoll-the-api-that-powers-the-modern-internet/)
- [RFC 6455 - The WebSocket Protocol](https://datatracker.ietf.org/doc/html/rfc6455)
- [Linux man pages - socket(7), epoll(7)](https://man7.org/linux/man-pages/)
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)
- Unix Network Programming (W. Richard Stevens)
- [libuv - Cross-platform async I/O](https://libuv.org/)

# TCP와 UDP

## 목차
1. [개요](#개요)
2. [TCP (Transmission Control Protocol)](#tcp-transmission-control-protocol)
3. [3-way Handshake](#3-way-handshake)
4. [4-way Handshake](#4-way-handshake)
5. [흐름 제어 (Flow Control)](#흐름-제어-flow-control)
6. [혼잡 제어 (Congestion Control)](#혼잡-제어-congestion-control)
7. [UDP (User Datagram Protocol)](#udp-user-datagram-protocol)
8. [TCP vs UDP 비교](#tcp-vs-udp-비교)
9. [UDP 사용 사례](#udp-사용-사례)
10. [실무 적용](#실무-적용)
---

## 개요

TCP와 UDP는 OSI 7계층 중 4계층(전송 계층)에서 동작하는 핵심 프로토콜입니다. 두 프로토콜은 RFC 793(TCP)과 RFC 768(UDP)에 정의되어 있으며, 애플리케이션 간의 데이터 전송을 담당합니다.

### 전송 계층의 역할

```
┌─────────────────────────────────────────────────────────────┐
│                    전송 계층의 핵심 기능                       │
├─────────────────────────────────────────────────────────────┤
│  1. 프로세스 간 통신 (Process-to-Process Communication)      │
│     - 포트 번호를 통한 프로세스 식별                           │
│                                                             │
│  2. 세그멘테이션 (Segmentation)                              │
│     - 큰 데이터를 작은 세그먼트로 분할                         │
│                                                             │
│  3. 연결 관리 (Connection Management)                       │
│     - TCP의 연결 설정/해제                                   │
│                                                             │
│  4. 신뢰성 보장 (Reliability)                                │
│     - 오류 검출, 순서 제어, 재전송 (TCP)                      │
│                                                             │
│  5. 흐름 제어 (Flow Control)                                 │
│     - 송수신 속도 조절 (TCP)                                 │
│                                                             │
│  6. 혼잡 제어 (Congestion Control)                          │
│     - 네트워크 혼잡 대응 (TCP)                               │
└─────────────────────────────────────────────────────────────┘
```

---

## TCP (Transmission Control Protocol)

### TCP 개요

TCP는 연결 지향적(Connection-Oriented) 프로토콜로, 신뢰성 있는 데이터 전송을 보장합니다. RFC 793에 정의되어 있으며, 1981년 9월에 발표되었습니다.

### TCP 특징

| 특징 | 설명 |
|------|------|
| **연결 지향** | 데이터 전송 전 3-way Handshake로 연결 설정 |
| **신뢰성** | 데이터 도착 보장, 순서 보장, 오류 검출 |
| **흐름 제어** | 수신자의 처리 능력에 맞춰 전송 속도 조절 |
| **혼잡 제어** | 네트워크 혼잡 시 전송 속도 조절 |
| **전이중 통신** | 양방향 동시 데이터 전송 가능 |
| **바이트 스트림** | 데이터를 연속된 바이트 스트림으로 처리 |

### TCP 헤더 구조

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────────────┼───────────────────────────────┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┴───────────────────────────────┤
│                        Sequence Number                        │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                      │
├───────┬───────┬─┬─┬─┬─┬─┬─┬───────────────────────────────────┤
│ Data  │       │U│A│P│R│S│F│                                   │
│Offset │ Rsrvd │R│C│S│S│Y│I│            Window                 │
│(4bit) │(3bit) │G│K│H│T│N│N│           (16 bits)               │
├───────┴───────┴─┴─┴─┴─┴─┴─┼───────────────────────────────────┤
│         Checksum          │         Urgent Pointer            │
├───────────────────────────┴───────────────────────────────────┤
│                    Options (0-40 bytes)                       │
├───────────────────────────────────────────────────────────────┤
│                         Data (payload)                        │
└───────────────────────────────────────────────────────────────┘

최소 헤더 크기: 20 bytes
최대 헤더 크기: 60 bytes (옵션 포함)
```

### TCP 플래그 (Control Bits)

| 플래그 | 이름 | 설명 |
|--------|------|------|
| **URG** | Urgent | 긴급 데이터 포함, Urgent Pointer 유효 |
| **ACK** | Acknowledgment | 확인 응답, Acknowledgment Number 유효 |
| **PSH** | Push | 즉시 상위 계층으로 데이터 전달 요청 |
| **RST** | Reset | 연결 강제 종료 |
| **SYN** | Synchronize | 연결 설정 요청, 시퀀스 번호 동기화 |
| **FIN** | Finish | 연결 종료 요청 |

### TCP 상태 다이어그램

```
                              ┌─────────────┐
                              │   CLOSED    │
                              └──────┬──────┘
                   ┌────────────────┼────────────────┐
            passive open      active open        passive open
                   │                │                │
                   ▼                ▼                │
            ┌─────────────┐  ┌─────────────┐        │
            │   LISTEN    │  │  SYN_SENT   │        │
            └──────┬──────┘  └──────┬──────┘        │
                   │                │                │
             recv SYN         recv SYN+ACK          │
            send SYN+ACK        send ACK            │
                   │                │                │
                   ▼                ▼                │
            ┌─────────────┐  ┌─────────────┐        │
            │  SYN_RCVD   │──│ ESTABLISHED │◄───────┘
            └─────────────┘  └──────┬──────┘
                                    │
                   ┌────────────────┼────────────────┐
            close, send FIN   recv FIN, send ACK   close, send FIN
                   │                │                │
                   ▼                ▼                ▼
            ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
            │  FIN_WAIT_1 │  │ CLOSE_WAIT  │  │  FIN_WAIT_2 │
            └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
                   │                │                │
             recv ACK          close,           recv FIN,
                   │         send FIN           send ACK
                   ▼                │                │
            ┌─────────────┐        ▼                │
            │  FIN_WAIT_2 │  ┌─────────────┐        │
            └──────┬──────┘  │  LAST_ACK   │        │
                   │         └──────┬──────┘        │
             recv FIN,             │                │
            send ACK          recv ACK             │
                   │                │                │
                   ▼                ▼                ▼
            ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
            │  TIME_WAIT  │  │   CLOSED    │  │  TIME_WAIT  │
            └──────┬──────┘  └─────────────┘  └──────┬──────┘
                   │                                 │
              timeout=2MSL                      timeout=2MSL
                   │                                 │
                   ▼                                 ▼
            ┌─────────────┐                   ┌─────────────┐
            │   CLOSED    │                   │   CLOSED    │
            └─────────────┘                   └─────────────┘
```

---

## 3-way Handshake

### 개요

3-way Handshake는 TCP 연결을 설정하는 과정입니다. 양쪽 호스트가 서로의 초기 시퀀스 번호(ISN)를 교환하고 동기화합니다.

### 동작 과정

```
    Client                                           Server
      │                                                │
      │  1. SYN (seq=x)                               │
      │ ─────────────────────────────────────────────>│
      │                                                │
      │       SYN 플래그 설정                          │
      │       클라이언트 ISN = x                       │
      │                                                │
      │  2. SYN + ACK (seq=y, ack=x+1)                │
      │ <─────────────────────────────────────────────│
      │                                                │
      │       SYN, ACK 플래그 설정                     │
      │       서버 ISN = y                             │
      │       ack = x+1 (클라이언트 seq + 1)           │
      │                                                │
      │  3. ACK (seq=x+1, ack=y+1)                    │
      │ ─────────────────────────────────────────────>│
      │                                                │
      │       ACK 플래그 설정                          │
      │       ack = y+1 (서버 seq + 1)                │
      │                                                │
      │  ═══════════ ESTABLISHED ═══════════          │
      │                                                │
      ▼                                                ▼
```

### 왜 3-way Handshake가 필요한가?

RFC 793에서는 다음과 같이 설명합니다:

> "The principal reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion."

**주요 이유**:

1. **양방향 통신 확인**: 클라이언트 → 서버, 서버 → 클라이언트 양쪽 통신 가능 확인
2. **시퀀스 번호 동기화**: 양측의 ISN(Initial Sequence Number) 교환 및 동기화
3. **중복 연결 방지**: 과거의 지연된 SYN 패킷으로 인한 잘못된 연결 설정 방지
4. **리소스 낭비 방지**: 연결 의사 확인 후 리소스 할당

### ISN (Initial Sequence Number)

```python
# ISN은 예측하기 어렵도록 랜덤하게 생성됨
# 보안상의 이유로 단순 증가 방식을 사용하지 않음

# 이론적 ISN 생성 (실제 구현은 OS마다 다름)
import time
import random

def generate_isn():
    """
    ISN 생성 예시
    - 시간 기반 + 랜덤 요소
    - 실제 구현은 더 복잡한 알고리즘 사용
    """
    time_component = int(time.time() * 250000) & 0xFFFFFFFF
    random_component = random.randint(0, 0xFFFF)
    return (time_component + random_component) & 0xFFFFFFFF
```

### 3-way Handshake 코드 예시

```python
import socket

# 서버 측
def tcp_server():
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind(('0.0.0.0', 8080))
    server_socket.listen(5)  # SYN 큐 크기 설정

    print("서버 대기 중... (LISTEN 상태)")

    # accept()에서 3-way handshake 완료
    client_socket, addr = server_socket.accept()
    print(f"연결 수립: {addr} (ESTABLISHED 상태)")

    # 데이터 통신
    data = client_socket.recv(1024)
    print(f"수신: {data.decode()}")

    client_socket.close()
    server_socket.close()

# 클라이언트 측
def tcp_client():
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # connect()에서 3-way handshake 수행
    # 1. SYN 전송
    # 2. SYN+ACK 수신
    # 3. ACK 전송
    client_socket.connect(('127.0.0.1', 8080))
    print("연결 완료 (ESTABLISHED 상태)")

    client_socket.send(b"Hello, Server!")
    client_socket.close()
```

---

## 4-way Handshake

### 개요

4-way Handshake는 TCP 연결을 종료하는 과정입니다. 양쪽 모두 데이터 전송 완료를 확인한 후 연결을 안전하게 종료합니다.

### 동작 과정

```
    Client                                           Server
      │                                                │
      │  1. FIN (seq=u)                               │
      │ ─────────────────────────────────────────────>│
      │                                                │
      │       FIN_WAIT_1 상태                          │
      │       "더 이상 보낼 데이터 없음"                 │
      │                                                │
      │  2. ACK (ack=u+1)                             │
      │ <─────────────────────────────────────────────│
      │                                                │
      │       FIN_WAIT_2 상태          CLOSE_WAIT 상태 │
      │                               (서버는 아직      │
      │                                데이터 전송 가능) │
      │                                                │
      │  3. FIN (seq=w)                               │
      │ <─────────────────────────────────────────────│
      │                                                │
      │                               LAST_ACK 상태    │
      │       "서버도 데이터 전송 완료"                  │
      │                                                │
      │  4. ACK (ack=w+1)                             │
      │ ─────────────────────────────────────────────>│
      │                                                │
      │       TIME_WAIT 상태           CLOSED 상태     │
      │       (2MSL 대기)                              │
      │                                                │
      │       ─────── 2MSL timeout ───────            │
      │                                                │
      │       CLOSED 상태                              │
      ▼                                                ▼
```

### 왜 4-way Handshake가 필요한가?

**3-way가 아닌 4-way인 이유**:
- TCP는 전이중(Full-Duplex) 통신
- 각 방향의 연결을 독립적으로 종료해야 함
- 한쪽이 FIN을 보내도 반대쪽은 계속 데이터 전송 가능 (Half-Close)

### TIME_WAIT 상태

```
TIME_WAIT 상태의 필요성:

1. 지연된 패킷 처리
   ┌──────────────────────────────────────────────────────────┐
   │  이전 연결의 지연된 패킷이 새 연결에 영향주는 것을 방지    │
   │  2MSL (Maximum Segment Lifetime) 동안 대기                │
   │  일반적으로 MSL = 60초, 따라서 TIME_WAIT = 120초          │
   └──────────────────────────────────────────────────────────┘

2. 마지막 ACK 손실 대비
   ┌──────────────────────────────────────────────────────────┐
   │  마지막 ACK가 손실되면 서버는 FIN을 재전송함              │
   │  클라이언트가 TIME_WAIT에서 이를 처리할 수 있어야 함      │
   └──────────────────────────────────────────────────────────┘
```

### TIME_WAIT 관련 이슈 및 해결

```bash
# TIME_WAIT 상태 확인
netstat -an | grep TIME_WAIT | wc -l

# 많은 TIME_WAIT이 문제가 되는 경우:
# - 서버가 클라이언트 역할을 할 때 (외부 API 호출 등)
# - 포트 고갈 가능성

# Linux 커널 파라미터 조정
# /etc/sysctl.conf

# TIME_WAIT 소켓 재사용 허용
net.ipv4.tcp_tw_reuse = 1

# TIME_WAIT 타임아웃 (기본값 유지 권장)
# net.ipv4.tcp_fin_timeout = 60
```

```python
# 서버에서 TIME_WAIT 줄이기
import socket

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# SO_REUSEADDR: TIME_WAIT 상태의 포트 재사용 허용
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# SO_LINGER: 연결 종료 방식 제어
# l_onoff=1, l_linger=0: RST로 즉시 종료 (TIME_WAIT 없음, 주의 필요)
# server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER,
#                          struct.pack('ii', 1, 0))

server_socket.bind(('0.0.0.0', 8080))
```

---

## 흐름 제어 (Flow Control)

### 개요

흐름 제어는 송신자가 수신자의 처리 능력을 초과하지 않도록 데이터 전송 속도를 조절하는 메커니즘입니다.

### Sliding Window

TCP는 Sliding Window 메커니즘을 사용하여 흐름 제어를 구현합니다.

```
Sliding Window 개념:

송신자 버퍼:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  ▲                       ▲                   ▲
  │     Sent but not     │    Can be sent    │
  │     ACKed            │                   │
  │                      │                   │
  └──────────────────────┴───────────────────┘
        Sending Window (rwnd)

수신자 버퍼:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │   │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┘
  ▲               ▲               ▲
  │  Received &   │   Available   │
  │  Processed    │   Buffer      │
  │               │               │
  └───────────────┴───────────────┘
       Receive Window (rwnd)
```

### Window Size 동작

```
Window Size 변화 예시:

1. 초기 상태 (rwnd = 4000 bytes)
   Client                              Server
      │                                   │
      │  ────── SEQ=1, 1000 bytes ──────> │
      │  ────── SEQ=1001, 1000 bytes ──>  │
      │                                   │
      │  <────── ACK=2001, Win=2000 ───── │
      │                                   │
      │  (서버 버퍼 2000 bytes 사용 중)     │
      │  (수신 가능: 2000 bytes)           │
      │                                   │

2. 수신자 버퍼 가득 참 (Zero Window)
   Client                              Server
      │                                   │
      │  ────── SEQ=2001, 2000 bytes ──>  │
      │                                   │
      │  <────── ACK=4001, Win=0 ─────── │
      │                                   │
      │  (서버 버퍼 가득 참)               │
      │  (Window Probe 전송 시작)          │
      │                                   │

3. Window Update
   Client                              Server
      │                                   │
      │  ────── Window Probe ──────────>  │
      │                                   │
      │  <────── ACK=4001, Win=4000 ───── │
      │                                   │
      │  (서버 버퍼 비워짐)                │
      │  (전송 재개)                       │
```

### Silly Window Syndrome 방지

```
문제: 작은 크기의 데이터가 반복적으로 전송되어 효율 저하

해결 방법:

1. Nagle 알고리즘 (송신 측)
   - 작은 데이터를 모아서 한 번에 전송
   - 첫 번째 세그먼트 즉시 전송
   - ACK 받기 전까지 후속 데이터 버퍼링
   - MSS 이상 모이면 전송

2. Clark 해결책 (수신 측)
   - 버퍼 공간이 MSS 이상이거나 절반 이상 비었을 때만
     Window Update 광고

3. 지연 ACK (Delayed ACK)
   - ACK를 즉시 보내지 않고 잠시 대기
   - 보낼 데이터와 함께 ACK 전송 (Piggybacking)
```

```python
import socket

# Nagle 알고리즘 비활성화 (실시간성이 중요한 경우)
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

# TCP_CORK: Nagle과 유사하지만 더 공격적인 버퍼링
# Linux 전용
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_CORK, 1)
# ... 데이터 전송 ...
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_CORK, 0)  # flush
```

---

## 혼잡 제어 (Congestion Control)

### 개요

혼잡 제어는 네트워크 전체의 혼잡을 방지하기 위해 송신자가 데이터 전송 속도를 조절하는 메커니즘입니다. 흐름 제어가 수신자 기준이라면, 혼잡 제어는 네트워크 기준입니다.

### 혼잡 윈도우 (Congestion Window, cwnd)

```
실제 전송 가능한 데이터 양:
min(cwnd, rwnd)

- cwnd: 혼잡 윈도우 (송신자가 네트워크 상태에 따라 결정)
- rwnd: 수신 윈도우 (수신자가 광고)
```

### 혼잡 제어 알고리즘

#### 1. Slow Start

```
Slow Start 동작:

cwnd 변화:
│cwnd
│  (MSS 단위)
│
│ 64 ─┐                               ┌─
│     │                               │
│ 32 ─┤                         ┌─────┘
│     │                         │
│ 16 ─┤                   ┌─────┘
│     │                   │
│  8 ─┤             ┌─────┘
│     │             │
│  4 ─┤       ┌─────┘
│     │       │
│  2 ─┤   ┌───┘
│     │   │
│  1 ─┼───┘
│     │
└─────┴─────┬─────┬─────┬─────┬─────┬─── RTT
            1     2     3     4     5

- 초기 cwnd = 1 MSS (또는 IW: Initial Window)
- 매 ACK 수신 시 cwnd += 1 MSS
- 결과적으로 RTT당 cwnd가 2배로 증가 (지수적 증가)
- ssthresh(slow start threshold) 도달 시 Congestion Avoidance로 전환
```

#### 2. Congestion Avoidance (AIMD)

```
AIMD (Additive Increase Multiplicative Decrease):

│cwnd
│
│    ┌──┐ 패킷 손실!
│   ╱    ╲
│  ╱      ╲  ssthresh = cwnd/2
│ ╱        ╲ cwnd = ssthresh
│╱          ╲────────────────────
│            ╲
│             ╲───────╱╲
│                    ╱  ╲
│                   ╱    ╲
│                  ╱      ╲
└──────────────────────────────────── time

Additive Increase (가산 증가):
- 매 RTT마다 cwnd += 1 MSS
- 선형적 증가

Multiplicative Decrease (승산 감소):
- 패킷 손실 감지 시 ssthresh = cwnd / 2
- cwnd = ssthresh (또는 1 MSS, 알고리즘에 따라 다름)
```

#### 3. Fast Retransmit

```
Fast Retransmit 동작:

송신자                                     수신자
   │                                          │
   │ ──────── SEQ=1 ────────────────────────> │
   │ ──────── SEQ=2 ────────────────────────> │
   │ ──────── SEQ=3 (손실) ────────X          │
   │ ──────── SEQ=4 ────────────────────────> │
   │                                          │
   │ <──────── ACK=3 (1번째 중복 ACK) ─────── │
   │                                          │
   │ ──────── SEQ=5 ────────────────────────> │
   │                                          │
   │ <──────── ACK=3 (2번째 중복 ACK) ─────── │
   │                                          │
   │ ──────── SEQ=6 ────────────────────────> │
   │                                          │
   │ <──────── ACK=3 (3번째 중복 ACK) ─────── │
   │                                          │
   │ ═════ 3개 중복 ACK → 즉시 재전송! ═════   │
   │                                          │
   │ ──────── SEQ=3 (재전송) ───────────────> │
   │                                          │

- Timeout을 기다리지 않고 3개의 중복 ACK 수신 시 즉시 재전송
- 네트워크가 여전히 동작 중임을 의미 (패킷이 도착하고 있으므로)
```

#### 4. Fast Recovery

```
Fast Recovery (TCP Reno):

│cwnd
│
│    ssthresh (새로운)
│    ↓
│    ├───────────────────────────┐
│    │                           │
│    │   Fast Recovery           │ Congestion
│ ───┼─────────────────────      │ Avoidance
│    │  3 중복 ACK               │
│    │  cwnd = ssthresh + 3      │
│    │                           │
│    │                           │
│────┴───────────────────────────┴──────── time
    손실                        새 ACK 수신
    발생                        (cwnd = ssthresh)

- 3개 중복 ACK 시 cwnd를 1로 줄이지 않음
- ssthresh = cwnd / 2
- cwnd = ssthresh + 3 (중복 ACK 수만큼)
- Slow Start를 건너뛰고 바로 Congestion Avoidance
```

### TCP 혼잡 제어 변형

```
┌─────────────────────────────────────────────────────────────────────┐
│                    TCP 혼잡 제어 알고리즘 비교                        │
├───────────┬─────────────────────────────────────────────────────────┤
│ TCP Tahoe │ - Slow Start + Congestion Avoidance + Fast Retransmit  │
│ (1988)    │ - 손실 시 cwnd = 1로 리셋                               │
├───────────┼─────────────────────────────────────────────────────────┤
│ TCP Reno  │ - Tahoe + Fast Recovery                                │
│ (1990)    │ - 3 중복 ACK 시 cwnd = ssthresh로 설정                  │
├───────────┼─────────────────────────────────────────────────────────┤
│ TCP New   │ - 부분적 ACK 처리 개선                                  │
│ Reno      │ - 여러 패킷 손실 시 성능 개선                            │
├───────────┼─────────────────────────────────────────────────────────┤
│ TCP CUBIC │ - Linux 기본 알고리즘                                   │
│ (현재)    │ - 고대역폭, 고지연 네트워크에 최적화                     │
│           │ - 3차 함수 기반 cwnd 증가                                │
├───────────┼─────────────────────────────────────────────────────────┤
│ TCP BBR   │ - Google 개발                                           │
│ (2016)    │ - 손실 기반이 아닌 대역폭/RTT 측정 기반                  │
│           │ - 버퍼블로트 문제 해결                                   │
└───────────┴─────────────────────────────────────────────────────────┘
```

---

## UDP (User Datagram Protocol)

### UDP 개요

UDP는 비연결형(Connectionless) 프로토콜로, 신뢰성보다 속도를 우선시합니다. RFC 768에 정의되어 있으며, 1980년 8월에 발표되었습니다.

### UDP 특징

| 특징 | 설명 |
|------|------|
| **비연결형** | 연결 설정 과정 없이 즉시 데이터 전송 |
| **신뢰성 없음** | 데이터 도착, 순서 보장 없음 |
| **경량** | 헤더 오버헤드 최소 (8 bytes) |
| **빠른 전송** | 연결 설정/해제 오버헤드 없음 |
| **브로드캐스트/멀티캐스트** | 일대다 통신 지원 |
| **메시지 경계 유지** | 전송 단위 그대로 수신 |

### UDP 헤더 구조

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────────────┼───────────────────────────────┤
│          Source Port          │       Destination Port        │
├───────────────────────────────┼───────────────────────────────┤
│            Length             │           Checksum            │
├───────────────────────────────┴───────────────────────────────┤
│                             Data                              │
└───────────────────────────────────────────────────────────────┘

헤더 크기: 8 bytes (TCP의 20+ bytes와 비교)

- Source Port (16 bits): 송신자 포트
- Destination Port (16 bits): 수신자 포트
- Length (16 bits): UDP 헤더 + 데이터 길이
- Checksum (16 bits): 오류 검출 (IPv4에서는 선택, IPv6에서는 필수)
```

### UDP 코드 예시

```python
import socket

# UDP 서버
def udp_server():
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    server_socket.bind(('0.0.0.0', 8080))

    print("UDP 서버 대기 중...")

    while True:
        # 연결 설정 없이 바로 데이터 수신
        data, client_addr = server_socket.recvfrom(1024)
        print(f"수신 from {client_addr}: {data.decode()}")

        # 응답 전송
        server_socket.sendto(b"ACK", client_addr)

# UDP 클라이언트
def udp_client():
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

    # connect() 없이 바로 전송
    server_addr = ('127.0.0.1', 8080)
    client_socket.sendto(b"Hello, UDP Server!", server_addr)

    # 응답 수신 (타임아웃 설정 권장)
    client_socket.settimeout(5.0)
    try:
        data, addr = client_socket.recvfrom(1024)
        print(f"응답: {data.decode()}")
    except socket.timeout:
        print("응답 타임아웃")

    client_socket.close()
```

---

## TCP vs UDP 비교

### 상세 비교표

| 구분 | TCP | UDP |
|------|-----|-----|
| **연결 방식** | 연결 지향 (3-way Handshake) | 비연결 |
| **신뢰성** | 보장 (재전송, 순서 보장) | 보장 안함 |
| **헤더 크기** | 20-60 bytes | 8 bytes |
| **속도** | 상대적으로 느림 | 빠름 |
| **흐름 제어** | 있음 (Sliding Window) | 없음 |
| **혼잡 제어** | 있음 (AIMD, Slow Start 등) | 없음 |
| **순서 보장** | 보장 | 보장 안함 |
| **오류 검출** | Checksum + 재전송 | Checksum만 |
| **전송 단위** | 바이트 스트림 | 데이터그램 (메시지) |
| **브로드캐스트** | 지원 안함 | 지원 |
| **멀티캐스트** | 지원 안함 | 지원 |
| **적용 분야** | 웹, 이메일, 파일 전송 | DNS, 스트리밍, 게임, VoIP |

### 시각적 비교

```
TCP 통신 과정:
┌────────┐                                    ┌────────┐
│ Client │                                    │ Server │
└───┬────┘                                    └───┬────┘
    │                                             │
    │  ─────── SYN ──────────────────────────>   │
    │  <────── SYN+ACK ────────────────────────  │
    │  ─────── ACK ──────────────────────────>   │  연결 설정
    │                                             │  (3-way)
    │  ═══════ DATA ═══════════════════════>     │
    │  <═══════ ACK ════════════════════════     │  데이터 전송
    │  ═══════ DATA ═══════════════════════>     │  (신뢰성 보장)
    │  <═══════ ACK ════════════════════════     │
    │                                             │
    │  ─────── FIN ──────────────────────────>   │
    │  <────── ACK ────────────────────────────  │  연결 종료
    │  <────── FIN ────────────────────────────  │  (4-way)
    │  ─────── ACK ──────────────────────────>   │
    ▼                                             ▼

UDP 통신 과정:
┌────────┐                                    ┌────────┐
│ Client │                                    │ Server │
└───┬────┘                                    └───┬────┘
    │                                             │
    │  ═══════ DATA ═══════════════════════>     │
    │  ═══════ DATA ═══════════════════════>     │  바로 전송
    │  ═══════ DATA ═══════════════════════>     │  (연결 없음)
    │                                             │
    ▼                                             ▼
```

---

## UDP 사용 사례

### 1. DNS (Domain Name System)

```
DNS에서 UDP를 사용하는 이유:

1. 요청/응답이 단순하고 작음 (512 bytes 이하)
2. 연결 설정 오버헤드 제거로 빠른 응답
3. DNS 서버의 다수 클라이언트 처리 용이
4. 응답 없으면 클라이언트가 재시도

※ 응답이 512 bytes 초과 시 TCP 사용 (Zone Transfer 등)
```

```python
# DNS 쿼리 예시 (UDP 포트 53)
import socket
import struct

def simple_dns_query(domain, dns_server='8.8.8.8'):
    """간단한 DNS A 레코드 쿼리"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(5.0)

    # DNS 쿼리 패킷 구성 (간략화)
    transaction_id = 0x1234
    flags = 0x0100  # Standard query
    questions = 1

    # ... DNS 패킷 생성 로직 ...

    sock.sendto(query_packet, (dns_server, 53))
    response, _ = sock.recvfrom(512)

    # ... 응답 파싱 ...
    return parsed_response
```

### 2. 실시간 스트리밍

```
스트리밍에서 UDP를 사용하는 이유:

1. 실시간성이 중요 (지연 < 신뢰성)
2. 일부 패킷 손실은 허용 가능
3. 재전송으로 인한 지연은 더 나쁜 사용자 경험 제공
4. 버퍼링으로 일부 손실 보상 가능

사용 프로토콜:
- RTP (Real-time Transport Protocol)
- RTSP (Real-Time Streaming Protocol)
- WebRTC (브라우저 실시간 통신)
```

```
비디오 스트리밍 패킷 손실 시나리오:

TCP 사용 시:
Frame 1 ─> [수신] ─> 재생
Frame 2 ─> [손실] ─> 재전송 대기... ─> 버퍼링... ─> 재생
Frame 3 ─> [대기] ─────────────────────> 재생
                     ↑
                   지연 발생!

UDP 사용 시:
Frame 1 ─> [수신] ─> 재생
Frame 2 ─> [손실] ─> 건너뛰기 (약간의 화질 저하)
Frame 3 ─> [수신] ─> 즉시 재생
                     ↑
                  실시간 유지!
```

### 3. 온라인 게임

```
게임에서 UDP를 사용하는 이유:

1. 실시간 위치/상태 업데이트 (최신 정보가 중요)
2. 오래된 패킷은 의미 없음 (재전송 불필요)
3. 낮은 지연시간 필수
4. 작은 패킷 빈번한 전송

게임 네트워크 설계:
- 위치 동기화: UDP (실시간)
- 채팅: TCP (신뢰성)
- 결제/인증: TCP (신뢰성)
```

```python
# 게임 서버 UDP 예시
import socket
import json
import time

class GameServer:
    def __init__(self, host='0.0.0.0', port=7777):
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.socket.bind((host, port))
        self.players = {}  # {client_addr: player_state}

    def broadcast_game_state(self):
        """모든 플레이어에게 게임 상태 브로드캐스트"""
        game_state = {
            'timestamp': time.time(),
            'players': self.players
        }
        data = json.dumps(game_state).encode()

        for addr in self.players:
            self.socket.sendto(data, addr)

    def handle_player_update(self, data, addr):
        """플레이어 위치 업데이트 처리"""
        update = json.loads(data.decode())

        # 오래된 패킷 무시 (최신 정보만 반영)
        if addr in self.players:
            if update['timestamp'] < self.players[addr].get('timestamp', 0):
                return  # 오래된 패킷 무시

        self.players[addr] = update
```

### 4. VoIP (Voice over IP)

```
VoIP에서 UDP를 사용하는 이유:

1. 실시간 음성 전달 필수
2. 약간의 패킷 손실은 허용 (음질 저하지만 통화 가능)
3. 20-30ms 이하 지연 필요
4. TCP 재전송으로 인한 지연은 통화 품질 심각하게 저하

사용 프로토콜:
- RTP over UDP (음성 데이터)
- SIP over UDP/TCP (시그널링)
```

### 5. QUIC 프로토콜 (HTTP/3)

```
QUIC: UDP 기반의 신뢰성 있는 프로토콜

┌─────────────────────────────────────────────────────────────┐
│                        QUIC 특징                            │
├─────────────────────────────────────────────────────────────┤
│ • UDP 위에서 동작하지만 TCP와 유사한 신뢰성 제공              │
│ • 연결 설정 시간 단축 (0-RTT, 1-RTT)                        │
│ • Head-of-Line Blocking 문제 해결                          │
│ • 내장 TLS 1.3 암호화                                       │
│ • 연결 마이그레이션 지원                                     │
└─────────────────────────────────────────────────────────────┘

HTTP/2 over TCP vs HTTP/3 over QUIC:

TCP + TLS 1.3:
Client ──SYN──> Server
Client <─SYN/ACK─ Server
Client ──ACK──> Server      } 1 RTT (TCP)
Client ──ClientHello──> Server
Client <─ServerHello─ Server } 1-2 RTT (TLS)
총 2-3 RTT

QUIC (1-RTT):
Client ──Initial──> Server
Client <─Handshake─ Server  } 1 RTT (통합)
총 1 RTT
```

---

## 실무 적용

### TCP 튜닝 (Linux)

```bash
# /etc/sysctl.conf 또는 /etc/sysctl.d/99-tcp-tuning.conf

# TCP 버퍼 크기 조정 (대역폭 * RTT)
# BDP = Bandwidth (bytes/sec) * RTT (sec)
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# TCP 혼잡 제어 알고리즘
net.ipv4.tcp_congestion_control = bbr  # 또는 cubic

# TIME_WAIT 관련
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 30

# SYN 플러드 방어
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 4096

# Keep-Alive 설정
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15

# 적용
sudo sysctl -p
```

### 네트워크 상태 모니터링

```bash
# TCP 연결 상태 확인
ss -s  # 요약
ss -tan  # 모든 TCP 연결
ss -tan state established  # ESTABLISHED 상태만
ss -tan state time-wait | wc -l  # TIME_WAIT 개수

# netstat (legacy)
netstat -an | grep ESTABLISHED | wc -l

# TCP 재전송 확인
netstat -s | grep -i retrans

# 혼잡 제어 알고리즘 확인
sysctl net.ipv4.tcp_congestion_control

# 네트워크 처리량 모니터링
sar -n DEV 1  # 1초 간격으로 네트워크 통계
iftop         # 실시간 대역폭 모니터링
```

### 패킷 캡처 및 분석

```bash
# tcpdump로 TCP 핸드셰이크 캡처
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0' -n

# 특정 호스트와의 TCP 통신
sudo tcpdump -i eth0 host 192.168.1.100 and tcp -w capture.pcap

# UDP 트래픽 캡처
sudo tcpdump -i eth0 udp port 53 -n

# Wireshark 분석 필터
# TCP 재전송
tcp.analysis.retransmission

# 중복 ACK
tcp.analysis.duplicate_ack

# Zero Window
tcp.analysis.zero_window
```

---

## 참고 자료

- [RFC 793 - Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793)
- [RFC 768 - User Datagram Protocol](https://datatracker.ietf.org/doc/html/rfc768)
- [RFC 5681 - TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681)
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://datatracker.ietf.org/doc/html/rfc9000)
- [Wikipedia - Transmission Control Protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
- TCP/IP Illustrated, Volume 1: The Protocols (W. Richard Stevens)

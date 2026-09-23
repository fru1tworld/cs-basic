# TCP

TCP는 연결 안에서 순서가 보장되는 신뢰성 있는 바이트 스트림을 제공한다. 손실된 데이터를 재전송하더라도 영구적인 네트워크 장애가 나면 전달을 완료하지 못할 수 있다. 또한 TCP의 ACK를 받았다는 것과 상대 애플리케이션이 업무 처리를 마쳤다는 것은 구분해야 한다. [TCP 명세 RFC 9293](https://www.rfc-editor.org/rfc/rfc9293.html)

- 핵심 특징:
- 연결 지향 (Connection-oriented)
- 신뢰성 보장 (Reliable delivery)
- 순서 보장 (Ordered delivery)
- 흐름 제어 (Flow control)
- 혼잡 제어 (Congestion control)

## TCP 개요

- TCP는 연결 지향적(Connection-Oriented) 프로토콜로, 신뢰성 있는 데이터 전송을 보장함. RFC 793에 정의되어 있으며, 1981년 9월에 발표되었음.

## TCP의 특성

- 연결 지향:
  - 설명: 데이터 전송 전 3-way Handshake로 연결 설정
- 신뢰성:
  - 설명: 손실 재전송, 순서 복원, 오류 검출을 제공하며 복구 불가능한 장애에서는 연결 실패가 발생할 수 있음
- 흐름 제어:
  - 설명: 수신자의 처리 능력에 맞춰 전송 속도 조절
- 혼잡 제어:
  - 설명: 네트워크 혼잡 시 전송 속도 조절
- 전이중 통신:
  - 설명: 양방향 동시 데이터 전송 가능
- 바이트 스트림:
  - 설명: 데이터를 연속된 바이트 스트림으로 처리

## TCP 헤더 구조

- 고정 부분은 20바이트이며, 옵션과 패딩을 포함한 최대 헤더 길이는 60바이트임.
- 필드는 다음 순서로 배치됨. 제어 비트의 배치는 [RFC 9293의 헤더 정의](https://www.rfc-editor.org/rfc/rfc9293.html#section-3.1)를 기준으로 설명함.
  - Source Port와 Destination Port: 각각 16비트이며 송신과 수신 프로세스의 포트를 식별함.
  - Sequence Number: 32비트이며 세그먼트의 시퀀스 번호임.
  - Acknowledgment Number: 32비트이며 ACK 플래그가 설정되었을 때 다음에 기대하는 시퀀스 번호를 나타냄.
  - Data Offset: 4비트이며 헤더 길이를 32비트 워드 단위로 나타냄.
  - Reserved: RFC 9293에서는 4비트로 정의함.
  - Control Bits: CWR, ECE, URG, ACK, PSH, RST, SYN, FIN의 8비트임.
  - Window: 16비트이며 수신 윈도우를 알림. Window Scale을 협상한 경우 실제 크기를 해석할 때 배율을 적용함.
  - Checksum과 Urgent Pointer: 각각 16비트임. 체크섬은 오류 검출에 사용하며 URG가 설정되면 긴급 포인터가 유효함.
  - Options와 Padding: 최대 40바이트이며 패딩은 헤더를 32비트 경계에 맞춤.
- Data는 헤더 뒤의 페이로드임.

## TCP 플래그 (Control Bits)

- 플래그: URG:
  - 이름: Urgent
  - 설명: 긴급 데이터 포함, Urgent Pointer 유효
- 플래그: ACK:
  - 이름: Acknowledgment
  - 설명: 확인 응답, Acknowledgment Number 유효
- 플래그: PSH:
  - 이름: Push
  - 설명: 즉시 상위 계층으로 데이터 전달 요청
- 플래그: RST:
  - 이름: Reset
  - 설명: 연결 강제 종료
- 플래그: SYN:
  - 이름: Synchronize
  - 설명: 연결 설정 요청, 시퀀스 번호 동기화
- 플래그: FIN:
  - 이름: Finish
  - 설명: 연결 종료 요청

## TCP 상태 다이어그램

- CLOSED는 연결이 없는 상태임.
- 수동 열기에서는 LISTEN으로 들어가 SYN을 기다림. SYN을 받으면 SYN과 ACK를 보내고 SYN_RCVD로 이동하며, 상대 ACK를 받으면 ESTABLISHED가 됨.
- 능동 열기에서는 SYN을 보내고 SYN_SENT가 됨. SYN과 ACK를 받으면 ACK를 보내고 ESTABLISHED로 이동함.
- ESTABLISHED에서 먼저 종료하는 쪽은 FIN을 보내고 FIN_WAIT_1로 이동함.
  - 자신의 FIN에 대한 ACK를 받으면 FIN_WAIT_2가 됨.
  - 상대 FIN을 받으면 ACK를 보내고 TIME_WAIT로 이동함.
  - 자신의 FIN에 대한 ACK보다 상대 FIN이 먼저 도착하는 동시 종료에서는 CLOSING을 거쳐 ACK를 받은 뒤 TIME_WAIT로 이동할 수 있음.
- 상대의 FIN을 먼저 받은 쪽은 ACK를 보내고 CLOSE_WAIT에 머무름. 애플리케이션이 종료하면 FIN을 보내고 LAST_ACK로 이동하며, 상대 ACK를 받으면 CLOSED가 됨.
- TIME_WAIT는 2MSL 대기 후 CLOSED로 이동함.

### TIME_WAIT 상태

- 연결 종료 후 2MSL(Maximum Segment Lifetime) 동안 대기
- 보통 60초 ~ 4분
- 목적:
  - 지연된 패킷 처리 (같은 포트로 새 연결 시 혼란 방지)
  - 마지막 ACK 손실 시 재전송 가능

## 3-way Handshake

### 개요

TCP는 데이터를 주고받기 전에 3-way Handshake로 연결을 설정한다. 이 과정에서 양쪽 호스트가 초기 시퀀스 번호(ISN)를 교환해 이후 데이터의 순서를 확인할 기준을 맞춘다.

### 동작 과정

1. 클라이언트는 자신의 ISN을 `x`로 정하고 `SYN(seq=x)`을 보낸다.
2. 서버는 자신의 ISN `y`와 클라이언트의 SYN을 확인하는 번호를 담아 `SYN+ACK(seq=y, ack=x+1)`을 보낸다.
3. 클라이언트는 `ACK(seq=x+1, ack=y+1)`로 서버의 SYN을 확인한다. 서버도 이 ACK를 받으면 연결이 수립된다.

### 왜 3-way Handshake가 필요한가?

- RFC 793에서는 다음과 같이 설명함:

- "The principal reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion."

- 주요 이유:

- 양방향 통신 확인: 클라이언트 → 서버, 서버 → 클라이언트 양쪽 통신 가능 확인
- 시퀀스 번호 동기화: 양측의 ISN(Initial Sequence Number) 교환 및 동기화
- 중복 연결 방지: 과거의 지연된 SYN 패킷으로 인한 잘못된 연결 설정 방지
- 리소스 낭비 방지: 연결 의사 확인 후 리소스 할당

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

## 4-way Handshake

### 개요

- 4-way Handshake는 TCP 연결을 종료하는 과정임. 양쪽 모두 데이터 전송 완료를 확인한 후 연결을 안전하게 종료함.

### 동작 과정

- Client, Server
- FIN (seq=u)
- FIN_WAIT_1 상태
- "더 이상 보낼 데이터 없음"
- ACK (ack=u+1)
- FIN_WAIT_2 상태, CLOSE_WAIT 상태
- (서버는 아직
- 데이터 전송 가능)
- FIN (seq=w)
- LAST_ACK 상태
- "서버도 데이터 전송 완료"
- ACK (ack=w+1)
- TIME_WAIT 상태, CLOSED 상태
- (2MSL 대기)
- 2MSL timeout
- CLOSED 상태

### 왜 4-way Handshake가 필요한가?

TCP는 양방향으로 데이터를 보내는 전이중(Full-Duplex) 통신이므로 각 방향의 전송을 독립적으로 종료한다. 한쪽이 FIN을 보내도 반대쪽에는 아직 보낼 데이터가 남아 있을 수 있다. 이처럼 한 방향만 먼저 닫는 상태를 Half-Close라고 한다.

### TIME_WAIT 상태

- TIME_WAIT 상태의 필요성:
- 지연된 패킷 처리
- 이전 연결의 지연된 패킷이 새 연결에 영향주는 것을 방지
- 2MSL (Maximum Segment Lifetime) 동안 대기
- 일반적으로 MSL = 60초, 따라서 TIME_WAIT = 120초
- 마지막 ACK 손실 대비
- 마지막 ACK가 손실되면 서버는 FIN을 재전송함
- 클라이언트가 TIME_WAIT에서 이를 처리할 수 있어야 함

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

## 흐름 제어 (Flow Control)

- 수신자의 처리 속도에 맞춰 송신 속도를 조절함.

### 개요

- 흐름 제어는 송신자가 수신자의 처리 능력을 초과하지 않도록 데이터 전송 속도를 조절하는 메커니즘임.

### Sliding Window

- TCP는 수신자가 알린 수신 윈도우 `rwnd`로 보내도 되는 데이터의 범위를 제한함.
- 송신 버퍼는 확인된 데이터, 전송했지만 아직 ACK를 받지 못한 데이터, 윈도우 안에서 추가로 전송 가능한 데이터, 윈도우 밖의 데이터로 구분됨.
- ACK가 진행하면 확인된 범위를 제외하면서 윈도우의 시작 위치도 앞으로 이동함.
- TCP 시퀀스 번호와 윈도우는 바이트 단위로 해석함.
- 별도의 1~10 예시에서 1과 2가 확인되고 윈도우 크기가 5라면 허용 범위는 3~7이며, 8~10은 현재 윈도우 밖임.
- 수신 버퍼에서 애플리케이션이 데이터를 읽어 공간을 확보하면 새 수신 윈도우를 알릴 수 있음.
- 수신 윈도우가 0이면 일반적인 새 데이터 전송을 멈추고, Window Probe로 수신 가능한 공간이 생겼는지 확인함.

### Window Size 동작

- Window Size 변화 예시:
- 초기 상태 (rwnd = 4000 bytes)
- Client, Server
- Client → SEQ=1, 1000 bytes → Server
- Client → SEQ=1001, 1000 bytes → Server
- Server → ACK=2001, Win=2000 → Client
- (서버 버퍼 2000 bytes 사용 중)
- (수신 가능: 2000 bytes)
- 수신자 버퍼 가득 참 (Zero Window)
- Client, Server
- Client → SEQ=2001, 2000 bytes → Server
- Server → ACK=4001, Win=0 → Client
- (서버 버퍼 가득 참)
- (Window Probe 전송 시작)
- Window Update
- Client, Server
- Client → Window Probe → Server
- Server → ACK=4001, Win=4000 → Client
- (서버 버퍼 비워짐)
- (전송 재개)

### Silly Window Syndrome 방지

- 문제: 작은 크기의 데이터가 반복적으로 전송되어 효율 저하
- 해결 방법:
- Nagle 알고리즘 (송신 측)
- 작은 데이터를 모아서 한 번에 전송
- 첫 번째 세그먼트 즉시 전송
- ACK 받기 전까지 후속 데이터 버퍼링
- MSS 이상 모이면 전송
- Clark 해결책 (수신 측)
- 버퍼 공간이 MSS 이상이거나 절반 이상 비었을 때만
- Window Update 광고
- 지연 ACK (Delayed ACK)
- ACK를 즉시 보내지 않고 잠시 대기
- 보낼 데이터와 함께 ACK 전송 (Piggybacking)

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

## 혼잡 제어 (Congestion Control)

- 네트워크 상황에 맞춰 전송 속도를 조절함.

### 개요

수신자에게 여유가 있어도 중간 네트워크에 데이터가 몰릴 수 있다. 혼잡 제어는 이런 상황에 맞춰 송신 속도를 조절한다. 수신자의 처리 능력을 기준으로 하는 흐름 제어와 함께 적용해야 한다.

### 혼잡 윈도우 (Congestion Window, cwnd)

- 실제 전송 가능한 데이터 양:
- min(cwnd, rwnd)
- cwnd: 혼잡 윈도우 (송신자가 네트워크 상태에 따라 결정)
- rwnd: 수신 윈도우 (수신자가 광고)

### 혼잡 제어 알고리즘

#### Slow Start

- Slow Start 동작:
- cwnd 변화:
- cwnd
- (MSS 단위)
- 64
- 32
- 16
- 8
- 4
- 2
- 1
- RTT
- 1, 2, 3, 4, 5
- 초기 cwnd = 1 MSS (또는 IW: Initial Window)
- 새로 전송한 데이터의 수신을 확인하는 ACK마다 cwnd를 늘림
- 세그먼트마다 ACK를 받는 경우 RTT당 cwnd가 대략 2배로 증가
- ssthresh(slow start threshold) 도달 시 Congestion Avoidance로 전환

#### Congestion Avoidance (AIMD)

- AIMD (Additive Increase Multiplicative Decrease):
- cwnd
- 패킷 손실!
- ssthresh = cwnd/2
- cwnd = ssthresh
- time
- Additive Increase (가산 증가):
- 매 RTT마다 cwnd += 1 MSS
- 선형적 증가
- Multiplicative Decrease (승산 감소):
- 패킷 손실 감지 시 ssthresh = cwnd / 2
- cwnd = ssthresh (또는 1 MSS, 알고리즘에 따라 다름)

#### Fast Retransmit

- Fast Retransmit 동작:
- 송신자, 수신자
- SEQ=1 →
- SEQ=2 →
- SEQ=3 (손실), X
- SEQ=4 →
- ← ACK=3 (1번째 중복 ACK)
- SEQ=5 →
- ← ACK=3 (2번째 중복 ACK)
- SEQ=6 →
- ← ACK=3 (3번째 중복 ACK)
- 3개 중복 ACK → 즉시 재전송!
- SEQ=3 (재전송) →
- Timeout을 기다리지 않고 3개의 중복 ACK 수신 시 즉시 재전송
- 네트워크가 여전히 동작 중임을 의미 (패킷이 도착하고 있으므로)

#### Fast Recovery

- Fast Recovery (TCP Reno):
- cwnd
- ssthresh (새로운)
- Fast Recovery, Congestion
- Avoidance
- 3 중복 ACK
- cwnd = ssthresh + 3
- time
- 손실, 새 ACK 수신
- 발생, (cwnd = ssthresh)
- 3개 중복 ACK 시 cwnd를 1로 줄이지 않음
- ssthresh = cwnd / 2
- cwnd = ssthresh + 3 (중복 ACK 수만큼)
- Slow Start를 건너뛰고 바로 Congestion Avoidance

### TCP 혼잡 제어 변형

- TCP 혼잡 제어 알고리즘 비교
- TCP Tahoe, - Slow Start + Congestion Avoidance + Fast Retransmit
- (1988), - 손실 시 cwnd = 1로 리셋
- TCP Reno, - Tahoe + Fast Recovery
- (1990), - 3 중복 ACK 시 cwnd = ssthresh로 설정
- TCP New, - 부분적 ACK 처리 개선
- Reno, - 여러 패킷 손실 시 성능 개선
- TCP CUBIC, - Linux 기본 알고리즘
- (현재), - 고대역폭, 고지연 네트워크에 최적화
- 3차 함수 기반 cwnd 증가
- TCP BBR, - Google 개발
- (2016), - 손실 기반이 아닌 대역폭/RTT 측정 기반
- 버퍼블로트 문제 해결

### CWND (Congestion Window)

- CWND, Congestion Avoidance
- / Slow Start
- /________________> Time

### 알고리즘

- 1. Slow Start
- 초기 CWND = 1 MSS (Maximum Segment Size)
- 새로 전송한 데이터의 수신을 확인하는 ACK마다 CWND 증가
- ssthresh(threshold)에 도달하면 Congestion Avoidance로 전환

Slow Start에서는 새 데이터의 수신을 확인하는 ACK를 받을 때마다 혼잡 윈도(cwnd)를 늘린다. 이렇게 늘어난 값을 RTT 단위로 보면, 세그먼트마다 ACK를 받는 경우 대략 두 배씩 커진다. ACK를 묶어서 보내는 경우에는 증가 속도가 달라질 수 있다.

- 2. Congestion Avoidance
- CWND를 선형적으로 증가 (RTT당 1 MSS)
- 패킷 손실 감지 시 CWND 감소

- 3. Fast Recovery
- 3 duplicate ACK 발생 시
- ssthresh = CWND / 2
- CWND = ssthresh + 3
- Congestion Avoidance로 전환

- CWND, \*
- /, \ (packet loss)
- /, \______ Fast Recovery
- ________________> Time

## TCP 3-Way Handshake (연결 수립)

- 클라이언트와 서버 간 연결을 수립하는 과정임.

- Client, Server
- Client → SYN (seq=x) , (1) 연결 요청 → Server
- Server → SYN+ACK (seq=y, ack=x+1) --, (2) 요청 수락 + 연결 요청 → Client
- Client → ACK (ack=y+1) , (3) 연결 확인 → Server
- [Connection Established]

### 단계별 설명

- Step 1: SYN (Synchronize)
- 클라이언트가 서버에 연결 요청
- 클라이언트의 초기 Sequence Number(ISN) 전송
- 클라이언트 상태: `CLOSED` → `SYN_SENT`

- Step 2: SYN + ACK
- 서버가 클라이언트 요청 수락
- 서버의 ISN 전송 + 클라이언트 ISN에 대한 ACK
- 서버 상태: `LISTEN` → `SYN_RECEIVED`

- Step 3: ACK
- 클라이언트가 서버의 SYN에 대한 ACK 전송
- 클라이언트 상태: `SYN_SENT` → `ESTABLISHED`
- 서버 상태: `SYN_RECEIVED` → `ESTABLISHED`

### 왜 3-Way인가?

- 2-Way의 문제:
- Client, Server
- Client → SYN , (지연됨) → Server
- Client → SYN → Server
- Server → SYN+ACK → Client
- (지연된 SYN 도착)
- Server → SYN+ACK, (불필요한 연결!) → Client

- 3-Way는 양쪽 모두 송신/수신 능력을 확인할 수 있음.

## TCP 4-Way Handshake (연결 종료)

- Client, Server
- Client → FIN (seq=x) , (1) 종료 요청 → Server
- Server → ACK (ack=x+1), (2) ACK → Client
- [Server 데이터 전송 완료]
- Server → FIN (seq=y), (3) 서버 종료 요청 → Client
- Client → ACK (ack=y+1) , (4) ACK → Server
- [Connection Closed]

### 왜 4-Way인가?

- 클라이언트가 FIN을 보내도 서버는 아직 보낼 데이터가 있을 수 있음
- 서버는 ACK 먼저 보내고, 데이터 전송 완료 후 FIN 전송
- Half-Close: 한쪽만 먼저 종료 가능

## 신뢰성 보장 메커니즘

### Sequence Number와 ACK

- Client, Server
- Client → Data(seq=1000, 100bytes) → Server
- Server → ACK(ack=1100), "1100번부터 보내줘" → Client
- Client → Data(seq=1100, 100bytes) → Server
- Server → ACK(ack=1200) → Client

### 재전송 (Retransmission)

- 1. 타임아웃 재전송
- Client, Server
- Client → Data(seq=1000) → Server
- (패킷 손실)
- [Timeout!]
- Client → Data(seq=1000) , (재전송) → Server
- Server → ACK(ack=1100) → Client

- 2. 빠른 재전송 (Fast Retransmit)
- Client, Server
- Client → Data(seq=1000) → Server
- Client → Data(seq=1100) , (손실) → Server
- Client → Data(seq=1200) → Server
- Server → ACK(ack=1100), (중복 ACK 1) → Client
- Client → Data(seq=1300) → Server
- Server → ACK(ack=1100), (중복 ACK 2) → Client
- Client → Data(seq=1400) → Server
- Server → ACK(ack=1100), (중복 ACK 3) → Client
- Client → [3 duplicate ACKs  재전송] → Server
- Client → Data(seq=1100) , (재전송) → Server

## 참고 자료

- [RFC 793 - Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793)
- [RFC 768 - User Datagram Protocol](https://datatracker.ietf.org/doc/html/rfc768)
- [RFC 5681 - TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681)
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://datatracker.ietf.org/doc/html/rfc9000)
- [Wikipedia - Transmission Control Protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
- TCP/IP Illustrated, Volume 1: The Protocols (W. Richard Stevens)

- [RFC 793 - TCP](https://tools.ietf.org/html/rfc793)
- [TCP/IP Illustrated, Volume 1](https://www.amazon.com/TCP-Illustrated-Vol-Addison-Wesley-Professional/dp/0201633469)

## 관련 학습

- [UDP](02-UDP.md)
- [전송 프로토콜 선택](03-전송-프로토콜-선택.md)
- [TLS와 HTTPS](../../보안/암호화와-전송-보안/02-TLS와-HTTPS.md)

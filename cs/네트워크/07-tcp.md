# TCP (Transmission Control Protocol)

## 개요

TCP는 **신뢰성 있는 데이터 전송**을 보장하는 전송 계층 프로토콜입니다.

**핵심 특징**:
- 연결 지향 (Connection-oriented)
- 신뢰성 보장 (Reliable delivery)
- 순서 보장 (Ordered delivery)
- 흐름 제어 (Flow control)
- 혼잡 제어 (Congestion control)

## 핵심 개념

### 1. TCP 세그먼트 구조

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 필드 | 크기 | 설명 |
|------|------|------|
| Source/Dest Port | 16bit each | 송신/수신 포트 |
| Sequence Number | 32bit | 데이터의 순서 번호 |
| ACK Number | 32bit | 다음에 받을 데이터 번호 |
| Flags | 6bit | SYN, ACK, FIN, RST, PSH, URG |
| Window | 16bit | 수신 윈도우 크기 (흐름 제어) |

### 2. TCP 3-Way Handshake (연결 수립)

클라이언트와 서버 간 연결을 수립하는 과정입니다.

```
Client                                Server
   |                                    |
   |  -------- SYN (seq=x) -------->    |  (1) 연결 요청
   |                                    |
   |  <--- SYN+ACK (seq=y, ack=x+1) --  |  (2) 요청 수락 + 연결 요청
   |                                    |
   |  -------- ACK (ack=y+1) ------->   |  (3) 연결 확인
   |                                    |
   |        [Connection Established]    |
```

#### 단계별 설명

**Step 1: SYN (Synchronize)**
- 클라이언트가 서버에 연결 요청
- 클라이언트의 초기 Sequence Number(ISN) 전송
- 클라이언트 상태: `CLOSED` → `SYN_SENT`

**Step 2: SYN + ACK**
- 서버가 클라이언트 요청 수락
- 서버의 ISN 전송 + 클라이언트 ISN에 대한 ACK
- 서버 상태: `LISTEN` → `SYN_RECEIVED`

**Step 3: ACK**
- 클라이언트가 서버의 SYN에 대한 ACK 전송
- 클라이언트 상태: `SYN_SENT` → `ESTABLISHED`
- 서버 상태: `SYN_RECEIVED` → `ESTABLISHED`

#### 왜 3-Way인가?

**2-Way의 문제**:
```
Client                    Server
   |  --- SYN --->           |  (지연됨)
   |  --- SYN --->           |
   |  <-- SYN+ACK ---        |
   |  (지연된 SYN 도착)
   |  <-- SYN+ACK ---        |  (불필요한 연결!)
```

3-Way는 양쪽 모두 **송신/수신 능력을 확인**할 수 있습니다.

### 3. TCP 4-Way Handshake (연결 종료)

```
Client                                Server
   |                                    |
   |  -------- FIN (seq=x) -------->    |  (1) 종료 요청
   |                                    |
   |  <------- ACK (ack=x+1) --------   |  (2) ACK
   |                                    |
   |        [Server 데이터 전송 완료]    |
   |                                    |
   |  <------- FIN (seq=y) ---------    |  (3) 서버 종료 요청
   |                                    |
   |  -------- ACK (ack=y+1) ------->   |  (4) ACK
   |                                    |
   |        [Connection Closed]         |
```

#### 왜 4-Way인가?

- 클라이언트가 FIN을 보내도 서버는 아직 보낼 데이터가 있을 수 있음
- 서버는 ACK 먼저 보내고, 데이터 전송 완료 후 FIN 전송
- **Half-Close**: 한쪽만 먼저 종료 가능

### 4. TCP 상태 다이어그램

```
                              +---------+
                              |  CLOSED |
                              +---------+
                                   |
                         (passive open)
                                   |
                                   v
          (active open)      +---------+
     +-------------------->  |  LISTEN |
     |                       +---------+
     |                            |
     |                       (rcv SYN, send SYN+ACK)
     |                            |
     v                            v
+---------+                  +---------+
|SYN_SENT |                  |SYN_RCVD |
+---------+                  +---------+
     |                            |
(rcv SYN+ACK, send ACK)      (rcv ACK)
     |                            |
     +-----------> +---------+ <--+
                   |ESTABLISHED|
                   +---------+
                        |
              (close, send FIN)
                        |
                        v
                   +---------+
                   |FIN_WAIT1|
                   +---------+
                        |
                   (rcv ACK)
                        |
                        v
                   +---------+
                   |FIN_WAIT2|
                   +---------+
                        |
                   (rcv FIN, send ACK)
                        |
                        v
                   +---------+
                   |TIME_WAIT|
                   +---------+
                        |
                   (2MSL timeout)
                        |
                        v
                   +---------+
                   |  CLOSED |
                   +---------+
```

#### TIME_WAIT 상태

- 연결 종료 후 **2MSL(Maximum Segment Lifetime)** 동안 대기
- 보통 60초 ~ 4분
- **목적**:
  1. 지연된 패킷 처리 (같은 포트로 새 연결 시 혼란 방지)
  2. 마지막 ACK 손실 시 재전송 가능

### 5. 신뢰성 보장 메커니즘

#### Sequence Number와 ACK

```
Client                              Server
   |  -- Data(seq=1000, 100bytes) -->  |
   |  <------ ACK(ack=1100) ---------  |  "1100번부터 보내줘"
   |  -- Data(seq=1100, 100bytes) -->  |
   |  <------ ACK(ack=1200) ---------  |
```

#### 재전송 (Retransmission)

**1. 타임아웃 재전송**
```
Client                              Server
   |  -- Data(seq=1000) ---------->    |
   |        (패킷 손실)                |
   |  [Timeout!]                       |
   |  -- Data(seq=1000) ---------->    |  (재전송)
   |  <------ ACK(ack=1100) --------   |
```

**2. 빠른 재전송 (Fast Retransmit)**
```
Client                              Server
   |  -- Data(seq=1000) ---------->    |
   |  -- Data(seq=1100) ---------->    |  (손실)
   |  -- Data(seq=1200) ---------->    |
   |  <------ ACK(ack=1100) --------   |  (중복 ACK 1)
   |  -- Data(seq=1300) ---------->    |
   |  <------ ACK(ack=1100) --------   |  (중복 ACK 2)
   |  -- Data(seq=1400) ---------->    |
   |  <------ ACK(ack=1100) --------   |  (중복 ACK 3)
   |  [3 duplicate ACKs → 재전송]       |
   |  -- Data(seq=1100) ---------->    |  (재전송)
```

### 6. 흐름 제어 (Flow Control)

수신자의 처리 속도에 맞춰 송신 속도를 조절합니다.

#### Sliding Window

```
송신 윈도우:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
 ACK  ACK [전송가능]       [전송불가]
 완료 완료
       └──────────────┘
           윈도우 크기 = 5
```

**Window Size = 0 (Zero Window)**
- 수신자 버퍼가 가득 참
- 송신자는 전송 중단
- 주기적으로 Window Probe 패킷으로 윈도우 크기 확인

### 7. 혼잡 제어 (Congestion Control)

네트워크 상황에 맞춰 전송 속도를 조절합니다.

#### CWND (Congestion Window)

```
     ^
CWND |          Congestion Avoidance
     |         /
     |        /
     |    ___/
     |   /
     |  / Slow Start
     | /
     |/________________> Time
```

#### 알고리즘

**1. Slow Start**
- 초기 CWND = 1 MSS (Maximum Segment Size)
- ACK 받을 때마다 CWND 2배 증가
- ssthresh(threshold)에 도달하면 Congestion Avoidance로 전환

**2. Congestion Avoidance**
- CWND를 선형적으로 증가 (RTT당 1 MSS)
- 패킷 손실 감지 시 CWND 감소

**3. Fast Recovery**
- 3 duplicate ACK 발생 시
- ssthresh = CWND / 2
- CWND = ssthresh + 3
- Congestion Avoidance로 전환

```
        ^
   CWND |    *
        |   / \
        |  /   \ (packet loss)
        | /     \
        |/       \______ Fast Recovery
        |________________> Time
```

## 실무 적용

### TCP 튜닝 파라미터

```bash
# Linux TCP 파라미터
sysctl net.ipv4.tcp_max_syn_backlog=1024    # SYN 큐 크기
sysctl net.ipv4.tcp_syncookies=1             # SYN Flood 방어
sysctl net.ipv4.tcp_tw_reuse=1               # TIME_WAIT 소켓 재사용
sysctl net.core.somaxconn=65535              # Listen 백로그 크기
```

### Keep-Alive

```
# HTTP Keep-Alive
Connection: Keep-Alive

# TCP Keep-Alive (기본 2시간)
net.ipv4.tcp_keepalive_time = 7200
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_intvl = 75
```

## 참고 자료

- [RFC 793 - TCP](https://tools.ietf.org/html/rfc793)
- [TCP/IP Illustrated, Volume 1](https://www.amazon.com/TCP-Illustrated-Vol-Addison-Wesley-Professional/dp/0201633469)

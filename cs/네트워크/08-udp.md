# UDP (User Datagram Protocol)

## 개요

UDP는 **비연결형, 비신뢰성** 전송 계층 프로토콜입니다.

**핵심 특징**:
- 비연결형 (Connectionless)
- 비신뢰성 (Unreliable)
- 순서 보장 없음
- 빠른 전송
- 낮은 오버헤드

## 핵심 개념

### 1. UDP 데이터그램 구조

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            Length             |           Checksum            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| 필드 | 크기 | 설명 |
|------|------|------|
| Source Port | 16bit | 송신 포트 |
| Destination Port | 16bit | 수신 포트 |
| Length | 16bit | 헤더 + 데이터 길이 |
| Checksum | 16bit | 오류 검출 (선택적) |

**UDP 헤더**: 8바이트 (TCP는 최소 20바이트)

### 2. UDP의 특성

#### 비연결형 (Connectionless)
- 3-Way Handshake **없음**
- 바로 데이터 전송

```
Client                    Server
   |  --- Data --->          |
   |  --- Data --->          |
   |  --- Data --->          |
```

#### 비신뢰성 (Unreliable)
- 패킷 손실 감지/복구 **없음**
- 순서 보장 **없음**
- ACK **없음**

```
송신: 1, 2, 3, 4, 5
수신: 1, 3, 5  (2, 4 손실)
또는: 3, 1, 5, 2, 4  (순서 뒤섞임)
```

#### Best-Effort Delivery
- "최선을 다하지만 보장은 안 함"
- 전송 실패해도 재전송 없음

### 3. UDP vs TCP 비교

| 특성 | TCP | UDP |
|------|-----|-----|
| 연결 | 연결 지향 | 비연결형 |
| 신뢰성 | 보장 | 비보장 |
| 순서 | 보장 | 비보장 |
| 속도 | 상대적 느림 | 빠름 |
| 오버헤드 | 높음 (20+ bytes) | 낮음 (8 bytes) |
| 흐름 제어 | 있음 | 없음 |
| 혼잡 제어 | 있음 | 없음 |
| 1:1 / 1:N | 1:1 | 1:1, 1:N, N:N |

### 4. UDP 사용 사례

#### 실시간 스트리밍
```
[Video Frame 1] → 손실 → 재전송? → 이미 늦음!
                  ↓
              다음 프레임으로 진행
```

- 약간의 손실보다 **지연이 더 치명적**
- 손실된 프레임은 무시하고 다음 프레임 재생

#### 온라인 게임
```
Player Position: (x=100, y=200, t=0.001s)
Player Position: (x=101, y=201, t=0.002s)  ← 손실
Player Position: (x=102, y=202, t=0.003s)

손실된 위치 정보는 이미 과거 → 재전송 불필요
```

#### DNS 조회
```
Client → DNS Server: "google.com의 IP?"
Client ← DNS Server: "142.250.196.46"
```

- 짧은 요청/응답
- 실패 시 애플리케이션 레벨에서 재시도

#### VoIP (인터넷 전화)
- 실시간 음성 전송
- 패킷 손실 → 잠깐의 잡음 (허용 가능)
- 지연 → 대화 불가능

#### 브로드캐스트/멀티캐스트
```
서버 → [수신자1, 수신자2, 수신자3, ...]
```

- TCP는 1:1 연결만 가능
- UDP는 1:N 전송 가능

### 5. UDP 기반 프로토콜

| 프로토콜 | 용도 |
|----------|------|
| DNS | 도메인 이름 해석 |
| DHCP | IP 주소 자동 할당 |
| SNMP | 네트워크 관리 |
| RTP | 실시간 미디어 전송 |
| QUIC | HTTP/3 기반 프로토콜 |

### 6. UDP의 장점과 한계

#### 장점
1. **속도**: 연결 설정 없이 바로 전송
2. **낮은 오버헤드**: 작은 헤더 크기
3. **유연성**: 애플리케이션이 자체 신뢰성 메커니즘 구현 가능
4. **브로드캐스트/멀티캐스트** 지원

#### 한계
1. **데이터 손실** 가능
2. **순서 보장 없음**
3. **혼잡 제어 없음** → 네트워크에 부담
4. **보안 취약** (스푸핑 용이)

### 7. UDP에 신뢰성 추가하기

애플리케이션 레벨에서 구현합니다.

```
Application Layer에서:
- Sequence Number 추가
- ACK 구현
- 재전송 로직
- 순서 재조립
```

**QUIC 프로토콜**:
- UDP 위에서 동작
- TCP의 신뢰성 + UDP의 속도
- HTTP/3의 기반 프로토콜

## 실무 적용

### UDP 소켓 프로그래밍 (Python)

```python
# 서버
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 9999))

while True:
    data, addr = sock.recvfrom(1024)
    print(f"Received from {addr}: {data}")
    sock.sendto(b"ACK", addr)

# 클라이언트
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.sendto(b"Hello", ('127.0.0.1', 9999))
response, _ = sock.recvfrom(1024)
```

### UDP 사용 시 고려사항

1. **MTU (Maximum Transmission Unit)**
   - 이더넷 MTU: 1500 bytes
   - UDP 페이로드: ~1472 bytes (헤더 제외)
   - 큰 데이터는 분할 필요

2. **패킷 손실 대비**
   - 중요 데이터는 TCP 사용
   - 또는 애플리케이션 레벨 재전송

3. **방화벽 설정**
   - UDP는 상태 추적 어려움
   - 방화벽에서 별도 설정 필요

## 참고 자료

- [RFC 768 - UDP](https://tools.ietf.org/html/rfc768)
- [RFC 9000 - QUIC](https://tools.ietf.org/html/rfc9000)

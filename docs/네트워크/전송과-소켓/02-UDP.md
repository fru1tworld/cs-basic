# UDP

## UDP의 특성과 한계

UDP는 연결을 설정하지 않고 데이터그램을 보내는 비연결형(Connectionless) 프로토콜이다. 연결 관리 부담이 적은 대신 도착 여부나 순서를 보장하지 않으므로, 필요한 보장은 애플리케이션에서 마련해야 한다. 1980년 8월 발표된 RFC 768에 정의되어 있다.

- 특징: 비연결형:
  - 설명: 연결 설정 과정 없이 즉시 데이터 전송
- 특징: 신뢰성 없음:
  - 설명: 데이터 도착, 순서 보장 없음
- 특징: 경량:
  - 설명: 헤더 오버헤드 최소 (8 bytes)
- 특징: 빠른 전송:
  - 설명: 연결 설정/해제 오버헤드 없음
- 특징: 브로드캐스트/멀티캐스트:
  - 설명: 일대다 통신 지원
- 특징: 메시지 경계 유지:
  - 설명: 전송 단위 그대로 수신

### 비연결형 (Connectionless)

클라이언트는 TCP의 3-way Handshake 같은 연결 설정 없이 서버로 데이터를 보낸다. 다음 데이터그램을 보낼 때도 별도의 연결 수립을 기다리지 않는다.

### 비신뢰성 (Unreliable)

UDP 자체에는 ACK나 손실 복구 기능이 없다. 1, 2, 3, 4, 5 순서로 보내도 2와 4가 사라져 1, 3, 5만 받거나, 3, 1, 5, 2, 4처럼 다른 순서로 받을 수 있다.

### Best-Effort Delivery

- "최선을 다하지만 보장은 안 함"
- 전송 실패해도 재전송 없음

### 장점

- 속도: 연결 설정 없이 바로 전송
- 낮은 오버헤드: 작은 헤더 크기
- 유연성: 애플리케이션이 자체 신뢰성 메커니즘 구현 가능
- 브로드캐스트/멀티캐스트 지원

### 한계

- 데이터 손실 가능
- 순서 보장 없음
- 혼잡 제어 없음 → 네트워크에 부담
- 보안 취약 (스푸핑 용이)

## UDP 헤더 구조

- UDP 헤더는 8바이트이며 송신 포트, 수신 포트, 길이, 체크섬이 각각 16비트를 차지함.
- 첫 32비트에는 송신 포트와 수신 포트, 다음 32비트에는 길이와 체크섬이 순서대로 배치됨. 이후에 페이로드가 이어짐.
- 길이는 UDP 헤더와 페이로드를 합친 옥텟 수이며 최소 8임.
- 체크섬은 오류 검출에 사용됨. IPv4에서는 생략할 수 있으나 IPv6에서는 기본적으로 필수이며, 특정 UDP 터널에 한정한 예외가 존재함. [RFC 768](https://www.rfc-editor.org/rfc/rfc768), [RFC 8200](https://www.rfc-editor.org/rfc/rfc8200.html#section-8.1)
- TCP의 최소 헤더 크기는 20바이트임.

## UDP 코드 예시

아래 서버는 `recvfrom`으로 데이터와 송신자 주소를 함께 받고, 그 주소에 `sendto`로 응답한다. 응답 문자열 `ACK`는 이 예제의 애플리케이션이 직접 보내는 것이며, UDP 프로토콜의 확인 응답은 아니다.

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

## UDP 사용 사례

### DNS

- DNS에서 UDP를 사용하는 이유:
- 기본 DNS UDP 메시지는 512바이트 한계를 사용하며 EDNS로 더 큰 수신 크기를 광고할 수 있음
- 연결 설정 오버헤드 제거로 빠른 응답
- DNS 서버의 다수 클라이언트 처리 용이
- 응답 없으면 클라이언트가 재시도
- 응답이 광고된 UDP 크기를 넘어서 잘리면 TCP 재시도가 필요함. 영역 전송도 TCP를 사용함. [EDNS 명세](https://www.rfc-editor.org/rfc/rfc6891.html)

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

- Client → DNS Server: "google.com의 IP?"
- Client ← DNS Server: "142.250.196.46"

- 짧은 요청/응답
- 실패 시 애플리케이션 레벨에서 재시도

### 실시간 스트리밍

- 스트리밍에서 UDP를 사용하는 이유:
- 실시간성이 중요 (지연 < 신뢰성)
- 일부 패킷 손실은 허용 가능
- 재전송으로 인한 지연은 더 나쁜 사용자 경험 제공
- 버퍼링으로 일부 손실 보상 가능
- 사용 프로토콜:
- RTP (Real-time Transport Protocol)
- RTSP (Real-Time Streaming Protocol)
- WebRTC (브라우저 실시간 통신)

- 비디오 스트리밍 패킷 손실 시나리오:
- TCP 사용 시:
- Frame 1 → [수신] → 재생
- Frame 2 → [손실] → 재전송 대기... → 버퍼링... → 재생
- Frame 3 → [대기] → 재생
- 지연 발생!
- UDP 사용 시:
- Frame 1 → [수신] → 재생
- Frame 2 → [손실] → 건너뛰기 (약간의 화질 저하)
- Frame 3 → [수신] → 즉시 재생
- 실시간 유지!

- [Video Frame 1] → 손실 → 재전송? → 이미 늦음!
- 다음 프레임으로 진행

- 약간의 손실보다 지연이 더 치명적
- 손실된 프레임은 무시하고 다음 프레임 재생

### 온라인 게임

- 게임에서 UDP를 사용하는 이유:
- 실시간 위치/상태 업데이트 (최신 정보가 중요)
- 오래된 패킷은 의미 없음 (재전송 불필요)
- 낮은 지연시간 필수
- 작은 패킷 빈번한 전송
- 게임 네트워크 설계:
- 위치 동기화: UDP (실시간)
- 채팅: TCP (신뢰성)
- 결제/인증: TCP (신뢰성)

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

- Player Position: (x=100, y=200, t=0.001s)
- Player Position: (x=101, y=201, t=0.002s), ← 손실
- Player Position: (x=102, y=202, t=0.003s)
- 손실된 위치 정보는 이미 과거 → 재전송 불필요

### VoIP

- VoIP에서 UDP를 사용하는 이유:
- 실시간 음성 전달 필수
- 약간의 패킷 손실은 허용 (음질 저하지만 통화 가능)
- 20-30ms 이하 지연 필요
- TCP 재전송으로 인한 지연은 통화 품질 심각하게 저하
- 사용 프로토콜:
- RTP over UDP (음성 데이터)
- SIP over UDP/TCP (시그널링)

- 실시간 음성 전송
- 패킷 손실 → 잠깐의 잡음 (허용 가능)
- 지연 → 대화 불가능

### QUIC 프로토콜 (HTTP/3)

- QUIC: UDP 기반의 신뢰성 있는 프로토콜
- QUIC 특징
- UDP 위에서 동작하지만 TCP와 유사한 신뢰성 제공
- 연결 설정 시간 단축 (0-RTT, 1-RTT)
- Head-of-Line Blocking 문제 해결
- 내장 TLS 1.3 암호화
- 연결 마이그레이션 지원
- HTTP/2 over TCP vs HTTP/3 over QUIC:
- TCP + TLS 1.3:
- Client, SYN→ Server
- Client ←SYN/ACK, Server
- Client, ACK→ Server, } 1 RTT (TCP)
- Client, ClientHello→ Server
- Client ←ServerHello, Server } 1-2 RTT (TLS)
- 총 2-3 RTT
- QUIC (1-RTT):
- Client, Initial→ Server
- Client ←Handshake, Server, } 1 RTT (통합)
- 총 1 RTT

### 브로드캐스트/멀티캐스트

- 서버 → [수신자1, 수신자2, 수신자3, ...]

- TCP는 1:1 연결만 가능
- UDP는 1:N 전송 가능

## UDP 기반 프로토콜

- 프로토콜: DNS:
  - 용도: 도메인 이름 해석
- 프로토콜: DHCP:
  - 용도: IP 주소 자동 할당
- 프로토콜: SNMP:
  - 용도: 네트워크 관리
- 프로토콜: RTP:
  - 용도: 실시간 미디어 전송
- 프로토콜: QUIC:
  - 용도: HTTP/3 기반 프로토콜

## UDP에 신뢰성 추가하기

UDP 위에서 신뢰성이 필요하다면 시퀀스 번호로 순서를 구분하고, ACK와 재전송으로 손실에 대응하며, 도착한 데이터를 재조립해야 한다. HTTP/3의 기반인 QUIC도 UDP 위에서 동작하면서 독립된 신뢰성 스트림, 손실 복구, 흐름 제어와 혼잡 제어를 제공한다.

## 참고 자료

- [RFC 793 - Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793)
- [RFC 768 - User Datagram Protocol](https://datatracker.ietf.org/doc/html/rfc768)
- [RFC 5681 - TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681)
- [RFC 9000 - QUIC: A UDP-Based Multiplexed and Secure Transport](https://datatracker.ietf.org/doc/html/rfc9000)
- [Wikipedia - Transmission Control Protocol](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
- TCP/IP Illustrated, Volume 1: The Protocols (W. Richard Stevens)

- [RFC 768 - UDP](https://tools.ietf.org/html/rfc768)
- [RFC 9000 - QUIC](https://tools.ietf.org/html/rfc9000)

## 관련 학습

- [TCP](01-TCP.md)
- [전송 프로토콜 선택](03-전송-프로토콜-선택.md)

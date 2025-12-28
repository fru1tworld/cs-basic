# OSI 7계층 모델

## 목차
1. [개요](#개요)
2. [OSI 7계층 상세](#osi-7계층-상세)
3. [TCP/IP 모델과 비교](#tcpip-모델과-비교)
4. [각 계층의 프로토콜](#각-계층의-프로토콜)
5. [캡슐화와 역캡슐화](#캡슐화와-역캡슐화)
6. [PDU (Protocol Data Unit)](#pdu-protocol-data-unit)
7. [실무 적용](#실무-적용)
---

## 개요

OSI(Open Systems Interconnection) 7계층 모델은 국제표준화기구(ISO)에서 1984년에 발표한 네트워크 통신을 위한 개념적 프레임워크입니다. 복잡한 데이터 전송 과정을 7개의 독립적인 계층으로 분리하여, 각 계층이 특정 기능을 담당하도록 설계되었습니다.

### OSI 모델의 목적

1. **표준화**: 서로 다른 벤더의 장비들이 상호 운용될 수 있도록 표준 제공
2. **모듈화**: 각 계층을 독립적으로 개발 및 업그레이드 가능
3. **문제 해결**: 네트워크 문제 발생 시 특정 계층으로 문제를 격리하여 디버깅 용이
4. **교육**: 네트워크 개념을 체계적으로 학습할 수 있는 프레임워크 제공

---

## OSI 7계층 상세

```
┌─────────────────────────────────────────────────────────────┐
│                    OSI 7계층 모델                             │
├─────────────────────────────────────────────────────────────┤
│  7. Application  │ 사용자 인터페이스, 네트워크 서비스          │
│  6. Presentation │ 데이터 형식 변환, 암호화, 압축              │
│  5. Session      │ 세션 관리, 동기화                          │
│  4. Transport    │ 종단간 신뢰성 있는 데이터 전송              │
│  3. Network      │ 논리적 주소 지정, 라우팅                    │
│  2. Data Link    │ 물리적 주소 지정, 오류 검출                 │
│  1. Physical     │ 비트 전송, 물리적 연결                      │
└─────────────────────────────────────────────────────────────┘
```

### 1계층: 물리 계층 (Physical Layer)

**역할**: 비트 스트림을 실제 물리적 신호(전기, 광, 무선)로 변환하여 전송

**핵심 개념**:
- 비트(0과 1)를 전기 신호, 광 신호, 전파로 변환
- 물리적 토폴로지 정의 (버스, 스타, 링, 메시)
- 전송 모드 결정 (단방향, 반이중, 전이중)
- 비트 동기화

**장비**: 허브, 리피터, 케이블, 모뎀, 네트워크 인터페이스 카드(NIC)

**표준 예시**:
- IEEE 802.3 (이더넷 물리 계층)
- RS-232 (시리얼 통신)
- DSL, ISDN

```
전기 신호 예시:
+5V  ──┐     ┌──┐     ┌──
       │     │  │     │
 0V ───┴─────┴──┴─────┴──
       1  0  1  1  0  1
```

### 2계층: 데이터 링크 계층 (Data Link Layer)

**역할**: 인접 노드 간의 신뢰성 있는 데이터 전송, 물리적 주소(MAC) 사용

**핵심 개념**:
- **프레임화(Framing)**: 비트 스트림을 프레임 단위로 구분
- **물리적 주소 지정**: MAC 주소 사용 (48비트, 예: `00:1A:2B:3C:4D:5E`)
- **오류 검출**: CRC(Cyclic Redundancy Check) 사용
- **흐름 제어**: 송수신 속도 조절
- **매체 접근 제어(MAC)**: CSMA/CD, CSMA/CA

**하위 계층**:
- **LLC (Logical Link Control)**: 상위 계층과 인터페이스
- **MAC (Media Access Control)**: 물리적 매체 접근 관리

**장비**: 스위치, 브릿지

```
이더넷 프레임 구조:
┌──────────┬──────────┬──────────┬──────┬─────────┬─────┐
│Preamble  │Dest MAC  │Source MAC│Type  │Payload  │FCS  │
│(8 bytes) │(6 bytes) │(6 bytes) │(2)   │(46-1500)│(4)  │
└──────────┴──────────┴──────────┴──────┴─────────┴─────┘
```

### 3계층: 네트워크 계층 (Network Layer)

**역할**: 서로 다른 네트워크 간의 데이터 전송, 논리적 주소(IP) 사용, 라우팅

**핵심 개념**:
- **논리적 주소 지정**: IP 주소 사용
- **라우팅**: 최적의 경로 결정
- **패킷 분할 및 재조립**: MTU에 따른 프래그멘테이션
- **혼잡 제어**: 네트워크 혼잡 시 패킷 전송 조절

**장비**: 라우터, L3 스위치

**프로토콜**: IP, ICMP, IGMP, ARP, RARP, OSPF, BGP, RIP

```
IPv4 헤더 구조:
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│Version│  IHL  │    DSCP   │ECN│         Total Length          │
├───────┴───────┴───────────┴───┼───────────────────────────────┤
│         Identification        │Flags│    Fragment Offset      │
├───────────────────────────────┼─────┴─────────────────────────┤
│      TTL      │   Protocol    │       Header Checksum         │
├───────────────┴───────────────┴───────────────────────────────┤
│                       Source IP Address                       │
├───────────────────────────────────────────────────────────────┤
│                    Destination IP Address                     │
├───────────────────────────────────────────────────────────────┤
│                    Options (if IHL > 5)                       │
└───────────────────────────────────────────────────────────────┘
```

### 4계층: 전송 계층 (Transport Layer)

**역할**: 종단 간(End-to-End) 신뢰성 있는 데이터 전송, 포트 번호를 통한 프로세스 식별

**핵심 개념**:
- **세그멘테이션**: 데이터를 세그먼트로 분할
- **연결 관리**: TCP의 연결 설정/해제
- **신뢰성 보장**: 순서 제어, 오류 복구, 재전송
- **흐름 제어**: Sliding Window
- **혼잡 제어**: Slow Start, AIMD

**프로토콜**: TCP, UDP, SCTP

```python
# Python으로 보는 TCP/UDP 포트 개념
import socket

# TCP 소켓 - 연결 지향
tcp_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
tcp_socket.bind(('0.0.0.0', 8080))  # 포트 8080에 바인딩

# UDP 소켓 - 비연결 지향
udp_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
udp_socket.bind(('0.0.0.0', 8081))  # 포트 8081에 바인딩
```

### 5계층: 세션 계층 (Session Layer)

**역할**: 통신 세션의 설정, 관리, 종료. 대화 제어 및 동기화

**핵심 개념**:
- **세션 설정**: 통신 세션 초기화
- **세션 유지**: 연결 상태 관리
- **세션 종료**: 정상적인 세션 종료 처리
- **동기점(Checkpoint)**: 긴 데이터 전송 시 중간 동기점 설정
- **대화 제어**: 반이중/전이중 통신 관리

**프로토콜/API**: NetBIOS, RPC(Remote Procedure Call), SQL, NFS

```
세션 관리 예시 - HTTP 세션:
┌────────┐                          ┌────────┐
│ Client │                          │ Server │
└───┬────┘                          └───┬────┘
    │  ──── Session Establishment ────> │
    │  <──── Session Acknowledgment ─── │
    │                                   │
    │  ──────── Data Exchange ───────>  │
    │  <──────── Response ────────────  │
    │                                   │
    │  ──── Session Termination ─────>  │
    │  <──── Termination ACK ─────────  │
    ▼                                   ▼
```

### 6계층: 표현 계층 (Presentation Layer)

**역할**: 데이터 형식 변환, 암호화/복호화, 압축/해제

**핵심 개념**:
- **데이터 변환**: 서로 다른 시스템 간 데이터 형식 변환
- **문자 인코딩**: ASCII, EBCDIC, UTF-8 변환
- **암호화/복호화**: SSL/TLS에서의 암호화 처리
- **압축/해제**: 데이터 압축으로 전송 효율 향상

**예시**:
- JPEG, GIF, PNG (이미지 포맷)
- MPEG, AVI (비디오 포맷)
- SSL/TLS (암호화)
- ASCII, EBCDIC, UTF-8 (문자 인코딩)

```
데이터 변환 예시:
┌─────────────────────────────────────────────────┐
│ Application Data: "Hello, World!"               │
│                     ↓                           │
│ Presentation Layer Processing:                  │
│   1. 문자 인코딩: UTF-8 → 바이트 배열            │
│   2. 압축: GZIP 압축                            │
│   3. 암호화: AES-256 암호화                     │
│                     ↓                           │
│ Encrypted Data: 0x8F2A3B4C5D6E7F...             │
└─────────────────────────────────────────────────┘
```

### 7계층: 응용 계층 (Application Layer)

**역할**: 사용자와 네트워크 간의 인터페이스 제공, 네트워크 서비스 직접 접근

**핵심 개념**:
- 사용자 애플리케이션에 네트워크 서비스 제공
- 네트워크 투명성 제공
- 리소스 공유 가능

**프로토콜**:
- **HTTP/HTTPS**: 웹 브라우징 (포트 80/443)
- **FTP**: 파일 전송 (포트 20/21)
- **SMTP**: 이메일 발송 (포트 25/587)
- **POP3/IMAP**: 이메일 수신 (포트 110/143)
- **DNS**: 도메인 이름 해석 (포트 53)
- **SSH**: 보안 원격 접속 (포트 22)
- **Telnet**: 원격 접속 (포트 23)

```python
# HTTP 요청 예시 (Application Layer)
import requests

# GET 요청 - 웹 서버에서 데이터 조회
response = requests.get('https://api.example.com/users')

# POST 요청 - 웹 서버에 데이터 전송
response = requests.post(
    'https://api.example.com/users',
    json={'name': 'John', 'email': 'john@example.com'}
)
```

---

## TCP/IP 모델과 비교

### 계층 매핑

```
    OSI 7계층 모델                    TCP/IP 4계층 모델
┌─────────────────────┐          ┌─────────────────────┐
│  7. Application     │          │                     │
├─────────────────────┤          │  4. Application     │
│  6. Presentation    │   ───>   │     (응용 계층)      │
├─────────────────────┤          │                     │
│  5. Session         │          │                     │
├─────────────────────┤          ├─────────────────────┤
│  4. Transport       │   ───>   │  3. Transport       │
│                     │          │     (전송 계층)      │
├─────────────────────┤          ├─────────────────────┤
│  3. Network         │   ───>   │  2. Internet        │
│                     │          │     (인터넷 계층)    │
├─────────────────────┤          ├─────────────────────┤
│  2. Data Link       │          │  1. Network Access  │
├─────────────────────┤   ───>   │     (네트워크       │
│  1. Physical        │          │      접근 계층)     │
└─────────────────────┘          └─────────────────────┘
```

### 상세 비교

| 구분 | OSI 모델 | TCP/IP 모델 |
|------|----------|-------------|
| **계층 수** | 7계층 | 4계층 |
| **개발** | ISO (국제표준화기구) | 미국 국방부 (DARPA) |
| **목적** | 이론적/개념적 표준 | 실용적/구현 중심 |
| **프로토콜 의존성** | 프로토콜 독립적 | 특정 프로토콜에 의존 |
| **개발 시기** | 1984년 | 1970년대 |
| **사용** | 교육, 문제 해결 참조 | 실제 인터넷 통신 |
| **유연성** | 높음 (각 계층 독립적) | 상대적으로 낮음 |
| **실무 적용** | 참조 모델 | 실제 구현 표준 |

### TCP/IP 4계층 상세

**1. 네트워크 접근 계층 (Network Access Layer)**
- OSI 1-2계층에 해당
- 물리적 네트워크 매체에 대한 접근 담당
- 이더넷, Wi-Fi, PPP 등

**2. 인터넷 계층 (Internet Layer)**
- OSI 3계층에 해당
- IP 주소 지정 및 라우팅
- IP, ICMP, ARP

**3. 전송 계층 (Transport Layer)**
- OSI 4계층에 해당
- 종단 간 데이터 전송
- TCP, UDP

**4. 응용 계층 (Application Layer)**
- OSI 5-7계층에 해당
- 사용자 애플리케이션 지원
- HTTP, FTP, SMTP, DNS

---

## 각 계층의 프로토콜

### 프로토콜 종합 표

| 계층 | 대표 프로토콜 | 설명 |
|------|--------------|------|
| **7. Application** | HTTP, HTTPS, FTP, SMTP, DNS, SSH, Telnet, SNMP | 사용자 애플리케이션 서비스 |
| **6. Presentation** | SSL/TLS, JPEG, MPEG, ASCII, EBCDIC | 데이터 변환, 암호화 |
| **5. Session** | NetBIOS, RPC, PPTP, SIP | 세션 관리 |
| **4. Transport** | TCP, UDP, SCTP | 종단 간 전송 |
| **3. Network** | IP, ICMP, IGMP, ARP, RARP, OSPF, BGP, RIP | 라우팅, 주소 지정 |
| **2. Data Link** | Ethernet, PPP, HDLC, Frame Relay, ATM | 프레이밍, MAC |
| **1. Physical** | RS-232, V.35, 100BASE-TX, IEEE 802.11 | 물리적 전송 |

### 주요 프로토콜 상세

#### HTTP/HTTPS (Application Layer)
```
HTTP 요청 구조:
GET /api/users HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: application/json
Connection: keep-alive

HTTP 응답 구조:
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1234

{"users": [...]}
```

#### TCP (Transport Layer)
```
TCP 헤더 구조 (20 bytes minimum):
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
│       │       │G│K│H│T│N│N│                                   │
├───────┴───────┴─┴─┴─┴─┴─┴─┼───────────────────────────────────┤
│           Checksum        │         Urgent Pointer            │
├───────────────────────────┴───────────────────────────────────┤
│                    Options (variable)                         │
└───────────────────────────────────────────────────────────────┘
```

#### IP (Network Layer)
- **IPv4**: 32비트 주소 (예: 192.168.1.1)
- **IPv6**: 128비트 주소 (예: 2001:0db8:85a3::8a2e:0370:7334)

```python
# Python에서 IP 주소 다루기
import ipaddress

# IPv4 네트워크 생성
network = ipaddress.ip_network('192.168.1.0/24')
print(f"네트워크: {network}")
print(f"네트마스크: {network.netmask}")
print(f"호스트 수: {network.num_addresses - 2}")  # 브로드캐스트, 네트워크 주소 제외

# 특정 IP가 네트워크에 속하는지 확인
ip = ipaddress.ip_address('192.168.1.100')
print(f"{ip} in {network}: {ip in network}")
```

---

## 캡슐화와 역캡슐화

### 캡슐화 (Encapsulation)

데이터가 송신 측에서 하위 계층으로 내려갈 때, 각 계층에서 헤더(그리고 때로는 트레일러)를 추가하는 과정입니다.

```
송신 측 캡슐화 과정:

Application Layer:    [      Data      ]
                              ↓
Presentation Layer:   [  Data (암호화/변환)  ]
                              ↓
Session Layer:        [     Data + 세션 정보    ]
                              ↓
Transport Layer:     [TCP Header][    Data    ]  → 세그먼트
                              ↓
Network Layer:       [IP Header][TCP][  Data  ]  → 패킷
                              ↓
Data Link Layer:   [Frame H][IP][TCP][Data][FCS] → 프레임
                              ↓
Physical Layer:    01101001 10110010 ...         → 비트 스트림
```

### 역캡슐화 (Decapsulation)

수신 측에서 데이터가 상위 계층으로 올라갈 때, 각 계층에서 헤더를 제거하는 과정입니다.

```
수신 측 역캡슐화 과정:

Physical Layer:    01101001 10110010 ...
                              ↓
Data Link Layer:   [Frame H][IP][TCP][Data][FCS] → FCS 검증, Frame H 제거
                              ↓
Network Layer:       [IP Header][TCP][  Data  ]  → IP 헤더 검사 후 제거
                              ↓
Transport Layer:     [TCP Header][    Data    ]  → TCP 헤더 검사 후 제거
                              ↓
Session Layer:        [     Data + 세션 정보    ]
                              ↓
Presentation Layer:   [  Data (복호화/변환)  ]
                              ↓
Application Layer:    [      Data      ]        → 애플리케이션에 전달
```

### 캡슐화 예시 코드

```python
# 캡슐화 개념의 간단한 시뮬레이션
class NetworkStack:
    def __init__(self):
        self.data = None

    def application_layer(self, data):
        """7계층: 애플리케이션 데이터 생성"""
        print(f"[Application] 원본 데이터: {data}")
        return data

    def transport_layer(self, data, src_port=8080, dst_port=80):
        """4계층: TCP 헤더 추가"""
        tcp_header = f"TCP|src:{src_port}|dst:{dst_port}|seq:1000|"
        segment = tcp_header + data
        print(f"[Transport] 세그먼트: {segment}")
        return segment

    def network_layer(self, segment, src_ip="192.168.1.1", dst_ip="10.0.0.1"):
        """3계층: IP 헤더 추가"""
        ip_header = f"IP|src:{src_ip}|dst:{dst_ip}|ttl:64|"
        packet = ip_header + segment
        print(f"[Network] 패킷: {packet}")
        return packet

    def datalink_layer(self, packet, src_mac="AA:BB:CC:DD:EE:FF",
                       dst_mac="11:22:33:44:55:66"):
        """2계층: 이더넷 헤더와 트레일러 추가"""
        eth_header = f"ETH|src:{src_mac}|dst:{dst_mac}|"
        frame = eth_header + packet + "|CRC"
        print(f"[Data Link] 프레임: {frame}")
        return frame

    def send(self, message):
        """캡슐화 과정 시뮬레이션"""
        print("=== 캡슐화 과정 ===")
        data = self.application_layer(message)
        segment = self.transport_layer(data)
        packet = self.network_layer(segment)
        frame = self.datalink_layer(packet)
        return frame

# 실행
stack = NetworkStack()
stack.send("Hello, World!")
```

---

## PDU (Protocol Data Unit)

PDU는 각 계층에서 처리되는 데이터 단위를 의미합니다.

### 계층별 PDU

| 계층 | PDU 명칭 | 설명 |
|------|----------|------|
| 7. Application | Data/Message | 사용자 데이터 |
| 6. Presentation | Data | 변환된 데이터 |
| 5. Session | Data | 세션 정보 포함 데이터 |
| 4. Transport | **Segment** (TCP) / **Datagram** (UDP) | 포트 정보 포함 |
| 3. Network | **Packet** | IP 주소 포함 |
| 2. Data Link | **Frame** | MAC 주소, CRC 포함 |
| 1. Physical | **Bit** | 전기/광 신호 |

### PDU 구조 시각화

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Frame (L2 PDU)                            │
├─────────────┬───────────────────────────────────────────────────────┤
│ Ethernet    │                    Packet (L3 PDU)                    │
│ Header      ├─────────────┬─────────────────────────────────────────┤
│ (14 bytes)  │ IP Header   │          Segment (L4 PDU)               │
│             │ (20+ bytes) ├─────────────┬───────────────────────────┤
│             │             │ TCP Header  │       Application Data    │
│             │             │ (20+ bytes) │                           │
├─────────────┼─────────────┼─────────────┼───────────────────────────┤
│ Dest MAC    │ Src IP      │ Src Port    │                           │
│ Src MAC     │ Dest IP     │ Dest Port   │       Payload             │
│ Type        │ TTL, etc.   │ Seq, ACK    │                           │
├─────────────┴─────────────┴─────────────┴───────────────────────────┤
│                              FCS (4 bytes)                          │
└─────────────────────────────────────────────────────────────────────┘
```

### MTU와 MSS

```
MTU (Maximum Transmission Unit):
- 네트워크 계층에서 전송할 수 있는 최대 패킷 크기
- 이더넷 기본 MTU: 1500 bytes

MSS (Maximum Segment Size):
- TCP 세그먼트의 최대 데이터 크기
- MSS = MTU - IP Header (20) - TCP Header (20) = 1460 bytes

┌────────────────────── MTU (1500 bytes) ──────────────────────┐
│                                                              │
│  ┌─────────┐ ┌─────────┐ ┌───────────────────────────────┐  │
│  │IP Header│ │TCP Header│ │     Payload (MSS)             │  │
│  │(20 bytes)│ │(20 bytes) │ │     (1460 bytes max)         │  │
│  └─────────┘ └─────────┘ └───────────────────────────────┘  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 실무 적용

### 네트워크 문제 해결 접근법

```
Layer-by-Layer Troubleshooting:

1계층 (Physical):
   - 케이블 연결 확인
   - LED 상태 확인
   - 명령어: `ethtool eth0`, `mii-tool`

2계층 (Data Link):
   - MAC 주소 확인
   - ARP 테이블 확인
   - 명령어: `arp -a`, `ip link show`

3계층 (Network):
   - IP 주소, 라우팅 확인
   - ping 테스트
   - 명령어: `ip addr`, `ip route`, `ping`, `traceroute`

4계층 (Transport):
   - 포트 연결 확인
   - TCP/UDP 상태 확인
   - 명령어: `netstat -tuln`, `ss -tuln`, `telnet host port`

5-7계층 (Application):
   - 서비스 상태 확인
   - 로그 분석
   - 명령어: `curl`, `wget`, `nslookup`, `dig`
```

### 실무 진단 스크립트

```bash
#!/bin/bash
# network_diagnose.sh - 계층별 네트워크 진단

TARGET_HOST=$1
TARGET_PORT=${2:-80}

echo "=== 네트워크 진단 시작 ==="
echo "대상: $TARGET_HOST:$TARGET_PORT"
echo ""

# L1-L2: 로컬 네트워크 인터페이스 확인
echo "[L1-L2] 네트워크 인터페이스 상태:"
ip link show | grep -E "^[0-9]|state"
echo ""

# L3: IP 라우팅 확인
echo "[L3] 라우팅 테이블:"
ip route
echo ""

echo "[L3] Ping 테스트:"
ping -c 3 $TARGET_HOST
echo ""

echo "[L3] Traceroute:"
traceroute -m 10 $TARGET_HOST 2>/dev/null || tracepath $TARGET_HOST
echo ""

# L4: 포트 연결 테스트
echo "[L4] 포트 연결 테스트:"
timeout 5 bash -c "echo > /dev/tcp/$TARGET_HOST/$TARGET_PORT" 2>/dev/null && \
    echo "포트 $TARGET_PORT 연결 성공" || \
    echo "포트 $TARGET_PORT 연결 실패"
echo ""

# L7: HTTP 테스트 (웹 서버인 경우)
if [ $TARGET_PORT -eq 80 ] || [ $TARGET_PORT -eq 443 ]; then
    echo "[L7] HTTP 응답 테스트:"
    curl -I -s -o /dev/null -w "HTTP 상태: %{http_code}\n응답시간: %{time_total}s\n" \
        http://$TARGET_HOST:$TARGET_PORT
fi

echo "=== 진단 완료 ==="
```

### Wireshark 분석 팁

```
Wireshark 필터 예시:

# IP 주소 필터
ip.addr == 192.168.1.100

# TCP 포트 필터
tcp.port == 443

# HTTP 요청만
http.request

# TCP 핸드셰이크
tcp.flags.syn == 1

# TCP 재전송
tcp.analysis.retransmission

# DNS 쿼리
dns.flags.response == 0

# 특정 호스트의 모든 트래픽
host 192.168.1.100

# TCP 오류 분석
tcp.analysis.flags
```

---

## 참고 자료

- [ISO/IEC 7498-1:1994 - OSI Basic Reference Model](https://www.iso.org/standard/20269.html)
- [RFC 1122 - Requirements for Internet Hosts](https://datatracker.ietf.org/doc/html/rfc1122)
- [Fortinet - TCP/IP Model vs OSI Model](https://www.fortinet.com/resources/cyberglossary/tcp-ip-model-vs-osi-model)
- [GeeksforGeeks - OSI and TCP/IP Model](https://www.geeksforgeeks.org/computer-networks/difference-between-osi-model-and-tcp-ip-model/)
- [Imperva - OSI Model 7 Layers Explained](https://www.imperva.com/learn/application-security/osi-model/)
- TCP/IP Illustrated, Volume 1: The Protocols (W. Richard Stevens)

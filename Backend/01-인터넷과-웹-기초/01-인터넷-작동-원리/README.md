# 인터넷 작동 원리

> 인터넷의 핵심 동작 원리를 다룹니다.

## 목차
1. [패킷 스위칭과 회선 스위칭](#1-패킷-스위칭과-회선-스위칭)
2. [IP 주소 체계](#2-ip-주소-체계-ipv4-ipv6)
3. [라우팅 프로토콜](#3-라우팅-프로토콜-bgp-ospf)
4. [인터넷 구조](#4-인터넷-구조-isp-ixp-tier)
---

## 1. 패킷 스위칭과 회선 스위칭

### 1.1 회선 스위칭 (Circuit Switching)

회선 스위칭은 **통신 시작 전에 송신자와 수신자 사이에 전용 경로(회선)를 설정**하는 방식입니다.

```
[발신자] ----전용 회선---- [교환기] ----전용 회선---- [수신자]
           (독점 사용)              (독점 사용)
```

**특징:**
- 통신 전 연결 설정 단계(Call Setup) 필요
- 연결 동안 대역폭이 보장됨
- 자원의 비효율적 사용 (통화 중 침묵 시간에도 회선 점유)
- 대표적 예: 전통적인 전화 네트워크 (PSTN)

**회선 스위칭의 대역폭 계산:**
```
총 대역폭: 1 Mbps
회선당 대역폭: 100 Kbps
동시 연결 가능 수: 1,000,000 / 100,000 = 10개
```

### 1.2 패킷 스위칭 (Packet Switching)

패킷 스위칭은 **데이터를 작은 패킷 단위로 분할하여 전송**하는 방식입니다. 현대 인터넷의 기반 기술입니다.

```
원본 데이터: [ABCDEFGHIJ]
     ↓ 분할
패킷 1: [ABC | Header1]  →  라우터1  →  라우터3  →  [목적지]
패킷 2: [DEF | Header2]  →  라우터2  →  라우터4  →  [목적지]
패킷 3: [GHIJ | Header3] →  라우터1  →  라우터4  →  [목적지]
     ↓ 재조립
수신 데이터: [ABCDEFGHIJ]
```

**패킷 구조 (RFC 791 - IPv4):**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             Data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 1.3 패킷 스위칭 방식 비교

| 구분 | 데이터그램 방식 | 가상 회선 방식 |
|------|----------------|---------------|
| 연결 설정 | 불필요 | 필요 |
| 패킷 경로 | 독립적 (다를 수 있음) | 동일 경로 |
| 순서 보장 | X (재정렬 필요) | O |
| 예시 | IP, UDP | ATM, MPLS, TCP |

### 1.4 Store-and-Forward vs Cut-Through

```python
# Store-and-Forward 지연 시간 계산
def store_and_forward_delay(packet_size, link_bandwidth, num_hops):
    """
    패킷 전체를 수신한 후 다음 홉으로 전송

    Args:
        packet_size: 패킷 크기 (bits)
        link_bandwidth: 링크 대역폭 (bps)
        num_hops: 홉 수

    Returns:
        total_delay: 총 전송 지연 (seconds)
    """
    transmission_delay = packet_size / link_bandwidth
    total_delay = num_hops * transmission_delay
    return total_delay

# 예시: 1500 bytes 패킷, 100 Mbps 링크, 3홉
packet_size = 1500 * 8  # 12,000 bits
link_bandwidth = 100 * 10**6  # 100 Mbps
num_hops = 3

delay = store_and_forward_delay(packet_size, link_bandwidth, num_hops)
print(f"Store-and-Forward 지연: {delay * 1000:.3f} ms")  # 0.360 ms
```

### 1.5 회선 스위칭 vs 패킷 스위칭 비교표

| 특성 | 회선 스위칭 | 패킷 스위칭 |
|------|------------|------------|
| 연결 설정 | 필요 (Call Setup) | 불필요 |
| 대역폭 | 고정 할당 | 동적 공유 |
| 지연 시간 | 일정 | 가변적 (큐잉 지연) |
| 자원 효율성 | 낮음 | 높음 |
| 신뢰성 | 경로 장애 시 통신 불가 | 대체 경로 사용 가능 |
| 적합한 서비스 | 음성 통화 | 데이터 통신 |

---

## 2. IP 주소 체계 (IPv4, IPv6)

### 2.1 IPv4 (RFC 791)

**주소 구조:** 32비트, 약 43억 개의 주소 (2^32 = 4,294,967,296)

```
IP 주소: 192.168.1.100
이진수:  11000000.10101000.00000001.01100100

클래스 기반 주소 체계 (Legacy):
┌─────────┬───────────────────────────────────────┬─────────────────────┐
│  클래스  │          네트워크 범위                 │        용도          │
├─────────┼───────────────────────────────────────┼─────────────────────┤
│    A    │ 0.0.0.0 ~ 127.255.255.255             │ 대규모 네트워크       │
│    B    │ 128.0.0.0 ~ 191.255.255.255           │ 중규모 네트워크       │
│    C    │ 192.0.0.0 ~ 223.255.255.255           │ 소규모 네트워크       │
│    D    │ 224.0.0.0 ~ 239.255.255.255           │ 멀티캐스트            │
│    E    │ 240.0.0.0 ~ 255.255.255.255           │ 예약됨               │
└─────────┴───────────────────────────────────────┴─────────────────────┘
```

**CIDR (Classless Inter-Domain Routing) - RFC 4632:**
```python
# CIDR 표기법과 서브넷 계산
def calculate_subnet_info(cidr_notation):
    """
    CIDR 표기법으로 서브넷 정보 계산

    Args:
        cidr_notation: "192.168.1.0/24" 형태의 CIDR 표기

    Returns:
        dict: 네트워크 정보
    """
    import ipaddress

    network = ipaddress.ip_network(cidr_notation, strict=False)

    return {
        "network_address": str(network.network_address),
        "broadcast_address": str(network.broadcast_address),
        "netmask": str(network.netmask),
        "num_hosts": network.num_addresses - 2,  # 네트워크, 브로드캐스트 제외
        "host_range": f"{network.network_address + 1} ~ {network.broadcast_address - 1}",
        "prefix_length": network.prefixlen
    }

# 사용 예시
info = calculate_subnet_info("192.168.1.0/24")
print(f"네트워크 주소: {info['network_address']}")
print(f"브로드캐스트: {info['broadcast_address']}")
print(f"서브넷 마스크: {info['netmask']}")
print(f"사용 가능 호스트 수: {info['num_hosts']}")
```

**사설 IP 주소 (RFC 1918):**
```
10.0.0.0/8       (10.0.0.0 ~ 10.255.255.255)      - 16,777,216 주소
172.16.0.0/12    (172.16.0.0 ~ 172.31.255.255)    - 1,048,576 주소
192.168.0.0/16   (192.168.0.0 ~ 192.168.255.255)  - 65,536 주소
```

### 2.2 IPv6 (RFC 8200)

**주소 구조:** 128비트, 약 3.4 x 10^38 개의 주소

```
IPv6 주소 형식:
2001:0db8:85a3:0000:0000:8a2e:0370:7334

축약 규칙:
1. 선행 0 생략: 0db8 → db8
2. 연속된 0 그룹은 ::로 한 번만 생략 가능

축약 후: 2001:db8:85a3::8a2e:370:7334
```

**IPv6 주소 유형:**
```
┌──────────────────┬─────────────────────────────────────────────────┐
│     주소 유형     │                    설명                          │
├──────────────────┼─────────────────────────────────────────────────┤
│ 유니캐스트        │ 단일 인터페이스 식별                              │
│  - 글로벌        │ 2000::/3 (인터넷에서 라우팅 가능)                  │
│  - 링크 로컬     │ fe80::/10 (로컬 링크 내에서만 유효)                │
│  - 유니크 로컬   │ fc00::/7 (사설 네트워크, IPv4의 RFC 1918 대응)     │
│ 멀티캐스트       │ ff00::/8 (그룹 통신)                              │
│ 애니캐스트       │ 유니캐스트와 동일 형식 (가장 가까운 노드로 전달)     │
└──────────────────┴─────────────────────────────────────────────────┘
```

**IPv6 헤더 구조 (40 bytes 고정):**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                         Source Address                        +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                                                               |
+                      Destination Address                      +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 2.3 IPv4 vs IPv6 비교

| 특성 | IPv4 | IPv6 |
|------|------|------|
| 주소 크기 | 32비트 | 128비트 |
| 주소 표기 | 점-십진수 (192.168.1.1) | 콜론-16진수 (2001:db8::1) |
| 헤더 크기 | 20-60 bytes (가변) | 40 bytes (고정) |
| 체크섬 | 있음 | 없음 (상위 계층에 위임) |
| NAT 필요성 | 높음 | 낮음 |
| IPSec | 선택적 | 필수 지원 |
| 주소 설정 | DHCP 또는 수동 | SLAAC 또는 DHCPv6 |

### 2.4 IPv6 전환 메커니즘

```python
# NAT64 변환 예시 개념
class NAT64Translator:
    """
    NAT64: IPv6 전용 클라이언트가 IPv4 서버에 접근할 수 있도록 변환
    RFC 6146에 정의됨
    """

    def __init__(self, nat64_prefix="64:ff9b::/96"):
        self.prefix = nat64_prefix

    def ipv4_to_ipv6(self, ipv4_address):
        """
        IPv4 주소를 NAT64 IPv6 주소로 변환

        예: 192.0.2.1 → 64:ff9b::192.0.2.1 (64:ff9b::c000:201)
        """
        octets = [int(x) for x in ipv4_address.split('.')]
        ipv6_suffix = f"{octets[0]:02x}{octets[1]:02x}:{octets[2]:02x}{octets[3]:02x}"
        return f"64:ff9b::{ipv6_suffix}"

    def translate_packet(self, ipv6_packet):
        """
        IPv6 패킷을 IPv4 패킷으로 변환 (개념적 구현)
        """
        # 헤더 변환
        # - Traffic Class → ToS
        # - Flow Label → 무시
        # - Hop Limit → TTL
        # - 주소 변환
        pass

# 사용 예시
translator = NAT64Translator()
print(translator.ipv4_to_ipv6("192.0.2.1"))  # 64:ff9b::c000:201
```

---

## 3. 라우팅 프로토콜 (BGP, OSPF)

### 3.1 라우팅 프로토콜 분류

```
라우팅 프로토콜
├── IGP (Interior Gateway Protocol) - AS 내부
│   ├── Distance Vector
│   │   ├── RIP (Routing Information Protocol)
│   │   └── EIGRP (Enhanced IGRP)
│   └── Link State
│       ├── OSPF (Open Shortest Path First)
│       └── IS-IS (Intermediate System to IS)
│
└── EGP (Exterior Gateway Protocol) - AS 간
    └── BGP (Border Gateway Protocol) - Path Vector
```

### 3.2 OSPF (Open Shortest Path First)

**RFC 2328 (OSPFv2), RFC 5340 (OSPFv3)**

OSPF는 **링크 상태 라우팅 프로토콜**로, 각 라우터가 네트워크 토폴로지 전체의 정보를 유지하고 Dijkstra 알고리즘을 사용하여 최단 경로를 계산합니다.

**OSPF 영역 구조:**
```
                    ┌─────────────────────────────┐
                    │        Area 0 (Backbone)     │
                    │    ┌───┐     ┌───┐          │
                    │    │ABR│─────│ABR│          │
                    │    └─┬─┘     └─┬─┘          │
                    └──────┼─────────┼────────────┘
                           │         │
              ┌────────────┼─┐     ┌─┼────────────┐
              │  Area 1    │ │     │ │   Area 2   │
              │   ┌───┐    │ │     │ │   ┌───┐    │
              │   │ R │    │ │     │ │   │ R │    │
              │   └───┘    │ │     │ │   └───┘    │
              └────────────┴─┘     └─┴────────────┘

ABR: Area Border Router (영역 경계 라우터)
```

**OSPF 패킷 유형:**
```
┌──────────┬─────────────────────────────────────────────────────┐
│  타입    │                        설명                          │
├──────────┼─────────────────────────────────────────────────────┤
│ Hello    │ 이웃 발견 및 관계 유지 (기본 10초 간격)                │
│ DBD      │ Database Description - LSA 헤더 요약 교환             │
│ LSR      │ Link State Request - 특정 LSA 요청                   │
│ LSU      │ Link State Update - LSA 전달                        │
│ LSAck    │ Link State Acknowledgment - LSA 수신 확인            │
└──────────┴─────────────────────────────────────────────────────┘
```

**OSPF 메트릭 계산:**
```python
# OSPF 비용(Cost) 계산
def calculate_ospf_cost(bandwidth_bps, reference_bandwidth=100_000_000):
    """
    OSPF 인터페이스 비용 계산

    Cost = Reference Bandwidth / Interface Bandwidth
    기본 Reference Bandwidth: 100 Mbps

    Args:
        bandwidth_bps: 인터페이스 대역폭 (bps)
        reference_bandwidth: 참조 대역폭 (기본 100 Mbps)

    Returns:
        cost: OSPF 비용 (최소 1)
    """
    cost = reference_bandwidth / bandwidth_bps
    return max(1, int(cost))

# 예시
interfaces = {
    "FastEthernet (100 Mbps)": 100_000_000,
    "GigabitEthernet (1 Gbps)": 1_000_000_000,
    "10GigabitEthernet (10 Gbps)": 10_000_000_000,
}

for name, bandwidth in interfaces.items():
    cost = calculate_ospf_cost(bandwidth)
    print(f"{name}: Cost = {cost}")

# 출력:
# FastEthernet (100 Mbps): Cost = 1
# GigabitEthernet (1 Gbps): Cost = 1  # 기본 참조 대역폭으로는 구분 불가
# 10GigabitEthernet (10 Gbps): Cost = 1

# 고속 링크 구분을 위해 참조 대역폭 조정 필요
for name, bandwidth in interfaces.items():
    cost = calculate_ospf_cost(bandwidth, reference_bandwidth=10_000_000_000)
    print(f"{name}: Cost = {cost} (ref: 10 Gbps)")

# 출력:
# FastEthernet (100 Mbps): Cost = 100
# GigabitEthernet (1 Gbps): Cost = 10
# 10GigabitEthernet (10 Gbps): Cost = 1
```

**Dijkstra 알고리즘 (SPF):**
```python
import heapq
from collections import defaultdict

def dijkstra_spf(graph, source):
    """
    Dijkstra 알고리즘을 사용한 최단 경로 계산
    OSPF가 내부적으로 사용하는 알고리즘

    Args:
        graph: 인접 리스트 형태의 그래프 {node: [(neighbor, cost), ...]}
        source: 출발 노드

    Returns:
        distances: 각 노드까지의 최단 거리
        previous: 최단 경로 역추적용 이전 노드
    """
    distances = defaultdict(lambda: float('inf'))
    distances[source] = 0
    previous = {}

    # 우선순위 큐: (거리, 노드)
    pq = [(0, source)]
    visited = set()

    while pq:
        current_dist, current = heapq.heappop(pq)

        if current in visited:
            continue
        visited.add(current)

        for neighbor, cost in graph[current]:
            distance = current_dist + cost

            if distance < distances[neighbor]:
                distances[neighbor] = distance
                previous[neighbor] = current
                heapq.heappush(pq, (distance, neighbor))

    return dict(distances), previous

# OSPF 네트워크 토폴로지 예시
ospf_network = {
    'R1': [('R2', 10), ('R3', 5)],
    'R2': [('R1', 10), ('R3', 2), ('R4', 1)],
    'R3': [('R1', 5), ('R2', 2), ('R4', 9)],
    'R4': [('R2', 1), ('R3', 9)]
}

distances, previous = dijkstra_spf(ospf_network, 'R1')
print(f"R1에서 각 라우터까지의 최단 비용: {distances}")
# {'R1': 0, 'R3': 5, 'R2': 7, 'R4': 8}
```

### 3.3 BGP (Border Gateway Protocol)

**RFC 4271 (BGP-4)**

BGP는 **자율 시스템(AS) 간 라우팅을 담당하는 Path Vector 프로토콜**입니다. 인터넷의 핵심 라우팅 프로토콜입니다.

**BGP 특징:**
- TCP 포트 179 사용 (신뢰성 있는 전송)
- 정책 기반 라우팅 (Policy-based Routing)
- 느린 수렴 속도 (안정성 우선)
- AS-PATH를 통한 루프 방지

**BGP 메시지 유형:**
```
┌──────────┬─────────────────────────────────────────────────────┐
│  메시지   │                        설명                          │
├──────────┼─────────────────────────────────────────────────────┤
│ OPEN     │ BGP 세션 시작, AS 번호/Hold Time 등 협상              │
│ UPDATE   │ 경로 정보 광고 (NLRI) 또는 철회 (Withdrawn)           │
│ KEEPALIVE│ 세션 유지 확인 (기본 60초 Hold Time의 1/3)            │
│ NOTIFICATION│ 오류 통보 및 세션 종료                            │
└──────────┴─────────────────────────────────────────────────────┘
```

**BGP Path Attributes:**
```python
# BGP 경로 선택 과정 (Best Path Selection)
class BGPPathSelection:
    """
    BGP Best Path Selection Algorithm
    우선순위 순서대로 적용
    """

    SELECTION_ORDER = [
        ("Weight", "높을수록 우선 (Cisco 전용, 로컬)"),
        ("Local Preference", "높을수록 우선 (AS 내부)"),
        ("Locally Originated", "자체 생성 경로 우선"),
        ("AS-PATH Length", "짧을수록 우선"),
        ("Origin", "IGP > EGP > Incomplete"),
        ("MED", "낮을수록 우선 (Multi-Exit Discriminator)"),
        ("eBGP over iBGP", "eBGP 경로 우선"),
        ("IGP Metric", "넥스트홉까지 IGP 비용 낮은 것"),
        ("Router ID", "낮은 Router ID 우선"),
    ]

    @classmethod
    def display_selection_order(cls):
        print("BGP Best Path Selection Order:")
        for i, (attr, desc) in enumerate(cls.SELECTION_ORDER, 1):
            print(f"  {i}. {attr}: {desc}")

BGPPathSelection.display_selection_order()
```

**iBGP vs eBGP:**
```
┌─────────────────────────────────────────────────────────────────┐
│                         AS 65001                                │
│     ┌─────┐                               ┌─────┐               │
│     │ R1  │◄─────── iBGP (AS 내부) ──────►│ R2  │               │
│     └──┬──┘                               └──┬──┘               │
│        │                                     │                  │
└────────┼─────────────────────────────────────┼──────────────────┘
         │ eBGP (AS 간)                        │ eBGP
         │                                     │
┌────────┼─────────────┐         ┌─────────────┼──────────────────┐
│  AS 65002            │         │   AS 65003                     │
│     ┌──┴──┐          │         │          ┌──┴──┐               │
│     │ R3  │          │         │          │ R4  │               │
│     └─────┘          │         │          └─────┘               │
└──────────────────────┘         └────────────────────────────────┘
```

**iBGP Full-Mesh 문제와 Route Reflector:**
```
iBGP는 다음 홉 변경 없음 → Full-Mesh 필요
n개 라우터: n(n-1)/2 세션 필요

해결책: Route Reflector (RR)
┌────────────────────────────────────────┐
│              ┌────┐                    │
│              │ RR │ (Route Reflector)  │
│              └─┬──┘                    │
│         ┌─────┼─────┐                  │
│         ▼     ▼     ▼                  │
│       ┌───┐ ┌───┐ ┌───┐               │
│       │R1 │ │R2 │ │R3 │ (Clients)     │
│       └───┘ └───┘ └───┘               │
└────────────────────────────────────────┘
```

### 3.4 OSPF vs BGP 비교

| 특성 | OSPF | BGP |
|------|------|-----|
| 유형 | IGP (내부) | EGP (외부) |
| 알고리즘 | Link State (Dijkstra) | Path Vector |
| 메트릭 | 비용 (대역폭 기반) | 정책 기반 (AS-PATH 등) |
| 수렴 속도 | 빠름 | 느림 |
| 확장성 | 중간 (영역으로 분할) | 높음 (인터넷 규모) |
| 프로토콜 | IP (프로토콜 89) | TCP (포트 179) |
| 사용 범위 | 단일 AS 내부 | AS 간 |

---

## 4. 인터넷 구조 (ISP, IXP, Tier)

### 4.1 ISP 계층 구조

```
                        ┌─────────────────────────┐
                        │     Tier 1 ISP          │
                        │  (글로벌 백본)           │
                        │  예: AT&T, NTT, Lumen   │
                        └───────────┬─────────────┘
                                    │ Transit
                    ┌───────────────┼───────────────┐
                    │               │               │
            ┌───────┴───────┐ ┌─────┴─────┐ ┌──────┴──────┐
            │   Tier 2 ISP  │ │ Tier 2 ISP│ │  Tier 2 ISP │
            │ (지역/국가)    │ │           │ │             │
            └───────┬───────┘ └─────┬─────┘ └──────┬──────┘
                    │               │               │
        ┌───────────┼───────┐       │       ┌──────┼──────┐
        │           │       │       │       │      │      │
   ┌────┴────┐ ┌────┴────┐  │  ┌────┴────┐  │ ┌────┴────┐ │
   │Tier 3   │ │Tier 3   │  │  │Tier 3   │  │ │Tier 3   │ │
   │(로컬)   │ │(로컬)   │  │  │(로컬)   │  │ │(로컬)   │ │
   └────┬────┘ └────┬────┘  │  └────┬────┘  │ └────┬────┘ │
        │           │       │       │       │      │      │
   [고객/기업]  [고객/기업]      [고객/기업]      [고객/기업]
```

**Tier 정의:**

| Tier | 특징 | 비용 관계 | 예시 |
|------|------|----------|------|
| **Tier 1** | 전체 인터넷에 Transit 없이 도달 가능 | Peering만 (무료) | AT&T, NTT, Lumen, Telia |
| **Tier 2** | 일부는 Peering, 일부는 Transit 구매 | Transit 비용 발생 | KT, SK브로드밴드, LG U+ |
| **Tier 3** | 상위 ISP로부터 Transit 구매 | Transit 비용 지불 | 지역 케이블 회사 |

### 4.2 IXP (Internet Exchange Point)

**IXP는 여러 ISP들이 트래픽을 직접 교환할 수 있는 물리적 인프라입니다.**

```
                    ┌─────────────────────────────────┐
                    │            IXP                  │
                    │    ┌───────────────────┐        │
        ISP A ──────┼───►│   L2 Switch       │◄───────┼────── ISP B
                    │    │  (고속 스위칭)     │        │
        ISP C ──────┼───►│                   │◄───────┼────── ISP D
                    │    └───────────────────┘        │
        ISP E ──────┼───►        │        ◄──────────┼────── CDN
                    │            │                    │
                    │    ┌───────┴───────┐            │
                    │    │ Route Server  │            │
                    │    │ (BGP 세션 관리)│            │
                    │    └───────────────┘            │
                    └─────────────────────────────────┘
```

**IXP의 장점:**
- **지연 감소**: 트래픽이 상위 ISP를 거치지 않고 직접 교환
- **비용 절감**: Transit 비용 대신 저렴한 Peering 비용
- **대역폭 증가**: 병목 현상 감소
- **지역 트래픽 최적화**: 로컬 트래픽이 지역 내에서 처리

**대표적인 IXP:**
```
┌──────────────────┬─────────────────────┬───────────────────┐
│       이름        │        위치          │    참여 네트워크   │
├──────────────────┼─────────────────────┼───────────────────┤
│ DE-CIX           │ 프랑크푸르트          │      1000+        │
│ AMS-IX           │ 암스테르담            │       800+        │
│ LINX             │ 런던                 │       900+        │
│ Equinix IX       │ 글로벌               │       2000+       │
│ KINX             │ 서울                 │        200+       │
└──────────────────┴─────────────────────┴───────────────────┘
```

### 4.3 Peering vs Transit

```python
# Peering과 Transit의 차이
class InterconnectionType:
    """
    ISP 간 연결 유형 설명
    """

    @staticmethod
    def peering():
        """
        Peering (피어링)
        - 두 네트워크 간 직접 트래픽 교환
        - 일반적으로 무료 (Settlement-free)
        - 각자의 고객 트래픽만 교환
        """
        return {
            "type": "Peering",
            "cost": "일반적으로 무료",
            "traffic": "각자의 고객 트래픽만",
            "relationship": "대등한 관계",
            "example": "Tier 1 ISP 간, 대형 콘텐츠 사업자와 ISP"
        }

    @staticmethod
    def transit():
        """
        Transit (트랜짓)
        - 상위 ISP가 인터넷 전체에 대한 라우팅 제공
        - 비용 발생 (구매하는 쪽이 지불)
        - 전체 인터넷에 접근 가능
        """
        return {
            "type": "Transit",
            "cost": "대역폭 기반 과금 (예: Mbps당 월 $X)",
            "traffic": "인터넷 전체",
            "relationship": "고객-제공자 관계",
            "example": "Tier 2 ISP가 Tier 1으로부터 구매"
        }

# BGP 관점에서의 차이
"""
Peering 관계 (AS 65001 ↔ AS 65002):
- AS 65001은 자신의 고객 경로만 AS 65002에 광고
- AS 65002도 자신의 고객 경로만 AS 65001에 광고
- 서로의 Transit 고객 경로는 광고하지 않음

Transit 관계 (AS 65001 → AS 65003):
- AS 65003 (고객)은 자신의 경로를 AS 65001 (제공자)에 광고
- AS 65001은 인터넷 전체 경로를 AS 65003에 광고
- AS 65001은 AS 65003의 경로를 다른 곳에 광고
"""
```

### 4.4 BGP와 인터넷 구조

```
AS 65001 (Tier 1)          AS 65002 (Tier 1)
      │                          │
      │ Peering                  │
      ├──────────────────────────┤
      │                          │
      │ Transit                  │ Transit
      │                          │
 AS 65010 (Tier 2)         AS 65020 (Tier 2)
      │                          │
      │ Peering (IXP에서)         │
      ├──────────────────────────┤
      │                          │
      │ Transit                  │ Transit
      │                          │
 AS 65100 (Tier 3)         AS 65200 (Tier 3)
      │                          │
 [고객 네트워크]             [고객 네트워크]
```

**인터넷 라우팅 테이블 규모 (2024년 기준):**
- IPv4: 약 100만 개 이상의 프리픽스
- IPv6: 약 20만 개 이상의 프리픽스
- 활성 AS 번호: 약 75,000개 이상

### 4.5 CDN과 인터넷 구조

```
전통적인 구조:
[사용자] → [ISP] → [Tier 2] → [Tier 1] → [Origin Server]

CDN 적용 구조:
                     ┌─────────────────────────────────────┐
                     │              CDN 네트워크            │
[사용자] → [ISP] ──►│  [Edge Server]  ◄── [Origin에서 캐시] │
                     │    (가까운 위치)                     │
                     └─────────────────────────────────────┘

CDN 배치 전략:
1. IXP에 캐시 서버 배치
2. ISP 내부에 캐시 서버 배치 (Deep Peering)
3. 대형 CDN은 자체 글로벌 네트워크 운영
```

---

## 참고 자료

- [RFC 791 - Internet Protocol (IPv4)](https://datatracker.ietf.org/doc/html/rfc791)
- [RFC 8200 - Internet Protocol, Version 6 (IPv6)](https://datatracker.ietf.org/doc/html/rfc8200)
- [RFC 4271 - A Border Gateway Protocol 4 (BGP-4)](https://datatracker.ietf.org/doc/html/rfc4271)
- [RFC 2328 - OSPF Version 2](https://datatracker.ietf.org/doc/html/rfc2328)
- [RFC 6146 - NAT64](https://datatracker.ietf.org/doc/html/rfc6146)
- [APNIC Blog - IP addresses through 2024](https://blog.apnic.net/2025/01/13/ip-addresses-through-2024/)

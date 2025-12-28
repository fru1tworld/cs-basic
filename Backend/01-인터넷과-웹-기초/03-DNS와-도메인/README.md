# DNS와 도메인

> 인터넷의 전화번호부라 불리는 DNS의 동작 원리와 보안 메커니즘을 다룹니다.

## 목차
1. [DNS 계층 구조](#1-dns-계층-구조)
2. [DNS 레코드 타입](#2-dns-레코드-타입)
3. [DNS 쿼리 과정](#3-dns-쿼리-과정)
4. [DNS 보안](#4-dns-보안-dnssec-doh-dot)
---

## 1. DNS 계층 구조

### 1.1 DNS란?

DNS(Domain Name System)는 **사람이 읽을 수 있는 도메인 이름을 IP 주소로 변환**하는 분산 데이터베이스 시스템입니다.

```
www.example.com  →  DNS  →  93.184.216.34

사람: "www.example.com으로 접속하고 싶어"
컴퓨터: "IP 주소가 필요해" → DNS 쿼리 → "93.184.216.34구나!"
```

### 1.2 도메인 네임 구조

```
                    FQDN (Fully Qualified Domain Name)
                    ┌───────────────────────────────────┐
                    │                                   │
                    www.mail.example.com.
                    │     │      │      │
                    │     │      │      └─ Root (암시적 또는 명시적)
                    │     │      └──────── TLD (Top-Level Domain)
                    │     └─────────────── SLD (Second-Level Domain)
                    └───────────────────── Subdomain/Host


도메인 계층 (오른쪽에서 왼쪽으로):
┌─────────────────────────────────────────────────────────────────┐
│                          . (Root)                               │
│                             │                                   │
│         ┌───────────────────┼───────────────────┐               │
│         │                   │                   │               │
│        com                 org                 net              │
│         │                   │                   │               │
│    ┌────┴────┐         ┌────┴────┐              │               │
│    │         │         │         │              │               │
│ example   google    wikipedia  mozilla          │               │
│    │         │         │         │              │               │
│   www      mail      www        │              ...              │
│    │                             │                              │
│ (A: 93.184.216.34)              ...                             │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 TLD 분류

```
┌──────────────────────────────────────────────────────────────────┐
│                         TLD 분류                                 │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│ gTLD (Generic TLD) - 일반 최상위 도메인                           │
│   ├── 원조 gTLD: .com, .net, .org, .edu, .gov, .mil             │
│   ├── 신규 gTLD: .app, .dev, .io, .xyz, .cloud, .blog           │
│   └── 스폰서 gTLD: .aero, .museum, .coop                         │
│                                                                  │
│ ccTLD (Country Code TLD) - 국가 코드 도메인                       │
│   ├── .kr (한국), .jp (일본), .uk (영국), .de (독일)             │
│   └── 특수 용도: .io (영국령), .tv (투발루), .ai (앵귈라)         │
│                                                                  │
│ Infrastructure TLD                                               │
│   └── .arpa (Address and Routing Parameter Area)                │
│       - in-addr.arpa (IPv4 역방향 DNS)                          │
│       - ip6.arpa (IPv6 역방향 DNS)                               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 1.4 DNS 서버 계층

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DNS 서버 계층                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Root DNS Servers                           │  │
│  │   전 세계 13개 Root 서버 (A ~ M)                               │  │
│  │   - 실제로는 Anycast로 수백 대 운영                            │  │
│  │   - TLD 네임서버 정보 보유                                     │  │
│  │   a.root-servers.net ~ m.root-servers.net                    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    TLD DNS Servers                            │  │
│  │   - .com, .net, .org, .kr 등 TLD별 관리                       │  │
│  │   - 해당 TLD 아래 도메인의 권한 네임서버 정보 보유              │  │
│  │   예: a.gtld-servers.net (Verisign이 .com 관리)              │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              Authoritative DNS Servers                        │  │
│  │   - 특정 도메인의 실제 DNS 레코드 보유                          │  │
│  │   - 도메인 소유자가 관리 (또는 DNS 호스팅 서비스)               │  │
│  │   예: ns1.example.com, ns2.example.com                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                      │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              Recursive/Caching DNS Servers                    │  │
│  │   - 클라이언트 대신 DNS 쿼리 수행                              │  │
│  │   - 결과 캐싱으로 성능 향상                                    │  │
│  │   예: 8.8.8.8 (Google), 1.1.1.1 (Cloudflare), ISP DNS         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.5 Root 서버 목록 (2024)

```
┌──────┬──────────────────────────────┬─────────────────────────────┐
│ 이름  │            운영자             │          IPv4/IPv6          │
├──────┼──────────────────────────────┼─────────────────────────────┤
│  A   │ Verisign                     │ 198.41.0.4 / 2001:503:ba3e::2:30 │
│  B   │ USC-ISI                      │ 170.247.170.2 / 2801:1b8:10::b   │
│  C   │ Cogent Communications        │ 192.33.4.12 / 2001:500:2::c      │
│  D   │ University of Maryland       │ 199.7.91.13 / 2001:500:2d::d     │
│  E   │ NASA Ames Research Center    │ 192.203.230.10 / 2001:500:a8::e  │
│  F   │ Internet Systems Consortium  │ 192.5.5.241 / 2001:500:2f::f     │
│  G   │ US DoD Network Info Center   │ 192.112.36.4 / 2001:500:12::d0d  │
│  H   │ US Army Research Lab         │ 198.97.190.53 / 2001:500:1::53   │
│  I   │ Netnod                       │ 192.36.148.17 / 2001:7fe::53     │
│  J   │ Verisign                     │ 192.58.128.30 / 2001:503:c27::2:30 │
│  K   │ RIPE NCC                     │ 193.0.14.129 / 2001:7fd::1       │
│  L   │ ICANN                        │ 199.7.83.42 / 2001:500:9f::42    │
│  M   │ WIDE Project                 │ 202.12.27.33 / 2001:dc3::35      │
└──────┴──────────────────────────────┴─────────────────────────────┘

※ Anycast: 동일 IP로 전 세계 수백 대 서버 운영
   사용자는 가장 가까운 서버에 자동 연결
```

---

## 2. DNS 레코드 타입

### 2.1 주요 레코드 타입 (RFC 1035, RFC 3596 등)

```
┌──────────┬───────────────────────────────────────────────────────────┐
│  타입    │                        설명                                │
├──────────┼───────────────────────────────────────────────────────────┤
│    A     │ 도메인 → IPv4 주소                                        │
│   AAAA   │ 도메인 → IPv6 주소                                        │
│  CNAME   │ 도메인 → 다른 도메인 (별칭)                                │
│   MX     │ 메일 서버 지정                                             │
│   TXT    │ 텍스트 정보 (SPF, DKIM, 도메인 검증 등)                    │
│   NS     │ 네임서버 지정                                              │
│   SOA    │ 영역의 권한 시작 (Zone의 메타데이터)                        │
│   PTR    │ IP 주소 → 도메인 (역방향 DNS)                              │
│   SRV    │ 서비스 위치 (프로토콜, 포트 포함)                           │
│   CAA    │ 인증 기관 권한 부여                                        │
└──────────┴───────────────────────────────────────────────────────────┘
```

### 2.2 A 레코드와 AAAA 레코드

```bash
# A 레코드 (IPv4)
example.com.        300    IN    A     93.184.216.34
www.example.com.    300    IN    A     93.184.216.34

# AAAA 레코드 (IPv6)
example.com.        300    IN    AAAA  2606:2800:220:1:248:1893:25c8:1946

# dig 명령어로 조회
$ dig example.com A +short
93.184.216.34

$ dig example.com AAAA +short
2606:2800:220:1:248:1893:25c8:1946
```

**TTL (Time To Live):**
```python
# TTL 설정 가이드
class TTLStrategy:
    """
    TTL 설정 전략
    """

    # 짧은 TTL (60-300초)
    # - 장점: 변경 시 빠른 전파
    # - 단점: DNS 쿼리 증가, 장애 시 영향 큼
    # - 사용: 로드밸런서, 장애 조치, 배포 중

    # 긴 TTL (3600-86400초)
    # - 장점: DNS 캐시 효율, 성능 향상
    # - 단점: 변경 전파 느림
    # - 사용: 안정적인 정적 리소스

    RECOMMENDATIONS = {
        "production_stable": 3600,      # 1시간 (일반적인 설정)
        "production_dynamic": 300,      # 5분 (자주 변경)
        "before_migration": 60,         # 1분 (마이그레이션 전 낮춤)
        "during_migration": 60,         # 마이그레이션 중
        "after_migration": 3600,        # 안정화 후 다시 높임
    }
```

### 2.3 CNAME 레코드

```bash
# CNAME (Canonical Name) - 별칭
www.example.com.     300    IN    CNAME    example.com.
blog.example.com.    300    IN    CNAME    example.github.io.
cdn.example.com.     300    IN    CNAME    d111111abcdef8.cloudfront.net.

# CNAME 체인 예시
alias.example.com.   →  target.example.com.  →  93.184.216.34
        CNAME               CNAME/A                   최종 IP
```

**CNAME 제약 사항:**
```
❌ Zone Apex (루트 도메인)에 CNAME 사용 불가

잘못된 설정:
example.com.    IN    CNAME    other.com.    # 불가능!

이유:
- CNAME은 해당 이름의 모든 레코드를 대체
- Zone Apex에는 SOA, NS 레코드가 필수
- CNAME과 다른 레코드 공존 불가

해결책:
1. A/AAAA 레코드 직접 사용
2. ALIAS/ANAME 레코드 (일부 DNS 제공자)
3. CNAME Flattening (Cloudflare 등)
```

### 2.4 MX 레코드

```bash
# MX (Mail Exchanger) 레코드
# 우선순위가 낮을수록 먼저 시도
example.com.    300    IN    MX    10 mail1.example.com.
example.com.    300    IN    MX    20 mail2.example.com.
example.com.    300    IN    MX    30 backup.example.com.

# 메일 전송 순서:
# 1. mail1.example.com (우선순위 10) 시도
# 2. 실패 시 mail2.example.com (우선순위 20) 시도
# 3. 실패 시 backup.example.com (우선순위 30) 시도

# 메일 서버 A 레코드도 필요
mail1.example.com.    IN    A    192.0.2.1
mail2.example.com.    IN    A    192.0.2.2
```

### 2.5 TXT 레코드

```bash
# TXT 레코드 - 다양한 용도로 사용

# 1. SPF (Sender Policy Framework) - 이메일 발신 서버 인증
example.com.    IN    TXT    "v=spf1 include:_spf.google.com include:sendgrid.net -all"

# SPF 문법:
# v=spf1          : SPF 버전
# include:domain  : 해당 도메인의 SPF 정책 포함
# a               : 도메인의 A 레코드 IP 허용
# mx              : MX 서버 IP 허용
# ip4:192.0.2.1   : 특정 IP 허용
# -all            : 나머지는 거부 (hard fail)
# ~all            : 나머지는 의심 (soft fail)

# 2. DKIM (DomainKeys Identified Mail) - 이메일 서명
selector1._domainkey.example.com.    IN    TXT    "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEB..."

# 3. DMARC (Domain-based Message Authentication)
_dmarc.example.com.    IN    TXT    "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"

# 4. 도메인 소유권 검증 (Google, Let's Encrypt 등)
example.com.    IN    TXT    "google-site-verification=abcdefg..."
_acme-challenge.example.com.    IN    TXT    "challenge-token..."
```

### 2.6 NS 레코드와 SOA 레코드

```bash
# NS (Name Server) 레코드
example.com.    IN    NS    ns1.example.com.
example.com.    IN    NS    ns2.example.com.

# Glue Record (네임서버가 같은 도메인 내에 있을 때)
ns1.example.com.    IN    A    192.0.2.1
ns2.example.com.    IN    A    192.0.2.2

# SOA (Start of Authority) 레코드
example.com.    IN    SOA    ns1.example.com. admin.example.com. (
                    2024010101  ; Serial (YYYYMMDDNN 형식 권장)
                    3600        ; Refresh (1시간)
                    600         ; Retry (10분)
                    604800      ; Expire (1주)
                    86400       ; Minimum TTL (1일)
                )

# SOA 필드 설명:
# - MNAME: Primary 네임서버
# - RNAME: 관리자 이메일 (@ → .)
# - Serial: 영역 버전 (변경 시 증가)
# - Refresh: Secondary가 Primary 확인 주기
# - Retry: Refresh 실패 시 재시도 간격
# - Expire: 연락 안 될 때 데이터 만료 시간
# - Minimum: Negative 캐싱 TTL
```

### 2.7 SRV 레코드와 PTR 레코드

```bash
# SRV (Service) 레코드
# _service._proto.name.    TTL    IN    SRV    priority weight port target.

# LDAP 서비스
_ldap._tcp.example.com.    IN    SRV    0 100 389 ldap.example.com.

# SIP 서비스
_sip._tcp.example.com.     IN    SRV    10 60 5060 sip1.example.com.
_sip._tcp.example.com.     IN    SRV    10 40 5060 sip2.example.com.

# SRV 필드:
# priority: 우선순위 (낮을수록 먼저)
# weight: 같은 우선순위 내 가중치 (로드밸런싱)
# port: 서비스 포트
# target: 대상 호스트


# PTR (Pointer) 레코드 - 역방향 DNS
# IP → 도메인 변환

# IPv4 역방향 (in-addr.arpa)
# 192.0.2.1 → 1.2.0.192.in-addr.arpa
1.2.0.192.in-addr.arpa.    IN    PTR    mail.example.com.

# IPv6 역방향 (ip6.arpa)
# 2001:db8::1 → 1.0.0.0.0.0.0.0...8.b.d.0.1.0.0.2.ip6.arpa
# (각 4비트를 역순으로)

# PTR 용도:
# - 이메일 서버 검증 (Forward-Confirmed Reverse DNS)
# - 로그의 IP를 읽기 쉽게 변환
# - 일부 서비스의 인증
```

### 2.8 CAA 레코드 (RFC 8659)

```bash
# CAA (Certificate Authority Authorization)
# 어떤 CA가 인증서를 발급할 수 있는지 지정

example.com.    IN    CAA    0 issue "letsencrypt.org"
example.com.    IN    CAA    0 issue "digicert.com"
example.com.    IN    CAA    0 issuewild "letsencrypt.org"
example.com.    IN    CAA    0 iodef "mailto:security@example.com"

# CAA 태그:
# issue: 일반 인증서 발급 허용 CA
# issuewild: 와일드카드 인증서 발급 허용 CA
# iodef: 정책 위반 시 보고 받을 주소

# CAA가 없으면: 모든 CA가 발급 가능
# CAA가 있으면: 명시된 CA만 발급 가능
```

---

## 3. DNS 쿼리 과정

### 3.1 Recursive vs Iterative 쿼리

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Recursive Query (재귀적)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   클라이언트 ───── "www.example.com은?" ─────► 리졸버               │
│       │                                           │                 │
│       │                                     (리졸버가 대신 조회)     │
│       │                                           │                 │
│       │                                     Root, TLD, Auth 순회    │
│       │                                           │                 │
│   클라이언트 ◄──── "93.184.216.34야" ──────── 리졸버               │
│                                                                     │
│   특징: 클라이언트는 한 번만 질문, 리졸버가 모든 것 처리              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      Iterative Query (반복적)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   클라이언트 ───── "www.example.com?" ─────► Root Server           │
│       │                                           │                 │
│       │       "com은 여기 물어봐" (Referral)      │                 │
│       │ ◄─────────────────────────────────────────┤                 │
│       │                                                             │
│       ├───── "www.example.com?" ─────► TLD Server (.com)           │
│       │                                           │                 │
│       │   "example.com은 여기" (Referral)         │                 │
│       │ ◄─────────────────────────────────────────┤                 │
│       │                                                             │
│       ├───── "www.example.com?" ─────► Auth Server (example.com)   │
│       │                                           │                 │
│       │      "93.184.216.34" (Answer)             │                 │
│       │ ◄─────────────────────────────────────────┤                 │
│                                                                     │
│   특징: 클라이언트가 직접 여러 서버에 질문                           │
│   실제로는 리졸버가 이 방식으로 동작                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 전체 DNS 쿼리 과정

```
┌────────────────────────────────────────────────────────────────────────┐
│                 www.example.com 조회 전체 과정                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌──────────┐                                                          │
│  │  사용자   │                                                          │
│  │ 브라우저  │                                                          │
│  └────┬─────┘                                                          │
│       │                                                                │
│       │ 1. "www.example.com" 입력                                      │
│       ▼                                                                │
│  ┌──────────┐                                                          │
│  │ 브라우저  │ 2. 브라우저 DNS 캐시 확인                                 │
│  │  캐시    │    → 있으면 바로 사용                                     │
│  └────┬─────┘                                                          │
│       │ (캐시 없음)                                                    │
│       ▼                                                                │
│  ┌──────────┐                                                          │
│  │   OS     │ 3. OS DNS 캐시 확인 (/etc/hosts 포함)                    │
│  │  캐시    │    → 있으면 바로 사용                                     │
│  └────┬─────┘                                                          │
│       │ (캐시 없음)                                                    │
│       ▼                                                                │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐   │
│  │Recursive │ 4-a. │   Root   │      │   TLD    │      │   Auth   │   │
│  │ Resolver │─────►│  Server  │      │  Server  │      │  Server  │   │
│  │(8.8.8.8) │      │          │      │  (.com)  │      │(example) │   │
│  │          │◄─────│ Referral │      │          │      │          │   │
│  │          │      │ to .com  │      │          │      │          │   │
│  │          │      └──────────┘      │          │      │          │   │
│  │          │ 4-b.                   │          │      │          │   │
│  │          │────────────────────────►          │      │          │   │
│  │          │◄────────────────────────│ Referral│      │          │   │
│  │          │                        │to example│      │          │   │
│  │          │                        └──────────┘      │          │   │
│  │          │ 4-c.                                     │          │   │
│  │          │──────────────────────────────────────────►          │   │
│  │          │◄──────────────────────────────────────────│ Answer: │   │
│  │          │                                          │93.184.. │   │
│  └────┬─────┘                                          └──────────┘   │
│       │                                                                │
│       │ 5. 캐시에 저장 + 응답 반환                                      │
│       ▼                                                                │
│  ┌──────────┐                                                          │
│  │ 브라우저  │ 6. TCP 연결 → HTTP 요청                                  │
│  └──────────┘                                                          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 3.3 DNS 메시지 구조 (RFC 1035)

```
                                    1  1  1  1  1  1
      0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                      ID                       |  트랜잭션 식별자
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |  플래그
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    QDCOUNT                    |  질문 섹션 수
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ANCOUNT                    |  응답 레코드 수
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    NSCOUNT                    |  권한 레코드 수
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                    ARCOUNT                    |  추가 레코드 수
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                    Question                   /  질문 섹션
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                    Answer                     /  응답 섹션
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                   Authority                   /  권한 섹션
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                   Additional                  /  추가 섹션
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+

플래그 상세:
- QR: 0=Query, 1=Response
- Opcode: 0=표준 쿼리
- AA: Authoritative Answer
- TC: Truncated (UDP 512바이트 초과)
- RD: Recursion Desired
- RA: Recursion Available
- RCODE: 0=No Error, 3=NXDOMAIN
```

### 3.4 Python DNS 쿼리 예제

```python
import dns.resolver
import dns.query
import dns.message
import dns.rdatatype

def dns_lookup_simple(domain, record_type='A'):
    """
    단순 DNS 조회 (시스템 리졸버 사용)
    """
    try:
        answers = dns.resolver.resolve(domain, record_type)
        results = []
        for rdata in answers:
            results.append({
                'value': str(rdata),
                'ttl': answers.rrset.ttl
            })
        return results
    except dns.resolver.NXDOMAIN:
        return {'error': 'Domain not found'}
    except dns.resolver.NoAnswer:
        return {'error': f'No {record_type} record'}

def trace_dns_resolution(domain):
    """
    DNS 해석 과정 추적 (Root부터 순차적으로)
    """
    # Root 서버 시작
    root_servers = ['198.41.0.4', '199.9.14.201', '192.33.4.12']

    name = dns.name.from_text(domain)
    depth = 0
    trace = []

    current_servers = root_servers
    current_name = dns.name.root

    while True:
        # 현재 서버에 쿼리
        for server in current_servers:
            try:
                query = dns.message.make_query(name, dns.rdatatype.A)
                response = dns.query.udp(query, server, timeout=5)

                trace.append({
                    'depth': depth,
                    'server': server,
                    'is_authoritative': bool(response.flags & dns.flags.AA),
                    'answers': len(response.answer),
                    'authorities': len(response.authority)
                })

                # 응답에 Answer가 있으면 완료
                if response.answer:
                    for rrset in response.answer:
                        trace.append({
                            'result': str(rrset),
                            'type': 'answer'
                        })
                    return trace

                # Referral 처리 (Authority 섹션에서 다음 서버 찾기)
                if response.authority:
                    # Additional 섹션에서 NS의 IP 찾기
                    next_servers = []
                    for rrset in response.additional:
                        if rrset.rdtype == dns.rdatatype.A:
                            for rdata in rrset:
                                next_servers.append(str(rdata))

                    if next_servers:
                        current_servers = next_servers[:3]
                        depth += 1
                        break

            except Exception as e:
                continue

        else:
            trace.append({'error': 'Resolution failed'})
            break

    return trace

# 사용 예시
print("=== A 레코드 조회 ===")
print(dns_lookup_simple('www.google.com', 'A'))

print("\n=== MX 레코드 조회 ===")
print(dns_lookup_simple('google.com', 'MX'))

print("\n=== TXT 레코드 조회 ===")
print(dns_lookup_simple('google.com', 'TXT'))
```

### 3.5 dig 명령어 활용

```bash
# 기본 A 레코드 조회
$ dig example.com

# 특정 레코드 타입 조회
$ dig example.com MX
$ dig example.com TXT
$ dig example.com NS
$ dig example.com ANY  # 모든 레코드 (일부 서버에서 거부)

# 간략한 출력
$ dig example.com +short
93.184.216.34

# 추적 모드 (Root부터 전체 과정)
$ dig example.com +trace

# 특정 DNS 서버로 쿼리
$ dig @8.8.8.8 example.com
$ dig @1.1.1.1 example.com

# 역방향 DNS
$ dig -x 93.184.216.34

# TCP 사용 (대용량 응답)
$ dig example.com +tcp

# DNSSEC 정보 포함
$ dig example.com +dnssec

# 응답 시간 측정
$ dig example.com | grep "Query time"
;; Query time: 23 msec
```

---

## 4. DNS 보안 (DNSSEC, DoH, DoT)

### 4.1 DNS 보안 위협

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DNS 보안 위협                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ 1. DNS Spoofing / Cache Poisoning                                  │
│    ┌─────────┐                        ┌─────────┐                   │
│    │ 사용자   │ ─── "bank.com?" ───►  │  DNS    │                   │
│    │         │                        │ Server  │                   │
│    │         │ ◄─ "악성 IP" ───────── │(poisoned)│                  │
│    └─────────┘                        └─────────┘                   │
│         │                                                           │
│         └──────────────────► 피싱 사이트                             │
│                                                                     │
│ 2. Man-in-the-Middle (평문 DNS 도청)                                │
│    ┌─────────┐            ┌─────────┐            ┌─────────┐        │
│    │ 사용자   │ ─ DNS ──► │ 공격자   │ ─ DNS ──► │  DNS    │        │
│    │         │ ◄─────────  │(도청/변조)│ ◄──────── │ Server  │        │
│    └─────────┘            └─────────┘            └─────────┘        │
│                                                                     │
│ 3. DNS Tunneling (데이터 유출)                                       │
│    DNS 쿼리에 데이터를 인코딩하여 방화벽 우회                          │
│    data.encoded.malicious-domain.com                                │
│                                                                     │
│ 4. DDoS Amplification Attack                                        │
│    작은 쿼리로 큰 응답 유도 → 피해자에게 반사                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 DNSSEC (DNS Security Extensions)

**DNSSEC은 DNS 응답의 무결성과 인증을 제공합니다.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DNSSEC 동작 원리                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Chain of Trust (신뢰 체인)                                         │
│                                                                     │
│  ┌───────────────────────────────────────────────────┐              │
│  │                    Root Zone                       │              │
│  │   Trust Anchor (루트 키 - OS/리졸버에 내장)          │              │
│  │   KSK로 ZSK 서명, ZSK로 하위 레코드 서명            │              │
│  └───────────────────────────┬───────────────────────┘              │
│                              │ DS 레코드                             │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────┐              │
│  │                    .com Zone                       │              │
│  │   .com KSK/ZSK로 서명                              │              │
│  │   example.com의 DS 레코드 보유                     │              │
│  └───────────────────────────┬───────────────────────┘              │
│                              │ DS 레코드                             │
│                              ▼                                      │
│  ┌───────────────────────────────────────────────────┐              │
│  │                 example.com Zone                   │              │
│  │   example.com KSK/ZSK로 모든 레코드 서명           │              │
│  │   RRSIG: www.example.com A의 서명                  │              │
│  └───────────────────────────────────────────────────┘              │
│                                                                     │
│  레코드 검증: 서명(RRSIG) → 공개키(DNSKEY) → DS → 상위 DS → Root     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**DNSSEC 관련 레코드:**
```bash
# DNSKEY: 영역의 공개키
example.com.    IN    DNSKEY    256 3 13 (
                      oJMRESz5E4gYzS/q6XDrvU1qMPYIjCWzJaOau8XNEZeq
                      ... )

# 256 = Zone Signing Key (ZSK)
# 257 = Key Signing Key (KSK)
# 3 = DNSSEC 프로토콜
# 13 = ECDSAP256SHA256 알고리즘

# RRSIG: 레코드 서명
www.example.com.    IN    RRSIG    A 13 3 300 (
                          20241231235959 20241201000000 12345
                          example.com.
                          signature... )

# DS (Delegation Signer): 하위 영역 키의 해시 (상위 영역에 저장)
example.com.    IN    DS    12345 13 2 (
                      hash-of-child-dnskey... )

# NSEC/NSEC3: 존재하지 않는 레코드 증명 (Negative Answer)
example.com.    IN    NSEC    www.example.com. A MX TXT RRSIG NSEC DNSKEY
```

**DNSSEC 검증:**
```bash
# dig로 DNSSEC 조회
$ dig example.com +dnssec

# AD 플래그 확인 (Authenticated Data)
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 2

# DNSSEC 검증 과정 추적
$ dig example.com +trace +dnssec
```

### 4.3 DNS over HTTPS (DoH)

**DoH는 DNS 쿼리를 HTTPS로 암호화합니다 (RFC 8484).**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DNS over HTTPS (DoH)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  기존 DNS (평문):                                                    │
│  Client ─── UDP:53 ──► DNS Server                                   │
│  (누구나 쿼리 내용 볼 수 있음)                                        │
│                                                                     │
│  DoH:                                                               │
│  Client ─── HTTPS:443 ──► DoH Server                                │
│  (암호화, 일반 HTTPS 트래픽과 구분 불가)                               │
│                                                                     │
│  DoH 요청 형식:                                                      │
│  GET https://dns.example.com/dns-query?dns=AAABAAABAAA...           │
│  POST https://dns.example.com/dns-query                             │
│       Content-Type: application/dns-message                         │
│       Body: <binary DNS message>                                    │
│                                                                     │
│  응답:                                                               │
│  Content-Type: application/dns-message                              │
│  Body: <binary DNS response>                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**DoH 서버 예시:**
```
Google:      https://dns.google/dns-query
Cloudflare:  https://cloudflare-dns.com/dns-query
             https://1.1.1.1/dns-query
Quad9:       https://dns.quad9.net/dns-query
```

**Python DoH 클라이언트:**
```python
import httpx
import dns.message
import base64

async def doh_query(domain: str, record_type: str = 'A') -> dict:
    """
    DNS over HTTPS 쿼리 수행
    """
    # DNS 메시지 생성
    query = dns.message.make_query(domain, record_type)
    wire = query.to_wire()

    # Base64 URL-safe 인코딩 (GET 요청용)
    dns_b64 = base64.urlsafe_b64encode(wire).decode().rstrip('=')

    async with httpx.AsyncClient(http2=True) as client:
        # GET 방식
        response = await client.get(
            f"https://cloudflare-dns.com/dns-query?dns={dns_b64}",
            headers={"Accept": "application/dns-message"}
        )

        # 응답 파싱
        dns_response = dns.message.from_wire(response.content)

        results = []
        for rrset in dns_response.answer:
            for rdata in rrset:
                results.append({
                    'name': str(rrset.name),
                    'type': dns.rdatatype.to_text(rrset.rdtype),
                    'ttl': rrset.ttl,
                    'value': str(rdata)
                })

        return results

# 사용 예시
import asyncio
result = asyncio.run(doh_query('example.com', 'A'))
print(result)
```

### 4.4 DNS over TLS (DoT)

**DoT는 TLS로 DNS 쿼리를 암호화합니다 (RFC 7858).**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DNS over TLS (DoT)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  전용 포트 853 사용                                                  │
│                                                                     │
│  Client ─── TLS:853 ──► DoT Server                                  │
│                                                                     │
│  연결 과정:                                                          │
│  1. TCP 연결 (포트 853)                                              │
│  2. TLS Handshake                                                   │
│  3. 암호화된 DNS 쿼리/응답                                            │
│                                                                     │
│  DoH vs DoT 비교:                                                    │
│  ┌────────────┬────────────────────┬────────────────────┐            │
│  │    특성    │       DoH          │       DoT          │            │
│  ├────────────┼────────────────────┼────────────────────┤            │
│  │    포트    │ 443 (HTTPS 공유)   │ 853 (전용)         │            │
│  │  프로토콜  │ HTTP/2, HTTP/3     │ TCP + TLS          │            │
│  │   식별성   │ 어려움             │ 쉬움               │            │
│  │   차단     │ 어려움             │ 포트 차단 가능      │            │
│  │   구현     │ 애플리케이션 레벨   │ OS/리졸버 레벨     │            │
│  │   오버헤드 │ 약간 높음          │ 낮음               │            │
│  └────────────┴────────────────────┴────────────────────┘            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.5 DNS over QUIC (DoQ)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DNS over QUIC (DoQ) - RFC 9250                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  QUIC의 장점을 DNS에 적용:                                           │
│  - 0-RTT 연결 재개로 빠른 쿼리                                        │
│  - UDP 기반으로 HOL Blocking 없음                                    │
│  - 연결 마이그레이션 지원                                             │
│                                                                     │
│  포트: UDP 853 (DoT와 동일 포트, 다른 프로토콜)                        │
│                                                                     │
│  현재 상태 (2024):                                                   │
│  - 일부 DNS 제공자에서 지원 시작                                      │
│  - AdGuard DNS, NextDNS 등에서 사용 가능                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.6 DNSSEC vs DoH/DoT 비교

```
┌────────────────────────────────────────────────────────────────────┐
│              DNSSEC vs 암호화된 DNS (DoH/DoT) 비교                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│                    DNSSEC                  DoH/DoT                 │
│  ┌─────────────┬─────────────────────┬─────────────────────┐       │
│  │   목적      │ 무결성 + 인증        │ 기밀성 (암호화)       │       │
│  │   보호 대상 │ DNS 데이터 자체      │ 전송 중 쿼리/응답    │       │
│  │   암호화    │ X (서명만)          │ O                    │       │
│  │   위변조 방지│ O                   │ O (경로상)           │       │
│  │   프라이버시│ X (쿼리 노출)        │ O                    │       │
│  │   배포 필요 │ 모든 DNS 서버       │ 리졸버만             │       │
│  └─────────────┴─────────────────────┴─────────────────────┘       │
│                                                                    │
│  결론: DNSSEC과 DoH/DoT는 보완적 관계                                │
│        최상의 보안을 위해 둘 다 사용 권장                             │
│                                                                    │
│        DNSSEC: "받은 데이터가 진짜인지 검증"                          │
│        DoH/DoT: "쿼리를 아무도 볼 수 없게 암호화"                      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [RFC 1035 - Domain Names - Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035)
- [RFC 4033/4034/4035 - DNSSEC](https://datatracker.ietf.org/doc/html/rfc4033)
- [RFC 8484 - DNS over HTTPS (DoH)](https://datatracker.ietf.org/doc/html/rfc8484)
- [RFC 7858 - DNS over TLS (DoT)](https://datatracker.ietf.org/doc/html/rfc7858)
- [RFC 9250 - DNS over QUIC (DoQ)](https://datatracker.ietf.org/doc/html/rfc9250)
- [Cloudflare - DNS over TLS vs DNS over HTTPS](https://www.cloudflare.com/learning/dns/dns-over-tls/)
- [Root Servers](https://www.iana.org/domains/root/servers)

# 프록시와 VPN

## 목차
1. [개요](#개요)
2. [Forward Proxy](#forward-proxy)
3. [Reverse Proxy](#reverse-proxy)
4. [Transparent Proxy](#transparent-proxy)
5. [VPN 프로토콜](#vpn-프로토콜)
6. [터널링](#터널링)
7. [실무 적용](#실무-적용)
---

## 개요

프록시와 VPN은 네트워크 통신에서 중간자 역할을 수행하여 보안, 익명성, 접근 제어, 성능 최적화 등 다양한 목적으로 사용됩니다.

### 기본 개념 비교

```
┌─────────────────────────────────────────────────────────────┐
│                    프록시 vs VPN                             │
├────────────────────────────┬────────────────────────────────┤
│         프록시              │            VPN                 │
├────────────────────────────┼────────────────────────────────┤
│ • 애플리케이션 레벨         │ • 네트워크 레벨 (전체 트래픽)   │
│ • 특정 프로토콜만 처리      │ • 모든 프로토콜 처리            │
│   (HTTP, SOCKS 등)         │                                │
│ • 암호화 선택적             │ • 전체 트래픽 암호화            │
│ • 설정이 간단               │ • 전용 클라이언트 필요          │
│ • 웹 캐싱, 필터링에 유용    │ • 원격 네트워크 접근에 유용     │
└────────────────────────────┴────────────────────────────────┘
```

### 네트워크 위치에 따른 분류

```
Forward Proxy (클라이언트 측):

┌────────┐     ┌─────────────┐     ┌────────────┐
│ Client │ ──> │ Forward     │ ──> │ Internet   │
│        │     │ Proxy       │     │ Server     │
└────────┘     └─────────────┘     └────────────┘
    클라이언트가 프록시를 통해 외부 접근


Reverse Proxy (서버 측):

┌────────┐     ┌─────────────┐     ┌────────────┐
│ Client │ ──> │ Reverse     │ ──> │ Backend    │
│        │     │ Proxy       │     │ Server     │
└────────┘     └─────────────┘     └────────────┘
    외부 요청을 서버 대신 수신


VPN (네트워크 연결):

┌────────┐     ┌─────────────┐     ┌────────────┐
│ Client │ ══> │ Encrypted   │ ══> │ VPN        │ ──> Internet
│        │     │ Tunnel      │     │ Server     │
└────────┘     └─────────────┘     └────────────┘
    암호화된 터널로 전체 트래픽 전송
```

---

## Forward Proxy

### 개요

Forward Proxy(정방향 프록시)는 클라이언트를 대신하여 외부 서버에 요청을 전달하는 프록시입니다.

```
Forward Proxy 동작:

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   ┌────────┐                                                │
│   │Client A│──┐                                             │
│   └────────┘  │      ┌──────────────┐     ┌────────────┐   │
│               ├─────>│              │     │            │   │
│   ┌────────┐  │      │   Forward    │────>│  Internet  │   │
│   │Client B│──┼─────>│   Proxy      │     │  Server    │   │
│   └────────┘  │      │              │<────│            │   │
│               │      └──────────────┘     └────────────┘   │
│   ┌────────┐  │                                             │
│   │Client C│──┘                                             │
│   └────────┘                                                │
│                                                             │
│   Internal Network            DMZ           Internet        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 주요 기능

| 기능 | 설명 |
|------|------|
| **익명성** | 클라이언트 IP 숨김, 프록시 IP로 요청 |
| **접근 제어** | 특정 사이트 차단/허용 |
| **캐싱** | 응답 캐싱으로 대역폭 절약 및 속도 향상 |
| **콘텐츠 필터링** | 악성 코드, 부적절한 콘텐츠 차단 |
| **로깅/모니터링** | 사용자 활동 기록 |
| **대역폭 제어** | 트래픽 제한 및 QoS |

### HTTP Forward Proxy 구현 (Python)

```python
import socket
import threading

class ForwardProxy:
    """간단한 HTTP Forward Proxy 구현"""

    def __init__(self, host='0.0.0.0', port=8888):
        self.host = host
        self.port = port
        self.blocked_domains = {'blocked.com', 'malware.net'}

    def start(self):
        server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server.bind((self.host, self.port))
        server.listen(100)

        print(f"Proxy server listening on {self.host}:{self.port}")

        while True:
            client_socket, addr = server.accept()
            thread = threading.Thread(
                target=self.handle_client,
                args=(client_socket, addr)
            )
            thread.daemon = True
            thread.start()

    def parse_request(self, request):
        """HTTP 요청 파싱"""
        lines = request.split(b'\r\n')
        first_line = lines[0].decode()
        method, url, version = first_line.split()

        # CONNECT 메서드 (HTTPS)
        if method == 'CONNECT':
            host, port = url.split(':')
            return method, host, int(port), None

        # HTTP URL에서 호스트 추출
        if url.startswith('http://'):
            url = url[7:]
            if '/' in url:
                host_port, path = url.split('/', 1)
                path = '/' + path
            else:
                host_port = url
                path = '/'

            if ':' in host_port:
                host, port = host_port.split(':')
                port = int(port)
            else:
                host = host_port
                port = 80
        else:
            # Host 헤더에서 추출
            for line in lines[1:]:
                if line.lower().startswith(b'host:'):
                    host = line.split(b':')[1].strip().decode()
                    break
            port = 80
            path = url

        return method, host, port, path

    def handle_client(self, client_socket, addr):
        """클라이언트 요청 처리"""
        try:
            request = client_socket.recv(4096)
            if not request:
                return

            method, host, port, path = self.parse_request(request)

            # 도메인 차단 확인
            if host in self.blocked_domains:
                response = (
                    b"HTTP/1.1 403 Forbidden\r\n"
                    b"Content-Type: text/html\r\n\r\n"
                    b"<h1>Access Denied</h1>"
                )
                client_socket.send(response)
                return

            print(f"[{method}] {host}:{port}")

            if method == 'CONNECT':
                # HTTPS 터널링 (CONNECT)
                self.handle_connect(client_socket, host, port)
            else:
                # HTTP 요청 전달
                self.forward_request(client_socket, host, port, request)

        except Exception as e:
            print(f"Error: {e}")
        finally:
            client_socket.close()

    def handle_connect(self, client_socket, host, port):
        """HTTPS CONNECT 메서드 처리 (터널링)"""
        try:
            remote_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            remote_socket.connect((host, port))

            # 연결 성공 응답
            client_socket.send(b"HTTP/1.1 200 Connection Established\r\n\r\n")

            # 양방향 터널링
            self.tunnel(client_socket, remote_socket)

        except Exception as e:
            client_socket.send(b"HTTP/1.1 502 Bad Gateway\r\n\r\n")

    def tunnel(self, client_socket, remote_socket):
        """양방향 데이터 터널링"""
        client_socket.setblocking(False)
        remote_socket.setblocking(False)

        import select
        sockets = [client_socket, remote_socket]

        while True:
            readable, _, exceptional = select.select(sockets, [], sockets, 60)

            if exceptional:
                break

            for sock in readable:
                try:
                    data = sock.recv(4096)
                    if not data:
                        return

                    if sock is client_socket:
                        remote_socket.send(data)
                    else:
                        client_socket.send(data)
                except:
                    return

    def forward_request(self, client_socket, host, port, request):
        """HTTP 요청 전달"""
        try:
            remote_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            remote_socket.connect((host, port))
            remote_socket.send(request)

            while True:
                response = remote_socket.recv(4096)
                if not response:
                    break
                client_socket.send(response)

            remote_socket.close()
        except Exception as e:
            print(f"Forward error: {e}")

if __name__ == "__main__":
    proxy = ForwardProxy()
    proxy.start()
```

### SOCKS Proxy

```
SOCKS vs HTTP Proxy:

┌────────────────────────┬────────────────────────────────────┐
│     HTTP Proxy         │         SOCKS Proxy                │
├────────────────────────┼────────────────────────────────────┤
│ • HTTP/HTTPS만 지원     │ • 모든 TCP 트래픽 지원             │
│ • 애플리케이션 레벨     │ • 세션 레벨 (OSI 5계층)            │
│ • 요청 수정 가능        │ • 데이터 투명 전달                 │
│ • 캐싱 가능             │ • 캐싱 불가                        │
└────────────────────────┴────────────────────────────────────┘

SOCKS5 프로토콜:
1. 클라이언트 → 프록시: 인증 방법 협상
2. 프록시 → 클라이언트: 인증 방법 선택
3. 클라이언트 → 프록시: 연결 요청 (CONNECT/BIND/UDP ASSOCIATE)
4. 프록시 → 클라이언트: 연결 결과
5. 데이터 전송
```

```python
# SOCKS5 클라이언트 사용 예시
import socks
import socket

# SOCKS5 프록시 설정
socks.set_default_proxy(socks.SOCKS5, "127.0.0.1", 1080)
socket.socket = socks.socksocket

# 이후 모든 소켓 통신이 SOCKS5 프록시를 통해 전달
import requests
response = requests.get('https://api.example.com/data')
```

---

## Reverse Proxy

### 개요

Reverse Proxy(역방향 프록시)는 서버를 대신하여 클라이언트 요청을 수신하고 백엔드 서버에 전달합니다.

```
Reverse Proxy 구조:

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Internet         Reverse Proxy        Backend Servers      │
│                                                             │
│  ┌────────┐      ┌─────────────┐      ┌────────────────┐   │
│  │Client 1│──┐   │             │  ┌──>│ Web Server 1   │   │
│  └────────┘  │   │             │  │   └────────────────┘   │
│              │   │   Nginx/    │  │                        │
│  ┌────────┐  ├──>│   HAProxy   │──┼──>┌────────────────┐   │
│  │Client 2│  │   │             │  │   │ Web Server 2   │   │
│  └────────┘  │   │             │  │   └────────────────┘   │
│              │   │ - SSL 종료   │  │                        │
│  ┌────────┐  │   │ - 로드밸런싱│  └──>┌────────────────┐   │
│  │Client 3│──┘   │ - 캐싱      │      │ Web Server 3   │   │
│  └────────┘      │ - 압축      │      └────────────────┘   │
│                  └─────────────┘                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 주요 기능

| 기능 | 설명 |
|------|------|
| **로드 밸런싱** | 트래픽을 여러 백엔드 서버에 분산 |
| **SSL/TLS 종료** | HTTPS 암호화/복호화를 프록시에서 처리 |
| **캐싱** | 정적 콘텐츠 캐싱으로 백엔드 부하 감소 |
| **압축** | Gzip/Brotli 압축으로 응답 크기 감소 |
| **보안** | 백엔드 서버 IP 숨김, WAF 기능 |
| **헤더 조작** | 요청/응답 헤더 추가/수정/제거 |
| **레이트 리미팅** | 요청 속도 제한 |
| **A/B 테스팅** | 트래픽을 다른 버전으로 분기 |

### Nginx Reverse Proxy 설정

```nginx
# /etc/nginx/nginx.conf

http {
    # Upstream 서버 정의
    upstream backend {
        # 로드 밸런싱 방식
        # - (default) round-robin
        # - least_conn: 최소 연결 수
        # - ip_hash: 클라이언트 IP 기반 세션 고정
        # - hash: 커스텀 키 기반

        least_conn;

        server 10.0.0.1:8080 weight=3;  # 가중치
        server 10.0.0.2:8080 weight=2;
        server 10.0.0.3:8080 backup;     # 백업 서버

        # Keep-Alive 연결 풀
        keepalive 32;
    }

    # 캐싱 설정
    proxy_cache_path /var/cache/nginx levels=1:2
                     keys_zone=my_cache:10m
                     max_size=10g
                     inactive=60m
                     use_temp_path=off;

    server {
        listen 80;
        listen 443 ssl http2;
        server_name example.com;

        # SSL 설정
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

        # 압축
        gzip on;
        gzip_types text/plain text/css application/json application/javascript;
        gzip_min_length 1024;

        # 레이트 리미팅
        limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

        location /api/ {
            # 레이트 리미팅 적용
            limit_req zone=api_limit burst=20 nodelay;

            # 프록시 설정
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";

            # 헤더 전달
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 타임아웃
            proxy_connect_timeout 30s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            # 버퍼링
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }

        # 정적 파일 캐싱
        location /static/ {
            proxy_cache my_cache;
            proxy_cache_valid 200 1d;
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout updating;

            proxy_pass http://backend;

            add_header X-Cache-Status $upstream_cache_status;
        }

        # WebSocket 프록시
        location /ws/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 86400s;
        }
    }
}
```

### HAProxy 설정

```haproxy
# /etc/haproxy/haproxy.cfg

global
    maxconn 50000
    log /dev/log local0
    stats socket /var/run/haproxy.sock mode 660 level admin

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor except 127.0.0.0/8
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/haproxy/certs/example.pem

    # HTTPS 리다이렉트
    http-request redirect scheme https unless { ssl_fc }

    # ACL 기반 라우팅
    acl is_api path_beg /api/
    acl is_static path_beg /static/

    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend web_servers

backend api_servers
    balance leastconn
    option httpchk GET /health

    # 헤더 추가
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }

    # 서버 정의
    server api1 10.0.0.1:8080 check weight 100
    server api2 10.0.0.2:8080 check weight 100
    server api3 10.0.0.3:8080 check weight 100 backup

backend web_servers
    balance roundrobin
    cookie SERVERID insert indirect nocache

    server web1 10.0.1.1:8080 check cookie s1
    server web2 10.0.1.2:8080 check cookie s2

backend static_servers
    balance uri
    server static1 10.0.2.1:8080 check
    server static2 10.0.2.2:8080 check

# 통계 페이지
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
```

### API Gateway 패턴

```
API Gateway vs Reverse Proxy:

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Reverse Proxy (L7 로드밸런서)                             │
│   • 트래픽 분산                                             │
│   • SSL 종료                                                │
│   • 기본적인 라우팅                                         │
│                                                             │
│   API Gateway (+ 비즈니스 로직)                             │
│   • 인증/인가                                               │
│   • 요청/응답 변환                                          │
│   • API 버저닝                                              │
│   • 레이트 리미팅                                           │
│   • 요청 집계 (Aggregation)                                 │
│   • 프로토콜 변환 (REST ↔ gRPC)                             │
│   • API 분석/모니터링                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘

API Gateway 예시 (Kong, AWS API Gateway, Envoy):

┌────────┐     ┌─────────────────┐     ┌───────────────┐
│        │     │                 │     │ User Service  │
│        │     │   API Gateway   │────>│               │
│ Client │────>│                 │     └───────────────┘
│        │     │ • Auth          │
│        │     │ • Rate Limit    │────>┌───────────────┐
│        │     │ • Logging       │     │ Order Service │
│        │     │ • Transform     │     └───────────────┘
└────────┘     └─────────────────┘
                       │
                       └────────────>┌───────────────┐
                                     │ Product Svc   │
                                     └───────────────┘
```

---

## Transparent Proxy

### 개요

Transparent Proxy(투명 프록시)는 클라이언트가 프록시의 존재를 인식하지 못하는 상태에서 동작하는 프록시입니다.

```
Transparent Proxy 동작:

일반 프록시 (명시적 설정 필요):
┌────────┐      ┌─────────┐      ┌────────┐
│ Client │ ───> │  Proxy  │ ───> │ Server │
└────────┘      └─────────┘      └────────┘
  ↑
  프록시 설정 필요
  (브라우저/시스템 설정)

Transparent Proxy (설정 불필요):
┌────────┐      ┌─────────┐      ┌────────┐
│ Client │ ───> │  Proxy  │ ───> │ Server │
└────────┘      └─────────┘      └────────┘
  ↑                  ↑
  직접 연결 시도     네트워크 장비가
                    트래픽을 가로채서
                    프록시로 전달
```

### 동작 방식

```
Transparent Proxy 구현 방법:

1. 라우터/스위치 수준 리다이렉션
   - WCCP (Web Cache Communication Protocol)
   - Policy-based Routing

2. 방화벽 규칙 (iptables/nftables)
   - NAT 테이블에서 REDIRECT
   - DNAT으로 프록시 서버로 전송

3. DNS 기반
   - DNS 응답을 조작하여 프록시 IP 반환

iptables 예시:
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  # 포트 80 트래픽을 프록시(3128)로 리다이렉션               │
│  iptables -t nat -A PREROUTING -p tcp --dport 80 \         │
│           -j REDIRECT --to-port 3128                        │
│                                                             │
│  # 또는 다른 서버의 프록시로 DNAT                           │
│  iptables -t nat -A PREROUTING -p tcp --dport 80 \         │
│           -j DNAT --to-destination 192.168.1.100:3128       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Squid Transparent Proxy 설정

```squid
# /etc/squid/squid.conf

# Transparent 모드 설정
http_port 3128 transparent

# 또는 SSL Bump (HTTPS 검사) - 보안상 주의 필요
# https_port 3129 intercept ssl-bump \
#     cert=/etc/squid/ssl/squid.pem \
#     generate-host-certificates=on

# ACL 정의
acl localnet src 192.168.0.0/16
acl SSL_ports port 443
acl Safe_ports port 80 443 21 70 210 280 488 591 777 1025-65535

# 접근 제어
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localnet
http_access deny all

# 캐싱 설정
cache_dir ufs /var/spool/squid 10000 16 256
maximum_object_size 100 MB
cache_mem 256 MB

# 로깅
access_log /var/log/squid/access.log squid
cache_log /var/log/squid/cache.log

# 성능 튜닝
dns_nameservers 8.8.8.8 8.8.4.4
```

### Transparent vs Explicit vs Reverse Proxy

| 특성 | Transparent | Explicit | Reverse |
|------|-------------|----------|---------|
| **클라이언트 설정** | 불필요 | 필요 | 불필요 |
| **클라이언트 인식** | 못함 | 알고 있음 | 못함 |
| **위치** | 네트워크 경로 | 클라이언트 설정 | 서버 앞단 |
| **주요 용도** | ISP 캐싱, 필터링 | 기업 보안 | 로드밸런싱 |
| **HTTPS 처리** | 복잡 (SSL 검사 필요) | CONNECT 메서드 | SSL 종료 |

---

## VPN 프로토콜

### VPN 개요

VPN(Virtual Private Network)은 공용 네트워크를 통해 안전한 사설 네트워크 연결을 제공합니다.

```
VPN 구성 유형:

1. Remote Access VPN (원격 접속):
   ┌────────────┐     인터넷      ┌─────────────────────┐
   │ 원격 사용자 │ ══════════════> │   회사 VPN 서버     │
   │            │  암호화 터널     │   → 회사 네트워크   │
   └────────────┘                  └─────────────────────┘

2. Site-to-Site VPN (사이트 간):
   ┌─────────────┐     인터넷      ┌─────────────┐
   │ 본사 네트워크│ ══════════════> │ 지사 네트워크│
   │ VPN Gateway │  암호화 터널     │ VPN Gateway │
   └─────────────┘                  └─────────────┘

3. Client-to-Site VPN:
   개인 기기에서 회사 네트워크로 안전하게 접속
```

### IPsec

IPsec(Internet Protocol Security)은 네트워크 계층에서 IP 패킷을 암호화하고 인증하는 프로토콜 스위트입니다.

```
IPsec 구성 요소:

┌─────────────────────────────────────────────────────────────┐
│                       IPsec                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  프로토콜:                                                   │
│  ┌─────────────────────────┐  ┌─────────────────────────┐  │
│  │ AH (Authentication      │  │ ESP (Encapsulating      │  │
│  │     Header)             │  │      Security Payload)  │  │
│  │ • 인증만                 │  │ • 인증 + 암호화         │  │
│  │ • 무결성 보장            │  │ • 기밀성, 무결성 보장   │  │
│  └─────────────────────────┘  └─────────────────────────┘  │
│                                                             │
│  키 관리:                                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ IKE (Internet Key Exchange)                         │   │
│  │ • IKEv1 / IKEv2                                     │   │
│  │ • SA (Security Association) 협상                    │   │
│  │ • 키 교환 (Diffie-Hellman)                          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  모드:                                                       │
│  ┌───────────────────────┐  ┌───────────────────────────┐  │
│  │ Transport Mode        │  │ Tunnel Mode              │  │
│  │ • 호스트 간 통신       │  │ • 네트워크 간 통신        │  │
│  │ • 원본 IP 헤더 유지    │  │ • 새 IP 헤더 추가        │  │
│  └───────────────────────┘  └───────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```
IPsec Transport vs Tunnel Mode:

Transport Mode:
┌──────────┬──────────────┬─────────────┐
│ IP 헤더   │ ESP/AH 헤더  │ Payload     │
│ (원본)   │              │ (암호화)    │
└──────────┴──────────────┴─────────────┘
  └───────────────────────────────────┘
            인증 범위

Tunnel Mode:
┌──────────┬──────────────┬──────────┬─────────────┐
│ 새 IP    │ ESP/AH 헤더  │ 원본 IP  │ Payload     │
│ 헤더     │              │ 헤더     │             │
└──────────┴──────────────┴──────────┴─────────────┘
                          └────────────────────────┘
                                  암호화 범위
```

### OpenVPN

OpenVPN은 SSL/TLS 기반의 오픈소스 VPN 솔루션입니다.

```
OpenVPN 특징:

• SSL/TLS 기반 암호화
• TCP/UDP 모두 지원
• 다양한 인증 방식 (인증서, ID/PW, 2FA)
• 크로스 플랫폼 지원
• NAT/방화벽 친화적

OpenVPN 서버 설정 (/etc/openvpn/server.conf):
```

```conf
# /etc/openvpn/server.conf

# 네트워크 설정
port 1194
proto udp
dev tun

# 인증서 경로
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh2048.pem

# VPN 네트워크
server 10.8.0.0 255.255.255.0

# 클라이언트 라우팅
push "route 192.168.1.0 255.255.255.0"
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"

# 연결 설정
keepalive 10 120
cipher AES-256-GCM
auth SHA256
user nobody
group nogroup
persist-key
persist-tun

# 로깅
status /var/log/openvpn-status.log
log-append /var/log/openvpn.log
verb 3
```

```conf
# OpenVPN 클라이언트 설정 (client.ovpn)

client
dev tun
proto udp
remote vpn.example.com 1194
resolv-retry infinite
nobind
persist-key
persist-tun

ca ca.crt
cert client.crt
key client.key

remote-cert-tls server
cipher AES-256-GCM
auth SHA256
verb 3
```

### WireGuard

WireGuard는 최신 암호화를 사용하는 간단하고 빠른 VPN 프로토콜입니다.

```
WireGuard 특징:

┌─────────────────────────────────────────────────────────────┐
│                     WireGuard                               │
├─────────────────────────────────────────────────────────────┤
│ • 코드베이스: ~4,000줄 (OpenVPN: ~100,000줄)                │
│ • 암호화: ChaCha20, Poly1305, Curve25519, BLAKE2s          │
│ • 성능: OpenVPN 대비 약 3-4배 빠름                          │
│ • UDP만 사용 (더 빠르고 효율적)                             │
│ • 커널 레벨 구현 (Linux 5.6+)                               │
│ • Cryptokey Routing (공개키 기반 라우팅)                    │
└─────────────────────────────────────────────────────────────┘
```

```bash
# WireGuard 설치 (Ubuntu)
sudo apt install wireguard

# 키 생성
wg genkey | tee privatekey | wg pubkey > publickey
```

```ini
# /etc/wireguard/wg0.conf (서버)

[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY

# IP 포워딩 활성화
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# 클라이언트 1
PublicKey = CLIENT1_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32

[Peer]
# 클라이언트 2
PublicKey = CLIENT2_PUBLIC_KEY
AllowedIPs = 10.0.0.3/32
```

```ini
# /etc/wireguard/wg0.conf (클라이언트)

[Interface]
Address = 10.0.0.2/24
PrivateKey = CLIENT_PRIVATE_KEY
DNS = 8.8.8.8

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0    # 모든 트래픽 VPN으로
PersistentKeepalive = 25        # NAT 유지
```

```bash
# WireGuard 인터페이스 시작
sudo wg-quick up wg0

# 상태 확인
sudo wg show

# 인터페이스 중지
sudo wg-quick down wg0
```

### VPN 프로토콜 비교

| 특성 | IPsec/IKEv2 | OpenVPN | WireGuard |
|------|-------------|---------|-----------|
| **속도** | 빠름 | 보통 | 매우 빠름 |
| **보안** | 높음 | 높음 | 높음 |
| **코드 크기** | 매우 큼 | 큼 (~100K) | 작음 (~4K) |
| **설정 복잡도** | 복잡 | 중간 | 간단 |
| **프로토콜** | UDP/TCP | UDP/TCP | UDP만 |
| **방화벽 우회** | 어려움 | 쉬움 (443포트) | 중간 |
| **모바일 지원** | 내장 (iOS/Android) | 앱 필요 | 앱 필요 |
| **기업 환경** | 최적 | 좋음 | 신흥 |

---

## 터널링

### 터널링 개요

터널링은 한 프로토콜의 패킷을 다른 프로토콜로 캡슐화하여 전송하는 기술입니다.

```
터널링 개념:

원본 패킷:
┌──────────────┬─────────────────────────┐
│ Original     │ Original                │
│ Header       │ Payload                 │
└──────────────┴─────────────────────────┘

캡슐화된 패킷:
┌──────────────┬──────────────┬─────────────────────────┐
│ Tunnel       │ Original     │ Original                │
│ Header       │ Header       │ Payload                 │
└──────────────┴──────────────┴─────────────────────────┘
  외부 전송용       원본 패킷 (암호화 가능)
```

### 주요 터널링 프로토콜

#### GRE (Generic Routing Encapsulation)

```
GRE 터널:

┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Router A                                 Router B         │
│  ┌───────┐       GRE Tunnel              ┌───────┐        │
│  │ eth0  │◄═══════════════════════════►  │ eth0  │        │
│  │10.0.1.1│       (IP-in-IP)             │10.0.2.1│        │
│  └───────┘                               └───────┘        │
│      │                                       │             │
│  ┌───────┐                               ┌───────┐        │
│  │Network│                               │Network│        │
│  │192.168.1.0/24                         │192.168.2.0/24  │
│  └───────┘                               └───────┘        │
│                                                            │
└────────────────────────────────────────────────────────────┘

Linux GRE 설정:
```

```bash
# Router A에서 GRE 터널 생성
sudo ip tunnel add gre1 mode gre remote 10.0.2.1 local 10.0.1.1 ttl 255
sudo ip link set gre1 up
sudo ip addr add 172.16.1.1/30 dev gre1
sudo ip route add 192.168.2.0/24 via 172.16.1.2

# Router B에서 GRE 터널 생성
sudo ip tunnel add gre1 mode gre remote 10.0.1.1 local 10.0.2.1 ttl 255
sudo ip link set gre1 up
sudo ip addr add 172.16.1.2/30 dev gre1
sudo ip route add 192.168.1.0/24 via 172.16.1.1
```

#### SSH 터널링

```bash
# 1. Local Port Forwarding
# 로컬 8080 → SSH 서버 → 원격 서버 80
ssh -L 8080:remote-server:80 user@ssh-server

# 로컬에서 접근: http://localhost:8080

# 2. Remote Port Forwarding
# 원격 8080 → SSH 서버 → 로컬 80
ssh -R 8080:localhost:80 user@ssh-server

# 외부에서 접근: http://ssh-server:8080

# 3. Dynamic Port Forwarding (SOCKS Proxy)
ssh -D 1080 user@ssh-server

# 브라우저에서 SOCKS5 프록시로 localhost:1080 설정

# 4. 백그라운드 실행
ssh -f -N -L 8080:remote-server:80 user@ssh-server
# -f: 백그라운드
# -N: 원격 명령 실행 안함
```

```
SSH 터널링 시나리오:

1. Local Port Forwarding:
   ┌─────────┐      ┌────────────┐      ┌─────────────┐
   │ Local   │ SSH  │ SSH Server │      │ DB Server   │
   │ :8080   │─────>│            │─────>│ :5432       │
   └─────────┘      └────────────┘      └─────────────┘
   방화벽 뒤의 DB에 안전하게 접근

2. Remote Port Forwarding:
   ┌─────────┐      ┌────────────┐      ┌─────────────┐
   │ Local   │ SSH  │ SSH Server │      │ 외부 사용자  │
   │ :3000   │─────>│ :8080      │<─────│             │
   └─────────┘      └────────────┘      └─────────────┘
   로컬 개발 서버를 외부에 노출

3. SOCKS Proxy:
   ┌─────────┐      ┌────────────┐      ┌─────────────┐
   │ Local   │ SSH  │ SSH Server │      │ Any Server  │
   │ :1080   │─────>│            │─────>│ anywhere    │
   └─────────┘      └────────────┘      └─────────────┘
   모든 트래픽을 SSH 서버를 통해 전송
```

#### SSL/TLS 터널링

```python
# Python에서 SSL 소켓 터널링
import socket
import ssl

def ssl_tunnel():
    # SSL 컨텍스트 생성
    context = ssl.create_default_context()

    # 일반 소켓 생성
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

    # SSL로 래핑
    ssl_sock = context.wrap_socket(sock, server_hostname='example.com')

    # 연결
    ssl_sock.connect(('example.com', 443))

    # HTTPS 요청
    ssl_sock.send(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")

    # 응답 수신
    response = ssl_sock.recv(4096)
    print(response.decode())

    ssl_sock.close()
```

#### stunnel (SSL 터널링)

```ini
# /etc/stunnel/stunnel.conf

# 클라이언트 모드 - 평문 트래픽을 SSL로 암호화
[mysql-client]
client = yes
accept = 127.0.0.1:3306
connect = remote-db:13306

# 서버 모드 - SSL 트래픽을 평문으로 복호화
[mysql-server]
accept = 13306
connect = 127.0.0.1:3306
cert = /etc/stunnel/stunnel.pem
```

---

## 실무 적용

### 프록시 선택 가이드

```
사용 사례별 추천:

1. 기업 웹 필터링/캐싱
   → Squid, Blue Coat

2. 로드 밸런싱
   → Nginx, HAProxy, Envoy

3. API Gateway
   → Kong, AWS API Gateway, Envoy

4. 개인 프라이버시
   → SOCKS5 프록시, VPN

5. 개발/디버깅
   → Charles Proxy, Fiddler, mitmproxy
```

### VPN 선택 가이드

```
사용 사례별 추천:

1. 기업 환경 (Site-to-Site)
   → IPsec/IKEv2

2. 원격 근무자
   → WireGuard, OpenVPN

3. 최대 성능 필요
   → WireGuard

4. 방화벽 우회 필요
   → OpenVPN (TCP 443)

5. 기존 인프라 통합
   → IPsec (대부분의 네트워크 장비 지원)
```

### 보안 모범 사례

```
프록시 보안:

1. 인증 적용
   - 프록시 접근에 인증 요구
   - 클라이언트 인증서 사용

2. 로깅 및 모니터링
   - 모든 요청 로깅
   - 이상 트래픽 감지

3. 접근 제어
   - 화이트리스트/블랙리스트 관리
   - IP 기반 접근 제한

4. SSL/TLS 사용
   - 프록시 통신 암호화
   - 인증서 검증

VPN 보안:

1. 강력한 암호화
   - AES-256, ChaCha20
   - PFS (Perfect Forward Secrecy)

2. 인증서 관리
   - 정기적인 인증서 갱신
   - 인증서 폐기 목록 관리

3. 다단계 인증
   - 인증서 + 비밀번호 + OTP

4. 네트워크 분리
   - VPN 사용자 네트워크 세그먼트 분리
   - 최소 권한 원칙 적용
```

### 모니터링 및 로깅

```bash
# Nginx 프록시 로그 분석
tail -f /var/log/nginx/access.log | \
    awk '{print $1, $4, $7, $9}'

# HAProxy 상태 확인
echo "show stat" | socat stdio /var/run/haproxy.sock

# WireGuard 상태
sudo wg show

# OpenVPN 연결 상태
sudo cat /var/log/openvpn-status.log

# IPsec SA 확인
sudo ipsec statusall
```

---

## 참고 자료

- [RFC 7230 - HTTP/1.1 Message Syntax and Routing](https://datatracker.ietf.org/doc/html/rfc7230)
- [RFC 4301 - Security Architecture for IP](https://datatracker.ietf.org/doc/html/rfc4301) (IPsec)
- [WireGuard White Paper](https://www.wireguard.com/papers/wireguard.pdf)
- [OpenVPN Documentation](https://openvpn.net/community-resources/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [HAProxy Documentation](https://www.haproxy.org/download/2.6/doc/configuration.txt)
- [Squid Proxy Documentation](http://www.squid-cache.org/Doc/)
- [IVPN - VPN Protocol Comparison](https://www.ivpn.net/pptp-vs-ipsec-ikev2-vs-openvpn-vs-wireguard/)

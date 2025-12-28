# 리버스 프록시

## 목차
1. [프록시 개념](#프록시-개념)
2. [프록시 vs 리버스 프록시](#프록시-vs-리버스-프록시)
3. [헤더 처리](#헤더-처리)
4. [SSL Termination](#ssl-termination)
5. [캐싱 프록시](#캐싱-프록시)
6. [실전 설정 예제](#실전-설정-예제)
---

## 프록시 개념

**프록시(Proxy)**는 클라이언트와 서버 사이에서 중계 역할을 하는 서버입니다. "대리인"이라는 의미처럼, 요청과 응답을 대신 전달합니다.

### 프록시의 역할

```
┌──────────────────────────────────────────────────────────────┐
│                     Proxy의 주요 기능                         │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. 보안 (Security)                                           │
│     - 클라이언트/서버 IP 은닉                                  │
│     - 악성 트래픽 필터링                                       │
│     - SSL/TLS 처리                                            │
│                                                               │
│  2. 성능 (Performance)                                        │
│     - 캐싱으로 응답 속도 향상                                  │
│     - 압축으로 대역폭 절약                                     │
│     - 로드 밸런싱                                              │
│                                                               │
│  3. 접근 제어 (Access Control)                                │
│     - 콘텐츠 필터링                                           │
│     - 인증/인가                                               │
│     - Rate Limiting                                           │
│                                                               │
│  4. 로깅 및 모니터링                                           │
│     - 트래픽 분석                                              │
│     - 감사 로그                                                │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 프록시 vs 리버스 프록시

### Forward Proxy (정방향 프록시)

**클라이언트 측**에서 동작하며, 클라이언트를 대신하여 외부 서버에 요청합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Forward Proxy (정방향 프록시)                  │
│                                                                  │
│   ┌────────┐                              ┌────────────────┐    │
│   │Client A│──┐                     ┌────▶│ Web Server 1   │    │
│   └────────┘  │    ┌────────────┐   │     └────────────────┘    │
│               │    │            │   │                           │
│   ┌────────┐  ├───▶│  Forward   │───┤     ┌────────────────┐    │
│   │Client B│──┤    │   Proxy    │   ├────▶│ Web Server 2   │    │
│   └────────┘  │    │            │   │     └────────────────┘    │
│               │    └────────────┘   │                           │
│   ┌────────┐  │                     │     ┌────────────────┐    │
│   │Client C│──┘                     └────▶│ Web Server 3   │    │
│   └────────┘                              └────────────────┘    │
│                                                                  │
│   [내부 네트워크]         [프록시]          [외부 인터넷]          │
└─────────────────────────────────────────────────────────────────┘
```

#### 사용 사례
- **기업 네트워크**: 직원들의 인터넷 접근 제어
- **익명성 보장**: 클라이언트 IP 숨기기
- **캐싱**: 자주 접근하는 콘텐츠 캐싱
- **접근 제한**: 특정 사이트 차단

#### 대표적인 Forward Proxy
- **Squid**: 기업용 캐싱 프록시
- **Privoxy**: 개인정보 보호 프록시
- **Charles/Fiddler**: 개발용 디버깅 프록시

### Reverse Proxy (리버스 프록시)

**서버 측**에서 동작하며, 서버를 대신하여 클라이언트 요청을 처리합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Reverse Proxy (리버스 프록시)                  │
│                                                                  │
│   ┌────────┐                              ┌────────────────┐    │
│   │Client A│──┐                     ┌────▶│ Backend 1      │    │
│   └────────┘  │    ┌────────────┐   │     │ (App Server)   │    │
│               │    │            │   │     └────────────────┘    │
│   ┌────────┐  ├───▶│  Reverse   │───┤                           │
│   │Client B│──┤    │   Proxy    │   │     ┌────────────────┐    │
│   └────────┘  │    │            │   ├────▶│ Backend 2      │    │
│               │    └────────────┘   │     │ (App Server)   │    │
│   ┌────────┐  │                     │     └────────────────┘    │
│   │Client C│──┘                     │                           │
│   └────────┘                        │     ┌────────────────┐    │
│                                     └────▶│ Backend 3      │    │
│                                           │ (App Server)   │    │
│   [외부 인터넷]         [프록시]          └────────────────┘    │
│                                           [내부 네트워크]        │
└─────────────────────────────────────────────────────────────────┘
```

#### 사용 사례
- **로드 밸런싱**: 트래픽 분산
- **SSL Termination**: HTTPS 처리
- **캐싱**: 정적 콘텐츠 캐싱
- **보안**: 백엔드 서버 보호, WAF
- **압축**: Gzip 압축

#### 대표적인 Reverse Proxy
- **Nginx**: 가장 널리 사용
- **HAProxy**: 고성능 로드 밸런서
- **Traefik**: 클라우드 네이티브
- **Envoy**: 서비스 메시

### 비교 표

| 특성 | Forward Proxy | Reverse Proxy |
|------|--------------|---------------|
| 위치 | 클라이언트 측 | 서버 측 |
| 보호 대상 | 클라이언트 | 서버 |
| IP 은닉 | 클라이언트 IP | 서버 IP |
| 설정 주체 | 클라이언트/조직 | 서버 운영자 |
| 대표 사례 | 기업 프록시, VPN | Nginx, HAProxy |

---

## 헤더 처리

리버스 프록시를 거치면 **원본 클라이언트 정보가 손실**됩니다. 이를 보존하기 위해 특수 헤더를 사용합니다.

### X-Forwarded-For (XFF)

클라이언트의 **원본 IP 주소**를 전달합니다.

```
[단일 프록시]
Client (203.0.113.50) → Proxy (192.168.1.1) → Backend

X-Forwarded-For: 203.0.113.50

[다중 프록시]
Client → Proxy1 → Proxy2 → Backend

X-Forwarded-For: 203.0.113.50, 192.168.1.1
                 └── 원본    └── 첫 번째 프록시
```

#### Nginx 설정

```nginx
server {
    location / {
        proxy_pass http://backend;

        # 클라이언트 IP를 XFF 헤더에 추가
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # $proxy_add_x_forwarded_for는 기존 XFF에 $remote_addr을 추가
        # 기존 XFF가 없으면: $remote_addr만
        # 기존 XFF가 있으면: $existing_xff, $remote_addr
    }
}
```

### X-Real-IP

**첫 번째 클라이언트의 IP**만 전달합니다 (체인 아님).

```nginx
server {
    location / {
        proxy_pass http://backend;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### X-Forwarded-Proto

원본 요청의 **프로토콜(HTTP/HTTPS)**을 전달합니다.

```nginx
server {
    listen 443 ssl;

    location / {
        proxy_pass http://backend;
        proxy_set_header X-Forwarded-Proto $scheme;  # https
    }
}
```

### X-Forwarded-Host

원본 요청의 **Host 헤더**를 전달합니다.

```nginx
server {
    location / {
        proxy_pass http://backend;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header Host $host;  # 원본 Host 유지
    }
}
```

### 완전한 헤더 설정 예제

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    location / {
        proxy_pass http://backend;

        # 표준 프록시 헤더
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Port $server_port;

        # 연결 설정
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # keepalive 활성화

        # 타임아웃
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

### 백엔드에서 실제 클라이언트 IP 얻기

#### Node.js (Express)

```javascript
const express = require('express');
const app = express();

// trust proxy 설정 (프록시 뒤에 있을 때)
app.set('trust proxy', true);

app.get('/', (req, res) => {
    // req.ip는 X-Forwarded-For의 첫 번째 IP를 반환
    const clientIP = req.ip;

    // 또는 직접 헤더 접근
    const xff = req.headers['x-forwarded-for'];
    const realIP = req.headers['x-real-ip'];

    res.json({ clientIP, xff, realIP });
});
```

#### Spring Boot (Java)

```java
@RestController
public class ClientIPController {

    @GetMapping("/ip")
    public String getClientIP(HttpServletRequest request) {
        String xff = request.getHeader("X-Forwarded-For");
        if (xff != null && !xff.isEmpty()) {
            // 첫 번째 IP가 원본 클라이언트
            return xff.split(",")[0].trim();
        }

        String realIP = request.getHeader("X-Real-IP");
        if (realIP != null && !realIP.isEmpty()) {
            return realIP;
        }

        return request.getRemoteAddr();
    }
}
```

### 보안 고려사항

> **경고**: X-Forwarded-For는 **클라이언트가 조작할 수 있습니다!**

```
# 악의적인 클라이언트가 위조된 XFF 전송
Client → Proxy: X-Forwarded-For: 10.0.0.1 (위조)

# 프록시가 추가
Proxy → Backend: X-Forwarded-For: 10.0.0.1, 203.0.113.50
                                  └── 위조    └── 실제
```

#### 해결책: 신뢰할 수 있는 프록시만 허용

```nginx
# 내부 프록시 IP만 신뢰
set_real_ip_from 10.0.0.0/8;
set_real_ip_from 192.168.0.0/16;
set_real_ip_from 172.16.0.0/12;
real_ip_header X-Forwarded-For;
real_ip_recursive on;  # 가장 오른쪽의 신뢰할 수 없는 IP 사용
```

```nginx
# 또는 신뢰할 수 없는 XFF 제거
map $http_x_forwarded_for $client_ip {
    default $remote_addr;
}

# 새로운 XFF 시작
proxy_set_header X-Forwarded-For $remote_addr;
```

---

## SSL Termination

**SSL Termination**은 리버스 프록시에서 HTTPS를 처리하고, 백엔드에는 HTTP로 통신하는 방식입니다.

### 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                      SSL Termination                             │
│                                                                  │
│   ┌────────┐   HTTPS    ┌────────────┐   HTTP    ┌──────────┐  │
│   │        │ ──────────▶│            │──────────▶│          │  │
│   │ Client │            │  Reverse   │           │ Backend  │  │
│   │        │◀────────── │   Proxy    │◀──────────│          │  │
│   └────────┘   HTTPS    │ (SSL Term) │   HTTP    └──────────┘  │
│                         └────────────┘                          │
│                               │                                 │
│                         ┌─────┴─────┐                          │
│                         │ SSL 인증서 │                          │
│                         │ 개인키     │                          │
│                         └───────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

### 장점

| 장점 | 설명 |
|------|------|
| **성능** | 백엔드 서버의 CPU 부하 감소 |
| **관리 용이** | 인증서를 한 곳에서만 관리 |
| **확장성** | 백엔드 스케일아웃이 쉬움 |
| **비용 절감** | 내부 통신에 인증서 불필요 |

### 단점 및 고려사항

| 단점 | 해결책 |
|------|--------|
| 내부 통신 평문 | 내부 네트워크 보안 강화, 또는 End-to-End TLS |
| 프록시가 단일 실패점 | 프록시 이중화 |
| 컴플라이언스 이슈 | 규정에 따라 End-to-End 암호화 필요 |

### Nginx SSL Termination 설정

```nginx
# HTTP → HTTPS 리다이렉트
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}

# HTTPS 서버 (SSL Termination)
server {
    listen 443 ssl http2;
    server_name example.com;

    # SSL 인증서
    ssl_certificate /etc/ssl/certs/fullchain.pem;
    ssl_certificate_key /etc/ssl/private/privkey.pem;

    # SSL 설정
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
        # 백엔드는 HTTP
        proxy_pass http://backend;

        # 프로토콜 정보 전달 (백엔드에서 HTTPS 여부 확인용)
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Ssl on;

        # 기타 헤더
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### End-to-End TLS (SSL Passthrough)

백엔드까지 HTTPS를 유지해야 하는 경우:

```nginx
# SSL Passthrough (L4 프록시)
stream {
    upstream backend_ssl {
        server 192.168.1.10:443;
        server 192.168.1.11:443;
    }

    server {
        listen 443;
        proxy_pass backend_ssl;
        proxy_protocol on;  # 클라이언트 IP 전달
    }
}
```

### SSL Re-encryption

프록시에서 복호화 후 백엔드에 다시 암호화:

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/frontend.pem;
    ssl_certificate_key /etc/ssl/private/frontend.key;

    location / {
        # 백엔드도 HTTPS
        proxy_pass https://backend;

        # 백엔드 인증서 검증
        proxy_ssl_verify on;
        proxy_ssl_trusted_certificate /etc/ssl/certs/backend-ca.pem;
        proxy_ssl_certificate /etc/ssl/certs/client.pem;
        proxy_ssl_certificate_key /etc/ssl/private/client.key;
    }
}
```

---

## 캐싱 프록시

리버스 프록시에서 **응답을 캐시**하여 백엔드 부하를 줄이고 응답 속도를 높입니다.

### Nginx 프록시 캐시

#### 기본 설정

```nginx
http {
    # 캐시 영역 정의
    proxy_cache_path /var/cache/nginx
                     levels=1:2                    # 디렉토리 구조 (예: /a/bc/)
                     keys_zone=my_cache:10m        # 메타데이터 저장소 (10MB, ~80,000 키)
                     max_size=10g                  # 최대 캐시 크기
                     inactive=60m                  # 60분 동안 미사용 시 삭제
                     use_temp_path=off;            # 임시 파일 없이 직접 저장

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend;

            # 캐시 사용
            proxy_cache my_cache;

            # 캐시 키 정의
            proxy_cache_key $scheme$proxy_host$request_uri;

            # 캐시 유효 시간
            proxy_cache_valid 200 302 10m;  # 200, 302 응답은 10분
            proxy_cache_valid 404 1m;       # 404 응답은 1분
            proxy_cache_valid any 5m;       # 나머지는 5분

            # 캐시 상태 헤더 추가 (디버깅용)
            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

#### 캐시 상태 값 ($upstream_cache_status)

| 상태 | 의미 |
|------|------|
| `HIT` | 캐시에서 제공 |
| `MISS` | 캐시 없음, 백엔드에서 가져옴 |
| `EXPIRED` | 캐시 만료, 백엔드에서 갱신 |
| `STALE` | 만료된 캐시 제공 (백엔드 오류 시) |
| `UPDATING` | 갱신 중, 이전 캐시 제공 |
| `REVALIDATED` | 조건부 요청으로 캐시 확인 |
| `BYPASS` | 캐시 우회 |

### 고급 캐시 설정

```nginx
http {
    proxy_cache_path /var/cache/nginx
                     levels=1:2
                     keys_zone=my_cache:10m
                     max_size=10g
                     inactive=60m
                     use_temp_path=off;

    server {
        location /api/ {
            proxy_pass http://backend;
            proxy_cache my_cache;

            # 캐시 키 (쿼리 파라미터 포함)
            proxy_cache_key $scheme$proxy_host$uri$is_args$args;

            # 캐시 잠금 (동시 요청 시 하나만 백엔드 접근)
            proxy_cache_lock on;
            proxy_cache_lock_timeout 5s;

            # Stale 캐시 사용 조건 (백엔드 오류 시)
            proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;

            # 백그라운드 갱신 (사용자는 이전 캐시 응답 받음)
            proxy_cache_background_update on;

            # 최소 사용 횟수 후 캐시 (불필요한 캐시 방지)
            proxy_cache_min_uses 2;

            # 캐시 우회 조건
            proxy_cache_bypass $http_cache_control;
            proxy_no_cache $http_pragma $http_authorization;

            # 캐시 유효 시간
            proxy_cache_valid 200 10m;
            proxy_cache_valid 404 1m;

            add_header X-Cache-Status $upstream_cache_status;
        }

        # 정적 파일 (긴 캐시)
        location /static/ {
            proxy_pass http://backend;
            proxy_cache my_cache;
            proxy_cache_valid 200 1d;
            proxy_cache_valid 404 1m;

            add_header X-Cache-Status $upstream_cache_status;
        }

        # 캐시하지 않을 경로
        location /api/user/ {
            proxy_pass http://backend;
            proxy_no_cache 1;
            proxy_cache_bypass 1;
        }
    }
}
```

### FastCGI 캐시 (PHP 등)

```nginx
http {
    # FastCGI 캐시 영역
    fastcgi_cache_path /var/cache/nginx/fastcgi
                       levels=1:2
                       keys_zone=fastcgi_cache:10m
                       max_size=5g
                       inactive=60m;

    server {
        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
            include fastcgi_params;

            # 캐시 설정
            fastcgi_cache fastcgi_cache;
            fastcgi_cache_key $scheme$request_method$host$request_uri;
            fastcgi_cache_valid 200 10m;
            fastcgi_cache_valid 404 1m;

            # 로그인 사용자는 캐시 우회
            fastcgi_cache_bypass $cookie_logged_in;
            fastcgi_no_cache $cookie_logged_in;

            add_header X-Cache-Status $upstream_cache_status;
        }
    }
}
```

### 캐시 퍼지 (삭제)

Nginx OSS는 기본적으로 캐시 퍼지를 지원하지 않습니다.

#### 방법 1: 캐시 파일 직접 삭제

```bash
# 전체 캐시 삭제
rm -rf /var/cache/nginx/*

# Nginx 재시작 (메모리 캐시 정리)
nginx -s reload
```

#### 방법 2: ngx_cache_purge 모듈 (서드파티)

```nginx
location ~ /purge(/.*) {
    allow 127.0.0.1;
    allow 192.168.1.0/24;
    deny all;
    proxy_cache_purge my_cache $scheme$proxy_host$1$is_args$args;
}
```

```bash
# 특정 URL 캐시 삭제
curl -X PURGE http://example.com/purge/api/products
```

### Nginx vs Varnish 캐시 비교

| 특성 | Nginx | Varnish |
|------|-------|---------|
| 주 용도 | 웹 서버 + 캐시 | 전용 캐시 가속기 |
| 설정 언어 | nginx.conf | VCL (Varnish Configuration Language) |
| SSL 지원 | 네이티브 지원 | 별도 프록시 필요 (Hitch) |
| 성능 | 높음 | 매우 높음 (캐시 특화) |
| 캐시 퍼지 | 제한적 (모듈 필요) | 내장 지원 |
| 유연성 | 중간 | 높음 (VCL로 세밀한 제어) |
| 메모리 사용 | 효율적 | 적극적인 메모리 사용 |
| 학습 곡선 | 완만함 | VCL 학습 필요 |

#### 권장 사용 사례

```
Nginx 캐시 권장:
├─ 이미 Nginx를 웹 서버로 사용 중
├─ 간단한 캐싱 요구사항
├─ SSL Termination과 캐시를 함께 처리
└─ 리소스가 제한된 환경

Varnish 권장:
├─ 복잡한 캐시 로직 필요
├─ 매우 높은 트래픽
├─ Edge Side Includes (ESI) 필요
└─ 세밀한 캐시 제어 필요
```

---

## 실전 설정 예제

### 마이크로서비스 API 게이트웨이

```nginx
upstream user_service {
    server 192.168.1.10:8001;
    server 192.168.1.11:8001;
    keepalive 32;
}

upstream order_service {
    server 192.168.1.20:8002;
    server 192.168.1.21:8002;
    keepalive 32;
}

upstream product_service {
    server 192.168.1.30:8003;
    server 192.168.1.31:8003;
    keepalive 32;
}

# 캐시 설정
proxy_cache_path /var/cache/nginx/api
                 levels=1:2
                 keys_zone=api_cache:10m
                 max_size=1g
                 inactive=10m;

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/ssl/certs/api.example.com.pem;
    ssl_certificate_key /etc/ssl/private/api.example.com.key;

    # 공통 프록시 설정
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 사용자 서비스
    location /api/users {
        proxy_pass http://user_service;
        proxy_no_cache 1;  # 사용자 정보는 캐시하지 않음
    }

    # 주문 서비스
    location /api/orders {
        proxy_pass http://order_service;
        proxy_no_cache 1;
    }

    # 상품 서비스 (캐시 적용)
    location /api/products {
        proxy_pass http://product_service;
        proxy_cache api_cache;
        proxy_cache_valid 200 5m;
        proxy_cache_use_stale error timeout updating;
        add_header X-Cache-Status $upstream_cache_status;
    }

    # 헬스 체크 엔드포인트
    location /health {
        return 200 "OK";
        add_header Content-Type text/plain;
    }
}
```

### 웹소켓 지원

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    # 일반 HTTP 요청
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
    }

    # 웹소켓 (Socket.IO, WS 등)
    location /socket.io/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # 웹소켓 타임아웃 (기본 60초보다 길게)
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

---

## 참고 자료

- [Nginx Reverse Proxy Documentation](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Nginx Caching Documentation](https://docs.nginx.com/nginx/admin-guide/content-cache/content-caching/)
- [X-Forwarded-For - Wikipedia](https://en.wikipedia.org/wiki/X-Forwarded-For)
- [DigitalOcean - SSL Termination](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-load-balancing-with-ssl-termination)
- [Nginx vs Varnish - KeyCDN](https://www.keycdn.com/support/varnish-vs-nginx)

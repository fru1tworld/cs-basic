# Nginx

## Nginx 개요

Nginx(발음: "engine-x")는 동시에 10,000개의 연결을 효율적으로 처리하는 C10K 문제를 해결하기 위해 Igor Sysoev가 2002년에 개발한 웹 서버다. 요청을 직접 처리하는 웹 서버뿐 아니라 백엔드 앞에서 동작하는 리버스 프록시, 로드 밸런서, HTTP 캐시로도 사용한다.

### 주요 특징

- 고성능: 적은 메모리로 수만 개의 동시 연결 처리
- 이벤트 기반: Non-blocking I/O 모델
- 확장성: 모듈 기반 아키텍처
- 안정성: 무중단 설정 리로드 지원

## 아키텍처

### Event-driven, Non-blocking 아키텍처

- Apache와 같은 전통적인 웹 서버가 프로세스/스레드 기반 모델(연결당 하나의 프로세스 또는 스레드)을 사용하는 것과 달리, Nginx는 비동기 이벤트 기반 모델을 사용함.

Master 프로세스는 설정을 읽고 검증하며, 포트를 바인딩하고 Worker를 관리한다. reload나 stop 같은 시그널도 Master가 처리한다. 실제 연결은 여러 Worker가 나누어 맡고, 각 Worker의 이벤트 루프가 여러 연결의 작업을 진행한다.

### Master-Worker 프로세스 모델

#### Master Process

- root 권한으로 실행
- 설정 파일 읽기 및 유효성 검사
- Worker 프로세스 생성 및 관리
- 무중단 설정 리로드 처리
- 로그 파일 재오픈

#### Worker Process

- 실제 클라이언트 요청 처리
- 각 Worker는 단일 스레드로 동작
- 이벤트 루프를 통해 수천 개의 연결을 동시 처리
- Non-blocking I/O 사용으로 문맥 교환 최소화

### 이벤트 루프 동작 방식

새 연결이 들어오면 이벤트로 등록하고 요청을 처리한다. I/O를 기다리는 동안에는 다른 연결의 이벤트를 처리하며, 해당 연결에서 작업을 진행할 수 있게 되면 처리를 재개한다.

- epoll (Linux), kqueue (BSD/macOS), IOCP (Windows) 등 OS 레벨 I/O 멀티플렉싱 사용

### Apache vs Nginx 처리 방식 비교

- 모델:
  - Apache (Prefork): 프로세스 기반
  - Nginx: 이벤트 기반
- 연결당 리소스:
  - Apache (Prefork): 프로세스 1개 (~10MB)
  - Nginx: 이벤트 핸들러 (~수KB)
- 10,000 연결 시 메모리:
  - Apache (Prefork): ~100GB
  - Nginx: ~수십MB
- 문맥 교환:
  - Apache (Prefork): 빈번함
  - Nginx: 최소화
- 정적 파일 처리:
  - Apache (Prefork): 상대적으로 느림
  - Nginx: 매우 빠름

## 설정 문법

### 설정 파일 구조

```nginx
# 메인 컨텍스트 (전역 설정)
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# events 컨텍스트
events {
    worker_connections 1024;
    use epoll;
}

# http 컨텍스트
http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # server 블록
    server {
        listen 80;
        server_name example.com;

        # location 블록
        location / {
            root /var/www/html;
            index index.html;
        }
    }
}
```

### server 블록

`server` 블록은 가상 호스트를 정의한다. 포트와 도메인별로 블록을 나누면 하나의 Nginx 인스턴스에서 여러 사이트를 처리할 수 있다. 아래 예제는 HTTP, HTTPS, HTTPS로의 리다이렉트를 각각 보여 준다.

```nginx
# HTTP 서버
server {
    listen 80;                          # 포트 설정
    listen [::]:80;                     # IPv6
    server_name example.com www.example.com;  # 도메인

    root /var/www/example.com;          # 문서 루트
    index index.html index.htm;         # 기본 인덱스 파일

    # 로그 설정
    access_log /var/log/nginx/example.access.log;
    error_log /var/log/nginx/example.error.log;
}

# HTTPS 서버
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.com.crt;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # ... 추가 설정
}

# HTTP를 HTTPS로 리다이렉트
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}
```

### location 블록

`server`가 요청을 받을 사이트를 정한다면, `location`은 그 안에서 URI에 따라 처리 방법을 선택한다. 아래 예제에서는 정확히 일치하는 경로, 접두사, 정규식으로 요청을 구분한다.

```nginx
server {
    listen 80;
    server_name example.com;

    # 정확히 매칭 (highest priority)
    location = / {
        # example.com/ 만 매칭
        return 200 "Exact match for root";
    }

    # 우선순위 prefix 매칭
    location ^~ /images/ {
        # /images/로 시작하는 URI (정규식보다 우선)
        root /var/www;
    }

    # 정규식 매칭 (대소문자 구분)
    location ~ \.php$ {
        # .php로 끝나는 URI
        fastcgi_pass unix:/var/run/php/php8.0-fpm.sock;
        include fastcgi_params;
    }

    # 정규식 매칭 (대소문자 무시)
    location ~* \.(jpg|jpeg|png|gif|ico)$ {
        # 이미지 파일
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 일반 prefix 매칭
    location /api/ {
        # /api/로 시작하는 URI
        proxy_pass http://backend;
    }

    # 기본 매칭 (lowest priority)
    location / {
        # 위의 어느 것도 매칭되지 않을 때
        try_files $uri $uri/ =404;
    }
}
```

#### Location 매칭 우선순위

- 1:
  - 수식자: `=`
  - 설명: 정확히 일치
  - 예시: `location = /favicon.ico`
- 2:
  - 수식자: `^~`
  - 설명: 우선 prefix
  - 예시: `location ^~ /images/`
- 3:
  - 수식자: `~`
  - 설명: 정규식 (대소문자 구분)
  - 예시: `location ~ \.php$`
- 4:
  - 수식자: `~*`
  - 설명: 정규식 (대소문자 무시)
  - 예시: `location ~* \.(jpg\|png)$`
- 5:
  - 수식자: 없음
  - 설명: 일반 prefix
  - 예시: `location /api/`

### upstream 블록

백엔드가 여러 대라면 `upstream`으로 그룹을 정의하고 `proxy_pass`에서 참조한다. 다음 설정들은 가중치, 연결 수, 클라이언트 IP에 따라 서버를 선택하는 방법을 비교한다.

```nginx
# 기본 upstream 설정
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

# 가중치 기반 로드 밸런싱
upstream backend_weighted {
    server 192.168.1.10:8080 weight=5;   # 50%의 트래픽
    server 192.168.1.11:8080 weight=3;   # 30%의 트래픽
    server 192.168.1.12:8080 weight=2;   # 20%의 트래픽
}

# Least Connections 알고리즘
upstream backend_lc {
    least_conn;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

# IP Hash (세션 유지)
upstream backend_hash {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

# 헬스체크 및 백업 서버
upstream backend_advanced {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 backup;  # 위 서버들 장애 시에만 사용
    server 192.168.1.13:8080 down;    # 일시적으로 비활성화

    keepalive 32;  # 연결 풀 유지
}

server {
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # keepalive 활성화
    }
}
```

## 성능 튜닝

### worker_processes

- Worker 프로세스 수를 설정함. 일반적으로 CPU 코어 수와 동일하게 설정함.

```nginx
# 자동 감지 (권장)
worker_processes auto;

# 수동 설정
worker_processes 4;

# CPU 코어별 Worker 할당 (CPU affinity)
worker_processes 4;
worker_cpu_affinity 0001 0010 0100 1000;  # 각 Worker를 특정 코어에 바인딩

# 자동 CPU affinity (권장)
worker_cpu_affinity auto;
```

#### 컨테이너 환경 주의사항

```nginx
# Docker/Kubernetes 환경에서는 auto가 호스트 코어 수를 감지할 수 있음
# 컨테이너에 할당된 CPU 수에 맞게 명시적으로 설정

# 예: 2 CPU 제한된 컨테이너
worker_processes 2;
```

### worker_connections

- 각 Worker가 처리할 수 있는 최대 동시 연결 수임.

```nginx
events {
    # 기본값: 512, 일반적으로 1024~4096 권장
    worker_connections 4096;

    # 새 연결을 한 번에 여러 개 수락
    multi_accept on;

    # 이벤트 처리 방식 지정 (Linux)
    use epoll;
}
```

#### 최대 연결 수 계산

- 최대 클라이언트 수 = worker_processes × worker_connections
- 예: 4 Workers × 4096 connections = 16,384 동시 연결
- 실제로는 시스템의 파일 디스크립터 제한(~65,535)에 의해 제한됨
- 리버스 프록시 사용 시, 클라이언트 연결 + 백엔드 연결로 2배 필요

### 파일 디스크립터 제한

```nginx
# nginx.conf
worker_rlimit_nofile 100000;  # Worker당 열 수 있는 최대 파일 수
```

```bash
# 시스템 레벨 설정 (/etc/security/limits.conf)
nginx soft nofile 100000
nginx hard nofile 100000

# 또는 systemd 서비스 파일
[Service]
LimitNOFILE=100000
```

### 버퍼 최적화

```nginx
http {
    # 클라이언트 요청 버퍼
    client_body_buffer_size 16k;
    client_header_buffer_size 1k;
    client_max_body_size 100m;
    large_client_header_buffers 4 8k;

    # 프록시 버퍼
    proxy_buffer_size 4k;
    proxy_buffers 8 16k;
    proxy_busy_buffers_size 24k;

    # 타임아웃 설정
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout 10;
    keepalive_timeout 65;
    keepalive_requests 1000;
}
```

### 종합 성능 최적화 설정

```nginx
user nginx;
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 100000;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # 로그 포맷
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time"';

    access_log /var/log/nginx/access.log main buffer=16k flush=2m;

    # 성능 최적화
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;

    # Keepalive
    keepalive_timeout 65;
    keepalive_requests 1000;

    # 해시 테이블 크기
    types_hash_max_size 2048;
    server_names_hash_bucket_size 64;

    # 버전 정보 숨기기
    server_tokens off;

    include /etc/nginx/conf.d/*.conf;
}
```

## SSL/TLS 설정

### 기본 SSL 설정

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # 인증서 설정
    ssl_certificate /etc/ssl/certs/example.com.fullchain.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # 프로토콜 버전 (TLS 1.2, 1.3만 허용)
    ssl_protocols TLSv1.2 TLSv1.3;

    # 암호화 스위트
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # 클라이언트가 암호화 스위트 선택 (성능상 유리)
    ssl_prefer_server_ciphers off;
}
```

### Qualys SSL Labs A+ 등급 설정

```nginx
# SSL 세션 설정
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

# Diffie-Hellman 파라미터 (4096비트 권장)
# 생성: openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
ssl_dhparam /etc/ssl/certs/dhparam.pem;

# OCSP Stapling (핸드셰이크 시간 단축)
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;

# HSTS (HTTP Strict Transport Security)
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

### 완전한 SSL 서버 블록

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.example.com;

    ssl_certificate /etc/ssl/certs/example.com.fullchain.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    return 301 https://example.com$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    # 인증서
    ssl_certificate /etc/ssl/certs/example.com.fullchain.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # 프로토콜 및 암호화
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # 세션
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # DH 파라미터
    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # 보안 헤더
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;

    root /var/www/example.com;
    index index.html;
}
```

### Let's Encrypt 자동 갱신

```nginx
# Certbot webroot 인증을 위한 location
server {
    listen 80;
    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$server_name$request_uri;
    }
}
```

```bash
# 인증서 발급
certbot certonly --webroot -w /var/www/certbot -d example.com

# 자동 갱신 테스트
certbot renew --dry-run

# 크론잡 설정 (자동 갱신)
0 0 1 * * /usr/bin/certbot renew --quiet && systemctl reload nginx
```

## Gzip 압축

### 기본 Gzip 설정

```nginx
http {
    # Gzip 활성화
    gzip on;

    # 압축 레벨 (1-9, 기본값 1)
    # 높을수록 압축률 좋지만 CPU 사용량 증가
    gzip_comp_level 5;

    # 압축할 최소 크기 (너무 작으면 오히려 비효율)
    gzip_min_length 256;

    # 프록시 요청에도 압축 적용
    gzip_proxied any;

    # Vary 헤더 추가 (캐시 서버 호환성)
    gzip_vary on;

    # 압축할 MIME 타입
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/ld+json
        application/manifest+json
        application/rss+xml
        application/vnd.geo+json
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        application/xhtml+xml
        application/xml
        font/opentype
        image/bmp
        image/svg+xml
        image/x-icon
        text/cache-manifest
        text/css
        text/plain
        text/vcard
        text/vnd.rim.location.xloc
        text/vtt
        text/x-component
        text/x-cross-domain-policy
        text/xml;

    # 이미 압축된 파일 사용 (사전 압축)
    gzip_static on;
}
```

### 압축 레벨별 성능 비교

- 1-2:
  - 압축률: 낮음
  - CPU 사용량: 낮음
  - 권장 상황: CPU 병목, 빠른 응답 필요
- 4-5:
  - 압축률: 중간
  - CPU 사용량: 중간
  - 권장 상황: 일반적인 웹 서비스 (권장)
- 6-9:
  - 압축률: 높음
  - CPU 사용량: 높음
  - 권장 상황: 대역폭 제한, 정적 파일 사전 압축

### 사전 압축 (gzip_static)

- 빌드 시점에 미리 압축된 파일을 제공하여 런타임 CPU 사용량을 줄임.

```bash
# 빌드 스크립트에서 사전 압축
find /var/www/html -type f \( -name "*.js" -o -name "*.css" -o -name "*.html" \) \
  -exec gzip -9 -k {} \;

# 결과: script.js와 script.js.gz 둘 다 존재
# Nginx는 클라이언트가 gzip을 지원하면 .gz 파일 제공
```

```nginx
location /static/ {
    gzip_static on;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

## Rate Limiting

- Nginx의 Rate Limiting은 Leaky Bucket 알고리즘을 사용하여 요청 속도를 제한함.

### 요청 수 제한 (limit_req)

```nginx
http {
    # 제한 영역 정의
    # $binary_remote_addr: 클라이언트 IP (4바이트 IPv4, 16바이트 IPv6)
    # zone=mylimit:10m: 10MB 공유 메모리 (약 160,000 IP 저장 가능)
    # rate=10r/s: 초당 10개 요청
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

    # 분당 요청 제한
    limit_req_zone $binary_remote_addr zone=permin:10m rate=60r/m;

    server {
        # 기본 적용 (초과 요청 즉시 거부)
        location /api/ {
            limit_req zone=mylimit;
            proxy_pass http://backend;
        }

        # burst 허용 (초과 요청을 큐에 대기)
        location /api/search {
            limit_req zone=mylimit burst=20;
            proxy_pass http://backend;
        }

        # burst + nodelay (버스트 요청 즉시 처리)
        location /api/submit {
            limit_req zone=mylimit burst=20 nodelay;
            proxy_pass http://backend;
        }
    }
}
```

#### burst와 nodelay 이해

- rate=10r/s, burst=20 설정 시:
- [burst 없음]
- 100ms마다 1개 요청 허용
- 초과 요청 즉시 503 반환
- [burst=20]
- 초과 요청을 20개까지 큐에 대기
- 큐의 요청은 100ms 간격으로 처리
- 20개 초과 시 503 반환
- [burst=20 nodelay]
- 처음 21개 요청(1 + burst) 즉시 처리
- 이후 100ms마다 버스트 슬롯 1개 회복
- 슬롯 소진 후 초과 요청은 503 반환

### 연결 수 제한 (limit_conn)

```nginx
http {
    # IP당 연결 수 제한
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    # 서버당 연결 수 제한
    limit_conn_zone $server_name zone=servers:10m;

    server {
        # IP당 최대 10개 연결
        limit_conn addr 10;

        # 다운로드 영역: IP당 1개 연결, 속도 제한
        location /download/ {
            limit_conn addr 1;
            limit_rate 100k;  # 100KB/s 속도 제한
        }
    }
}
```

### 로그인/API 보호 예제

```nginx
http {
    # 로그인 시도 제한 (분당 5회)
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

    # API 전체 제한 (초당 100회)
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;

    # 커스텀 에러 코드
    limit_req_status 429;
    limit_conn_status 429;

    server {
        # 로그인 엔드포인트 보호
        location /api/login {
            limit_req zone=login burst=3 nodelay;

            # 429 에러 페이지
            error_page 429 = @rate_limited;

            proxy_pass http://backend;
        }

        location @rate_limited {
            default_type application/json;
            return 429 '{"error": "Too Many Requests", "retry_after": 60}';
        }

        # 일반 API
        location /api/ {
            limit_req zone=api burst=50 nodelay;
            proxy_pass http://backend;
        }
    }
}
```

### 화이트리스트 설정

```nginx
http {
    # geo 모듈로 IP 매핑
    geo $limit {
        default 1;
        10.0.0.0/8 0;      # 내부 네트워크 제외
        192.168.0.0/16 0;  # 내부 네트워크 제외
    }

    # 제한 대상만 키로 사용
    map $limit $limit_key {
        0 "";
        1 $binary_remote_addr;
    }

    limit_req_zone $limit_key zone=api:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=api burst=20;
            proxy_pass http://backend;
        }
    }
}
```

## 참고 자료

- [Nginx 공식 문서](https://nginx.org/en/docs/)
- [Nginx Blog - Inside NGINX](https://blog.nginx.org/blog/inside-nginx-how-we-designed-for-performance-scale)
- [Nginx Architecture](https://blog.nginx.org/nginx-architecture)
- [Nginx Performance Tuning](https://www.nginx.com/blog/tuning-nginx/)
- [Nginx Rate Limiting](https://blog.nginx.org/blog/rate-limiting-nginx)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)

## 관련 학습

- [Apache](02-Apache.md)

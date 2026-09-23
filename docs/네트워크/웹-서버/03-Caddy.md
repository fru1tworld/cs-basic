# Caddy

- Caddy는 Go로 작성된 오픈소스 웹 서버이며 자동 HTTPS와 Caddyfile 설정을 제공함.

## 왜 Caddy인가?

인증서를 별도 도구로 관리하는 구성에서는 발급뿐 아니라 웹 서버 연결과 갱신도 설정해야 한다. Apache나 Nginx에 Certbot을 연결하는 예를 들면 다음 작업이 필요하다.

- Let's Encrypt에서 certbot 설치
- 인증서 발급 명령 실행
- 웹 서버 설정 파일에 인증서 경로 추가
- Cron job으로 인증서 갱신 자동화 설정

Caddy는 이 인증서 관리 과정을 서버에 통합한다. 아래 예제처럼 도메인을 지정하면 자동 HTTPS가 인증서 발급과 갱신을 맡는다.

```caddyfile
# Nginx에서 HTTPS 설정 (약 20줄 이상)
server {
    listen 443 ssl;
    server_name example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    # ... 추가 SSL 설정
}

# Caddy에서 HTTPS 설정 (1줄)
example.com
```

## 핵심 특징

### 자동 HTTPS (Automatic HTTPS)

- Caddy의 가장 핵심적인 기능으로, 다음을 자동으로 처리함:

- 기능: 인증서 발급:
  - 설명: ACME 프로토콜을 통해 Let's Encrypt에서 자동 발급
- 기능: 인증서 갱신:
  - 설명: 만료 전 자동 갱신 (별도 설정 불필요)
- 기능: HTTP -> HTTPS 리다이렉트:
  - 설명: 포트 80 요청을 자동으로 443으로 리다이렉트
- 기능: OCSP Stapling:
  - 설명: 인증서 상태 확인을 위한 OCSP 응답 자동 캐싱
- 기능: 이중화된 CA:
  - 설명: Let's Encrypt 실패 시 ZeroSSL로 자동 폴백

```caddyfile
# 이것만으로 자동 HTTPS가 활성화됨
example.com {
    respond "Hello, secure world!"
}
```

### HTTP/3 및 QUIC 지원

- Caddy 2.6부터 HTTP/3를 기본으로 활성화한 최초의 범용 웹 서버가 됨. HTTP/3는 UDP 기반의 QUIC 프로토콜을 사용하여 더 빠른 연결 수립과 향상된 성능을 제공함.

```yaml
# Docker에서 HTTP/3 지원을 위한 포트 설정
ports:
  - "80:80"
  - "443:443"
  - "443:443/udp"  # HTTP/3를 위한 UDP 포트
```

### Go 기반의 장점

- 단일 바이너리: 외부 의존성 없이 하나의 실행 파일로 배포
- 크로스 플랫폼: Linux, macOS, Windows, BSD 등 다양한 플랫폼 지원
- 높은 동시성: Go의 고루틴을 활용한 효율적인 동시 처리
- 메모리 안전성: Go의 가비지 컬렉션으로 메모리 누수 방지

### 기본 내장 기능

- 별도 모듈 설치 없이 제공되는 기능:

- 자동 HTTPS 및 인증서 관리
- HTTP/3 및 QUIC 지원
- 리버스 프록시
- 로드 밸런싱
- Gzip/Zstd 압축
- 정적 파일 서버
- Virtual Host
- 액세스 로그
- 헤더 조작
- Basic Auth

## Caddyfile 설정 방법

Caddy는 JSON과 Caddyfile로 설정할 수 있다. 직접 설정을 작성할 때는 사이트 주소 아래에 지시어를 배치하는 Caddyfile을 사용하면 구조를 읽기 쉽다.

### 기본 문법

```caddyfile
# 기본 구조
사이트주소 {
    directive [matcher] [args...]
}
```

### Global Options Block

- Caddyfile 최상단에 위치하며, 전역 설정을 정의함:

```caddyfile
{
    # 이메일 설정 (Let's Encrypt 알림용)
    email admin@example.com

    # ACME CA 서버 (테스트 시 staging 사용 권장)
    acme_ca https://acme-staging-v02.api.letsencrypt.org/directory

    # 관리자 API 활성화
    admin localhost:2019

    # HTTP/3 활성화 (2.6+ 기본값)
    servers {
        protocols h1 h2 h3
    }
}
```

### 주요 Directive

- Directive: `root`:
  - 설명: 정적 파일 루트 디렉터리
  - 예시: `root * /var/www`
- Directive: `file_server`:
  - 설명: 정적 파일 서버 활성화
  - 예시: `file_server browse`
- Directive: `reverse_proxy`:
  - 설명: 리버스 프록시 설정
  - 예시: `reverse_proxy localhost:8080`
- Directive: `encode`:
  - 설명: 압축 설정
  - 예시: `encode gzip zstd`
- Directive: `header`:
  - 설명: HTTP 헤더 조작
  - 예시: `header Cache-Control "max-age=3600"`
- Directive: `redir`:
  - 설명: 리다이렉트
  - 예시: `redir /old /new permanent`
- Directive: `respond`:
  - 설명: 직접 응답 반환
  - 예시: `respond "Hello World" 200`
- Directive: `tls`:
  - 설명: TLS 설정
  - 예시: `tls internal`
- Directive: `log`:
  - 설명: 로그 설정
  - 예시: `log { output file /var/log/access.log }`
- Directive: `basic_auth`:
  - 설명: 기본 인증
  - 예시: `basic_auth /admin/*`

### Matcher (요청 매칭)

모든 요청에 같은 설정을 적용할 필요는 없다. Matcher로 경로나 헤더 조건을 지정하면 해당 요청에만 지시어가 적용된다. 아래 예제는 API 경로, 정적 파일, WebSocket 요청을 각각 구분한다.

```caddyfile
# 모든 요청에 적용
root * /var/www

# 특정 경로에만 적용
reverse_proxy /api/* localhost:3000

# Named matcher 정의
@static {
    path *.css *.js *.png *.jpg *.gif
}
header @static Cache-Control "public, max-age=31536000"

# 여러 조건 조합
@websockets {
    header Connection *Upgrade*
    header Upgrade websocket
}
reverse_proxy @websockets localhost:6001
```

### 여러 사이트 설정

```caddyfile
# 첫 번째 사이트
example.com {
    root * /var/www/example
    file_server
}

# 두 번째 사이트
api.example.com {
    reverse_proxy localhost:3000
}

# 서브도메인 와일드카드
*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }

    @blog host blog.example.com
    handle @blog {
        reverse_proxy localhost:2368
    }

    @shop host shop.example.com
    handle @shop {
        reverse_proxy localhost:4000
    }
}
```

## 리버스 프록시 설정

### 기본 리버스 프록시

```caddyfile
example.com {
    reverse_proxy localhost:8080
}
```

### 로드 밸런싱

- 여러 업스트림 서버로 트래픽을 분산함:

```caddyfile
example.com {
    reverse_proxy {
        to localhost:8001 localhost:8002 localhost:8003

        # 로드 밸런싱 정책
        lb_policy round_robin

        # 재시도 설정
        lb_retries 3
        lb_try_duration 5s
        lb_try_interval 250ms
    }
}
```

#### 로드 밸런싱 정책

- 정책: `random`:
  - 설명: 무작위 선택 (기본값)
- 정책: `round_robin`:
  - 설명: 순차적 분배
- 정책: `least_conn`:
  - 설명: 연결 수가 가장 적은 서버 선택
- 정책: `first`:
  - 설명: 설정 순서대로 우선 사용
- 정책: `ip_hash`:
  - 설명: 클라이언트 IP 기반 고정
- 정책: `uri_hash`:
  - 설명: URI 기반 고정
- 정책: `random_choose <n>`:
  - 설명: n개 무작위 선택 후 부하가 적은 것 선택

### 헬스 체크

트래픽을 여러 서버로 나눈 뒤에는 요청을 받을 수 없는 서버를 걸러야 한다. 아래 설정은 실제 요청의 실패를 기록하는 패시브 검사와 `/health`를 주기적으로 호출하는 액티브 검사를 함께 사용한다.

```caddyfile
example.com {
    reverse_proxy {
        to 10.0.1.1:80 10.0.1.2:80 10.0.1.3:80

        lb_policy least_conn

        # 패시브 헬스 체크 (요청 실패 기반)
        fail_duration 10s
        max_fails 3

        # 액티브 헬스 체크 (주기적 확인)
        health_uri /health
        health_interval 30s
        health_timeout 5s
        health_status 200
    }
}
```

### 헤더 조작

```caddyfile
example.com {
    reverse_proxy localhost:8080 {
        # 업스트림으로 전달할 헤더 설정
        header_up Host {upstream_hostport}
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}

        # 클라이언트로 응답할 헤더 설정
        header_down -Server  # Server 헤더 제거
        header_down X-Powered-By "Caddy"
    }
}
```

### WebSocket 프록시

```caddyfile
example.com {
    reverse_proxy /ws/* localhost:8080 {
        # WebSocket을 위한 설정
        header_up Connection {http.request.header.Connection}
        header_up Upgrade {http.request.header.Upgrade}
    }
}
```

### 경로별 라우팅

같은 도메인에서 API와 정적 파일, 프론트엔드를 제공하려면 `handle`로 경로별 처리를 나눈다. 아래 예제는 `/api/*`를 API 서버로 보내고 `/static/*`는 파일로 제공하며, 나머지는 프론트엔드 서버로 전달한다.

```caddyfile
example.com {
    # API 서버
    handle /api/* {
        reverse_proxy localhost:3000
    }

    # 정적 파일
    handle /static/* {
        root * /var/www
        file_server
    }

    # 프론트엔드 SPA
    handle {
        reverse_proxy localhost:8080
    }
}
```

## 자동 TLS/SSL 인증서 관리

### ACME 프로토콜

- Caddy는 ACME(Automatic Certificate Management Environment) 프로토콜을 사용하여 Let's Encrypt와 통신함.

#### 지원하는 Challenge 유형

- Challenge: HTTP-01:
  - 설명: 포트 80으로 검증 파일 요청
  - 사용 조건: 포트 80 접근 가능
- Challenge: TLS-ALPN-01:
  - 설명: 포트 443으로 TLS 핸드셰이크
  - 사용 조건: 포트 443 접근 가능
- Challenge: DNS-01:
  - 설명: DNS TXT 레코드로 검증
  - 사용 조건: 와일드카드 인증서, 내부망

### 기본 HTTPS 설정

```caddyfile
# 도메인만 지정하면 자동 HTTPS 활성화
example.com {
    respond "Secure!"
}
```

### TLS 세부 설정

```caddyfile
example.com {
    tls {
        # ACME 이메일
        email admin@example.com

        # TLS 버전 제한
        protocols tls1.2 tls1.3

        # 암호화 스위트 지정
        ciphers TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384

        # 곡선 설정
        curves x25519 secp256r1 secp384r1
    }
}
```

### 와일드카드 인증서 (DNS Challenge)

- 와일드카드 인증서는 DNS-01 Challenge가 필수임:

```caddyfile
{
    email admin@example.com
}

*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }

    @www host www.example.com
    handle @www {
        redir https://example.com{uri}
    }

    handle {
        reverse_proxy localhost:8080
    }
}
```

### 내부/개발 환경 인증서

```caddyfile
# 내부 CA 사용 (localhost, 사설 IP용)
localhost {
    tls internal
    respond "Local development"
}

# 자체 인증서 사용
example.internal {
    tls /path/to/cert.pem /path/to/key.pem
}
```

### On-Demand TLS

- 요청 시점에 동적으로 인증서를 발급받는 기능:

```caddyfile
{
    on_demand_tls {
        # 허용할 도메인 확인 API
        ask https://api.example.com/check-domain

        # 버스트 제한
        burst 5
        interval 2m
    }
}

https:// {
    tls {
        on_demand
    }
    reverse_proxy localhost:8080
}
```

### 테스트 환경 설정

- Let's Encrypt Rate Limit을 피하기 위해 staging 환경 사용:

```caddyfile
{
    # Staging CA 사용 (테스트용)
    acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}

example.com {
    respond "Testing HTTPS"
}
```

### 인증서 저장 및 관리

- # 기본 저장 위치
- # Linux: ~/.local/share/caddy/
- # macOS: ~/Library/Application Support/Caddy/
- # Windows: %AppData%\Caddy\
- # 인증서 정보 확인
- caddy list-certificates
- # 설정 검증
- caddy validate --config /etc/caddy/Caddyfile

## Nginx/Apache와의 비교

### 종합 비교표

- 항목: 언어:
  - Caddy: Go
  - Nginx: C
  - Apache: C
- 항목: 자동 HTTPS:
  - Caddy: 기본 내장
  - Nginx: 별도 설정 필요
  - Apache: 별도 설정 필요
- 항목: 설정 난이도:
  - Caddy: 쉬움
  - Nginx: 중간
  - Apache: 복잡함
- 항목: HTTP/3:
  - Caddy: 기본 지원 (2.6+)
  - Nginx: nginx-quic 별도
  - Apache: mod_http2
- 항목: 동적 설정:
  - Caddy: API 지원
  - Nginx: reload 필요
  - Apache: .htaccess
- 항목: 성능:
  - Caddy: 높음
  - Nginx: 매우 높음
  - Apache: 중간
- 항목: 메모리 사용:
  - Caddy: 낮음
  - Nginx: 매우 낮음
  - Apache: 높음
- 항목: 생태계:
  - Caddy: 성장 중
  - Nginx: 매우 넓음
  - Apache: 매우 넓음
- 항목: 엔터프라이즈:
  - Caddy: 기본 무료
  - Nginx: Plus 유료
  - Apache: 무료

### 설정 파일 비교

- 정적 파일 서버 + 리버스 프록시

```nginx
# Nginx (약 40줄)
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;

    root /var/www/html;
    index index.html;

    location /api/ {
        proxy_pass http://localhost:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    gzip on;
    gzip_types text/plain application/json application/javascript text/css;
}
```

```caddyfile
# Caddy
example.com {
    root * /var/www/html

    handle /api/* {
        reverse_proxy localhost:3000
    }

    handle {
        try_files {path} {path}/
        file_server
    }

    encode gzip
}
```

### 성능 비교

- 벤치마크 결과에 따르면:

- 정적 파일: Nginx와 Caddy가 비슷한 성능, Apache는 다소 낮음
- 리버스 프록시: Caddy가 HTTPS 환경에서 Nginx보다 약 4배 빠른 응답
- 버스트 트래픽: Nginx가 가장 안정적 (202.19 RPS, 최저 레이턴시)
- 동시 연결: Nginx > Caddy > Apache

### 언제 어떤 서버를 선택할까?

- Caddy를 선택해야 할 때:
- 빠른 HTTPS 설정이 필요한 경우
- 개인 프로젝트, 스타트업, 중소규모 서비스
- API 게이트웨이, 마이크로서비스
- 개발/스테이징 환경
- Go 기반 인프라

- Nginx를 선택해야 할 때:
- 대규모 트래픽 처리가 필요한 경우
- 섬세한 성능 튜닝이 필요한 경우
- 기존 Nginx 인프라 확장
- CDN, 대용량 미디어 서빙
- 레거시 시스템 통합

- Apache를 선택해야 할 때:
- .htaccess 기반 설정이 필요한 경우
- PHP 기반 레거시 애플리케이션 (WordPress 등)
- 동적 모듈 로딩이 필요한 경우
- 복잡한 URL 재작성 규칙

## 실제 설정 예제

### Docker Compose 프로덕션 배포

```yaml
# docker-compose.yml
version: "3.8"

services:
  caddy:
    image: caddy:2.7-alpine
    container_name: caddy
    restart: unless-stopped
    cap_add:
      - NET_ADMIN  # HTTP/3 버퍼 크기 최적화용
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./site:/srv
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - DOMAIN=example.com
    networks:
      - web

  api:
    image: your-api:latest
    container_name: api
    restart: unless-stopped
    expose:
      - "3000"
    networks:
      - web

  frontend:
    image: your-frontend:latest
    container_name: frontend
    restart: unless-stopped
    expose:
      - "8080"
    networks:
      - web

networks:
  web:
    driver: bridge

volumes:
  caddy_data:
  caddy_config:
```

```caddyfile
# Caddyfile
{
    email admin@example.com
    servers {
        protocols h1 h2 h3
    }
}

{$DOMAIN} {
    # 로그 설정
    log {
        output file /var/log/caddy/access.log {
            roll_size 100mb
            roll_keep 5
        }
        format json
    }

    # 보안 헤더
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
        -Server
    }

    # API 라우팅
    handle /api/* {
        reverse_proxy api:3000 {
            header_up X-Real-IP {remote_host}
            header_up X-Forwarded-For {remote_host}
            header_up X-Forwarded-Proto {scheme}
        }
    }

    # 프론트엔드 SPA
    handle {
        reverse_proxy frontend:8080
    }

    # 압축
    encode gzip zstd
}

# 헬스 체크 엔드포인트
:8080 {
    respond /health "OK" 200
}
```

### 마이크로서비스 API 게이트웨이

```caddyfile
{
    email devops@company.com
    admin localhost:2019
}

api.company.com {
    # Rate Limiting (별도 플러그인 필요)
    # rate_limit {
    #     zone dynamic_zone {
    #         key {remote_host}
    #         events 100
    #         window 1m
    #     }
    # }

    # CORS 설정
    @cors_preflight method OPTIONS
    handle @cors_preflight {
        header Access-Control-Allow-Origin "*"
        header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS"
        header Access-Control-Allow-Headers "Content-Type, Authorization"
        respond "" 204
    }

    # 서비스별 라우팅
    handle /v1/users/* {
        reverse_proxy user-service:8001 {
            lb_policy round_robin
            health_uri /health
            health_interval 10s
        }
    }

    handle /v1/orders/* {
        reverse_proxy order-service-1:8002 order-service-2:8002 {
            lb_policy least_conn
            lb_retries 2
            fail_duration 30s
        }
    }

    handle /v1/products/* {
        reverse_proxy product-service:8003
    }

    # 기본 응답
    handle {
        respond "API Gateway" 200
    }

    encode gzip

    log {
        output file /var/log/caddy/api-access.log
        format json
    }
}
```

### 정적 사이트 + SPA 호스팅

```caddyfile
example.com {
    root * /var/www/example

    # SPA 라우팅: 파일이 없으면 index.html로
    try_files {path} /index.html

    file_server

    # 정적 자산 캐싱
    @static {
        path *.css *.js *.png *.jpg *.jpeg *.gif *.ico *.svg *.woff *.woff2
    }
    header @static Cache-Control "public, max-age=31536000, immutable"

    # HTML은 캐시하지 않음
    @html {
        path *.html
    }
    header @html Cache-Control "no-cache, no-store, must-revalidate"

    encode gzip
}
```

### WordPress/PHP 호스팅

```caddyfile
wordpress.example.com {
    root * /var/www/wordpress

    # PHP-FPM 연결
    php_fastcgi unix//run/php/php8.2-fpm.sock {
        # 또는 TCP 연결
        # php_fastcgi localhost:9000
    }

    file_server

    # 보안: 민감한 파일 접근 차단
    @blocked {
        path /wp-config.php
        path /wp-includes/*.php
        path /wp-content/uploads/*.php
    }
    respond @blocked 403

    encode gzip
}
```

### 개발 환경 (로컬 HTTPS)

```caddyfile
{
    # 로컬 개발용 내부 CA 사용
    local_certs
    auto_https disable_redirects
}

localhost:3000 {
    tls internal
    reverse_proxy localhost:8080
}

# 여러 프로젝트 동시 개발
project1.localhost {
    tls internal
    reverse_proxy localhost:8081
}

project2.localhost {
    tls internal
    reverse_proxy localhost:8082
}
```

### Caddy 관리 명령어

```bash
# Caddy 설치 (Linux)
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

# 설정 검증
caddy validate --config /etc/caddy/Caddyfile

# Caddy 시작
caddy start

# 설정 리로드 (무중단)
caddy reload --config /etc/caddy/Caddyfile

# 포맷팅
caddy fmt --overwrite /etc/caddy/Caddyfile

# 인증서 목록 확인
caddy list-certificates

# API로 설정 확인
curl localhost:2019/config/

# Docker에서 리로드
docker compose exec -w /etc/caddy caddy caddy reload
```

## 참고 자료

- [Caddy 공식 사이트](https://caddyserver.com/)
- [Caddy 공식 문서](https://caddyserver.com/docs/)
- [Automatic HTTPS 문서](https://caddyserver.com/docs/automatic-https)
- [Caddyfile 문법](https://caddyserver.com/docs/caddyfile)
- [Caddyfile Directives](https://caddyserver.com/docs/caddyfile/directives)
- [reverse_proxy Directive](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy)
- [TLS Directive](https://caddyserver.com/docs/caddyfile/directives/tls)
- [Global Options](https://caddyserver.com/docs/caddyfile/options)
- [Common Caddyfile Patterns](https://caddyserver.com/docs/caddyfile/patterns)

- [Caddy GitHub Repository](https://github.com/caddyserver/caddy)
- [CertMagic - ACME 라이브러리](https://github.com/caddyserver/certmagic)

- [Caddy Official Docker Image](https://hub.docker.com/_/caddy)
- [Docker 배포 가이드](https://www.docker.com/blog/deploying-web-applications-quicker-and-easier-with-caddy-2/)

- [Caddy Community Forum](https://caddy.community/)
- [Caddy Discord](https://discord.gg/caddy)

## 관련 학습

- [Apache](02-Apache.md)

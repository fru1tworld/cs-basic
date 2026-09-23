# Apache

## Apache 개요

- Apache HTTP Server는 1995년에 시작된 오픈소스 웹 서버로, 오랫동안 세계에서 가장 널리 사용되는 웹 서버였음. 모듈식 아키텍처와 .htaccess 기반의 유연한 설정이 특징임.

### 주요 특징

- 모듈식 아키텍처: 필요한 기능만 로드하여 사용
- 유연한 설정: .htaccess로 디렉터리별 설정 가능
- 광범위한 호환성: 다양한 스크립팅 언어 지원 (PHP, Perl, Python 등)
- 풍부한 문서: 20년 이상의 축적된 자료

### 설정 파일 구조

- /etc/httpd/, # RHEL/CentOS
- /etc/apache2/, # Debian/Ubuntu
- apache2.conf, # 메인 설정 파일
- ports.conf, # 포트 설정
- mods-available/, # 사용 가능한 모듈
- mods-enabled/, # 활성화된 모듈 (심볼릭 링크)
- sites-available/, # 사용 가능한 가상 호스트
- sites-enabled/, # 활성화된 가상 호스트 (심볼릭 링크)
- conf-available/, # 추가 설정

## MPM (Multi-Processing Module)

Apache의 요청 처리 방식은 MPM(Multi-Processing Module)에 따라 달라진다. 연결을 프로세스에 맡길지, 한 프로세스의 여러 스레드로 나눌지, 유휴 연결을 별도로 관리할지를 이 모듈이 결정한다.

### Prefork MPM

Prefork는 각 연결을 별도 자식 프로세스로 처리한다. Master가 자식 프로세스를 관리하고, 각 자식이 Conn 1, Conn 2 같은 연결을 하나씩 맡는다.

#### 특징

- 안정성: 프로세스 격리로 하나의 요청 실패가 다른 요청에 영향 없음
- 호환성: 스레드 안전하지 않은 라이브러리와도 호환
- 메모리 사용량: 높음 (프로세스당 ~10-50MB)
- PHP mod_php와 함께 사용 시 필수

#### 설정

```apache
# /etc/httpd/conf.modules.d/00-mpm.conf 또는
# /etc/apache2/mods-available/mpm_prefork.conf

<IfModule mpm_prefork_module>
    StartServers             5      # 시작 시 생성할 자식 프로세스 수
    MinSpareServers          5      # 유휴 상태로 유지할 최소 프로세스
    MaxSpareServers         10      # 유휴 상태로 유지할 최대 프로세스
    MaxRequestWorkers      250      # 최대 동시 연결 수
    MaxConnectionsPerChild   0      # 프로세스당 처리할 최대 요청 (0=무제한)
</IfModule>
```

#### 메모리 계산

- 필요 메모리 = MaxRequestWorkers × 프로세스당 메모리
- 예: 250 × 50MB = 12.5GB
- # 권장: 시스템 메모리의 70-80% 이내로 설정
- # 8GB 서버 → MaxRequestWorkers ≈ 100-120

### Worker MPM

Worker는 여러 자식 프로세스를 만들고 각 프로세스 안에 여러 스레드를 두는 방식이다. 같은 프로세스의 스레드들이 메모리를 공유하므로, 연결마다 프로세스를 두는 Prefork보다 메모리 사용을 줄일 수 있다.

#### 특징

- 효율성: Prefork보다 적은 메모리로 더 많은 연결 처리
- 스레드 공유: 같은 프로세스 내 스레드들이 메모리 공유
- 제약: 스레드 안전한 라이브러리만 사용 가능
- PHP-FPM과 함께 사용

#### 설정

```apache
<IfModule mpm_worker_module>
    StartServers             3      # 시작 시 생성할 프로세스 수
    MinSpareThreads         75      # 유휴 스레드 최소 수
    MaxSpareThreads        250      # 유휴 스레드 최대 수
    ThreadsPerChild         25      # 프로세스당 스레드 수
    MaxRequestWorkers      400      # 최대 동시 연결 (프로세스 × 스레드)
    MaxConnectionsPerChild   0
</IfModule>
```

### Event MPM

- Apache 2.4 기본 MPM으로, Worker MPM을 개선한 이벤트 기반 모델임.

- Master Process
- Child Process, Child Process, Child Process
- Listener, Listener, Listener
- Thread, Thread, Thread
- Worker, Worker, Worker
- Threads, Threads, Threads

#### 핵심 개선점: Keep-Alive 처리

Worker MPM에서는 요청을 처리한 스레드가 Keep-Alive 연결의 다음 요청을 기다리며 점유될 수 있다. Event MPM은 유휴 Keep-Alive 연결을 리스너 스레드가 관리하도록 분리한다. 덕분에 요청 처리를 마친 워커 스레드는 다른 요청을 맡을 수 있다.

#### 특징

- Keep-Alive 효율화: 전용 리스너 스레드가 유휴 연결 관리
- 높은 동시성: Worker보다 더 많은 동시 연결 처리 가능
- Nginx와 유사: 이벤트 기반 접근 방식
- 현대적 선택: PHP-FPM과 함께 권장

#### 설정

```apache
<IfModule mpm_event_module>
    StartServers             3
    MinSpareThreads         75
    MaxSpareThreads        250
    ThreadsPerChild         25
    MaxRequestWorkers      400
    MaxConnectionsPerChild   0

    # Event MPM 전용 설정
    AsyncRequestWorkerFactor 2  # 비동기 요청 처리 배율
</IfModule>
```

### MPM 비교 표

- 특성: 처리 방식:
  - Prefork: 프로세스
  - Worker: 프로세스+스레드
  - Event: 이벤트 기반
- 특성: 메모리 효율:
  - Prefork: 낮음
  - Worker: 중간
  - Event: 높음
- 특성: 동시 연결:
  - Prefork: 낮음
  - Worker: 중간
  - Event: 높음
- 특성: Keep-Alive:
  - Prefork: 비효율적
  - Worker: 비효율적
  - Event: 효율적
- 특성: PHP 호환:
  - Prefork: mod_php
  - Worker: PHP-FPM
  - Event: PHP-FPM
- 특성: 안정성:
  - Prefork: 높음
  - Worker: 중간
  - Event: 중간
- 특성: 사용 사례:
  - Prefork: 레거시, mod_php
  - Worker: 일반
  - Event: 고성능 (권장)

### MPM 확인 및 변경

```bash
# 현재 MPM 확인
apachectl -V | grep MPM
# 또는
httpd -V | grep MPM

# Ubuntu/Debian에서 MPM 변경
sudo a2dismod mpm_prefork
sudo a2enmod mpm_event
sudo systemctl restart apache2

# RHEL/CentOS에서 MPM 변경 (httpd.conf 또는 00-mpm.conf 수정)
# LoadModule mpm_event_module modules/mod_mpm_event.so
```

## 모듈 시스템

- Apache의 강력한 기능은 모듈을 통해 제공됨.

### 핵심 모듈 분류

- Apache HTTP Server
- Core Modules (핵심)
- core, http_core, mod_so
- MPM Modules (멀티프로세싱)
- mpm_prefork, mpm_worker, mpm_event
- Auth Modules (인증)
- mod_auth_basic, mod_auth_digest, mod_authn_file
- Access Modules (접근 제어)
- mod_authz_host, mod_authz_user, mod_access_compat
- Handler Modules (콘텐츠 처리)
- mod_cgi, mod_php, mod_proxy, mod_proxy_fcgi
- Filter Modules (필터링)
- mod_deflate, mod_filter, mod_substitute
- Logging Modules (로깅)
- mod_log_config, mod_logio
- Security Modules (보안)
- mod_ssl, mod_security, mod_evasive

### 주요 모듈 상세

#### mod_ssl (SSL/TLS)

```apache
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/example.com.crt
    SSLCertificateKeyFile /etc/ssl/private/example.com.key
    SSLCertificateChainFile /etc/ssl/certs/chain.pem

    # 보안 설정
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    SSLHonorCipherOrder off

    # HSTS
    Header always set Strict-Transport-Security "max-age=63072000"
</VirtualHost>
```

#### mod_proxy (리버스 프록시)

```apache
<VirtualHost *:80>
    ServerName example.com

    ProxyPreserveHost On
    ProxyPass / http://localhost:8080/
    ProxyPassReverse / http://localhost:8080/

    # 로드 밸런싱
    <Proxy balancer://mycluster>
        BalancerMember http://192.168.1.10:8080 loadfactor=1
        BalancerMember http://192.168.1.11:8080 loadfactor=1
        ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPass /api balancer://mycluster
    ProxyPassReverse /api balancer://mycluster
</VirtualHost>
```

#### mod_deflate (압축)

```apache
<IfModule mod_deflate.c>
    # 압축할 MIME 타입
    AddOutputFilterByType DEFLATE text/html text/plain text/xml
    AddOutputFilterByType DEFLATE text/css text/javascript
    AddOutputFilterByType DEFLATE application/javascript application/json

    # 압축 제외 (이미 압축된 파일)
    SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png|ico)$ no-gzip

    # 압축 레벨 (1-9)
    DeflateCompressionLevel 6
</IfModule>
```

#### mod_headers (헤더 조작)

```apache
<IfModule mod_headers.c>
    # 보안 헤더
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Content-Security-Policy "default-src 'self'"

    # 캐시 제어
    <FilesMatch "\.(ico|pdf|flv|jpg|jpeg|png|gif|js|css|swf)$">
        Header set Cache-Control "max-age=2592000, public"
    </FilesMatch>

    # 서버 정보 제거
    Header unset Server
    Header unset X-Powered-By
</IfModule>
```

### 모듈 관리 명령어

```bash
# 사용 가능한 모듈 목록 (Debian/Ubuntu)
apache2ctl -M

# 모듈 활성화/비활성화
sudo a2enmod rewrite
sudo a2dismod autoindex

# 설정 테스트
sudo apachectl configtest

# 재시작
sudo systemctl restart apache2
```

## .htaccess

- .htaccess는 디렉터리 수준에서 Apache 설정을 오버라이드할 수 있는 분산 설정 파일임.

### 기본 구조

```apache
# /var/www/html/.htaccess

# 기본 설정
Options -Indexes +FollowSymLinks
DirectoryIndex index.php index.html

# 에러 페이지
ErrorDocument 404 /errors/404.html
ErrorDocument 500 /errors/500.html
```

### .htaccess 활성화

```apache
# httpd.conf 또는 사이트 설정
<Directory /var/www/html>
    # AllowOverride 설정으로 .htaccess 허용
    AllowOverride All
    # 또는 특정 기능만 허용
    # AllowOverride AuthConfig Indexes Limit FileInfo
    Require all granted
</Directory>
```

### AllowOverride 옵션

- 옵션: `None`:
  - 허용하는 지시어: .htaccess 무시
- 옵션: `All`:
  - 허용하는 지시어: 모든 지시어 허용
- 옵션: `AuthConfig`:
  - 허용하는 지시어: 인증 관련 지시어
- 옵션: `FileInfo`:
  - 허용하는 지시어: MIME 타입, ErrorDocument 등
- 옵션: `Indexes`:
  - 허용하는 지시어: DirectoryIndex, Options Indexes 등
- 옵션: `Limit`:
  - 허용하는 지시어: Allow, Deny, Order 등

### 보안 설정 예제

```apache
# 디렉토리 목록 비활성화
Options -Indexes

# .htaccess 및 민감한 파일 보호
<FilesMatch "^\.ht">
    Require all denied
</FilesMatch>

<FilesMatch "(^#.*#|\.(bak|config|dist|fla|inc|ini|log|psd|sh|sql|sw[op])|~)$">
    Require all denied
</FilesMatch>

# 특정 IP만 허용
<RequireAll>
    Require ip 192.168.1.0/24
    Require ip 10.0.0.0/8
</RequireAll>

# 특정 IP 차단
<RequireAll>
    Require all granted
    Require not ip 192.168.1.100
</RequireAll>
```

### HTTP to HTTPS 리다이렉트

```apache
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

### 비밀번호 보호

```apache
# .htaccess
AuthType Basic
AuthName "Restricted Area"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user

# .htpasswd 생성
# htpasswd -c /etc/apache2/.htpasswd username
```

### 성능 고려사항

`.htaccess`를 허용하면 Apache가 요청마다 디렉터리 트리를 따라 설정 파일을 찾는다. 메인 설정을 수정할 수 있는 환경이라면 아래처럼 `httpd.conf`의 `<Directory>` 블록에 설정하고 `.htaccess` 탐색을 끄는 방법을 고려할 수 있다.

```apache
# 권장: httpd.conf에서 직접 설정
<Directory /var/www/html>
    AllowOverride None  # .htaccess 비활성화
    Options -Indexes +FollowSymLinks
    # 필요한 설정을 여기에 직접 작성
</Directory>
```

## mod_rewrite

- mod_rewrite는 URL 재작성을 위한 강력한 모듈임.

### 기본 문법

```apache
RewriteEngine On

# 기본 형식
RewriteRule 패턴 대상 [플래그]

# 조건과 함께
RewriteCond 테스트문자열 조건패턴 [플래그]
RewriteRule 패턴 대상 [플래그]
```

### RewriteRule 예제

```apache
# 기본 리다이렉트
RewriteRule ^old-page\.html$ /new-page.html [R=301,L]

# 동적 URL을 정적 URL로
# /product/123 → /product.php?id=123
RewriteRule ^product/([0-9]+)/?$ product.php?id=$1 [L,QSA]

# .php 확장자 숨기기
# /about → /about.php
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_FILENAME}.php -f
RewriteRule ^(.*)$ $1.php [L]

# www 추가
RewriteCond %{HTTP_HOST} !^www\. [NC]
RewriteRule ^(.*)$ http://www.%{HTTP_HOST}/$1 [R=301,L]

# www 제거
RewriteCond %{HTTP_HOST} ^www\.(.*)$ [NC]
RewriteRule ^(.*)$ http://%1/$1 [R=301,L]
```

### RewriteCond 조건

```apache
# 파일이 존재하지 않을 때만
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ index.php?q=$1 [L,QSA]

# HTTPS가 아닐 때
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# 특정 User-Agent 차단
RewriteCond %{HTTP_USER_AGENT} (bot|crawler|spider) [NC]
RewriteRule .* - [F,L]

# 특정 시간대만 허용
RewriteCond %{TIME_HOUR}%{TIME_MIN} >0800
RewriteCond %{TIME_HOUR}%{TIME_MIN} <1800
RewriteRule ^admin - [L]
RewriteRule ^admin - [F]
```

### 주요 플래그

- 플래그: `L`:
  - 의미: Last
  - 설명: 이 규칙이 마지막, 이후 규칙 무시
- 플래그: `R=301`:
  - 의미: Redirect
  - 설명: 301 리다이렉트
- 플래그: `R=302`:
  - 의미: Redirect
  - 설명: 302 임시 리다이렉트
- 플래그: `NC`:
  - 의미: No Case
  - 설명: 대소문자 무시
- 플래그: `QSA`:
  - 의미: Query String Append
  - 설명: 기존 쿼리스트링 유지
- 플래그: `F`:
  - 의미: Forbidden
  - 설명: 403 에러 반환
- 플래그: `G`:
  - 의미: Gone
  - 설명: 410 에러 반환
- 플래그: `NE`:
  - 의미: No Escape
  - 설명: URL 인코딩 방지
- 플래그: `PT`:
  - 의미: Pass Through
  - 설명: 다른 핸들러로 전달

### 실용적인 예제들

#### SPA (Single Page Application) 라우팅

```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /

    # 실제 파일/디렉토리가 있으면 그대로 제공
    RewriteCond %{REQUEST_FILENAME} -f [OR]
    RewriteCond %{REQUEST_FILENAME} -d
    RewriteRule ^ - [L]

    # 나머지는 index.html로
    RewriteRule ^ index.html [L]
</IfModule>
```

#### API 버전 라우팅

```apache
# /api/v1/users → /api/v1/index.php?endpoint=users
RewriteRule ^api/v([0-9]+)/(.*)$ api/v$1/index.php?endpoint=$2 [L,QSA]
```

#### 핫링크 방지

```apache
RewriteCond %{HTTP_REFERER} !^$
RewriteCond %{HTTP_REFERER} !^https?://(www\.)?example\.com [NC]
RewriteRule \.(jpg|jpeg|png|gif|webp)$ - [F,NC]
```

#### 언어별 리다이렉트

```apache
# 브라우저 언어에 따라 리다이렉트
RewriteCond %{HTTP:Accept-Language} ^ko [NC]
RewriteRule ^$ /ko/ [L,R=302]

RewriteCond %{HTTP:Accept-Language} ^ja [NC]
RewriteRule ^$ /ja/ [L,R=302]

RewriteRule ^$ /en/ [L,R=302]
```

## Apache vs Nginx 비교

### 아키텍처 비교

- [Apache]
- Master Process
- Process, Process, Process
- or, or, or
- Thread, Thread, Thread
- Req1, Req2, Req3
- [Nginx]
- Master Process
- Worker, Worker, Worker
- Event, Event, Event
- Loop, Loop, Loop
- Req1-N, ReqN+1, Req...

### 상세 비교표

- 특성: 아키텍처:
  - Apache: 프로세스/스레드 기반
  - Nginx: 이벤트 기반
- 특성: 동시 연결:
  - Apache: 수천
  - Nginx: 수만~수십만
- 특성: 메모리 사용:
  - Apache: 높음
  - Nginx: 낮음
- 특성: 정적 파일:
  - Apache: 상대적으로 느림
  - Nginx: 매우 빠름
- 특성: 동적 콘텐츠:
  - Apache: mod_php 내장 가능
  - Nginx: 외부 프로세서 필요
- 특성: 설정 유연성:
  - Apache: .htaccess (런타임)
  - Nginx: 설정 변경 시 reload
- 특성: 모듈 로딩:
  - Apache: 런타임 가능 (DSO)
  - Nginx: 컴파일 시점
- 특성: 문서/생태계:
  - Apache: 매우 풍부
  - Nginx: 점점 증가
- 특성: 학습 곡선:
  - Apache: 완만함
  - Nginx: 상대적으로 가파름

### 사용 사례별 권장

- Use Case Matrix
- Apache 권장:
- 레거시 PHP 애플리케이션 (mod_php)
- 공유 호스팅 환경 (.htaccess 필요)
- 복잡한 URL 재작성 규칙
- 다양한 모듈 필요 (mod_security 등)
- Nginx 권장:
- 고성능 리버스 프록시
- 정적 파일 서빙
- 마이크로서비스 API 게이트웨이
- 컨테이너 환경 (가벼움)
- 동시 연결이 많은 서비스
- 하이브리드 (Apache + Nginx):
- Nginx: 프론트 (정적 파일, SSL, 캐싱)
- Apache: 백엔드 (동적 콘텐츠, .htaccess)

### 성능 벤치마크 (일반적 경향)

- 정적 파일 제공 (동시 1000 연결)
- Nginx, ████████████████████████ 15000/s
- Apache ████████████ 7500/s
- PHP (PHP-FPM, 동시 100 연결)
- Nginx, ████████████████ 800/s
- Apache ██████████████ 700/s
- 메모리 사용량 (1000 동시 연결)
- Apache ████████████████████ ~500MB
- Nginx, ████ ~50MB

## 참고 자료

- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/2.4/)
- [Apache MPM Documentation](https://httpd.apache.org/docs/2.4/mpm.html)
- [mod_rewrite Documentation](https://httpd.apache.org/docs/current/mod/mod_rewrite.html)
- [Apache Security Tips](https://httpd.apache.org/docs/2.4/misc/security_tips.html)
- [Red Hat - Apache MPM Comparison](https://access.redhat.com/solutions/40234)

## 관련 학습

- [Nginx](01-Nginx.md)
- [Caddy](03-Caddy.md)

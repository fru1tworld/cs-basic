# Apache HTTP Server

## 목차
1. [Apache 개요](#apache-개요)
2. [MPM (Multi-Processing Module)](#mpm-multi-processing-module)
3. [모듈 시스템](#모듈-시스템)
4. [.htaccess](#htaccess)
5. [mod_rewrite](#mod_rewrite)
6. [Apache vs Nginx 비교](#apache-vs-nginx-비교)
---

## Apache 개요

Apache HTTP Server는 1995년에 시작된 오픈소스 웹 서버로, 오랫동안 세계에서 가장 널리 사용되는 웹 서버였습니다. **모듈식 아키텍처**와 **.htaccess** 기반의 유연한 설정이 특징입니다.

### 주요 특징
- **모듈식 아키텍처**: 필요한 기능만 로드하여 사용
- **유연한 설정**: .htaccess로 디렉토리별 설정 가능
- **광범위한 호환성**: 다양한 스크립팅 언어 지원 (PHP, Perl, Python 등)
- **풍부한 문서**: 20년 이상의 축적된 자료

### 설정 파일 구조

```
/etc/httpd/                     # RHEL/CentOS
/etc/apache2/                   # Debian/Ubuntu
├── apache2.conf                # 메인 설정 파일
├── ports.conf                  # 포트 설정
├── mods-available/             # 사용 가능한 모듈
├── mods-enabled/               # 활성화된 모듈 (심볼릭 링크)
├── sites-available/            # 사용 가능한 가상 호스트
├── sites-enabled/              # 활성화된 가상 호스트 (심볼릭 링크)
└── conf-available/             # 추가 설정
```

---

## MPM (Multi-Processing Module)

Apache는 **MPM**을 통해 요청 처리 방식을 결정합니다. MPM은 Apache가 클라이언트 연결을 어떻게 처리할지 정의하는 핵심 모듈입니다.

### Prefork MPM

**각 연결을 별도의 프로세스**로 처리하는 전통적인 방식입니다.

```
                    ┌─────────────────┐
                    │  Master Process │
                    │  (root 권한)     │
                    └────────┬────────┘
                             │
        ┌──────────┬────────┼────────┬──────────┐
        ▼          ▼        ▼        ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Child   │ │ Child   │ │ Child   │ │ Child   │ │ Child   │
   │ Process │ │ Process │ │ Process │ │ Process │ │ Process │
   │         │ │         │ │         │ │         │ │         │
   │ Conn 1  │ │ Conn 2  │ │ Conn 3  │ │ Conn 4  │ │ Conn 5  │
   └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

#### 특징
- **안정성**: 프로세스 격리로 하나의 요청 실패가 다른 요청에 영향 없음
- **호환성**: 스레드 안전하지 않은 라이브러리와도 호환
- **메모리 사용량**: 높음 (프로세스당 ~10-50MB)
- **PHP mod_php와 함께 사용 시 필수**

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

```
필요 메모리 = MaxRequestWorkers × 프로세스당 메모리

예: 250 × 50MB = 12.5GB

# 권장: 시스템 메모리의 70-80% 이내로 설정
# 8GB 서버 → MaxRequestWorkers ≈ 100-120
```

### Worker MPM

**멀티프로세스 + 멀티스레드** 하이브리드 방식입니다.

```
                    ┌─────────────────┐
                    │  Master Process │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │Child Process│     │Child Process│     │Child Process│
   │             │     │             │     │             │
   │ ┌─────────┐ │     │ ┌─────────┐ │     │ ┌─────────┐ │
   │ │Thread 1 │ │     │ │Thread 1 │ │     │ │Thread 1 │ │
   │ │Thread 2 │ │     │ │Thread 2 │ │     │ │Thread 2 │ │
   │ │Thread 3 │ │     │ │Thread 3 │ │     │ │Thread 3 │ │
   │ │  ...    │ │     │ │  ...    │ │     │ │  ...    │ │
   │ │Thread N │ │     │ │Thread N │ │     │ │Thread N │ │
   │ └─────────┘ │     │ └─────────┘ │     │ └─────────┘ │
   └─────────────┘     └─────────────┘     └─────────────┘
```

#### 특징
- **효율성**: Prefork보다 적은 메모리로 더 많은 연결 처리
- **스레드 공유**: 같은 프로세스 내 스레드들이 메모리 공유
- **제약**: 스레드 안전한 라이브러리만 사용 가능
- **PHP-FPM과 함께 사용**

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

**Apache 2.4 기본 MPM**으로, Worker MPM을 개선한 이벤트 기반 모델입니다.

```
                    ┌─────────────────┐
                    │  Master Process │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │Child Process│     │Child Process│     │Child Process│
   │             │     │             │     │             │
   │ ┌─────────┐ │     │ ┌─────────┐ │     │ ┌─────────┐ │
   │ │Listener │─┼─────┼─│Listener │─┼─────┼─│Listener │ │
   │ │ Thread  │ │     │ │ Thread  │ │     │ │ Thread  │ │
   │ └────┬────┘ │     │ └────┬────┘ │     │ └────┬────┘ │
   │      │      │     │      │      │     │      │      │
   │ ┌────▼────┐ │     │ ┌────▼────┐ │     │ ┌────▼────┐ │
   │ │Worker   │ │     │ │Worker   │ │     │ │Worker   │ │
   │ │Threads  │ │     │ │Threads  │ │     │ │Threads  │ │
   │ └─────────┘ │     │ └─────────┘ │     │ └─────────┘ │
   └─────────────┘     └─────────────┘     └─────────────┘
```

#### 핵심 개선점: Keep-Alive 처리

```
[Worker MPM]
스레드 1: 요청 처리 → Keep-Alive 대기 (idle) → 다음 요청
         └── 대기 중에도 스레드가 점유됨

[Event MPM]
리스너 스레드: Keep-Alive 연결 관리
워커 스레드 1: 요청 처리 → 즉시 해제 → 다른 요청 처리
             └── Keep-Alive는 리스너가 관리, 워커는 바로 해제
```

#### 특징
- **Keep-Alive 효율화**: 전용 리스너 스레드가 유휴 연결 관리
- **높은 동시성**: Worker보다 더 많은 동시 연결 처리 가능
- **Nginx와 유사**: 이벤트 기반 접근 방식
- **현대적 선택**: PHP-FPM과 함께 권장

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

| 특성 | Prefork | Worker | Event |
|------|---------|--------|-------|
| 처리 방식 | 프로세스 | 프로세스+스레드 | 이벤트 기반 |
| 메모리 효율 | 낮음 | 중간 | 높음 |
| 동시 연결 | 낮음 | 중간 | 높음 |
| Keep-Alive | 비효율적 | 비효율적 | 효율적 |
| PHP 호환 | mod_php | PHP-FPM | PHP-FPM |
| 안정성 | 높음 | 중간 | 중간 |
| 사용 사례 | 레거시, mod_php | 일반 | 고성능 (권장) |

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

---

## 모듈 시스템

Apache의 강력한 기능은 **모듈**을 통해 제공됩니다.

### 핵심 모듈 분류

```
┌─────────────────────────────────────────────────────────────┐
│                      Apache HTTP Server                       │
├─────────────────────────────────────────────────────────────┤
│  Core Modules (핵심)                                          │
│  - core, http_core, mod_so                                   │
├─────────────────────────────────────────────────────────────┤
│  MPM Modules (멀티프로세싱)                                    │
│  - mpm_prefork, mpm_worker, mpm_event                        │
├─────────────────────────────────────────────────────────────┤
│  Auth Modules (인증)                                          │
│  - mod_auth_basic, mod_auth_digest, mod_authn_file           │
├─────────────────────────────────────────────────────────────┤
│  Access Modules (접근 제어)                                    │
│  - mod_authz_host, mod_authz_user, mod_access_compat         │
├─────────────────────────────────────────────────────────────┤
│  Handler Modules (콘텐츠 처리)                                 │
│  - mod_cgi, mod_php, mod_proxy, mod_proxy_fcgi               │
├─────────────────────────────────────────────────────────────┤
│  Filter Modules (필터링)                                       │
│  - mod_deflate, mod_filter, mod_substitute                   │
├─────────────────────────────────────────────────────────────┤
│  Logging Modules (로깅)                                        │
│  - mod_log_config, mod_logio                                 │
├─────────────────────────────────────────────────────────────┤
│  Security Modules (보안)                                       │
│  - mod_ssl, mod_security, mod_evasive                        │
└─────────────────────────────────────────────────────────────┘
```

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

---

## .htaccess

**.htaccess**는 디렉토리 수준에서 Apache 설정을 오버라이드할 수 있는 분산 설정 파일입니다.

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

| 옵션 | 허용하는 지시어 |
|------|---------------|
| `None` | .htaccess 무시 |
| `All` | 모든 지시어 허용 |
| `AuthConfig` | 인증 관련 지시어 |
| `FileInfo` | MIME 타입, ErrorDocument 등 |
| `Indexes` | DirectoryIndex, Options Indexes 등 |
| `Limit` | Allow, Deny, Order 등 |

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

> **.htaccess 사용 시 성능 저하**
>
> Apache는 모든 요청마다 디렉토리 트리를 탐색하며 .htaccess를 찾습니다.
> 가능하면 **httpd.conf**의 `<Directory>` 블록에 직접 설정하는 것이 좋습니다.

```apache
# 권장: httpd.conf에서 직접 설정
<Directory /var/www/html>
    AllowOverride None  # .htaccess 비활성화
    Options -Indexes +FollowSymLinks
    # 필요한 설정을 여기에 직접 작성
</Directory>
```

---

## mod_rewrite

**mod_rewrite**는 URL 재작성을 위한 강력한 모듈입니다.

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

| 플래그 | 의미 | 설명 |
|-------|------|------|
| `L` | Last | 이 규칙이 마지막, 이후 규칙 무시 |
| `R=301` | Redirect | 301 리다이렉트 |
| `R=302` | Redirect | 302 임시 리다이렉트 |
| `NC` | No Case | 대소문자 무시 |
| `QSA` | Query String Append | 기존 쿼리스트링 유지 |
| `F` | Forbidden | 403 에러 반환 |
| `G` | Gone | 410 에러 반환 |
| `NE` | No Escape | URL 인코딩 방지 |
| `PT` | Pass Through | 다른 핸들러로 전달 |

### 실용적인 예제들

#### 1. SPA (Single Page Application) 라우팅

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

#### 2. API 버전 라우팅

```apache
# /api/v1/users → /api/v1/index.php?endpoint=users
RewriteRule ^api/v([0-9]+)/(.*)$ api/v$1/index.php?endpoint=$2 [L,QSA]
```

#### 3. 핫링크 방지

```apache
RewriteCond %{HTTP_REFERER} !^$
RewriteCond %{HTTP_REFERER} !^https?://(www\.)?example\.com [NC]
RewriteRule \.(jpg|jpeg|png|gif|webp)$ - [F,NC]
```

#### 4. 언어별 리다이렉트

```apache
# 브라우저 언어에 따라 리다이렉트
RewriteCond %{HTTP:Accept-Language} ^ko [NC]
RewriteRule ^$ /ko/ [L,R=302]

RewriteCond %{HTTP:Accept-Language} ^ja [NC]
RewriteRule ^$ /ja/ [L,R=302]

RewriteRule ^$ /en/ [L,R=302]
```

---

## Apache vs Nginx 비교

### 아키텍처 비교

```
[Apache]
┌──────────────────────────────────────┐
│          Master Process              │
└──────────────┬───────────────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌───────┐  ┌───────┐  ┌───────┐
│Process│  │Process│  │Process│
│ or    │  │ or    │  │ or    │
│Thread │  │Thread │  │Thread │
└───────┘  └───────┘  └───────┘
    │          │          │
   Req1       Req2       Req3

[Nginx]
┌──────────────────────────────────────┐
│          Master Process              │
└──────────────┬───────────────────────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌───────┐  ┌───────┐  ┌───────┐
│Worker │  │Worker │  │Worker │
│       │  │       │  │       │
│Event  │  │Event  │  │Event  │
│Loop   │  │Loop   │  │Loop   │
│       │  │       │  │       │
│Req1-N │  │ReqN+1 │  │Req... │
└───────┘  └───────┘  └───────┘
```

### 상세 비교표

| 특성 | Apache | Nginx |
|------|--------|-------|
| **아키텍처** | 프로세스/스레드 기반 | 이벤트 기반 |
| **동시 연결** | 수천 | 수만~수십만 |
| **메모리 사용** | 높음 | 낮음 |
| **정적 파일** | 상대적으로 느림 | 매우 빠름 |
| **동적 콘텐츠** | mod_php 내장 가능 | 외부 프로세서 필요 |
| **설정 유연성** | .htaccess (런타임) | 설정 변경 시 reload |
| **모듈 로딩** | 런타임 가능 (DSO) | 컴파일 시점 |
| **문서/생태계** | 매우 풍부 | 점점 증가 |
| **학습 곡선** | 완만함 | 상대적으로 가파름 |

### 사용 사례별 권장

```
┌─────────────────────────────────────────────────────────────┐
│                    Use Case Matrix                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Apache 권장:                                                │
│  ├─ 레거시 PHP 애플리케이션 (mod_php)                         │
│  ├─ 공유 호스팅 환경 (.htaccess 필요)                         │
│  ├─ 복잡한 URL 재작성 규칙                                    │
│  └─ 다양한 모듈 필요 (mod_security 등)                        │
│                                                              │
│  Nginx 권장:                                                 │
│  ├─ 고성능 리버스 프록시                                      │
│  ├─ 정적 파일 서빙                                           │
│  ├─ 마이크로서비스 API 게이트웨이                             │
│  ├─ 컨테이너 환경 (가벼움)                                    │
│  └─ 동시 연결이 많은 서비스                                   │
│                                                              │
│  하이브리드 (Apache + Nginx):                                │
│  ├─ Nginx: 프론트 (정적 파일, SSL, 캐싱)                      │
│  └─ Apache: 백엔드 (동적 콘텐츠, .htaccess)                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 성능 벤치마크 (일반적 경향)

```
정적 파일 제공 (동시 1000 연결)
┌──────────────────────────────────────────┐
│ Nginx  ████████████████████████ 15000/s  │
│ Apache ████████████ 7500/s               │
└──────────────────────────────────────────┘

PHP (PHP-FPM, 동시 100 연결)
┌──────────────────────────────────────────┐
│ Nginx  ████████████████ 800/s            │
│ Apache ██████████████ 700/s              │
└──────────────────────────────────────────┘

메모리 사용량 (1000 동시 연결)
┌──────────────────────────────────────────┐
│ Apache ████████████████████ ~500MB       │
│ Nginx  ████ ~50MB                        │
└──────────────────────────────────────────┘
```

---

## 참고 자료

- [Apache HTTP Server Documentation](https://httpd.apache.org/docs/2.4/)
- [Apache MPM Documentation](https://httpd.apache.org/docs/2.4/mpm.html)
- [mod_rewrite Documentation](https://httpd.apache.org/docs/current/mod/mod_rewrite.html)
- [Apache Security Tips](https://httpd.apache.org/docs/2.4/misc/security_tips.html)
- [Red Hat - Apache MPM Comparison](https://access.redhat.com/solutions/40234)

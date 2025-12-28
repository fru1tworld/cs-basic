# HTTP와 HTTPS

> 웹의 핵심 프로토콜인 HTTP의 발전 과정과 보안 메커니즘을 다룹니다.

## 목차
1. [HTTP 버전별 비교](#1-http-버전별-비교)
2. [HTTP 메서드와 상태 코드](#2-http-메서드와-상태-코드)
3. [헤더 구조와 주요 헤더](#3-헤더-구조와-주요-헤더)
4. [HTTPS와 TLS Handshake](#4-https와-tls-handshake)
5. [HTTP/3와 QUIC 프로토콜](#5-http3와-quic-프로토콜)
---

## 1. HTTP 버전별 비교

### 1.1 HTTP 발전 역사

```
1991: HTTP/0.9 - 단순 GET 요청만 지원
1996: HTTP/1.0 - RFC 1945
1997: HTTP/1.1 - RFC 2068 → RFC 2616 → RFC 7230-7235 (2014)
2015: HTTP/2   - RFC 7540 → RFC 9113 (2022)
2022: HTTP/3   - RFC 9114 (QUIC 기반)
```

### 1.2 HTTP/1.0 (RFC 1945)

**특징:**
- 요청당 새로운 TCP 연결 (비연결형)
- 상태 코드, 헤더 개념 도입
- Content-Type을 통한 다양한 미디어 타입 지원

```http
GET /index.html HTTP/1.0
Host: www.example.com
User-Agent: Mozilla/5.0

HTTP/1.0 200 OK
Content-Type: text/html
Content-Length: 1234

<html>...</html>
```

**문제점:**
```
요청 1: TCP 연결 → 요청 → 응답 → TCP 종료
요청 2: TCP 연결 → 요청 → 응답 → TCP 종료
요청 3: TCP 연결 → 요청 → 응답 → TCP 종료

→ 매 요청마다 3-way handshake 오버헤드 발생
```

### 1.3 HTTP/1.1 (RFC 7230-7235)

**주요 개선 사항:**

```
1. Persistent Connection (Keep-Alive)
   - 기본적으로 연결 유지
   - Connection: close로 명시적 종료

2. Pipelining
   - 응답 대기 없이 여러 요청 전송 가능
   - 하지만 HOL Blocking 문제 존재

3. Host 헤더 필수화
   - 가상 호스팅 지원

4. Chunked Transfer Encoding
   - Content-Length 없이 스트리밍 가능

5. 추가 메서드
   - PUT, PATCH, DELETE, OPTIONS, TRACE, CONNECT
```

**HTTP/1.1 Keep-Alive:**
```
┌────────┐                              ┌────────┐
│ Client │                              │ Server │
└───┬────┘                              └───┬────┘
    │                                       │
    │ ─────── TCP 3-way Handshake ──────►  │
    │                                       │
    │ ─────── Request 1 (GET /a.html) ───► │
    │ ◄────── Response 1 ─────────────────  │
    │                                       │
    │ ─────── Request 2 (GET /b.css) ────► │
    │ ◄────── Response 2 ─────────────────  │
    │                                       │
    │ ─────── Request 3 (GET /c.js) ─────► │
    │ ◄────── Response 3 ─────────────────  │
    │                                       │
    │ ─────── Connection: close ─────────► │
    │                                       │
```

**HTTP/1.1 Pipelining의 HOL Blocking:**
```
Pipelining 시도:
    │ ─────── Request 1 ───────────────►  │
    │ ─────── Request 2 ───────────────►  │
    │ ─────── Request 3 ───────────────►  │

응답은 순서대로만 가능:
    │ ◄────── Response 1 (느림) ─────────  │ ← Request 2,3이 대기
    │ ◄────── Response 2 ─────────────────  │
    │ ◄────── Response 3 ─────────────────  │

→ Response 1이 느리면 나머지도 지연됨 (Head-of-Line Blocking)
```

**실무에서의 HTTP/1.1 최적화 기법:**
```html
<!-- Domain Sharding: 여러 도메인으로 병렬 연결 -->
<img src="https://static1.example.com/image1.jpg">
<img src="https://static2.example.com/image2.jpg">
<img src="https://static3.example.com/image3.jpg">

<!-- 브라우저는 도메인당 6-8개 동시 연결 허용 -->

<!-- 리소스 번들링 -->
<script src="bundle.min.js"></script>  <!-- 여러 JS를 하나로 -->
<link rel="stylesheet" href="styles.min.css">  <!-- 여러 CSS를 하나로 -->

<!-- 이미지 스프라이트 -->
<div class="icon icon-home"></div>
<style>
.icon { background-image: url('sprite.png'); }
.icon-home { background-position: 0 0; }
.icon-user { background-position: -20px 0; }
</style>
```

### 1.4 HTTP/2 (RFC 9113)

**핵심 개념: 바이너리 프레이밍 계층**

```
HTTP/1.1: 텍스트 기반
GET /resource HTTP/1.1\r\n
Host: example.com\r\n
\r\n

HTTP/2: 바이너리 프레임
┌─────────────────────────────────────────┐
│ Length (24bit) │ Type (8bit) │ Flags    │
├─────────────────────────────────────────┤
│ R │           Stream Identifier (31bit) │
├─────────────────────────────────────────┤
│                Payload                   │
└─────────────────────────────────────────┘
```

**HTTP/2 주요 기능:**

```
1. Multiplexing (다중화)
┌─────────────────────────────────────────────────────────┐
│                    단일 TCP 연결                         │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐           │
│  │Str 1│  │Str 3│  │Str 1│  │Str 5│  │Str 3│  ...      │
│  │Frame│  │Frame│  │Frame│  │Frame│  │Frame│           │
│  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘           │
└─────────────────────────────────────────────────────────┘

→ 여러 스트림이 하나의 연결에서 인터리빙됨
→ HOL Blocking 해결 (HTTP 레벨에서는)

2. Header Compression (HPACK)
┌──────────────────────────────────────────────────────────┐
│ 기존 HTTP/1.1 (반복되는 헤더가 매번 전송됨):              │
│ Request 1: Host: example.com, Accept: text/html, ...     │
│ Request 2: Host: example.com, Accept: text/html, ...     │
│ Request 3: Host: example.com, Accept: text/html, ...     │
├──────────────────────────────────────────────────────────┤
│ HTTP/2 HPACK (인덱스로 참조):                            │
│ Request 1: [Full Headers] → Static/Dynamic Table 구축    │
│ Request 2: [Index 62, Index 63, ...]                     │
│ Request 3: [Index 62, Index 63, ...]                     │
│                                                          │
│ 헤더 크기 85-95% 감소 가능                                │
└──────────────────────────────────────────────────────────┘

3. Server Push
┌────────┐                              ┌────────┐
│ Client │                              │ Server │
└───┬────┘                              └───┬────┘
    │                                       │
    │ ─────── GET /index.html ────────────► │
    │                                       │
    │ ◄────── PUSH_PROMISE (styles.css) ──  │
    │ ◄────── PUSH_PROMISE (app.js) ──────  │
    │ ◄────── Response (index.html) ──────  │
    │ ◄────── Response (styles.css) ──────  │
    │ ◄────── Response (app.js) ──────────  │
    │                                       │

→ 클라이언트가 요청하기 전에 필요한 리소스 미리 전송

4. Stream Prioritization
┌────────────────────────────────────────┐
│ Stream 1: Weight 256 (CSS - 높음)       │
│ Stream 3: Weight 128 (이미지 - 중간)    │
│ Stream 5: Weight 64  (분석 스크립트)    │
│                                        │
│ 의존성 트리로 우선순위 표현 가능          │
└────────────────────────────────────────┘
```

**HTTP/2 Python 예제 (httpx):**
```python
import httpx

# HTTP/2 클라이언트 사용
async def fetch_with_http2():
    async with httpx.AsyncClient(http2=True) as client:
        # 동시에 여러 요청 (단일 TCP 연결에서 멀티플렉싱)
        responses = await asyncio.gather(
            client.get("https://example.com/api/users"),
            client.get("https://example.com/api/posts"),
            client.get("https://example.com/api/comments"),
        )

        for r in responses:
            print(f"HTTP Version: {r.http_version}")  # HTTP/2
            print(f"Status: {r.status_code}")
```

**HTTP/2 Nginx 설정:**
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.crt;
    ssl_certificate_key /etc/ssl/private/example.key;

    # HTTP/2 Server Push
    location = /index.html {
        http2_push /css/styles.css;
        http2_push /js/app.js;
    }

    # HTTP/2 설정 튜닝
    http2_max_concurrent_streams 128;
    http2_idle_timeout 3m;
}
```

### 1.5 HTTP/1.1 vs HTTP/2 비교표

| 특성 | HTTP/1.1 | HTTP/2 |
|------|----------|--------|
| 프로토콜 형식 | 텍스트 | 바이너리 |
| 연결 | 도메인당 여러 TCP 연결 | 단일 TCP 연결 |
| 요청 처리 | 순차적 (HOL Blocking) | 병렬 (Multiplexing) |
| 헤더 압축 | 없음 | HPACK |
| 서버 푸시 | 없음 | 지원 |
| 우선순위 | 없음 | 스트림 우선순위 |
| TLS 필수 | 아니오 | 사실상 필수 (브라우저) |

---

## 2. HTTP 메서드와 상태 코드

### 2.1 HTTP 메서드 (RFC 9110)

```
┌──────────────────────────────────────────────────────────────────────┐
│  메서드  │  안전성  │  멱등성  │  캐시 가능  │         용도            │
├──────────────────────────────────────────────────────────────────────┤
│  GET     │    O    │    O    │     O      │ 리소스 조회              │
│  HEAD    │    O    │    O    │     O      │ 헤더만 조회              │
│  POST    │    X    │    X    │   조건부   │ 리소스 생성, 처리        │
│  PUT     │    X    │    O    │     X      │ 리소스 전체 교체         │
│  PATCH   │    X    │    X    │     X      │ 리소스 부분 수정         │
│  DELETE  │    X    │    O    │     X      │ 리소스 삭제              │
│  OPTIONS │    O    │    O    │     X      │ 지원 메서드 확인 (CORS)  │
│  TRACE   │    O    │    O    │     X      │ 루프백 테스트            │
│  CONNECT │    X    │    X    │     X      │ 터널 설정 (프록시)       │
└──────────────────────────────────────────────────────────────────────┘

안전성(Safe): 서버 상태를 변경하지 않음
멱등성(Idempotent): 동일 요청을 여러 번 해도 결과가 같음
```

**멱등성(Idempotency) 상세 설명:**
```python
# 멱등한 작업의 예
class UserAPI:
    def get_user(self, user_id):
        """GET은 멱등함 - 여러 번 호출해도 같은 결과"""
        return self.db.find(user_id)

    def delete_user(self, user_id):
        """DELETE는 멱등함 - 이미 삭제된 걸 삭제해도 결과는 "없음""""
        # 첫 번째 호출: 사용자 삭제됨
        # 두 번째 호출: 이미 없음 (결과적으로 같은 상태)
        self.db.delete(user_id)

    def update_user(self, user_id, data):
        """PUT은 멱등함 - 전체 교체이므로 여러 번 해도 같은 결과"""
        self.db.replace(user_id, data)

# 멱등하지 않은 작업의 예
class OrderAPI:
    def create_order(self, order_data):
        """POST는 멱등하지 않음 - 호출할 때마다 새 주문 생성"""
        return self.db.insert(order_data)  # 매번 새 레코드

    def increment_view_count(self, post_id):
        """PATCH로 카운트 증가는 멱등하지 않음"""
        self.db.increment(post_id, 'views')  # 매번 증가
```

**RESTful API 설계 예시:**
```http
# 사용자 목록 조회
GET /api/v1/users HTTP/1.1

# 특정 사용자 조회
GET /api/v1/users/123 HTTP/1.1

# 사용자 생성
POST /api/v1/users HTTP/1.1
Content-Type: application/json

{"name": "John", "email": "john@example.com"}

# 사용자 정보 전체 수정 (멱등)
PUT /api/v1/users/123 HTTP/1.1
Content-Type: application/json

{"name": "John Updated", "email": "john.new@example.com", "age": 30}

# 사용자 정보 부분 수정 (주의: 구현에 따라 멱등성 달라짐)
PATCH /api/v1/users/123 HTTP/1.1
Content-Type: application/json

{"name": "John Patched"}

# 사용자 삭제
DELETE /api/v1/users/123 HTTP/1.1
```

### 2.2 HTTP 상태 코드 (RFC 9110)

```
1xx: Informational (정보)
├── 100 Continue          - 요청 계속 진행
├── 101 Switching Protocols - 프로토콜 전환 (WebSocket)
└── 103 Early Hints       - 리소스 힌트 (preload)

2xx: Success (성공)
├── 200 OK                - 성공
├── 201 Created           - 리소스 생성됨 (POST)
├── 202 Accepted          - 요청 수락됨 (비동기 처리)
├── 204 No Content        - 성공, 본문 없음 (DELETE)
└── 206 Partial Content   - 범위 요청 성공

3xx: Redirection (리다이렉션)
├── 301 Moved Permanently - 영구 이동 (GET으로 변경될 수 있음)
├── 302 Found             - 임시 이동 (GET으로 변경될 수 있음)
├── 303 See Other         - GET으로 다른 URI 조회
├── 304 Not Modified      - 캐시 사용 (조건부 요청)
├── 307 Temporary Redirect - 임시 이동 (메서드 유지)
└── 308 Permanent Redirect - 영구 이동 (메서드 유지)

4xx: Client Error (클라이언트 오류)
├── 400 Bad Request       - 잘못된 요청
├── 401 Unauthorized      - 인증 필요
├── 403 Forbidden         - 권한 없음
├── 404 Not Found         - 리소스 없음
├── 405 Method Not Allowed - 허용되지 않는 메서드
├── 408 Request Timeout   - 요청 타임아웃
├── 409 Conflict          - 충돌 (동시 수정 등)
├── 413 Payload Too Large - 요청 본문이 너무 큼
├── 415 Unsupported Media Type - 지원하지 않는 미디어 타입
├── 422 Unprocessable Entity - 문법은 맞지만 처리 불가
├── 429 Too Many Requests - 요청 횟수 초과 (Rate Limit)
└── 499 Client Closed Request - 클라이언트가 연결 종료 (Nginx)

5xx: Server Error (서버 오류)
├── 500 Internal Server Error - 서버 내부 오류
├── 501 Not Implemented   - 기능 미구현
├── 502 Bad Gateway       - 게이트웨이 오류
├── 503 Service Unavailable - 서비스 이용 불가
├── 504 Gateway Timeout   - 게이트웨이 타임아웃
└── 599 Network Connect Timeout - 네트워크 연결 타임아웃
```

**자주 혼동되는 상태 코드 비교:**

```python
# 301 vs 302 vs 307 vs 308 비교
class RedirectHandler:
    """
    리다이렉션 상태 코드의 차이점
    """

    def redirect_301(self):
        """
        301 Moved Permanently
        - 영구 이동
        - 브라우저가 캐시함
        - POST → GET으로 변경될 수 있음 (역사적 이유)
        - SEO: 검색 엔진이 새 URL로 인덱싱
        """
        # 사용 예: 도메인 변경, URL 구조 영구 변경
        return redirect('/new-url', code=301)

    def redirect_302(self):
        """
        302 Found
        - 임시 이동
        - 브라우저가 캐시하지 않음
        - POST → GET으로 변경될 수 있음
        - SEO: 원래 URL 유지
        """
        # 사용 예: 로그인 후 원래 페이지로
        return redirect('/temp-url', code=302)

    def redirect_307(self):
        """
        307 Temporary Redirect
        - 임시 이동
        - 메서드와 본문 유지 (POST는 POST로)
        - HTTP/1.1에서 302의 모호함 해결
        """
        # 사용 예: POST 요청을 다른 서버로 임시 전달
        return redirect('/temp-url', code=307)

    def redirect_308(self):
        """
        308 Permanent Redirect
        - 영구 이동
        - 메서드와 본문 유지
        - 301의 모호함 해결
        """
        # 사용 예: API 엔드포인트 영구 변경 (POST 유지 필요시)
        return redirect('/new-api-url', code=308)


# 401 vs 403 비교
class AuthErrorHandler:
    """
    인증/인가 오류 구분
    """

    def handle_401(self):
        """
        401 Unauthorized (인증 실패)
        - "당신이 누군지 모르겠다"
        - WWW-Authenticate 헤더와 함께 반환
        - 로그인하면 해결될 수 있음
        """
        # 예: 토큰 없음, 토큰 만료
        return {"error": "Authentication required"}, 401

    def handle_403(self):
        """
        403 Forbidden (인가 실패)
        - "당신이 누군지는 알지만, 권한이 없다"
        - 인증해도 해결 안 됨
        - 다른 계정이나 권한 필요
        """
        # 예: 관리자 전용 페이지에 일반 사용자 접근
        return {"error": "You don't have permission"}, 403
```

---

## 3. 헤더 구조와 주요 헤더

### 3.1 헤더 분류

```
┌──────────────────────────────────────────────────────────────────┐
│                        HTTP 헤더 분류                            │
├──────────────────────────────────────────────────────────────────┤
│ General Headers (일반 헤더)                                      │
│   - 요청과 응답 모두에 사용                                       │
│   - Cache-Control, Connection, Date, Transfer-Encoding          │
├──────────────────────────────────────────────────────────────────┤
│ Request Headers (요청 헤더)                                      │
│   - 클라이언트 → 서버 정보 전달                                   │
│   - Host, User-Agent, Accept, Authorization, Cookie             │
├──────────────────────────────────────────────────────────────────┤
│ Response Headers (응답 헤더)                                     │
│   - 서버 → 클라이언트 정보 전달                                   │
│   - Server, Set-Cookie, WWW-Authenticate, Location              │
├──────────────────────────────────────────────────────────────────┤
│ Entity/Representation Headers (엔티티 헤더)                      │
│   - 본문 관련 정보                                               │
│   - Content-Type, Content-Length, Content-Encoding, ETag        │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 주요 요청 헤더

```http
# 필수 헤더
Host: api.example.com                    # 요청 대상 호스트 (HTTP/1.1 필수)

# 콘텐츠 협상 (Content Negotiation)
Accept: application/json, text/html;q=0.9, */*;q=0.1
Accept-Language: ko-KR,ko;q=0.9,en-US;q=0.8
Accept-Encoding: gzip, deflate, br       # 지원하는 압축 방식
Accept-Charset: utf-8

# 인증
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Cookie: session_id=abc123; user_pref=dark

# 조건부 요청 (Conditional Requests)
If-Modified-Since: Wed, 21 Oct 2024 07:28:00 GMT
If-None-Match: "686897696a7c876b7e"      # ETag 값
If-Match: "686897696a7c876b7e"           # 동시성 제어

# 캐시 제어
Cache-Control: no-cache                   # 캐시 사용 전 검증 필요
Cache-Control: max-age=0                  # 즉시 만료

# 기타
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...
Referer: https://example.com/page        # 이전 페이지
Origin: https://example.com              # CORS 요청 출처
Range: bytes=0-1023                      # 부분 요청
```

### 3.3 주요 응답 헤더

```http
# 캐시 관련
Cache-Control: public, max-age=31536000  # 1년 캐시
Cache-Control: private, no-store         # 캐시 금지
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Last-Modified: Wed, 21 Oct 2024 07:28:00 GMT
Expires: Thu, 21 Oct 2025 07:28:00 GMT   # 레거시, Cache-Control 우선

# 콘텐츠 관련
Content-Type: application/json; charset=utf-8
Content-Length: 1234
Content-Encoding: gzip
Content-Disposition: attachment; filename="report.pdf"

# 보안 관련
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'
Referrer-Policy: strict-origin-when-cross-origin

# CORS 관련
Access-Control-Allow-Origin: https://trusted.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400            # Preflight 캐시 시간

# 기타
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
Location: https://example.com/new-resource  # 리다이렉션
Retry-After: 120                         # 429/503 응답 시 재시도 시간
```

### 3.4 Cache-Control 디렉티브 상세

```python
# Cache-Control 완벽 가이드
class CacheControlDirectives:
    """
    Cache-Control 헤더의 모든 디렉티브 설명
    """

    # === 캐시 가능 여부 ===

    PUBLIC = "public"
    # 모든 캐시(CDN, 프록시, 브라우저)에 저장 가능
    # 예: 정적 리소스, 공개 API 응답

    PRIVATE = "private"
    # 브라우저 캐시에만 저장 (CDN, 프록시 X)
    # 예: 사용자별 맞춤 데이터

    NO_STORE = "no-store"
    # 어디에도 캐시하지 않음
    # 예: 민감한 금융 정보

    # === 재검증 ===

    NO_CACHE = "no-cache"
    # 캐시 저장은 하지만, 사용 전 항상 서버 검증
    # 이름과 달리 캐시를 "사용하지 않는다"가 아님!

    MUST_REVALIDATE = "must-revalidate"
    # 만료 후 반드시 서버 검증 (오프라인 시 504 반환)

    PROXY_REVALIDATE = "proxy-revalidate"
    # 프록시/CDN만 재검증 필요

    # === 만료 시간 ===

    MAX_AGE = "max-age=3600"
    # 3600초(1시간) 동안 신선함

    S_MAXAGE = "s-maxage=86400"
    # 공유 캐시(CDN)에서의 만료 시간 (max-age 오버라이드)

    STALE_WHILE_REVALIDATE = "stale-while-revalidate=60"
    # 만료 후 60초간 오래된 캐시 제공하면서 백그라운드 갱신

    STALE_IF_ERROR = "stale-if-error=300"
    # 원본 서버 오류 시 300초간 오래된 캐시 제공

    # === 기타 ===

    IMMUTABLE = "immutable"
    # 절대 변경되지 않음 (버전 번호가 포함된 URL)

    NO_TRANSFORM = "no-transform"
    # 프록시가 콘텐츠 변환(이미지 압축 등) 금지


# 실제 사용 예시
CACHE_STRATEGIES = {
    "static_assets": "public, max-age=31536000, immutable",
    # 버전 해시가 포함된 정적 파일 (app.a1b2c3.js)

    "api_public": "public, max-age=300, stale-while-revalidate=60",
    # 공개 API, 5분 캐시 + 백그라운드 갱신

    "api_private": "private, max-age=60, must-revalidate",
    # 사용자별 API, 1분 캐시 + 엄격한 재검증

    "html_pages": "no-cache",
    # HTML 페이지, 항상 서버 확인 (ETag/Last-Modified 활용)

    "sensitive_data": "private, no-store",
    # 민감한 데이터, 캐시 금지
}
```

### 3.5 CORS (Cross-Origin Resource Sharing)

```
┌─────────────────────────────────────────────────────────────────┐
│                     CORS Preflight Flow                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   브라우저 (https://app.com)          서버 (https://api.com)    │
│          │                                    │                 │
│          │ ─── OPTIONS /api/data ───────────► │                 │
│          │     Origin: https://app.com        │                 │
│          │     Access-Control-Request-Method: POST              │
│          │     Access-Control-Request-Headers: Content-Type     │
│          │                                    │                 │
│          │ ◄── 200 OK ───────────────────────  │                │
│          │     Access-Control-Allow-Origin: https://app.com     │
│          │     Access-Control-Allow-Methods: GET, POST          │
│          │     Access-Control-Allow-Headers: Content-Type       │
│          │     Access-Control-Max-Age: 86400                    │
│          │                                    │                 │
│          │ ─── POST /api/data ──────────────► │                 │
│          │     Origin: https://app.com        │                 │
│          │     Content-Type: application/json │                 │
│          │                                    │                 │
│          │ ◄── 200 OK ───────────────────────  │                │
│          │     Access-Control-Allow-Origin: https://app.com     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Simple Request vs Preflight Request:**
```python
# Simple Request (Preflight 없음)
simple_conditions = """
Simple Request 조건 (모두 충족해야 함):
1. 메서드: GET, HEAD, POST 중 하나
2. 헤더: Accept, Accept-Language, Content-Language, Content-Type만
3. Content-Type: application/x-www-form-urlencoded,
                 multipart/form-data,
                 text/plain 중 하나
"""

# Preflight가 필요한 경우
preflight_triggers = """
Preflight 트리거:
1. PUT, DELETE, PATCH 등의 메서드
2. Custom 헤더 (Authorization, X-Custom-Header 등)
3. Content-Type: application/json
4. XMLHttpRequest.upload 사용
"""
```

**Express.js CORS 설정 예시:**
```javascript
const cors = require('cors');

// 기본 설정
app.use(cors());

// 상세 설정
const corsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.example.com',
      'https://admin.example.com'
    ];

    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-ID'],
  credentials: true,  // 쿠키 허용
  maxAge: 86400,      // Preflight 캐시 24시간
  exposedHeaders: ['X-Total-Count', 'X-Page-Count']  // 클라이언트 접근 허용
};

app.use(cors(corsOptions));
```

---

## 4. HTTPS와 TLS Handshake

### 4.1 HTTPS 개요

```
┌───────────────────────────────────────────────────────────────┐
│                      HTTPS = HTTP + TLS                       │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│   Application Layer      │  HTTP (GET, POST, ...)            │
│   ────────────────────────────────────────────────────        │
│   Security Layer         │  TLS (암호화, 인증, 무결성)         │
│   ────────────────────────────────────────────────────        │
│   Transport Layer        │  TCP                               │
│   ────────────────────────────────────────────────────        │
│   Network Layer          │  IP                                │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**TLS가 제공하는 보안:**
1. **기밀성 (Confidentiality)**: 대칭키 암호화로 데이터 보호
2. **무결성 (Integrity)**: MAC (Message Authentication Code)로 변조 감지
3. **인증 (Authentication)**: 인증서로 서버(선택적으로 클라이언트) 신원 확인

### 4.2 TLS 1.2 Handshake

```
┌────────────────────────────────────────────────────────────────────┐
│                     TLS 1.2 Full Handshake (2-RTT)                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Client                                              Server       │
│      │                                                   │         │
│      │ ─────── ClientHello ─────────────────────────────►│         │
│      │         (TLS version, cipher suites,              │  RTT 1  │
│      │          random, session_id, extensions)          │         │
│      │                                                   │         │
│      │ ◄─────── ServerHello ─────────────────────────────│         │
│      │          (selected cipher, random)                │         │
│      │ ◄─────── Certificate ─────────────────────────────│         │
│      │ ◄─────── ServerKeyExchange (ECDHE params) ────────│         │
│      │ ◄─────── ServerHelloDone ─────────────────────────│         │
│      │                                                   │         │
│      │ ─────── ClientKeyExchange ───────────────────────►│         │
│      │         (ECDHE public key)                        │  RTT 2  │
│      │ ─────── ChangeCipherSpec ────────────────────────►│         │
│      │ ─────── Finished (encrypted) ────────────────────►│         │
│      │                                                   │         │
│      │ ◄─────── ChangeCipherSpec ────────────────────────│         │
│      │ ◄─────── Finished (encrypted) ────────────────────│         │
│      │                                                   │         │
│      │ ═══════ Application Data (encrypted) ════════════►│         │
│      │ ◄═══════ Application Data (encrypted) ════════════│         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 4.3 TLS 1.3 Handshake (RFC 8446)

**TLS 1.3의 주요 개선사항:**
- 1-RTT Handshake (0-RTT 재연결 지원)
- 불안전한 암호 스위트 제거 (RSA 키 교환, SHA-1, RC4, DES 등)
- 핸드셰이크 메시지 암호화 (ServerHello 이후)
- Forward Secrecy 필수화

```
┌────────────────────────────────────────────────────────────────────┐
│                     TLS 1.3 Full Handshake (1-RTT)                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Client                                              Server       │
│      │                                                   │         │
│      │ ─────── ClientHello ─────────────────────────────►│         │
│      │         + key_share (ECDHE public key)            │         │
│      │         + supported_versions (TLS 1.3)            │         │
│      │         + signature_algorithms                    │         │
│      │         + psk_key_exchange_modes (optional)       │  RTT 1  │
│      │                                                   │         │
│      │ ◄─────── ServerHello ─────────────────────────────│         │
│      │          + key_share (ECDHE public key)           │         │
│      │          + supported_versions                     │         │
│      │ ◄─────── {EncryptedExtensions} ───────────────────│         │
│      │ ◄─────── {Certificate} ───────────────────────────│         │
│      │ ◄─────── {CertificateVerify} ─────────────────────│         │
│      │ ◄─────── {Finished} ──────────────────────────────│         │
│      │                                                   │         │
│      │ ─────── {Finished} ──────────────────────────────►│         │
│      │                                                   │         │
│      │ ═══════ [Application Data] ══════════════════════►│         │
│                                                                    │
│   { } = 암호화된 핸드셰이크 메시지                                   │
│   [ ] = 암호화된 애플리케이션 데이터                                 │
└────────────────────────────────────────────────────────────────────┘
```

**TLS 1.3 0-RTT Resumption:**
```
┌────────────────────────────────────────────────────────────────────┐
│                     TLS 1.3 0-RTT Resumption                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Client                                              Server       │
│      │                                                   │         │
│      │ ─────── ClientHello ─────────────────────────────►│         │
│      │         + early_data                              │         │
│      │         + pre_shared_key                          │         │
│      │ ─────── [0-RTT Application Data] ────────────────►│  0-RTT! │
│      │                                                   │         │
│      │ ◄─────── ServerHello ─────────────────────────────│         │
│      │ ◄─────── {EncryptedExtensions}                    │         │
│      │          + early_data (accepted)                  │         │
│      │ ◄─────── {Finished} ──────────────────────────────│         │
│      │                                                   │         │
│      │ ─────── {EndOfEarlyData} ────────────────────────►│         │
│      │ ─────── {Finished} ──────────────────────────────►│         │
│                                                                    │
│   주의: 0-RTT 데이터는 Replay 공격에 취약할 수 있음                  │
│   멱등하지 않은 요청에는 사용 주의                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 4.4 인증서 체인 검증

```
┌─────────────────────────────────────────────────────────────────┐
│                     Certificate Chain                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────┐                   │
│   │           Root CA Certificate            │ ← 브라우저/OS에   │
│   │         (Self-signed, 신뢰 앵커)         │   미리 내장됨     │
│   └──────────────────┬──────────────────────┘                   │
│                      │ signs                                    │
│                      ▼                                          │
│   ┌─────────────────────────────────────────┐                   │
│   │      Intermediate CA Certificate         │ ← 중간 인증서    │
│   │       (Root CA가 발급)                   │                  │
│   └──────────────────┬──────────────────────┘                   │
│                      │ signs                                    │
│                      ▼                                          │
│   ┌─────────────────────────────────────────┐                   │
│   │         End-Entity Certificate           │ ← 서버 인증서    │
│   │      (www.example.com)                   │                  │
│   └─────────────────────────────────────────┘                   │
│                                                                 │
│   검증 순서: End-Entity → Intermediate → Root (역방향)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**인증서 검증 과정:**
```python
# 인증서 검증 개념적 구현
class CertificateValidator:
    def __init__(self, trusted_roots):
        self.trusted_roots = trusted_roots  # 신뢰하는 Root CA 목록

    def validate_chain(self, cert_chain, hostname):
        """
        인증서 체인 검증

        Args:
            cert_chain: [서버 인증서, 중간 CA, ...]
            hostname: 접속하려는 도메인

        Returns:
            bool: 검증 성공 여부
        """
        checks = [
            self.check_not_expired(cert_chain),
            self.check_signature_chain(cert_chain),
            self.check_trusted_root(cert_chain),
            self.check_hostname(cert_chain[0], hostname),
            self.check_revocation(cert_chain),  # OCSP, CRL
            self.check_key_usage(cert_chain[0]),
        ]
        return all(checks)

    def check_not_expired(self, cert_chain):
        """모든 인증서의 유효 기간 확인"""
        for cert in cert_chain:
            if cert.not_before > now or cert.not_after < now:
                return False
        return True

    def check_signature_chain(self, cert_chain):
        """각 인증서가 상위 CA에 의해 서명되었는지 확인"""
        for i in range(len(cert_chain) - 1):
            if not cert_chain[i].verify_signature(cert_chain[i + 1].public_key):
                return False
        return True

    def check_hostname(self, server_cert, hostname):
        """
        인증서의 CN 또는 SAN이 hostname과 일치하는지 확인

        Subject Alternative Name (SAN):
        - DNS:www.example.com
        - DNS:*.example.com (와일드카드)
        """
        return hostname in server_cert.get_subject_alt_names()
```

### 4.5 TLS 1.2 vs TLS 1.3 비교

| 특성 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| Handshake RTT | 2-RTT | 1-RTT (0-RTT 재연결) |
| 암호화 시작 시점 | Finished 후 | ServerHello 직후 |
| Forward Secrecy | 선택적 | 필수 (ECDHE만) |
| RSA 키 교환 | 지원 | 제거됨 |
| 지원 암호 스위트 | 수백 개 | 5개 (AEAD만) |
| 메시지 암호화 | 일부 | 대부분 (ServerHello 제외) |
| 0-RTT | 미지원 | 지원 (재연결 시) |

**TLS 1.3 지원 암호 스위트 (RFC 8446):**
```
TLS_AES_128_GCM_SHA256
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_CCM_SHA256
TLS_AES_128_CCM_8_SHA256
```

---

## 5. HTTP/3와 QUIC 프로토콜

### 5.1 HTTP/2의 한계

```
HTTP/2 over TCP의 문제: TCP 레벨 Head-of-Line Blocking

┌─────────────────────────────────────────────────────────────────┐
│                      단일 TCP 연결                               │
│                                                                 │
│   Stream 1  Stream 2  Stream 3  Stream 4  Stream 5             │
│     ▼         ▼         ▼         ▼         ▼                  │
│   [pkt1]   [pkt2]   [pkt3]   [pkt4]   [pkt5]                   │
│     │         │         X         │         │                  │
│     │         │         │         │         │                  │
│     ▼         ▼         │         │         │                  │
│   [도착]    [도착]   [손실!]    [대기]    [대기]   ← TCP는 순서 보장 │
│                         │                                      │
│                    재전송 대기                                   │
│                                                                 │
│   Stream 3의 패킷 손실이 Stream 4, 5 전달도 차단!                │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 QUIC 프로토콜 (RFC 9000)

**QUIC = Quick UDP Internet Connections**

```
┌──────────────────────────────────────────────────────────────────┐
│                     Protocol Stack 비교                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│   HTTP/2 over TCP             HTTP/3 over QUIC                  │
│   ┌──────────────────┐        ┌──────────────────┐              │
│   │      HTTP/2      │        │      HTTP/3      │              │
│   ├──────────────────┤        ├──────────────────┤              │
│   │       TLS        │        │       QUIC       │              │
│   ├──────────────────┤        │  (TLS 1.3 포함)  │              │
│   │       TCP        │        ├──────────────────┤              │
│   ├──────────────────┤        │       UDP        │              │
│   │       IP         │        ├──────────────────┤              │
│   └──────────────────┘        │       IP         │              │
│                               └──────────────────┘              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**QUIC의 핵심 특징:**

```
1. 독립적인 스트림 (Stream-level HOL Blocking 해결)
┌─────────────────────────────────────────────────────────────────┐
│                      QUIC 연결                                   │
│                                                                 │
│   Stream 1  Stream 2  Stream 3  Stream 4  Stream 5             │
│     ▼         ▼         ▼         ▼         ▼                  │
│   [pkt1]   [pkt2]   [pkt3]   [pkt4]   [pkt5]                   │
│     │         │         X         │         │                  │
│     ▼         ▼                   ▼         ▼                  │
│   [도착]    [도착]   [재전송]    [도착]    [도착]                 │
│                         │                                      │
│   Stream 3만 대기, 다른 스트림은 정상 진행!                        │
└─────────────────────────────────────────────────────────────────┘

2. 0-RTT 연결 설정 (이전 연결 정보 활용)
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   TCP + TLS 1.3:  1 RTT (TCP) + 1 RTT (TLS) = 2 RTT            │
│   QUIC (최초):    1 RTT (QUIC + TLS 통합)                       │
│   QUIC (재연결):  0 RTT (저장된 키로 즉시 데이터 전송)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

3. Connection Migration
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   기존 TCP: IP/Port 변경 시 새 연결 필요                          │
│            WiFi → LTE 전환 시 연결 끊김                          │
│                                                                 │
│   QUIC: Connection ID로 연결 식별                                │
│         IP 변경되어도 Connection ID로 연결 유지                   │
│         모바일 환경에서 끊김 없는 경험                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 QUIC 핸드셰이크

```
┌────────────────────────────────────────────────────────────────────┐
│                     QUIC 1-RTT Handshake                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Client                                              Server       │
│      │                                                   │         │
│      │ ─────── Initial Packet ──────────────────────────►│         │
│      │         (CRYPTO frame: TLS ClientHello)           │         │
│      │         (Connection ID, Version)                  │         │
│      │                                                   │  1 RTT  │
│      │ ◄─────── Initial Packet ──────────────────────────│         │
│      │          (CRYPTO frame: TLS ServerHello)          │         │
│      │ ◄─────── Handshake Packet ────────────────────────│         │
│      │          (CRYPTO: EncryptedExtensions,            │         │
│      │           Certificate, CertificateVerify,         │         │
│      │           Finished)                               │         │
│      │                                                   │         │
│      │ ─────── Handshake Packet ────────────────────────►│         │
│      │         (CRYPTO: Finished)                        │         │
│      │ ─────── 1-RTT Packet (Application Data) ─────────►│         │
│      │                                                   │         │
│      │ ◄═════════════════════════════════════════════════│         │
│      │          1-RTT Packet (Application Data)          │         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 5.4 HTTP/3 특징

```python
# HTTP/3 vs HTTP/2 비교
class HTTP3Features:
    """
    HTTP/3의 주요 특징
    """

    HEADER_COMPRESSION = "QPACK"
    # HTTP/2의 HPACK 대신 QPACK 사용
    # QPACK: QUIC에 최적화된 헤더 압축
    # 순서 보장이 없는 QUIC에서 HOL Blocking 방지

    STREAM_PRIORITIZATION = "Extensible Priorities (RFC 9218)"
    # HTTP/2의 복잡한 의존성 트리 대신
    # 단순한 Urgency + Incremental 모델

    SERVER_PUSH = "Deprecated"
    # HTTP/2에서 잘 활용되지 않아 사실상 폐기

    CONNECTION_MIGRATION = True
    # IP 변경 시에도 연결 유지

    ALWAYS_ENCRYPTED = True
    # 암호화가 프로토콜에 내장 (TLS 1.3 필수)
```

**QPACK vs HPACK:**
```
HPACK (HTTP/2):
- 동적 테이블 업데이트가 순서에 의존
- TCP의 순서 보장에 의존

QPACK (HTTP/3):
- 인코더/디코더 스트림 분리
- 순서 없이 도착해도 정상 동작
- 약간의 압축 효율 감소 (HOL Blocking 방지 트레이드오프)
```

### 5.5 HTTP/2 vs HTTP/3 비교

| 특성 | HTTP/2 | HTTP/3 |
|------|--------|--------|
| 전송 계층 | TCP | QUIC (UDP 기반) |
| TLS | 별도 계층 | 통합 (TLS 1.3) |
| 연결 설정 | TCP + TLS = 2-3 RTT | 1 RTT (0-RTT 재연결) |
| HOL Blocking | TCP 레벨에서 발생 | 스트림 독립 (해결) |
| 헤더 압축 | HPACK | QPACK |
| Connection Migration | 미지원 | 지원 |
| 패킷 손실 복구 | TCP 재전송 | 스트림별 독립 복구 |
| 미들박스 호환성 | 양호 | NAT 타임아웃 이슈 가능 |

### 5.6 실무에서의 HTTP/3 적용

**Nginx HTTP/3 설정:**
```nginx
# nginx.conf
http {
    server {
        listen 443 ssl;
        listen 443 quic reuseport;  # HTTP/3

        ssl_certificate /etc/ssl/certs/example.crt;
        ssl_certificate_key /etc/ssl/private/example.key;

        # TLS 1.3 필수
        ssl_protocols TLSv1.3;

        # QUIC 관련 설정
        ssl_early_data on;  # 0-RTT 활성화

        # Alt-Svc 헤더로 HTTP/3 지원 알림
        add_header Alt-Svc 'h3=":443"; ma=86400';

        location / {
            root /var/www/html;
        }
    }
}
```

**Curl로 HTTP/3 테스트:**
```bash
# HTTP/3 요청
curl --http3 https://example.com

# 상세 정보 확인
curl -v --http3 https://example.com 2>&1 | grep -i "using http"
# * using HTTP/3

# Alt-Svc 헤더 확인
curl -I https://cloudflare.com 2>&1 | grep -i alt-svc
# alt-svc: h3=":443"; ma=86400
```

---

## 참고 자료

- [RFC 9110 - HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [RFC 9113 - HTTP/2](https://datatracker.ietf.org/doc/html/rfc9113)
- [RFC 9114 - HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)
- [RFC 9000 - QUIC](https://datatracker.ietf.org/doc/html/rfc9000)
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [MDN Web Docs - HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [Cloudflare - A Detailed Look at RFC 8446 (TLS 1.3)](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
- [web.dev - HTTP/2](https://web.dev/articles/performance-http2)

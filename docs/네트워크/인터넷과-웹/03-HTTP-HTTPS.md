# HTTP HTTPS

- HTTP/2 Server Push 예시는 과거 동작을 설명함. Chrome은 106부터 지원을 중단했고 Nginx의 관련 지시어는 1.25.1부터 폐기되었음. [Chrome 변경 안내](https://developer.chrome.com/blog/removing-push), [Nginx HTTP/2 모듈](https://nginx.org/en/docs/http/ngx_http_v2_module.html)

HTTP는 요청과 응답으로 웹 리소스를 주고받는다. 버전이 바뀌면서 연결을 재사용하고 여러 요청을 동시에 처리하는 방식도 달라졌다. 이 변화부터 살펴본 뒤 메서드, 상태 코드, 헤더를 통해 요청의 의미와 처리 조건을 읽어 본다.

## HTTP 버전별 비교

### HTTP 발전 역사

- 1991: HTTP/0.9 - 단순 GET 요청만 지원
- 1996: HTTP/1.0 - RFC 1945
- 1997: HTTP/1.1 - RFC 2068 → RFC 2616 → RFC 7230-7235 (2014)
- 2015: HTTP/2, - RFC 7540 → RFC 9113 (2022)
- 2022: HTTP/3, - RFC 9114 (QUIC 기반)

### HTTP/1.0 (RFC 1945)

- 특징:
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

요청마다 TCP 연결을 만들고 응답 뒤 종료하면, 리소스가 세 개일 때 연결 설정도 세 번 반복한다. 이 3-way Handshake 비용을 줄이기 위해 다음 버전에서는 연결 재사용이 기본 동작이 된다.

### HTTP/1.1 (RFC 7230-7235)

- 주요 개선 사항:

- Persistent Connection (Keep-Alive)
- 기본적으로 연결 유지
- Connection: close로 명시적 종료
- Pipelining
- 응답 대기 없이 여러 요청 전송 가능
- 하지만 HOL Blocking 문제 존재
- Host 헤더 필수화
- 가상 호스팅 지원
- Chunked Transfer Encoding
- Content-Length 없이 스트리밍 가능
- 추가 메서드
- PUT, PATCH, DELETE, OPTIONS, TRACE, CONNECT

- HTTP/1.1 Keep-Alive:
- Client, Server
- Client → TCP 3-way Handshake → Server
- Client → Request 1 (GET /a.html) → Server
- Server → Response 1 → Client
- Client → Request 2 (GET /b.css) → Server
- Server → Response 2 → Client
- Client → Request 3 (GET /c.js) → Server
- Server → Response 3 → Client
- Client → Connection: close → Server

Pipelining을 사용하면 Request 1의 응답을 기다리지 않고 Request 2와 3을 보낼 수 있다. 하지만 응답은 요청 순서대로 보내야 하므로 Response 1이 늦어지면 나머지도 기다린다. 이것이 HTTP/1.1의 Head-of-Line(HOL) Blocking이다.

- 실무에서의 HTTP/1.1 최적화 기법:

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

### HTTP/2 (RFC 9113)

- 핵심 개념: 바이너리 프레이밍 계층

- HTTP/1.1: 텍스트 기반
- GET /resource HTTP/1.1\r\n
- Host: example.com\r\n
- \r\n
- HTTP/2: 바이너리 프레임
  - 고정 헤더 9바이트는 Length 24비트, Type 8비트, Flags 8비트, 예약 비트 R 1비트, Stream Identifier 31비트 순서임.
  - Length가 나타내는 길이의 Payload가 헤더 뒤에 이어짐. [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113.html#section-4.1) 참고.

- HTTP/2 주요 기능:

- Multiplexing (다중화)
- 단일 TCP 연결
- Str 1, Str 3, Str 1, Str 5, Str 3, ...
- Frame, Frame, Frame, Frame, Frame
- → 여러 스트림이 하나의 연결에서 인터리빙됨
- → HOL Blocking 해결 (HTTP 레벨에서는)
- Header Compression (HPACK)
- 기존 HTTP/1.1 (반복되는 헤더가 매번 전송됨):
- Request 1: Host: example.com, Accept: text/html, ...
- Request 2: Host: example.com, Accept: text/html, ...
- Request 3: Host: example.com, Accept: text/html, ...
- HTTP/2 HPACK (인덱스로 참조):
- Request 1: [Full Headers] → Static/Dynamic Table 구축
- Request 2: [Index 62, Index 63, ...]
- Request 3: [Index 62, Index 63, ...]
- 헤더 크기 85-95% 감소 가능
- Server Push
- Client, Server
- Client → GET /index.html → Server
- Server → PUSH_PROMISE (styles.css) → Client
- Server → PUSH_PROMISE (app.js) → Client
- Server → Response (index.html) → Client
- Server → Response (styles.css) → Client
- Server → Response (app.js) → Client
- Client → 클라이언트가 요청하기 전에 필요한 리소스 미리 전송 → Server
- Stream Prioritization
- Stream 1: Weight 256 (CSS - 높음)
- Stream 3: Weight 128 (이미지 - 중간)
- Stream 5: Weight 64, (분석 스크립트)
- 의존성 트리로 우선순위 표현 가능

- HTTP/2 Python 예제 (httpx):

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

- HTTP/2 Nginx 설정:

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

### HTTP/1.1 vs HTTP/2 비교표

- 특성: 프로토콜 형식:
  - HTTP/1.1: 텍스트
  - HTTP/2: 바이너리
- 특성: 연결:
  - HTTP/1.1: 도메인당 여러 TCP 연결
  - HTTP/2: 단일 TCP 연결
- 특성: 요청 처리:
  - HTTP/1.1: 순차적 (HOL Blocking)
  - HTTP/2: 병렬 (Multiplexing)
- 특성: 헤더 압축:
  - HTTP/1.1: 없음
  - HTTP/2: HPACK
- 특성: 서버 푸시:
  - HTTP/1.1: 없음
  - HTTP/2: 지원
- 특성: 우선순위:
  - HTTP/1.1: 없음
  - HTTP/2: 스트림 우선순위
- 특성: TLS 필수:
  - HTTP/1.1: 아니오
  - HTTP/2: 사실상 필수 (브라우저)

## HTTP 메서드와 상태 코드

### HTTP 메서드 (RFC 9110)

- GET:
  - 안전성: O
  - 멱등성: O
  - 캐시 가능: O
  - 용도: 리소스 조회
- HEAD:
  - 안전성: O
  - 멱등성: O
  - 캐시 가능: O
  - 용도: 헤더만 조회
- POST:
  - 안전성: X
  - 멱등성: X
  - 캐시 가능: 조건부
  - 용도: 리소스 생성, 처리
- PUT:
  - 안전성: X
  - 멱등성: O
  - 캐시 가능: X
  - 용도: 리소스 전체 교체
- PATCH:
  - 안전성: X
  - 멱등성: X
  - 캐시 가능: X
  - 용도: 리소스 부분 수정
- DELETE:
  - 안전성: X
  - 멱등성: O
  - 캐시 가능: X
  - 용도: 리소스 삭제
- OPTIONS:
  - 안전성: O
  - 멱등성: O
  - 캐시 가능: X
  - 용도: 지원 메서드 확인 (CORS)
- TRACE:
  - 안전성: O
  - 멱등성: O
  - 캐시 가능: X
  - 용도: 루프백 테스트
- CONNECT:
  - 안전성: X
  - 멱등성: X
  - 캐시 가능: X
  - 용도: 터널 설정 (프록시)
- 안전성(Safe): 서버 상태를 변경하지 않음
- 멱등성(Idempotent): 동일 요청을 여러 번 해도 결과가 같음

- 멱등성(Idempotency) 상세 설명:

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

- RESTful API 설계 예시:

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

### HTTP 상태 코드 (RFC 9110)

- 1xx: Informational (정보)
- 100 Continue, - 요청 계속 진행
- 101 Switching Protocols - 프로토콜 전환 (WebSocket)
- 103 Early Hints, - 리소스 힌트 (preload)
- 2xx: Success (성공)
- 200 OK, - 성공
- 201 Created, - 리소스 생성됨 (POST)
- 202 Accepted, - 요청 수락됨 (비동기 처리)
- 204 No Content, - 성공, 본문 없음 (DELETE)
- 206 Partial Content, - 범위 요청 성공
- 3xx: Redirection (리다이렉션)
- 301 Moved Permanently - 영구 이동 (GET으로 변경될 수 있음)
- 302 Found, - 임시 이동 (GET으로 변경될 수 있음)
- 303 See Other, - GET으로 다른 URI 조회
- 304 Not Modified, - 캐시 사용 (조건부 요청)
- 307 Temporary Redirect - 임시 이동 (메서드 유지)
- 308 Permanent Redirect - 영구 이동 (메서드 유지)
- 4xx: Client Error (클라이언트 오류)
- 400 Bad Request, - 잘못된 요청
- 401 Unauthorized, - 인증 필요
- 403 Forbidden, - 권한 없음
- 404 Not Found, - 리소스 없음
- 405 Method Not Allowed - 허용되지 않는 메서드
- 408 Request Timeout, - 요청 타임아웃
- 409 Conflict, - 충돌 (동시 수정 등)
- 413 Payload Too Large - 요청 본문이 너무 큼
- 415 Unsupported Media Type - 지원하지 않는 미디어 타입
- 422 Unprocessable Entity - 문법은 맞지만 처리 불가
- 429 Too Many Requests - 요청 횟수 초과 (Rate Limit)
- 499 Client Closed Request - 클라이언트가 연결 종료 (Nginx)
- 5xx: Server Error (서버 오류)
- 500 Internal Server Error - 서버 내부 오류
- 501 Not Implemented, - 기능 미구현
- 502 Bad Gateway, - 게이트웨이 오류
- 503 Service Unavailable - 서비스 이용 불가
- 504 Gateway Timeout, - 게이트웨이 타임아웃
- 599 Network Connect Timeout - 네트워크 연결 타임아웃

- 자주 혼동되는 상태 코드 비교:

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

## 헤더 구조와 주요 헤더

### 헤더 분류

- HTTP 헤더 분류
- General Headers (일반 헤더)
- 요청과 응답 모두에 사용
- Cache-Control, Connection, Date, Transfer-Encoding
- Request Headers (요청 헤더)
- 클라이언트 → 서버 정보 전달
- Host, User-Agent, Accept, Authorization, Cookie
- Response Headers (응답 헤더)
- 서버 → 클라이언트 정보 전달
- Server, Set-Cookie, WWW-Authenticate, Location
- Entity/Representation Headers (엔티티 헤더)
- 본문 관련 정보
- Content-Type, Content-Length, Content-Encoding, ETag

### 주요 요청 헤더

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

### 주요 응답 헤더

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

### Cache-Control 디렉티브 상세

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

## HTTP/3와 QUIC 프로토콜

### HTTP/2의 한계

HTTP/2는 여러 스트림의 프레임을 섞어 보내지만, 그 아래에서는 하나의 TCP 바이트 스트림을 사용한다. 중간 패킷이 손실되면 뒤에 도착한 바이트도 재전송을 기다리므로, 손실된 데이터와 다른 HTTP 스트림도 영향을 받는다. HTTP 수준의 다중화만으로 TCP 수준의 HOL Blocking까지 없애지는 못한다.

### QUIC 프로토콜 (RFC 9000)

- QUIC = Quick UDP Internet Connections

- Protocol Stack 비교
- HTTP/2 over TCP, HTTP/3 over QUIC
- HTTP/2, HTTP/3
- TLS, QUIC
- (TLS 1.3 포함)
- TCP
- UDP
- IP
- IP

- QUIC의 핵심 특징:

- 독립적인 스트림 (Stream-level HOL Blocking 해결)
- QUIC 연결
- Stream 1, Stream 2, Stream 3, Stream 4, Stream 5
- [pkt1], [pkt2], [pkt3], [pkt4], [pkt5]
- X
- [도착], [도착], [재전송], [도착], [도착]
- Stream 3만 대기, 다른 스트림은 정상 진행!
- 0-RTT 연결 설정 (이전 연결 정보 활용)
- TCP + TLS 1.3:, 1 RTT (TCP) + 1 RTT (TLS) = 2 RTT
- QUIC (최초):, 1 RTT (QUIC + TLS 통합)
- QUIC (재연결):, 0 RTT (저장된 키로 즉시 데이터 전송)
- Connection Migration
- 기존 TCP: IP/Port 변경 시 새 연결 필요
- WiFi → LTE 전환 시 연결 끊김
- QUIC: Connection ID로 연결 식별
- IP 변경되어도 Connection ID로 연결 유지
- 모바일 환경에서 끊김 없는 경험

### QUIC 핸드셰이크

- QUIC 1-RTT Handshake
- Client, Server
- Client → Initial Packet → Server
- (CRYPTO frame: TLS ClientHello)
- (Connection ID, Version)
- 1 RTT
- Server → Initial Packet → Client
- (CRYPTO frame: TLS ServerHello)
- Server → Handshake Packet → Client
- (CRYPTO: EncryptedExtensions
- Certificate, CertificateVerify
- Finished)
- Client → Handshake Packet → Server
- (CRYPTO: Finished)
- Client → 1-RTT Packet (Application Data) → Server
- 1-RTT Packet (Application Data)

### HTTP/3 특징

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

- QPACK vs HPACK:
- HPACK (HTTP/2):
- 동적 테이블 업데이트가 순서에 의존
- TCP의 순서 보장에 의존
- QPACK (HTTP/3):
- 인코더/디코더 스트림 분리
- 순서 없이 도착해도 정상 동작
- 약간의 압축 효율 감소 (HOL Blocking 방지 트레이드오프)

### HTTP/2 vs HTTP/3 비교

- 특성: 전송 계층:
  - HTTP/2: TCP
  - HTTP/3: QUIC (UDP 기반)
- 특성: TLS:
  - HTTP/2: 별도 계층
  - HTTP/3: 통합 (TLS 1.3)
- 특성: 연결 설정:
  - HTTP/2: TCP + TLS = 2-3 RTT
  - HTTP/3: 1 RTT (0-RTT 재연결)
- 특성: HOL Blocking:
  - HTTP/2: TCP 레벨에서 발생
  - HTTP/3: 스트림 독립 (해결)
- 특성: 헤더 압축:
  - HTTP/2: HPACK
  - HTTP/3: QPACK
- 특성: Connection Migration:
  - HTTP/2: 미지원
  - HTTP/3: 지원
- 특성: 패킷 손실 복구:
  - HTTP/2: TCP 재전송
  - HTTP/3: 스트림별 독립 복구
- 특성: 미들박스 호환성:
  - HTTP/2: 양호
  - HTTP/3: NAT 타임아웃 이슈 가능

### 실무에서의 HTTP/3 적용

- Nginx HTTP/3 설정:

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

- Curl로 HTTP/3 테스트:

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

## 참고 자료

- [RFC 9110 - HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [RFC 9113 - HTTP/2](https://datatracker.ietf.org/doc/html/rfc9113)
- [RFC 9114 - HTTP/3](https://datatracker.ietf.org/doc/html/rfc9114)
- [RFC 9000 - QUIC](https://datatracker.ietf.org/doc/html/rfc9000)
- [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [MDN Web Docs - HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- [Cloudflare - A Detailed Look at RFC 8446 (TLS 1.3)](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
- [web.dev - HTTP/2](https://web.dev/articles/performance-http2)

## 관련 학습

- [인터넷 작동 원리](02-인터넷-작동-원리.md)
- [DNS와 도메인](04-DNS와-도메인.md)
- [CORS](../../보안/웹-보안/02-CORS.md)
- [TLS와 HTTPS](../../보안/암호화와-전송-보안/02-TLS와-HTTPS.md)

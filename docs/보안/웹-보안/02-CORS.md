# CORS

## 헤더 구조와 주요 헤더

### CORS (Cross-Origin Resource Sharing)

`https://app.com`의 브라우저 코드가 `https://api.com/api/data`로 JSON을 보내는 상황을 보자. 브라우저는 먼저 `OPTIONS /api/data`로 사전 요청(Preflight)을 보낸다. 여기에는 `Origin: https://app.com`, `Access-Control-Request-Method: POST`, `Access-Control-Request-Headers: Content-Type`을 담아 실제로 보낼 요청을 알린다.

서버는 `200 OK`와 함께 허용 출처를 `https://app.com`, 허용 메서드를 `GET, POST`, 허용 헤더를 `Content-Type`으로 응답한다. `Access-Control-Max-Age: 86400`은 사전 요청 결과의 캐시 시간을 지정한다. 허용 응답을 받은 브라우저는 `Origin: https://app.com`과 `Content-Type: application/json`을 담아 실제 POST 요청을 보낸다. 실제 응답에도 `Access-Control-Allow-Origin: https://app.com`이 포함된다.

- Simple Request vs Preflight Request:

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

- Express.js CORS 설정 예시:

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

## CORS란?

앞의 흐름처럼 서버가 HTTP 헤더로 다른 출처의 리소스 접근을 허용하는 메커니즘이 CORS(Cross-Origin Resource Sharing)다. 브라우저는 이 헤더를 보고 교차 출처 응답에 접근할 수 있는지 판단한다.

- 동일 출처 요청 (Same-Origin):
- https://example.com/page.html → https://example.com/api/data 가능
- 교차 출처 요청 (Cross-Origin):
- https://example.com/page.html → https://api.example.com/data 불가 (기본 차단)
- https://example.com/page.html → https://api.example.com/data 가능 (CORS 허용 시)

## 왜 CORS가 필요한가?

- CORS가 없다면:
- 악성 사이트 (evil.com), 피해자가 로그인한 사이트 (bank.com)
- `<script>`
- fetch(, GET /api/accounts
- 'bank.com/, →, Cookie: session=x
- api/accounts'
- ), ←, { accounts: [...] }
- `</script>`, 민감 정보 유출!
- Same-Origin Policy로 이러한 공격을 기본적으로 차단함.
- CORS는 필요한 경우에만 명시적으로 허용함.

## Same-Origin Policy

### Origin의 정의

Origin은 스킴(Protocol), 호스트(Domain), 포트의 조합이다. `https://example.com:443/path/page.html`에서 Origin은 경로를 제외한 `https://example.com:443`이며, HTTPS의 기본 포트인 443은 생략할 수 있다.

### Same-Origin 판단 예시

- URL A: https://example.com/a:
  - URL B: https://example.com/b
  - 동일 출처?: Yes
  - 이유: 경로만 다름
- URL A: https://example.com:
  - URL B: https://example.com:443
  - 동일 출처?: Yes
  - 이유: 443은 HTTPS 기본 포트
- URL A: http://example.com:
  - URL B: https://example.com
  - 동일 출처?: No
  - 이유: 스킴이 다름
- URL A: https://example.com:
  - URL B: https://api.example.com
  - 동일 출처?: No
  - 이유: 호스트가 다름
- URL A: https://example.com:
  - URL B: https://example.com:8080
  - 동일 출처?: No
  - 이유: 포트가 다름
- URL A: https://example.com:
  - URL B: https://example.org
  - 동일 출처?: No
  - 이유: 호스트가 다름

### SOP가 적용되는 것과 아닌 것

```javascript
// SOP가 적용됨 (CORS 필요)
fetch('https://api.other.com/data')  // 다른 출처 API 호출
const xhr = new XMLHttpRequest();
xhr.open('GET', 'https://api.other.com/data');

// SOP가 적용되지 않음 (자유롭게 사용 가능)
<img src="https://other.com/image.jpg">      // 이미지 로드
<script src="https://cdn.other.com/lib.js">  // 스크립트 로드
<link href="https://cdn.other.com/style.css"> // CSS 로드
<iframe src="https://other.com/page">        // iframe (단, 내용 접근은 제한)
<video src="https://media.other.com/video.mp4">
<audio src="https://media.other.com/audio.mp3">
```

## CORS 헤더

### 응답 헤더 (서버 → 브라우저)

- 헤더: `Access-Control-Allow-Origin`:
  - 설명: 허용된 출처
  - 예시: `https://example.com` 또는 `*`
- 헤더: `Access-Control-Allow-Methods`:
  - 설명: 허용된 HTTP 메서드
  - 예시: `GET, POST, PUT, DELETE`
- 헤더: `Access-Control-Allow-Headers`:
  - 설명: 허용된 요청 헤더
  - 예시: `Content-Type, Authorization`
- 헤더: `Access-Control-Allow-Credentials`:
  - 설명: 자격증명 허용 여부
  - 예시: `true`
- 헤더: `Access-Control-Expose-Headers`:
  - 설명: JS에서 접근 가능한 응답 헤더
  - 예시: `X-Custom-Header`
- 헤더: `Access-Control-Max-Age`:
  - 설명: Preflight 캐시 시간(초)
  - 예시: `86400`

### 요청 헤더 (브라우저 → 서버, Preflight)

- 헤더: `Origin`:
  - 설명: 요청 출처
  - 예시: `https://example.com`
- 헤더: `Access-Control-Request-Method`:
  - 설명: 실제 요청 메서드
  - 예시: `POST`
- 헤더: `Access-Control-Request-Headers`:
  - 설명: 실제 요청에 포함될 헤더
  - 예시: `Content-Type, X-Custom`

### Access-Control-Allow-Origin

```http
# 특정 출처만 허용 (권장)
Access-Control-Allow-Origin: https://example.com

# 모든 출처 허용 (주의 필요)
Access-Control-Allow-Origin: *

# 여러 출처 허용 시 (동적 처리 필요)
# 헤더에는 하나의 출처만 지정 가능
# 서버에서 요청 Origin을 확인하여 동적으로 설정
```

```java
// 동적 Origin 처리
@Component
public class DynamicCorsFilter implements Filter {

    private static final Set<String> ALLOWED_ORIGINS = Set.of(
        "https://app.example.com",
        "https://admin.example.com",
        "https://mobile.example.com"
    );

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String origin = request.getHeader("Origin");

        if (origin != null && ALLOWED_ORIGINS.contains(origin)) {
            response.setHeader("Access-Control-Allow-Origin", origin);
            response.setHeader("Vary", "Origin");  // 캐시 구분을 위해 필수
        }

        chain.doFilter(request, response);
    }
}
```

### Access-Control-Allow-Credentials

```javascript
// 클라이언트: credentials 포함 요청
fetch('https://api.example.com/data', {
    credentials: 'include'  // 쿠키, HTTP 인증 정보 포함
});

// 또는
const xhr = new XMLHttpRequest();
xhr.withCredentials = true;
```

```http
# 서버 응답 헤더
Access-Control-Allow-Origin: https://app.example.com  # * 사용 불가!
Access-Control-Allow-Credentials: true
```

위 요청처럼 `credentials: 'include'`를 사용하면 서버는 `Access-Control-Allow-Origin`에 특정 출처를 지정해야 한다. 이 경우에는 모든 출처를 뜻하는 `*`를 사용할 수 없다.

### Access-Control-Expose-Headers

```http
# 기본적으로 JavaScript에서 접근 가능한 응답 헤더 (CORS-safelisted):
# - Cache-Control
# - Content-Language
# - Content-Length
# - Content-Type
# - Expires
# - Last-Modified
# - Pragma

# 추가 헤더를 JavaScript에서 접근 가능하게 하려면:
Access-Control-Expose-Headers: X-Custom-Header, X-Request-Id
```

```javascript
// 클라이언트에서 커스텀 헤더 접근
fetch('https://api.example.com/data')
    .then(response => {
        // Expose-Headers에 명시된 헤더만 접근 가능
        console.log(response.headers.get('X-Custom-Header'));
        console.log(response.headers.get('X-Request-Id'));
    });
```

## Preflight Request

### Preflight가 필요한 경우

- 브라우저는 "안전하지 않은" 요청을 보내기 전에 OPTIONS 메서드로 Preflight 요청을 먼저 보냄.

- Simple Request (Preflight 없음):
- 메서드: GET, HEAD, POST만
- Content-Type: application/x-www-form-urlencoded
- multipart/form-data
- text/plain만
- 커스텀 헤더 없음 (Accept, Accept-Language 등 기본 헤더만)
- Preflighted Request (Preflight 필요):
- 메서드: PUT, DELETE, PATCH 등
- Content-Type: application/json 등
- 커스텀 헤더: Authorization, X-Custom-Header 등

### Preflight 요청 흐름

- Client, Server
- OPTIONS /api/data HTTP/1.1
- Origin: https://app.example.com
- Access-Control-Request-Method: POST
- Access-Control-Request-Headers:
- Content-Type, Authorization
- HTTP/1.1 204 No Content
- Access-Control-Allow-Origin:
- https://app.example.com
- Access-Control-Allow-Methods:
- GET, POST, PUT, DELETE
- Access-Control-Allow-Headers:
- Content-Type, Authorization
- Access-Control-Max-Age: 86400
- POST /api/data HTTP/1.1
- Origin: https://app.example.com
- Content-Type: application/json
- Authorization: Bearer token123
- {"name": "John"}
- HTTP/1.1 200 OK
- Access-Control-Allow-Origin:
- https://app.example.com
- {"id": 1, "name": "John"}

### Preflight 캐싱

```http
# Preflight 응답 캐싱 (성능 최적화)
Access-Control-Max-Age: 86400  # 24시간 동안 캐시

# 브라우저별 최대값:
# - Chrome: 7200 (2시간)
# - Firefox: 86400 (24시간)
# - Safari: 600 (10분)
```

### Spring에서 Preflight 처리

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(86400);  // Preflight 캐시 24시간
    }
}

// Spring Security와 함께 사용 시
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // CORS 설정 활성화
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // OPTIONS 요청은 인증 없이 허용
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS).permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("https://app.example.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(86400L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

## CORS 설정 Best Practice

### 와일드카드(*) 사용 금지

```java
// 취약한 설정
@Configuration
public class BadCorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("*")           // 모든 출처 허용 - 위험!
            .allowedMethods("*")           // 모든 메서드 허용 - 위험!
            .allowedHeaders("*");
    }
}

// 안전한 설정
@Configuration
public class SafeCorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "https://app.example.com",
                "https://admin.example.com"
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("Content-Type", "Authorization")
            .allowCredentials(true);
    }
}
```

### Origin 검증

```java
@Component
public class StrictCorsFilter implements Filter {

    private static final Set<String> ALLOWED_ORIGINS = Set.of(
        "https://app.example.com",
        "https://admin.example.com"
    );

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String origin = request.getHeader("Origin");

        if (origin != null) {
            // 정확한 문자열 매칭
            if (ALLOWED_ORIGINS.contains(origin)) {
                response.setHeader("Access-Control-Allow-Origin", origin);
                response.setHeader("Vary", "Origin");
            } else {
                // 허용되지 않은 Origin
                log.warn("Blocked CORS request from: {}", origin);
                response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                return;
            }
        }

        chain.doFilter(request, response);
    }
}

// 주의: 정규식이나 부분 문자열 매칭은 위험할 수 있음
// 취약 예: origin.endsWith("example.com")
// 공격: evil-example.com도 매칭됨!
```

### null Origin 차단

```java
// null Origin은 file:// 프로토콜이나 리다이렉트에서 발생
// 절대 허용하지 말 것!

public void doFilter(...) {
    String origin = request.getHeader("Origin");

    // null Origin 차단
    if ("null".equals(origin)) {
        log.warn("Blocked null Origin request");
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        return;
    }

    // ...
}
```

### 환경별 CORS 설정

```java
@Configuration
public class CorsConfig {

    @Value("${cors.allowed-origins}")
    private List<String> allowedOrigins;

    @Bean
    @Profile("development")
    public CorsConfigurationSource devCorsConfiguration() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(
            "http://localhost:3000",
            "http://localhost:8080"
        ));
        config.setAllowedMethods(List.of("*"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }

    @Bean
    @Profile("production")
    public CorsConfigurationSource prodCorsConfiguration() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(allowedOrigins);  // 설정 파일에서 로드
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of("Content-Type", "Authorization"));
        config.setAllowCredentials(true);
        config.setMaxAge(86400L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

```yaml
# application-production.yml
cors:
  allowed-origins:
    - https://app.example.com
    - https://admin.example.com
```

### CORS 로깅 및 모니터링

```java
@Component
@Slf4j
public class CorsLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String origin = request.getHeader("Origin");
        String method = request.getMethod();

        if (origin != null) {
            log.debug("CORS Request - Origin: {}, Method: {}, Path: {}",
                origin, method, request.getRequestURI());
        }

        filterChain.doFilter(request, response);

        // 응답 후 CORS 헤더 확인
        String allowOrigin = response.getHeader("Access-Control-Allow-Origin");
        if (origin != null && allowOrigin == null) {
            log.warn("CORS Request Blocked - Origin: {}, Path: {}",
                origin, request.getRequestURI());
        }
    }
}
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

- [MDN - Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Fetch Standard - CORS Protocol](https://fetch.spec.whatwg.org/#cors-protocol)
- [OWASP - CORS](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny)
- [Spring CORS Documentation](https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html)

## 관련 학습

- [OWASP Top 10](01-OWASP-Top-10.md)
- [SQL Injection XSS CSRF](03-SQL-Injection-XSS-CSRF.md)
- [HTTP HTTPS](../../네트워크/인터넷과-웹/03-HTTP-HTTPS.md)
- [TLS와 HTTPS](../암호화와-전송-보안/02-TLS와-HTTPS.md)

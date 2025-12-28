# CORS (Cross-Origin Resource Sharing)

## 목차
1. [개요](#개요)
2. [Same-Origin Policy](#same-origin-policy)
3. [CORS 헤더](#cors-헤더)
4. [Preflight Request](#preflight-request)
5. [CORS 설정 Best Practice](#cors-설정-best-practice)
6. [실무 구현](#실무-구현)
---

## 개요

### CORS란?

CORS(Cross-Origin Resource Sharing)는 브라우저가 다른 출처(Origin)의 리소스에 접근할 수 있도록 허용하는 HTTP 헤더 기반 메커니즘입니다.

```
동일 출처 요청 (Same-Origin):
https://example.com/page.html → https://example.com/api/data ✓

교차 출처 요청 (Cross-Origin):
https://example.com/page.html → https://api.example.com/data ✗ (기본 차단)
https://example.com/page.html → https://api.example.com/data ✓ (CORS 허용 시)
```

### 왜 CORS가 필요한가?

```
CORS가 없다면:

악성 사이트 (evil.com)                   피해자가 로그인한 사이트 (bank.com)
┌────────────────────┐                  ┌────────────────────┐
│  <script>          │                  │                    │
│    fetch(          │                  │  GET /api/accounts │
│      'bank.com/    │─────────────────►│  Cookie: session=x │
│       api/accounts'│                  │                    │
│    )               │◄─────────────────│  { accounts: [...] }
│  </script>         │    민감 정보 유출!  │                    │
└────────────────────┘                  └────────────────────┘

Same-Origin Policy로 이러한 공격을 기본적으로 차단합니다.
CORS는 필요한 경우에만 명시적으로 허용합니다.
```

---

## Same-Origin Policy

### Origin의 정의

Origin은 **스킴(Protocol) + 호스트(Domain) + 포트**의 조합입니다.

```
URL: https://example.com:443/path/page.html

Origin = https:// + example.com + :443
       = https://example.com:443
       (HTTPS 기본 포트 443은 생략 가능)
```

### Same-Origin 판단 예시

| URL A | URL B | 동일 출처? | 이유 |
|-------|-------|----------|------|
| https://example.com/a | https://example.com/b | Yes | 경로만 다름 |
| https://example.com | https://example.com:443 | Yes | 443은 HTTPS 기본 포트 |
| http://example.com | https://example.com | No | 스킴이 다름 |
| https://example.com | https://api.example.com | No | 호스트가 다름 |
| https://example.com | https://example.com:8080 | No | 포트가 다름 |
| https://example.com | https://example.org | No | 호스트가 다름 |

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

---

## CORS 헤더

### 응답 헤더 (서버 → 브라우저)

| 헤더 | 설명 | 예시 |
|------|------|------|
| `Access-Control-Allow-Origin` | 허용된 출처 | `https://example.com` 또는 `*` |
| `Access-Control-Allow-Methods` | 허용된 HTTP 메서드 | `GET, POST, PUT, DELETE` |
| `Access-Control-Allow-Headers` | 허용된 요청 헤더 | `Content-Type, Authorization` |
| `Access-Control-Allow-Credentials` | 자격증명 허용 여부 | `true` |
| `Access-Control-Expose-Headers` | JS에서 접근 가능한 응답 헤더 | `X-Custom-Header` |
| `Access-Control-Max-Age` | Preflight 캐시 시간(초) | `86400` |

### 요청 헤더 (브라우저 → 서버, Preflight)

| 헤더 | 설명 | 예시 |
|------|------|------|
| `Origin` | 요청 출처 | `https://example.com` |
| `Access-Control-Request-Method` | 실제 요청 메서드 | `POST` |
| `Access-Control-Request-Headers` | 실제 요청에 포함될 헤더 | `Content-Type, X-Custom` |

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

> **주의**: `credentials: 'include'`와 함께 사용할 때는 `Access-Control-Allow-Origin: *`를 사용할 수 없습니다. 반드시 특정 출처를 지정해야 합니다.

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

---

## Preflight Request

### Preflight가 필요한 경우

브라우저는 "안전하지 않은" 요청을 보내기 전에 OPTIONS 메서드로 Preflight 요청을 먼저 보냅니다.

```
Simple Request (Preflight 없음):
- 메서드: GET, HEAD, POST만
- Content-Type: application/x-www-form-urlencoded,
                multipart/form-data,
                text/plain만
- 커스텀 헤더 없음 (Accept, Accept-Language 등 기본 헤더만)

Preflighted Request (Preflight 필요):
- 메서드: PUT, DELETE, PATCH 등
- Content-Type: application/json 등
- 커스텀 헤더: Authorization, X-Custom-Header 등
```

### Preflight 요청 흐름

```
Client                                          Server
   │                                               │
   │  1. OPTIONS /api/data HTTP/1.1                │
   │     Origin: https://app.example.com           │
   │     Access-Control-Request-Method: POST       │
   │     Access-Control-Request-Headers:           │
   │       Content-Type, Authorization             │
   │───────────────────────────────────────────────►
   │                                               │
   │  2. HTTP/1.1 204 No Content                   │
   │     Access-Control-Allow-Origin:              │
   │       https://app.example.com                 │
   │     Access-Control-Allow-Methods:             │
   │       GET, POST, PUT, DELETE                  │
   │     Access-Control-Allow-Headers:             │
   │       Content-Type, Authorization             │
   │     Access-Control-Max-Age: 86400             │
   │◄───────────────────────────────────────────────
   │                                               │
   │  3. POST /api/data HTTP/1.1                   │
   │     Origin: https://app.example.com           │
   │     Content-Type: application/json            │
   │     Authorization: Bearer token123            │
   │     {"name": "John"}                          │
   │───────────────────────────────────────────────►
   │                                               │
   │  4. HTTP/1.1 200 OK                           │
   │     Access-Control-Allow-Origin:              │
   │       https://app.example.com                 │
   │     {"id": 1, "name": "John"}                 │
   │◄───────────────────────────────────────────────
```

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

---

## CORS 설정 Best Practice

### 1. 와일드카드(*) 사용 금지

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

### 2. Origin 검증

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

### 3. null Origin 차단

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

### 4. 환경별 CORS 설정

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

### 5. CORS 로깅 및 모니터링

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

---

## 실무 구현

### Spring Boot 전체 CORS 설정

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Value("${cors.allowed-origins}")
    private List<String> allowedOrigins;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())  // API 서버의 경우
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS).permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();

        // 허용할 출처
        configuration.setAllowedOrigins(allowedOrigins);

        // 허용할 메서드
        configuration.setAllowedMethods(List.of(
            "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"
        ));

        // 허용할 헤더
        configuration.setAllowedHeaders(List.of(
            "Authorization",
            "Content-Type",
            "X-Requested-With",
            "Accept",
            "Origin",
            "Access-Control-Request-Method",
            "Access-Control-Request-Headers"
        ));

        // 클라이언트에 노출할 헤더
        configuration.setExposedHeaders(List.of(
            "Access-Control-Allow-Origin",
            "Access-Control-Allow-Credentials",
            "X-Request-Id"
        ));

        // 자격증명 허용
        configuration.setAllowCredentials(true);

        // Preflight 캐시 시간
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

### 컨트롤러 레벨 CORS

```java
// 클래스 레벨
@RestController
@RequestMapping("/api/users")
@CrossOrigin(
    origins = {"https://app.example.com", "https://admin.example.com"},
    methods = {RequestMethod.GET, RequestMethod.POST},
    allowedHeaders = {"Content-Type", "Authorization"},
    exposedHeaders = {"X-Custom-Header"},
    allowCredentials = "true",
    maxAge = 3600
)
public class UserController {

    // 모든 메서드에 위 CORS 설정 적용
    @GetMapping
    public List<User> getUsers() {
        return userService.findAll();
    }

    // 메서드 레벨에서 오버라이드 가능
    @CrossOrigin(origins = "https://internal.example.com")
    @GetMapping("/internal")
    public List<User> getInternalUsers() {
        return userService.findInternalUsers();
    }
}
```

### Nginx CORS 설정

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    location /api/ {
        # Preflight 요청 처리
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' '$http_origin' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
            add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, X-Requested-With' always;
            add_header 'Access-Control-Allow-Credentials' 'true' always;
            add_header 'Access-Control-Max-Age' 86400;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        # 실제 요청 처리
        add_header 'Access-Control-Allow-Origin' '$http_origin' always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Expose-Headers' 'X-Request-Id' always;

        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# Origin 화이트리스트 검증 (map 사용)
map $http_origin $cors_origin {
    default "";
    "https://app.example.com" "$http_origin";
    "https://admin.example.com" "$http_origin";
}

# location 블록에서 사용
add_header 'Access-Control-Allow-Origin' '$cors_origin' always;
```

### React/Vue 프록시 설정 (개발용)

```javascript
// React (create-react-app) - setupProxy.js
const { createProxyMiddleware } = require('http-proxy-middleware');

module.exports = function(app) {
    app.use(
        '/api',
        createProxyMiddleware({
            target: 'http://localhost:8080',
            changeOrigin: true,
        })
    );
};

// Vue (vue.config.js)
module.exports = {
    devServer: {
        proxy: {
            '/api': {
                target: 'http://localhost:8080',
                changeOrigin: true,
            }
        }
    }
};
```

---

## 참고 자료

- [MDN - Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Fetch Standard - CORS Protocol](https://fetch.spec.whatwg.org/#cors-protocol)
- [OWASP - CORS](https://owasp.org/www-community/attacks/CORS_OriginHeaderScrutiny)
- [Spring CORS Documentation](https://docs.spring.io/spring-framework/reference/web/webmvc-cors.html)

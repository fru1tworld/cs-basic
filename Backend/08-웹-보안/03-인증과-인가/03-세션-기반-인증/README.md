# 세션 기반 인증

## 목차
1. [개요](#개요)
2. [세션 동작 원리](#세션-동작-원리)
3. [세션 관리](#세션-관리)
4. [쿠키 보안](#쿠키-보안)
5. [세션 공격과 방어](#세션-공격과-방어)
6. [분산 환경에서의 세션](#분산-환경에서의-세션)
7. [실무 구현](#실무-구현)
---

## 개요

### 세션 기반 인증이란?

세션 기반 인증은 서버 측에 사용자 상태를 저장하고, 클라이언트에는 세션 식별자(Session ID)만 전달하는 **Stateful** 인증 방식입니다.

```
┌──────────────┐                      ┌──────────────┐
│    Client    │                      │    Server    │
│   (Browser)  │                      │              │
└──────┬───────┘                      └──────┬───────┘
       │                                      │
  1. 로그인 요청                               │
     (username, password)                      │
       │──────────────────────────────────────►│
       │                              ┌────────┴────────┐
       │                              │  사용자 인증     │
       │                              │  세션 생성       │
       │                              │  Session Store  │
       │                              │     에 저장      │
       │                              └────────┬────────┘
  2. Set-Cookie: sessionId=abc123              │
       │◄──────────────────────────────────────│
       │                                       │
  3. 요청 + Cookie: sessionId=abc123           │
       │──────────────────────────────────────►│
       │                              ┌────────┴────────┐
       │                              │ Session Store   │
       │                              │ 에서 조회       │
       │                              │ 사용자 정보     │
       │                              │ 확인            │
       │                              └────────┬────────┘
  4. 응답                                      │
       │◄──────────────────────────────────────│
```

### 세션 vs JWT 비교

| 특성 | 세션 기반 | JWT 기반 |
|------|----------|----------|
| 상태 관리 | Stateful (서버 저장) | Stateless (토큰에 포함) |
| 확장성 | 세션 공유 필요 | 서버 독립적 |
| 보안성 | 즉시 무효화 가능 | 만료까지 유효 |
| 성능 | 세션 저장소 조회 필요 | 로컬 검증만 필요 |
| 크기 | Session ID만 전송 | 클레임 포함 (상대적으로 큼) |
| 사용 사례 | 전통적 웹 앱 | API, MSA, 모바일 |

---

## 세션 동작 원리

### 세션 생성 과정

```java
// 1. 로그인 처리 및 세션 생성
@PostMapping("/login")
public ResponseEntity<String> login(
    @RequestBody LoginRequest request,
    HttpServletRequest httpRequest
) {
    // 사용자 인증
    User user = authService.authenticate(request.username(), request.password());

    // 기존 세션 무효화 (세션 고정 공격 방지)
    HttpSession oldSession = httpRequest.getSession(false);
    if (oldSession != null) {
        oldSession.invalidate();
    }

    // 새 세션 생성
    HttpSession session = httpRequest.getSession(true);

    // 세션에 사용자 정보 저장
    session.setAttribute("userId", user.getId());
    session.setAttribute("username", user.getUsername());
    session.setAttribute("roles", user.getRoles());
    session.setAttribute("loginTime", Instant.now());
    session.setAttribute("loginIp", httpRequest.getRemoteAddr());

    // 세션 타임아웃 설정 (30분)
    session.setMaxInactiveInterval(30 * 60);

    return ResponseEntity.ok("로그인 성공");
}
```

### 세션 ID 구조

```
JSESSIONID (Java/Tomcat):
  - 예: JSESSIONID=ABC123DEF456GHI789
  - 일반적으로 32자 랜덤 문자열
  - 암호학적으로 안전한 난수 생성기 사용

ASP.NET_SessionId (.NET):
  - 예: ASP.NET_SessionId=xyzabc123456

PHPSESSID (PHP):
  - 예: PHPSESSID=abc123def456
```

```java
// 세션 ID 생성 (개념적 코드)
public class SessionIdGenerator {

    private static final SecureRandom secureRandom = new SecureRandom();
    private static final int SESSION_ID_LENGTH = 32;

    public String generateSessionId() {
        byte[] bytes = new byte[SESSION_ID_LENGTH];
        secureRandom.nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

### 세션 저장소 종류

| 저장소 | 특징 | 장점 | 단점 |
|--------|------|------|------|
| **메모리 (In-Memory)** | 서버 메모리에 저장 | 빠름, 간단 | 서버 재시작 시 소멸, 확장 어려움 |
| **파일 시스템** | 디스크에 저장 | 영속성 | 느림, 분산 어려움 |
| **데이터베이스** | RDBMS에 저장 | 영속성, 쿼리 가능 | 상대적으로 느림 |
| **Redis** | 인메모리 DB | 빠름, 분산 지원, TTL | 추가 인프라 필요 |
| **Memcached** | 분산 캐시 | 빠름, 확장성 | 영속성 없음 |

---

## 세션 관리

### 세션 타임아웃 설정

```java
// Spring Boot 설정
@Configuration
public class SessionConfig {

    @Bean
    public ServletContextInitializer servletContextInitializer() {
        return servletContext -> {
            // 세션 타임아웃 30분
            servletContext.setSessionTimeout(30);
        };
    }
}

// application.yml
server:
  servlet:
    session:
      timeout: 30m  # 30분
      cookie:
        name: SESSIONID
        http-only: true
        secure: true
        same-site: strict
```

### 세션 이벤트 리스너

```java
@Component
public class SessionEventListener implements HttpSessionListener,
                                             HttpSessionAttributeListener {

    private final AuditService auditService;
    private final AtomicInteger activeSessions = new AtomicInteger(0);

    @Override
    public void sessionCreated(HttpSessionEvent event) {
        HttpSession session = event.getSession();
        activeSessions.incrementAndGet();

        auditService.log(AuditEvent.builder()
            .eventType("SESSION_CREATED")
            .sessionId(session.getId())
            .timestamp(Instant.now())
            .build());
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent event) {
        HttpSession session = event.getSession();
        activeSessions.decrementAndGet();

        Long userId = (Long) session.getAttribute("userId");

        auditService.log(AuditEvent.builder()
            .eventType("SESSION_DESTROYED")
            .sessionId(session.getId())
            .userId(userId)
            .timestamp(Instant.now())
            .build());
    }

    @Override
    public void attributeAdded(HttpSessionBindingEvent event) {
        if ("userId".equals(event.getName())) {
            auditService.log(AuditEvent.builder()
                .eventType("USER_LOGIN")
                .sessionId(event.getSession().getId())
                .userId((Long) event.getValue())
                .build());
        }
    }

    public int getActiveSessionCount() {
        return activeSessions.get();
    }
}
```

### 동시 세션 제어

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                // 동시 세션 수 제한
                .maximumSessions(1)
                // 새 로그인 시 기존 세션 만료
                .maxSessionsPreventsLogin(false)
                // 세션 만료 시 리다이렉트 URL
                .expiredUrl("/session-expired")
            );
        return http.build();
    }

    @Bean
    public SessionRegistry sessionRegistry() {
        return new SessionRegistryImpl();
    }
}

// 특정 사용자의 모든 세션 강제 만료
@Service
public class SessionManagementService {

    private final SessionRegistry sessionRegistry;

    public void expireAllSessions(String username) {
        List<Object> principals = sessionRegistry.getAllPrincipals();

        for (Object principal : principals) {
            if (principal instanceof UserDetails userDetails) {
                if (userDetails.getUsername().equals(username)) {
                    List<SessionInformation> sessions =
                        sessionRegistry.getAllSessions(principal, false);

                    for (SessionInformation session : sessions) {
                        session.expireNow();
                    }
                }
            }
        }
    }

    public List<ActiveSession> getActiveSessions(String username) {
        return sessionRegistry.getAllPrincipals().stream()
            .filter(p -> p instanceof UserDetails)
            .filter(p -> ((UserDetails) p).getUsername().equals(username))
            .flatMap(p -> sessionRegistry.getAllSessions(p, false).stream())
            .map(this::toActiveSession)
            .toList();
    }
}
```

### 세션 클러스터링

```java
// Spring Session + Redis
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class RedisSessionConfig {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("redis.example.com");
        config.setPort(6379);
        config.setPassword(RedisPassword.of("password"));
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setCookieName("SESSION");
        serializer.setCookiePath("/");
        serializer.setDomainName("example.com");
        serializer.setUseSecureCookie(true);
        serializer.setUseHttpOnlyCookie(true);
        serializer.setSameSite("Strict");
        return serializer;
    }
}
```

---

## 쿠키 보안

### 쿠키 속성

```http
Set-Cookie: SESSIONID=abc123;
            Path=/;
            Domain=example.com;
            Max-Age=1800;
            Expires=Wed, 01 Jan 2025 00:00:00 GMT;
            Secure;
            HttpOnly;
            SameSite=Strict
```

| 속성 | 설명 | 보안 효과 |
|------|------|----------|
| **Secure** | HTTPS에서만 전송 | 네트워크 스니핑 방지 |
| **HttpOnly** | JavaScript 접근 불가 | XSS 공격 방지 |
| **SameSite** | 동일 사이트 요청에서만 전송 | CSRF 공격 방지 |
| **Domain** | 쿠키 전송 도메인 제한 | 범위 제한 |
| **Path** | 쿠키 전송 경로 제한 | 범위 제한 |
| **Max-Age/Expires** | 쿠키 만료 시간 | 장기 노출 방지 |

### SameSite 속성

```java
// SameSite 속성 값

// Strict: 동일 사이트 요청에서만 쿠키 전송
// - 외부 링크 클릭 시에도 쿠키 미전송
// - 가장 안전하지만 사용성 저하 가능
Cookie.setAttribute("SameSite", "Strict");

// Lax (기본값): 안전한 HTTP 메서드(GET)의 최상위 네비게이션에서만 전송
// - 링크 클릭 시 쿠키 전송됨
// - POST 폼 제출 시에는 미전송
Cookie.setAttribute("SameSite", "Lax");

// None: 모든 요청에서 전송 (Secure 필수)
// - 크로스 사이트 요청에도 전송
// - CSRF에 취약
Cookie.setAttribute("SameSite", "None");
Cookie.setSecure(true);  // 필수
```

### Spring Security 쿠키 설정

```java
@Configuration
public class CookieConfig {

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();

        serializer.setCookieName("SESSIONID");
        serializer.setCookiePath("/");
        serializer.setDomainName("example.com");
        serializer.setUseSecureCookie(true);      // Secure
        serializer.setUseHttpOnlyCookie(true);    // HttpOnly
        serializer.setSameSite("Strict");         // SameSite
        serializer.setCookieMaxAge(1800);         // 30분

        return serializer;
    }
}

// 또는 application.yml
server:
  servlet:
    session:
      cookie:
        name: SESSIONID
        path: /
        domain: example.com
        max-age: 1800
        secure: true
        http-only: true
        same-site: strict
```

---

## 세션 공격과 방어

### 1. 세션 고정 공격 (Session Fixation)

```
공격 시나리오:
1. 공격자가 사이트에서 세션 ID 획득 (예: ABC123)
2. 공격자가 피해자에게 세션 ID를 강제 설정
   (URL 파라미터, XSS 등)
3. 피해자가 해당 세션 ID로 로그인
4. 공격자가 같은 세션 ID로 피해자 계정 접근

┌───────────┐        ┌───────────┐        ┌───────────┐
│  Attacker │        │   Victim  │        │   Server  │
└─────┬─────┘        └─────┬─────┘        └─────┬─────┘
      │                     │                    │
 1. 세션 ID 획득             │                    │
      │─────────────────────────────────────────►│
      │◄────── Session ID: ABC123 ───────────────│
      │                     │                    │
 2. 피해자에게 세션 ID 전달    │                    │
      │─── http://site.com?sessionid=ABC123 ────►│
      │                     │                    │
 3. 피해자 로그인             │                    │
      │                     │───── Login ───────►│
      │                     │◄─ Session: ABC123 ─│
      │                     │                    │
 4. 공격자가 같은 세션으로 접근  │                    │
      │──────── Session ID: ABC123 ─────────────►│
      │◄───── 피해자 권한으로 응답 ──────────────────│
```

```java
// 방어: 로그인 시 세션 ID 재생성

// Spring Security 기본 설정 (자동 적용)
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                // 로그인 시 새 세션 생성, 기존 속성 복사
                .sessionFixation().migrateSession()
                // 또는: 완전히 새 세션 (.newSession())
            );
        return http.build();
    }
}

// 수동 구현
@PostMapping("/login")
public ResponseEntity<String> login(HttpServletRequest request, ...) {
    // 1. 인증 수행
    User user = authService.authenticate(...);

    // 2. 기존 세션의 속성 백업
    HttpSession oldSession = request.getSession(false);
    Map<String, Object> attributes = new HashMap<>();
    if (oldSession != null) {
        Enumeration<String> names = oldSession.getAttributeNames();
        while (names.hasMoreElements()) {
            String name = names.nextElement();
            attributes.put(name, oldSession.getAttribute(name));
        }
        oldSession.invalidate();  // 기존 세션 무효화
    }

    // 3. 새 세션 생성
    HttpSession newSession = request.getSession(true);

    // 4. 속성 복원 (필요한 것만)
    attributes.forEach(newSession::setAttribute);

    // 5. 사용자 정보 저장
    newSession.setAttribute("userId", user.getId());

    return ResponseEntity.ok("로그인 성공");
}
```

### 2. 세션 하이재킹 (Session Hijacking)

```
공격 방법:
1. 네트워크 스니핑 (MITM)
2. XSS를 통한 쿠키 탈취
3. 세션 ID 추측 (약한 생성기)
4. 물리적 접근 (공용 컴퓨터)
```

```java
// 방어 전략

// 1. HTTPS 강제
@Configuration
public class HttpsConfig {

    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
        tomcat.addConnectorCustomizers(connector -> {
            connector.setScheme("https");
            connector.setSecure(true);
        });
        return tomcat;
    }
}

// 2. 세션 바인딩 (IP, User-Agent)
@Component
public class SessionBindingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        HttpSession session = request.getSession(false);

        if (session != null && session.getAttribute("userId") != null) {
            String storedIp = (String) session.getAttribute("clientIp");
            String storedUserAgent = (String) session.getAttribute("userAgent");

            String currentIp = request.getRemoteAddr();
            String currentUserAgent = request.getHeader("User-Agent");

            // IP 변경 감지 (모바일 환경에서는 유연하게)
            if (storedIp != null && !storedIp.equals(currentIp)) {
                log.warn("세션 IP 변경 감지: {} -> {}", storedIp, currentIp);
                // 재인증 요구 또는 경고
            }

            // User-Agent 변경 감지
            if (storedUserAgent != null && !storedUserAgent.equals(currentUserAgent)) {
                log.warn("세션 User-Agent 변경 감지");
                session.invalidate();
                response.sendError(HttpServletResponse.SC_UNAUTHORIZED);
                return;
            }
        }

        filterChain.doFilter(request, response);
    }
}

// 3. 세션 타임아웃 및 활동 모니터링
@Service
public class SessionSecurityService {

    public void validateSession(HttpSession session, HttpServletRequest request) {
        // 마지막 활동 시간 확인
        Instant lastActivity = (Instant) session.getAttribute("lastActivity");
        Instant now = Instant.now();

        // 절대 타임아웃 (8시간)
        Instant loginTime = (Instant) session.getAttribute("loginTime");
        if (loginTime != null &&
            Duration.between(loginTime, now).toHours() > 8) {
            session.invalidate();
            throw new SessionExpiredException("세션 절대 시간 초과");
        }

        // 유휴 타임아웃 (30분)
        if (lastActivity != null &&
            Duration.between(lastActivity, now).toMinutes() > 30) {
            session.invalidate();
            throw new SessionExpiredException("세션 유휴 시간 초과");
        }

        // 활동 시간 업데이트
        session.setAttribute("lastActivity", now);
    }
}
```

### 3. 세션 예측 공격 (Session Prediction)

```java
// 취약한 세션 ID 생성 (하지 말 것!)
public String generateWeakSessionId() {
    // 취약: 예측 가능한 값
    return String.valueOf(System.currentTimeMillis());
}

// 안전한 세션 ID 생성
public String generateSecureSessionId() {
    SecureRandom random = new SecureRandom();
    byte[] bytes = new byte[32];  // 256 bits
    random.nextBytes(bytes);
    return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
}

// Tomcat은 기본적으로 SecureRandom 사용
// context.xml에서 확인/설정 가능
```

### 4. CSRF 공격과 세션

```html
<!-- 공격자 사이트의 악성 폼 -->
<form action="https://bank.com/transfer" method="POST" id="evil-form">
    <input type="hidden" name="to" value="attacker-account">
    <input type="hidden" name="amount" value="1000000">
</form>
<script>document.getElementById('evil-form').submit();</script>

<!-- 피해자가 bank.com에 로그인된 상태에서 공격자 사이트 방문 -->
<!-- 쿠키가 자동으로 전송되어 공격 성공 -->
```

```java
// 방어: CSRF 토큰

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                // 또는 세션에 저장
                // .csrfTokenRepository(new HttpSessionCsrfTokenRepository())
            );
        return http.build();
    }
}

// Thymeleaf에서 자동 포함
<form method="post" action="/transfer">
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}"/>
    ...
</form>

// SPA에서 CSRF 토큰 사용
// 1. 쿠키에서 CSRF 토큰 읽기 (XSRF-TOKEN)
// 2. 요청 헤더에 포함 (X-XSRF-TOKEN)
axios.defaults.xsrfCookieName = 'XSRF-TOKEN';
axios.defaults.xsrfHeaderName = 'X-XSRF-TOKEN';
```

---

## 분산 환경에서의 세션

### Sticky Session

```
┌───────────┐     ┌───────────────┐     ┌───────────┐
│  Client   │────►│ Load Balancer │────►│  Server 1 │ (세션 저장)
└───────────┘     │ (IP/Cookie    │     ├───────────┤
                  │  기반 라우팅)   │     │  Server 2 │
                  └───────────────┘     ├───────────┤
                                        │  Server 3 │
                                        └───────────┘

장점: 간단함, 추가 인프라 불필요
단점: 서버 장애 시 세션 손실, 불균형 부하
```

### Session Replication

```
┌───────────┐     ┌───────────────┐     ┌───────────┐
│  Client   │────►│ Load Balancer │────►│  Server 1 │◄──┐
└───────────┘     └───────────────┘     ├───────────┤   │
                                        │  Server 2 │◄──┼── 세션 복제
                                        ├───────────┤   │
                                        │  Server 3 │◄──┘
                                        └───────────┘

장점: 서버 장애에도 세션 유지
단점: 네트워크 오버헤드, 동기화 지연
```

### 외부 세션 저장소 (권장)

```
┌───────────┐     ┌───────────────┐     ┌───────────┐
│  Client   │────►│ Load Balancer │────►│  Server 1 │──┐
└───────────┘     └───────────────┘     ├───────────┤  │
                                        │  Server 2 │──┼──► Redis
                                        ├───────────┤  │
                                        │  Server 3 │──┘
                                        └───────────┘

장점: 확장성, 서버 독립성
단점: 추가 인프라, 네트워크 지연
```

### Spring Session + Redis 구현

```java
// build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'org.springframework.session:spring-session-data-redis'
}

// 설정
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class RedisSessionConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        return new LettuceConnectionFactory(
            new RedisStandaloneConfiguration(redisHost, redisPort)
        );
    }

    // 세션 직렬화 설정
    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        return new GenericJackson2JsonRedisSerializer();
    }
}
```

```yaml
# application.yml
spring:
  session:
    store-type: redis
    redis:
      namespace: myapp:session
  redis:
    host: redis.example.com
    port: 6379
    password: ${REDIS_PASSWORD}
    lettuce:
      pool:
        max-active: 10
        max-idle: 5
        min-idle: 2
```

```java
// Redis에 저장된 세션 구조
// Key: spring:session:sessions:<session-id>
// 예: spring:session:sessions:abc123-def456

// Hash 구조:
// - creationTime: 1704063600000
// - lastAccessedTime: 1704065400000
// - maxInactiveInterval: 1800
// - sessionAttr:userId: 12345
// - sessionAttr:username: "john"
```

---

## 실무 구현

### 완전한 세션 관리 서비스

```java
@Service
@Slf4j
public class SessionManagementService {

    private final SessionRegistry sessionRegistry;
    private final AuditService auditService;
    private final RedisTemplate<String, Object> redisTemplate;

    // 로그인 처리
    @Transactional
    public LoginResult login(LoginRequest request, HttpServletRequest httpRequest) {
        // 1. 사용자 인증
        User user = authService.authenticate(request.username(), request.password());

        // 2. 기존 세션 확인 및 처리
        handleExistingSessions(user.getUsername(), request.forceLogin());

        // 3. 세션 고정 공격 방지 - 새 세션 생성
        HttpSession oldSession = httpRequest.getSession(false);
        if (oldSession != null) {
            oldSession.invalidate();
        }
        HttpSession session = httpRequest.getSession(true);

        // 4. 세션 속성 설정
        session.setAttribute("userId", user.getId());
        session.setAttribute("username", user.getUsername());
        session.setAttribute("roles", user.getRoles());
        session.setAttribute("loginTime", Instant.now());
        session.setAttribute("clientIp", getClientIp(httpRequest));
        session.setAttribute("userAgent", httpRequest.getHeader("User-Agent"));

        // 5. 감사 로그
        auditService.logLogin(user, httpRequest);

        return LoginResult.success(session.getId());
    }

    // 로그아웃 처리
    public void logout(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            Long userId = (Long) session.getAttribute("userId");

            // 감사 로그
            auditService.logLogout(userId);

            // 세션 무효화
            session.invalidate();
        }
    }

    // 전체 기기에서 로그아웃
    public void logoutFromAllDevices(String username) {
        List<SessionInformation> sessions = sessionRegistry.getAllSessions(
            new UsernamePasswordAuthenticationToken(username, null, Collections.emptyList()),
            false
        );

        sessions.forEach(SessionInformation::expireNow);

        // Redis에서 직접 삭제
        String pattern = "spring:session:sessions:*";
        Set<String> keys = redisTemplate.keys(pattern);
        for (String key : keys) {
            Object storedUsername = redisTemplate.opsForHash()
                .get(key, "sessionAttr:username");
            if (username.equals(storedUsername)) {
                redisTemplate.delete(key);
            }
        }

        auditService.logLogoutAll(username);
    }

    // 활성 세션 목록 조회
    public List<ActiveSessionDto> getActiveSessions(String username) {
        return sessionRegistry.getAllSessions(
            new UsernamePasswordAuthenticationToken(username, null, Collections.emptyList()),
            false
        ).stream()
        .map(this::toActiveSessionDto)
        .toList();
    }

    private ActiveSessionDto toActiveSessionDto(SessionInformation info) {
        return ActiveSessionDto.builder()
            .sessionId(info.getSessionId())
            .lastRequest(info.getLastRequest())
            .expired(info.isExpired())
            .build();
    }

    private void handleExistingSessions(String username, boolean forceLogin) {
        List<SessionInformation> existingSessions = sessionRegistry.getAllSessions(
            new UsernamePasswordAuthenticationToken(username, null, Collections.emptyList()),
            false
        );

        if (!existingSessions.isEmpty()) {
            if (!forceLogin) {
                throw new ConcurrentSessionException("이미 다른 기기에서 로그인 중입니다");
            }
            // 기존 세션 강제 만료
            existingSessions.forEach(SessionInformation::expireNow);
        }
    }

    private String getClientIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddr();
        } else {
            ip = ip.split(",")[0].trim();
        }
        return ip;
    }
}
```

### 세션 보안 필터

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 1)
public class SessionSecurityFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        HttpSession session = request.getSession(false);

        if (session != null && session.getAttribute("userId") != null) {
            // 1. 절대 세션 타임아웃 확인 (8시간)
            Instant loginTime = (Instant) session.getAttribute("loginTime");
            if (loginTime != null &&
                Duration.between(loginTime, Instant.now()).toHours() > 8) {
                session.invalidate();
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                response.getWriter().write("{\"error\": \"세션이 만료되었습니다\"}");
                return;
            }

            // 2. User-Agent 변경 감지
            String storedUserAgent = (String) session.getAttribute("userAgent");
            String currentUserAgent = request.getHeader("User-Agent");
            if (storedUserAgent != null && !storedUserAgent.equals(currentUserAgent)) {
                log.warn("User-Agent 변경 감지 - 세션: {}", session.getId());
                session.invalidate();
                response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                return;
            }

            // 3. 마지막 활동 시간 업데이트
            session.setAttribute("lastActivity", Instant.now());
        }

        filterChain.doFilter(request, response);
    }
}
```

---

## 참고 자료

- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [Spring Session Documentation](https://spring.io/projects/spring-session)
- [RFC 6265 - HTTP State Management Mechanism](https://datatracker.ietf.org/doc/html/rfc6265)
- [SameSite Cookies Explained](https://web.dev/samesite-cookies-explained/)

# OAuth 2.0과 OpenID Connect

## 목차
1. [개요](#개요)
2. [OAuth 2.0 기본 개념](#oauth-20-기본-개념)
3. [OAuth 2.0 Grant Types](#oauth-20-grant-types)
4. [Authorization Code Flow with PKCE](#authorization-code-flow-with-pkce)
5. [OpenID Connect (OIDC)](#openid-connect-oidc)
6. [보안 Best Practices](#보안-best-practices)
7. [실무 구현](#실무-구현)
---

## 개요

### OAuth 2.0이란?

OAuth 2.0은 **인가(Authorization)** 프레임워크로, 사용자의 자격 증명을 공유하지 않고 제3자 애플리케이션에 리소스 접근 권한을 부여하는 표준입니다.

```
전통적 방식:                      OAuth 2.0 방식:
┌─────────────┐                  ┌─────────────┐
│  Third-party │                  │  Third-party │
│     App      │                  │     App      │
└──────┬──────┘                  └──────┬──────┘
       │ 사용자 ID/PW 직접 입력           │ Access Token
       ▼                                ▼
┌─────────────┐                  ┌─────────────┐
│   Resource  │                  │   Resource  │
│    Server   │                  │    Server   │
└─────────────┘                  └─────────────┘

문제점:                          장점:
- 앱이 비밀번호 저장              - 비밀번호 노출 없음
- 비밀번호 노출 위험              - 권한 범위 제한 가능
- 접근 취소 불가                  - 언제든 접근 취소 가능
```

### 관련 RFC

| RFC | 설명 |
|-----|------|
| RFC 6749 | OAuth 2.0 Framework |
| RFC 6750 | Bearer Token Usage |
| RFC 7636 | PKCE (Proof Key for Code Exchange) |
| RFC 7662 | Token Introspection |
| RFC 8252 | OAuth 2.0 for Native Apps |
| RFC 9700 | OAuth 2.0 Security Best Practice |

---

## OAuth 2.0 기본 개념

### 역할 (Roles)

```
┌─────────────────────────────────────────────────────────────────┐
│                         OAuth 2.0 Roles                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐        ┌──────────────────┐               │
│  │  Resource Owner  │        │  Authorization   │               │
│  │     (사용자)      │◄──────►│     Server       │               │
│  └────────┬─────────┘        │  (인가 서버)      │               │
│           │                  └────────┬─────────┘               │
│           │ 권한 위임                  │ Access Token 발급        │
│           ▼                          ▼                          │
│  ┌──────────────────┐        ┌──────────────────┐               │
│  │     Client       │───────►│ Resource Server  │               │
│  │   (클라이언트)    │ Token  │  (리소스 서버)    │               │
│  └──────────────────┘        └──────────────────┘               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

예시 (Google 로그인):
- Resource Owner: 사용자 (본인)
- Client: 제3자 앱 (우리 서비스)
- Authorization Server: Google OAuth Server
- Resource Server: Google API (프로필, 이메일 등)
```

### 토큰 유형

| 토큰 | 설명 | 수명 |
|------|------|------|
| **Access Token** | 리소스 접근 권한 | 짧음 (15분~1시간) |
| **Refresh Token** | Access Token 갱신용 | 김 (일~월 단위) |
| **Authorization Code** | Access Token 교환용 코드 | 매우 짧음 (10분) |
| **ID Token** (OIDC) | 사용자 신원 정보 | 짧음 |

### Scope (권한 범위)

```
# Google OAuth Scopes 예시
scope=openid email profile

# 세분화된 Scope
scope=https://www.googleapis.com/auth/gmail.readonly
      https://www.googleapis.com/auth/calendar.events

# GitHub OAuth Scopes
scope=read:user user:email repo
```

---

## OAuth 2.0 Grant Types

### 1. Authorization Code Grant (권장)

가장 안전하고 널리 사용되는 플로우입니다.

```
     ┌──────────┐                              ┌──────────────────┐
     │  User    │                              │  Authorization   │
     │ (Browser)│                              │     Server       │
     └────┬─────┘                              └────────┬─────────┘
          │                                             │
     ┌────┴─────┐                              ┌────────┴─────────┐
     │  Client  │                              │ Resource Server  │
     │   App    │                              │                  │
     └────┬─────┘                              └────────┬─────────┘
          │                                             │
    1. 로그인 버튼 클릭                                   │
          │                                             │
    2. ───────────────────────────────────────────────► │
       GET /authorize?                                  │
         response_type=code                             │
         &client_id=xxx                                 │
         &redirect_uri=https://app.com/callback         │
         &scope=openid profile email                    │
         &state=xyz123                                  │
          │                                             │
    3. ◄─────────────────────────────────────────────── │
       사용자에게 로그인 및 권한 동의 화면 표시              │
          │                                             │
    4. 사용자가 로그인하고 동의                            │
          │                                             │
    5. ◄─────────────────────────────────────────────── │
       302 Redirect to:                                 │
       https://app.com/callback?code=AUTH_CODE&state=xyz123
          │                                             │
    6. ───────────────────────────────────────────────► │
       POST /token                                      │
         grant_type=authorization_code                  │
         &code=AUTH_CODE                                │
         &redirect_uri=https://app.com/callback         │
         &client_id=xxx                                 │
         &client_secret=yyy                             │
          │                                             │
    7. ◄─────────────────────────────────────────────── │
       {                                                │
         "access_token": "...",                         │
         "token_type": "Bearer",                        │
         "expires_in": 3600,                            │
         "refresh_token": "...",                        │
         "scope": "openid profile email"                │
       }                                                │
          │                                             │
    8. ───────────────────────────────────────────────────────────► │
       GET /api/userinfo                                             │
       Authorization: Bearer {access_token}                          │
          │                                                          │
    9. ◄─────────────────────────────────────────────────────────── │
       { "sub": "123", "name": "John", "email": "john@example.com" } │
```

### 2. Client Credentials Grant

서버 간 통신(M2M)에 사용됩니다.

```
┌──────────────┐                        ┌──────────────────┐
│   Service A  │                        │  Authorization   │
│   (Client)   │                        │     Server       │
└──────┬───────┘                        └────────┬─────────┘
       │                                         │
  1. POST /token                                 │
     grant_type=client_credentials               │
     &client_id=xxx                              │
     &client_secret=yyy                          │
     &scope=read:data                            │
       │─────────────────────────────────────────►
       │                                         │
  2. { "access_token": "...", "expires_in": 3600 }
       ◄─────────────────────────────────────────│
       │                                         │
       │                        ┌────────────────┴───┐
       │                        │  Resource Server   │
       │                        │    (Service B)     │
       │                        └────────────────────┘
  3. GET /api/data                       │
     Authorization: Bearer {token}       │
       │─────────────────────────────────►
       │                                 │
  4. { "data": [...] }                   │
       ◄─────────────────────────────────│
```

### 3. Implicit Grant (Deprecated)

> **주의**: OAuth 2.1에서 제거되었습니다. PKCE를 사용한 Authorization Code Grant를 사용하세요.

```
문제점:
1. Access Token이 URL Fragment에 노출
2. Browser History에 기록됨
3. Referrer 헤더로 유출 가능
4. Token 탈취 시 방어 불가
```

### 4. Resource Owner Password Grant (Deprecated)

> **주의**: OAuth 2.1에서 제거되었습니다. 레거시 시스템 마이그레이션 시에만 사용을 고려하세요.

```
문제점:
1. 클라이언트에 비밀번호 노출
2. 피싱 공격에 취약
3. OAuth의 핵심 목적 위배
```

---

## Authorization Code Flow with PKCE

### PKCE (Proof Key for Code Exchange)란?

PKCE(RFC 7636)는 Public Client(모바일 앱, SPA)에서 Authorization Code를 안전하게 교환하기 위한 확장입니다. OAuth 2.1에서는 모든 클라이언트에 PKCE가 필수입니다.

### PKCE 동작 원리

```
1. 클라이언트가 code_verifier 생성 (43~128자 랜덤 문자열)
2. code_challenge = BASE64URL(SHA256(code_verifier)) 계산
3. 인가 요청 시 code_challenge 전송
4. 토큰 요청 시 code_verifier 전송
5. 서버가 code_verifier로 code_challenge 검증

       ┌────────────────┐                    ┌──────────────────┐
       │     Client     │                    │   Auth Server    │
       └───────┬────────┘                    └────────┬─────────┘
               │                                      │
  code_verifier = random(43~128)                      │
  code_challenge = BASE64URL(SHA256(code_verifier))   │
               │                                      │
  GET /authorize?                                     │
    response_type=code                                │
    &client_id=xxx                                    │
    &code_challenge=XXXX                              │
    &code_challenge_method=S256                       │
    &redirect_uri=...                                 │
    &state=...                                        │
               │──────────────────────────────────────►
               │                          code_challenge 저장
               │                                      │
               ◄────────── code=AUTH_CODE ────────────│
               │                                      │
  POST /token                                         │
    grant_type=authorization_code                     │
    &code=AUTH_CODE                                   │
    &code_verifier=원본값                              │
    &redirect_uri=...                                 │
               │──────────────────────────────────────►
               │                    BASE64URL(SHA256(code_verifier))
               │                    == 저장된 code_challenge 확인
               │                                      │
               ◄────── { access_token, ... } ─────────│
```

### PKCE 구현

```java
// PKCE 유틸리티
public class PKCEUtil {

    public static PKCEChallenge generate() throws Exception {
        // 1. code_verifier 생성 (43~128자)
        SecureRandom random = new SecureRandom();
        byte[] bytes = new byte[32];
        random.nextBytes(bytes);
        String codeVerifier = Base64.getUrlEncoder()
            .withoutPadding()
            .encodeToString(bytes);

        // 2. code_challenge 계산 (SHA256)
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(codeVerifier.getBytes(StandardCharsets.US_ASCII));
        String codeChallenge = Base64.getUrlEncoder()
            .withoutPadding()
            .encodeToString(hash);

        return new PKCEChallenge(codeVerifier, codeChallenge);
    }
}

public record PKCEChallenge(String codeVerifier, String codeChallenge) {}
```

```java
// Spring Security OAuth2 클라이언트 설정
@Configuration
public class OAuth2ClientConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .oauth2Login(oauth2 -> oauth2
                .authorizationEndpoint(authorization -> authorization
                    .authorizationRequestResolver(
                        pkceAuthorizationRequestResolver()
                    )
                )
            );
        return http.build();
    }

    private OAuth2AuthorizationRequestResolver pkceAuthorizationRequestResolver() {
        DefaultOAuth2AuthorizationRequestResolver resolver =
            new DefaultOAuth2AuthorizationRequestResolver(
                clientRegistrationRepository,
                "/oauth2/authorization"
            );

        resolver.setAuthorizationRequestCustomizer(
            OAuth2AuthorizationRequestCustomizers.withPkce()
        );

        return resolver;
    }
}
```

---

## OpenID Connect (OIDC)

### OIDC란?

OpenID Connect는 OAuth 2.0 위에 구축된 **인증(Authentication)** 계층입니다.

```
OAuth 2.0: "이 앱이 내 데이터에 접근해도 될까요?" (인가)
OIDC:     "이 사용자가 누구인가요?" (인증) + OAuth 2.0

┌─────────────────────────────────────────────┐
│            OpenID Connect (OIDC)             │
│  ┌─────────────────────────────────────────┐ │
│  │           OAuth 2.0                      │ │
│  │  ┌─────────────────────────────────────┐ │ │
│  │  │         HTTP / TLS                   │ │ │
│  │  └─────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────┘ │
│                                              │
│  + ID Token (JWT)                            │
│  + UserInfo Endpoint                         │
│  + Standard Scopes (openid, profile, email)  │
│  + Discovery                                 │
└─────────────────────────────────────────────┘
```

### ID Token

ID Token은 JWT 형식으로, 사용자 신원 정보를 담고 있습니다.

```json
// ID Token의 Payload (Claims)
{
  "iss": "https://accounts.google.com",      // 발급자
  "sub": "110169484474386276334",            // 사용자 고유 식별자
  "aud": "1234567890.apps.googleusercontent.com", // 클라이언트 ID
  "exp": 1704067200,                         // 만료 시간
  "iat": 1704063600,                         // 발급 시간
  "auth_time": 1704063500,                   // 인증 시간
  "nonce": "n-0S6_WzA2Mj",                  // Replay Attack 방지
  "acr": "urn:mace:incommon:iap:silver",    // 인증 컨텍스트 클래스
  "amr": ["pwd", "mfa"],                     // 인증 방법
  "azp": "1234567890.apps.googleusercontent.com", // Authorized party

  // 표준 프로필 클레임
  "name": "John Doe",
  "given_name": "John",
  "family_name": "Doe",
  "email": "john@example.com",
  "email_verified": true,
  "picture": "https://example.com/photo.jpg",
  "locale": "ko-KR"
}
```

### OIDC Scopes

| Scope | 반환되는 클레임 |
|-------|---------------|
| `openid` | sub (필수) |
| `profile` | name, family_name, given_name, nickname, picture, etc. |
| `email` | email, email_verified |
| `address` | address (우편주소) |
| `phone` | phone_number, phone_number_verified |

### OIDC 플로우

```java
// Spring Security OIDC 설정
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login/**", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(userInfo -> userInfo
                    .oidcUserService(oidcUserService())
                )
                .successHandler(oAuth2SuccessHandler())
            );
        return http.build();
    }

    @Bean
    public OidcUserService oidcUserService() {
        OidcUserService delegate = new OidcUserService();

        return request -> {
            OidcUser oidcUser = delegate.loadUser(request);

            // ID Token에서 클레임 추출
            String sub = oidcUser.getSubject();
            String email = oidcUser.getEmail();
            String name = oidcUser.getFullName();

            // 사용자 정보 저장/갱신 로직
            User user = userService.findOrCreateUser(sub, email, name);

            // 커스텀 권한 부여
            Set<GrantedAuthority> authorities = new HashSet<>(oidcUser.getAuthorities());
            authorities.addAll(getUserRoles(user));

            return new DefaultOidcUser(authorities, oidcUser.getIdToken(), oidcUser.getUserInfo());
        };
    }
}
```

### OIDC Discovery

```json
// /.well-known/openid-configuration
{
  "issuer": "https://auth.example.com",
  "authorization_endpoint": "https://auth.example.com/authorize",
  "token_endpoint": "https://auth.example.com/token",
  "userinfo_endpoint": "https://auth.example.com/userinfo",
  "jwks_uri": "https://auth.example.com/.well-known/jwks.json",
  "scopes_supported": ["openid", "profile", "email"],
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "claims_supported": ["sub", "name", "email", "picture"]
}
```

---

## 보안 Best Practices

### RFC 9700 권장 사항

```java
// 1. PKCE 필수 사용
@Bean
public OAuth2AuthorizationRequestResolver customResolver(
        ClientRegistrationRepository repo) {
    DefaultOAuth2AuthorizationRequestResolver resolver =
        new DefaultOAuth2AuthorizationRequestResolver(repo, "/oauth2/authorization");

    // PKCE 강제
    resolver.setAuthorizationRequestCustomizer(
        OAuth2AuthorizationRequestCustomizers.withPkce()
    );

    return resolver;
}

// 2. State 파라미터 검증 (CSRF 방지)
@Component
public class OAuth2AuthorizationCodeFilter {

    public void validateState(HttpServletRequest request, HttpSession session) {
        String requestState = request.getParameter("state");
        String sessionState = (String) session.getAttribute("oauth2_state");

        if (requestState == null || !requestState.equals(sessionState)) {
            throw new OAuth2AuthenticationException("State 불일치 - CSRF 공격 의심");
        }
    }
}

// 3. Redirect URI 엄격한 검증
@Configuration
public class AuthorizationServerConfig {

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
            .clientId("my-client")
            .clientSecret("{bcrypt}$2a$10$...")
            // 정확한 URI 매칭만 허용 (와일드카드 금지)
            .redirectUri("https://app.example.com/callback")
            .redirectUri("https://app.example.com/oauth2/callback")
            // Implicit, Password Grant 비활성화
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .build();

        return new InMemoryRegisteredClientRepository(client);
    }
}
```

### Token 보안

```java
// 4. Access Token 짧은 수명
@Bean
public TokenSettings tokenSettings() {
    return TokenSettings.builder()
        .accessTokenTimeToLive(Duration.ofMinutes(15))  // 15분
        .refreshTokenTimeToLive(Duration.ofDays(7))     // 7일
        .reuseRefreshTokens(false)  // Refresh Token Rotation
        .build();
}

// 5. Refresh Token Rotation 구현
@Service
public class TokenService {

    @Transactional
    public TokenResponse refreshAccessToken(String refreshToken) {
        // 기존 Refresh Token 검증
        RefreshToken storedToken = refreshTokenRepository.findByToken(refreshToken)
            .orElseThrow(() -> new InvalidTokenException("유효하지 않은 Refresh Token"));

        if (storedToken.isExpired()) {
            refreshTokenRepository.delete(storedToken);
            throw new TokenExpiredException("Refresh Token 만료");
        }

        // 기존 Refresh Token 폐기
        refreshTokenRepository.delete(storedToken);

        // 새 토큰 발급
        String newAccessToken = generateAccessToken(storedToken.getUser());
        String newRefreshToken = generateRefreshToken(storedToken.getUser());

        // 새 Refresh Token 저장
        refreshTokenRepository.save(new RefreshToken(
            newRefreshToken,
            storedToken.getUser(),
            Instant.now().plus(Duration.ofDays(7))
        ));

        return new TokenResponse(newAccessToken, newRefreshToken);
    }
}

// 6. Sender-Constrained Tokens (DPoP)
// Proof of Possession 토큰 - 토큰 도용 방지
@Component
public class DPoPValidator {

    public boolean validateDPoP(String dpopProof, String accessToken, HttpServletRequest request) {
        // DPoP 증명 검증 로직
        try {
            SignedJWT dpopJwt = SignedJWT.parse(dpopProof);
            JWTClaimsSet claims = dpopJwt.getJWTClaimsSet();

            // HTTP 메서드 확인
            if (!claims.getStringClaim("htm").equals(request.getMethod())) {
                return false;
            }

            // URI 확인
            if (!claims.getStringClaim("htu").equals(request.getRequestURL().toString())) {
                return false;
            }

            // Access Token 바인딩 확인
            String ath = Base64URL.encode(
                MessageDigest.getInstance("SHA-256")
                    .digest(accessToken.getBytes(StandardCharsets.US_ASCII))
            ).toString();

            return ath.equals(claims.getStringClaim("ath"));
        } catch (Exception e) {
            return false;
        }
    }
}
```

### Scope 최소화

```java
// 7. 최소 권한 원칙 적용
@GetMapping("/connect/google")
public String initiateGoogleLogin(HttpServletRequest request) {
    // 필요한 최소한의 scope만 요청
    String scope = "openid email";  // profile 불필요하면 제외

    String authorizationUrl = UriComponentsBuilder
        .fromUriString("https://accounts.google.com/o/oauth2/v2/auth")
        .queryParam("client_id", clientId)
        .queryParam("redirect_uri", redirectUri)
        .queryParam("response_type", "code")
        .queryParam("scope", scope)
        .queryParam("state", generateState(request))
        .queryParam("code_challenge", pkce.codeChallenge())
        .queryParam("code_challenge_method", "S256")
        .build()
        .toUriString();

    return "redirect:" + authorizationUrl;
}
```

---

## 실무 구현

### Spring Boot OAuth2 Resource Server

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          # 또는 jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```

```java
@Configuration
@EnableWebSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );
        return http.build();
    }

    private JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter =
            new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthorityPrefix("ROLE_");
        authoritiesConverter.setAuthoritiesClaimName("roles");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}
```

### Google OAuth2 로그인 구현

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, email, profile
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
```

```java
@RestController
public class OAuth2Controller {

    @GetMapping("/api/user")
    public Map<String, Object> user(@AuthenticationPrincipal OidcUser oidcUser) {
        return Map.of(
            "name", oidcUser.getFullName(),
            "email", oidcUser.getEmail(),
            "picture", oidcUser.getPicture(),
            "provider", "google"
        );
    }

    @GetMapping("/api/logout")
    public ResponseEntity<Void> logout(HttpServletRequest request, HttpServletResponse response) {
        SecurityContextLogoutHandler handler = new SecurityContextLogoutHandler();
        handler.logout(request, response, SecurityContextHolder.getContext().getAuthentication());

        return ResponseEntity.ok().build();
    }
}
```

### Token Introspection 구현

```java
@Service
public class TokenIntrospectionService {

    private final RestTemplate restTemplate;
    private final String introspectionEndpoint;
    private final String clientId;
    private final String clientSecret;

    public TokenInfo introspect(String accessToken) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        headers.setBasicAuth(clientId, clientSecret);

        MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
        body.add("token", accessToken);
        body.add("token_type_hint", "access_token");

        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(body, headers);

        TokenInfo response = restTemplate.postForObject(
            introspectionEndpoint,
            request,
            TokenInfo.class
        );

        if (response == null || !response.active()) {
            throw new InvalidTokenException("토큰이 유효하지 않습니다.");
        }

        return response;
    }
}

public record TokenInfo(
    boolean active,
    String sub,
    String clientId,
    List<String> scope,
    Long exp,
    Long iat
) {}
```

---

## 참고 자료

- [RFC 6749 - OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7636 - PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 9700 - OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/rfc9700/)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OWASP OAuth2 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)

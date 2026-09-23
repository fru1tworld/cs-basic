# OAuth 2.0

- Authorization Code 흐름에서는 PKCE를 적용하고 리디렉션 URI를 엄격하게 검증해야 함. Implicit 방식은 권장되지 않으며 Resource Owner Password Credentials 방식은 사용 금지임. 뒤의 레거시 흐름 설명은 기존 시스템 이해를 위한 내용임. [OAuth 2.0 보안 권고 RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html)

## 개요

OAuth 2.0은 권한 위임(Authorization Delegation)을 위한 표준 프로토콜이다. 사용자는 비밀번호를 제3자 애플리케이션에 넘기는 대신, 필요한 리소스에 접근할 권한을 부여한다.

## 역할 정의

이 권한 위임에는 네 가지 역할이 참여한다.

| 역할 | 담당 |
| --- | --- |
| Resource Owner | 권한을 부여하는 사용자 |
| Client | 권한을 받아 사용하는 제3자 애플리케이션 |
| Authorization Server | 토큰을 발급하는 인증 서버 |
| Resource Server | 보호된 리소스를 제공하는 API 서버 |

## Authorization Code Flow (권장)

- 가장 안전한 플로우로, 서버 사이드 애플리케이션에 적합함.

사용자가 클라이언트에서 로그인을 시작하면 인가 서버로 이동해 인증하고 권한 제공에 동의한다. 인가 서버는 Authorization Code를 담아 클라이언트로 리디렉션한다.

클라이언트는 이 코드를 인가 서버에 보내 Access Token과 Refresh Token으로 교환한다. 이후 Access Token을 담아 리소스 서버의 API를 호출하고 보호된 리소스를 받는다. 아래 코드는 리디렉션부터 코드 교환, 세션 생성까지의 흐름을 보여준다.

```javascript
// Step 1-2: 인증 페이지로 리다이렉트
app.get('/auth/google', (req, res) => {
  const authUrl = new URL('https://accounts.google.com/o/oauth2/v2/auth');
  authUrl.searchParams.set('client_id', process.env.GOOGLE_CLIENT_ID);
  authUrl.searchParams.set('redirect_uri', 'https://myapp.com/auth/callback');
  authUrl.searchParams.set('response_type', 'code');
  authUrl.searchParams.set('scope', 'openid email profile');
  authUrl.searchParams.set('state', generateState()); // CSRF 방지
  authUrl.searchParams.set('code_challenge', generateCodeChallenge()); // PKCE
  authUrl.searchParams.set('code_challenge_method', 'S256');

  res.redirect(authUrl.toString());
});

// Step 4-7: 콜백 처리 및 토큰 교환
app.get('/auth/callback', async (req, res) => {
  const { code, state } = req.query;

  // state 검증 (CSRF 방지)
  if (!verifyState(state)) {
    return res.status(400).json({ error: 'Invalid state' });
  }

  // 토큰 교환
  const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: 'https://myapp.com/auth/callback',
      client_id: process.env.GOOGLE_CLIENT_ID,
      client_secret: process.env.GOOGLE_CLIENT_SECRET,
      code_verifier: getCodeVerifier() // PKCE
    })
  });

  const tokens = await tokenResponse.json();
  // { access_token, refresh_token, expires_in, token_type, id_token }

  // 사용자 정보 조회 또는 세션 생성
  const userInfo = await getUserInfo(tokens.access_token);
  req.session.user = userInfo;

  res.redirect('/dashboard');
});
```

## Authorization Code Flow with PKCE

- Public Client(SPA, 모바일 앱)를 위한 확장으로, client_secret 없이 안전하게 동작함.

```javascript
// PKCE 구현
const crypto = require('crypto');

// Code Verifier: 43-128자의 랜덤 문자열
function generateCodeVerifier() {
  return crypto.randomBytes(32).toString('base64url');
}

// Code Challenge: Code Verifier의 SHA256 해시
function generateCodeChallenge(verifier) {
  return crypto
    .createHash('sha256')
    .update(verifier)
    .digest('base64url');
}

// 사용 예시
const codeVerifier = generateCodeVerifier();
const codeChallenge = generateCodeChallenge(codeVerifier);

// 1. 인증 요청 시 code_challenge 전송
// 2. 토큰 교환 시 code_verifier 전송
// 3. 서버가 code_verifier를 해시하여 code_challenge와 비교
```

### PKCE (Proof Key for Code Exchange)란?

- PKCE(RFC 7636)는 Public Client(모바일 앱, SPA)에서 Authorization Code를 안전하게 교환하기 위한 확장임. OAuth 2.1에서는 모든 클라이언트에 PKCE가 필수임.

### PKCE 동작 원리

클라이언트는 43~128자의 랜덤 문자열인 `code_verifier`를 만들고, `code_challenge = BASE64URL(SHA256(code_verifier))`를 계산한다. 인가 요청에는 원본 대신 `code_challenge`와 `code_challenge_method=S256`을 보낸다. 이때 `GET /authorize`에는 `response_type=code`, `client_id=xxx`, `redirect_uri`, `state`도 함께 담는다.

인가 서버가 `code_challenge`를 저장하고 `code=AUTH_CODE`를 반환하면, 클라이언트는 `POST /token`으로 코드를 교환한다. 이 요청에는 `grant_type=authorization_code`, `code=AUTH_CODE`, `redirect_uri`와 원본 `code_verifier`가 들어간다. 서버는 받은 원본을 같은 방식으로 해시해 저장한 `code_challenge`와 비교하고, 일치하면 `access_token` 등을 반환한다.

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

## Client Credentials Flow

- 서버 간 통신(Machine-to-Machine)에 사용됨.

```javascript
async function getM2MToken() {
  const response = await fetch('https://auth.example.com/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: process.env.CLIENT_ID,
      client_secret: process.env.CLIENT_SECRET,
      scope: 'read:users write:users'
    })
  });

  return response.json();
  // { access_token, expires_in, token_type }
}
```

## Refresh Token Flow

- Access Token 만료 시 새 토큰을 발급받음.

```javascript
async function refreshAccessToken(refreshToken) {
  const response = await fetch('https://auth.example.com/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'refresh_token',
      refresh_token: refreshToken,
      client_id: process.env.CLIENT_ID,
      client_secret: process.env.CLIENT_SECRET
    })
  });

  if (!response.ok) {
    throw new Error('Token refresh failed');
  }

  return response.json();
  // { access_token, refresh_token (optional), expires_in }
}
```

## OAuth 2.0 보안 모범 사례 (RFC 6819, OAuth 2.1)

- PKCE 필수 사용 (모든 클라이언트)
- Implicit Flow 사용 금지 (OAuth 2.1에서 제거)
- state 파라미터로 CSRF 방지
- redirect_uri 정확히 일치하도록 검증
- 짧은 Access Token 수명 (15분-1시간)
- Refresh Token 로테이션
- HTTPS 필수
- Token 안전한 저장 (httpOnly 쿠키 권장)

## OAuth 2.0이란?

- OAuth 2.0은 인가(Authorization) 프레임워크로, 사용자의 자격 증명을 공유하지 않고 제3자 애플리케이션에 리소스 접근 권한을 부여하는 표준임.

- App:
  - Third-party: App
- Resource:
  - Third-party: Resource
- Server:
  - Third-party: Server
- 전통적 방식:                      OAuth 2.0 방식:
- | 사용자 ID/PW 직접 입력           | Access Token
- 문제점:                          장점:
- 앱이 비밀번호 저장              - 비밀번호 노출 없음
- 비밀번호 노출 위험              - 권한 범위 제한 가능
- 접근 취소 불가                  - 언제든 접근 취소 가능

## 관련 RFC

- RFC: RFC 6749:
  - 설명: OAuth 2.0 Framework
- RFC: RFC 6750:
  - 설명: Bearer Token Usage
- RFC: RFC 7636:
  - 설명: PKCE (Proof Key for Code Exchange)
- RFC: RFC 7662:
  - 설명: Token Introspection
- RFC: RFC 8252:
  - 설명: OAuth 2.0 for Native Apps
- RFC: RFC 9700:
  - 설명: OAuth 2.0 Security Best Practice

## OAuth 2.0 기본 개념

### 역할 (Roles)

앞의 역할을 Google 로그인에 대입하면 Resource Owner는 사용자 본인, Client는 우리 서비스다. Google OAuth Server가 Authorization Server로서 Access Token을 발급하고, 우리 서비스는 이 토큰으로 Resource Server인 Google API에서 프로필이나 이메일 등의 리소스에 접근한다.

### 토큰 유형

- 토큰: Access Token:
  - 설명: 리소스 접근 권한
  - 수명: 짧음 (15분~1시간)
- 토큰: Refresh Token:
  - 설명: Access Token 갱신용
  - 수명: 김 (일~월 단위)
- 토큰: Authorization Code:
  - 설명: Access Token 교환용 코드
  - 수명: 매우 짧음 (10분)
- 토큰: ID Token (OIDC):
  - 설명: 사용자 신원 정보
  - 수명: 짧음

### Scope (권한 범위)

- # Google OAuth Scopes 예시
- scope=openid email profile
- # 세분화된 Scope
- scope=https://www.googleapis.com/auth/gmail.readonly
- https://www.googleapis.com/auth/calendar.events
- # GitHub OAuth Scopes
- scope=read:user user:email repo

## OAuth 2.0 Grant Types

### Authorization Code Grant (권장)

- 가장 안전하고 널리 사용되는 플로우임.

- (Browser):
  - Authorization: Server
- Client:
  - Authorization: Resource Server
- App:

- 1. 로그인 버튼 클릭                                   |
- 2. ───────────────────────────────────────────────► |
- GET /authorize?                                  |
- response_type=code                             |
- &client_id=xxx                                 |
- &redirect_uri=https://app.com/callback         |
- &scope=openid profile email                    |
- &state=xyz123                                  |
- 3. ◄─────────────────────────────────────────────── |
- 사용자에게 로그인 및 권한 동의 화면 표시              |
- 4. 사용자가 로그인하고 동의                            |
- 5. ◄─────────────────────────────────────────────── |
- 302 Redirect to:                                 |
- https://app.com/callback?code=AUTH_CODE&state=xyz123
- 6. ───────────────────────────────────────────────► |
- POST /token                                      |
- grant_type=authorization_code                  |
- &code=AUTH_CODE                                |
- &redirect_uri=https://app.com/callback         |
- &client_id=xxx                                 |
- &client_secret=yyy                             |
- 7. ◄─────────────────────────────────────────────── |
- "access_token": "...",                         |
- "token_type": "Bearer",                        |
- "expires_in": 3600,                            |
- "refresh_token": "...",                        |
- "scope": "openid profile email"                |
- 8. ───────────────────────────────────────────────────────────► |
- GET /api/userinfo                                             |
- Authorization: Bearer {access_token}                          |
- 9. ◄─────────────────────────────────────────────────────────── |
- { "sub": "123", "name": "John", "email": "john@example.com" } |

### Client Credentials Grant

- 서버 간 통신(M2M)에 사용됨.

- Service A, Authorization
- (Client), Server
- POST /token
- grant_type=client_credentials
- &client_id=xxx
- &client_secret=yyy
- &scope=read:data
- { "access_token": "...", "expires_in": 3600 }
- Resource Server
- (Service B)
- GET /api/data
- Authorization: Bearer {token}
- { "data": [...] }

### Implicit Grant (Deprecated)

- 주의: OAuth 2.1에서 제거되었음. PKCE를 사용한 Authorization Code Grant를 사용하세요.

- 문제점:
- Access Token이 URL Fragment에 노출
- Browser History에 기록됨
- Referrer 헤더로 유출 가능
- Token 탈취 시 방어 불가

### Resource Owner Password Grant (Deprecated)

- 주의: OAuth 2.1에서 제거되었음. 레거시 시스템 마이그레이션 시에만 사용을 고려하세요.

- 문제점:
- 클라이언트에 비밀번호 노출
- 피싱 공격에 취약
- OAuth의 핵심 목적 위배

## OpenID Connect (OIDC)

### OIDC란?

OAuth 2.0이 "이 앱이 내 데이터에 접근해도 될까요?"라는 인가 문제를 다룬다면, 로그인에는 "이 사용자가 누구인가요?"라는 인증 정보도 필요하다. OpenID Connect(OIDC)는 이를 위해 OAuth 2.0 위에 구축한 인증(Authentication) 계층이다.

HTTP/TLS와 OAuth 2.0을 기반으로 ID Token(JWT), UserInfo Endpoint, 표준 Scope(`openid`, `profile`, `email`), Discovery를 제공한다. 이 중 사용자 신원 정보는 다음 ID Token에서 확인할 수 있다.

### ID Token

- ID Token은 JWT 형식으로, 사용자 신원 정보를 담고 있음.

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

- Scope: `openid`:
  - 반환되는 클레임: sub (필수)
- Scope: `profile`:
  - 반환되는 클레임: name, family_name, given_name, nickname, picture, etc.
- Scope: `email`:
  - 반환되는 클레임: email, email_verified
- Scope: `address`:
  - 반환되는 클레임: address (우편주소)
- Scope: `phone`:
  - 반환되는 클레임: phone_number, phone_number_verified

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

## 참고 자료

- [RFC 6749 - OAuth 2.0](https://tools.ietf.org/html/rfc6749)
- [RFC 7519 - JWT](https://tools.ietf.org/html/rfc7519)
- [RFC 6819 - OAuth 2.0 Threat Model](https://tools.ietf.org/html/rfc6819)
- [OAuth 2.1 Draft](https://oauth.net/2.1/)
- [JWT.io](https://jwt.io/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Rate Limiting Algorithms](https://konghq.com/blog/how-to-design-a-scalable-rate-limiting-algorithm)

- [RFC 6749 - OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [RFC 7636 - PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 9700 - OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/rfc9700/)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OWASP OAuth2 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html)

## 관련 학습

- [인증과 인가 설계](01-인증과-인가-설계.md)
- [JWT](03-JWT.md)
- [TLS와 HTTPS](../암호화와-전송-보안/02-TLS와-HTTPS.md)

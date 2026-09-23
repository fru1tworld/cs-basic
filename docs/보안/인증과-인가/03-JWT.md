# JWT

## 개요

JWT(JSON Web Token)는 당사자 간에 정보를 전달하는 간결한 토큰 형식이다. 필요한 정보를 토큰 자체에 담는 self-contained 방식이며, 아래에서는 헤더와 페이로드에 서명을 붙이는 구조를 살펴본다.

## JWT 구조

아래 JWT는 `Header.Payload.Signature`처럼 점으로 구분한 세 부분으로 구성된다. `xxxxx.yyyyy.zzzzz`에서 첫 부분은 Header(헤더), 둘째는 Payload(내용), 셋째는 Signature(서명)에 해당한다.

## Header (헤더)

```json
{
  "alg": "HS256",    // 서명 알고리즘
  "typ": "JWT"       // 토큰 타입
}

// Base64Url 인코딩
// eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

- 지원 알고리즘:
- HMAC: HS256, HS384, HS512 (대칭키)
- RSA: RS256, RS384, RS512 (비대칭키)
- ECDSA: ES256, ES384, ES512 (비대칭키)

## Payload (페이로드)

```json
{
  // Registered Claims (등록된 클레임)
  "iss": "https://auth.example.com",     // Issuer (발급자)
  "sub": "user123",                       // Subject (주체)
  "aud": "https://api.example.com",       // Audience (대상)
  "exp": 1735344000,                      // Expiration (만료 시간)
  "nbf": 1735340400,                      // Not Before (시작 시간)
  "iat": 1735340400,                      // Issued At (발급 시간)
  "jti": "unique-token-id",               // JWT ID (고유 식별자)

  // Public Claims (공개 클레임)
  "email": "user@example.com",

  // Private Claims (비공개 클레임)
  "role": "admin",
  "permissions": ["read", "write"]
}
```

## Signature (서명)

서명은 인코딩한 헤더와 페이로드를 점으로 이어 붙인 값을 대상으로 계산한다. 다음 예제는 HMAC SHA256에서 비밀키를, RSA SHA256에서 개인키를 사용하는 모습을 비교한다.

```javascript
// HMAC SHA256 서명
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)

// RSA SHA256 서명
RSASHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  privateKey
)
```

## JWT 생성 및 검증 (Node.js)

```javascript
const jwt = require('jsonwebtoken');

// 비밀키 (환경 변수에서 로드)
const ACCESS_TOKEN_SECRET = process.env.ACCESS_TOKEN_SECRET;
const REFRESH_TOKEN_SECRET = process.env.REFRESH_TOKEN_SECRET;

class JWTService {
  // Access Token 생성
  generateAccessToken(user) {
    return jwt.sign(
      {
        sub: user.id,
        email: user.email,
        role: user.role,
        type: 'access'
      },
      ACCESS_TOKEN_SECRET,
      {
        expiresIn: '15m',       // 15분
        issuer: 'myapp.com',
        audience: 'myapp.com'
      }
    );
  }

  // Refresh Token 생성
  generateRefreshToken(user) {
    return jwt.sign(
      {
        sub: user.id,
        type: 'refresh',
        tokenVersion: user.tokenVersion  // 토큰 무효화용
      },
      REFRESH_TOKEN_SECRET,
      {
        expiresIn: '7d',        // 7일
        issuer: 'myapp.com'
      }
    );
  }

  // Access Token 검증
  verifyAccessToken(token) {
    try {
      return jwt.verify(token, ACCESS_TOKEN_SECRET, {
        issuer: 'myapp.com',
        audience: 'myapp.com'
      });
    } catch (error) {
      if (error.name === 'TokenExpiredError') {
        throw new Error('Token expired');
      }
      if (error.name === 'JsonWebTokenError') {
        throw new Error('Invalid token');
      }
      throw error;
    }
  }

  // Refresh Token 검증
  async verifyRefreshToken(token, user) {
    try {
      const decoded = jwt.verify(token, REFRESH_TOKEN_SECRET, {
        issuer: 'myapp.com'
      });

      // tokenVersion 확인 (강제 로그아웃 시 버전 증가)
      if (decoded.tokenVersion !== user.tokenVersion) {
        throw new Error('Token revoked');
      }

      return decoded;
    } catch (error) {
      throw error;
    }
  }
}
```

## JWT 인증 미들웨어

```javascript
function jwtAuth(jwtService) {
  return async (req, res, next) => {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        error: 'Access token required',
        code: 'MISSING_TOKEN'
      });
    }

    const token = authHeader.substring(7);

    try {
      const decoded = jwtService.verifyAccessToken(token);
      req.user = {
        id: decoded.sub,
        email: decoded.email,
        role: decoded.role
      };
      next();
    } catch (error) {
      if (error.message === 'Token expired') {
        return res.status(401).json({
          error: 'Token expired',
          code: 'TOKEN_EXPIRED'
        });
      }
      return res.status(401).json({
        error: 'Invalid token',
        code: 'INVALID_TOKEN'
      });
    }
  };
}
```

## JWT 토큰 갱신 엔드포인트

```javascript
app.post('/auth/refresh', async (req, res) => {
  const { refreshToken } = req.body;

  if (!refreshToken) {
    return res.status(400).json({ error: 'Refresh token required' });
  }

  try {
    // Refresh Token에서 사용자 ID 추출
    const decoded = jwt.decode(refreshToken);
    const user = await userService.findById(decoded.sub);

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    // Refresh Token 검증
    await jwtService.verifyRefreshToken(refreshToken, user);

    // 새 토큰 발급
    const newAccessToken = jwtService.generateAccessToken(user);
    const newRefreshToken = jwtService.generateRefreshToken(user);

    res.json({
      accessToken: newAccessToken,
      refreshToken: newRefreshToken
    });
  } catch (error) {
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});
```

## JWT 무효화 전략

토큰을 발급한 뒤에는 만료 전에 접근을 막아야 하는 경우도 생긴다. 아래에서는 서버에 토큰 버전을 저장하는 방법, Redis Blocklist를 확인하는 방법, Access Token의 수명을 짧게 두고 Refresh Token을 관리하는 방법을 비교한다.

```javascript
// 1. Token Version (DB에 저장)
// 강제 로그아웃 시 tokenVersion 증가

// 2. Blocklist (Redis)
async function revokeToken(token) {
  const decoded = jwt.decode(token);
  const ttl = decoded.exp - Math.floor(Date.now() / 1000);

  if (ttl > 0) {
    await redis.setex(`blocklist:${token}`, ttl, '1');
  }
}

async function isTokenRevoked(token) {
  const exists = await redis.exists(`blocklist:${token}`);
  return exists === 1;
}

// 3. 짧은 만료 시간 + Refresh Token
// Access Token: 15분
// Refresh Token: 7일 (DB에서 관리)
```

## JWT vs Session 비교

- 측면: 저장 위치:
  - JWT: 클라이언트
  - Session: 서버 (메모리/DB)
- 측면: 상태:
  - JWT: Stateless
  - Session: Stateful
- 측면: 확장성:
  - JWT: 높음 (서버 간 공유 불필요)
  - Session: 낮음 (세션 공유 필요)
- 측면: 무효화:
  - JWT: 어려움
  - Session: 쉬움
- 측면: 크기:
  - JWT: 큼 (매 요청 전송)
  - Session: 작음 (Session ID만)
- 측면: 보안:
  - JWT: 토큰 탈취 위험
  - Session: CSRF 위험

## JWT란?

- JWT(JSON Web Token)는 RFC 7519에 정의된 토큰 형식으로, JSON 객체를 안전하게 전송하기 위한 컴팩트하고 자가 포함적인(self-contained) 방식임.

- JWT 특징:
- Stateless: 서버에 세션 저장 불필요
- Self-contained: 필요한 정보를 토큰 자체에 포함
- Compact: URL, HTTP 헤더에 포함하기 적합
- Verifiable: 디지털 서명으로 무결성 검증 가능

## 세션 vs JWT 비교

세션 기반 인증에서는 로그인 시 서버가 세션 저장소에 상태를 저장하고 클라이언트에 쿠키를 전달한다. 이후 요청마다 Session ID로 저장소를 조회해 사용자를 확인한다.

여기서 비교하는 JWT 기반 인증에서는 서버가 로그인 결과로 JWT를 발급하고, 클라이언트가 localStorage나 쿠키에 토큰을 저장한다. 서버는 별도 세션 저장소를 조회하는 대신 요청에 담긴 토큰의 서명을 검증한다.

- 특성: 상태 저장:
  - 세션: 서버에 저장 (Stateful)
  - JWT: 토큰에 포함 (Stateless)
- 특성: 확장성:
  - 세션: 분산 환경에서 세션 공유 필요
  - JWT: 서버 간 공유 불필요
- 특성: 보안:
  - 세션: 세션 ID 탈취 시 무효화 쉬움
  - JWT: 토큰 탈취 시 만료까지 유효
- 특성: 크기:
  - 세션: Session ID만 전송 (작음)
  - JWT: 클레임 포함 (상대적으로 큼)
- 특성: 폐기:
  - 세션: 서버에서 즉시 무효화 가능
  - JWT: 즉시 무효화 어려움

## 기본 구조

- JWT = Header.Payload.Signature
- eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
- eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
- SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
- Header (JOSE Header)
- Base64URL 인코딩
- "alg": "HS256",, // 서명 알고리즘
- "typ": "JWT", // 토큰 타입
- Payload (Claims)
- Base64URL 인코딩
- "sub": "1234567890",, // Subject (사용자 식별자)
- "name": "John Doe",, // 사용자 정의 클레임
- "iat": 1516239022,, // Issued At (발급 시간)
- "exp": 1516242622, // Expiration (만료 시간)
- Signature
- Base64URL 인코딩
- HMACSHA256(
- base64UrlEncode(header) + "." +
- base64UrlEncode(payload)
- secret

## Registered Claims (등록된 클레임)

- 클레임: `iss`:
  - 이름: Issuer
  - 설명: 토큰 발급자
- 클레임: `sub`:
  - 이름: Subject
  - 설명: 토큰 주체 (사용자 ID)
- 클레임: `aud`:
  - 이름: Audience
  - 설명: 토큰 대상자 (수신자)
- 클레임: `exp`:
  - 이름: Expiration Time
  - 설명: 만료 시간 (UNIX timestamp)
- 클레임: `nbf`:
  - 이름: Not Before
  - 설명: 토큰 활성 시작 시간
- 클레임: `iat`:
  - 이름: Issued At
  - 설명: 토큰 발급 시간
- 클레임: `jti`:
  - 이름: JWT ID
  - 설명: 토큰 고유 식별자 (Replay 공격 방지)

## Payload 예시

```json
{
  // Registered Claims
  "iss": "https://auth.example.com",
  "sub": "user-12345",
  "aud": ["https://api.example.com", "https://app.example.com"],
  "exp": 1704067200,
  "nbf": 1704063600,
  "iat": 1704063600,
  "jti": "unique-token-id-abc123",

  // Public Claims (표준화된 이름 사용 권장)
  "name": "John Doe",
  "email": "john@example.com",
  "email_verified": true,

  // Private Claims (애플리케이션 특화)
  "roles": ["user", "admin"],
  "permissions": ["read:users", "write:users"],
  "tenant_id": "org-789"
}
```

## JWT 서명 알고리즘

### 대칭키 알고리즘 (HMAC)

```java
// HS256, HS384, HS512
// 장점: 빠름, 구현 간단
// 단점: 키 공유 필요, 발급자와 검증자가 동일 키 사용

// 서명 및 검증
public class HmacJwtService {
    private static final String SECRET = "very-long-secret-key-at-least-256-bits";

    public String createToken(Map<String, Object> claims) {
        return Jwts.builder()
            .setClaims(claims)
            .setIssuedAt(new Date())
            .setExpiration(Date.from(Instant.now().plus(Duration.ofHours(1))))
            .signWith(Keys.hmacShaKeyFor(SECRET.getBytes()), SignatureAlgorithm.HS256)
            .compact();
    }

    public Claims validateToken(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(Keys.hmacShaKeyFor(SECRET.getBytes()))
            .build()
            .parseClaimsJws(token)
            .getBody();
    }
}
```

### 비대칭키 알고리즘 (RSA, ECDSA)

```java
// RS256, RS384, RS512 (RSA)
// ES256, ES384, ES512 (ECDSA)
// 장점: 개인키는 발급자만, 공개키로 누구나 검증 가능
// 단점: HMAC보다 느림

public class RsaJwtService {
    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;

    public RsaJwtService() throws Exception {
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("RSA");
        keyGen.initialize(2048);
        KeyPair pair = keyGen.generateKeyPair();
        this.privateKey = (RSAPrivateKey) pair.getPrivate();
        this.publicKey = (RSAPublicKey) pair.getPublic();
    }

    // 발급 서버에서만 사용 (개인키 필요)
    public String createToken(Map<String, Object> claims) {
        return Jwts.builder()
            .setClaims(claims)
            .setIssuedAt(new Date())
            .setExpiration(Date.from(Instant.now().plus(Duration.ofHours(1))))
            .signWith(privateKey, SignatureAlgorithm.RS256)
            .compact();
    }

    // 어떤 서버에서든 검증 가능 (공개키만 필요)
    public Claims validateToken(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(publicKey)
            .build()
            .parseClaimsJws(token)
            .getBody();
    }
}
```

### ECDSA (권장)

```java
// ES256: P-256 + SHA-256
// 더 짧은 키로 RSA와 동등한 보안성
// 서명 크기도 작음

public class EcdsaJwtService {
    private final ECPrivateKey privateKey;
    private final ECPublicKey publicKey;

    public EcdsaJwtService() throws Exception {
        KeyPairGenerator keyGen = KeyPairGenerator.getInstance("EC");
        keyGen.initialize(new ECGenParameterSpec("secp256r1")); // P-256
        KeyPair pair = keyGen.generateKeyPair();
        this.privateKey = (ECPrivateKey) pair.getPrivate();
        this.publicKey = (ECPublicKey) pair.getPublic();
    }

    public String createToken(Map<String, Object> claims) {
        return Jwts.builder()
            .setClaims(claims)
            .signWith(privateKey, SignatureAlgorithm.ES256)
            .compact();
    }
}
```

### 알고리즘 비교

- 알고리즘: HS256:
  - 키 길이: 256 bits
  - 서명 크기: 32 bytes
  - 성능: 가장 빠름
  - 사용 사례: 단일 서버, 내부 서비스
- 알고리즘: RS256:
  - 키 길이: 2048 bits
  - 서명 크기: 256 bytes
  - 성능: 느림
  - 사용 사례: 분산 시스템, 공개키 배포
- 알고리즘: ES256:
  - 키 길이: 256 bits
  - 서명 크기: 64 bytes
  - 성능: 빠름
  - 사용 사례: 모바일, 분산 시스템 (권장)

## JWT 보안 취약점

### None Algorithm Attack

```java
// 취약한 코드 - alg 헤더를 신뢰
public Claims parseToken(String token) {
    // 위험: 토큰의 alg 헤더 값을 그대로 사용
    return Jwts.parserBuilder()
        .setSigningKeyResolver(new SigningKeyResolverAdapter() {
            @Override
            public Key resolveSigningKey(JwsHeader header, Claims claims) {
                String alg = header.getAlgorithm();
                if ("none".equals(alg)) {
                    return null; // 서명 없이 통과!
                }
                return getKey(alg);
            }
        })
        .build()
        .parseClaimsJws(token)
        .getBody();
}

// 공격 페이로드
// Header: { "alg": "none", "typ": "JWT" }
// Payload: { "sub": "admin", "role": "admin" }
// Signature: (없음)
```

```java
// 안전한 코드 - 알고리즘 화이트리스트
public Claims parseToken(String token) {
    return Jwts.parserBuilder()
        .setSigningKey(secretKey)
        // JJWT는 기본적으로 "none" 알고리즘을 거부함
        .build()
        .parseClaimsJws(token)
        .getBody();
}

// 또는 명시적 알고리즘 검증
public Claims parseTokenSafe(String token) {
    Jws<Claims> jws = Jwts.parserBuilder()
        .setSigningKey(secretKey)
        .build()
        .parseClaimsJws(token);

    String algorithm = jws.getHeader().getAlgorithm();
    if (!ALLOWED_ALGORITHMS.contains(algorithm)) {
        throw new SecurityException("허용되지 않은 알고리즘: " + algorithm);
    }

    return jws.getBody();
}
```

### Algorithm Confusion Attack

```java
// 취약한 코드 - RS256 공개키를 HS256 비밀키로 오용
// 공격자가 공개키(공개된)를 사용하여 HS256으로 서명
// 서버가 공개키를 HS256 비밀키로 사용하면 검증 통과

// 공격 시나리오:
// 1. 서버는 RS256 사용 (공개키로 검증)
// 2. 공격자가 alg를 HS256으로 변경
// 3. 공격자가 공개키(알려진)를 사용하여 HS256 서명 생성
// 4. 서버가 공개키를 HS256 비밀키로 잘못 사용하면 검증 성공!
```

```java
// 안전한 코드 - 알고리즘 고정
@Service
public class JwtService {
    private static final SignatureAlgorithm EXPECTED_ALGORITHM = SignatureAlgorithm.RS256;
    private final RSAPublicKey publicKey;

    public Claims parseToken(String token) {
        Jws<Claims> jws = Jwts.parserBuilder()
            .setSigningKey(publicKey)
            .build()
            .parseClaimsJws(token);

        // 알고리즘 확인
        if (!EXPECTED_ALGORITHM.getValue().equals(jws.getHeader().getAlgorithm())) {
            throw new SecurityException("알고리즘 불일치");
        }

        return jws.getBody();
    }
}
```

### 비밀키 브루트포스

```java
// 취약: 짧거나 예측 가능한 비밀키
String SECRET = "secret123";  // 너무 짧음
String SECRET = "your-256-bit-secret";  // 예측 가능

// 안전: 충분한 엔트로피
@Configuration
public class JwtConfig {

    @Bean
    public SecretKey jwtSecretKey() {
        // 최소 256 bits (32 bytes)
        byte[] keyBytes = new byte[32];
        new SecureRandom().nextBytes(keyBytes);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // 또는 환경 변수에서 로드
    @Bean
    public SecretKey jwtSecretKeyFromEnv(@Value("${jwt.secret}") String base64Secret) {
        byte[] keyBytes = Base64.getDecoder().decode(base64Secret);
        if (keyBytes.length < 32) {
            throw new IllegalArgumentException("JWT 비밀키는 최소 256 bits 이상이어야 합니다");
        }
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

### JKU/X5U Header Injection

```json
// 취약한 JWT 헤더
{
  "alg": "RS256",
  "typ": "JWT",
  "jku": "https://attacker.com/malicious-jwks.json"  // 공격자 서버
}
```

```java
// 안전한 코드 - JKU 화이트리스트
public class SecureJwkResolver implements SigningKeyResolver {
    private static final Set<String> ALLOWED_JKU_HOSTS = Set.of(
        "auth.example.com",
        "keys.example.com"
    );

    @Override
    public Key resolveSigningKey(JwsHeader header, Claims claims) {
        String jku = header.get("jku", String.class);

        if (jku != null) {
            try {
                URL url = new URL(jku);
                if (!ALLOWED_JKU_HOSTS.contains(url.getHost())) {
                    throw new SecurityException("신뢰할 수 없는 JKU: " + jku);
                }
            } catch (MalformedURLException e) {
                throw new SecurityException("잘못된 JKU URL");
            }
        }

        // JWK 로드 및 키 반환
        return loadKeyFromJwks(jku);
    }
}

// 더 안전한 방법: JKU 헤더 무시하고 설정된 JWKS URI만 사용
```

### 민감 정보 노출

```java
// 취약: Payload에 민감 정보 포함
{
  "sub": "12345",
  "password": "user_password",  // 절대 금지!
  "ssn": "123-45-6789",         // 주민번호 금지
  "credit_card": "4111..."      // 카드 번호 금지
}

// JWT Payload는 암호화되지 않고 Base64 인코딩만 됨
// 누구나 디코딩하여 내용 확인 가능
String payload = new String(Base64.getUrlDecoder().decode(jwtParts[1]));
```

```java
// 안전: 최소한의 정보만 포함
public String createToken(User user) {
    return Jwts.builder()
        .setSubject(user.getId().toString())  // ID만 포함
        .claim("roles", user.getRoles())      // 필요한 권한 정보
        // 민감 정보는 필요할 때 DB에서 조회
        .setIssuedAt(new Date())
        .setExpiration(Date.from(Instant.now().plus(Duration.ofMinutes(15))))
        .signWith(secretKey)
        .compact();
}
```

### 토큰 탈취 대응

```java
// JTI (JWT ID)를 사용한 토큰 블랙리스트
@Service
public class TokenBlacklistService {

    private final RedisTemplate<String, String> redisTemplate;

    // 토큰 무효화
    public void blacklistToken(String token) {
        Claims claims = parseToken(token);
        String jti = claims.getId();
        Date expiration = claims.getExpiration();

        // 토큰 만료 시간까지만 블랙리스트 유지
        long ttl = expiration.getTime() - System.currentTimeMillis();
        if (ttl > 0) {
            redisTemplate.opsForValue().set(
                "blacklist:" + jti,
                "revoked",
                ttl,
                TimeUnit.MILLISECONDS
            );
        }
    }

    // 블랙리스트 확인
    public boolean isBlacklisted(String token) {
        Claims claims = parseToken(token);
        String jti = claims.getId();
        return Boolean.TRUE.equals(redisTemplate.hasKey("blacklist:" + jti));
    }
}
```

## Refresh Token 전략

### Access Token + Refresh Token 플로우

- Client, Server
- 로그인 요청
- Access Token (15분) + Refresh Token (7일)
- API 요청 (Access Token)
- 응답
- ... (15분 후 Access Token 만료) ...
- API 요청 (만료된 Access Token)
- 401 Unauthorized
- 토큰 갱신 요청 (Refresh Token)
- 새 Access Token + 새 Refresh Token (Rotation)
- API 재요청 (새 Access Token)

### Refresh Token Rotation 구현

```java
@Service
@Transactional
public class TokenService {

    private final RefreshTokenRepository refreshTokenRepository;
    private final JwtUtil jwtUtil;

    public TokenPair createTokenPair(User user) {
        // Access Token 생성 (짧은 수명)
        String accessToken = jwtUtil.createAccessToken(
            user.getId(),
            user.getRoles(),
            Duration.ofMinutes(15)
        );

        // Refresh Token 생성 (긴 수명)
        String refreshToken = UUID.randomUUID().toString();

        // Refresh Token DB 저장
        RefreshTokenEntity entity = RefreshTokenEntity.builder()
            .token(hashToken(refreshToken))
            .userId(user.getId())
            .expiresAt(Instant.now().plus(Duration.ofDays(7)))
            .createdAt(Instant.now())
            .build();
        refreshTokenRepository.save(entity);

        return new TokenPair(accessToken, refreshToken);
    }

    public TokenPair refreshTokens(String refreshToken) {
        String hashedToken = hashToken(refreshToken);

        // Refresh Token 조회
        RefreshTokenEntity storedToken = refreshTokenRepository.findByToken(hashedToken)
            .orElseThrow(() -> new InvalidTokenException("유효하지 않은 Refresh Token"));

        // 만료 확인
        if (storedToken.getExpiresAt().isBefore(Instant.now())) {
            refreshTokenRepository.delete(storedToken);
            throw new TokenExpiredException("Refresh Token이 만료되었습니다");
        }

        // 이미 사용된 토큰인지 확인 (Rotation 탐지)
        if (storedToken.isUsed()) {
            // 토큰 재사용 감지 = 탈취 의심
            // 해당 사용자의 모든 Refresh Token 무효화
            refreshTokenRepository.deleteAllByUserId(storedToken.getUserId());
            throw new TokenReuseException("토큰 재사용이 감지되었습니다. 모든 세션이 종료됩니다.");
        }

        // 기존 토큰을 사용됨으로 표시
        storedToken.markAsUsed();
        refreshTokenRepository.save(storedToken);

        // 새 토큰 쌍 발급
        User user = userRepository.findById(storedToken.getUserId()).orElseThrow();
        return createTokenPair(user);
    }

    private String hashToken(String token) {
        return Hashing.sha256()
            .hashString(token, StandardCharsets.UTF_8)
            .toString();
    }
}

// 토큰 쌍 DTO
public record TokenPair(String accessToken, String refreshToken) {}
```

### Refresh Token 저장 위치

- 저장 위치: HttpOnly Cookie:
  - 장점: XSS 공격에 안전
  - 단점: CSRF 공격 가능
  - 권장: 웹 앱 (권장)
- 저장 위치: localStorage:
  - 장점: 접근 편리
  - 단점: XSS에 취약
  - 권장: 권장하지 않음
- 저장 위치: sessionStorage:
  - 장점: 탭 간 격리
  - 단점: XSS에 취약, 탭 닫으면 삭제
  - 권장: 권장하지 않음
- 저장 위치: Secure Storage (모바일):
  - 장점: 안전한 저장소
  - 단점: 플랫폼 의존
  - 권장: 모바일 앱 (권장)

```java
// HttpOnly Cookie로 Refresh Token 전송
@PostMapping("/auth/login")
public ResponseEntity<LoginResponse> login(
    @RequestBody LoginRequest request,
    HttpServletResponse response
) {
    TokenPair tokens = authService.authenticate(request);

    // Access Token은 응답 본문으로
    LoginResponse body = new LoginResponse(tokens.accessToken());

    // Refresh Token은 HttpOnly Cookie로
    Cookie refreshTokenCookie = new Cookie("refresh_token", tokens.refreshToken());
    refreshTokenCookie.setHttpOnly(true);
    refreshTokenCookie.setSecure(true);  // HTTPS only
    refreshTokenCookie.setPath("/api/auth/refresh");
    refreshTokenCookie.setMaxAge(7 * 24 * 60 * 60);  // 7일
    refreshTokenCookie.setAttribute("SameSite", "Strict");
    response.addCookie(refreshTokenCookie);

    return ResponseEntity.ok(body);
}

@PostMapping("/auth/refresh")
public ResponseEntity<RefreshResponse> refresh(
    @CookieValue("refresh_token") String refreshToken,
    HttpServletResponse response
) {
    TokenPair tokens = authService.refreshTokens(refreshToken);

    // 새 Refresh Token 쿠키 설정
    Cookie refreshTokenCookie = new Cookie("refresh_token", tokens.refreshToken());
    // ... 동일한 설정

    return ResponseEntity.ok(new RefreshResponse(tokens.accessToken()));
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

- [RFC 7519 - JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 8725 - JWT Best Current Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [Auth0 JWT Vulnerabilities](https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/)
- [PortSwigger JWT Attacks](https://portswigger.net/web-security/jwt)

## 관련 학습

- [OAuth 2.0](02-OAuth-2.0.md)
- [인증과 인가 설계](01-인증과-인가-설계.md)
- [세션 기반 인증](04-세션-기반-인증.md)

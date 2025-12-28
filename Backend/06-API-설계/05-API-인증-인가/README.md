# API 인증/인가

## 목차
1. [인증과 인가의 차이](#인증과-인가의-차이)
2. [API Key 방식](#api-key-방식)
3. [OAuth 2.0 플로우](#oauth-20-플로우)
4. [JWT 구조](#jwt-구조)
5. [Rate Limiting 구현](#rate-limiting-구현)
6. [실전 구현 예제](#실전-구현-예제)
---

## 인증과 인가의 차이

### 정의

```
┌─────────────────────────────────────────────────────────┐
│  인증 (Authentication) - "당신은 누구인가?"              │
├─────────────────────────────────────────────────────────┤
│  • 사용자의 신원을 확인하는 과정                         │
│  • 예: 로그인, 비밀번호 확인, 생체 인식                  │
│  • 결과: 신원 확인됨/안됨                                │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  인가 (Authorization) - "당신은 무엇을 할 수 있는가?"    │
├─────────────────────────────────────────────────────────┤
│  • 인증된 사용자의 권한을 확인하는 과정                  │
│  • 예: 관리자 페이지 접근, 파일 수정 권한                │
│  • 결과: 허용됨/거부됨                                   │
└─────────────────────────────────────────────────────────┘
```

### 예시 시나리오

```
1. 사용자가 로그인 시도 → 인증 (ID/PW 확인)
2. 로그인 성공 후 관리자 페이지 접근 → 인가 (권한 확인)

HTTP 상태 코드:
- 401 Unauthorized → 인증 실패 (로그인 필요)
- 403 Forbidden → 인가 실패 (권한 없음)
```

### 인증 방식 비교

| 방식 | 장점 | 단점 | 사용 사례 |
|------|------|------|-----------|
| API Key | 간단, 구현 용이 | 보안 취약, 권한 제어 어려움 | 서버 간 통신, 공개 API |
| Basic Auth | 간단, 표준 지원 | 매 요청마다 자격증명 전송 | 내부 서비스, 개발 환경 |
| OAuth 2.0 | 안전, 표준화, 권한 위임 | 복잡한 구현 | 소셜 로그인, 서드파티 API |
| JWT | Stateless, 확장성 | 토큰 무효화 어려움 | SPA, 마이크로서비스 |
| Session | 서버 제어 용이 | 서버 상태 유지 필요 | 전통적 웹 애플리케이션 |

---

## API Key 방식

### 개요

API Key는 클라이언트를 식별하는 고유한 문자열입니다. 가장 간단한 인증 방식이지만 제한된 보안을 제공합니다.

### API Key 전달 방법

```http
# 1. Header (권장)
GET /api/users HTTP/1.1
X-API-Key: your-api-key-here

# 2. Query Parameter (비권장 - 로그에 노출)
GET /api/users?api_key=your-api-key-here

# 3. Authorization Header
GET /api/users HTTP/1.1
Authorization: ApiKey your-api-key-here
```

### API Key 생성

```javascript
const crypto = require('crypto');

// 1. 랜덤 문자열 (추천)
function generateApiKey() {
  return crypto.randomBytes(32).toString('hex');
  // 결과: "a1b2c3d4e5f6..."  (64자)
}

// 2. UUID v4
const { v4: uuidv4 } = require('uuid');
function generateApiKeyUUID() {
  return uuidv4().replace(/-/g, '');
  // 결과: "550e8400e29b41d4a716446655440000"
}

// 3. Prefix 포함 (식별 용이)
function generateApiKeyWithPrefix(prefix = 'sk') {
  const key = crypto.randomBytes(24).toString('base64').replace(/[^a-zA-Z0-9]/g, '');
  return `${prefix}_${key}`;
  // 결과: "sk_aB3cD4eF5gH6iJ7kL8mN9o"
}
```

### API Key 저장 및 검증

```javascript
const bcrypt = require('bcrypt');

class ApiKeyService {
  constructor(db) {
    this.db = db;
  }

  // API Key 생성 및 저장
  async createApiKey(userId, name) {
    // 원본 키 생성
    const rawKey = `sk_live_${crypto.randomBytes(24).toString('base64url')}`;

    // 해시하여 저장 (원본은 저장 X)
    const hashedKey = await bcrypt.hash(rawKey, 10);

    // 키의 prefix만 저장 (사용자가 식별할 수 있도록)
    const prefix = rawKey.substring(0, 12);

    await this.db.apiKeys.create({
      userId,
      name,
      prefix,
      hashedKey,
      createdAt: new Date(),
      lastUsedAt: null,
      isActive: true
    });

    // 원본 키는 이때만 반환 (다시 조회 불가)
    return { key: rawKey, prefix };
  }

  // API Key 검증
  async validateApiKey(rawKey) {
    const prefix = rawKey.substring(0, 12);

    // prefix로 후보 조회
    const candidates = await this.db.apiKeys.find({
      prefix,
      isActive: true
    });

    for (const candidate of candidates) {
      const isValid = await bcrypt.compare(rawKey, candidate.hashedKey);
      if (isValid) {
        // 마지막 사용 시간 업데이트
        await this.db.apiKeys.updateOne(
          { _id: candidate._id },
          { lastUsedAt: new Date() }
        );
        return { valid: true, userId: candidate.userId };
      }
    }

    return { valid: false };
  }

  // API Key 폐기
  async revokeApiKey(keyId, userId) {
    const result = await this.db.apiKeys.updateOne(
      { _id: keyId, userId },
      { isActive: false, revokedAt: new Date() }
    );
    return result.modifiedCount > 0;
  }
}
```

### Express 미들웨어

```javascript
function apiKeyAuth(apiKeyService) {
  return async (req, res, next) => {
    const apiKey = req.headers['x-api-key'] || req.query.api_key;

    if (!apiKey) {
      return res.status(401).json({
        error: 'API key is required',
        code: 'MISSING_API_KEY'
      });
    }

    try {
      const result = await apiKeyService.validateApiKey(apiKey);

      if (!result.valid) {
        return res.status(401).json({
          error: 'Invalid API key',
          code: 'INVALID_API_KEY'
        });
      }

      req.userId = result.userId;
      next();
    } catch (error) {
      console.error('API key validation error:', error);
      res.status(500).json({
        error: 'Authentication failed',
        code: 'AUTH_ERROR'
      });
    }
  };
}

// 사용
app.use('/api', apiKeyAuth(apiKeyService));
```

### API Key 모범 사례

```
1. 보안
   - HTTPS 필수
   - 키 해싱 저장 (원본 저장 X)
   - 환경 변수로 관리
   - 주기적 로테이션

2. 운영
   - 키별 권한 범위 설정 (scopes)
   - 사용량 모니터링
   - 비활성 키 정리
   - 감사 로그 기록

3. 사용자 경험
   - 여러 키 생성 허용 (용도별)
   - 키 이름/설명 지원
   - 마지막 사용 시간 표시
   - 즉시 폐기 기능
```

---

## OAuth 2.0 플로우

### 개요

OAuth 2.0은 권한 위임(Authorization Delegation)을 위한 표준 프로토콜입니다. 사용자가 제3자 애플리케이션에 비밀번호를 공유하지 않고도 리소스 접근 권한을 부여할 수 있습니다.

### 역할 정의

```
┌─────────────────────────────────────────────────────────┐
│                    OAuth 2.0 역할                        │
├─────────────────────────────────────────────────────────┤
│ Resource Owner    : 사용자 (권한을 부여하는 주체)        │
│ Client            : 제3자 애플리케이션                   │
│ Authorization Server : 인증 서버 (토큰 발급)             │
│ Resource Server   : 보호된 리소스 서버 (API 서버)        │
└─────────────────────────────────────────────────────────┘
```

### 1. Authorization Code Flow (권장)

가장 안전한 플로우로, 서버 사이드 애플리케이션에 적합합니다.

```
┌─────────────────────────────────────────────────────────┐
│                Authorization Code Flow                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  User        Client          Auth Server    Resource    │
│   │            │                  │          Server     │
│   │            │                  │            │        │
│   │──(1) Login────>              │            │        │
│   │            │                  │            │        │
│   │<─(2) Redirect to Auth Server─│            │        │
│   │                               │            │        │
│   │──(3) Authenticate & Consent──>│            │        │
│   │                               │            │        │
│   │<─(4) Redirect with Auth Code──│            │        │
│   │                               │            │        │
│   │            │<─(5) Auth Code───│            │        │
│   │            │                  │            │        │
│   │            │──(6) Exchange ───>│            │        │
│   │            │   Code for Token │            │        │
│   │            │                  │            │        │
│   │            │<─(7) Access Token─│            │        │
│   │            │    + Refresh Token│            │        │
│   │            │                  │            │        │
│   │            │──(8) API Call ───────────────>│        │
│   │            │   with Access Token           │        │
│   │            │                               │        │
│   │            │<─(9) Protected Resource───────│        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

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

### 2. Authorization Code Flow with PKCE

Public Client(SPA, 모바일 앱)를 위한 확장으로, client_secret 없이 안전하게 동작합니다.

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

### 3. Client Credentials Flow

서버 간 통신(Machine-to-Machine)에 사용됩니다.

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

### 4. Refresh Token Flow

Access Token 만료 시 새 토큰을 발급받습니다.

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

### OAuth 2.0 보안 모범 사례 (RFC 6819, OAuth 2.1)

```
1. PKCE 필수 사용 (모든 클라이언트)
2. Implicit Flow 사용 금지 (OAuth 2.1에서 제거)
3. state 파라미터로 CSRF 방지
4. redirect_uri 정확히 일치하도록 검증
5. 짧은 Access Token 수명 (15분-1시간)
6. Refresh Token 로테이션
7. HTTPS 필수
8. Token 안전한 저장 (httpOnly 쿠키 권장)
```

---

## JWT 구조

### 개요

JWT(JSON Web Token)는 당사자 간에 정보를 안전하게 전송하기 위한 컴팩트하고 자가 포함적인 방식입니다.

### JWT 구조

```
┌─────────────────────────────────────────────────────────┐
│                      JWT 구조                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Header.Payload.Signature                               │
│                                                         │
│  xxxxx.yyyyy.zzzzz                                      │
│    │      │     │                                       │
│    │      │     └─ Signature (서명)                     │
│    │      └─ Payload (내용)                             │
│    └─ Header (헤더)                                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Header (헤더)

```json
{
  "alg": "HS256",    // 서명 알고리즘
  "typ": "JWT"       // 토큰 타입
}

// Base64Url 인코딩
// eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

**지원 알고리즘:**
- HMAC: HS256, HS384, HS512 (대칭키)
- RSA: RS256, RS384, RS512 (비대칭키)
- ECDSA: ES256, ES384, ES512 (비대칭키)

### Payload (페이로드)

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

### Signature (서명)

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

### JWT 생성 및 검증 (Node.js)

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

### JWT 인증 미들웨어

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

### JWT 토큰 갱신 엔드포인트

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

### JWT 무효화 전략

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

### JWT vs Session 비교

| 측면 | JWT | Session |
|------|-----|---------|
| 저장 위치 | 클라이언트 | 서버 (메모리/DB) |
| 상태 | Stateless | Stateful |
| 확장성 | 높음 (서버 간 공유 불필요) | 낮음 (세션 공유 필요) |
| 무효화 | 어려움 | 쉬움 |
| 크기 | 큼 (매 요청 전송) | 작음 (Session ID만) |
| 보안 | 토큰 탈취 위험 | CSRF 위험 |

---

## Rate Limiting 구현

### 개요

Rate Limiting은 일정 시간 내 API 호출 횟수를 제한하여 서버를 보호하고 공정한 사용을 보장합니다.

### Rate Limiting 알고리즘

#### 1. Fixed Window (고정 윈도우)

```
┌─────────────────────────────────────────────────────────┐
│              Fixed Window (1분에 100회 제한)             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Window 1 (00:00-00:59)    Window 2 (01:00-01:59)       │
│  ┌────────────────────┐    ┌────────────────────┐       │
│  │ ▮▮▮▮▮... (100회)   │    │ ▮▮▮... (새 카운트) │       │
│  └────────────────────┘    └────────────────────┘       │
│                                                         │
│  문제: 경계 시점에 버스트 가능 (00:59에 100회 + 01:00에 100회) │
└─────────────────────────────────────────────────────────┘
```

```javascript
class FixedWindowRateLimiter {
  constructor(redis, limit, windowMs) {
    this.redis = redis;
    this.limit = limit;
    this.windowMs = windowMs;
  }

  async isAllowed(key) {
    const windowKey = `ratelimit:${key}:${Math.floor(Date.now() / this.windowMs)}`;

    const current = await this.redis.incr(windowKey);

    if (current === 1) {
      await this.redis.pexpire(windowKey, this.windowMs);
    }

    return {
      allowed: current <= this.limit,
      remaining: Math.max(0, this.limit - current),
      resetAt: Math.ceil(Date.now() / this.windowMs) * this.windowMs
    };
  }
}
```

#### 2. Sliding Window Log (슬라이딩 윈도우 로그)

```javascript
class SlidingWindowLogRateLimiter {
  constructor(redis, limit, windowMs) {
    this.redis = redis;
    this.limit = limit;
    this.windowMs = windowMs;
  }

  async isAllowed(key) {
    const now = Date.now();
    const windowStart = now - this.windowMs;
    const redisKey = `ratelimit:${key}`;

    // 윈도우 밖의 요청 제거
    await this.redis.zremrangebyscore(redisKey, 0, windowStart);

    // 현재 요청 수 확인
    const count = await this.redis.zcard(redisKey);

    if (count >= this.limit) {
      return { allowed: false, remaining: 0 };
    }

    // 현재 요청 추가
    await this.redis.zadd(redisKey, now, `${now}-${Math.random()}`);
    await this.redis.pexpire(redisKey, this.windowMs);

    return {
      allowed: true,
      remaining: this.limit - count - 1
    };
  }
}
```

#### 3. Sliding Window Counter (슬라이딩 윈도우 카운터)

Fixed Window와 Sliding Window Log의 장점을 결합합니다.

```javascript
class SlidingWindowCounterRateLimiter {
  constructor(redis, limit, windowMs) {
    this.redis = redis;
    this.limit = limit;
    this.windowMs = windowMs;
  }

  async isAllowed(key) {
    const now = Date.now();
    const currentWindow = Math.floor(now / this.windowMs);
    const previousWindow = currentWindow - 1;
    const windowProgress = (now % this.windowMs) / this.windowMs;

    const currentKey = `ratelimit:${key}:${currentWindow}`;
    const previousKey = `ratelimit:${key}:${previousWindow}`;

    const [currentCount, previousCount] = await Promise.all([
      this.redis.get(currentKey),
      this.redis.get(previousKey)
    ]);

    // 가중 평균 계산
    const weightedCount =
      (parseInt(previousCount) || 0) * (1 - windowProgress) +
      (parseInt(currentCount) || 0);

    if (weightedCount >= this.limit) {
      return { allowed: false, remaining: 0 };
    }

    await this.redis.incr(currentKey);
    await this.redis.pexpire(currentKey, this.windowMs * 2);

    return {
      allowed: true,
      remaining: Math.floor(this.limit - weightedCount - 1)
    };
  }
}
```

#### 4. Token Bucket (토큰 버킷)

버스트 트래픽을 허용하면서 평균 속도를 제한합니다.

```javascript
class TokenBucketRateLimiter {
  constructor(redis, capacity, refillRate, refillInterval) {
    this.redis = redis;
    this.capacity = capacity;           // 최대 토큰 수
    this.refillRate = refillRate;       // 충전 토큰 수
    this.refillInterval = refillInterval; // 충전 간격 (ms)
  }

  async isAllowed(key, tokensRequired = 1) {
    const now = Date.now();
    const bucketKey = `bucket:${key}`;

    // Lua 스크립트로 원자적 처리
    const script = `
      local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'lastRefill')
      local tokens = tonumber(bucket[1]) or tonumber(ARGV[1])
      local lastRefill = tonumber(bucket[2]) or tonumber(ARGV[4])

      local elapsed = tonumber(ARGV[4]) - lastRefill
      local refillCount = math.floor(elapsed / tonumber(ARGV[3]))
      tokens = math.min(tonumber(ARGV[1]), tokens + refillCount * tonumber(ARGV[2]))

      if refillCount > 0 then
        lastRefill = lastRefill + refillCount * tonumber(ARGV[3])
      end

      local allowed = tokens >= tonumber(ARGV[5])
      if allowed then
        tokens = tokens - tonumber(ARGV[5])
      end

      redis.call('HMSET', KEYS[1], 'tokens', tokens, 'lastRefill', lastRefill)
      redis.call('PEXPIRE', KEYS[1], tonumber(ARGV[6]))

      return {allowed and 1 or 0, tokens}
    `;

    const result = await this.redis.eval(
      script,
      1,
      bucketKey,
      this.capacity,
      this.refillRate,
      this.refillInterval,
      now,
      tokensRequired,
      this.refillInterval * 2
    );

    return {
      allowed: result[0] === 1,
      remaining: result[1]
    };
  }
}
```

### Rate Limiting 미들웨어 구현

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redis = new Redis();

// 기본 Rate Limiter
const apiLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
  }),
  windowMs: 60 * 1000,    // 1분
  max: 100,               // 최대 100회
  message: {
    error: 'Too many requests',
    code: 'RATE_LIMIT_EXCEEDED',
    retryAfter: 60
  },
  standardHeaders: true,   // RateLimit-* 헤더 포함
  legacyHeaders: false,    // X-RateLimit-* 헤더 비포함
  keyGenerator: (req) => {
    // 인증된 사용자는 사용자 ID로, 아니면 IP로 구분
    return req.user?.id || req.ip;
  },
  skip: (req) => {
    // 특정 경로 제외
    return req.path === '/health';
  }
});

// 티어별 Rate Limiter
function createTieredRateLimiter() {
  return async (req, res, next) => {
    const user = req.user;
    let limit, windowMs;

    // 사용자 티어에 따른 제한
    switch (user?.tier) {
      case 'enterprise':
        limit = 10000;
        windowMs = 60000;
        break;
      case 'pro':
        limit = 1000;
        windowMs = 60000;
        break;
      default:
        limit = 100;
        windowMs = 60000;
    }

    const limiter = new SlidingWindowCounterRateLimiter(redis, limit, windowMs);
    const result = await limiter.isAllowed(user?.id || req.ip);

    // 헤더 설정
    res.set({
      'RateLimit-Limit': limit,
      'RateLimit-Remaining': result.remaining,
      'RateLimit-Reset': Math.ceil(Date.now() / windowMs) * windowMs
    });

    if (!result.allowed) {
      return res.status(429).json({
        error: 'Too many requests',
        code: 'RATE_LIMIT_EXCEEDED',
        retryAfter: Math.ceil(windowMs / 1000)
      });
    }

    next();
  };
}

// 사용
app.use('/api', apiLimiter);
app.use('/api/v2', createTieredRateLimiter());
```

### Rate Limit 응답 헤더

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 95
RateLimit-Reset: 1735344060

# 429 응답 시
HTTP/1.1 429 Too Many Requests
Retry-After: 60
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 1735344060
```

---

## 실전 구현 예제

### 통합 인증 시스템 (Express)

```javascript
// auth/index.js
const express = require('express');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');

const router = express.Router();

// 회원가입
router.post('/register', async (req, res) => {
  try {
    const { email, password, name } = req.body;

    // 이메일 중복 확인
    const existing = await userService.findByEmail(email);
    if (existing) {
      return res.status(409).json({ error: 'Email already exists' });
    }

    // 비밀번호 해싱
    const hashedPassword = await bcrypt.hash(password, 12);

    // 사용자 생성
    const user = await userService.create({
      email,
      password: hashedPassword,
      name,
      tokenVersion: 0
    });

    // 토큰 발급
    const accessToken = jwtService.generateAccessToken(user);
    const refreshToken = jwtService.generateRefreshToken(user);

    // Refresh Token은 httpOnly 쿠키로 설정
    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000 // 7일
    });

    res.status(201).json({
      user: { id: user.id, email: user.email, name: user.name },
      accessToken
    });
  } catch (error) {
    res.status(500).json({ error: 'Registration failed' });
  }
});

// 로그인
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;

    const user = await userService.findByEmail(email);
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    const validPassword = await bcrypt.compare(password, user.password);
    if (!validPassword) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    const accessToken = jwtService.generateAccessToken(user);
    const refreshToken = jwtService.generateRefreshToken(user);

    res.cookie('refreshToken', refreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.json({
      user: { id: user.id, email: user.email, name: user.name },
      accessToken
    });
  } catch (error) {
    res.status(500).json({ error: 'Login failed' });
  }
});

// 로그아웃
router.post('/logout', jwtAuth, async (req, res) => {
  try {
    // 모든 기기에서 로그아웃 (tokenVersion 증가)
    await userService.incrementTokenVersion(req.user.id);

    res.clearCookie('refreshToken');
    res.json({ message: 'Logged out successfully' });
  } catch (error) {
    res.status(500).json({ error: 'Logout failed' });
  }
});

// 토큰 갱신
router.post('/refresh', async (req, res) => {
  try {
    const refreshToken = req.cookies.refreshToken;
    if (!refreshToken) {
      return res.status(401).json({ error: 'Refresh token required' });
    }

    const decoded = jwt.decode(refreshToken);
    const user = await userService.findById(decoded.sub);

    if (!user) {
      return res.status(401).json({ error: 'User not found' });
    }

    await jwtService.verifyRefreshToken(refreshToken, user);

    const newAccessToken = jwtService.generateAccessToken(user);
    const newRefreshToken = jwtService.generateRefreshToken(user);

    res.cookie('refreshToken', newRefreshToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'strict',
      maxAge: 7 * 24 * 60 * 60 * 1000
    });

    res.json({ accessToken: newAccessToken });
  } catch (error) {
    res.clearCookie('refreshToken');
    res.status(401).json({ error: 'Invalid refresh token' });
  }
});

module.exports = router;
```

### RBAC (Role-Based Access Control)

```javascript
// middleware/rbac.js
function checkPermission(requiredPermissions) {
  return (req, res, next) => {
    const userRole = req.user.role;
    const userPermissions = rolePermissions[userRole] || [];

    const hasPermission = requiredPermissions.every(
      perm => userPermissions.includes(perm) || userPermissions.includes('*')
    );

    if (!hasPermission) {
      return res.status(403).json({
        error: 'Insufficient permissions',
        code: 'FORBIDDEN',
        required: requiredPermissions
      });
    }

    next();
  };
}

// 역할별 권한 정의
const rolePermissions = {
  admin: ['*'],
  moderator: ['users:read', 'posts:read', 'posts:write', 'posts:delete', 'comments:delete'],
  user: ['posts:read', 'posts:write', 'comments:read', 'comments:write'],
  guest: ['posts:read', 'comments:read']
};

// 사용 예시
router.get('/users', jwtAuth, checkPermission(['users:read']), getUsers);
router.delete('/posts/:id', jwtAuth, checkPermission(['posts:delete']), deletePost);
```

---

## 참고 자료

- [RFC 6749 - OAuth 2.0](https://tools.ietf.org/html/rfc6749)
- [RFC 7519 - JWT](https://tools.ietf.org/html/rfc7519)
- [RFC 6819 - OAuth 2.0 Threat Model](https://tools.ietf.org/html/rfc6819)
- [OAuth 2.1 Draft](https://oauth.net/2.1/)
- [JWT.io](https://jwt.io/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Rate Limiting Algorithms](https://konghq.com/blog/how-to-design-a-scalable-rate-limiting-algorithm)

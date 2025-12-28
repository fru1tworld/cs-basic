# 로드 밸런싱

## 목차
1. [로드 밸런싱 개요](#로드-밸런싱-개요)
2. [로드 밸런싱 알고리즘](#로드-밸런싱-알고리즘)
3. [L4 vs L7 로드 밸런싱](#l4-vs-l7-로드-밸런싱)
4. [Health Check](#health-check)
5. [Session Persistence (Sticky Session)](#session-persistence-sticky-session)
6. [HAProxy](#haproxy)
7. [클라우드 로드 밸런서](#클라우드-로드-밸런서)
---

## 로드 밸런싱 개요

**로드 밸런싱(Load Balancing)**은 들어오는 네트워크 트래픽을 여러 서버에 분산하여 단일 서버의 과부하를 방지하고 가용성과 응답성을 높이는 기술입니다.

### 로드 밸런싱이 필요한 이유

```
[로드 밸런서 없이]
                    ┌──────────────┐
                    │              │
 100,000 요청 ─────▶│  단일 서버   │  ← 과부하, 장애 위험
                    │              │
                    └──────────────┘

[로드 밸런서 사용]
                    ┌──────────────┐
                ┌──▶│   서버 1     │  33,333 요청
                │   └──────────────┘
 100,000 요청 ──┼──▶┌──────────────┐
   ┌────────┐   │   │   서버 2     │  33,333 요청
   │  Load  │───┤   └──────────────┘
   │Balancer│   │   ┌──────────────┐
   └────────┘   └──▶│   서버 3     │  33,333 요청
                    └──────────────┘
```

### 로드 밸런싱의 장점

| 장점 | 설명 |
|------|------|
| **고가용성 (HA)** | 서버 장애 시 다른 서버로 자동 전환 |
| **확장성** | 서버 추가로 수평 확장 가능 |
| **유연성** | 무중단 배포, 점진적 롤아웃 |
| **성능** | 트래픽 분산으로 응답 시간 개선 |
| **비용 효율** | 저렴한 서버 여러 대 vs 고가 서버 한 대 |

### 로드 밸런서 유형

```
┌─────────────────────────────────────────────────────────────┐
│                    로드 밸런서 분류                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [배포 위치별]                                                │
│  ├─ 하드웨어 LB: F5 BIG-IP, Citrix ADC                       │
│  ├─ 소프트웨어 LB: HAProxy, Nginx, Envoy                     │
│  └─ 클라우드 LB: AWS ALB/NLB, GCP Cloud LB, Azure LB         │
│                                                              │
│  [OSI 계층별]                                                 │
│  ├─ L4 (Transport): IP + Port 기반 라우팅                    │
│  └─ L7 (Application): HTTP 헤더, URL, 쿠키 기반 라우팅        │
│                                                              │
│  [구성 방식별]                                                │
│  ├─ Active-Passive: 하나가 주, 하나가 대기                    │
│  └─ Active-Active: 모두 트래픽 처리                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 로드 밸런싱 알고리즘

### Round Robin (라운드 로빈)

가장 기본적인 알고리즘으로, 요청을 **순차적으로 각 서버에 분배**합니다.

```
요청 1 → 서버 A
요청 2 → 서버 B
요청 3 → 서버 C
요청 4 → 서버 A  (다시 처음으로)
요청 5 → 서버 B
...
```

#### Nginx 설정

```nginx
upstream backend {
    # 기본값이 Round Robin
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

#### HAProxy 설정

```haproxy
backend web_servers
    balance roundrobin
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
    server web3 192.168.1.12:8080 check
```

#### 장단점

| 장점 | 단점 |
|------|------|
| 간단하고 구현이 쉬움 | 서버 성능 차이 고려 안 함 |
| 오버헤드 없음 | 요청 처리 시간이 다른 경우 불균형 |
| 예측 가능한 분배 | 세션 유지 안 됨 |

### Weighted Round Robin (가중 라운드 로빈)

서버에 **가중치(Weight)**를 부여하여 성능에 따라 트래픽을 다르게 분배합니다.

```
서버 A (weight=5): 5번 중 5번 선택
서버 B (weight=3): 10번 중 3번 선택
서버 C (weight=2): 10번 중 2번 선택

실제 분배:
요청 1-5 → 서버 A
요청 6-8 → 서버 B
요청 9-10 → 서버 C
```

#### Nginx 설정

```nginx
upstream backend {
    server 192.168.1.10:8080 weight=5;  # 50%
    server 192.168.1.11:8080 weight=3;  # 30%
    server 192.168.1.12:8080 weight=2;  # 20%
}
```

#### HAProxy 설정

```haproxy
backend web_servers
    balance roundrobin
    server web1 192.168.1.10:8080 weight 5 check
    server web2 192.168.1.11:8080 weight 3 check
    server web3 192.168.1.12:8080 weight 2 check
```

#### 사용 사례
- 서버 사양이 다른 경우 (CPU, 메모리)
- 새 서버 점진적 도입 (카나리 배포)
- 특정 서버에 더 많은 트래픽 유도

### Least Connections (최소 연결)

**현재 연결 수가 가장 적은 서버**에 요청을 분배합니다.

```
서버 A: 현재 연결 10개 ←
서버 B: 현재 연결 15개
서버 C: 현재 연결 20개

새 요청 → 서버 A (연결 가장 적음)
```

#### Nginx 설정

```nginx
upstream backend {
    least_conn;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

#### HAProxy 설정

```haproxy
backend web_servers
    balance leastconn
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
    server web3 192.168.1.12:8080 check
```

#### 장단점

| 장점 | 단점 |
|------|------|
| 요청 처리 시간이 다를 때 효과적 | 연결 수 추적 오버헤드 |
| 동적으로 부하 분산 | 짧은 요청에는 비효율적 |
| 긴 연결(WebSocket)에 적합 | 상태 관리 필요 |

#### 적합한 사용 사례
- 요청 처리 시간이 불균일한 경우
- 장시간 연결 (WebSocket, 파일 다운로드)
- DB 연결 풀

### Weighted Least Connections (가중 최소 연결)

Least Connections에 **가중치**를 적용합니다.

```nginx
upstream backend {
    least_conn;
    server 192.168.1.10:8080 weight=5;
    server 192.168.1.11:8080 weight=3;
    server 192.168.1.12:8080 weight=2;
}
```

### IP Hash

클라이언트 **IP 주소를 해시**하여 항상 같은 서버로 라우팅합니다.

```
Client IP 203.0.113.50 → hash → 서버 A
Client IP 203.0.113.51 → hash → 서버 B
Client IP 203.0.113.52 → hash → 서버 C

# 같은 IP는 항상 같은 서버
Client IP 203.0.113.50 → 서버 A (항상)
```

#### Nginx 설정

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}
```

#### HAProxy 설정

```haproxy
backend web_servers
    balance source
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
    server web3 192.168.1.12:8080 check
```

#### 장단점

| 장점 | 단점 |
|------|------|
| 세션 유지 (Sticky) | 부하 불균형 가능 |
| 추가 설정 불필요 | NAT 뒤의 사용자는 같은 서버로 |
| 간단한 세션 친화성 | 서버 추가/제거 시 재분배 |

### Generic Hash (일반 해시)

**지정된 키**를 해시하여 서버를 선택합니다. URI, 쿠키, 헤더 등 다양한 값을 키로 사용할 수 있습니다.

```nginx
upstream backend {
    hash $request_uri consistent;  # URI 기반 해시
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    server 192.168.1.12:8080;
}

# 또는 쿠키 기반
upstream backend {
    hash $cookie_sessionid consistent;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

#### Consistent Hash (일관된 해시)

`consistent` 키워드를 사용하면 서버 추가/제거 시 **최소한의 요청만 재분배**됩니다.

```
[일반 해시]
서버 추가 시: 모든 요청 재분배

[Consistent Hash]
서버 추가 시: 1/N 요청만 재분배 (N = 서버 수)
```

### 알고리즘 비교 표

| 알고리즘 | 세션 유지 | 부하 분산 | 복잡도 | 사용 사례 |
|---------|----------|----------|--------|----------|
| Round Robin | X | 균등 | 낮음 | 상태 없는 서비스 |
| Weighted RR | X | 가중 균등 | 낮음 | 서버 성능이 다를 때 |
| Least Conn | X | 동적 | 중간 | 처리 시간 불균일 |
| IP Hash | O | 해시 | 낮음 | 간단한 세션 유지 |
| Generic Hash | O | 해시 | 중간 | 캐시 최적화, URL 기반 |

---

## L4 vs L7 로드 밸런싱

### OSI 모델 기준

```
┌─────────────────────────────────────────────────────────────┐
│                      OSI 7 Layer                            │
├─────────────────────────────────────────────────────────────┤
│  Layer 7: Application   ← L7 로드 밸런싱 (HTTP, HTTPS)       │
│  Layer 6: Presentation                                      │
│  Layer 5: Session                                           │
│  Layer 4: Transport     ← L4 로드 밸런싱 (TCP, UDP)          │
│  Layer 3: Network                                           │
│  Layer 2: Data Link                                         │
│  Layer 1: Physical                                          │
└─────────────────────────────────────────────────────────────┘
```

### L4 (Transport Layer) 로드 밸런싱

**IP 주소와 포트** 기반으로 트래픽을 분산합니다. 패킷 내용을 검사하지 않습니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    L4 로드 밸런싱                            │
│                                                              │
│   Client ──▶ [L4 LB] ──▶ Backend                            │
│              │                                               │
│              ├─ IP: 203.0.113.50                            │
│              ├─ Port: 443                                   │
│              └─ Protocol: TCP                               │
│                                                              │
│   "패킷 내용은 모름, IP+Port만 보고 분배"                      │
└─────────────────────────────────────────────────────────────┘
```

#### 특징
- **빠른 처리**: 패킷 헤더만 검사
- **낮은 레이턴시**: 마이크로초 단위
- **프로토콜 무관**: HTTP, HTTPS, DB, 게임 등 모든 TCP/UDP
- **SSL Passthrough**: TLS 복호화 없이 전달

#### Nginx L4 설정 (stream 모듈)

```nginx
stream {
    upstream backend_tcp {
        server 192.168.1.10:3306;
        server 192.168.1.11:3306;
    }

    server {
        listen 3306;
        proxy_pass backend_tcp;
        proxy_connect_timeout 1s;
    }
}
```

#### HAProxy L4 설정

```haproxy
frontend mysql_front
    bind *:3306
    mode tcp
    default_backend mysql_servers

backend mysql_servers
    mode tcp
    balance roundrobin
    server db1 192.168.1.10:3306 check
    server db2 192.168.1.11:3306 check
```

### L7 (Application Layer) 로드 밸런싱

**HTTP 헤더, URL, 쿠키, 콘텐츠** 등 애플리케이션 레벨 정보 기반으로 라우팅합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    L7 로드 밸런싱                            │
│                                                              │
│   Client ──▶ [L7 LB] ──▶ Backend                            │
│              │                                               │
│              ├─ URL: /api/users                             │
│              ├─ Host: api.example.com                       │
│              ├─ Cookie: session=abc123                      │
│              ├─ Header: Authorization: Bearer xxx           │
│              └─ Method: POST                                │
│                                                              │
│   "HTTP 요청을 완전히 파싱하고 라우팅 결정"                    │
└─────────────────────────────────────────────────────────────┘
```

#### 특징
- **콘텐츠 기반 라우팅**: URL, 헤더, 쿠키 기반
- **SSL Termination**: TLS 복호화 후 라우팅
- **캐싱**: 자주 요청되는 콘텐츠 캐싱
- **WAF 통합**: 보안 검사 가능
- **요청 수정**: 헤더 추가/제거, URL 재작성

#### Nginx L7 설정

```nginx
upstream api_users {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

upstream api_orders {
    server 192.168.1.20:8080;
    server 192.168.1.21:8080;
}

upstream static_files {
    server 192.168.1.30:80;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # URL 기반 라우팅
    location /api/users {
        proxy_pass http://api_users;
    }

    location /api/orders {
        proxy_pass http://api_orders;
    }

    location /static {
        proxy_pass http://static_files;
    }

    # 헤더 기반 라우팅
    location /api/v2 {
        if ($http_x_api_version = "2") {
            proxy_pass http://api_v2;
        }
        proxy_pass http://api_v1;
    }
}
```

#### HAProxy L7 설정

```haproxy
frontend http_front
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem
    mode http

    # ACL 정의
    acl is_api path_beg /api
    acl is_static path_beg /static
    acl is_users path_beg /api/users
    acl is_admin hdr(X-Admin-Key) -m found

    # 라우팅 규칙
    use_backend api_users if is_users
    use_backend admin_servers if is_admin
    use_backend static_servers if is_static
    default_backend web_servers

backend api_users
    mode http
    balance leastconn
    server api1 192.168.1.10:8080 check
    server api2 192.168.1.11:8080 check

backend static_servers
    mode http
    balance roundrobin
    server static1 192.168.1.30:80 check

backend web_servers
    mode http
    balance roundrobin
    server web1 192.168.1.40:8080 check
```

### L4 vs L7 비교

| 특성 | L4 | L7 |
|------|----|----|
| 라우팅 기준 | IP + Port | URL, 헤더, 쿠키 등 |
| 성능 | 매우 빠름 | 상대적으로 느림 |
| 기능성 | 단순 | 풍부함 |
| SSL 처리 | Passthrough | Termination |
| 헬스체크 | TCP 연결 | HTTP 상태 코드 |
| 캐싱 | 불가 | 가능 |
| 보안 검사 | 불가 | WAF 통합 가능 |
| 사용 사례 | DB, 게임, 범용 | 웹 애플리케이션 |

### 하이브리드 아키텍처

```
                    ┌──────────────────────────────────────┐
                    │            인터넷                     │
                    └──────────────┬───────────────────────┘
                                   │
                           ┌───────┴───────┐
                           │   L4 LB (NLB) │  ← TCP 로드밸런싱
                           │   (HA Pair)   │    고가용성
                           └───────┬───────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
       ┌──────┴──────┐      ┌──────┴──────┐      ┌──────┴──────┐
       │  L7 LB 1    │      │  L7 LB 2    │      │  L7 LB 3    │
       │  (Nginx)    │      │  (Nginx)    │      │  (Nginx)    │
       └──────┬──────┘      └──────┬──────┘      └──────┬──────┘
              │                    │                    │
     ┌────────┼────────┐  ┌────────┼────────┐  ┌────────┼────────┐
     ▼        ▼        ▼  ▼        ▼        ▼  ▼        ▼        ▼
  [App 1] [App 2] [App 3] ...
```

---

## Health Check

**헬스체크**는 백엔드 서버의 상태를 모니터링하여 장애 서버로의 트래픽을 차단합니다.

### 헬스체크 유형

```
┌─────────────────────────────────────────────────────────────┐
│                    헬스체크 유형                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Active Health Check]                                       │
│  로드밸런서가 주기적으로 서버에 요청을 보내 상태 확인            │
│                                                              │
│    LB ──ping──▶ Server                                      │
│    LB ◀──pong── Server  (정상)                              │
│    LB ──ping──▶ Server                                      │
│    LB ◀──X──── Server   (장애 → 제외)                       │
│                                                              │
│  [Passive Health Check]                                      │
│  실제 트래픽 처리 중 발생하는 에러를 모니터링                    │
│                                                              │
│    Client ──req──▶ LB ──▶ Server                            │
│    Client ◀──500── LB ◀── Server (에러 누적)                │
│    ... (N회 에러 시 서버 제외)                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Nginx 헬스체크

#### Passive Health Check (Nginx OSS)

```nginx
upstream backend {
    server 192.168.1.10:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:8080 max_fails=3 fail_timeout=30s;
    server 192.168.1.12:8080 backup;  # 백업 서버
}

# max_fails=3: 3번 실패 시 서버 제외
# fail_timeout=30s: 30초 동안 제외, 30초 후 재시도
```

#### Active Health Check (Nginx Plus 전용)

```nginx
upstream backend {
    zone backend_zone 64k;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s fails=3 passes=2;
        # interval: 5초마다 체크
        # fails: 3번 실패 시 제외
        # passes: 2번 성공 시 복귀
    }
}
```

### HAProxy 헬스체크

#### TCP 헬스체크

```haproxy
backend web_servers
    mode tcp
    balance roundrobin

    # 기본 TCP 연결 체크
    option tcp-check

    server web1 192.168.1.10:8080 check inter 5s fall 3 rise 2
    server web2 192.168.1.11:8080 check inter 5s fall 3 rise 2

# check: 헬스체크 활성화
# inter 5s: 5초 간격
# fall 3: 3번 실패 시 제외
# rise 2: 2번 성공 시 복귀
```

#### HTTP 헬스체크

```haproxy
backend web_servers
    mode http
    balance roundrobin

    # HTTP OPTIONS 요청으로 헬스체크
    option httpchk OPTIONS /health

    # 또는 GET 요청
    option httpchk GET /health
    http-check expect status 200

    server web1 192.168.1.10:8080 check inter 5s fall 3 rise 2
    server web2 192.168.1.11:8080 check inter 5s fall 3 rise 2
```

#### 고급 HTTP 헬스체크

```haproxy
backend api_servers
    mode http
    balance roundrobin

    option httpchk
    http-check connect
    http-check send meth GET uri /health hdr Host example.com
    http-check expect status 200
    http-check expect string "status":"healthy"

    server api1 192.168.1.10:8080 check inter 3s fall 2 rise 3
    server api2 192.168.1.11:8080 check inter 3s fall 2 rise 3
```

### 헬스체크 엔드포인트 구현

#### Node.js (Express)

```javascript
const express = require('express');
const app = express();

// 간단한 헬스체크
app.get('/health', (req, res) => {
    res.status(200).json({ status: 'healthy' });
});

// 상세 헬스체크 (DB, 외부 서비스 연결 확인)
app.get('/health/detailed', async (req, res) => {
    const health = {
        status: 'healthy',
        timestamp: new Date().toISOString(),
        checks: {}
    };

    // DB 연결 확인
    try {
        await db.ping();
        health.checks.database = { status: 'up' };
    } catch (error) {
        health.checks.database = { status: 'down', error: error.message };
        health.status = 'unhealthy';
    }

    // Redis 연결 확인
    try {
        await redis.ping();
        health.checks.redis = { status: 'up' };
    } catch (error) {
        health.checks.redis = { status: 'down', error: error.message };
        health.status = 'unhealthy';
    }

    const statusCode = health.status === 'healthy' ? 200 : 503;
    res.status(statusCode).json(health);
});
```

#### Spring Boot

```java
@RestController
@RequestMapping("/health")
public class HealthController {

    @Autowired
    private DataSource dataSource;

    @GetMapping
    public ResponseEntity<Map<String, Object>> health() {
        Map<String, Object> response = new HashMap<>();
        response.put("status", "UP");
        response.put("timestamp", Instant.now().toString());
        return ResponseEntity.ok(response);
    }

    @GetMapping("/detailed")
    public ResponseEntity<Map<String, Object>> detailedHealth() {
        Map<String, Object> response = new HashMap<>();
        boolean isHealthy = true;

        // DB 체크
        try (Connection conn = dataSource.getConnection()) {
            response.put("database", "UP");
        } catch (SQLException e) {
            response.put("database", "DOWN: " + e.getMessage());
            isHealthy = false;
        }

        response.put("status", isHealthy ? "UP" : "DOWN");
        return isHealthy
            ? ResponseEntity.ok(response)
            : ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).body(response);
    }
}
```

### 헬스체크 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────┐
│                 헬스체크 베스트 프랙티스                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Liveness vs Readiness 구분                              │
│     /health/live: 프로세스 살아있음                          │
│     /health/ready: 트래픽 받을 준비됨                        │
│                                                              │
│  2. 빠른 응답                                                │
│     헬스체크 엔드포인트는 100ms 이내 응답                     │
│     무거운 DB 쿼리 지양                                      │
│                                                              │
│  3. 캐스케이드 실패 방지                                     │
│     외부 의존성 장애 시에도 적절히 처리                       │
│     (예: DB 다운 시에도 /health/live는 200)                  │
│                                                              │
│  4. 적절한 간격 설정                                         │
│     너무 짧으면 오버헤드, 너무 길면 장애 감지 지연             │
│     일반적으로 5-10초 권장                                   │
│                                                              │
│  5. Graceful Degradation                                    │
│     일부 기능 장애 시에도 부분적 서비스 제공                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Session Persistence (Sticky Session)

**세션 지속성(Sticky Session)**은 같은 사용자의 요청을 항상 같은 서버로 라우팅하는 기술입니다.

### 필요한 이유

```
[Sticky Session 없이]

요청 1 (로그인) → 서버 A → 세션 저장
요청 2 (장바구니) → 서버 B → 세션 없음! (로그인 풀림)

[Sticky Session 사용]

요청 1 (로그인) → 서버 A → 세션 저장
요청 2 (장바구니) → 서버 A → 세션 있음 (정상)
요청 3 (결제) → 서버 A → 세션 있음 (정상)
```

### 구현 방법

#### 1. IP Hash

```nginx
upstream backend {
    ip_hash;
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
}
```

**단점:**
- NAT 뒤의 사용자들은 같은 서버로
- 모바일 사용자는 IP가 자주 변경됨

#### 2. Cookie 기반 (권장)

서버 식별자를 쿠키에 저장합니다.

##### Nginx (Nginx Plus)

```nginx
upstream backend {
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

##### HAProxy

```haproxy
backend web_servers
    mode http
    balance roundrobin

    # 쿠키 삽입
    cookie SERVERID insert indirect nocache

    server web1 192.168.1.10:8080 cookie s1 check
    server web2 192.168.1.11:8080 cookie s2 check
```

#### 3. URL 기반

```haproxy
backend web_servers
    mode http
    balance url_param sessionid
    server web1 192.168.1.10:8080 check
    server web2 192.168.1.11:8080 check
```

### Sticky Session의 문제점

```
┌─────────────────────────────────────────────────────────────┐
│              Sticky Session의 문제점                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 부하 불균형                                              │
│     특정 서버에 "무거운" 사용자가 몰릴 수 있음                  │
│                                                              │
│  2. 서버 장애 시 세션 손실                                    │
│     서버 다운 → 해당 서버의 모든 사용자 세션 손실               │
│                                                              │
│  3. 스케일 인/아웃 어려움                                     │
│     새 서버 추가해도 기존 사용자는 이동 안 함                   │
│     서버 제거 시 세션 마이그레이션 필요                        │
│                                                              │
│  4. 배포 복잡성                                               │
│     롤링 업데이트 시 세션 처리 고려 필요                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 대안: 외부 세션 스토어

**Sticky Session 대신 공유 세션 스토어**를 사용하는 것이 현대적인 접근법입니다.

```
┌─────────────────────────────────────────────────────────────┐
│                  외부 세션 스토어 구조                        │
│                                                              │
│            ┌────────────────┐                               │
│     ┌─────▶│  Redis/Memcached│◀─────┐                       │
│     │      │  (Session Store)│      │                       │
│     │      └────────────────┘      │                       │
│     │                              │                        │
│  ┌──┴───┐                      ┌──┴───┐                    │
│  │서버 A│                      │서버 B│                    │
│  └──────┘                      └──────┘                    │
│     ▲                              ▲                        │
│     │                              │                        │
│     └──────────┬───────────────────┘                        │
│                │                                            │
│         ┌──────┴──────┐                                    │
│         │ Load Balancer│                                    │
│         └─────────────┘                                    │
│                                                              │
│  장점:                                                       │
│  - 어떤 서버로 가도 세션 접근 가능                            │
│  - 서버 장애 시 세션 유지                                     │
│  - 스케일링 용이                                             │
│  - Sticky Session 불필요                                     │
└─────────────────────────────────────────────────────────────┘
```

#### Redis 세션 스토어 예시 (Node.js)

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const redisClient = createClient({
    url: 'redis://192.168.1.100:6379'
});
redisClient.connect();

app.use(session({
    store: new RedisStore({ client: redisClient }),
    secret: 'your-secret-key',
    resave: false,
    saveUninitialized: false,
    cookie: { secure: true, maxAge: 86400000 }
}));
```

---

## HAProxy

**HAProxy (High Availability Proxy)**는 고성능 TCP/HTTP 로드 밸런서이자 프록시 서버입니다.

### 기본 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    HAProxy 구조                              │
│                                                              │
│   ┌──────────────┐                                          │
│   │   global     │ ← 전역 설정                               │
│   └──────────────┘                                          │
│                                                              │
│   ┌──────────────┐                                          │
│   │   defaults   │ ← 기본값 설정                             │
│   └──────────────┘                                          │
│                                                              │
│   ┌──────────────┐     ┌──────────────┐                     │
│   │   frontend   │────▶│   backend    │                     │
│   │ (리스너)     │     │ (서버 풀)    │                     │
│   └──────────────┘     └──────────────┘                     │
│                                                              │
│   ┌──────────────┐                                          │
│   │    listen    │ ← frontend + backend 결합                 │
│   └──────────────┘                                          │
└─────────────────────────────────────────────────────────────┘
```

### 기본 설정 예제

```haproxy
# /etc/haproxy/haproxy.cfg

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # SSL 설정
    tune.ssl.default-dh-param 2048
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2

#---------------------------------------------------------------------
# Default settings
#---------------------------------------------------------------------
defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

#---------------------------------------------------------------------
# Frontend - HTTP (Redirect to HTTPS)
#---------------------------------------------------------------------
frontend http_front
    bind *:80
    mode http
    redirect scheme https code 301 if !{ ssl_fc }

#---------------------------------------------------------------------
# Frontend - HTTPS
#---------------------------------------------------------------------
frontend https_front
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem
    mode http

    # HTTP/2 활성화
    # bind *:443 ssl crt /etc/ssl/certs/example.com.pem alpn h2,http/1.1

    # 보안 헤더
    http-response set-header Strict-Transport-Security "max-age=63072000"

    # ACL 정의
    acl is_api path_beg /api
    acl is_static path_beg /static

    # 라우팅
    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend web_servers

#---------------------------------------------------------------------
# Backend - Web Servers
#---------------------------------------------------------------------
backend web_servers
    mode http
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    # 헤더 설정
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }

    server web1 192.168.1.10:8080 check inter 5s fall 3 rise 2
    server web2 192.168.1.11:8080 check inter 5s fall 3 rise 2
    server web3 192.168.1.12:8080 check inter 5s fall 3 rise 2 backup

#---------------------------------------------------------------------
# Backend - API Servers
#---------------------------------------------------------------------
backend api_servers
    mode http
    balance leastconn

    option httpchk GET /api/health
    http-check expect status 200

    # 커넥션 재사용
    option http-keep-alive

    server api1 192.168.1.20:8080 check inter 3s fall 2 rise 3
    server api2 192.168.1.21:8080 check inter 3s fall 2 rise 3

#---------------------------------------------------------------------
# Backend - Static Files
#---------------------------------------------------------------------
backend static_servers
    mode http
    balance roundrobin

    # 캐시 헤더
    http-response set-header Cache-Control "public, max-age=86400"

    server static1 192.168.1.30:80 check

#---------------------------------------------------------------------
# Stats Dashboard
#---------------------------------------------------------------------
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
    stats auth admin:password
```

### 주요 기능

#### Rate Limiting

```haproxy
frontend https_front
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem

    # Stick table로 요청 추적
    stick-table type ip size 100k expire 30s store http_req_rate(10s)

    # 요청 카운트
    http-request track-sc0 src

    # Rate limit (10초에 100회 초과 시 거부)
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }
```

#### 연결 제한

```haproxy
frontend https_front
    # IP당 최대 연결 수
    stick-table type ip size 100k expire 30s store conn_cur
    http-request track-sc1 src
    http-request deny if { sc_conn_cur(1) gt 10 }
```

#### 헤더 조작

```haproxy
backend web_servers
    mode http

    # 요청 헤더 추가/수정
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-header X-Real-IP %[src]
    http-request del-header X-Powered-By

    # 응답 헤더 추가/수정
    http-response set-header X-Frame-Options DENY
    http-response del-header Server
```

#### ACL과 조건부 라우팅

```haproxy
frontend https_front
    bind *:443 ssl crt /etc/ssl/certs/example.com.pem

    # ACL 정의
    acl is_api path_beg /api
    acl is_websocket hdr(Upgrade) -i WebSocket
    acl is_admin src 192.168.1.0/24
    acl is_post method POST
    acl has_auth_header hdr(Authorization) -m found
    acl is_json hdr(Content-Type) -i application/json

    # 조건부 라우팅
    use_backend websocket_servers if is_websocket
    use_backend api_servers if is_api is_json
    use_backend admin_servers if is_admin
    http-request deny if is_post !has_auth_header

    default_backend web_servers
```

### HAProxy vs Nginx 비교

| 특성 | HAProxy | Nginx |
|------|---------|-------|
| 주 용도 | 로드 밸런서 | 웹 서버 + 로드 밸런서 |
| L4 지원 | 뛰어남 | 지원 (stream 모듈) |
| L7 지원 | 뛰어남 | 뛰어남 |
| 설정 복잡도 | 중간 | 중간 |
| 정적 파일 서빙 | 제한적 | 뛰어남 |
| 캐싱 | 제한적 | 뛰어남 |
| 성능 | 매우 높음 | 높음 |
| 헬스체크 | 풍부한 옵션 | 기본적 (Plus에서 확장) |
| 통계/모니터링 | 내장 대시보드 | 별도 설정 필요 |
| 실시간 설정 변경 | Runtime API | reload 필요 |

---

## 클라우드 로드 밸런서

### AWS 로드 밸런서

```
┌─────────────────────────────────────────────────────────────┐
│                   AWS 로드 밸런서 종류                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Application Load Balancer (ALB)]                          │
│  - L7 로드 밸런서                                            │
│  - HTTP/HTTPS, WebSocket 지원                               │
│  - 경로 기반, 호스트 기반 라우팅                              │
│  - Lambda, ECS와 통합                                        │
│                                                              │
│  [Network Load Balancer (NLB)]                              │
│  - L4 로드 밸런서                                            │
│  - TCP, UDP, TLS 지원                                       │
│  - 초저지연 (마이크로초)                                     │
│  - 고정 IP 지원                                              │
│                                                              │
│  [Gateway Load Balancer (GWLB)]                             │
│  - L3 + L4 로드 밸런서                                       │
│  - 방화벽, IDS/IPS 등 어플라이언스 앞단                       │
│  - GENEVE 프로토콜                                           │
│                                                              │
│  [Classic Load Balancer (CLB)] - 레거시                      │
│  - L4 + L7                                                   │
│  - EC2-Classic 지원 (deprecated)                             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### GCP 로드 밸런서

| 종류 | 계층 | 범위 | 사용 사례 |
|------|-----|------|----------|
| HTTP(S) LB | L7 | 글로벌 | 웹 애플리케이션 |
| TCP Proxy | L4 | 글로벌 | SSL Termination |
| SSL Proxy | L4 | 글로벌 | SSL 오프로드 |
| Network LB | L4 | 리전 | TCP/UDP |
| Internal LB | L4 | 리전 | 내부 서비스 |

---

## 참고 자료

- [HAProxy Official Documentation](https://docs.haproxy.org/)
- [HAProxy Configuration Tutorials](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/)
- [Nginx Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/)
- [DigitalOcean - HAProxy Introduction](https://www.digitalocean.com/community/tutorials/an-introduction-to-haproxy-and-load-balancing-concepts)
- [AWS Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
- [L4 vs L7 Load Balancing - A10 Networks](https://www.a10networks.com/glossary/how-do-layer-4-and-layer-7-load-balancing-differ/)

# 수평 확장 vs 수직 확장 (Horizontal vs Vertical Scaling)

## 목차
1. [개요](#개요)
2. [Scale Up vs Scale Out](#scale-up-vs-scale-out)
3. [Stateless 설계](#stateless-설계)
4. [세션 관리 전략](#세션-관리-전략)
5. [데이터베이스 확장](#데이터베이스-확장)
6. [실전 적용 가이드](#실전-적용-가이드)
---

## 개요

시스템 확장성(Scalability)은 증가하는 트래픽과 데이터를 효과적으로 처리할 수 있는 능력을 의미합니다. Michael Nygard의 "Release It!"에서 강조하듯이, 확장성은 시스템 안정성의 핵심 요소입니다.

> "확장 가능한 시스템은 트래픽 증가에 선형적으로 대응할 수 있어야 한다." - Release It!

---

## Scale Up vs Scale Out

### Scale Up (수직 확장)

```
┌─────────────────────────────────────────┐
│         Scale Up (Vertical)             │
├─────────────────────────────────────────┤
│                                         │
│    ┌─────────┐      ┌─────────────┐    │
│    │ Server  │  →   │   Server    │    │
│    │  4 CPU  │      │   16 CPU    │    │
│    │  8 GB   │      │   64 GB     │    │
│    │  SSD    │      │   NVMe SSD  │    │
│    └─────────┘      └─────────────┘    │
│                                         │
│    기존 서버         업그레이드된 서버   │
└─────────────────────────────────────────┘
```

**정의**: 단일 서버의 하드웨어 성능을 향상시키는 방법

**장점**:
- 구현이 단순함 (코드 수정 불필요)
- 데이터 일관성 유지 용이
- 네트워크 오버헤드 없음
- 모놀리식 애플리케이션에 적합

**단점**:
- 하드웨어 한계 존재 (물리적 제약)
- 비용 효율성 감소 (고성능 하드웨어일수록 가격 급등)
- 단일 장애점(SPOF) 문제
- 다운타임 발생 가능

### Scale Out (수평 확장)

```
┌─────────────────────────────────────────────────────┐
│              Scale Out (Horizontal)                  │
├─────────────────────────────────────────────────────┤
│                                                      │
│    ┌─────────┐      ┌─────────┐  ┌─────────┐       │
│    │ Server  │  →   │ Server1 │  │ Server2 │       │
│    │         │      └─────────┘  └─────────┘       │
│    │         │      ┌─────────┐  ┌─────────┐       │
│    └─────────┘      │ Server3 │  │ Server4 │       │
│                     └─────────┘  └─────────┘       │
│                                                      │
│    기존 서버             여러 서버로 분산            │
└─────────────────────────────────────────────────────┘
```

**정의**: 여러 서버를 추가하여 부하를 분산시키는 방법

**장점**:
- 이론적으로 무한 확장 가능
- 고가용성(High Availability) 달성
- 비용 효율적 (상용 하드웨어 사용)
- 점진적 확장 가능

**단점**:
- 구현 복잡도 증가
- 데이터 일관성 관리 어려움
- 네트워크 오버헤드 발생
- Stateless 설계 필요

### 비교 테이블

| 구분 | Scale Up | Scale Out |
|------|----------|-----------|
| 복잡도 | 낮음 | 높음 |
| 비용 효율성 | 낮음 (비선형 증가) | 높음 (선형 증가) |
| 가용성 | 낮음 (SPOF) | 높음 |
| 확장 한계 | 있음 | 이론적 무한 |
| 적합한 환경 | 예측 가능한 워크로드 | 가변적 워크로드 |
| 대표 사례 | 금융 시스템, RDBMS | 웹 서비스, NoSQL |

### 하이브리드 접근법

실제 환경에서는 두 가지 전략을 조합하여 사용합니다:

```java
/**
 * 하이브리드 확장 전략 예시
 * - 애플리케이션 서버: Scale Out
 * - 데이터베이스: Scale Up + Read Replica
 */
@Configuration
public class HybridScalingConfig {

    // Primary DB - Scale Up으로 성능 향상
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://primary-db:3306/app");
        config.setMaximumPoolSize(50); // 고성능 서버에서 더 많은 커넥션 처리
        return new HikariDataSource(config);
    }

    // Read Replica - Scale Out으로 읽기 분산
    @Bean
    public DataSource readReplicaDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://read-replica:3306/app");
        config.setMaximumPoolSize(30);
        config.setReadOnly(true);
        return new HikariDataSource(config);
    }
}
```

---

## Stateless 설계

### Stateless vs Stateful

```
┌──────────────────────────────────────────────────────────────┐
│                    Stateful vs Stateless                      │
├────────────────────────────┬─────────────────────────────────┤
│        Stateful            │          Stateless              │
├────────────────────────────┼─────────────────────────────────┤
│                            │                                  │
│  Client ──→ Server A       │  Client ──→ Load Balancer       │
│              │             │                │                 │
│         [Session]          │    ┌───────────┼───────────┐    │
│              │             │    ↓           ↓           ↓    │
│         처리 계속          │  Server A  Server B  Server C   │
│                            │    │           │           │    │
│  서버 A 장애 시            │    └───────────┴───────────┘    │
│  세션 유실!                │              ↓                  │
│                            │    [External Session Store]     │
│                            │                                  │
└────────────────────────────┴─────────────────────────────────┘
```

### Stateless 설계 원칙

```java
/**
 * Stateless 서비스 설계 예시
 *
 * 핵심 원칙:
 * 1. 서버에 상태를 저장하지 않음
 * 2. 모든 요청에 필요한 정보 포함
 * 3. 외부 저장소를 통한 상태 관리
 */
@RestController
@RequestMapping("/api/orders")
public class StatelessOrderController {

    private final OrderService orderService;
    private final TokenService tokenService;

    /**
     * Stateless 요청 처리
     * - 토큰에서 사용자 정보 추출
     * - 서버 메모리에 사용자 상태 저장하지 않음
     */
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestHeader("Authorization") String authHeader,
            @RequestBody OrderRequest request) {

        // 토큰에서 사용자 정보 추출 (서버 상태 의존 없음)
        String token = authHeader.replace("Bearer ", "");
        UserContext userContext = tokenService.parseToken(token);

        // 요청 처리 (상태 없는 처리)
        Order order = orderService.createOrder(userContext, request);

        return ResponseEntity.ok(new OrderResponse(order));
    }
}

/**
 * Stateful 안티패턴 예시 (피해야 할 패턴)
 */
@RestController
public class StatefulAntiPatternController {

    // ❌ 잘못된 예시: 서버 메모리에 상태 저장
    private Map<String, ShoppingCart> userCarts = new HashMap<>();

    @PostMapping("/cart/add")
    public void addToCart(HttpSession session, @RequestBody Item item) {
        // ❌ HttpSession 사용 - 서버 affinity 필요
        String userId = (String) session.getAttribute("userId");

        // ❌ 인스턴스 변수에 상태 저장
        userCarts.computeIfAbsent(userId, k -> new ShoppingCart())
                 .addItem(item);
    }
}
```

### Stateless 설계를 위한 체크리스트

```java
/**
 * Stateless 설계 검증 체크리스트
 */
public class StatelessDesignChecklist {

    // 1. 인메모리 캐시 대신 분산 캐시 사용
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)))
            .build();
    }

    // 2. 파일 저장소 대신 오브젝트 스토리지 사용
    @Bean
    public AmazonS3 amazonS3() {
        return AmazonS3ClientBuilder.standard()
            .withRegion(Regions.AP_NORTHEAST_2)
            .build();
    }

    // 3. 로컬 스케줄러 대신 분산 스케줄러 사용
    @Bean
    public SchedulerFactoryBean schedulerFactory(DataSource dataSource) {
        SchedulerFactoryBean factory = new SchedulerFactoryBean();
        factory.setDataSource(dataSource); // DB 기반 스케줄링
        factory.setOverwriteExistingJobs(true);
        return factory;
    }
}
```

---

## 세션 관리 전략

### 1. Sticky Session (세션 고정)

```
┌────────────────────────────────────────────────────┐
│                 Sticky Session                      │
├────────────────────────────────────────────────────┤
│                                                     │
│   Client A ──→ ┌──────────────┐ ──→ Server 1       │
│                │     Load     │                     │
│   Client B ──→ │   Balancer   │ ──→ Server 2       │
│                │  (Cookie/IP) │                     │
│   Client C ──→ └──────────────┘ ──→ Server 1       │
│                                                     │
│   * 같은 클라이언트는 항상 같은 서버로 라우팅       │
│   * 서버 장애 시 세션 유실                          │
└────────────────────────────────────────────────────┘
```

```nginx
# Nginx Sticky Session 설정
upstream backend {
    ip_hash;  # IP 기반 sticky session
    server backend1.example.com:8080;
    server backend2.example.com:8080;
    server backend3.example.com:8080;
}

# 또는 Cookie 기반
upstream backend {
    sticky cookie srv_id expires=1h domain=.example.com path=/;
    server backend1.example.com:8080;
    server backend2.example.com:8080;
}
```

### 2. 세션 클러스터링

```java
/**
 * Tomcat 세션 클러스터링 설정
 * 장점: 세션 복제로 장애 대응
 * 단점: 네트워크 오버헤드, 확장성 제한
 */
@Configuration
public class SessionClusterConfig {

    @Bean
    public TomcatServletWebServerFactory tomcatFactory() {
        return new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                // DeltaManager: 모든 노드에 세션 복제
                DeltaManager manager = new DeltaManager();
                manager.setExpireSessionsOnShutdown(false);
                manager.setNotifyListenersOnReplication(true);
                context.setManager(manager);
            }
        };
    }
}
```

### 3. 외부 세션 저장소 (권장)

```java
/**
 * Redis 기반 세션 관리 (Spring Session)
 * 가장 권장되는 방식 - 완전한 Stateless 달성
 */
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
public class RedisSessionConfig {

    @Bean
    public LettuceConnectionFactory connectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("redis-cluster.example.com");
        config.setPort(6379);
        config.setPassword(RedisPassword.of("secret"));
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
        // JSON 직렬화로 디버깅 용이성 확보
        return new GenericJackson2JsonRedisSerializer();
    }
}

/**
 * 세션 사용 예시
 */
@RestController
public class SessionController {

    @GetMapping("/session/info")
    public Map<String, Object> getSessionInfo(HttpSession session) {
        Map<String, Object> info = new HashMap<>();
        info.put("sessionId", session.getId());
        info.put("creationTime", session.getCreationTime());
        info.put("lastAccessedTime", session.getLastAccessedTime());
        // Redis에 자동 저장/조회됨
        return info;
    }
}
```

### 4. JWT 기반 Stateless 인증 (권장)

```java
/**
 * JWT 기반 완전 Stateless 인증
 * 세션 저장소 자체가 필요 없음
 */
@Component
public class JwtTokenProvider {

    private final SecretKey secretKey;
    private final long accessTokenValidityMs = 3600000; // 1시간
    private final long refreshTokenValidityMs = 604800000; // 7일

    public JwtTokenProvider(@Value("${jwt.secret}") String secret) {
        this.secretKey = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    /**
     * Access Token 생성
     * - 필요한 사용자 정보를 토큰에 포함
     * - 서버는 토큰만으로 인증/인가 처리
     */
    public String createAccessToken(UserDetails userDetails) {
        Date now = new Date();
        Date validity = new Date(now.getTime() + accessTokenValidityMs);

        return Jwts.builder()
            .setSubject(userDetails.getUsername())
            .claim("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()))
            .claim("userId", ((CustomUserDetails) userDetails).getUserId())
            .setIssuedAt(now)
            .setExpiration(validity)
            .signWith(secretKey, SignatureAlgorithm.HS256)
            .compact();
    }

    /**
     * 토큰 검증 및 파싱
     * - 서버 상태 없이 토큰만으로 사용자 정보 추출
     */
    public Authentication getAuthentication(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(secretKey)
            .build()
            .parseClaimsJws(token)
            .getBody();

        String username = claims.getSubject();
        List<String> roles = claims.get("roles", List.class);
        Long userId = claims.get("userId", Long.class);

        List<SimpleGrantedAuthority> authorities = roles.stream()
            .map(SimpleGrantedAuthority::new)
            .collect(Collectors.toList());

        UserDetails userDetails = new CustomUserDetails(userId, username, "", authorities);
        return new UsernamePasswordAuthenticationToken(userDetails, "", authorities);
    }
}
```

### 세션 관리 전략 비교

| 전략 | 확장성 | 가용성 | 복잡도 | 사용 사례 |
|------|--------|--------|--------|-----------|
| Sticky Session | 낮음 | 낮음 | 낮음 | 레거시 시스템 |
| 세션 클러스터링 | 중간 | 중간 | 중간 | 소규모 클러스터 |
| Redis Session | 높음 | 높음 | 중간 | 대규모 웹 서비스 |
| JWT | 매우 높음 | 매우 높음 | 중간 | API 서버, MSA |

---

## 데이터베이스 확장

### 1. 읽기 복제 (Read Replica)

```
┌──────────────────────────────────────────────────────────┐
│                    Read Replica Pattern                   │
├──────────────────────────────────────────────────────────┤
│                                                           │
│   Application                                             │
│       │                                                   │
│       ├── Write ──→ Primary DB (Master)                  │
│       │                    │                              │
│       │                    │ Replication                  │
│       │                    ↓                              │
│       └── Read ───→ Read Replica 1                       │
│           Read ───→ Read Replica 2                       │
│           Read ───→ Read Replica 3                       │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

```java
/**
 * 읽기/쓰기 분리를 위한 DataSource Routing
 */
public class ReplicationRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        return isReadOnly ? DataSourceType.REPLICA : DataSourceType.PRIMARY;
    }
}

@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {

        ReplicationRoutingDataSource routingDataSource = new ReplicationRoutingDataSource();

        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(DataSourceType.PRIMARY, primary);
        dataSourceMap.put(DataSourceType.REPLICA, replica);

        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(primary);

        return routingDataSource;
    }

    @Bean
    public DataSource dataSource(DataSource routingDataSource) {
        return new LazyConnectionDataSourceProxy(routingDataSource);
    }
}

/**
 * 읽기 전용 트랜잭션은 자동으로 Replica로 라우팅
 */
@Service
@Transactional(readOnly = true)
public class ProductQueryService {

    private final ProductRepository productRepository;

    // Replica에서 읽기
    public List<Product> findAllProducts() {
        return productRepository.findAll();
    }

    // Replica에서 읽기
    public Product findById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }
}

@Service
@Transactional
public class ProductCommandService {

    // Primary에서 쓰기
    public Product createProduct(ProductCreateRequest request) {
        return productRepository.save(request.toEntity());
    }
}
```

### 2. 샤딩 (Sharding)

```
┌──────────────────────────────────────────────────────────────┐
│                    Database Sharding                          │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│   user_id: 1~1000000        user_id: 1000001~2000000         │
│          │                           │                        │
│          ↓                           ↓                        │
│   ┌─────────────┐             ┌─────────────┐                │
│   │   Shard 1   │             │   Shard 2   │                │
│   │  (DB West)  │             │  (DB East)  │                │
│   └─────────────┘             └─────────────┘                │
│                                                               │
│   Sharding Key: user_id                                       │
│   Strategy: Range-based Sharding                              │
└──────────────────────────────────────────────────────────────┘
```

```java
/**
 * 샤딩 전략 구현
 */
public interface ShardingStrategy {
    String determineShard(Object shardKey);
}

/**
 * 해시 기반 샤딩 전략
 * - 균등 분산에 유리
 * - 범위 쿼리에 불리
 */
public class HashBasedShardingStrategy implements ShardingStrategy {

    private final int shardCount;
    private final List<String> shardNames;

    public HashBasedShardingStrategy(int shardCount) {
        this.shardCount = shardCount;
        this.shardNames = IntStream.range(0, shardCount)
            .mapToObj(i -> "shard_" + i)
            .collect(Collectors.toList());
    }

    @Override
    public String determineShard(Object shardKey) {
        int hash = Math.abs(shardKey.hashCode());
        int shardIndex = hash % shardCount;
        return shardNames.get(shardIndex);
    }
}

/**
 * 범위 기반 샤딩 전략
 * - 범위 쿼리에 유리
 * - 핫스팟 발생 가능성
 */
public class RangeBasedShardingStrategy implements ShardingStrategy {

    private final NavigableMap<Long, String> rangeMap = new TreeMap<>();

    public RangeBasedShardingStrategy() {
        rangeMap.put(0L, "shard_0");           // 0 ~ 999999
        rangeMap.put(1000000L, "shard_1");     // 1000000 ~ 1999999
        rangeMap.put(2000000L, "shard_2");     // 2000000 ~
    }

    @Override
    public String determineShard(Object shardKey) {
        Long key = (Long) shardKey;
        return rangeMap.floorEntry(key).getValue();
    }
}

/**
 * 샤딩 DataSource Router
 */
public class ShardingDataSourceRouter extends AbstractRoutingDataSource {

    private static final ThreadLocal<String> currentShard = new ThreadLocal<>();
    private final ShardingStrategy shardingStrategy;

    public static void setShardKey(Object shardKey) {
        // ShardingStrategy를 통해 샤드 결정
        // 실제 구현에서는 의존성 주입 필요
    }

    public static void clearShardKey() {
        currentShard.remove();
    }

    @Override
    protected Object determineCurrentLookupKey() {
        return currentShard.get();
    }
}

/**
 * 샤딩 적용 서비스
 */
@Service
public class ShardedUserService {

    private final ShardingStrategy shardingStrategy;
    private final UserRepository userRepository;

    @Transactional
    public User createUser(UserCreateRequest request) {
        // 사용자 ID로 샤드 결정
        Long userId = generateUserId();

        try {
            ShardingDataSourceRouter.setShardKey(userId);
            User user = new User(userId, request.getName(), request.getEmail());
            return userRepository.save(user);
        } finally {
            ShardingDataSourceRouter.clearShardKey();
        }
    }

    public User findUser(Long userId) {
        try {
            ShardingDataSourceRouter.setShardKey(userId);
            return userRepository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));
        } finally {
            ShardingDataSourceRouter.clearShardKey();
        }
    }
}
```

### 3. 파티셔닝

```sql
-- MySQL Range Partitioning 예시
CREATE TABLE orders (
    id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2),
    status VARCHAR(20),
    PRIMARY KEY (id, order_date)
) PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- List Partitioning (지역별)
CREATE TABLE customers (
    id BIGINT NOT NULL,
    name VARCHAR(100),
    region VARCHAR(20),
    PRIMARY KEY (id, region)
) PARTITION BY LIST COLUMNS(region) (
    PARTITION p_korea VALUES IN ('seoul', 'busan', 'daegu'),
    PARTITION p_usa VALUES IN ('new_york', 'la', 'chicago'),
    PARTITION p_japan VALUES IN ('tokyo', 'osaka', 'kyoto')
);

-- 파티션 관리 (오래된 데이터 아카이브)
ALTER TABLE orders DROP PARTITION p2022;
```

### 데이터베이스 확장 전략 비교

| 전략 | 구현 복잡도 | 확장성 | 데이터 일관성 | 적합한 상황 |
|------|-------------|--------|---------------|-------------|
| Read Replica | 낮음 | 중간 | 높음 (지연 있음) | 읽기 비중 높은 서비스 |
| Sharding | 높음 | 높음 | 중간 | 대용량 쓰기 |
| Partitioning | 중간 | 중간 | 높음 | 시계열/로그 데이터 |

---

## 실전 적용 가이드

### 확장 전략 결정 플로우차트

```
시작
  │
  ▼
현재 병목이 CPU/메모리인가?
  │
  ├── Yes ──→ Scale Up 고려
  │            - 비용 대비 효과 분석
  │            - 하드웨어 한계 확인
  │
  └── No
      │
      ▼
    단일 서버로 처리 가능한가?
      │
      ├── Yes ──→ Scale Up
      │
      └── No
          │
          ▼
        애플리케이션이 Stateless인가?
          │
          ├── Yes ──→ Scale Out
          │
          └── No
              │
              ▼
            Stateless로 리팩토링
              │
              └──→ Scale Out
```

### Auto Scaling 설정 (Kubernetes)

```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  minReplicas: 3
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 5분간 안정화
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # 즉시 확장
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

```yaml
# Vertical Pod Autoscaler
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: backend-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
```

---

## 참고 자료

- [Release It!: Design and Deploy Production-Ready Software](https://pragprog.com/titles/mnee2/release-it-second-edition/) - Michael Nygard
- [Horizontal vs Vertical Scaling - DigitalOcean](https://www.digitalocean.com/resources/articles/horizontal-scaling-vs-vertical-scaling)
- [System Design: Horizontal and Vertical Scaling - GeeksforGeeks](https://www.geeksforgeeks.org/system-design/system-design-horizontal-and-vertical-scaling/)
- [Spring Session with Redis](https://docs.spring.io/spring-session/reference/guides/boot-redis.html)
- [Kubernetes HPA Documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

# 통합 테스트 (Integration Testing)

## 목차
1. [통합 테스트 개요](#통합-테스트-개요)
2. [테스트 컨테이너 (Testcontainers)](#테스트-컨테이너-testcontainers)
3. [데이터베이스 테스트](#데이터베이스-테스트)
4. [API 테스트](#api-테스트)
5. [테스트 격리](#테스트-격리)
---

## 통합 테스트 개요

### 정의

통합 테스트는 여러 컴포넌트가 함께 동작할 때 올바르게 작동하는지 검증하는 테스트이다. 단위 테스트와 달리 실제 의존성(데이터베이스, 외부 API, 메시지 큐 등)을 사용한다.

### 단위 테스트 vs 통합 테스트

```
단위 테스트:
┌─────────────────┐
│  UserService    │ ◄── 테스트 대상
├─────────────────┤
│  Mock Repository│ ◄── Mock으로 대체
└─────────────────┘

통합 테스트:
┌─────────────────┐
│  UserService    │ ◄── 테스트 대상
├─────────────────┤
│  UserRepository │ ◄── 실제 구현체
├─────────────────┤
│  PostgreSQL DB  │ ◄── 실제 데이터베이스
└─────────────────┘
```

### 통합 테스트의 범위

```
      ┌─────────────────────────────────────────────────┐
      │              통합 테스트 범위                     │
      │  ┌─────────┐   ┌─────────┐   ┌─────────┐        │
      │  │Controller│──►│ Service │──►│Repository│       │
      │  └─────────┘   └─────────┘   └─────────┘        │
      │       │                            │            │
      │       ▼                            ▼            │
      │  ┌─────────┐                 ┌─────────┐        │
      │  │ 외부 API │                 │   DB    │        │
      │  └─────────┘                 └─────────┘        │
      └─────────────────────────────────────────────────┘
```

---

## 테스트 컨테이너 (Testcontainers)

### 개념

Testcontainers는 Docker 컨테이너를 프로그래밍 방식으로 관리하여 통합 테스트에서 실제 의존성을 제공하는 라이브러리이다.

### 장점

1. **프로덕션 환경과 동일한 기술 스택 사용**
   - H2 대신 실제 PostgreSQL
   - 임베디드 Redis 대신 실제 Redis

2. **테스트 격리**
   - 각 테스트마다 깨끗한 상태의 컨테이너
   - 테스트 간 데이터 오염 방지

3. **CI/CD 친화적**
   - Docker만 있으면 어디서든 실행 가능
   - 로컬과 CI 환경에서 동일한 결과

### Java/Spring Boot 예시

```java
// build.gradle
dependencies {
    testImplementation 'org.testcontainers:testcontainers:1.19.3'
    testImplementation 'org.testcontainers:postgresql:1.19.3'
    testImplementation 'org.testcontainers:junit-jupiter:1.19.3'
}
```

```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        // 동적으로 생성된 컨테이너 정보를 Spring에 주입
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    void shouldSaveAndFindUser() {
        // Given
        User user = new User("John", "john@example.com");

        // When
        User saved = userRepository.save(user);
        User found = userRepository.findById(saved.getId()).orElseThrow();

        // Then
        assertThat(found.getName()).isEqualTo("John");
        assertThat(found.getEmail()).isEqualTo("john@example.com");
    }

    @Test
    void shouldFindUsersByNameContaining() {
        // Given
        userRepository.saveAll(List.of(
            new User("John Doe", "john@example.com"),
            new User("Jane Doe", "jane@example.com"),
            new User("Bob Smith", "bob@example.com")
        ));

        // When
        List<User> users = userRepository.findByNameContaining("Doe");

        // Then
        assertThat(users).hasSize(2);
        assertThat(users).extracting(User::getName)
            .containsExactlyInAnyOrder("John Doe", "Jane Doe");
    }
}
```

### 싱글톤 컨테이너 패턴 (성능 최적화)

```java
// 모든 테스트 클래스에서 공유하는 싱글톤 컨테이너
public abstract class AbstractIntegrationTest {

    static final PostgreSQLContainer<?> postgres;
    static final GenericContainer<?> redis;

    static {
        // 한 번만 시작하고 JVM 종료 시 자동 정리
        postgres = new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
            .withReuse(true);  // 컨테이너 재사용

        redis = new GenericContainer<>("redis:7-alpine")
            .withExposedPorts(6379)
            .withReuse(true);

        postgres.start();
        redis.start();
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
}

// 개별 테스트 클래스
class OrderServiceIntegrationTest extends AbstractIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Test
    void shouldCreateOrder() {
        // 컨테이너가 이미 실행 중
        Order order = orderService.createOrder(new CreateOrderRequest(...));
        assertThat(order.getId()).isNotNull();
    }
}
```

### Python 예시

```python
# requirements.txt
# testcontainers==3.7.1
# pytest==7.4.3

import pytest
from testcontainers.postgres import PostgresContainer
from testcontainers.redis import RedisContainer
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="session")
def postgres_container():
    """세션 범위의 PostgreSQL 컨테이너"""
    with PostgresContainer("postgres:15-alpine") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def db_engine(postgres_container):
    """데이터베이스 엔진"""
    engine = create_engine(postgres_container.get_connection_url())
    # 테이블 생성
    Base.metadata.create_all(engine)
    return engine

@pytest.fixture
def db_session(db_engine):
    """각 테스트마다 새로운 세션 (트랜잭션 롤백)"""
    connection = db_engine.connect()
    transaction = connection.begin()
    Session = sessionmaker(bind=connection)
    session = Session()

    yield session

    session.close()
    transaction.rollback()
    connection.close()

@pytest.fixture(scope="session")
def redis_container():
    """세션 범위의 Redis 컨테이너"""
    with RedisContainer("redis:7-alpine") as redis:
        yield redis


class TestUserRepository:

    def test_should_save_and_find_user(self, db_session):
        # Given
        repository = UserRepository(db_session)
        user = User(name="John", email="john@example.com")

        # When
        saved = repository.save(user)
        found = repository.find_by_id(saved.id)

        # Then
        assert found is not None
        assert found.name == "John"
        assert found.email == "john@example.com"

    def test_should_find_users_by_email_domain(self, db_session):
        # Given
        repository = UserRepository(db_session)
        repository.save(User(name="John", email="john@example.com"))
        repository.save(User(name="Jane", email="jane@example.com"))
        repository.save(User(name="Bob", email="bob@other.com"))

        # When
        users = repository.find_by_email_domain("example.com")

        # Then
        assert len(users) == 2
```

### Node.js/TypeScript 예시

```typescript
// testcontainers 패키지 사용
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { RedisContainer, StartedRedisContainer } from '@testcontainers/redis';
import { Pool } from 'pg';
import Redis from 'ioredis';

describe('Integration Tests', () => {
    let postgresContainer: StartedPostgreSqlContainer;
    let redisContainer: StartedRedisContainer;
    let pool: Pool;
    let redis: Redis;

    beforeAll(async () => {
        // 컨테이너 시작 (병렬로)
        [postgresContainer, redisContainer] = await Promise.all([
            new PostgreSqlContainer('postgres:15-alpine').start(),
            new RedisContainer('redis:7-alpine').start()
        ]);

        // 연결 설정
        pool = new Pool({
            connectionString: postgresContainer.getConnectionUri()
        });

        redis = new Redis({
            host: redisContainer.getHost(),
            port: redisContainer.getPort()
        });

        // 스키마 생성
        await pool.query(`
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email VARCHAR(100) UNIQUE NOT NULL,
                created_at TIMESTAMP DEFAULT NOW()
            )
        `);
    }, 60000);

    afterAll(async () => {
        await pool.end();
        await redis.quit();
        await postgresContainer.stop();
        await redisContainer.stop();
    });

    beforeEach(async () => {
        // 각 테스트 전 데이터 정리
        await pool.query('TRUNCATE users RESTART IDENTITY CASCADE');
        await redis.flushall();
    });

    describe('UserRepository', () => {
        it('should save and find user', async () => {
            // Given
            const userRepository = new UserRepository(pool);

            // When
            const saved = await userRepository.save({
                name: 'John',
                email: 'john@example.com'
            });
            const found = await userRepository.findById(saved.id);

            // Then
            expect(found).toBeDefined();
            expect(found?.name).toBe('John');
            expect(found?.email).toBe('john@example.com');
        });
    });

    describe('CacheService', () => {
        it('should cache user data in Redis', async () => {
            // Given
            const cacheService = new CacheService(redis);
            const user = { id: 1, name: 'John', email: 'john@example.com' };

            // When
            await cacheService.set('user:1', user, 3600);
            const cached = await cacheService.get<typeof user>('user:1');

            // Then
            expect(cached).toEqual(user);
        });
    });
});
```

---

## 데이터베이스 테스트

### 테스트 데이터 관리 전략

#### 1. 트랜잭션 롤백 방식

```java
@SpringBootTest
@Transactional  // 각 테스트 후 자동 롤백
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Test
    void shouldCreateUser() {
        // Given & When
        User user = userService.createUser("John", "john@example.com");

        // Then
        assertThat(userRepository.findById(user.getId())).isPresent();

        // 테스트 종료 후 자동 롤백
    }
}
```

**주의사항:**
```java
// 트랜잭션 롤백 방식의 함정
@Test
@Transactional
void shouldFailForDuplicateEmail() {
    userService.createUser("John", "john@example.com");

    // 문제: 같은 트랜잭션 내에서는 DB 제약조건이
    // 즉시 체크되지 않을 수 있음
    assertThrows(DataIntegrityViolationException.class, () ->
        userService.createUser("Jane", "john@example.com")
    );
    // 실제로는 커밋 시점에 예외 발생
}

// 해결: flush 호출
@Test
@Transactional
void shouldFailForDuplicateEmail() {
    userService.createUser("John", "john@example.com");
    entityManager.flush();  // 즉시 DB에 반영

    assertThrows(DataIntegrityViolationException.class, () -> {
        userService.createUser("Jane", "john@example.com");
        entityManager.flush();
    });
}
```

#### 2. 명시적 정리 방식

```java
@SpringBootTest
class OrderServiceIntegrationTest {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        // 외래 키 순서 고려한 정리
        orderRepository.deleteAll();
        userRepository.deleteAll();
    }

    @Test
    void shouldCreateOrderForExistingUser() {
        // Given
        User user = userRepository.save(new User("John", "john@example.com"));

        // When
        Order order = orderService.createOrder(user.getId(), List.of(
            new OrderItem("Product A", 1000, 2),
            new OrderItem("Product B", 500, 3)
        ));

        // Then
        assertThat(order.getTotalAmount()).isEqualTo(3500);
    }
}
```

#### 3. 테스트 데이터 빌더 패턴

```java
public class TestDataBuilder {

    private final EntityManager em;

    public TestDataBuilder(EntityManager em) {
        this.em = em;
    }

    public UserBuilder aUser() {
        return new UserBuilder(em);
    }

    public OrderBuilder anOrder() {
        return new OrderBuilder(em);
    }

    public static class UserBuilder {
        private final EntityManager em;
        private String name = "Default User";
        private String email = "default@example.com";
        private UserType type = UserType.REGULAR;

        UserBuilder(EntityManager em) {
            this.em = em;
        }

        public UserBuilder withName(String name) {
            this.name = name;
            return this;
        }

        public UserBuilder withEmail(String email) {
            this.email = email;
            return this;
        }

        public UserBuilder asPremium() {
            this.type = UserType.PREMIUM;
            return this;
        }

        public User build() {
            User user = new User(name, email, type);
            em.persist(user);
            return user;
        }
    }

    public static class OrderBuilder {
        private final EntityManager em;
        private User user;
        private List<OrderItem> items = new ArrayList<>();
        private OrderStatus status = OrderStatus.PENDING;

        OrderBuilder(EntityManager em) {
            this.em = em;
        }

        public OrderBuilder forUser(User user) {
            this.user = user;
            return this;
        }

        public OrderBuilder withItem(String name, int price, int quantity) {
            items.add(new OrderItem(name, price, quantity));
            return this;
        }

        public OrderBuilder withStatus(OrderStatus status) {
            this.status = status;
            return this;
        }

        public Order build() {
            Order order = new Order(user, items, status);
            em.persist(order);
            return order;
        }
    }
}

// 테스트에서 사용
@Test
void shouldApplyPremiumDiscount() {
    // Given
    User premiumUser = testData.aUser()
        .withName("John")
        .asPremium()
        .build();

    Order order = testData.anOrder()
        .forUser(premiumUser)
        .withItem("Laptop", 1000000, 1)
        .withItem("Mouse", 50000, 2)
        .build();

    // When
    int discount = discountService.calculateDiscount(order);

    // Then
    assertThat(discount).isEqualTo(110000);  // 10% 프리미엄 할인
}
```

### 마이그레이션 도구와 함께 사용

```java
// Flyway 사용
@Testcontainers
@SpringBootTest
class DatabaseMigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.flyway.enabled", () -> "true");
        registry.add("spring.flyway.locations", () -> "classpath:db/migration");
    }

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    void shouldApplyAllMigrations() {
        // 마이그레이션이 정상 적용되었는지 확인
        Integer tableCount = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public'",
            Integer.class
        );
        assertThat(tableCount).isGreaterThan(0);
    }

    @Test
    void shouldHaveCorrectSchema() {
        // users 테이블 구조 확인
        List<Map<String, Object>> columns = jdbcTemplate.queryForList(
            "SELECT column_name, data_type FROM information_schema.columns " +
            "WHERE table_name = 'users' ORDER BY ordinal_position"
        );

        assertThat(columns).extracting(c -> c.get("column_name"))
            .contains("id", "name", "email", "created_at");
    }
}
```

---

## API 테스트

### Spring Boot REST API 테스트

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserApiIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    @LocalServerPort
    private int port;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    void shouldCreateUser() {
        // Given
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com");

        // When
        ResponseEntity<UserResponse> response = restTemplate.postForEntity(
            "/api/users",
            request,
            UserResponse.class
        );

        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();
        assertThat(response.getBody().getName()).isEqualTo("John");
        assertThat(response.getBody().getId()).isNotNull();

        // DB 확인
        assertThat(userRepository.findAll()).hasSize(1);
    }

    @Test
    void shouldReturnNotFoundForNonExistentUser() {
        // When
        ResponseEntity<ErrorResponse> response = restTemplate.getForEntity(
            "/api/users/999",
            ErrorResponse.class
        );

        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NOT_FOUND);
        assertThat(response.getBody().getMessage()).contains("User not found");
    }

    @Test
    void shouldValidateUserInput() {
        // Given - 유효하지 않은 요청
        CreateUserRequest request = new CreateUserRequest("", "invalid-email");

        // When
        ResponseEntity<ErrorResponse> response = restTemplate.postForEntity(
            "/api/users",
            request,
            ErrorResponse.class
        );

        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.BAD_REQUEST);
        assertThat(response.getBody().getErrors())
            .containsKeys("name", "email");
    }

    @Test
    void shouldPaginateUsers() {
        // Given
        for (int i = 0; i < 25; i++) {
            userRepository.save(new User("User" + i, "user" + i + "@example.com"));
        }

        // When
        ResponseEntity<PagedResponse<UserResponse>> response = restTemplate.exchange(
            "/api/users?page=0&size=10",
            HttpMethod.GET,
            null,
            new ParameterizedTypeReference<>() {}
        );

        // Then
        assertThat(response.getBody().getContent()).hasSize(10);
        assertThat(response.getBody().getTotalElements()).isEqualTo(25);
        assertThat(response.getBody().getTotalPages()).isEqualTo(3);
    }
}
```

### WebTestClient 사용 (WebFlux)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class ReactiveApiIntegrationTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void shouldCreateUser() {
        webTestClient.post()
            .uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new CreateUserRequest("John", "john@example.com"))
            .exchange()
            .expectStatus().isCreated()
            .expectBody()
            .jsonPath("$.id").isNotEmpty()
            .jsonPath("$.name").isEqualTo("John")
            .jsonPath("$.email").isEqualTo("john@example.com");
    }

    @Test
    void shouldStreamUsers() {
        webTestClient.get()
            .uri("/api/users/stream")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(UserResponse.class)
            .hasSize(10);
    }
}
```

### Python FastAPI 테스트

```python
import pytest
from fastapi.testclient import TestClient
from httpx import AsyncClient
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.database import get_db, Base
from app.models import User

@pytest.fixture(scope="session")
def postgres_container():
    with PostgresContainer("postgres:15-alpine") as postgres:
        yield postgres

@pytest.fixture(scope="session")
def db_engine(postgres_container):
    engine = create_engine(postgres_container.get_connection_url())
    Base.metadata.create_all(engine)
    return engine

@pytest.fixture
def db_session(db_engine):
    connection = db_engine.connect()
    transaction = connection.begin()
    Session = sessionmaker(bind=connection)
    session = Session()

    yield session

    session.close()
    transaction.rollback()
    connection.close()

@pytest.fixture
def client(db_session):
    def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as test_client:
        yield test_client
    app.dependency_overrides.clear()


class TestUserAPI:

    def test_create_user(self, client):
        # Given
        payload = {"name": "John", "email": "john@example.com"}

        # When
        response = client.post("/api/users", json=payload)

        # Then
        assert response.status_code == 201
        data = response.json()
        assert data["name"] == "John"
        assert data["email"] == "john@example.com"
        assert "id" in data

    def test_get_user(self, client, db_session):
        # Given
        user = User(name="John", email="john@example.com")
        db_session.add(user)
        db_session.commit()
        db_session.refresh(user)

        # When
        response = client.get(f"/api/users/{user.id}")

        # Then
        assert response.status_code == 200
        data = response.json()
        assert data["name"] == "John"

    def test_user_not_found(self, client):
        # When
        response = client.get("/api/users/999")

        # Then
        assert response.status_code == 404
        assert "not found" in response.json()["detail"].lower()

    def test_validation_error(self, client):
        # Given - 유효하지 않은 이메일
        payload = {"name": "John", "email": "invalid-email"}

        # When
        response = client.post("/api/users", json=payload)

        # Then
        assert response.status_code == 422
        errors = response.json()["detail"]
        assert any("email" in str(e).lower() for e in errors)

    @pytest.mark.asyncio
    async def test_concurrent_requests(self, db_session):
        """비동기 클라이언트로 동시 요청 테스트"""
        async with AsyncClient(app=app, base_url="http://test") as ac:
            tasks = [
                ac.post("/api/users", json={"name": f"User{i}", "email": f"user{i}@example.com"})
                for i in range(10)
            ]
            responses = await asyncio.gather(*tasks)

            assert all(r.status_code == 201 for r in responses)
```

### 외부 API 통합 테스트 (WireMock)

```java
@SpringBootTest
@AutoConfigureWireMock(port = 0)  // 랜덤 포트
class PaymentServiceIntegrationTest {

    @Autowired
    private PaymentService paymentService;

    @Value("${wiremock.server.port}")
    private int wireMockPort;

    @BeforeEach
    void setUp() {
        // 외부 결제 API 모킹
        stubFor(post(urlPathEqualTo("/api/payments"))
            .withRequestBody(matchingJsonPath("$.amount"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "paymentId": "PAY-12345",
                        "status": "APPROVED",
                        "transactionId": "TXN-67890"
                    }
                    """)));
    }

    @Test
    void shouldProcessPayment() {
        // Given
        PaymentRequest request = new PaymentRequest(10000, "CREDIT_CARD");

        // When
        PaymentResult result = paymentService.processPayment(request);

        // Then
        assertThat(result.getStatus()).isEqualTo("APPROVED");
        assertThat(result.getTransactionId()).isEqualTo("TXN-67890");

        // WireMock 호출 검증
        verify(postRequestedFor(urlPathEqualTo("/api/payments"))
            .withRequestBody(matchingJsonPath("$.amount", equalTo("10000"))));
    }

    @Test
    void shouldHandlePaymentFailure() {
        // Given - 결제 실패 시나리오
        stubFor(post(urlPathEqualTo("/api/payments"))
            .willReturn(aResponse()
                .withStatus(400)
                .withBody("""
                    {
                        "error": "INSUFFICIENT_FUNDS",
                        "message": "Not enough balance"
                    }
                    """)));

        PaymentRequest request = new PaymentRequest(1000000, "CREDIT_CARD");

        // When & Then
        assertThrows(PaymentFailedException.class, () ->
            paymentService.processPayment(request)
        );
    }

    @Test
    void shouldHandleTimeout() {
        // Given - 타임아웃 시나리오
        stubFor(post(urlPathEqualTo("/api/payments"))
            .willReturn(aResponse()
                .withStatus(200)
                .withFixedDelay(5000)));  // 5초 지연

        PaymentRequest request = new PaymentRequest(10000, "CREDIT_CARD");

        // When & Then
        assertThrows(PaymentTimeoutException.class, () ->
            paymentService.processPayment(request)
        );
    }
}
```

---

## 테스트 격리

### 테스트 격리가 중요한 이유

```
❌ 격리되지 않은 테스트:
Test A → 데이터 생성 → 정리 안 함
Test B → Test A의 데이터로 인해 실패

✅ 격리된 테스트:
Test A → 데이터 생성 → 정리
Test B → 깨끗한 상태에서 시작 → 성공
```

### 격리 전략

#### 1. 트랜잭션 롤백

```java
@SpringBootTest
@Transactional
class IsolatedTest {
    // 각 테스트 후 자동 롤백
}
```

#### 2. 테이블 TRUNCATE

```java
@SpringBootTest
class IsolatedTest {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @BeforeEach
    void cleanUp() {
        jdbcTemplate.execute("SET REFERENTIAL_INTEGRITY FALSE");
        jdbcTemplate.execute("TRUNCATE TABLE orders");
        jdbcTemplate.execute("TRUNCATE TABLE users");
        jdbcTemplate.execute("SET REFERENTIAL_INTEGRITY TRUE");
    }
}
```

```python
@pytest.fixture(autouse=True)
def clean_database(db_session):
    """각 테스트 전후 데이터베이스 정리"""
    yield
    # 테스트 후 정리
    db_session.execute(text("TRUNCATE TABLE orders CASCADE"))
    db_session.execute(text("TRUNCATE TABLE users CASCADE"))
    db_session.commit()
```

#### 3. 테스트별 독립 스키마

```java
@SpringBootTest
class SchemaIsolatedTest {

    @Autowired
    private DataSource dataSource;

    private String schemaName;

    @BeforeEach
    void setUp() {
        schemaName = "test_" + UUID.randomUUID().toString().replace("-", "");

        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("CREATE SCHEMA " + schemaName);
            conn.createStatement().execute("SET SCHEMA " + schemaName);
            // 테이블 생성...
        }
    }

    @AfterEach
    void tearDown() {
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().execute("DROP SCHEMA " + schemaName + " CASCADE");
        }
    }
}
```

#### 4. 컨테이너 재생성

```java
// 느리지만 완벽한 격리
@Testcontainers
class FullyIsolatedTest {

    @Container
    PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    // 각 테스트 클래스마다 새로운 컨테이너
}
```

### 병렬 테스트 실행

```java
// junit-platform.properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent
```

```java
@SpringBootTest
@Execution(ExecutionMode.CONCURRENT)  // 병렬 실행
class ParallelTest {

    @Test
    void test1() {
        // 독립적인 데이터 사용
        User user = createUniqueUser();
    }

    @Test
    void test2() {
        // 독립적인 데이터 사용
        User user = createUniqueUser();
    }

    private User createUniqueUser() {
        String uniqueEmail = "user-" + UUID.randomUUID() + "@example.com";
        return new User("Test User", uniqueEmail);
    }
}
```

### 격리 체크리스트

```markdown
## 테스트 격리 체크리스트

### 데이터베이스
- [ ] 각 테스트 시작 시 깨끗한 상태인가?
- [ ] 외래 키 제약조건을 고려한 정리 순서인가?
- [ ] 시퀀스/Auto Increment 값이 리셋되는가?

### 외부 서비스
- [ ] Mock/Stub이 각 테스트마다 리셋되는가?
- [ ] WireMock 스텁이 테스트 간 충돌하지 않는가?

### 상태
- [ ] static 변수를 공유하지 않는가?
- [ ] 싱글톤 빈의 상태를 변경하지 않는가?
- [ ] 파일 시스템을 사용한다면 정리되는가?

### 시간/랜덤
- [ ] 시간에 의존하는 테스트가 고정된 시간을 사용하는가?
- [ ] 랜덤값이 필요한 경우 시드가 고정되어 있는가?
```

---

## 참고 자료

- Testcontainers Documentation - https://testcontainers.com/
- Spring Boot Testing Guide - https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing
- WireMock Documentation - http://wiremock.org/docs/
- Flyway Documentation - https://flywaydb.org/documentation/
- Docker Best Practices - https://www.docker.com/blog/testcontainers-best-practices/

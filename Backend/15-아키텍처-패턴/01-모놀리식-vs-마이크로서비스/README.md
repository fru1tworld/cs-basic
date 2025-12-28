# 모놀리식 vs 마이크로서비스 아키텍처

## 목차
1. [개요](#개요)
2. [모놀리식 아키텍처](#모놀리식-아키텍처)
3. [마이크로서비스 아키텍처](#마이크로서비스-아키텍처)
4. [장단점 비교](#장단점-비교)
5. [분해 전략](#분해-전략)
6. [통신 패턴](#통신-패턴)
7. [마이크로서비스 안티패턴](#마이크로서비스-안티패턴)
8. [마이그레이션 전략](#마이그레이션-전략)
9. [실무 적용 가이드](#실무-적용-가이드)
---

## 개요

소프트웨어 아키텍처 선택은 시스템의 확장성, 유지보수성, 개발 생산성에 직접적인 영향을 미친다. 모놀리식과 마이크로서비스는 각각의 장단점이 있으며, 프로젝트의 특성과 조직의 역량에 따라 적절한 선택이 필요하다.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        아키텍처 선택 스펙트럼                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Monolithic    →    Modular     →    SOA    →    Microservices     │
│  (단일 배포)       Monolith         (서비스      (독립 배포)           │
│                  (모듈화된 단일)    지향)                              │
│                                                                     │
│  ◀────────────────────────────────────────────────────────────────▶ │
│  단순성                                                  복잡성      │
│  빠른 초기 개발                                    높은 확장성        │
│  낮은 운영 복잡도                              독립적 배포/확장        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 모놀리식 아키텍처

### 정의

모놀리식 아키텍처는 **하나의 코드베이스**에서 모든 비즈니스 기능을 수행하는 전통적인 소프트웨어 개발 모델이다. 모든 컴포넌트가 단일 프로세스 내에서 실행되며, 하나의 배포 단위로 관리된다.

### 구조

```
┌─────────────────────────────────────────────────────────────┐
│                    Monolithic Application                    │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Presentation Layer                  │   │
│  │    (Web UI, REST API, GraphQL, WebSocket)           │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Business Layer                      │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │   │
│  │  │   User   │  │  Order   │  │     Payment      │   │   │
│  │  │ Service  │  │ Service  │  │     Service      │   │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │   │
│  │  │ Product  │  │ Inventory│  │   Notification   │   │   │
│  │  │ Service  │  │ Service  │  │     Service      │   │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Data Access Layer                   │   │
│  │              (Single Database Connection)            │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
└─────────────────────────────────────────────────────────────┘
                             │
                   ┌─────────────────┐
                   │    Database     │
                   │   (Single DB)   │
                   └─────────────────┘
```

### 코드 예제: Spring Boot 모놀리식 구조

```java
// 단일 애플리케이션 구조
// src/main/java/com/example/monolith/
├── MonolithApplication.java
├── controller/
│   ├── UserController.java
│   ├── OrderController.java
│   └── ProductController.java
├── service/
│   ├── UserService.java
│   ├── OrderService.java
│   └── ProductService.java
├── repository/
│   ├── UserRepository.java
│   ├── OrderRepository.java
│   └── ProductRepository.java
└── entity/
    ├── User.java
    ├── Order.java
    └── Product.java
```

```java
// OrderService.java - 모놀리식에서의 서비스 간 직접 호출
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final UserService userService;        // 직접 의존성
    private final ProductService productService;  // 직접 의존성
    private final InventoryService inventoryService;
    private final PaymentService paymentService;

    public Order createOrder(CreateOrderRequest request) {
        // 1. 사용자 검증 - 직접 메서드 호출
        User user = userService.findById(request.getUserId());
        if (!user.isActive()) {
            throw new UserNotActiveException();
        }

        // 2. 상품 검증 및 재고 확인 - 동일 트랜잭션
        Product product = productService.findById(request.getProductId());
        inventoryService.checkAndReserve(product.getId(), request.getQuantity());

        // 3. 결제 처리
        PaymentResult payment = paymentService.process(
            user.getId(),
            product.getPrice() * request.getQuantity()
        );

        // 4. 주문 생성 - 모든 것이 하나의 트랜잭션
        Order order = Order.builder()
            .user(user)
            .product(product)
            .quantity(request.getQuantity())
            .paymentId(payment.getId())
            .status(OrderStatus.CREATED)
            .build();

        return orderRepository.save(order);
    }
}
```

### 특징

| 특징 | 설명 |
|------|------|
| **단일 배포** | 전체 애플리케이션이 하나의 단위로 배포 |
| **공유 데이터베이스** | 모든 모듈이 동일한 데이터베이스 사용 |
| **동기화된 트랜잭션** | ACID 트랜잭션 보장 용이 |
| **프로세스 내 통신** | 네트워크 오버헤드 없는 메서드 호출 |
| **단일 기술 스택** | 전체 애플리케이션이 동일 언어/프레임워크 사용 |

---

## 마이크로서비스 아키텍처

### 정의

마이크로서비스 아키텍처는 **독립적으로 배포 가능한 작은 서비스들의 집합**으로 애플리케이션을 구성하는 아키텍처 스타일이다. 각 서비스는 자체 비즈니스 로직과 데이터베이스를 가지며, 잘 정의된 API를 통해 통신한다.

### 구조

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Microservices Architecture                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                           ┌─────────────────┐                               │
│                           │   API Gateway   │                               │
│                           │  (Kong/Nginx)   │                               │
│                           └────────┬────────┘                               │
│                                    │                                         │
│         ┌──────────────────────────┼──────────────────────────┐             │
│         │                          │                          │              │
│  ┌──────┴──────┐           ┌──────┴──────┐           ┌───────┴─────┐       │
│  │    User     │           │    Order    │           │   Product   │       │
│  │   Service   │           │   Service   │           │   Service   │       │
│  │  (Node.js)  │◄─────────►│   (Java)    │◄─────────►│  (Python)   │       │
│  └──────┬──────┘           └──────┬──────┘           └──────┬──────┘       │
│         │                          │                         │              │
│  ┌──────┴──────┐           ┌──────┴──────┐           ┌──────┴──────┐       │
│  │  User DB    │           │  Order DB   │           │ Product DB  │       │
│  │ (PostgreSQL)│           │   (MySQL)   │           │  (MongoDB)  │       │
│  └─────────────┘           └─────────────┘           └─────────────┘       │
│                                                                              │
│  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐       │
│  │  Payment    │           │  Inventory  │           │Notification │       │
│  │  Service    │           │   Service   │           │   Service   │       │
│  │   (Go)      │           │   (Rust)    │           │  (Node.js)  │       │
│  └──────┬──────┘           └──────┬──────┘           └──────┬──────┘       │
│         │                          │                         │              │
│  ┌──────┴──────┐           ┌──────┴──────┐           ┌──────┴──────┐       │
│  │ Payment DB  │           │ Inventory   │           │   Redis     │       │
│  │  (MySQL)    │           │   (Redis)   │           │  (Cache)    │       │
│  └─────────────┘           └─────────────┘           └─────────────┘       │
│                                                                              │
│                     ┌──────────────────────────┐                            │
│                     │    Message Broker        │                            │
│                     │   (Kafka / RabbitMQ)     │                            │
│                     └──────────────────────────┘                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 코드 예제: 마이크로서비스 간 통신

#### Order Service (Java/Spring Boot)

```java
// OrderService.java - HTTP 동기 통신
@Service
@Slf4j
public class OrderService {

    private final OrderRepository orderRepository;
    private final UserServiceClient userClient;
    private final ProductServiceClient productClient;
    private final InventoryServiceClient inventoryClient;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // 1. 외부 서비스 호출 (HTTP/REST)
        UserDto user = userClient.getUser(request.getUserId());
        ProductDto product = productClient.getProduct(request.getProductId());

        // 2. 재고 예약 (HTTP 호출 with Circuit Breaker)
        inventoryClient.reserve(product.getId(), request.getQuantity());

        // 3. 주문 생성 (로컬 트랜잭션)
        Order order = Order.builder()
            .userId(user.getId())
            .productId(product.getId())
            .quantity(request.getQuantity())
            .totalAmount(product.getPrice().multiply(
                BigDecimal.valueOf(request.getQuantity())))
            .status(OrderStatus.PENDING)
            .build();

        Order savedOrder = orderRepository.save(order);

        // 4. 이벤트 발행 (비동기)
        kafkaTemplate.send("order-events",
            new OrderCreatedEvent(savedOrder.getId(), savedOrder.getUserId()));

        return savedOrder;
    }
}
```

```java
// UserServiceClient.java - Feign Client
@FeignClient(name = "user-service", fallback = UserServiceFallback.class)
public interface UserServiceClient {

    @GetMapping("/api/users/{id}")
    UserDto getUser(@PathVariable("id") Long id);

    @GetMapping("/api/users/{id}/validate")
    boolean validateUser(@PathVariable("id") Long id);
}

// Circuit Breaker 설정
@Component
@Slf4j
public class UserServiceFallback implements UserServiceClient {

    @Override
    public UserDto getUser(Long id) {
        log.warn("User service unavailable, returning cached user for id: {}", id);
        // 캐시된 데이터 반환 또는 기본값
        return UserDto.builder()
            .id(id)
            .status(UserStatus.UNKNOWN)
            .build();
    }

    @Override
    public boolean validateUser(Long id) {
        // 장애 시 기본적으로 검증 통과 (비즈니스 요구사항에 따라)
        return true;
    }
}
```

#### Resilience4j Circuit Breaker 설정

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 5s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
        recordExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
  retry:
    instances:
      userService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
```

---

## 장단점 비교

### 비교 표

| 측면 | 모놀리식 | 마이크로서비스 |
|------|----------|----------------|
| **개발 초기 속도** | 빠름 | 느림 (인프라 설정 필요) |
| **배포 복잡도** | 낮음 (단일 배포) | 높음 (여러 서비스 조율) |
| **확장성** | 전체 애플리케이션 스케일링 | 서비스별 독립 스케일링 |
| **기술 다양성** | 단일 기술 스택 | 폴리글랏 (다중 언어/프레임워크) |
| **장애 격리** | 낮음 (하나의 버그가 전체 영향) | 높음 (서비스 단위 격리) |
| **트랜잭션 관리** | ACID 보장 용이 | Saga 패턴 필요 (복잡) |
| **테스트** | 통합 테스트 용이 | 서비스 간 계약 테스트 필요 |
| **디버깅** | 단순 (로컬 스택 트레이스) | 복잡 (분산 트레이싱 필요) |
| **팀 구조** | 기능별 팀 | 서비스별 자율 팀 |
| **운영 비용** | 낮음 | 높음 (인프라, 모니터링) |

### 상세 비교 다이어그램

```
                    모놀리식                    마이크로서비스
                    ────────                    ────────────────

개발 속도     ████████████░░ (빠름)         ████░░░░░░░░░░ (느림)
              초기 개발에 유리                인프라 설정 오버헤드

확장성        ████░░░░░░░░░░ (제한적)        ████████████░░ (유연함)
              전체 앱 스케일링                서비스별 독립 확장

배포 주기     ████░░░░░░░░░░ (느림)          ████████████░░ (빠름)
              전체 배포 필요                  독립적 배포 가능

장애 격리     ████░░░░░░░░░░ (취약)          ████████████░░ (견고함)
              연쇄 장애 가능                  서비스 단위 격리

운영 복잡도   ████████████░░ (단순)          ████░░░░░░░░░░ (복잡)
              단일 시스템 관리                분산 시스템 관리

데이터 일관성 ████████████░░ (용이)          ████░░░░░░░░░░ (어려움)
              ACID 트랜잭션                   최종 일관성
```

### 언제 무엇을 선택할까?

#### 모놀리식이 적합한 경우
- 스타트업 초기 단계 (빠른 MVP 개발)
- 소규모 팀 (5명 이하)
- 도메인이 명확하지 않은 경우
- 분산 시스템 운영 경험이 부족한 팀
- 단순한 비즈니스 로직

#### 마이크로서비스가 적합한 경우
- 대규모 조직 (여러 팀이 독립적으로 개발)
- 높은 확장성 요구
- 다양한 기술 스택 필요
- 독립적인 배포 주기 필요
- 성숙한 DevOps 문화

---

## 분해 전략

### 1. 비즈니스 역량 기반 분해 (Decompose by Business Capability)

```
┌─────────────────────────────────────────────────────────────────────┐
│                     E-Commerce 비즈니스 역량                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
│  │   고객 관리    │  │   상품 관리    │  │      주문 관리         │   │
│  │   Capability  │  │   Capability  │  │     Capability        │   │
│  ├───────────────┤  ├───────────────┤  ├───────────────────────┤   │
│  │ - 회원가입     │  │ - 상품 등록   │  │ - 주문 생성           │   │
│  │ - 로그인      │  │ - 재고 관리   │  │ - 주문 조회           │   │
│  │ - 프로필 관리  │  │ - 가격 관리   │  │ - 주문 취소           │   │
│  │ - 인증/인가   │  │ - 카테고리    │  │ - 배송 추적           │   │
│  └───────────────┘  └───────────────┘  └───────────────────────┘   │
│          │                  │                      │                │
│          ▼                  ▼                      ▼                │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
│  │ User Service  │  │Product Service│  │    Order Service      │   │
│  └───────────────┘  └───────────────┘  └───────────────────────┘   │
│                                                                      │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
│  │   결제 관리    │  │   배송 관리    │  │      알림 관리         │   │
│  │   Capability  │  │   Capability  │  │     Capability        │   │
│  ├───────────────┤  ├───────────────┤  ├───────────────────────┤   │
│  │ - 결제 처리   │  │ - 배송 생성   │  │ - 이메일 발송         │   │
│  │ - 환불 처리   │  │ - 배송 추적   │  │ - SMS 발송            │   │
│  │ - 결제 내역   │  │ - 배송사 연동 │  │ - 푸시 알림           │   │
│  └───────────────┘  └───────────────┘  └───────────────────────┘   │
│          │                  │                      │                │
│          ▼                  ▼                      ▼                │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │
│  │Payment Service│  │Shipping Svc   │  │ Notification Service  │   │
│  └───────────────┘  └───────────────┘  └───────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. 서브도메인 기반 분해 (DDD Subdomain)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Domain-Driven Decomposition                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Core Domain (핵심 도메인)                  │    │
│  │         비즈니스 차별화 요소, 가장 높은 투자 우선순위            │    │
│  │  ┌─────────────────┐    ┌──────────────────────────────┐    │    │
│  │  │   주문 처리      │    │         가격 책정             │    │    │
│  │  │ Order Processing│    │        Pricing Engine        │    │    │
│  │  └─────────────────┘    └──────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                Supporting Domain (지원 도메인)                │    │
│  │              핵심은 아니지만 비즈니스에 특화된 기능               │    │
│  │  ┌─────────────────┐    ┌──────────────────────────────┐    │    │
│  │  │   재고 관리      │    │         고객 지원             │    │    │
│  │  │    Inventory    │    │      Customer Support        │    │    │
│  │  └─────────────────┘    └──────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                 Generic Domain (일반 도메인)                  │    │
│  │            범용 솔루션, 외부 서비스 또는 오픈소스 활용            │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │    │
│  │  │   인증   │  │   결제   │  │  이메일  │  │   알림   │    │    │
│  │  │  (Auth0) │  │ (Stripe) │  │(SendGrid)│  │ (FCM)   │    │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. Strangler Fig Pattern (교살자 패턴)

점진적으로 모놀리식을 마이크로서비스로 전환하는 패턴:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Strangler Fig Pattern 진행 단계                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Phase 1: 초기 상태                                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                        Monolith                              │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │    │
│  │  │  User   │ │  Order  │ │ Product │ │ Payment │           │    │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Phase 2: API Gateway 도입 + 첫 번째 서비스 분리                      │
│                     ┌─────────────┐                                 │
│                     │ API Gateway │                                 │
│                     └──────┬──────┘                                 │
│              ┌─────────────┼─────────────┐                          │
│              ▼             ▼             ▼                          │
│  ┌───────────────┐    ┌─────────────────────────────────────┐      │
│  │ User Service  │    │           Monolith                   │      │
│  │   (신규)      │    │  ┌─────────┐ ┌─────────┐ ┌─────────┐│      │
│  └───────────────┘    │  │  Order  │ │ Product │ │ Payment ││      │
│                       │  └─────────┘ └─────────┘ └─────────┘│      │
│                       └─────────────────────────────────────┘      │
│                                                                      │
│  Phase 3: 점진적 분리                                                │
│                     ┌─────────────┐                                 │
│                     │ API Gateway │                                 │
│                     └──────┬──────┘                                 │
│         ┌──────────────────┼──────────────────┐                     │
│         ▼          ▼       ▼       ▼          ▼                     │
│  ┌───────────┐ ┌───────┐ ┌───────────────────────┐ ┌───────────┐   │
│  │   User    │ │ Order │ │      Monolith         │ │  Payment  │   │
│  │  Service  │ │Service│ │  ┌─────────────────┐  │ │  Service  │   │
│  └───────────┘ └───────┘ │  │     Product     │  │ └───────────┘   │
│                          │  └─────────────────┘  │                 │
│                          └───────────────────────┘                 │
│                                                                      │
│  Phase 4: 완전 분리 (Monolith 제거)                                  │
│                     ┌─────────────┐                                 │
│                     │ API Gateway │                                 │
│                     └──────┬──────┘                                 │
│      ┌─────────┬───────────┼───────────┬─────────┐                 │
│      ▼         ▼           ▼           ▼         ▼                 │
│  ┌───────┐ ┌───────┐ ┌─────────┐ ┌─────────┐ ┌───────┐            │
│  │ User  │ │ Order │ │ Product │ │ Payment │ │ ...   │            │
│  │Service│ │Service│ │ Service │ │ Service │ │Service│            │
│  └───────┘ └───────┘ └─────────┘ └─────────┘ └───────┘            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 코드 예제: Strangler Pattern 구현

```java
// API Gateway에서 라우팅 결정
@Component
public class ServiceRouter {

    private final FeatureToggleService featureToggle;
    private final MonolithClient monolithClient;
    private final UserServiceClient userServiceClient;

    public UserDto getUser(Long userId) {
        // Feature Toggle로 신규 서비스 라우팅 제어
        if (featureToggle.isEnabled("use-new-user-service")) {
            try {
                return userServiceClient.getUser(userId);
            } catch (ServiceUnavailableException e) {
                // Fallback to monolith
                log.warn("User service unavailable, falling back to monolith");
                return monolithClient.getUser(userId);
            }
        }
        return monolithClient.getUser(userId);
    }
}
```

---

## 통신 패턴

### 1. 동기 통신 (Synchronous)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      동기 통신 패턴                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  REST/HTTP                                                          │
│  ┌─────────────┐     HTTP Request      ┌─────────────┐             │
│  │   Order     │ ──────────────────►   │    User     │             │
│  │   Service   │ ◄──────────────────   │   Service   │             │
│  └─────────────┘     HTTP Response     └─────────────┘             │
│                                                                      │
│  gRPC (고성능 바이너리 프로토콜)                                      │
│  ┌─────────────┐     Protobuf/HTTP2    ┌─────────────┐             │
│  │   Order     │ ════════════════════  │  Inventory  │             │
│  │   Service   │                       │   Service   │             │
│  └─────────────┘                       └─────────────┘             │
│                                                                      │
│  장점: 단순함, 즉각적 응답, 디버깅 용이                               │
│  단점: 강한 결합, 연쇄 장애, 확장성 제한                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. 비동기 통신 (Asynchronous)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      비동기 통신 패턴                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Message Queue (Point-to-Point)                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │   Order     │───►│   Queue     │───►│  Payment    │             │
│  │   Service   │    │ (RabbitMQ)  │    │  Service    │             │
│  └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                      │
│  Event Bus (Publish-Subscribe)                                      │
│  ┌─────────────┐         ┌─────────────┐                           │
│  │   Order     │────────►│   Kafka     │                           │
│  │   Service   │         │   Topic     │                           │
│  └─────────────┘         └──────┬──────┘                           │
│                                 │                                   │
│          ┌──────────────────────┼──────────────────────┐           │
│          ▼                      ▼                      ▼           │
│  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐    │
│  │  Inventory  │        │Notification │        │  Analytics  │    │
│  │   Service   │        │   Service   │        │   Service   │    │
│  └─────────────┘        └─────────────┘        └─────────────┘    │
│                                                                      │
│  장점: 느슨한 결합, 장애 격리, 높은 확장성, 재시도 용이               │
│  단점: 복잡성 증가, 최종 일관성, 디버깅 어려움                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 코드 예제: Kafka 이벤트 기반 통신

```java
// 이벤트 발행 (Order Service)
@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .orderId(order.getId())
            .userId(order.getUserId())
            .productId(order.getProductId())
            .quantity(order.getQuantity())
            .totalAmount(order.getTotalAmount())
            .timestamp(Instant.now())
            .build();

        kafkaTemplate.send("order-events", order.getId().toString(), event);
    }
}

// 이벤트 소비 (Inventory Service)
@Service
@Slf4j
public class InventoryEventConsumer {

    private final InventoryService inventoryService;

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void handleOrderEvent(OrderCreatedEvent event) {
        log.info("Received order event: {}", event.getOrderId());

        try {
            inventoryService.reserveStock(
                event.getProductId(),
                event.getQuantity()
            );
            log.info("Stock reserved for order: {}", event.getOrderId());
        } catch (InsufficientStockException e) {
            log.error("Failed to reserve stock for order: {}", event.getOrderId());
            // 보상 이벤트 발행
            publishStockReservationFailed(event);
        }
    }
}
```

### 3. Saga Pattern (분산 트랜잭션)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Choreography-based Saga                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  성공 시나리오:                                                       │
│                                                                      │
│  Order      Inventory     Payment      Shipping     Notification    │
│  Service    Service       Service      Service      Service         │
│     │           │            │            │             │           │
│     │ OrderCreated         │            │             │           │
│     │──────────►│           │            │             │           │
│     │           │ StockReserved         │             │           │
│     │           │───────────►│           │             │           │
│     │           │            │ PaymentProcessed        │           │
│     │           │            │───────────►│            │           │
│     │           │            │            │ ShippingCreated        │
│     │           │            │            │────────────►│          │
│     │           │            │            │             │ NotifySent│
│     │◄──────────────────────────────────────────────────│          │
│                                                                      │
│  실패 시나리오 (보상 트랜잭션):                                        │
│                                                                      │
│  Order      Inventory     Payment                                   │
│  Service    Service       Service                                   │
│     │           │            │                                      │
│     │ OrderCreated         │                                      │
│     │──────────►│           │                                      │
│     │           │ StockReserved                                    │
│     │           │───────────►│                                      │
│     │           │            │ PaymentFailed (잔액 부족)             │
│     │           │◄───────────│                                      │
│     │           │ StockReleased (보상)                              │
│     │◄──────────│                                                   │
│     │ OrderCancelled (보상)                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```java
// Saga Orchestrator Pattern 구현
@Service
@Slf4j
public class OrderSagaOrchestrator {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final ShippingClient shippingClient;

    @Transactional
    public OrderResult processOrder(CreateOrderCommand command) {
        Order order = createOrder(command);

        try {
            // Step 1: 재고 예약
            ReservationResult reservation = inventoryClient.reserve(
                command.getProductId(),
                command.getQuantity()
            );
            order.setReservationId(reservation.getId());

            // Step 2: 결제 처리
            PaymentResult payment = paymentClient.process(
                command.getUserId(),
                order.getTotalAmount()
            );
            order.setPaymentId(payment.getId());

            // Step 3: 배송 생성
            ShippingResult shipping = shippingClient.create(
                order.getId(),
                command.getShippingAddress()
            );
            order.setShippingId(shipping.getId());

            order.setStatus(OrderStatus.CONFIRMED);
            return OrderResult.success(order);

        } catch (InventoryException e) {
            // 재고 예약 실패 - 보상 불필요
            order.setStatus(OrderStatus.FAILED);
            return OrderResult.failure("Insufficient stock");

        } catch (PaymentException e) {
            // 결제 실패 - 재고 예약 취소
            compensateInventory(order.getReservationId());
            order.setStatus(OrderStatus.FAILED);
            return OrderResult.failure("Payment failed");

        } catch (ShippingException e) {
            // 배송 생성 실패 - 결제 취소 + 재고 예약 취소
            compensatePayment(order.getPaymentId());
            compensateInventory(order.getReservationId());
            order.setStatus(OrderStatus.FAILED);
            return OrderResult.failure("Shipping creation failed");
        }
    }

    private void compensateInventory(String reservationId) {
        try {
            inventoryClient.cancelReservation(reservationId);
        } catch (Exception e) {
            log.error("Failed to compensate inventory: {}", reservationId, e);
            // 보상 실패 시 재시도 큐에 추가
            compensationRetryQueue.add(new InventoryCompensation(reservationId));
        }
    }

    private void compensatePayment(String paymentId) {
        try {
            paymentClient.refund(paymentId);
        } catch (Exception e) {
            log.error("Failed to compensate payment: {}", paymentId, e);
            compensationRetryQueue.add(new PaymentCompensation(paymentId));
        }
    }
}
```

---

## 마이크로서비스 안티패턴

### 1. 분산 모놀리스 (Distributed Monolith)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    안티패턴: 분산 모놀리스                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  특징: 서비스가 분리되어 있지만 강하게 결합되어 있음                    │
│                                                                      │
│  ┌─────────────┐     동기 호출      ┌─────────────┐                 │
│  │   Order     │◄──────────────────►│    User     │                 │
│  │   Service   │     (강한 결합)     │   Service   │                 │
│  └──────┬──────┘                    └──────┬──────┘                 │
│         │                                   │                        │
│         │      동시 배포 필요!               │                        │
│         │◄─────────────────────────────────►│                        │
│                                                                      │
│  문제점:                                                             │
│  - 한 서비스 변경 시 다른 서비스도 함께 배포 필요                       │
│  - 장애 전파 (한 서비스 장애가 연쇄적으로 전체 영향)                    │
│  - 결국 모놀리스보다 복잡한 시스템이 됨                                │
│                                                                      │
│  해결책:                                                             │
│  - 비동기 이벤트 기반 통신 도입                                       │
│  - API 버저닝과 하위 호환성 유지                                      │
│  - Circuit Breaker 패턴 적용                                        │
│  - 각 서비스의 독립적인 데이터 저장소                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. 공유 데이터베이스 (Shared Database)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    안티패턴: 공유 데이터베이스                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│       ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│       │    Order    │  │    User     │  │   Product   │            │
│       │   Service   │  │   Service   │  │   Service   │            │
│       └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
│              │                │                │                    │
│              └────────────────┼────────────────┘                    │
│                               │                                     │
│                      ┌────────┴────────┐                           │
│                      │  Shared Database│  ◄── 단일 장애점           │
│                      │    (Monolith)   │      스키마 변경 영향 전체  │
│                      └─────────────────┘      확장성 제한           │
│                                                                      │
│  문제점:                                                             │
│  - 스키마 변경이 모든 서비스에 영향                                   │
│  - 단일 장애점 (Single Point of Failure)                            │
│  - 서비스 간 암묵적 결합                                             │
│  - 독립적 확장 불가                                                  │
│                                                                      │
│  해결책: Database per Service                                       │
│       ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│       │    Order    │  │    User     │  │   Product   │            │
│       │   Service   │  │   Service   │  │   Service   │            │
│       └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │
│              │                │                │                    │
│       ┌──────┴──────┐ ┌──────┴──────┐ ┌───────┴─────┐             │
│       │  Order DB   │ │  User DB    │ │ Product DB  │             │
│       └─────────────┘ └─────────────┘ └─────────────┘             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. 과도한 서비스 분해 (Nano-services / Too Fine-Grained)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    안티패턴: 나노서비스                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  극단적 분해 예시 (피해야 함):                                        │
│                                                                      │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │
│  │Email    │ │Email    │ │Email    │ │Email    │ │Email    │      │
│  │Validator│ │Template │ │Sender   │ │Logger   │ │Queue    │      │
│  │Service  │ │Service  │ │Service  │ │Service  │ │Service  │      │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘      │
│       │           │           │           │           │            │
│       └───────────┴───────────┴───────────┴───────────┘            │
│                          너무 많은 네트워크 호출!                     │
│                                                                      │
│  문제점:                                                             │
│  - 과도한 네트워크 지연                                              │
│  - 운영 복잡도 증가                                                  │
│  - 분산 트랜잭션 관리 어려움                                          │
│  - 인프라 비용 증가                                                  │
│                                                                      │
│  적절한 분해 예시:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Notification Service                      │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │    │
│  │  │ Validate │  │ Template │  │  Send    │  │  Logger  │    │    │
│  │  │ Module   │  │ Module   │  │ Module   │  │  Module  │    │    │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. 수다스러운 서비스 (Chatty Services)

```java
// 안티패턴: 수다스러운 서비스 통신
public class OrderService {

    public OrderDetails getOrderDetails(Long orderId) {
        Order order = orderRepository.findById(orderId);

        // 문제: 너무 많은 개별 API 호출!
        UserDto user = userClient.getUser(order.getUserId());           // 호출 1
        UserAddress address = userClient.getAddress(order.getUserId()); // 호출 2
        UserPreferences prefs = userClient.getPreferences(order.getUserId()); // 호출 3

        ProductDto product = productClient.getProduct(order.getProductId()); // 호출 4
        ProductReviews reviews = productClient.getReviews(order.getProductId()); // 호출 5
        ProductStock stock = inventoryClient.getStock(order.getProductId()); // 호출 6

        ShippingStatus shipping = shippingClient.getStatus(order.getShippingId()); // 호출 7

        // 총 7번의 네트워크 호출 - 지연 시간 누적!
        return buildOrderDetails(order, user, address, prefs, product, reviews, stock, shipping);
    }
}

// 해결책 1: 벌크 API 설계
public class OptimizedOrderService {

    public OrderDetails getOrderDetails(Long orderId) {
        Order order = orderRepository.findById(orderId);

        // 벌크 조회 API 사용
        UserContext userContext = userClient.getUserContext(order.getUserId());
        // 한 번의 호출로 user, address, preferences 모두 조회

        ProductContext productContext = productClient.getProductContext(order.getProductId());
        // 한 번의 호출로 product, reviews, stock 모두 조회

        ShippingStatus shipping = shippingClient.getStatus(order.getShippingId());

        // 총 3번의 네트워크 호출로 감소
        return buildOrderDetails(order, userContext, productContext, shipping);
    }
}

// 해결책 2: API Composition 패턴 (BFF 또는 API Gateway)
@RestController
public class OrderBffController {

    @GetMapping("/orders/{orderId}/details")
    public Mono<OrderDetailsResponse> getOrderDetails(@PathVariable Long orderId) {
        return orderClient.getOrder(orderId)
            .flatMap(order -> {
                // 병렬로 여러 서비스 호출
                Mono<UserContext> userMono = userClient.getUserContext(order.getUserId());
                Mono<ProductContext> productMono = productClient.getProductContext(order.getProductId());
                Mono<ShippingStatus> shippingMono = shippingClient.getStatus(order.getShippingId());

                return Mono.zip(userMono, productMono, shippingMono)
                    .map(tuple -> buildResponse(order, tuple.getT1(), tuple.getT2(), tuple.getT3()));
            });
    }
}
```

### 5. 동기 호출 체인 (Synchronous Call Chain)

```
┌─────────────────────────────────────────────────────────────────────┐
│              안티패턴: 동기 호출 체인 (Death by a Thousand Cuts)       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Client → Gateway → Order → User → Auth → Permission → DB           │
│                       │                                             │
│                       └──► Payment → Account → Bank API             │
│                       │                                             │
│                       └──► Inventory → Warehouse → Supplier API     │
│                                                                      │
│  문제점:                                                             │
│  - 전체 지연 = 모든 호출 지연의 합                                    │
│  - 어느 하나라도 실패하면 전체 실패                                   │
│  - 타임아웃 설정 어려움                                              │
│                                                                      │
│  총 지연 시간: 50ms + 30ms + 20ms + 100ms + 200ms = 400ms+          │
│                                                                      │
│  해결책:                                                             │
│  1. 비동기 메시징으로 전환                                           │
│  2. 호출 체인 깊이 제한 (최대 2-3 홉)                                │
│  3. 캐싱 적극 활용                                                   │
│  4. Circuit Breaker + Fallback                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 안티패턴 체크리스트

| 안티패턴 | 증상 | 해결책 |
|---------|------|--------|
| **분산 모놀리스** | 서비스 변경 시 다른 서비스도 배포 필요 | 이벤트 기반 통신, API 버저닝 |
| **공유 데이터베이스** | 스키마 변경이 전체 영향 | Database per Service |
| **나노서비스** | 서비스가 너무 작고 많음 | 적절한 서비스 경계 재설정 |
| **수다스러운 서비스** | 많은 API 호출, 높은 지연 | 벌크 API, 캐싱 |
| **동기 호출 체인** | 깊은 호출 체인, 연쇄 장애 | 비동기 통신, 깊이 제한 |
| **스파게티 통합** | 서비스 간 복잡한 의존관계 | API Gateway, 이벤트 버스 |

---

## 마이그레이션 전략

### 단계별 마이그레이션 접근법

```
Phase 1: 분석 및 계획 (2-4주)
├── 현재 모놀리스 코드베이스 분석
├── 도메인 경계 식별 (Event Storming)
├── 의존성 매핑
├── 분해 우선순위 결정
└── 마이그레이션 로드맵 수립

Phase 2: 인프라 준비 (4-8주)
├── API Gateway 도입
├── Service Registry (Eureka, Consul)
├── 메시지 브로커 (Kafka, RabbitMQ)
├── 컨테이너 오케스트레이션 (Kubernetes)
├── CI/CD 파이프라인 구축
└── 모니터링/로깅 인프라 (ELK, Prometheus)

Phase 3: 첫 번째 서비스 분리 (4-6주)
├── 가장 독립적인 도메인 선택
├── Anti-Corruption Layer 구현
├── 데이터 마이그레이션
├── 트래픽 점진적 전환
└── 롤백 계획 준비

Phase 4: 반복적 분해 (지속)
├── 다음 서비스 분리
├── 통합 테스트 강화
├── 모니터링 지표 확인
└── 필요시 경계 조정
```

---

## 실무 적용 가이드

### 서비스 분해 기준

1. **Single Responsibility**: 하나의 서비스는 하나의 비즈니스 역량에 집중
2. **Autonomous**: 독립적으로 개발, 테스트, 배포 가능
3. **Loosely Coupled**: 다른 서비스에 대한 최소한의 의존성
4. **Highly Cohesive**: 관련된 기능들이 함께 있음

### 팀 구조 (Conway's Law)

```
"시스템의 설계는 그것을 만드는 조직의 커뮤니케이션 구조를 반영한다"
                                                    - Melvin Conway

┌─────────────────────────────────────────────────────────────────────┐
│                    조직 구조와 시스템 구조 매핑                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │  Order Team │  │  User Team  │  │Payment Team │                 │
│  │  (5 members)│  │  (4 members)│  │  (3 members)│                 │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                 │
│         │                │                │                         │
│         ▼                ▼                ▼                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │   Order     │  │    User     │  │   Payment   │                 │
│  │   Service   │  │   Service   │  │   Service   │                 │
│  └─────────────┘  └─────────────┘  └─────────────┘                 │
│                                                                      │
│  "역 콘웨이 법칙": 원하는 시스템 구조에 맞게 팀 구조를 조정             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [Microservices vs Monolith - Atlassian](https://www.atlassian.com/microservices/microservices-architecture/microservices-vs-monolith)
- [Monolithic vs Microservices - AWS](https://aws.amazon.com/compare/the-difference-between-monolithic-and-microservices-architecture/)
- [Decompose by Business Capability - microservices.io](https://microservices.io/patterns/decomposition/decompose-by-business-capability.html)
- [Microservices Patterns - O'Reilly](https://www.oreilly.com/library/view/microservices-patterns/9781617294549/)
- [10 Common Microservices Anti-Patterns - DesignGurus](https://www.designgurus.io/blog/10-common-microservices-anti-patterns)
- Sam Newman, "Building Microservices" 2nd Edition, O'Reilly
- Chris Richardson, "Microservices Patterns", Manning

---

*마지막 업데이트: 2025년 1월*

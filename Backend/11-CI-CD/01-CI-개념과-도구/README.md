# CI (Continuous Integration) 개념과 도구

## 목차
1. [CI (Continuous Integration) 개념](#1-ci-continuous-integration-개념)
2. [CI 파이프라인 설계](#2-ci-파이프라인-설계)
3. [빌드 자동화](#3-빌드-자동화)
4. [테스트 자동화](#4-테스트-자동화)
5. [정적 분석 도구](#5-정적-분석-도구)
---

## 1. CI (Continuous Integration) 개념

### 1.1 CI란 무엇인가?

**CI (Continuous Integration, 지속적 통합)** 는 개발자들이 코드 변경 사항을 공유 저장소에 자주(하루에 여러 번) 통합하는 소프트웨어 개발 방식이다. 각 통합은 자동화된 빌드와 테스트를 통해 검증되어, 통합 문제를 조기에 발견하고 해결할 수 있게 한다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    CI (Continuous Integration) 흐름              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Developer A ─┐                                                 │
│                │      ┌──────────┐    ┌──────────┐              │
│   Developer B ─┼─────▶│   Git    │───▶│    CI    │──▶ 피드백    │
│                │      │  Server  │    │  Server  │              │
│   Developer C ─┘      └──────────┘    └──────────┘              │
│                            │               │                     │
│                         Push           자동화된                  │
│                                     빌드/테스트                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 CI의 핵심 원칙

#### Martin Fowler의 CI 핵심 원칙

| 원칙 | 설명 |
|------|------|
| **단일 소스 저장소 유지** | 모든 소스 코드는 버전 관리 시스템에서 관리 |
| **빌드 자동화** | 한 번의 명령으로 전체 시스템 빌드 가능 |
| **자체 테스트 빌드** | 빌드 시 자동으로 테스트 실행 |
| **매일 메인 브랜치에 커밋** | 최소 하루에 한 번 통합 |
| **모든 커밋은 통합 머신에서 빌드** | 개발자 로컬이 아닌 CI 서버에서 검증 |
| **빌드 신속성 유지** | 10분 이내 빌드 완료 권장 |
| **프로덕션 환경 복제본에서 테스트** | 실제 환경과 유사한 환경에서 테스트 |
| **최신 실행 파일 쉽게 접근** | 누구나 최신 빌드 결과물 확인 가능 |
| **상황 가시성 확보** | 모든 팀원이 CI 상태 확인 가능 |
| **배포 자동화** | 자동화된 배포 프로세스 구축 |

### 1.3 CI의 이점

```
┌─────────────────────────────────────────────────────────────────┐
│                       CI 도입 전 vs 도입 후                       │
├────────────────────────────┬────────────────────────────────────┤
│         도입 전             │              도입 후               │
├────────────────────────────┼────────────────────────────────────┤
│ • 통합 지옥 (Integration    │ • 지속적이고 작은 단위의 통합        │
│   Hell) 발생               │                                     │
│ • 버그 발견이 늦음          │ • 버그 조기 발견                     │
│ • 수동 테스트에 의존        │ • 자동화된 테스트로 품질 보장         │
│ • 배포 준비에 많은 시간     │ • 언제든 배포 가능한 상태 유지        │
│ • 팀 간 충돌 빈번          │ • 충돌 조기 감지 및 해결              │
│ • 피드백 루프가 길다        │ • 빠른 피드백 (몇 분 내)             │
└────────────────────────────┴────────────────────────────────────┘
```

### 1.4 CI vs CD vs CD

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          CI/CD 파이프라인 스펙트럼                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │
│  │        CI        │  │        CD        │  │         CD           │   │
│  │  (Continuous     │  │  (Continuous     │  │   (Continuous        │   │
│  │   Integration)   │  │   Delivery)      │  │    Deployment)       │   │
│  ├──────────────────┤  ├──────────────────┤  ├──────────────────────┤   │
│  │ • 코드 통합      │  │ • 자동화된       │  │ • 완전 자동화된      │   │
│  │ • 빌드 자동화    │  │   릴리스 준비    │  │   프로덕션 배포      │   │
│  │ • 테스트 실행    │  │ • 수동 배포 승인 │  │ • 수동 개입 없음     │   │
│  │ • 코드 품질 검사 │  │ • 스테이징 배포  │  │ • 모든 단계 자동화   │   │
│  └────────┬─────────┘  └────────┬─────────┘  └──────────┬───────────┘   │
│           │                     │                       │                │
│           └─────────────────────┴───────────────────────┘                │
│                                                                          │
│  <────────────────── 자동화 수준 증가 ─────────────────────>             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. CI 파이프라인 설계

### 2.1 파이프라인 아키텍처

#### 기본 파이프라인 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CI 파이프라인 단계                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐        │
│  │ Source │──▶│ Build  │──▶│  Test  │──▶│Analyze │──▶│Artifact│        │
│  │  Code  │   │        │   │        │   │        │   │ Store  │        │
│  └────────┘   └────────┘   └────────┘   └────────┘   └────────┘        │
│      │            │            │            │            │              │
│   • Git       • 컴파일      • Unit       • 정적       • Docker         │
│   • Hook      • 의존성      • Integration  분석      • Registry        │
│   • Trigger     설치       • E2E        • 보안       • Nexus          │
│               • 패키징                    스캔       • S3             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 모듈식 파이프라인 설계

현대적인 CI 파이프라인은 **모듈식 아키텍처**를 채택하여 각 단계를 독립적으로 관리하고 재사용할 수 있다.

```yaml
# GitHub Actions 예시: 모듈식 파이프라인
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # Stage 1: 빌드
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-path: ${{ steps.build.outputs.path }}
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - name: Install dependencies
        run: npm ci
      - name: Build
        id: build
        run: |
          npm run build
          echo "path=dist" >> $GITHUB_OUTPUT
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: dist/

  # Stage 2: 테스트 (병렬 실행)
  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-type: [unit, integration]
    steps:
      - uses: actions/checkout@v4
      - name: Run ${{ matrix.test-type }} tests
        run: npm run test:${{ matrix.test-type }}

  # Stage 3: 코드 품질 분석
  analyze:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: SonarQube Scan
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

  # Stage 4: 보안 스캔
  security:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Snyk Security Scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

### 2.3 파이프라인 설계 원칙

#### 10분 빌드 원칙

Martin Fowler가 제안한 **10분 빌드 가이드라인**은 대부분의 현대 프로젝트가 10분 이내에 빌드를 완료해야 함을 권장한다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    빌드 시간 최적화 전략                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 병렬 실행                                                    │
│     ┌──────┐                                                    │
│     │ Test │───┐                                                │
│     │ Unit │   │                                                │
│     └──────┘   │   ┌──────────┐                                 │
│                ├──▶│ Complete │                                 │
│     ┌──────┐   │   └──────────┘                                 │
│     │ Test │───┘                                                │
│     │ E2E  │                                                    │
│     └──────┘                                                    │
│                                                                  │
│  2. 캐싱 활용                                                    │
│     • 의존성 캐싱 (node_modules, .m2, .gradle)                  │
│     • Docker 레이어 캐싱                                        │
│     • 빌드 결과물 캐싱                                          │
│                                                                  │
│  3. 증분 빌드                                                    │
│     • 변경된 파일만 재컴파일                                     │
│     • 영향받는 테스트만 실행                                     │
│                                                                  │
│  4. 테스트 최적화                                                │
│     • 테스트 분할 (Test Sharding)                               │
│     • 실패 우선 실행 (Fail Fast)                                │
│     • 불안정 테스트 격리                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 브랜치 전략과 CI

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     브랜치별 CI 파이프라인 전략                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Feature Branch          Develop Branch         Main/Production          │
│  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐          │
│  │ • Unit Test │        │ • Full Test │        │ • Full Test │          │
│  │ • Lint      │   ──▶  │ • Security  │   ──▶  │ • E2E Test  │          │
│  │ • Build     │  Merge │ • Analysis  │  Merge │ • Deploy    │          │
│  │             │        │ • Staging   │        │   Prod      │          │
│  └─────────────┘        └─────────────┘        └─────────────┘          │
│                                                                          │
│  실행 시간: ~3분         실행 시간: ~10분       실행 시간: ~15분          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 빌드 자동화

### 3.1 빌드 도구 비교

| 도구 | 언어/플랫폼 | 특징 |
|------|------------|------|
| **Gradle** | Java/Kotlin/Android | 선언적 DSL, 증분 빌드, 캐싱 |
| **Maven** | Java | XML 기반, 컨벤션 우선, 풍부한 플러그인 |
| **npm/yarn/pnpm** | JavaScript/Node.js | 패키지 관리 + 스크립트 실행 |
| **Make** | C/C++/범용 | 전통적, 범용성 |
| **Bazel** | 다중 언어 | 대규모 모노레포, 분산 빌드 |
| **Cargo** | Rust | 패키지 관리 + 빌드 통합 |

### 3.2 Gradle 빌드 자동화 예시

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
    id("org.springframework.boot") version "3.2.1"
    id("io.spring.dependency-management") version "1.1.4"
    id("org.sonarqube") version "4.4.1.3373"
    id("jacoco")
}

group = "com.example"
version = "1.0.0"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

// 테스트 설정
tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

// 코드 커버리지 리포트
tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required = true
        html.required = true
    }
}

// SonarQube 설정
sonarqube {
    properties {
        property("sonar.projectKey", "my-project")
        property("sonar.host.url", System.getenv("SONAR_HOST_URL") ?: "http://localhost:9000")
        property("sonar.login", System.getenv("SONAR_TOKEN") ?: "")
        property("sonar.coverage.jacoco.xmlReportPaths", "${layout.buildDirectory}/reports/jacoco/test/jacocoTestReport.xml")
    }
}

// 커스텀 빌드 태스크
tasks.register("ci") {
    group = "CI/CD"
    description = "Run all CI checks"
    dependsOn("clean", "build", "test", "jacocoTestReport", "sonarqube")
}
```

### 3.3 Docker 기반 빌드

```dockerfile
# Multi-stage build for Java application
FROM gradle:8.5-jdk21 AS build

WORKDIR /app

# 의존성 캐싱을 위해 먼저 복사
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle

# 의존성 다운로드 (캐시 활용)
RUN gradle dependencies --no-daemon

# 소스 코드 복사 및 빌드
COPY src ./src
RUN gradle build --no-daemon -x test

# Runtime stage
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

# 보안: non-root 사용자로 실행
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

COPY --from=build /app/build/libs/*.jar app.jar

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 3.4 빌드 캐싱 전략

```yaml
# GitHub Actions에서의 효과적인 캐싱
name: Build with Caching

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Gradle 캐싱
      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: gradle-${{ runner.os }}-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}
          restore-keys: |
            gradle-${{ runner.os }}-

      # Docker 레이어 캐싱
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: myapp:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 4. 테스트 자동화

### 4.1 테스트 피라미드

```
┌─────────────────────────────────────────────────────────────────┐
│                       테스트 피라미드                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                         /\                                       │
│                        /  \                                      │
│                       / E2E\        느림 / 비용 높음 / 적은 수    │
│                      /  Test \                                   │
│                     /──────────\                                 │
│                    / Integration\                                │
│                   /     Test     \                               │
│                  /────────────────\                              │
│                 /    Unit Test     \    빠름 / 비용 낮음 / 많은 수│
│                /____________________\                            │
│                                                                  │
│  권장 비율:                                                      │
│  • Unit Test: 70%                                                │
│  • Integration Test: 20%                                         │
│  • E2E Test: 10%                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 테스트 유형별 자동화

#### Unit Test (단위 테스트)

```java
// JUnit 5 + Mockito 예시
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentService paymentService;

    @InjectMocks
    private OrderService orderService;

    @Test
    @DisplayName("주문 생성 시 재고가 충분하면 성공한다")
    void createOrder_WithSufficientStock_ShouldSucceed() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(
            "product-1",
            5,
            BigDecimal.valueOf(10000)
        );

        when(orderRepository.save(any(Order.class)))
            .thenReturn(Order.of(request));
        when(paymentService.process(any()))
            .thenReturn(PaymentResult.success());

        // When
        OrderResponse response = orderService.createOrder(request);

        // Then
        assertThat(response).isNotNull();
        assertThat(response.getStatus()).isEqualTo(OrderStatus.CREATED);
        verify(orderRepository).save(any(Order.class));
        verify(paymentService).process(any());
    }

    @Test
    @DisplayName("재고 부족 시 예외가 발생한다")
    void createOrder_WithInsufficientStock_ShouldThrowException() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(
            "product-1",
            1000,
            BigDecimal.valueOf(10000)
        );

        when(orderRepository.checkStock(anyString(), anyInt()))
            .thenReturn(false);

        // When & Then
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(InsufficientStockException.class)
            .hasMessage("재고가 부족합니다");
    }
}
```

#### Integration Test (통합 테스트)

```java
// Spring Boot 통합 테스트
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("POST /api/orders - 주문 생성 통합 테스트")
    void createOrder_IntegrationTest() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(
            "product-1",
            2,
            BigDecimal.valueOf(50000)
        );

        // When
        ResponseEntity<OrderResponse> response = restTemplate.postForEntity(
            "/api/orders",
            request,
            OrderResponse.class
        );

        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody()).isNotNull();

        // DB 검증
        Order savedOrder = orderRepository.findById(response.getBody().getId())
            .orElseThrow();
        assertThat(savedOrder.getProductId()).isEqualTo("product-1");
        assertThat(savedOrder.getQuantity()).isEqualTo(2);
    }
}
```

#### E2E Test (End-to-End 테스트)

```javascript
// Playwright E2E 테스트 예시
import { test, expect } from '@playwright/test';

test.describe('주문 프로세스 E2E 테스트', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/');
  });

  test('사용자가 상품을 장바구니에 담고 주문을 완료한다', async ({ page }) => {
    // 상품 검색
    await page.fill('[data-testid="search-input"]', '노트북');
    await page.click('[data-testid="search-button"]');

    // 상품 선택
    await page.click('[data-testid="product-card"]:first-child');
    await expect(page.locator('[data-testid="product-title"]')).toBeVisible();

    // 장바구니 추가
    await page.click('[data-testid="add-to-cart"]');
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');

    // 장바구니로 이동
    await page.click('[data-testid="cart-icon"]');
    await expect(page).toHaveURL('/cart');

    // 주문 진행
    await page.click('[data-testid="checkout-button"]');

    // 배송 정보 입력
    await page.fill('[data-testid="address-input"]', '서울시 강남구');
    await page.fill('[data-testid="phone-input"]', '010-1234-5678');

    // 결제 완료
    await page.click('[data-testid="pay-button"]');

    // 주문 완료 확인
    await expect(page.locator('[data-testid="order-success"]')).toBeVisible();
    await expect(page.locator('[data-testid="order-number"]')).toContainText('ORD-');
  });
});
```

### 4.3 테스트 병렬화 및 분할

```yaml
# GitHub Actions - 테스트 분할 (Test Sharding)
name: Parallel Tests

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run tests (shard ${{ matrix.shard }}/4)
        run: |
          npm run test -- --shard=${{ matrix.shard }}/4 --coverage

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.shard }}
          path: coverage/

  # 커버리지 병합
  coverage:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: coverage-*
          merge-multiple: true

      - name: Merge coverage reports
        run: npx nyc merge coverage/ merged-coverage.json

      - name: Upload to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: merged-coverage.json
```

---

## 5. 정적 분석 도구

### 5.1 SonarQube

**SonarQube**는 코드 품질과 보안을 위한 정적 분석 플랫폼으로, 35개 이상의 프로그래밍 언어를 지원하며 6,500개 이상의 규칙을 제공한다.

#### SonarQube 분석 대상

```
┌─────────────────────────────────────────────────────────────────┐
│                    SonarQube 분석 카테고리                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │     Bugs     │  │ Vulnera-     │  │ Code Smells  │          │
│  │              │  │ bilities     │  │              │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ 런타임에서   │  │ 보안 취약점  │  │ 유지보수성   │          │
│  │ 잘못 동작할  │  │ 및 보안      │  │ 저하 코드    │          │
│  │ 코드         │  │ 핫스팟       │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Coverage    │  │ Duplications │  │  Technical   │          │
│  │              │  │              │  │    Debt      │          │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤          │
│  │ 테스트       │  │ 중복 코드    │  │ 품질 문제    │          │
│  │ 커버리지     │  │ 비율         │  │ 해결 소요    │          │
│  │              │  │              │  │ 시간         │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### SonarQube 설정 (sonar-project.properties)

```properties
# SonarQube 프로젝트 설정
sonar.projectKey=my-backend-project
sonar.projectName=My Backend Project
sonar.projectVersion=1.0.0

# 소스 및 테스트 디렉토리
sonar.sources=src/main
sonar.tests=src/test
sonar.java.binaries=build/classes

# 커버리지 리포트
sonar.coverage.jacoco.xmlReportPaths=build/reports/jacoco/test/jacocoTestReport.xml

# 제외 패턴
sonar.exclusions=**/generated/**,**/dto/**,**/config/**
sonar.coverage.exclusions=**/Application.java,**/config/**

# Quality Gate (통과 기준)
sonar.qualitygate.wait=true
sonar.qualitygate.timeout=300
```

#### Quality Gate 설정

```
┌─────────────────────────────────────────────────────────────────┐
│                   권장 Quality Gate 조건                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Metric                      │ Condition        │ Value          │
│  ────────────────────────────┼──────────────────┼───────────     │
│  Coverage on New Code        │ is less than     │ 80%            │
│  Duplicated Lines (%)        │ is greater than  │ 3%             │
│  Maintainability Rating      │ is worse than    │ A              │
│  Reliability Rating          │ is worse than    │ A              │
│  Security Rating             │ is worse than    │ A              │
│  Security Hotspots Reviewed  │ is less than     │ 100%           │
│                                                                  │
│  ⚠️  Quality Gate 실패 시 파이프라인 중단                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 ESLint (JavaScript/TypeScript)

```javascript
// .eslintrc.js - 엔터프라이즈급 설정
module.exports = {
  root: true,
  parser: '@typescript-eslint/parser',
  parserOptions: {
    ecmaVersion: 2024,
    sourceType: 'module',
    project: './tsconfig.json',
  },
  plugins: [
    '@typescript-eslint',
    'import',
    'security',
    'sonarjs',
  ],
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:@typescript-eslint/recommended-requiring-type-checking',
    'plugin:import/recommended',
    'plugin:import/typescript',
    'plugin:security/recommended',
    'plugin:sonarjs/recommended',
  ],
  rules: {
    // 타입 안정성
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/explicit-function-return-type': 'warn',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],

    // 코드 복잡도
    'complexity': ['error', { max: 10 }],
    'max-depth': ['error', 4],
    'max-lines-per-function': ['warn', { max: 50 }],

    // Import 정리
    'import/order': ['error', {
      'groups': ['builtin', 'external', 'internal', 'parent', 'sibling', 'index'],
      'newlines-between': 'always',
      'alphabetize': { order: 'asc' }
    }],

    // 보안
    'security/detect-object-injection': 'warn',
    'security/detect-non-literal-regexp': 'warn',

    // SonarJS (코드 품질)
    'sonarjs/cognitive-complexity': ['error', 15],
    'sonarjs/no-duplicate-string': ['error', 3],
  },
  overrides: [
    {
      files: ['*.test.ts', '*.spec.ts'],
      rules: {
        '@typescript-eslint/no-explicit-any': 'off',
        'max-lines-per-function': 'off',
      }
    }
  ]
};
```

### 5.3 기타 정적 분석 도구

| 도구 | 용도 | 특징 |
|------|------|------|
| **Checkstyle** | Java 코딩 컨벤션 | Google/Sun 스타일 가이드 지원 |
| **PMD** | Java 버그 패턴 탐지 | Copy-Paste Detector (CPD) 포함 |
| **SpotBugs** | Java 바이트코드 분석 | FindBugs 후속 프로젝트 |
| **Prettier** | 코드 포맷팅 | 언어 무관 포맷터 |
| **Snyk** | 보안 취약점 스캔 | 의존성 취약점 탐지 |
| **Trivy** | 컨테이너 보안 | Docker 이미지 취약점 스캔 |

### 5.4 CI 파이프라인에서 정적 분석 통합

```yaml
# GitHub Actions - 종합 정적 분석
name: Code Quality

on:
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint -- --format=json --output-file=eslint-report.json
        continue-on-error: true

      - name: Annotate Code
        uses: ataylorme/eslint-annotate-action@v2
        with:
          report-json: eslint-report.json

  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 전체 히스토리 필요 (blame 분석)

      - name: SonarQube Scan
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

      - name: SonarQube Quality Gate
        uses: sonarsource/sonarqube-quality-gate-action@master
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      - name: Run Trivy (Container)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
```

---

## 참고 자료

- [GitLab CI/CD Best Practices](https://about.gitlab.com/topics/ci-cd/continuous-integration-best-practices/)
- [SonarQube Documentation](https://docs.sonarsource.com/sonarqube-server/2025.1/)
- [CI/CD Pipeline Best Practices 2025](https://www.kellton.com/kellton-tech-blog/continuous-integration-deployment-best-practices-2025)
- [Martin Fowler - Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)
- [Testcontainers](https://testcontainers.com/)

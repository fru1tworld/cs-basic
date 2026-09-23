# REST API

## REST 개요

REST(Representational State Transfer)는 Roy Fielding이 2000년 박사 논문에서 제안한 아키텍처 스타일이다. 웹의 기존 기술과 HTTP 프로토콜을 활용해 분산 시스템을 설계하며, 클라이언트와 서버가 상호작용할 때 지켜야 할 제약 조건을 정한다.

### REST vs RESTful

REST가 제약 조건의 집합을 가리킨다면, RESTful은 그 원칙을 따른다는 뜻이다. 따라서 REST 원칙을 따르는 API를 "RESTful API"라고 부른다.

- 참고: 대부분의 "REST API"는 실제로 Richardson Maturity Model Level 2 수준이며, 완전한 REST(Level 3)를 구현하는 경우는 드뭄.

## REST 제약 조건

- Roy Fielding이 정의한 6가지 아키텍처 제약 조건:

### Client-Server (클라이언트-서버)

클라이언트와 서버는 HTTP로 통신하되 서로 다른 관심사를 맡는다. 클라이언트는 UI/UX를, 서버는 데이터 저장과 비즈니스 로직을 담당한다. 이렇게 역할을 분리하면 양쪽을 독립적으로 발전시킬 수 있다.

### Stateless (무상태)

```java
// Bad - 서버가 세션 상태를 유지
@GetMapping("/cart")
public Cart getCart(HttpSession session) {
    return (Cart) session.getAttribute("cart");
}

// Good - 모든 요청에 필요한 정보 포함
@GetMapping("/cart")
public Cart getCart(@RequestHeader("Authorization") String token) {
    String userId = jwtService.extractUserId(token);
    return cartService.getCart(userId);
}
```

각 요청은 이전 요청의 상태에 의존하지 않고 독립적으로 처리할 수 있어야 한다. 이를 위해 클라이언트는 요청을 처리하는 데 필요한 정보를 매번 요청에 포함한다.

### Cacheable (캐시 가능)

```http
HTTP/1.1 200 OK
Cache-Control: max-age=3600, public
ETag: "abc123"
Last-Modified: Wed, 21 Oct 2025 07:28:00 GMT

{
  "id": 1,
  "name": "Product"
}
```

응답에는 캐시 가능 여부를 명시해야 한다. 위 응답처럼 캐시 정책을 전달하면 클라이언트나 중간 계층이 응답을 재사용할 수 있어 확장성과 성능을 높일 수 있다.

### Uniform Interface (균일한 인터페이스)

- REST의 가장 핵심적인 제약 조건으로, 4가지 하위 제약 조건으로 구성:

#### 리소스 식별 (Identification of Resources)

```text
GET /users/123          # 사용자 123 식별
GET /orders/456/items   # 주문 456의 아이템들 식별
```

#### 표현을 통한 리소스 조작 (Manipulation through Representations)

```json
// 클라이언트가 보내는 표현으로 리소스 조작
PUT /users/123
Content-Type: application/json

{
  "name": "Updated Name",
  "email": "new@email.com"
}
```

#### 자기 서술적 메시지 (Self-descriptive Messages)

```http
GET /users/123 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer eyJhbGc...
```

#### HATEOAS (Hypermedia as the Engine of Application State)

```json
{
  "id": 123,
  "name": "John",
  "_links": {
    "self": { "href": "/users/123" },
    "orders": { "href": "/users/123/orders" },
    "update": { "href": "/users/123", "method": "PUT" }
  }
}
```

### Layered System (계층화 시스템)

요청은 `Client → Load Balancer → API Gateway → Application → Database`와 같은 경로를 거칠 수 있고, 캐시 계층도 둘 수 있다. 클라이언트는 자신이 직접 서버와 통신하는지 중간 계층과 통신하는지 알 수 없다. 이 제약 덕분에 클라이언트와의 인터페이스를 유지하면서 보안, 로드 밸런싱, 캐싱을 위한 계층을 추가할 수 있다.

### Code on Demand (선택적)

```html
<!-- 서버가 클라이언트에 실행 가능한 코드 전송 -->
<script src="https://api.example.com/scripts/widget.js"></script>
```

클라이언트는 JavaScript처럼 실행 가능한 코드를 서버에서 내려받을 수 있다. Code on Demand는 여섯 가지 제약 조건 중 유일하게 선택적으로 적용한다.

## Richardson Maturity Model

- Leonard Richardson이 2008년에 제안한 REST API 성숙도 모델:

- Level 3: Hypermedia Controls (HATEOAS), ← 완전한 REST
- Level 2: HTTP Verbs
- Level 1: Resources
- Level 0: The Swamp of POX

### Level 0: The Swamp of POX (Plain Old XML)

- 단일 엔드포인트, 단일 HTTP 메서드(보통 POST) 사용:

```http
POST /api HTTP/1.1
Content-Type: application/json

{
  "action": "getUser",
  "userId": 123
}

---

POST /api HTTP/1.1
Content-Type: application/json

{
  "action": "createOrder",
  "data": { ... }
}
```

- 특징:
- SOAP, XML-RPC 스타일
- HTTP를 단순 전송 프로토콜로만 사용
- 대부분의 레거시 시스템

### Level 1: Resources

- 개별 리소스에 대한 고유 URI 사용:

```http
POST /users HTTP/1.1
{ "action": "get", "id": 123 }

POST /orders HTTP/1.1
{ "action": "create", "items": [...] }
```

Level 1에서는 리소스별로 엔드포인트를 나누므로 URI로 리소스를 식별할 수 있다. 다만 위 예제에서는 여전히 모든 작업에 POST를 사용하므로, 조회와 생성의 차이는 요청 본문을 읽어야 알 수 있다.

### Level 2: HTTP Verbs

- HTTP 메서드를 의미에 맞게 사용:

```http
# 사용자 조회
GET /users/123 HTTP/1.1

# 사용자 생성
POST /users HTTP/1.1
Content-Type: application/json
{ "name": "John", "email": "john@example.com" }

# 사용자 수정
PUT /users/123 HTTP/1.1
Content-Type: application/json
{ "name": "John Updated" }

# 사용자 삭제
DELETE /users/123 HTTP/1.1

# 응답에 적절한 상태 코드 사용
HTTP/1.1 201 Created
Location: /users/124
```

- 특징:
- GET, POST, PUT, DELETE 등 HTTP 메서드 활용
- 적절한 HTTP 상태 코드 반환
- 현재 대부분의 "REST API"가 이 수준

### Level 3: Hypermedia Controls (HATEOAS)

```http
GET /users/123 HTTP/1.1

HTTP/1.1 200 OK
Content-Type: application/hal+json

{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "status": "active",
  "_links": {
    "self": {
      "href": "/users/123"
    },
    "orders": {
      "href": "/users/123/orders"
    },
    "deactivate": {
      "href": "/users/123/deactivate",
      "method": "POST"
    },
    "update": {
      "href": "/users/123",
      "method": "PUT"
    }
  },
  "_embedded": {
    "recentOrders": [
      {
        "id": 456,
        "total": 100.00,
        "_links": {
          "self": { "href": "/orders/456" }
        }
      }
    ]
  }
}
```

- 특징:
- 응답에 다음 가능한 액션의 하이퍼링크 포함
- 클라이언트가 API 구조를 하드코딩할 필요 없음
- API 진화에 유연함
- Roy Fielding이 말하는 진정한 REST

### 실무에서의 선택

- 상황: 내부 마이크로서비스:
  - 권장 레벨: Level 2
  - 이유: 구현 단순, 충분한 표현력
- 상황: 퍼블릭 API:
  - 권장 레벨: Level 2~3
  - 이유: 확장성, 클라이언트 편의성
- 상황: 대규모 분산 시스템:
  - 권장 레벨: Level 3
  - 이유: 느슨한 결합, API 진화 용이

## 리소스 네이밍 컨벤션

### 기본 원칙

#### 명사 사용, 동사 지양

```text
# Good
GET /users
GET /articles
POST /orders

# Bad
GET /getUsers
POST /createOrder
GET /retrieveArticles
```

#### 복수형 명사 사용

```text
# Good
GET /users
GET /users/123
GET /products

# Bad (일관성 없음)
GET /user
GET /user/123
GET /product
```

#### 소문자와 하이픈(-) 사용

```text
# Good
GET /user-profiles
GET /order-items

# Bad
GET /userProfiles    # camelCase
GET /user_profiles   # snake_case
GET /UserProfiles    # PascalCase
```

#### 계층 관계 표현

```text
# 주문에 속한 아이템들
GET /orders/123/items

# 사용자의 주소록
GET /users/456/addresses

# 조직의 팀의 멤버들
GET /organizations/789/teams/101/members
```

#### 파일 확장자 지양

```text
# Good
GET /users/123
Accept: application/json

# Bad
GET /users/123.json
GET /users/123.xml
```

### 실전 예시

- # 컬렉션
- GET, /products, # 모든 상품 조회
- POST, /products, # 상품 생성
- # 단일 리소스
- GET, /products/{id}, # 특정 상품 조회
- PUT, /products/{id}, # 상품 전체 수정
- PATCH, /products/{id}, # 상품 부분 수정
- DELETE, /products/{id}, # 상품 삭제
- # 중첩 리소스
- GET, /products/{id}/reviews, # 상품의 리뷰 목록
- POST, /products/{id}/reviews, # 상품에 리뷰 작성
- # 필터링/정렬/페이징 (쿼리 파라미터 사용)
- GET, /products?category=electronics&sort=price&order=asc&page=1&limit=20
- # 액션 (동사가 필요한 경우)
- POST, /users/{id}/activate, # 계정 활성화
- POST, /orders/{id}/cancel, # 주문 취소
- POST, /documents/{id}/publish # 문서 발행

### 안티패턴

```text
# 쿼리 파라미터로 CRUD 구분 - Bad
GET /api?method=getUser&id=123
GET /api?method=deleteUser&id=123

# 동사 사용 - Bad
GET /getUsers
POST /createUser
POST /deleteUser/123

# 일관성 없는 네이밍 - Bad
GET /user/123
GET /products/456
GET /GetOrders

# 너무 깊은 중첩 - Bad (3단계 이하 권장)
GET /countries/1/states/2/cities/3/districts/4/streets/5/buildings/6
```

## HTTP 메서드 사용법

### 메서드별 특성

- 메서드: GET:
  - 안전성: O
  - 멱등성: O
  - 캐시 가능: O
  - 요청 본문: X
  - 용도: 리소스 조회
- 메서드: POST:
  - 안전성: X
  - 멱등성: X
  - 캐시 가능: 조건부
  - 요청 본문: O
  - 용도: 리소스 생성, 복잡한 검색
- 메서드: PUT:
  - 안전성: X
  - 멱등성: O
  - 캐시 가능: X
  - 요청 본문: O
  - 용도: 리소스 전체 교체
- 메서드: PATCH:
  - 안전성: X
  - 멱등성: X\*
  - 캐시 가능: X
  - 요청 본문: O
  - 용도: 리소스 부분 수정
- 메서드: DELETE:
  - 안전성: X
  - 멱등성: O
  - 캐시 가능: X
  - 요청 본문: X
  - 용도: 리소스 삭제
- 메서드: HEAD:
  - 안전성: O
  - 멱등성: O
  - 캐시 가능: O
  - 요청 본문: X
  - 용도: 헤더만 조회
- 메서드: OPTIONS:
  - 안전성: O
  - 멱등성: O
  - 캐시 가능: X
  - 요청 본문: X
  - 용도: 지원 메서드 확인

- 안전성(Safe): 서버 상태를 변경하지 않음
- 멱등성(Idempotent): 여러 번 실행해도 결과가 동일

### GET - 리소스 조회

```http
# 컬렉션 조회
GET /users HTTP/1.1
Host: api.example.com
Accept: application/json

# 단일 리소스 조회
GET /users/123 HTTP/1.1

# 필터링, 정렬, 페이징
GET /users?role=admin&sort=-createdAt&page=1&limit=20 HTTP/1.1

# 필드 선택 (Sparse Fieldsets)
GET /users/123?fields=id,name,email HTTP/1.1
```

- 응답 예시:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=3600
ETag: "abc123"

{
  "data": [...],
  "meta": {
    "total": 100,
    "page": 1,
    "limit": 20
  }
}
```

### POST - 리소스 생성

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user"
}
```

- 응답:

```http
HTTP/1.1 201 Created
Location: /users/124
Content-Type: application/json

{
  "id": 124,
  "name": "John Doe",
  "email": "john@example.com",
  "role": "user",
  "createdAt": "2025-01-15T10:30:00Z"
}
```

### PUT vs PATCH

#### PUT - 전체 교체

```http
PUT /users/123 HTTP/1.1
Content-Type: application/json

{
  "name": "John Updated",
  "email": "john.updated@example.com",
  "role": "admin",
  "address": "New Address"
}
```

- 전체 리소스를 새 데이터로 교체
- 누락된 필드는 null 또는 기본값으로 설정
- 멱등성 보장

#### PATCH - 부분 수정

```http
PATCH /users/123 HTTP/1.1
Content-Type: application/json

{
  "email": "new.email@example.com"
}
```

```http
# JSON Patch (RFC 6902)
PATCH /users/123 HTTP/1.1
Content-Type: application/json-patch+json

[
  { "op": "replace", "path": "/email", "value": "new@example.com" },
  { "op": "add", "path": "/phone", "value": "010-1234-5678" },
  { "op": "remove", "path": "/nickname" }
]
```

PATCH는 지정된 필드만 수정하고 나머지 필드는 유지한다. 위 JSON Patch 예제에서는 `email`을 교체하고 `phone`을 추가하며 `nickname`을 제거한다.

### DELETE - 리소스 삭제

```http
DELETE /users/123 HTTP/1.1
Authorization: Bearer <token>
```

- 응답:

```http
HTTP/1.1 204 No Content
```

- 또는

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "message": "User successfully deleted",
  "deletedAt": "2025-01-15T10:30:00Z"
}
```

### 실전 코드 예시 (Spring Boot)

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping
    public ResponseEntity<Page<UserResponse>> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String role) {

        Page<UserResponse> users = userService.findUsers(page, size, role);
        return ResponseEntity.ok(users);
    }

    @GetMapping("/{id}")
    public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
        return userService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {

        UserResponse created = userService.create(request);
        URI location = URI.create("/api/v1/users/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UpdateUserRequest request) {

        return userService.update(id, request)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PatchMapping("/{id}")
    public ResponseEntity<UserResponse> patchUser(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {

        return userService.patch(id, updates)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

## HATEOAS

HATEOAS(Hypermedia As The Engine Of Application State)는 서버가 가능한 액션을 응답에 포함해 클라이언트와 동적으로 상호작용하는 REST의 핵심 원칙이다. 클라이언트가 모든 엔드포인트를 미리 알아야 하는 방식에서는 API 구조에 강하게 결합된다. HATEOAS에서는 응답에 담긴 링크를 따라 다음 액션을 선택하므로 이 결합을 줄일 수 있다.

### HAL (Hypertext Application Language)

- HAL은 HATEOAS 구현을 위한 가장 널리 사용되는 미디어 타입임.

```json
{
  "_links": {
    "self": { "href": "/orders/123" },
    "customer": { "href": "/customers/456" },
    "items": { "href": "/orders/123/items" },
    "cancel": {
      "href": "/orders/123/cancel",
      "method": "POST"
    }
  },
  "_embedded": {
    "items": [
      {
        "_links": {
          "self": { "href": "/products/789" }
        },
        "name": "Widget",
        "quantity": 2,
        "price": 9.99
      }
    ]
  },
  "id": 123,
  "status": "pending",
  "total": 19.98
}
```

### Spring HATEOAS 구현

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping("/{id}")
    public EntityModel<OrderResponse> getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id);
        OrderResponse response = OrderResponse.from(order);

        EntityModel<OrderResponse> model = EntityModel.of(response);

        // Self 링크
        model.add(linkTo(methodOn(OrderController.class)
            .getOrder(id)).withSelfRel());

        // 관련 리소스 링크
        model.add(linkTo(methodOn(CustomerController.class)
            .getCustomer(order.getCustomerId())).withRel("customer"));
        model.add(linkTo(methodOn(OrderController.class)
            .getOrderItems(id)).withRel("items"));

        // 상태에 따른 조건부 링크
        if (order.isCancellable()) {
            model.add(linkTo(methodOn(OrderController.class)
                .cancelOrder(id)).withRel("cancel"));
        }
        if (order.isPayable()) {
            model.add(linkTo(methodOn(OrderController.class)
                .payOrder(id)).withRel("pay"));
        }

        return model;
    }

    @GetMapping
    public CollectionModel<EntityModel<OrderResponse>> getAllOrders() {
        List<EntityModel<OrderResponse>> orders = orderService.findAll()
            .stream()
            .map(order -> EntityModel.of(OrderResponse.from(order),
                linkTo(methodOn(OrderController.class)
                    .getOrder(order.getId())).withSelfRel()))
            .collect(Collectors.toList());

        return CollectionModel.of(orders,
            linkTo(methodOn(OrderController.class)
                .getAllOrders()).withSelfRel());
    }
}
```

### HATEOAS의 장단점

- 장점:
- API 변경에 클라이언트 수정 최소화
- 자체 문서화 (Self-documenting)
- 클라이언트가 API 발견 가능 (Discoverable)
- API 진화에 유연

- 단점:
- 응답 크기 증가
- 구현 복잡도 증가
- 클라이언트 로직 복잡도 증가
- 단순한 API에는 과도한 설계

## API 버전 관리 전략

### URI Path 버전 관리 (가장 일반적)

```http
GET /api/v1/users HTTP/1.1
GET /api/v2/users HTTP/1.1
```

- 장점:
- 직관적이고 명확
- 캐싱 용이
- 브라우저에서 테스트 용이

- 단점:
- URI가 변경됨
- REST 원칙에 어긋남 (리소스 자체는 동일)

### Query Parameter 버전 관리

```http
GET /api/users?version=1 HTTP/1.1
GET /api/users?version=2 HTTP/1.1
```

- 장점:
- 버전 없이 호출 시 기본 버전 사용 가능
- URI 구조 유지

- 단점:
- 캐싱 복잡
- 필수 파라미터로 강제하기 어려움

### 커스텀 Header 버전 관리

```http
GET /api/users HTTP/1.1
X-API-Version: 1

GET /api/users HTTP/1.1
X-API-Version: 2
```

- 장점:
- URI 깔끔
- 버전 정보 분리

- 단점:
- 브라우저 테스트 어려움
- 표준 헤더 아님

### Accept Header (Content Negotiation)

```http
GET /api/users HTTP/1.1
Accept: application/vnd.example.v1+json

GET /api/users HTTP/1.1
Accept: application/vnd.example.v2+json
```

- 장점:
- HTTP 표준 활용
- REST 원칙에 가장 부합
- 미디어 타입과 버전을 함께 표현

- 단점:
- 구현 복잡
- 테스트 어려움

### 버전 관리 전략 비교

- 전략: URI Path:
  - GitHub: O
  - Google: O
  - Facebook: O
  - Twitter: O
- 전략: Query Param:

- 전략: Header:

- 전략: Accept:
  - GitHub: O

### 버전 관리 모범 사례

```java
// Spring Boot에서 URI 버전 관리
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping("/{id}")
    public UserResponseV1 getUser(@PathVariable Long id) {
        // V1 응답 형식
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserResponseV2 getUser(@PathVariable Long id) {
        // V2 응답 형식 (추가 필드 포함)
    }
}
```

### 버전 업그레이드 가이드라인

- 하위 호환성 유지가 원칙
  - 필드 추가 → 버전 유지
  - 필드 삭제/타입 변경 → 새 버전

- Deprecation 정책

```http
   HTTP/1.1 200 OK
   Deprecation: true
   Sunset: Sat, 31 Dec 2025 23:59:59 GMT
   Link: </api/v2/users>; rel="successor-version"
```

- 버전 수명 주기 관리
  - 새 버전 출시 후 이전 버전 최소 12개월 유지
  - Deprecation 알림 후 최소 6개월 후 종료

## 상태 코드 가이드

### 2xx - 성공

- 코드: 200:
  - 이름: OK
  - 용도: 일반적인 성공
- 코드: 201:
  - 이름: Created
  - 용도: 리소스 생성 성공
- 코드: 202:
  - 이름: Accepted
  - 용도: 비동기 처리 접수
- 코드: 204:
  - 이름: No Content
  - 용도: 성공, 응답 본문 없음 (DELETE)

### 3xx - 리다이렉션

- 코드: 301:
  - 이름: Moved Permanently
  - 용도: 영구적 이동
- 코드: 302:
  - 이름: Found
  - 용도: 임시 이동
- 코드: 304:
  - 이름: Not Modified
  - 용도: 캐시 사용

### 4xx - 클라이언트 오류

- 코드: 400:
  - 이름: Bad Request
  - 용도: 잘못된 요청 형식
- 코드: 401:
  - 이름: Unauthorized
  - 용도: 인증 필요
- 코드: 403:
  - 이름: Forbidden
  - 용도: 권한 없음
- 코드: 404:
  - 이름: Not Found
  - 용도: 리소스 없음
- 코드: 405:
  - 이름: Method Not Allowed
  - 용도: 지원하지 않는 메서드
- 코드: 409:
  - 이름: Conflict
  - 용도: 리소스 충돌
- 코드: 422:
  - 이름: Unprocessable Entity
  - 용도: 검증 실패
- 코드: 429:
  - 이름: Too Many Requests
  - 용도: 요청 횟수 초과

### 5xx - 서버 오류

- 코드: 500:
  - 이름: Internal Server Error
  - 용도: 서버 내부 오류
- 코드: 502:
  - 이름: Bad Gateway
  - 용도: 게이트웨이 오류
- 코드: 503:
  - 이름: Service Unavailable
  - 용도: 서비스 이용 불가
- 코드: 504:
  - 이름: Gateway Timeout
  - 용도: 게이트웨이 타임아웃

### 에러 응답 형식 (RFC 7807)

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Failed",
  "status": 422,
  "detail": "The request body contains invalid fields",
  "instance": "/api/v1/users",
  "errors": [
    {
      "field": "email",
      "message": "Must be a valid email address"
    },
    {
      "field": "age",
      "message": "Must be at least 18"
    }
  ],
  "timestamp": "2025-01-15T10:30:00Z",
  "traceId": "abc123xyz"
}
```

## 참고 자료

- [Roy Fielding's Dissertation - REST](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [RFC 7231 - HTTP/1.1 Semantics and Content](https://tools.ietf.org/html/rfc7231)
- [RFC 6902 - JSON Patch](https://tools.ietf.org/html/rfc6902)
- [RFC 7807 - Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807)
- [Richardson Maturity Model - Martin Fowler](https://www.martinfowler.com/articles/richardsonMaturityModel.html)
- [REST API Tutorial](https://restfulapi.net/)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)

## 관련 학습

- [GraphQL](02-GraphQL.md)

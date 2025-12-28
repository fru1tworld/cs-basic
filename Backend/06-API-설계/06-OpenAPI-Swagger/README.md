# OpenAPI/Swagger

## 목차
1. [OpenAPI 개요](#openapi-개요)
2. [OpenAPI 3.0 스펙](#openapi-30-스펙)
3. [Swagger UI/Editor](#swagger-uieditor)
4. [문서 자동화](#문서-자동화)
5. [Mock Server 구축](#mock-server-구축)
6. [실전 구현 예제](#실전-구현-예제)
---

## OpenAPI 개요

OpenAPI Specification(OAS)은 RESTful API를 기술하기 위한 표준 스펙입니다. 이전에는 Swagger Specification으로 알려졌으며, 2016년 Linux Foundation의 OpenAPI Initiative로 이관되었습니다.

### 역사

```
2010: Swagger 프로젝트 시작
2015: SmartBear가 Swagger 인수
2016: Swagger Specification → OpenAPI Specification 3.0
2017: OpenAPI 3.0.0 릴리스
2021: OpenAPI 3.1.0 릴리스 (JSON Schema 호환)
```

### OpenAPI 생태계

```
┌─────────────────────────────────────────────────────────┐
│                    OpenAPI Ecosystem                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────┐    ┌─────────────────┐            │
│  │ OpenAPI Spec    │    │ Swagger UI      │            │
│  │ (YAML/JSON)     │───>│ (Documentation) │            │
│  └────────┬────────┘    └─────────────────┘            │
│           │                                             │
│           ├────────────>┌─────────────────┐            │
│           │             │ Swagger Editor  │            │
│           │             │ (Design)        │            │
│           │             └─────────────────┘            │
│           │                                             │
│           ├────────────>┌─────────────────┐            │
│           │             │ Swagger Codegen │            │
│           │             │ (Code Gen)      │            │
│           │             └─────────────────┘            │
│           │                                             │
│           └────────────>┌─────────────────┐            │
│                         │ Mock Server     │            │
│                         │ (Prism, etc.)   │            │
│                         └─────────────────┘            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### API-First Design

```
┌─────────────────────────────────────────────────────────┐
│                    API-First Workflow                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Design    → OpenAPI 스펙 작성                       │
│  2. Review    → 이해관계자 검토 (Swagger UI)            │
│  3. Mock      → Mock 서버로 프론트엔드 개발 시작         │
│  4. Implement → 백엔드 구현                             │
│  5. Test      → 스펙 기반 자동 테스트                    │
│  6. Document  → 문서 자동 생성                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## OpenAPI 3.0 스펙

### 기본 구조

```yaml
# openapi.yaml
openapi: 3.0.3
info:
  title: My API
  description: API 설명
  version: 1.0.0
  contact:
    name: API Support
    email: support@example.com
    url: https://example.com/support
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT
  termsOfService: https://example.com/terms

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1
    description: Staging server
  - url: http://localhost:3000/v1
    description: Local development

tags:
  - name: Users
    description: 사용자 관련 API
  - name: Products
    description: 상품 관련 API

paths:
  /users:
    # ... 엔드포인트 정의

components:
  schemas:
    # ... 스키마 정의
  securitySchemes:
    # ... 인증 방식 정의
```

### Paths (경로)

```yaml
paths:
  /users:
    get:
      tags:
        - Users
      summary: 사용자 목록 조회
      description: |
        모든 사용자 목록을 페이지네이션과 함께 조회합니다.
        관리자만 접근 가능합니다.
      operationId: getUsers
      parameters:
        - name: page
          in: query
          description: 페이지 번호
          required: false
          schema:
            type: integer
            minimum: 1
            default: 1
        - name: limit
          in: query
          description: 페이지당 항목 수
          required: false
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
        - name: sort
          in: query
          description: 정렬 기준
          schema:
            type: string
            enum: [name, createdAt, -name, -createdAt]
            default: -createdAt
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserListResponse'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '403':
          $ref: '#/components/responses/Forbidden'
      security:
        - bearerAuth: []

    post:
      tags:
        - Users
      summary: 사용자 생성
      operationId: createUser
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
            examples:
              basic:
                summary: 기본 사용자
                value:
                  name: John Doe
                  email: john@example.com
                  role: user
              admin:
                summary: 관리자
                value:
                  name: Admin User
                  email: admin@example.com
                  role: admin
      responses:
        '201':
          description: 생성 성공
          headers:
            Location:
              description: 생성된 리소스 URL
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          description: 이메일 중복
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'

  /users/{userId}:
    get:
      tags:
        - Users
      summary: 사용자 상세 조회
      operationId: getUserById
      parameters:
        - name: userId
          in: path
          required: true
          description: 사용자 ID
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'

    put:
      tags:
        - Users
      summary: 사용자 정보 수정
      operationId: updateUser
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateUserRequest'
      responses:
        '200':
          description: 수정 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          $ref: '#/components/responses/NotFound'

    delete:
      tags:
        - Users
      summary: 사용자 삭제
      operationId: deleteUser
      parameters:
        - name: userId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '204':
          description: 삭제 성공
        '404':
          $ref: '#/components/responses/NotFound'
```

### Components - Schemas

```yaml
components:
  schemas:
    # 기본 사용자 스키마
    User:
      type: object
      required:
        - id
        - email
        - name
      properties:
        id:
          type: string
          format: uuid
          readOnly: true
          description: 사용자 고유 식별자
          example: "550e8400-e29b-41d4-a716-446655440000"
        email:
          type: string
          format: email
          description: 이메일 주소
          example: john@example.com
        name:
          type: string
          minLength: 1
          maxLength: 100
          description: 사용자 이름
          example: John Doe
        role:
          type: string
          enum: [user, admin, moderator]
          default: user
          description: 사용자 역할
        avatar:
          type: string
          format: uri
          nullable: true
          description: 프로필 이미지 URL
        createdAt:
          type: string
          format: date-time
          readOnly: true
          description: 생성 일시
        updatedAt:
          type: string
          format: date-time
          readOnly: true
          description: 수정 일시

    # 생성 요청
    CreateUserRequest:
      type: object
      required:
        - email
        - name
        - password
      properties:
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 1
          maxLength: 100
        password:
          type: string
          format: password
          minLength: 8
          description: 최소 8자, 대소문자, 숫자, 특수문자 포함
        role:
          type: string
          enum: [user, admin, moderator]
          default: user

    # 수정 요청
    UpdateUserRequest:
      type: object
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
        avatar:
          type: string
          format: uri
          nullable: true

    # 목록 응답 (페이지네이션)
    UserListResponse:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/User'
        meta:
          $ref: '#/components/schemas/PaginationMeta'

    # 페이지네이션 메타
    PaginationMeta:
      type: object
      properties:
        page:
          type: integer
          minimum: 1
        limit:
          type: integer
        total:
          type: integer
        totalPages:
          type: integer

    # 에러 응답
    Error:
      type: object
      required:
        - code
        - message
      properties:
        code:
          type: string
          description: 에러 코드
          example: VALIDATION_ERROR
        message:
          type: string
          description: 에러 메시지
          example: 입력값이 유효하지 않습니다
        details:
          type: array
          items:
            type: object
            properties:
              field:
                type: string
              message:
                type: string
```

### Components - Responses

```yaml
components:
  responses:
    BadRequest:
      description: 잘못된 요청
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: VALIDATION_ERROR
            message: 입력값이 유효하지 않습니다
            details:
              - field: email
                message: 유효한 이메일 형식이 아닙니다

    Unauthorized:
      description: 인증 필요
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: UNAUTHORIZED
            message: 인증이 필요합니다

    Forbidden:
      description: 권한 없음
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: FORBIDDEN
            message: 접근 권한이 없습니다

    NotFound:
      description: 리소스 없음
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: NOT_FOUND
            message: 요청한 리소스를 찾을 수 없습니다

    TooManyRequests:
      description: 요청 횟수 초과
      headers:
        Retry-After:
          description: 재시도까지 대기 시간 (초)
          schema:
            type: integer
        RateLimit-Limit:
          description: 요청 한도
          schema:
            type: integer
        RateLimit-Remaining:
          description: 남은 요청 횟수
          schema:
            type: integer
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: RATE_LIMIT_EXCEEDED
            message: 요청 횟수를 초과했습니다
```

### Components - Security Schemes

```yaml
components:
  securitySchemes:
    # Bearer Token (JWT)
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: JWT 토큰을 Authorization 헤더에 포함

    # API Key (Header)
    apiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key
      description: API Key를 헤더에 포함

    # API Key (Query)
    apiKeyQuery:
      type: apiKey
      in: query
      name: api_key
      description: API Key를 쿼리 파라미터로 전달

    # OAuth 2.0
    oauth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.example.com/oauth/authorize
          tokenUrl: https://auth.example.com/oauth/token
          refreshUrl: https://auth.example.com/oauth/refresh
          scopes:
            read:users: 사용자 정보 읽기
            write:users: 사용자 정보 수정
            admin: 관리자 권한

    # Basic Auth
    basicAuth:
      type: http
      scheme: basic

# 전역 보안 적용
security:
  - bearerAuth: []
```

### Parameters (재사용)

```yaml
components:
  parameters:
    # 경로 파라미터
    UserId:
      name: userId
      in: path
      required: true
      description: 사용자 ID
      schema:
        type: string
        format: uuid

    # 쿼리 파라미터 - 페이지네이션
    PageParam:
      name: page
      in: query
      description: 페이지 번호
      schema:
        type: integer
        minimum: 1
        default: 1

    LimitParam:
      name: limit
      in: query
      description: 페이지당 항목 수
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

    # 헤더 파라미터
    RequestIdHeader:
      name: X-Request-ID
      in: header
      description: 요청 추적 ID
      schema:
        type: string
        format: uuid

# 사용
paths:
  /users/{userId}:
    get:
      parameters:
        - $ref: '#/components/parameters/UserId'
```

### 고급 스키마 기능

```yaml
components:
  schemas:
    # oneOf - 하나만 일치
    PaymentMethod:
      oneOf:
        - $ref: '#/components/schemas/CreditCard'
        - $ref: '#/components/schemas/BankTransfer'
        - $ref: '#/components/schemas/PayPal'
      discriminator:
        propertyName: type
        mapping:
          credit_card: '#/components/schemas/CreditCard'
          bank_transfer: '#/components/schemas/BankTransfer'
          paypal: '#/components/schemas/PayPal'

    CreditCard:
      type: object
      required:
        - type
        - cardNumber
      properties:
        type:
          type: string
          enum: [credit_card]
        cardNumber:
          type: string
        expiryDate:
          type: string
          pattern: '^(0[1-9]|1[0-2])\/\d{2}$'

    # allOf - 모두 일치 (상속)
    AdminUser:
      allOf:
        - $ref: '#/components/schemas/User'
        - type: object
          properties:
            permissions:
              type: array
              items:
                type: string
            department:
              type: string

    # anyOf - 하나 이상 일치
    NotificationTarget:
      anyOf:
        - type: object
          properties:
            email:
              type: string
              format: email
        - type: object
          properties:
            phone:
              type: string

    # 추가 속성 (동적 키)
    Metadata:
      type: object
      additionalProperties:
        type: string
      example:
        key1: value1
        key2: value2
```

---

## Swagger UI/Editor

### Swagger UI 설정

```javascript
// Express + Swagger UI
const express = require('express');
const swaggerUi = require('swagger-ui-express');
const YAML = require('yamljs');

const app = express();

// OpenAPI 스펙 로드
const swaggerDocument = YAML.load('./openapi.yaml');

// Swagger UI 옵션
const swaggerOptions = {
  explorer: true,                    // API 탐색기 활성화
  customCss: '.swagger-ui .topbar { display: none }',  // 커스텀 CSS
  customSiteTitle: 'My API Documentation',
  swaggerOptions: {
    persistAuthorization: true,      // 인증 정보 유지
    displayRequestDuration: true,    // 요청 시간 표시
    filter: true,                    // 필터링 활성화
    deepLinking: true,               // 딥 링킹
    defaultModelsExpandDepth: 3,     // 모델 확장 깊이
    defaultModelExpandDepth: 3,
  }
};

// 라우트 설정
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerDocument, swaggerOptions));

// JSON/YAML 스펙 직접 제공
app.get('/openapi.json', (req, res) => {
  res.json(swaggerDocument);
});

app.listen(3000, () => {
  console.log('API docs at http://localhost:3000/api-docs');
});
```

### 다중 버전 지원

```javascript
const v1Spec = YAML.load('./specs/v1/openapi.yaml');
const v2Spec = YAML.load('./specs/v2/openapi.yaml');

// 버전별 문서
app.use('/api/v1/docs', swaggerUi.serveFiles(v1Spec), swaggerUi.setup(v1Spec));
app.use('/api/v2/docs', swaggerUi.serveFiles(v2Spec), swaggerUi.setup(v2Spec));

// 버전 선택 페이지
app.get('/docs', (req, res) => {
  res.send(`
    <h1>API Documentation</h1>
    <ul>
      <li><a href="/api/v1/docs">API v1</a></li>
      <li><a href="/api/v2/docs">API v2</a></li>
    </ul>
  `);
});
```

### Swagger Editor 사용

```bash
# Docker로 로컬 실행
docker run -d -p 8080:8080 swaggerapi/swagger-editor

# 온라인 에디터: https://editor.swagger.io/
```

### ReDoc (대안)

```javascript
const redoc = require('redoc-express');

app.get('/docs', redoc({
  title: 'API Documentation',
  specUrl: '/openapi.json',
  redocOptions: {
    theme: {
      colors: {
        primary: { main: '#1976d2' }
      }
    },
    hideDownloadButton: false,
    expandResponses: '200,201'
  }
}));
```

---

## 문서 자동화

### Code-First Approach (코드에서 스펙 생성)

#### 1. Spring Boot (springdoc-openapi)

```java
// build.gradle
dependencies {
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'
}

// application.yml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha

// OpenAPI 설정
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("My API")
                .version("1.0.0")
                .description("API 설명")
                .contact(new Contact()
                    .name("Support")
                    .email("support@example.com")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }
}

// Controller 어노테이션
@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "Users", description = "사용자 관리 API")
public class UserController {

    @Operation(
        summary = "사용자 목록 조회",
        description = "모든 사용자 목록을 페이지네이션과 함께 조회합니다"
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "성공",
            content = @Content(schema = @Schema(implementation = UserListResponse.class))),
        @ApiResponse(responseCode = "401", description = "인증 필요")
    })
    @GetMapping
    public ResponseEntity<UserListResponse> getUsers(
        @Parameter(description = "페이지 번호") @RequestParam(defaultValue = "1") int page,
        @Parameter(description = "페이지 크기") @RequestParam(defaultValue = "20") int size
    ) {
        // ...
    }

    @Operation(summary = "사용자 생성")
    @PostMapping
    public ResponseEntity<User> createUser(
        @RequestBody @Valid CreateUserRequest request
    ) {
        // ...
    }
}

// DTO 스키마
@Schema(description = "사용자 생성 요청")
public class CreateUserRequest {

    @Schema(description = "이메일", example = "john@example.com", required = true)
    @Email
    @NotBlank
    private String email;

    @Schema(description = "이름", example = "John Doe", minLength = 1, maxLength = 100)
    @Size(min = 1, max = 100)
    @NotBlank
    private String name;

    @Schema(description = "비밀번호", format = "password", minLength = 8)
    @Size(min = 8)
    @NotBlank
    private String password;
}
```

#### 2. NestJS (Swagger)

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('API 설명')
    .setVersion('1.0.0')
    .addBearerAuth()
    .addTag('Users', '사용자 관리')
    .build();

  const document = SwaggerModule.createDocument(app, config);

  // OpenAPI 스펙 파일로 저장
  const fs = require('fs');
  fs.writeFileSync('./openapi.json', JSON.stringify(document, null, 2));

  SwaggerModule.setup('api-docs', app, document);

  await app.listen(3000);
}
bootstrap();

// user.controller.ts
import { Controller, Get, Post, Body, Param, Query } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiBearerAuth } from '@nestjs/swagger';
import { CreateUserDto, UserDto, UserListDto } from './dto';

@ApiTags('Users')
@Controller('users')
@ApiBearerAuth()
export class UserController {

  @Get()
  @ApiOperation({ summary: '사용자 목록 조회' })
  @ApiResponse({ status: 200, type: UserListDto })
  async getUsers(
    @Query('page') page: number = 1,
    @Query('limit') limit: number = 20,
  ): Promise<UserListDto> {
    // ...
  }

  @Post()
  @ApiOperation({ summary: '사용자 생성' })
  @ApiResponse({ status: 201, type: UserDto })
  @ApiResponse({ status: 400, description: '잘못된 요청' })
  async createUser(@Body() dto: CreateUserDto): Promise<UserDto> {
    // ...
  }
}

// dto/create-user.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsNotEmpty, MinLength } from 'class-validator';

export class CreateUserDto {
  @ApiProperty({
    description: '이메일',
    example: 'john@example.com',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: '이름',
    example: 'John Doe',
    minLength: 1,
    maxLength: 100,
  })
  @IsNotEmpty()
  name: string;

  @ApiProperty({
    description: '비밀번호',
    format: 'password',
    minLength: 8,
  })
  @MinLength(8)
  password: string;
}
```

#### 3. FastAPI (Python)

```python
from fastapi import FastAPI, Query, Path, Body, HTTPException
from fastapi.openapi.utils import get_openapi
from pydantic import BaseModel, Field, EmailStr
from typing import List, Optional
from enum import Enum

app = FastAPI(
    title="My API",
    description="API 설명",
    version="1.0.0",
    openapi_tags=[
        {"name": "Users", "description": "사용자 관리 API"},
    ]
)

class UserRole(str, Enum):
    USER = "user"
    ADMIN = "admin"

class UserBase(BaseModel):
    email: EmailStr = Field(..., description="이메일")
    name: str = Field(..., min_length=1, max_length=100, description="이름")

class CreateUserRequest(UserBase):
    password: str = Field(..., min_length=8, description="비밀번호")
    role: UserRole = Field(default=UserRole.USER, description="역할")

    class Config:
        json_schema_extra = {
            "example": {
                "email": "john@example.com",
                "name": "John Doe",
                "password": "securePassword123",
                "role": "user"
            }
        }

class User(UserBase):
    id: str = Field(..., description="사용자 ID")
    role: UserRole
    created_at: str

@app.get("/users", response_model=List[User], tags=["Users"])
async def get_users(
    page: int = Query(1, ge=1, description="페이지 번호"),
    limit: int = Query(20, ge=1, le=100, description="페이지 크기")
):
    """
    사용자 목록을 조회합니다.

    - **page**: 페이지 번호 (1부터 시작)
    - **limit**: 페이지당 항목 수
    """
    pass

@app.post("/users", response_model=User, status_code=201, tags=["Users"])
async def create_user(user: CreateUserRequest = Body(...)):
    """
    새 사용자를 생성합니다.
    """
    pass

# OpenAPI 스펙 커스터마이징
def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    openapi_schema = get_openapi(
        title="My API",
        version="1.0.0",
        description="API 설명",
        routes=app.routes,
    )
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png"
    }
    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

### Spec-First Approach (스펙에서 코드 생성)

```bash
# OpenAPI Generator 설치
npm install @openapitools/openapi-generator-cli -g

# TypeScript 클라이언트 생성
openapi-generator-cli generate \
  -i openapi.yaml \
  -g typescript-axios \
  -o ./generated/client

# Spring Boot 서버 생성
openapi-generator-cli generate \
  -i openapi.yaml \
  -g spring \
  -o ./generated/server \
  --additional-properties=useTags=true,interfaceOnly=true

# Python 클라이언트 생성
openapi-generator-cli generate \
  -i openapi.yaml \
  -g python \
  -o ./generated/python-client
```

---

## Mock Server 구축

### Prism (Stoplight)

```bash
# 설치
npm install -g @stoplight/prism-cli

# Mock 서버 실행
prism mock openapi.yaml

# 동적 응답 (예시 기반)
prism mock openapi.yaml --dynamic

# 특정 포트
prism mock openapi.yaml -p 4010
```

```yaml
# openapi.yaml에 예시 추가
paths:
  /users:
    get:
      responses:
        '200':
          content:
            application/json:
              examples:
                basic:
                  value:
                    data:
                      - id: "1"
                        name: "John Doe"
                        email: "john@example.com"
                    meta:
                      total: 1
                      page: 1
```

### OpenAPI Mock (Node.js)

```javascript
// mock-server.js
const express = require('express');
const { createMockMiddleware } = require('openapi-backend');
const YAML = require('yamljs');

const app = express();
const spec = YAML.load('./openapi.yaml');

// Mock 미들웨어 생성
const mockMiddleware = createMockMiddleware({
  definition: spec,
  handlers: {
    // 커스텀 핸들러
    getUsers: async (c, req, res) => {
      return {
        data: [
          { id: '1', name: 'John Doe', email: 'john@example.com' },
          { id: '2', name: 'Jane Doe', email: 'jane@example.com' },
        ],
        meta: { total: 2, page: 1 }
      };
    },

    // 동적 응답
    getUserById: async (c, req, res) => {
      const { userId } = c.request.params;
      if (userId === 'not-found') {
        return res.status(404).json({
          code: 'NOT_FOUND',
          message: 'User not found'
        });
      }
      return {
        id: userId,
        name: 'Mock User',
        email: 'mock@example.com'
      };
    },

    // 기본 핸들러 (스펙의 예시 사용)
    notImplemented: async (c, req, res) => {
      return res.status(501).json({
        message: 'Not implemented yet'
      });
    }
  }
});

app.use(express.json());
app.use(mockMiddleware);

app.listen(4010, () => {
  console.log('Mock server running on http://localhost:4010');
});
```

### Docker Compose로 개발 환경 구성

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Mock API Server
  mock-api:
    image: stoplight/prism:4
    command: mock -h 0.0.0.0 /tmp/openapi.yaml
    volumes:
      - ./openapi.yaml:/tmp/openapi.yaml
    ports:
      - "4010:4010"

  # Swagger UI
  swagger-ui:
    image: swaggerapi/swagger-ui
    environment:
      SWAGGER_JSON: /openapi.yaml
      VALIDATOR_URL: null
    volumes:
      - ./openapi.yaml:/openapi.yaml
    ports:
      - "8080:8080"

  # Swagger Editor
  swagger-editor:
    image: swaggerapi/swagger-editor
    ports:
      - "8081:8080"
    volumes:
      - ./openapi.yaml:/tmp/openapi.yaml
    environment:
      SWAGGER_FILE: /tmp/openapi.yaml
```

### 검증 서버 (계약 테스트)

```javascript
// contract-test.js
const { OpenAPIBackend } = require('openapi-backend');
const request = require('supertest');

const api = new OpenAPIBackend({ definition: './openapi.yaml' });

// 실제 API 서버 테스트
describe('API Contract Tests', () => {
  beforeAll(async () => {
    await api.init();
  });

  it('GET /users should match OpenAPI spec', async () => {
    const response = await request(apiServer)
      .get('/users')
      .set('Authorization', 'Bearer token');

    // 응답 검증
    const validation = api.validateResponse(
      { status: response.status, body: response.body },
      { method: 'get', path: '/users' }
    );

    expect(validation.errors).toBeUndefined();
  });

  it('POST /users should validate request body', async () => {
    const requestBody = {
      email: 'invalid-email',  // 잘못된 이메일
      name: 'Test'
    };

    const validation = api.validateRequest({
      method: 'POST',
      path: '/users',
      body: requestBody
    });

    expect(validation.errors).toBeDefined();
  });
});
```

---

## 실전 구현 예제

### 완전한 OpenAPI 스펙 예시

```yaml
# openapi.yaml
openapi: 3.0.3

info:
  title: E-Commerce API
  description: |
    전자상거래 플랫폼 API입니다.

    ## 인증
    모든 API는 Bearer 토큰 인증이 필요합니다.
    ```
    Authorization: Bearer <token>
    ```

    ## Rate Limiting
    - 일반 사용자: 100 req/min
    - 프리미엄 사용자: 1000 req/min

    ## 에러 코드
    | 코드 | 설명 |
    |------|------|
    | VALIDATION_ERROR | 입력값 검증 실패 |
    | NOT_FOUND | 리소스 없음 |
    | UNAUTHORIZED | 인증 필요 |
  version: 1.0.0
  contact:
    name: API Support
    email: api@example.com

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://staging-api.example.com/v1
    description: Staging

tags:
  - name: Products
    description: 상품 관리
  - name: Orders
    description: 주문 관리
  - name: Users
    description: 사용자 관리

paths:
  /products:
    get:
      tags: [Products]
      summary: 상품 목록 조회
      operationId: listProducts
      parameters:
        - $ref: '#/components/parameters/PageParam'
        - $ref: '#/components/parameters/LimitParam'
        - name: category
          in: query
          schema:
            type: string
          description: 카테고리 필터
        - name: minPrice
          in: query
          schema:
            type: number
            minimum: 0
        - name: maxPrice
          in: query
          schema:
            type: number
            minimum: 0
        - name: sort
          in: query
          schema:
            type: string
            enum: [price, -price, name, -name, createdAt, -createdAt]
            default: -createdAt
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                allOf:
                  - $ref: '#/components/schemas/PaginatedResponse'
                  - type: object
                    properties:
                      data:
                        type: array
                        items:
                          $ref: '#/components/schemas/Product'
      security: []  # 인증 불필요

    post:
      tags: [Products]
      summary: 상품 등록
      operationId: createProduct
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateProductRequest'
          multipart/form-data:
            schema:
              type: object
              properties:
                data:
                  $ref: '#/components/schemas/CreateProductRequest'
                images:
                  type: array
                  items:
                    type: string
                    format: binary
      responses:
        '201':
          description: 생성 성공
          headers:
            Location:
              schema:
                type: string
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Product'
        '400':
          $ref: '#/components/responses/BadRequest'
      security:
        - bearerAuth: []

  /products/{productId}:
    parameters:
      - name: productId
        in: path
        required: true
        schema:
          type: string
          format: uuid

    get:
      tags: [Products]
      summary: 상품 상세 조회
      operationId: getProduct
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Product'
        '404':
          $ref: '#/components/responses/NotFound'
      security: []

  /orders:
    post:
      tags: [Orders]
      summary: 주문 생성
      operationId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: 주문 생성 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
      security:
        - bearerAuth: []

  /orders/{orderId}:
    get:
      tags: [Orders]
      summary: 주문 조회
      operationId: getOrder
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: 성공
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
      security:
        - bearerAuth: []

components:
  schemas:
    Product:
      type: object
      required: [id, name, price]
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        description:
          type: string
        price:
          type: number
          format: double
          minimum: 0
        category:
          type: string
        images:
          type: array
          items:
            type: string
            format: uri
        stock:
          type: integer
          minimum: 0
        createdAt:
          type: string
          format: date-time

    CreateProductRequest:
      type: object
      required: [name, price, category]
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 200
        description:
          type: string
          maxLength: 5000
        price:
          type: number
          minimum: 0
        category:
          type: string
        stock:
          type: integer
          minimum: 0
          default: 0

    Order:
      type: object
      properties:
        id:
          type: string
          format: uuid
        userId:
          type: string
          format: uuid
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        status:
          type: string
          enum: [pending, paid, shipped, delivered, cancelled]
        totalAmount:
          type: number
        createdAt:
          type: string
          format: date-time

    OrderItem:
      type: object
      properties:
        productId:
          type: string
          format: uuid
        quantity:
          type: integer
          minimum: 1
        unitPrice:
          type: number

    CreateOrderRequest:
      type: object
      required: [items]
      properties:
        items:
          type: array
          minItems: 1
          items:
            type: object
            required: [productId, quantity]
            properties:
              productId:
                type: string
                format: uuid
              quantity:
                type: integer
                minimum: 1
        shippingAddress:
          $ref: '#/components/schemas/Address'

    Address:
      type: object
      properties:
        street:
          type: string
        city:
          type: string
        postalCode:
          type: string
        country:
          type: string

    PaginatedResponse:
      type: object
      properties:
        meta:
          type: object
          properties:
            page:
              type: integer
            limit:
              type: integer
            total:
              type: integer
            totalPages:
              type: integer

    Error:
      type: object
      required: [code, message]
      properties:
        code:
          type: string
        message:
          type: string
        details:
          type: array
          items:
            type: object

  parameters:
    PageParam:
      name: page
      in: query
      schema:
        type: integer
        minimum: 1
        default: 1

    LimitParam:
      name: limit
      in: query
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20

  responses:
    BadRequest:
      description: 잘못된 요청
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

    NotFound:
      description: 리소스 없음
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

---

## 참고 자료

- [OpenAPI Specification 3.0](https://spec.openapis.org/oas/v3.0.3)
- [OpenAPI Specification 3.1](https://spec.openapis.org/oas/v3.1.0)
- [Swagger Documentation](https://swagger.io/docs/)
- [OpenAPI Generator](https://openapi-generator.tech/)
- [Stoplight Prism](https://stoplight.io/open-source/prism)
- [springdoc-openapi](https://springdoc.org/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)

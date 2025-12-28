# gRPC

## 목차
1. [gRPC 개요](#grpc-개요)
2. [Protocol Buffers 문법](#protocol-buffers-문법)
3. [4가지 통신 패턴](#4가지-통신-패턴)
4. [인터셉터 (Interceptor)](#인터셉터-interceptor)
5. [gRPC vs REST 비교](#grpc-vs-rest-비교)
6. [실전 구현 예제](#실전-구현-예제)
---

## gRPC 개요

gRPC는 Google이 개발한 고성능, 오픈소스 원격 프로시저 호출(RPC) 프레임워크입니다. HTTP/2를 전송 프로토콜로, Protocol Buffers를 인터페이스 정의 언어(IDL)로 사용합니다.

### 핵심 특징

```
┌─────────────────────────────────────────────────────────┐
│                      gRPC 아키텍처                       │
├─────────────────────────────────────────────────────────┤
│  Client Application                                      │
│       ↓ (Generated Stub)                                │
│  gRPC Client                                            │
│       ↓ (HTTP/2)                                        │
│  ═══════════════════ Network ═══════════════════        │
│       ↓ (HTTP/2)                                        │
│  gRPC Server                                            │
│       ↓ (Generated Skeleton)                            │
│  Server Application                                      │
└─────────────────────────────────────────────────────────┘
```

| 특징 | 설명 |
|------|------|
| HTTP/2 기반 | 멀티플렉싱, 헤더 압축, 양방향 스트리밍 |
| Protocol Buffers | 언어 중립적인 바이너리 직렬화 |
| 코드 생성 | .proto 파일에서 클라이언트/서버 코드 자동 생성 |
| 양방향 스트리밍 | 클라이언트-서버 간 실시간 데이터 교환 |
| 다중 언어 지원 | C++, Java, Python, Go, Node.js, C#, Ruby 등 |

### gRPC vs REST 간단 비교

```
REST (HTTP/1.1 + JSON):
┌──────────────────────────────────────────┐
│ POST /users                              │
│ Content-Type: application/json           │
│ { "name": "John", "email": "..." }       │
│                                          │
│ → 텍스트 기반, 사람이 읽기 쉬움            │
│ → 상대적으로 큰 페이로드                   │
└──────────────────────────────────────────┘

gRPC (HTTP/2 + Protobuf):
┌──────────────────────────────────────────┐
│ CreateUser(UserRequest)                  │
│ → 바이너리 직렬화, 작은 페이로드            │
│ → 타입 안전, 코드 자동 생성                │
│ → 스트리밍 네이티브 지원                   │
└──────────────────────────────────────────┘
```

---

## Protocol Buffers 문법

Protocol Buffers(Protobuf)는 구조화된 데이터를 직렬화하기 위한 언어 중립적 메커니즘입니다.

### 기본 구조

```protobuf
// user.proto
syntax = "proto3";  // Protobuf 버전 지정

package user;  // 패키지 네임스페이스

option go_package = "github.com/example/user";
option java_package = "com.example.user";
option java_multiple_files = true;

// 메시지 정의
message User {
  int64 id = 1;           // 필드 번호 (1~536,870,911, 19000-19999 예약)
  string name = 2;
  string email = 3;
  UserStatus status = 4;
  repeated string tags = 5;  // 배열
  optional string bio = 6;   // 선택적 필드 (proto3)
  map<string, string> metadata = 7;  // 맵
}

// 열거형
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;  // 첫 번째 값은 0이어야 함
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_BANNED = 3;
}
```

### 스칼라 타입

```protobuf
message ScalarTypes {
  // 정수 타입
  int32 signed_int = 1;      // -2^31 ~ 2^31-1
  int64 signed_long = 2;     // -2^63 ~ 2^63-1
  uint32 unsigned_int = 3;   // 0 ~ 2^32-1
  uint64 unsigned_long = 4;  // 0 ~ 2^64-1
  sint32 signed_int_opt = 5; // 음수에 효율적
  sint64 signed_long_opt = 6;
  fixed32 fixed_int = 7;     // 항상 4바이트 (큰 값에 효율적)
  fixed64 fixed_long = 8;    // 항상 8바이트

  // 부동소수점
  float single_precision = 9;
  double double_precision = 10;

  // 불리언
  bool is_active = 11;

  // 문자열/바이트
  string text = 12;          // UTF-8 또는 ASCII
  bytes binary_data = 13;    // 임의의 바이트 시퀀스
}
```

### 복합 타입

```protobuf
// 중첩 메시지
message Order {
  int64 id = 1;
  Customer customer = 2;
  repeated OrderItem items = 3;
  Address shipping_address = 4;

  // 중첩 정의
  message OrderItem {
    int64 product_id = 1;
    int32 quantity = 2;
    double price = 3;
  }
}

message Customer {
  int64 id = 1;
  string name = 2;
}

message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

// oneof - 여러 필드 중 하나만 설정
message Payment {
  int64 order_id = 1;
  double amount = 2;

  oneof payment_method {
    CreditCard credit_card = 3;
    BankTransfer bank_transfer = 4;
    PayPal paypal = 5;
  }
}

message CreditCard {
  string card_number = 1;
  string expiry = 2;
  string cvv = 3;
}

message BankTransfer {
  string account_number = 1;
  string bank_code = 2;
}

message PayPal {
  string email = 1;
}
```

### Well-Known Types

```protobuf
import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";
import "google/protobuf/empty.proto";
import "google/protobuf/wrappers.proto";
import "google/protobuf/any.proto";

message Event {
  string id = 1;
  google.protobuf.Timestamp created_at = 2;  // 타임스탬프
  google.protobuf.Duration duration = 3;      // 기간
  google.protobuf.StringValue nullable_name = 4;  // nullable 래퍼
  google.protobuf.Any payload = 5;            // 동적 타입
}

// Timestamp 사용 예시 (Go)
// created_at := timestamppb.Now()
// created_at := timestamppb.New(time.Now())

// Duration 사용 예시
// duration := durationpb.New(time.Hour * 2)
```

### 서비스 정의

```protobuf
// user_service.proto
syntax = "proto3";

package user.v1;

import "google/protobuf/empty.proto";
import "google/protobuf/field_mask.proto";

service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);

  // Server Streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);

  // Client Streaming
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateUsersResponse);

  // Bidirectional Streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);

  // 업데이트 (Partial Update with FieldMask)
  rpc UpdateUser(UpdateUserRequest) returns (User);

  // 삭제
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
}

message GetUserRequest {
  int64 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  string filter = 3;  // e.g., "status=ACTIVE"
}

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask update_mask = 2;  // 업데이트할 필드 지정
}

message DeleteUserRequest {
  int64 id = 1;
}

message BatchCreateUsersResponse {
  repeated User users = 1;
  int32 created_count = 2;
}

message ChatMessage {
  string user_id = 1;
  string content = 2;
  int64 timestamp = 3;
}
```

### 필드 번호 규칙

```protobuf
message FieldNumberRules {
  // 1-15: 1바이트로 인코딩 (자주 사용하는 필드에 할당)
  int64 id = 1;
  string name = 2;

  // 16-2047: 2바이트로 인코딩
  string description = 16;

  // 19000-19999: 예약됨 (사용 불가)

  // 필드 번호 예약 (하위 호환성)
  reserved 4, 5, 100 to 200;
  reserved "old_field", "deprecated_field";
}
```

---

## 4가지 통신 패턴

gRPC는 4가지 서비스 메서드 유형을 지원합니다.

### 1. Unary RPC (단항 RPC)

가장 기본적인 형태로, 일반 함수 호출과 유사합니다.

```protobuf
// 정의
rpc GetUser(GetUserRequest) returns (User);
```

```
Client                          Server
   │                               │
   │──── GetUserRequest ──────────>│
   │                               │
   │<─────── User ─────────────────│
   │                               │
```

**Go 서버 구현:**

```go
func (s *UserServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    // 요청 검증
    if req.Id <= 0 {
        return nil, status.Errorf(codes.InvalidArgument, "invalid user id: %d", req.Id)
    }

    // 데이터 조회
    user, err := s.repo.FindByID(ctx, req.Id)
    if err != nil {
        if errors.Is(err, repository.ErrNotFound) {
            return nil, status.Errorf(codes.NotFound, "user not found: %d", req.Id)
        }
        return nil, status.Errorf(codes.Internal, "failed to get user: %v", err)
    }

    return user.ToProto(), nil
}
```

**Go 클라이언트 호출:**

```go
func getUser(client pb.UserServiceClient, userID int64) (*pb.User, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    req := &pb.GetUserRequest{Id: userID}
    return client.GetUser(ctx, req)
}
```

### 2. Server Streaming RPC (서버 스트리밍)

클라이언트가 요청을 보내면 서버가 스트림으로 여러 응답을 보냅니다.

```protobuf
// 정의
rpc ListUsers(ListUsersRequest) returns (stream User);
```

```
Client                          Server
   │                               │
   │──── ListUsersRequest ────────>│
   │                               │
   │<─────── User 1 ───────────────│
   │<─────── User 2 ───────────────│
   │<─────── User 3 ───────────────│
   │<─────── ... ──────────────────│
   │<─────── (stream end) ─────────│
   │                               │
```

**서버 구현:**

```go
func (s *UserServer) ListUsers(req *pb.ListUsersRequest, stream pb.UserService_ListUsersServer) error {
    // 사용자 목록 조회
    users, err := s.repo.FindAll(stream.Context())
    if err != nil {
        return status.Errorf(codes.Internal, "failed to list users: %v", err)
    }

    for _, user := range users {
        // 컨텍스트 취소 확인
        if err := stream.Context().Err(); err != nil {
            return status.Errorf(codes.Canceled, "client canceled: %v", err)
        }

        // 각 사용자를 스트림으로 전송
        if err := stream.Send(user.ToProto()); err != nil {
            return status.Errorf(codes.Internal, "failed to send user: %v", err)
        }
    }

    return nil
}
```

**클라이언트 구현:**

```go
func listUsers(client pb.UserServiceClient) error {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    stream, err := client.ListUsers(ctx, &pb.ListUsersRequest{})
    if err != nil {
        return fmt.Errorf("failed to call ListUsers: %w", err)
    }

    for {
        user, err := stream.Recv()
        if err == io.EOF {
            break  // 스트림 종료
        }
        if err != nil {
            return fmt.Errorf("failed to receive user: %w", err)
        }

        fmt.Printf("Received user: %s\n", user.Name)
    }

    return nil
}
```

**사용 사례:**
- 대용량 데이터 목록 조회
- 실시간 로그 스트리밍
- 파일 다운로드
- 주식 시세 피드

### 3. Client Streaming RPC (클라이언트 스트리밍)

클라이언트가 스트림으로 여러 요청을 보내고 서버가 단일 응답을 반환합니다.

```protobuf
// 정의
rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateUsersResponse);
```

```
Client                          Server
   │                               │
   │──── CreateUserRequest 1 ─────>│
   │──── CreateUserRequest 2 ─────>│
   │──── CreateUserRequest 3 ─────>│
   │──── (stream end) ────────────>│
   │                               │
   │<── BatchCreateUsersResponse ──│
   │                               │
```

**서버 구현:**

```go
func (s *UserServer) BatchCreateUsers(stream pb.UserService_BatchCreateUsersServer) error {
    var users []*model.User

    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // 모든 요청 수신 완료, 일괄 처리
            createdUsers, err := s.repo.CreateMany(stream.Context(), users)
            if err != nil {
                return status.Errorf(codes.Internal, "failed to create users: %v", err)
            }

            // 응답 전송
            return stream.SendAndClose(&pb.BatchCreateUsersResponse{
                Users:        toProtoUsers(createdUsers),
                CreatedCount: int32(len(createdUsers)),
            })
        }
        if err != nil {
            return status.Errorf(codes.Internal, "failed to receive: %v", err)
        }

        // 요청을 수집
        users = append(users, &model.User{
            Name:  req.Name,
            Email: req.Email,
        })
    }
}
```

**클라이언트 구현:**

```go
func batchCreateUsers(client pb.UserServiceClient, users []*pb.CreateUserRequest) error {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    stream, err := client.BatchCreateUsers(ctx)
    if err != nil {
        return fmt.Errorf("failed to create stream: %w", err)
    }

    for _, user := range users {
        if err := stream.Send(user); err != nil {
            return fmt.Errorf("failed to send user: %w", err)
        }
    }

    // 스트림 종료 및 응답 수신
    response, err := stream.CloseAndRecv()
    if err != nil {
        return fmt.Errorf("failed to receive response: %w", err)
    }

    fmt.Printf("Created %d users\n", response.CreatedCount)
    return nil
}
```

**사용 사례:**
- 대량 데이터 일괄 업로드
- 파일 업로드
- 센서 데이터 집계

### 4. Bidirectional Streaming RPC (양방향 스트리밍)

클라이언트와 서버가 독립적으로 데이터를 스트리밍합니다.

```protobuf
// 정의
rpc Chat(stream ChatMessage) returns (stream ChatMessage);
```

```
Client                          Server
   │                               │
   │──── ChatMessage 1 ───────────>│
   │<──── ChatMessage A ───────────│
   │──── ChatMessage 2 ───────────>│
   │──── ChatMessage 3 ───────────>│
   │<──── ChatMessage B ───────────│
   │<──── ChatMessage C ───────────│
   │                               │
   (양쪽이 독립적으로 스트리밍)
```

**서버 구현:**

```go
func (s *ChatServer) Chat(stream pb.ChatService_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return status.Errorf(codes.Internal, "failed to receive: %v", err)
        }

        // 메시지 처리 및 브로드캐스트
        log.Printf("Received from %s: %s", msg.UserId, msg.Content)

        // 응답 전송 (에코 또는 처리된 결과)
        response := &pb.ChatMessage{
            UserId:    "server",
            Content:   fmt.Sprintf("Echo: %s", msg.Content),
            Timestamp: time.Now().Unix(),
        }

        if err := stream.Send(response); err != nil {
            return status.Errorf(codes.Internal, "failed to send: %v", err)
        }
    }
}
```

**클라이언트 구현:**

```go
func chat(client pb.ChatServiceClient) error {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    stream, err := client.Chat(ctx)
    if err != nil {
        return fmt.Errorf("failed to create stream: %w", err)
    }

    // 수신 고루틴
    go func() {
        for {
            msg, err := stream.Recv()
            if err == io.EOF {
                return
            }
            if err != nil {
                log.Printf("Receive error: %v", err)
                return
            }
            fmt.Printf("Server: %s\n", msg.Content)
        }
    }()

    // 송신
    messages := []string{"Hello", "How are you?", "Goodbye"}
    for _, content := range messages {
        msg := &pb.ChatMessage{
            UserId:    "client-1",
            Content:   content,
            Timestamp: time.Now().Unix(),
        }
        if err := stream.Send(msg); err != nil {
            return fmt.Errorf("failed to send: %w", err)
        }
        time.Sleep(time.Second)
    }

    stream.CloseSend()
    time.Sleep(2 * time.Second)  // 남은 응답 수신 대기
    return nil
}
```

**사용 사례:**
- 실시간 채팅
- 게임 서버 통신
- 협업 도구 (동시 편집)
- IoT 디바이스 양방향 통신

---

## 인터셉터 (Interceptor)

인터셉터는 gRPC의 미들웨어로, 요청/응답 처리 전후에 로직을 추가합니다.

### 인터셉터 타입

```
┌─────────────────────────────────────────────────────────┐
│                    Interceptor Types                     │
├─────────────────────────────────────────────────────────┤
│  Server Side                                            │
│  ├── UnaryServerInterceptor    (단항 RPC용)              │
│  └── StreamServerInterceptor   (스트리밍 RPC용)          │
│                                                         │
│  Client Side                                            │
│  ├── UnaryClientInterceptor    (단항 RPC용)              │
│  └── StreamClientInterceptor   (스트리밍 RPC용)          │
└─────────────────────────────────────────────────────────┘
```

### Unary 인터셉터 구현

```go
// 서버 사이드 - 로깅 인터셉터
func loggingUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()

    // 핸들러 실행 (실제 RPC 호출)
    resp, err := handler(ctx, req)

    // 로깅
    duration := time.Since(start)
    log.Printf(
        "method=%s duration=%v err=%v",
        info.FullMethod,
        duration,
        err,
    )

    return resp, err
}

// 서버 사이드 - 인증 인터셉터
func authUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // 메타데이터에서 토큰 추출
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing token")
    }

    token := strings.TrimPrefix(tokens[0], "Bearer ")

    // 토큰 검증
    user, err := validateToken(token)
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }

    // 컨텍스트에 사용자 정보 추가
    ctx = context.WithValue(ctx, userContextKey, user)

    return handler(ctx, req)
}

// 서버 사이드 - 복구 인터셉터 (패닉 처리)
func recoveryUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (resp interface{}, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("panic recovered: %v\nstack: %s", r, debug.Stack())
            err = status.Errorf(codes.Internal, "internal error")
        }
    }()

    return handler(ctx, req)
}
```

### Stream 인터셉터 구현

```go
// 서버 스트림 인터셉터
func loggingStreamInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    start := time.Now()

    // 래핑된 스트림으로 메시지 로깅
    wrappedStream := &loggingServerStream{ServerStream: ss}

    err := handler(srv, wrappedStream)

    log.Printf(
        "stream method=%s duration=%v sent=%d received=%d err=%v",
        info.FullMethod,
        time.Since(start),
        wrappedStream.sentCount,
        wrappedStream.receivedCount,
        err,
    )

    return err
}

type loggingServerStream struct {
    grpc.ServerStream
    sentCount     int
    receivedCount int
}

func (s *loggingServerStream) SendMsg(m interface{}) error {
    s.sentCount++
    return s.ServerStream.SendMsg(m)
}

func (s *loggingServerStream) RecvMsg(m interface{}) error {
    err := s.ServerStream.RecvMsg(m)
    if err == nil {
        s.receivedCount++
    }
    return err
}
```

### 클라이언트 인터셉터

```go
// 클라이언트 - 재시도 인터셉터
func retryUnaryInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    maxRetries := 3
    backoff := 100 * time.Millisecond

    var lastErr error
    for i := 0; i < maxRetries; i++ {
        err := invoker(ctx, method, req, reply, cc, opts...)
        if err == nil {
            return nil
        }

        // 재시도 가능한 에러인지 확인
        code := status.Code(err)
        if code != codes.Unavailable && code != codes.DeadlineExceeded {
            return err
        }

        lastErr = err
        log.Printf("Retry %d/%d for %s: %v", i+1, maxRetries, method, err)

        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(backoff):
            backoff *= 2  // 지수 백오프
        }
    }

    return lastErr
}

// 클라이언트 - 메타데이터 추가 인터셉터
func metadataUnaryInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    // 메타데이터 추가
    md := metadata.Pairs(
        "x-request-id", uuid.New().String(),
        "x-client-version", "1.0.0",
    )
    ctx = metadata.NewOutgoingContext(ctx, md)

    return invoker(ctx, method, req, reply, cc, opts...)
}
```

### 인터셉터 체이닝

```go
import "google.golang.org/grpc/middleware"

// 서버에 여러 인터셉터 적용
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryUnaryInterceptor,  // 가장 바깥쪽 (먼저 실행)
        loggingUnaryInterceptor,
        authUnaryInterceptor,
        rateLimitUnaryInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        recoveryStreamInterceptor,
        loggingStreamInterceptor,
        authStreamInterceptor,
    ),
)

// 클라이언트에 인터셉터 적용
conn, err := grpc.Dial(
    address,
    grpc.WithTransportCredentials(insecure.NewCredentials()),
    grpc.WithChainUnaryInterceptor(
        metadataUnaryInterceptor,
        retryUnaryInterceptor,
        loggingClientInterceptor,
    ),
)
```

### go-grpc-middleware 라이브러리

```go
import (
    grpc_middleware "github.com/grpc-ecosystem/go-grpc-middleware"
    grpc_auth "github.com/grpc-ecosystem/go-grpc-middleware/auth"
    grpc_zap "github.com/grpc-ecosystem/go-grpc-middleware/logging/zap"
    grpc_recovery "github.com/grpc-ecosystem/go-grpc-middleware/recovery"
    grpc_validator "github.com/grpc-ecosystem/go-grpc-middleware/validator"
    grpc_ratelimit "github.com/grpc-ecosystem/go-grpc-middleware/ratelimit"
)

server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        grpc_recovery.UnaryServerInterceptor(),
        grpc_zap.UnaryServerInterceptor(logger),
        grpc_auth.UnaryServerInterceptor(authFunc),
        grpc_validator.UnaryServerInterceptor(),
        grpc_ratelimit.UnaryServerInterceptor(limiter),
    ),
)
```

---

## gRPC vs REST 비교

### 상세 비교표

| 측면 | gRPC | REST |
|------|------|------|
| **프로토콜** | HTTP/2 | HTTP/1.1 (주로) |
| **데이터 형식** | Protocol Buffers (바이너리) | JSON/XML (텍스트) |
| **API 계약** | .proto 파일 (강력한 타입) | OpenAPI/Swagger (선택적) |
| **코드 생성** | 자동 생성 필수 | 선택적 |
| **스트리밍** | 네이티브 지원 (4가지 패턴) | 별도 구현 필요 |
| **브라우저 지원** | gRPC-Web 필요 | 네이티브 지원 |
| **캐싱** | 별도 구현 필요 | HTTP 캐싱 활용 |
| **디버깅** | 전문 도구 필요 | curl, 브라우저로 가능 |
| **성능** | 높음 (10배 이상 빠를 수 있음) | 상대적으로 낮음 |
| **학습 곡선** | 높음 | 낮음 |

### 성능 비교

```
페이로드 크기 비교 (동일 데이터):
┌─────────────────────────────────────────┐
│ JSON:     {"id":123,"name":"John"}      │
│           → 26 bytes                    │
│                                         │
│ Protobuf: [binary encoded]              │
│           → 8 bytes (약 70% 감소)        │
└─────────────────────────────────────────┘

직렬화/역직렬화 속도:
┌─────────────────────────────────────────┐
│ JSON:     직렬화 100ms, 역직렬화 120ms   │
│ Protobuf: 직렬화 20ms, 역직렬화 25ms     │
│           (약 5배 빠름)                  │
└─────────────────────────────────────────┘
```

### HTTP/2 장점

```
HTTP/1.1:
┌─────────────────────────────────────────┐
│ Request 1  ──────────────────>          │
│            <────────────────── Response │
│ Request 2  ──────────────────>          │
│            <────────────────── Response │
│ (Head-of-line blocking)                 │
└─────────────────────────────────────────┘

HTTP/2 (Multiplexing):
┌─────────────────────────────────────────┐
│ Request 1  ──────>                      │
│ Request 2  ──────>  (동시 전송)          │
│ Request 3  ──────>                      │
│            <────── Response 2           │
│            <────── Response 1           │
│            <────── Response 3           │
└─────────────────────────────────────────┘
```

### 언제 무엇을 선택할까?

**gRPC가 적합한 경우:**
- 마이크로서비스 간 내부 통신
- 실시간 양방향 통신 필요
- 높은 성능과 낮은 지연시간 요구
- 다중 언어 환경
- 타입 안전성이 중요한 경우

**REST가 적합한 경우:**
- 퍼블릭 API (브라우저 클라이언트)
- 단순한 CRUD 작업
- HTTP 캐싱이 중요한 경우
- 빠른 프로토타이핑
- 디버깅 용이성이 중요한 경우

**하이브리드 접근:**
```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway                           │
│                 (REST/GraphQL)                          │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│   Microservice A ←──gRPC──→ Microservice B              │
│         ↓                         ↓                     │
│   Microservice C ←──gRPC──→ Microservice D              │
└─────────────────────────────────────────────────────────┘
```

---

## 실전 구현 예제

### 프로젝트 구조

```
grpc-example/
├── proto/
│   ├── user.proto
│   └── order.proto
├── pb/                    # 생성된 코드
│   ├── user.pb.go
│   └── user_grpc.pb.go
├── server/
│   ├── main.go
│   └── handlers/
│       └── user.go
├── client/
│   └── main.go
├── Makefile
└── go.mod
```

### Makefile

```makefile
.PHONY: proto clean

proto:
	protoc --go_out=. --go_opt=paths=source_relative \
		--go-grpc_out=. --go-grpc_opt=paths=source_relative \
		proto/*.proto

clean:
	rm -rf pb/*.go

server:
	go run server/main.go

client:
	go run client/main.go
```

### 서버 전체 구현

```go
// server/main.go
package main

import (
	"context"
	"log"
	"net"
	"os"
	"os/signal"
	"syscall"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials"
	"google.golang.org/grpc/health"
	"google.golang.org/grpc/health/grpc_health_v1"
	"google.golang.org/grpc/reflection"

	pb "github.com/example/pb"
)

func main() {
	// TLS 설정 (프로덕션)
	creds, err := credentials.NewServerTLSFromFile("cert.pem", "key.pem")
	if err != nil {
		log.Fatalf("Failed to load TLS: %v", err)
	}

	// 서버 옵션
	opts := []grpc.ServerOption{
		grpc.Creds(creds),
		grpc.ChainUnaryInterceptor(
			recoveryInterceptor,
			loggingInterceptor,
			authInterceptor,
		),
		grpc.MaxRecvMsgSize(10 * 1024 * 1024), // 10MB
		grpc.MaxSendMsgSize(10 * 1024 * 1024),
	}

	server := grpc.NewServer(opts...)

	// 서비스 등록
	userServer := NewUserServer()
	pb.RegisterUserServiceServer(server, userServer)

	// Health 체크
	healthServer := health.NewServer()
	grpc_health_v1.RegisterHealthServer(server, healthServer)
	healthServer.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING)

	// 리플렉션 (개발용)
	reflection.Register(server)

	// 리스너 시작
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("Failed to listen: %v", err)
	}

	// Graceful shutdown
	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
		<-sigCh

		log.Println("Shutting down gracefully...")
		server.GracefulStop()
	}()

	log.Printf("Server listening on :50051")
	if err := server.Serve(lis); err != nil {
		log.Fatalf("Failed to serve: %v", err)
	}
}
```

### 에러 처리 패턴

```go
import (
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

func (s *UserServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
	// 입력 검증
	if req.Id <= 0 {
		return nil, status.Error(codes.InvalidArgument, "user id must be positive")
	}

	user, err := s.repo.FindByID(ctx, req.Id)
	if err != nil {
		// 에러 타입에 따른 적절한 상태 코드 반환
		switch {
		case errors.Is(err, ErrNotFound):
			return nil, status.Error(codes.NotFound, "user not found")
		case errors.Is(err, ErrPermissionDenied):
			return nil, status.Error(codes.PermissionDenied, "access denied")
		case errors.Is(err, context.DeadlineExceeded):
			return nil, status.Error(codes.DeadlineExceeded, "request timeout")
		default:
			// 내부 에러는 상세 정보 노출 X
			log.Printf("Internal error: %v", err)
			return nil, status.Error(codes.Internal, "internal error")
		}
	}

	return user.ToProto(), nil
}

// gRPC 상태 코드 가이드
// codes.OK                  - 성공
// codes.Canceled            - 클라이언트가 취소
// codes.Unknown             - 알 수 없는 에러
// codes.InvalidArgument     - 잘못된 인수
// codes.DeadlineExceeded    - 타임아웃
// codes.NotFound            - 리소스 없음
// codes.AlreadyExists       - 이미 존재
// codes.PermissionDenied    - 권한 없음
// codes.Unauthenticated     - 인증 필요
// codes.ResourceExhausted   - 리소스 고갈 (rate limit)
// codes.FailedPrecondition  - 사전 조건 실패
// codes.Aborted             - 작업 중단
// codes.OutOfRange          - 범위 초과
// codes.Unimplemented       - 미구현
// codes.Internal            - 내부 에러
// codes.Unavailable         - 서비스 이용 불가
// codes.DataLoss            - 데이터 손실
```

---

## 참고 자료

- [gRPC 공식 문서](https://grpc.io/docs/)
- [Protocol Buffers 가이드](https://developers.google.com/protocol-buffers/docs/proto3)
- [gRPC-Go GitHub](https://github.com/grpc/grpc-go)
- [go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [gRPC Concepts - Core Concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)

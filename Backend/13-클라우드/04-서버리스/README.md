# 서버리스 아키텍처 (Serverless Architecture)

## 목차
1. [개요](#개요)
2. [FaaS (Function as a Service)](#faas-function-as-a-service)
3. [Cold Start 문제와 해결책](#cold-start-문제와-해결책)
4. [서버리스 패턴](#서버리스-패턴)
5. [장단점과 적합한 사용 사례](#장단점과-적합한-사용-사례)
6. [Serverless Framework](#serverless-framework)
---

## 개요

서버리스(Serverless)는 클라우드 제공업체가 서버 관리를 완전히 담당하고, 개발자는 코드 작성에만 집중할 수 있는 클라우드 컴퓨팅 실행 모델입니다.

### 서버리스의 핵심 개념
```
┌─────────────────────────────────────────────────────────────────┐
│                    Serverless Computing                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  "서버가 없다"는 것이 아님!                                      │
│  → 서버 관리가 개발자에게서 추상화됨                             │
│                                                                  │
│  핵심 특징:                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 1. 자동 스케일링                                         │    │
│  │    - 0에서 무한대까지 자동 확장/축소                     │    │
│  │    - 요청 없으면 0으로 스케일 다운                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 2. 사용량 기반 과금                                      │    │
│  │    - 실행 시간만큼만 비용 지불                           │    │
│  │    - 유휴 시간 비용 없음                                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 3. 이벤트 기반 실행                                      │    │
│  │    - HTTP 요청, 메시지, 스케줄 등                        │    │
│  │    - 이벤트 발생 시에만 실행                             │    │
│  └─────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 4. 무상태(Stateless)                                     │    │
│  │    - 각 실행은 독립적                                    │    │
│  │    - 상태는 외부 저장소에 보관                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 서버리스 스펙트럼
```
┌─────────────────────────────────────────────────────────────────┐
│                   Serverless Spectrum                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  더 많은 제어 ◄──────────────────────────────► 더 많은 추상화   │
│                                                                  │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │
│  │   VM   │  │Container│ │ CaaS   │  │ FaaS   │  │ BaaS   │    │
│  │ (EC2)  │  │(Docker) │ │(Fargate│  │(Lambda)│  │(Firebase│    │
│  │        │  │         │ │CloudRun│  │        │  │Cognito)│    │
│  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘    │
│                                                                  │
│  IaaS          ←          PaaS          →          Serverless   │
│                                                                  │
│  서버리스 컴퓨팅 서비스:                                         │
│  - AWS: Lambda, Fargate, Aurora Serverless                      │
│  - GCP: Cloud Functions, Cloud Run, BigQuery                   │
│  - Azure: Functions, Container Apps, Cosmos DB Serverless      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## FaaS (Function as a Service)

FaaS는 서버리스 컴퓨팅의 핵심으로, 개별 함수 단위로 코드를 실행합니다.

### 주요 FaaS 서비스 비교
```
┌──────────────────┬─────────────────┬─────────────────┬─────────────────┐
│      특성        │   AWS Lambda    │ Cloud Functions │ Azure Functions │
├──────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 최대 실행 시간   │ 15분            │ 60분 (2nd Gen)  │ 무제한 (Premium)│
│ 메모리           │ 128MB - 10GB    │ 128MB - 32GB    │ 128MB - 32GB    │
│ 동시 실행        │ 1,000 (기본)    │ 제한 없음       │ 200 (기본)      │
│ Cold Start       │ 보통            │ 약간 길음       │ 보통            │
│ 언어 지원        │ 다양            │ 다양            │ 다양            │
│ 컨테이너 지원    │ 최대 10GB       │ Cloud Run       │ 지원            │
│ 무료 티어        │ 100만 요청/월   │ 200만 요청/월   │ 100만 요청/월   │
└──────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### FaaS 실행 모델
```
┌─────────────────────────────────────────────────────────────────────┐
│                      FaaS Execution Model                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  이벤트 소스                함수 실행                               │
│  ┌──────────┐                                                       │
│  │  HTTP    │──┐                                                    │
│  └──────────┘  │        ┌─────────────────────────────────┐        │
│  ┌──────────┐  │        │       Function Container        │        │
│  │  Queue   │──┼───────►│  ┌───────────────────────────┐  │        │
│  └──────────┘  │        │  │    Your Code (Handler)    │  │        │
│  ┌──────────┐  │        │  │                           │  │        │
│  │  Stream  │──┘        │  │  - Event Processing       │  │        │
│  └──────────┘           │  │  - Business Logic         │  │        │
│  ┌──────────┐           │  │  - External API Calls     │  │        │
│  │  Timer   │──────────►│  │                           │  │        │
│  └──────────┘           │  └───────────────────────────┘  │        │
│  ┌──────────┐           │                                 │        │
│  │  Storage │──────────►│  Runtime: Node.js/Python/Java   │        │
│  └──────────┘           │  Dependencies: node_modules/    │        │
│  ┌──────────┐           │                                 │        │
│  │   DB     │──────────►│  Cold Start: 컨테이너 초기화    │        │
│  │ Trigger  │           │  Warm Start: 재사용             │        │
│  └──────────┘           └─────────────────────────────────┘        │
│                                                                      │
│  수명 주기:                                                          │
│  1. 이벤트 수신                                                     │
│  2. 컨테이너 프로비저닝 (Cold Start) 또는 재사용 (Warm)            │
│  3. 런타임 초기화                                                   │
│  4. 핸들러 함수 실행                                                │
│  5. 응답 반환                                                       │
│  6. 컨테이너 유지 (일정 시간) 또는 종료                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 이벤트 소스 유형
```python
# AWS Lambda - 다양한 이벤트 소스 예시

# 1. API Gateway (HTTP)
def api_handler(event, context):
    """REST API 요청 처리"""
    http_method = event['httpMethod']
    path = event['path']
    body = json.loads(event.get('body', '{}'))

    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'message': 'Success'})
    }

# 2. S3 Event
def s3_handler(event, context):
    """S3 객체 업로드 처리"""
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        print(f"Processing: s3://{bucket}/{key}")
        # 이미지 리사이징, 메타데이터 추출 등

# 3. SQS Queue
def sqs_handler(event, context):
    """SQS 메시지 처리"""
    for record in event['Records']:
        message = json.loads(record['body'])
        # 메시지 처리 로직

    # 성공 시 메시지 자동 삭제 (batch failure 설정 가능)

# 4. DynamoDB Stream
def dynamodb_handler(event, context):
    """DynamoDB 변경 이벤트 처리"""
    for record in event['Records']:
        event_name = record['eventName']  # INSERT, MODIFY, REMOVE

        if event_name == 'INSERT':
            new_image = record['dynamodb']['NewImage']
            # 신규 레코드 처리
        elif event_name == 'MODIFY':
            old_image = record['dynamodb']['OldImage']
            new_image = record['dynamodb']['NewImage']
            # 변경 사항 처리

# 5. EventBridge (CloudWatch Events)
def scheduled_handler(event, context):
    """스케줄 기반 실행 (Cron)"""
    print(f"Scheduled execution at: {event['time']}")
    # 배치 작업, 정리 작업 등

# 6. Kinesis Stream
def kinesis_handler(event, context):
    """실시간 스트리밍 데이터 처리"""
    for record in event['Records']:
        payload = base64.b64decode(record['kinesis']['data'])
        data = json.loads(payload)
        # 실시간 분석, 집계 등
```

---

## Cold Start 문제와 해결책

Cold Start는 서버리스 환경에서 가장 중요한 성능 이슈 중 하나입니다.

### Cold Start란?
```
┌─────────────────────────────────────────────────────────────────────┐
│                      Cold Start vs Warm Start                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Cold Start (첫 실행 또는 스케일 아웃):                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. 컨테이너   │ 2. 런타임    │ 3. 코드 로드 │ 4. 핸들러   │    │
│  │    프로비저닝 │    초기화    │    및 초기화 │    실행     │    │
│  │    (100ms+)  │    (100ms+)  │    (100ms+)  │   (변동)    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  |◄─────────── Cold Start 지연시간 (수백ms ~ 수초) ──────────►|    │
│                                                                      │
│  Warm Start (컨테이너 재사용):                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                        │ 4. 핸들러   │         │
│  │            (재사용됨)                  │    실행     │         │
│  └─────────────────────────────────────────────────────────────┘    │
│  |◄───────────── 빠른 응답 (수십ms) ─────────────►|                │
│                                                                      │
│  Cold Start 발생 조건:                                               │
│  - 첫 번째 요청                                                     │
│  - 스케일 아웃으로 새 인스턴스 생성                                 │
│  - 코드 배포 후                                                     │
│  - 오랜 유휴 시간 후 (컨테이너 제거)                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Cold Start 영향 요인
```
┌─────────────────────────────────────────────────────────────────────┐
│                   Cold Start 영향 요인                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 런타임/언어                                                     │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ 빠름        Python ≈ Node.js < Go < .NET < Java       느림 │
│     │            (수십ms)                            (수백ms~초)   │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                      │
│  2. 패키지 크기                                                     │
│     ┌─────────────────────────────────────────────────────────┐    │
│     │ 작음                                                   큼   │
│     │ (수MB)  ────────────────────────────────────►  (수백MB)     │
│     │ 빠른 Cold Start                              느린 Cold Start │
│     └─────────────────────────────────────────────────────────┘    │
│                                                                      │
│  3. 메모리 할당                                                     │
│     - 메모리 증가 → CPU 성능 비례 증가                              │
│     - 128MB vs 1024MB: 초기화 시간 크게 단축                        │
│                                                                      │
│  4. VPC 연결                                                        │
│     - VPC 외부: 추가 지연 거의 없음                                 │
│     - VPC 내부: 과거 수초 → 현재 최적화됨 (수백ms)                  │
│                                                                      │
│  5. 초기화 코드                                                     │
│     - 핸들러 외부 코드는 Cold Start 시에만 실행                     │
│     - DB 연결, SDK 초기화 등                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Cold Start 해결 방법

#### 1. Provisioned Concurrency (AWS Lambda)
```yaml
# serverless.yml
functions:
  api:
    handler: handler.main
    provisionedConcurrency: 5  # 항상 5개 인스턴스 유지

    # 시간대별 설정 (비용 최적화)
    reservedConcurrency: 100  # 최대 동시 실행 수 제한
```

```python
# Terraform으로 Provisioned Concurrency 설정
resource "aws_lambda_provisioned_concurrency_config" "api" {
  function_name                     = aws_lambda_function.api.function_name
  provisioned_concurrent_executions = 5
  qualifier                         = aws_lambda_alias.api_live.name
}

# Application Auto Scaling으로 동적 조정
resource "aws_appautoscaling_target" "lambda" {
  max_capacity       = 50
  min_capacity       = 5
  resource_id        = "function:${aws_lambda_function.api.function_name}:${aws_lambda_alias.api_live.name}"
  scalable_dimension = "lambda:function:ProvisionedConcurrency"
  service_namespace  = "lambda"
}

resource "aws_appautoscaling_policy" "lambda" {
  name               = "LambdaProvisionedConcurrency"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.lambda.resource_id
  scalable_dimension = aws_appautoscaling_target.lambda.scalable_dimension
  service_namespace  = aws_appautoscaling_target.lambda.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value = 0.7  # 70% 활용률 유지

    predefined_metric_specification {
      predefined_metric_type = "LambdaProvisionedConcurrencyUtilization"
    }
  }
}
```

#### 2. 패키지 최적화
```python
# 나쁜 예: 불필요한 의존성 포함
# requirements.txt
boto3==1.28.0  # Lambda에 이미 포함됨
pandas==2.0.0  # 150MB+

# 좋은 예: 최소한의 의존성
# requirements.txt
aws-lambda-powertools==2.0.0

# 또는 Lambda Layer 사용
# 공통 라이브러리를 Layer로 분리
```

```bash
# 패키지 크기 최적화
# 1. 불필요한 파일 제거
find . -type d -name "__pycache__" -exec rm -rf {} +
find . -type d -name "*.dist-info" -exec rm -rf {} +
find . -type d -name "tests" -exec rm -rf {} +

# 2. 최소 설치
pip install --target ./package --no-cache-dir --only-binary :all: -r requirements.txt

# 3. 압축
cd package && zip -r9 ../deployment.zip .
```

#### 3. 초기화 코드 최적화
```python
import json
import os

# ========== 핸들러 외부 (Cold Start 시에만 실행) ==========

# DB 연결은 전역으로 (재사용)
import boto3
from functools import lru_cache

# Lazy initialization
_dynamodb_table = None
_secrets_client = None

def get_dynamodb_table():
    global _dynamodb_table
    if _dynamodb_table is None:
        dynamodb = boto3.resource('dynamodb')
        _dynamodb_table = dynamodb.Table(os.environ['TABLE_NAME'])
    return _dynamodb_table

@lru_cache(maxsize=1)
def get_secret(secret_name: str) -> dict:
    """비밀 값 캐싱 (Cold Start 시에만 호출)"""
    global _secrets_client
    if _secrets_client is None:
        _secrets_client = boto3.client('secretsmanager')

    response = _secrets_client.get_secret_value(SecretId=secret_name)
    return json.loads(response['SecretString'])

# ========== 핸들러 함수 (모든 요청에서 실행) ==========

def handler(event, context):
    """핸들러는 가볍게 유지"""
    table = get_dynamodb_table()

    user_id = event['pathParameters']['userId']
    response = table.get_item(Key={'userId': user_id})

    return {
        'statusCode': 200,
        'body': json.dumps(response.get('Item', {}))
    }
```

#### 4. SnapStart (Java - AWS Lambda)
```java
// AWS Lambda SnapStart - Java Cold Start 최적화

import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import org.crac.Core;
import org.crac.Resource;

public class MyHandler implements RequestHandler<Request, Response>, Resource {

    private final DynamoDbClient dynamoDbClient;
    private final String tableName;

    public MyHandler() {
        // Cold Start 시 초기화
        this.dynamoDbClient = DynamoDbClient.create();
        this.tableName = System.getenv("TABLE_NAME");

        // CRaC 리소스 등록
        Core.getGlobalContext().register(this);
    }

    @Override
    public void beforeCheckpoint(org.crac.Context<? extends Resource> context) {
        // 스냅샷 전 정리 작업
        // 예: 연결 종료, 임시 파일 삭제
    }

    @Override
    public void afterRestore(org.crac.Context<? extends Resource> context) {
        // 스냅샷 복원 후 재초기화
        // 예: 연결 재설정
    }

    @Override
    public Response handleRequest(Request request, Context context) {
        // 비즈니스 로직
        return new Response("Success");
    }
}
```

```yaml
# serverless.yml - SnapStart 활성화
functions:
  api:
    handler: com.example.MyHandler
    runtime: java21
    snapStart: true
    memorySize: 512
```

#### 5. Warming 전략 (2025년 최신)
```python
# CloudWatch Events를 사용한 Warming
# serverless.yml
functions:
  api:
    handler: handler.main
    events:
      - http:
          path: /api
          method: any
      - schedule:
          rate: rate(5 minutes)
          input:
            warmer: true
            concurrency: 10  # 동시에 10개 인스턴스 warm

# handler.py
def main(event, context):
    # Warming 요청 감지
    if event.get('warmer'):
        concurrency = event.get('concurrency', 1)
        if concurrency > 1:
            # 다른 인스턴스도 warm
            invoke_concurrent_warmers(concurrency - 1)
        return {'statusCode': 200, 'body': 'warmed'}

    # 실제 비즈니스 로직
    return process_request(event)

def invoke_concurrent_warmers(count):
    """Lambda를 재귀적으로 호출하여 동시에 warm"""
    import boto3
    lambda_client = boto3.client('lambda')

    for i in range(count):
        lambda_client.invoke(
            FunctionName=os.environ['AWS_LAMBDA_FUNCTION_NAME'],
            InvocationType='Event',  # 비동기
            Payload=json.dumps({'warmer': True, 'concurrency': 1})
        )
```

#### 6. 2025년 최신 Cold Start 최적화 기술
```
최신 연구 및 기술 (2025):

1. Transformer 기반 예측 모델
   - Cold Start 발생 예측
   - 최대 79% Cold Start 시간 감소
   - Azure 데이터셋 기반 검증

2. 리소스 직렬화 (Resource Serialization)
   - 웹 서비스 시나리오에서 86.78% 감소
   - 42초 → 5.5초로 단축
   - 크로스 플랫폼 53%+ 최적화

3. Prebaking Runtime Environments
   - 런타임 환경 사전 준비
   - Checkpoint/Restore 기술 활용
   - 샌드박스 최적화

4. FaaSLight
   - 애플리케이션 코드 최적화
   - 핵심 코드만 로드
   - 의존성 지연 로딩
```

---

## 서버리스 패턴

### 1. API Gateway + Lambda + DynamoDB (기본 패턴)
```
┌─────────────────────────────────────────────────────────────────────┐
│                    API Gateway + Lambda + DynamoDB                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Client                                                              │
│    │                                                                 │
│    ▼                                                                 │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    API Gateway                                │   │
│  │  - REST/HTTP API                                              │   │
│  │  - 요청 검증, 변환                                            │   │
│  │  - 인증/인가 (Cognito, Lambda Authorizer)                    │   │
│  │  - Rate Limiting, Throttling                                 │   │
│  │  - 응답 캐싱                                                  │   │
│  └─────────────────────────┬────────────────────────────────────┘   │
│                            │                                         │
│                            ▼                                         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      Lambda Function                          │   │
│  │  - 비즈니스 로직                                              │   │
│  │  - 입력 검증                                                  │   │
│  │  - 데이터 변환                                                │   │
│  └─────────────────────────┬────────────────────────────────────┘   │
│                            │                                         │
│                            ▼                                         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      DynamoDB                                 │   │
│  │  - Single-Table Design                                       │   │
│  │  - On-Demand Capacity                                        │   │
│  │  - Global Tables (다중 리전)                                 │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. 이벤트 소싱 패턴
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Event Sourcing Pattern                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Command                                                             │
│    │                                                                 │
│    ▼                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │  API Gateway │───►│   Lambda     │───►│  EventBridge │          │
│  │  (Command)   │    │  (Validate)  │    │   (Events)   │          │
│  └──────────────┘    └──────────────┘    └──────┬───────┘          │
│                                                  │                   │
│                    ┌─────────────────────────────┼──────────────┐   │
│                    │                             │              │   │
│                    ▼                             ▼              ▼   │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────┐│
│  │    Event Store       │  │   Read Model         │  │  External  ││
│  │  (DynamoDB/Kinesis)  │  │   (DynamoDB)         │  │  Systems   ││
│  │                      │  │                      │  │            ││
│  │  - 모든 이벤트 저장  │  │  - 쿼리 최적화 뷰   │  │  - Email   ││
│  │  - 불변 로그         │  │  - 다양한 프로젝션   │  │  - SMS     ││
│  │  - 감사 추적         │  │                      │  │  - Webhook ││
│  └──────────────────────┘  └──────────────────────┘  └────────────┘│
│                                                                      │
│  Query                                                               │
│    │                                                                 │
│    ▼                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │  API Gateway │───►│   Lambda     │───►│ Read Model   │          │
│  │   (Query)    │    │   (Read)     │    │ (DynamoDB)   │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. 팬아웃 패턴 (Fan-out)
```
┌─────────────────────────────────────────────────────────────────────┐
│                       Fan-out Pattern                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐                                                   │
│  │    S3        │  이미지 업로드                                    │
│  │   Bucket     │──────────────┐                                    │
│  └──────────────┘              │                                    │
│                                ▼                                    │
│                    ┌──────────────────────┐                         │
│                    │     SNS Topic        │                         │
│                    │   (Fan-out Hub)      │                         │
│                    └──────────┬───────────┘                         │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐               │
│         │                     │                     │               │
│         ▼                     ▼                     ▼               │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │ SQS Queue 1  │     │ SQS Queue 2  │     │ SQS Queue 3  │        │
│  │ (Thumbnail)  │     │ (Watermark)  │     │ (Metadata)   │        │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘        │
│         │                     │                    │                │
│         ▼                     ▼                    ▼                │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   Lambda     │     │   Lambda     │     │   Lambda     │        │
│  │  (Resize)    │     │  (Watermark) │     │  (Extract)   │        │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘        │
│         │                     │                    │                │
│         ▼                     ▼                    ▼                │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │     S3       │     │     S3       │     │   DynamoDB   │        │
│  │ (Thumbnails) │     │ (Watermarked)│     │  (Metadata)  │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Saga 패턴 (분산 트랜잭션)
```
┌─────────────────────────────────────────────────────────────────────┐
│                         Saga Pattern                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  주문 생성 Saga:                                                     │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Step Functions                              │  │
│  │  (Orchestrator)                                                │  │
│  │                                                                │  │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐       │  │
│  │  │ Create  │──►│ Reserve │──►│ Process │──►│ Confirm │       │  │
│  │  │ Order   │   │ Stock   │   │ Payment │   │ Order   │       │  │
│  │  └────┬────┘   └────┬────┘   └────┬────┘   └────┬────┘       │  │
│  │       │             │             │             │             │  │
│  │       │     실패 시 │     실패 시 │     실패 시 │             │  │
│  │       │             ▼             ▼             ▼             │  │
│  │       │      ┌─────────┐   ┌─────────┐   ┌─────────┐         │  │
│  │       │      │ Release │   │ Refund  │   │ Cancel  │         │  │
│  │       │      │ Stock   │   │ Payment │   │ Order   │         │  │
│  │       │      └─────────┘   └─────────┘   └─────────┘         │  │
│  │       │                                                       │  │
│  │       ▼        (보상 트랜잭션 - Compensating Transactions)    │  │
│  │  ┌─────────┐                                                  │  │
│  │  │ Notify  │  성공/실패 알림                                  │  │
│  │  │ Result  │                                                  │  │
│  │  └─────────┘                                                  │  │
│  │                                                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5. Step Functions을 활용한 Saga 구현
```json
{
  "Comment": "Order Processing Saga",
  "StartAt": "CreateOrder",
  "States": {
    "CreateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:createOrder",
      "Next": "ReserveStock",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "OrderFailed"
      }],
      "ResultPath": "$.order"
    },
    "ReserveStock": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:reserveStock",
      "Next": "ProcessPayment",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "CancelOrder"
      }],
      "ResultPath": "$.stock"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:processPayment",
      "Next": "ConfirmOrder",
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "ReleaseStock"
      }],
      "ResultPath": "$.payment"
    },
    "ConfirmOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:confirmOrder",
      "End": true
    },
    "ReleaseStock": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:releaseStock",
      "Next": "CancelOrder"
    },
    "CancelOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:cancelOrder",
      "Next": "OrderFailed"
    },
    "OrderFailed": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:notifyFailure",
      "End": true
    }
  }
}
```

### 6. 비동기 API 패턴
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Async API Pattern                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 요청 제출 (즉시 응답)                                            │
│  ┌─────────┐   ┌─────────────┐   ┌─────────┐   ┌─────────┐         │
│  │ Client  │──►│ API Gateway │──►│ Lambda  │──►│  SQS    │         │
│  └─────────┘   └─────────────┘   └────┬────┘   └─────────┘         │
│       ▲                               │                             │
│       │                               │ 202 Accepted                │
│       │                               │ Location: /jobs/{id}        │
│       │                               ▼                             │
│       │                         ┌─────────────┐                     │
│       │                         │  DynamoDB   │ (작업 상태 저장)     │
│       │                         └─────────────┘                     │
│       │                                                             │
│  2. 백그라운드 처리                                                  │
│       │                         ┌─────────┐   ┌─────────┐           │
│       │                         │  SQS    │──►│ Lambda  │──┐        │
│       │                         └─────────┘   └─────────┘  │        │
│       │                                            ▼       │        │
│       │                                     ┌─────────────┐│        │
│       │                                     │  DynamoDB   ││        │
│       │                                     │ (상태 업데이트)│       │
│       │                                     └─────────────┘│        │
│       │                                                    │        │
│  3. 상태 확인 (폴링)                                        │        │
│       │    ┌─────────────┐   ┌─────────┐   ┌─────────────┐│        │
│       └───►│ API Gateway │──►│ Lambda  │──►│  DynamoDB   ││        │
│            └─────────────┘   └─────────┘   └─────────────┘│        │
│                 │                                          │        │
│                 │ GET /jobs/{id}                          │        │
│                 │ → {"status": "completed", "result": ...}│        │
│                 │                                          │        │
│  4. 완료 알림 (선택적)                                      │        │
│            ┌─────────────┐◄───────────────────────────────┘        │
│            │   SNS/SQS   │                                          │
│            │  (Callback) │                                          │
│            └─────────────┘                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 장단점과 적합한 사용 사례

### 장점
```
┌─────────────────────────────────────────────────────────────────────┐
│                      서버리스 장점                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 비용 효율성                                                     │
│     ✓ 사용한 만큼만 지불 (실행 시간 기준)                           │
│     ✓ 유휴 시간 비용 없음                                          │
│     ✓ 인프라 관리 인력 비용 절감                                    │
│                                                                      │
│  2. 자동 확장성                                                     │
│     ✓ 트래픽에 따른 자동 스케일 업/다운                             │
│     ✓ 0에서 수천 개 인스턴스까지 자동 확장                          │
│     ✓ 용량 계획 불필요                                              │
│                                                                      │
│  3. 운영 부담 감소                                                  │
│     ✓ 서버 패치/업데이트 불필요                                     │
│     ✓ 인프라 모니터링 부담 감소                                     │
│     ✓ 고가용성 기본 제공                                            │
│                                                                      │
│  4. 빠른 배포                                                       │
│     ✓ 코드만 배포                                                   │
│     ✓ 인프라 프로비저닝 불필요                                      │
│     ✓ 짧은 개발-배포 사이클                                         │
│                                                                      │
│  5. 집중력                                                          │
│     ✓ 비즈니스 로직에 집중                                          │
│     ✓ 인프라 복잡성 추상화                                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 단점
```
┌─────────────────────────────────────────────────────────────────────┐
│                      서버리스 단점                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Cold Start 지연                                                 │
│     ✗ 첫 요청 또는 스케일 아웃 시 지연                              │
│     ✗ 레이턴시에 민감한 애플리케이션에 부적합                       │
│     ✗ Provisioned Concurrency 비용 추가                            │
│                                                                      │
│  2. 실행 시간 제한                                                  │
│     ✗ Lambda: 최대 15분                                             │
│     ✗ 장기 실행 작업에 부적합                                       │
│     ✗ Step Functions 등 추가 서비스 필요                            │
│                                                                      │
│  3. 벤더 종속                                                       │
│     ✗ 클라우드 제공업체별 다른 API                                  │
│     ✗ 마이그레이션 어려움                                           │
│     ✗ 특정 서비스에 의존                                            │
│                                                                      │
│  4. 디버깅/테스트 복잡성                                            │
│     ✗ 로컬 환경 재현 어려움                                         │
│     ✗ 분산 추적 필요                                                │
│     ✗ 로그 중앙화 필요                                              │
│                                                                      │
│  5. 비용 예측 어려움                                                │
│     ✗ 가변적 트래픽 시 비용 예측 곤란                               │
│     ✗ 지속적 고부하 시 VM보다 비쌀 수 있음                          │
│                                                                      │
│  6. 리소스 제한                                                     │
│     ✗ 메모리, 스토리지 제한                                         │
│     ✗ 동시 실행 수 제한                                             │
│     ✗ 패키지 크기 제한                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 적합한 사용 사례
```
┌─────────────────────────────────────────────────────────────────────┐
│                    적합한 사용 사례                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ✅ 적합:                                                           │
│                                                                      │
│  1. 이벤트 기반 처리                                                │
│     - 파일 업로드 처리                                              │
│     - 데이터 변환/ETL                                               │
│     - IoT 데이터 수집                                               │
│                                                                      │
│  2. API 백엔드                                                      │
│     - CRUD 작업                                                     │
│     - 마이크로서비스                                                │
│     - 웹훅 처리                                                     │
│                                                                      │
│  3. 스케줄 작업                                                     │
│     - 정기 보고서 생성                                              │
│     - 데이터 정리/백업                                              │
│     - 알림 발송                                                     │
│                                                                      │
│  4. 가변적 트래픽                                                   │
│     - 마케팅 캠페인                                                 │
│     - 계절적 트래픽                                                 │
│     - 스타트업 MVP                                                  │
│                                                                      │
│  ❌ 부적합:                                                         │
│                                                                      │
│  1. 지속적 고부하                                                   │
│     - 24/7 높은 트래픽                                              │
│     - 예측 가능한 일정 부하                                         │
│                                                                      │
│  2. 레이턴시 민감                                                   │
│     - 실시간 게임                                                   │
│     - 고빈도 거래 시스템                                            │
│                                                                      │
│  3. 장기 실행                                                       │
│     - 대용량 데이터 처리                                            │
│     - 복잡한 ML 학습                                                │
│                                                                      │
│  4. 특수 하드웨어 필요                                              │
│     - GPU 집약적 작업                                               │
│     - 특정 OS/커널 요구사항                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 비용 비교 분석
```
비용 분기점 분석 (대략적):

Lambda vs EC2 (비용 효율성):

요청 수/월     Lambda 비용      EC2 비용 (t3.medium)
─────────────────────────────────────────────────────
10만          ~$0.20          ~$30 (고정)
100만         ~$2.00          ~$30 (고정)
1,000만       ~$20.00         ~$30 (고정)
1억           ~$200.00        ~$30 (고정)  ← 분기점
10억          ~$2,000.00      ~$60+ (스케일 필요)

결론:
- 월 1억 요청 미만: Lambda가 유리
- 월 1억 요청 이상 (지속적): EC2/컨테이너 고려
- 가변적 트래픽: Lambda가 대부분 유리
```

---

## Serverless Framework

Serverless Framework는 서버리스 애플리케이션을 쉽게 개발하고 배포할 수 있게 해주는 오픈소스 프레임워크입니다.

### Serverless Framework 개요
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Serverless Framework                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  지원 클라우드:                                                      │
│  - AWS Lambda (가장 광범위한 지원)                                  │
│  - Azure Functions                                                  │
│  - Google Cloud Functions                                           │
│  - Cloudflare Workers                                               │
│  - Kubernetes (Knative)                                             │
│                                                                      │
│  주요 기능:                                                          │
│  - serverless.yml로 인프라 정의 (IaC)                               │
│  - 다양한 플러그인 생태계                                           │
│  - 로컬 개발/테스트 지원                                            │
│  - CI/CD 통합                                                       │
│  - Dashboard로 모니터링                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 기본 프로젝트 구조
```
my-serverless-app/
├── serverless.yml          # 메인 설정 파일
├── package.json            # Node.js 의존성
├── src/
│   ├── handlers/           # Lambda 핸들러
│   │   ├── users.js
│   │   └── orders.js
│   ├── lib/                # 공통 라이브러리
│   │   ├── dynamodb.js
│   │   └── middleware.js
│   └── utils/              # 유틸리티
│       └── response.js
├── tests/                  # 테스트
│   ├── unit/
│   └── integration/
├── .env.example            # 환경변수 템플릿
└── .github/
    └── workflows/
        └── deploy.yml      # CI/CD
```

### serverless.yml 상세 예시
```yaml
# serverless.yml
service: my-api

# 프레임워크 설정
frameworkVersion: '3'

# 사용자 정의 변수
custom:
  stage: ${opt:stage, 'dev'}
  tableName: ${self:service}-${self:custom.stage}-table

  # 플러그인 설정
  prune:
    automatic: true
    number: 3

  warmup:
    default:
      enabled: true
      events:
        - schedule: rate(5 minutes)
      concurrency: 10

# 플러그인
plugins:
  - serverless-webpack
  - serverless-offline
  - serverless-prune-plugin
  - serverless-plugin-warmup
  - serverless-domain-manager

# 프로바이더 설정
provider:
  name: aws
  runtime: nodejs20.x
  stage: ${self:custom.stage}
  region: ap-northeast-2

  # 메모리 및 타임아웃 (기본값)
  memorySize: 256
  timeout: 10

  # 환경변수
  environment:
    TABLE_NAME: ${self:custom.tableName}
    STAGE: ${self:custom.stage}

  # IAM 권한
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:Scan
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:UpdateItem
            - dynamodb:DeleteItem
          Resource:
            - !GetAtt UsersTable.Arn
            - !Join ['/', [!GetAtt UsersTable.Arn, 'index/*']]
        - Effect: Allow
          Action:
            - secretsmanager:GetSecretValue
          Resource:
            - !Ref ApiSecret

  # API Gateway 설정
  apiGateway:
    shouldStartNameWithService: true
    minimumCompressionSize: 1024
    binaryMediaTypes:
      - 'image/*'
      - 'application/pdf'

  # 로깅
  logs:
    restApi:
      accessLogging: true
      format: '{"requestId":"$context.requestId","ip":"$context.identity.sourceIp","method":"$context.httpMethod","path":"$context.path","status":"$context.status"}'

  # X-Ray 추적
  tracing:
    lambda: true
    apiGateway: true

# 함수 정의
functions:
  # 사용자 API
  getUser:
    handler: src/handlers/users.getUser
    description: Get user by ID
    memorySize: 512
    events:
      - http:
          path: /users/{id}
          method: get
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref CognitoAuthorizer

  createUser:
    handler: src/handlers/users.createUser
    events:
      - http:
          path: /users
          method: post
          cors: true
          request:
            schemas:
              application/json: ${file(schemas/create-user.json)}

  # 배치 처리
  processOrders:
    handler: src/handlers/orders.processOrders
    timeout: 900  # 15분
    memorySize: 1024
    reservedConcurrency: 5  # 동시 실행 제한
    events:
      - sqs:
          arn: !GetAtt OrdersQueue.Arn
          batchSize: 10
          maximumBatchingWindow: 60
          functionResponseType: ReportBatchItemFailures

  # 스케줄 작업
  dailyReport:
    handler: src/handlers/reports.generate
    timeout: 300
    events:
      - schedule:
          rate: cron(0 9 * * ? *)
          timezone: Asia/Seoul
          enabled: true
          input:
            reportType: daily

  # DynamoDB Stream
  onUserChange:
    handler: src/handlers/users.onChange
    events:
      - stream:
          type: dynamodb
          arn: !GetAtt UsersTable.StreamArn
          batchSize: 100
          startingPosition: LATEST
          maximumRetryAttempts: 3
          destinations:
            onFailure:
              type: sqs
              arn: !GetAtt DLQ.Arn

# 리소스 (CloudFormation)
resources:
  Resources:
    # DynamoDB 테이블
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:custom.tableName}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: PK
            AttributeType: S
          - AttributeName: SK
            AttributeType: S
          - AttributeName: GSI1PK
            AttributeType: S
          - AttributeName: GSI1SK
            AttributeType: S
        KeySchema:
          - AttributeName: PK
            KeyType: HASH
          - AttributeName: SK
            KeyType: RANGE
        GlobalSecondaryIndexes:
          - IndexName: GSI1
            KeySchema:
              - AttributeName: GSI1PK
                KeyType: HASH
              - AttributeName: GSI1SK
                KeyType: RANGE
            Projection:
              ProjectionType: ALL
        StreamSpecification:
          StreamViewType: NEW_AND_OLD_IMAGES

    # SQS 큐
    OrdersQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-${self:custom.stage}-orders
        VisibilityTimeout: 960
        RedrivePolicy:
          deadLetterTargetArn: !GetAtt DLQ.Arn
          maxReceiveCount: 3

    # Dead Letter Queue
    DLQ:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-${self:custom.stage}-dlq
        MessageRetentionPeriod: 1209600  # 14일

    # Secrets Manager
    ApiSecret:
      Type: AWS::SecretsManager::Secret
      Properties:
        Name: ${self:service}-${self:custom.stage}-api-secret
        GenerateSecretString:
          PasswordLength: 32

  Outputs:
    ApiUrl:
      Value: !Sub 'https://${ApiGatewayRestApi}.execute-api.${AWS::Region}.amazonaws.com/${self:custom.stage}'
    TableName:
      Value: !Ref UsersTable
```

### Lambda 핸들러 예시
```javascript
// src/handlers/users.js
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, PutCommand } from '@aws-sdk/lib-dynamodb';
import middy from '@middy/core';
import jsonBodyParser from '@middy/http-json-body-parser';
import httpErrorHandler from '@middy/http-error-handler';
import cors from '@middy/http-cors';
import { createError } from '@middy/util';

const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);
const TABLE_NAME = process.env.TABLE_NAME;

// 기본 핸들러
const getUserHandler = async (event) => {
  const { id } = event.pathParameters;

  const result = await docClient.send(new GetCommand({
    TableName: TABLE_NAME,
    Key: {
      PK: `USER#${id}`,
      SK: `USER#${id}`
    }
  }));

  if (!result.Item) {
    throw createError(404, 'User not found');
  }

  return {
    statusCode: 200,
    body: JSON.stringify(result.Item)
  };
};

const createUserHandler = async (event) => {
  const { email, name } = event.body;

  const userId = crypto.randomUUID();
  const now = new Date().toISOString();

  const user = {
    PK: `USER#${userId}`,
    SK: `USER#${userId}`,
    GSI1PK: `EMAIL#${email}`,
    GSI1SK: `USER#${userId}`,
    id: userId,
    email,
    name,
    createdAt: now,
    updatedAt: now
  };

  await docClient.send(new PutCommand({
    TableName: TABLE_NAME,
    Item: user,
    ConditionExpression: 'attribute_not_exists(PK)'
  }));

  return {
    statusCode: 201,
    body: JSON.stringify(user)
  };
};

// Middy 미들웨어 적용
export const getUser = middy(getUserHandler)
  .use(httpErrorHandler())
  .use(cors());

export const createUser = middy(createUserHandler)
  .use(jsonBodyParser())
  .use(httpErrorHandler())
  .use(cors());
```

### 로컬 개발 및 테스트
```bash
# 로컬 실행
npx serverless offline

# 특정 함수 로컬 호출
npx serverless invoke local \
    --function getUser \
    --path tests/events/get-user.json

# 단위 테스트
npm test

# 통합 테스트 (배포된 환경)
npx serverless invoke \
    --function getUser \
    --stage dev \
    --path tests/events/get-user.json
```

### 배포
```bash
# 개발 환경 배포
npx serverless deploy --stage dev

# 프로덕션 배포
npx serverless deploy --stage prod

# 특정 함수만 배포 (빠른 배포)
npx serverless deploy function --function getUser

# 배포 정보 확인
npx serverless info --stage prod

# 로그 확인
npx serverless logs --function getUser --tail

# 삭제
npx serverless remove --stage dev
```

### GitHub Actions CI/CD
```yaml
# .github/workflows/deploy.yml
name: Deploy Serverless

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

  deploy-dev:
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-northeast-2

      - name: Deploy to Dev
        run: npx serverless deploy --stage dev

  deploy-prod:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ap-northeast-2

      - name: Deploy to Production
        run: npx serverless deploy --stage prod
```

### SAM (Serverless Application Model) 비교
```yaml
# AWS SAM template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Runtime: python3.12
    Timeout: 30
    MemorySize: 256
    Tracing: Active
    Environment:
      Variables:
        TABLE_NAME: !Ref UsersTable

Resources:
  GetUserFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: src/handlers/users.get_user
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /users/{id}
            Method: GET
      Policies:
        - DynamoDBReadPolicy:
            TableName: !Ref UsersTable

  UsersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH

Outputs:
  ApiUrl:
    Value: !Sub 'https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod'
```

---

## 참고 자료

- [AWS Lambda 공식 문서](https://docs.aws.amazon.com/lambda/)
- [Serverless Framework 공식 문서](https://www.serverless.com/framework/docs/)
- [AWS Step Functions](https://docs.aws.amazon.com/step-functions/)
- [Cold Start 연구 논문 - Transformer 기반](https://arxiv.org/abs/2504.11338)
- [Serverless Best Practices](https://www.serverless.com/blog/serverless-deployment-best-practices)
- [AWS Well-Architected Serverless](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/)

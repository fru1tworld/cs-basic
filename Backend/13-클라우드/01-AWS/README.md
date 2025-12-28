# AWS (Amazon Web Services)

## 목차
1. [개요](#개요)
2. [핵심 서비스](#핵심-서비스)
3. [IAM (Identity and Access Management)](#iam-identity-and-access-management)
4. [AWS 네트워킹](#aws-네트워킹)
5. [Auto Scaling](#auto-scaling)
6. [아키텍처 패턴](#아키텍처-패턴)
---

## 개요

AWS(Amazon Web Services)는 전 세계에서 가장 큰 클라우드 서비스 제공업체로, 약 31%의 글로벌 시장 점유율을 보유하고 있습니다. 200개 이상의 완전 관리형 서비스를 제공하며, 컴퓨팅, 스토리지, 데이터베이스, 네트워킹, 머신러닝 등 다양한 영역을 포괄합니다.

### AWS의 글로벌 인프라
```
┌─────────────────────────────────────────────────────────────────┐
│                     AWS Global Infrastructure                    │
├─────────────────────────────────────────────────────────────────┤
│  Region (리전)                                                   │
│  ├── Availability Zone 1 (가용 영역)                             │
│  │   ├── Data Center A                                          │
│  │   └── Data Center B                                          │
│  ├── Availability Zone 2                                        │
│  │   ├── Data Center C                                          │
│  │   └── Data Center D                                          │
│  └── Availability Zone 3                                        │
│      └── Data Center E                                          │
│                                                                  │
│  Edge Location (엣지 로케이션) - CloudFront CDN                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 핵심 서비스

### 1. Amazon EC2 (Elastic Compute Cloud)

EC2는 AWS의 가장 기본적인 컴퓨팅 서비스로, 가상 서버(인스턴스)를 제공합니다.

#### 인스턴스 유형
| 패밀리 | 용도 | 사용 사례 |
|--------|------|-----------|
| **t3/t4g** | 범용 (버스트 가능) | 개발/테스트, 소규모 웹 서버 |
| **m6i/m7i** | 범용 (고정 성능) | 프로덕션 웹 서버, 애플리케이션 서버 |
| **c6i/c7i** | 컴퓨팅 최적화 | 배치 처리, 게임 서버, 과학 연산 |
| **r6i/r7i** | 메모리 최적화 | 인메모리 캐시, 대용량 DB |
| **p4d/p5** | GPU 최적화 | 머신러닝, 영상 처리 |
| **i4i** | 스토리지 최적화 | NoSQL DB, 데이터 웨어하우스 |

#### 구매 옵션
```
┌─────────────────────────────────────────────────────────────────┐
│                    EC2 구매 옵션 비교                            │
├─────────────┬───────────────┬──────────────┬───────────────────┤
│   옵션       │    할인율     │   약정 기간   │      특징          │
├─────────────┼───────────────┼──────────────┼───────────────────┤
│ On-Demand   │    0%        │    없음      │ 유연성 최대        │
│ Reserved    │   최대 72%   │   1~3년      │ 예측 가능한 워크로드 │
│ Spot        │   최대 90%   │    없음      │ 중단 가능 워크로드  │
│ Savings Plan│   최대 72%   │   1~3년      │ 유연한 할인        │
└─────────────┴───────────────┴──────────────┴───────────────────┘
```

#### EC2 핵심 코드 예시 (AWS SDK for Python - Boto3)
```python
import boto3

ec2 = boto3.resource('ec2')

# EC2 인스턴스 생성
instances = ec2.create_instances(
    ImageId='ami-0c55b159cbfafe1f0',  # Amazon Linux 2 AMI
    MinCount=1,
    MaxCount=1,
    InstanceType='t3.medium',
    KeyName='my-key-pair',
    SecurityGroupIds=['sg-0123456789abcdef0'],
    SubnetId='subnet-0123456789abcdef0',
    TagSpecifications=[
        {
            'ResourceType': 'instance',
            'Tags': [
                {'Key': 'Name', 'Value': 'MyWebServer'},
                {'Key': 'Environment', 'Value': 'Production'}
            ]
        }
    ],
    # IMDSv2 강제 적용 (보안 모범 사례)
    MetadataOptions={
        'HttpTokens': 'required',
        'HttpEndpoint': 'enabled'
    }
)
```

---

### 2. Amazon S3 (Simple Storage Service)

S3는 무제한 확장이 가능한 객체 스토리지 서비스입니다. 내구성 99.999999999% (11 9's)를 보장합니다.

#### 스토리지 클래스
```
┌─────────────────────────────────────────────────────────────────────┐
│                        S3 Storage Classes                            │
├──────────────────────┬─────────────┬─────────────┬─────────────────┤
│       클래스          │  가용성 SLA  │  최소 저장   │     용도        │
├──────────────────────┼─────────────┼─────────────┼─────────────────┤
│ S3 Standard          │   99.99%    │    없음     │ 자주 액세스     │
│ S3 Intelligent-Tier  │   99.9%     │    없음     │ 패턴 변화       │
│ S3 Standard-IA       │   99.9%     │   30일      │ 드문 액세스     │
│ S3 One Zone-IA       │   99.5%     │   30일      │ 재생성 가능 데이터│
│ S3 Glacier IR        │   99.9%     │   90일      │ 아카이브 (즉시) │
│ S3 Glacier Flexible  │   99.99%    │   90일      │ 아카이브 (분~시간)│
│ S3 Glacier Deep      │   99.99%    │   180일     │ 장기 아카이브   │
└──────────────────────┴─────────────┴─────────────┴─────────────────┘
```

#### S3 수명 주기 정책 예시
```json
{
    "Rules": [
        {
            "ID": "MoveToGlacierAfter90Days",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "logs/"
            },
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "STANDARD_IA"
                },
                {
                    "Days": 90,
                    "StorageClass": "GLACIER"
                }
            ],
            "Expiration": {
                "Days": 365
            }
        }
    ]
}
```

#### S3 보안 모범 사례
```python
import boto3
import json

s3 = boto3.client('s3')

# 버킷 정책: 특정 VPC 엔드포인트에서만 접근 허용
bucket_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RestrictToVPCEndpoint",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::my-secure-bucket",
                "arn:aws:s3:::my-secure-bucket/*"
            ],
            "Condition": {
                "StringNotEquals": {
                    "aws:sourceVpce": "vpce-1234567890abcdef0"
                }
            }
        }
    ]
}

s3.put_bucket_policy(
    Bucket='my-secure-bucket',
    Policy=json.dumps(bucket_policy)
)

# 기본 암호화 설정
s3.put_bucket_encryption(
    Bucket='my-secure-bucket',
    ServerSideEncryptionConfiguration={
        'Rules': [{
            'ApplyServerSideEncryptionByDefault': {
                'SSEAlgorithm': 'aws:kms',
                'KMSMasterKeyID': 'alias/my-s3-key'
            },
            'BucketKeyEnabled': True  # 비용 절감
        }]
    }
)
```

---

### 3. Amazon RDS (Relational Database Service)

RDS는 관계형 데이터베이스를 완전 관리형으로 제공합니다.

#### 지원 엔진
- Amazon Aurora (MySQL/PostgreSQL 호환, 최대 5배 성능 향상)
- MySQL, PostgreSQL, MariaDB
- Oracle Database, Microsoft SQL Server

#### Multi-AZ 배포 아키텍처
```
┌─────────────────────────────────────────────────────────────────┐
│                     Multi-AZ Deployment                          │
│                                                                  │
│  ┌─────────────────────┐     ┌─────────────────────┐            │
│  │   Availability      │     │   Availability      │            │
│  │     Zone A          │     │     Zone B          │            │
│  │                     │     │                     │            │
│  │  ┌───────────────┐  │     │  ┌───────────────┐  │            │
│  │  │   Primary     │  │     │  │   Standby     │  │            │
│  │  │   Instance    │◄─┼─────┼─►│   Instance    │  │            │
│  │  └───────────────┘  │     │  └───────────────┘  │            │
│  │         │           │     │         │           │            │
│  │         ▼           │     │         ▼           │            │
│  │  ┌───────────────┐  │     │  ┌───────────────┐  │            │
│  │  │    EBS        │  │ Sync│  │    EBS        │  │            │
│  │  │    Volume     │◄─┼─────┼─►│    Volume     │  │            │
│  │  └───────────────┘  │ Rep │  └───────────────┘  │            │
│  │                     │     │                     │            │
│  └─────────────────────┘     └─────────────────────┘            │
│                                                                  │
│  장애 발생 시: 자동 Failover (60-120초 이내)                      │
└─────────────────────────────────────────────────────────────────┘
```

#### Aurora Read Replica 구성
```python
import boto3

rds = boto3.client('rds')

# Aurora 읽기 전용 복제본 생성
response = rds.create_db_instance(
    DBInstanceIdentifier='my-aurora-replica-1',
    DBInstanceClass='db.r6g.large',
    Engine='aurora-mysql',
    DBClusterIdentifier='my-aurora-cluster',
    AvailabilityZone='ap-northeast-2b',
    Tags=[
        {'Key': 'Purpose', 'Value': 'ReadReplica'},
        {'Key': 'Environment', 'Value': 'Production'}
    ]
)

# 연결 문자열 예시 (읽기/쓰기 분리)
"""
Writer Endpoint: my-aurora-cluster.cluster-xxxx.ap-northeast-2.rds.amazonaws.com
Reader Endpoint: my-aurora-cluster.cluster-ro-xxxx.ap-northeast-2.rds.amazonaws.com
"""
```

---

### 4. AWS Lambda

Lambda는 서버리스 컴퓨팅 서비스로, 코드만 업로드하면 자동으로 실행 환경을 관리합니다.

#### Lambda 특성
| 항목 | 제한/특성 |
|------|----------|
| 최대 실행 시간 | 15분 |
| 메모리 | 128MB ~ 10,240MB |
| 임시 스토리지 | 최대 10GB (/tmp) |
| 동시 실행 | 기본 1,000 (조정 가능) |
| 배포 패키지 크기 | 50MB (압축), 250MB (압축 해제) |
| 컨테이너 이미지 | 최대 10GB |

#### 2025년 Lambda 주요 업데이트
- **Lambda Managed Instances**: EC2 컴퓨팅에서 Lambda 함수 실행 가능, 특수 하드웨어 접근 지원
- **Lambda Durable Functions**: 최대 1년까지 장기 실행 워크플로우 조정 가능

#### Lambda 함수 예시 (Python)
```python
import json
import boto3
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit

logger = Logger()
tracer = Tracer()
metrics = Metrics()

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

@logger.inject_lambda_context
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event, context):
    """
    API Gateway에서 호출되는 사용자 조회 Lambda 함수
    """
    try:
        # 경로 파라미터에서 사용자 ID 추출
        user_id = event['pathParameters']['userId']

        # DynamoDB에서 사용자 조회
        response = table.get_item(Key={'userId': user_id})

        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({'error': 'User not found'})
            }

        # 메트릭 기록
        metrics.add_metric(name='UserFetched', unit=MetricUnit.Count, value=1)

        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Cache-Control': 'max-age=60'
            },
            'body': json.dumps(response['Item'])
        }

    except Exception as e:
        logger.exception('Failed to fetch user')
        metrics.add_metric(name='UserFetchError', unit=MetricUnit.Count, value=1)

        return {
            'statusCode': 500,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Internal server error'})
        }
```

#### Lambda Provisioned Concurrency (Cold Start 방지)
```yaml
# serverless.yml
functions:
  api:
    handler: handler.main
    provisionedConcurrency: 5  # 항상 5개 인스턴스 웜 상태 유지
    events:
      - http:
          path: /users/{userId}
          method: get
```

---

### 5. Amazon VPC (Virtual Private Cloud)

VPC는 AWS 클라우드에서 논리적으로 격리된 네트워크 환경을 제공합니다.

*자세한 내용은 [AWS 네트워킹](#aws-네트워킹) 섹션 참조*

---

### 6. Amazon ECS (Elastic Container Service)

ECS는 Docker 컨테이너를 대규모로 실행하고 관리하기 위한 완전 관리형 서비스입니다.

#### ECS 아키텍처
```
┌─────────────────────────────────────────────────────────────────┐
│                        ECS Cluster                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    ECS Service                             │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │   Task 1    │  │   Task 2    │  │   Task 3    │       │  │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │       │  │
│  │  │ │Container│ │  │ │Container│ │  │ │Container│ │       │  │
│  │  │ │  (App)  │ │  │ │  (App)  │ │  │ │  (App)  │ │       │  │
│  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │       │  │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │       │  │
│  │  │ │Sidecar  │ │  │ │Sidecar  │ │  │ │Sidecar  │ │       │  │
│  │  │ │(Log/Mon)│ │  │ │(Log/Mon)│ │  │ │(Log/Mon)│ │       │  │
│  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  실행 환경: EC2 또는 Fargate (서버리스)                           │
└─────────────────────────────────────────────────────────────────┘
```

#### ECS Fargate Task Definition
```json
{
    "family": "my-api-task",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "512",
    "memory": "1024",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
    "containerDefinitions": [
        {
            "name": "api",
            "image": "123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/my-api:latest",
            "essential": true,
            "portMappings": [
                {
                    "containerPort": 8080,
                    "protocol": "tcp"
                }
            ],
            "environment": [
                {"name": "NODE_ENV", "value": "production"}
            ],
            "secrets": [
                {
                    "name": "DB_PASSWORD",
                    "valueFrom": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:db-password"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/my-api",
                    "awslogs-region": "ap-northeast-2",
                    "awslogs-stream-prefix": "ecs"
                }
            },
            "healthCheck": {
                "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
                "interval": 30,
                "timeout": 5,
                "retries": 3,
                "startPeriod": 60
            }
        }
    ]
}
```

---

### 7. Amazon EKS (Elastic Kubernetes Service)

EKS는 완전 관리형 Kubernetes 서비스입니다.

#### EKS vs ECS 비교
| 항목 | EKS | ECS |
|------|-----|-----|
| 오케스트레이터 | Kubernetes | AWS 자체 |
| 학습 곡선 | 가파름 | 완만함 |
| 이식성 | 높음 (K8s 표준) | AWS 종속 |
| 기능 | 풍부함 | 필수 기능 |
| 관리 복잡도 | 높음 | 낮음 |
| 생태계 | 광범위 | AWS 통합 |

#### EKS 클러스터 구성 (Terraform)
```hcl
# EKS 클러스터
resource "aws_eks_cluster" "main" {
  name     = "my-eks-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  version  = "1.29"

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
    public_access_cidrs     = ["0.0.0.0/0"]
  }

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  encryption_config {
    provider {
      key_arn = aws_kms_key.eks.arn
    }
    resources = ["secrets"]
  }
}

# Managed Node Group
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main-node-group"
  node_role_arn   = aws_iam_role.eks_node.arn
  subnet_ids      = var.private_subnet_ids

  scaling_config {
    desired_size = 3
    max_size     = 10
    min_size     = 1
  }

  instance_types = ["m6i.large"]
  capacity_type  = "ON_DEMAND"

  update_config {
    max_unavailable = 1
  }

  labels = {
    Environment = "production"
    NodeType    = "general"
  }
}
```

---

## IAM (Identity and Access Management)

IAM은 AWS 리소스에 대한 접근을 안전하게 제어하는 서비스입니다.

### IAM 핵심 개념
```
┌─────────────────────────────────────────────────────────────────┐
│                      IAM Components                              │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │     User     │    │    Group     │    │     Role     │      │
│  │   (사용자)    │    │    (그룹)    │    │    (역할)    │      │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘      │
│         │                   │                   │               │
│         └───────────────────┼───────────────────┘               │
│                             ▼                                   │
│                    ┌──────────────┐                             │
│                    │   Policy     │                             │
│                    │   (정책)     │                             │
│                    └──────────────┘                             │
│                             │                                   │
│         ┌───────────────────┼───────────────────┐               │
│         ▼                   ▼                   ▼               │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
│  │   Identity   │   │  Resource    │   │  Permission  │        │
│  │    Based     │   │   Based      │   │   Boundary   │        │
│  │   Policy     │   │   Policy     │   │    (경계)    │        │
│  └──────────────┘   └──────────────┘   └──────────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### IAM 모범 사례 (2025)

#### 1. 최소 권한 원칙 (Least Privilege)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowSpecificS3Actions",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::my-app-bucket/uploads/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-server-side-encryption": "aws:kms"
                }
            }
        }
    ]
}
```

#### 2. IAM Role 기반 접근 (임시 자격 증명)
```python
import boto3
from botocore.config import Config

# EC2 인스턴스 또는 Lambda에서 IAM Role 사용
# (Access Key 하드코딩 금지)
session = boto3.Session()
sts = session.client('sts')

# 현재 자격 증명 확인
identity = sts.get_caller_identity()
print(f"Account: {identity['Account']}")
print(f"Role: {identity['Arn']}")

# 다른 계정의 역할 가정 (Cross-Account Access)
assumed_role = sts.assume_role(
    RoleArn='arn:aws:iam::987654321098:role/CrossAccountRole',
    RoleSessionName='MySession',
    DurationSeconds=3600
)

credentials = assumed_role['Credentials']
cross_account_session = boto3.Session(
    aws_access_key_id=credentials['AccessKeyId'],
    aws_secret_access_key=credentials['SecretAccessKey'],
    aws_session_token=credentials['SessionToken']
)
```

#### 3. IAM Access Analyzer 활용
```python
import boto3

analyzer = boto3.client('accessanalyzer')

# Access Analyzer 생성
analyzer.create_analyzer(
    analyzerName='MyAccountAnalyzer',
    type='ACCOUNT',
    tags={'Purpose': 'SecurityReview'}
)

# 분석 결과 조회
findings = analyzer.list_findings(
    analyzerArn='arn:aws:access-analyzer:ap-northeast-2:123456789012:analyzer/MyAccountAnalyzer',
    filter={
        'status': {'eq': ['ACTIVE']},
        'resourceType': {'eq': ['AWS::S3::Bucket']}
    }
)

for finding in findings['findings']:
    print(f"Resource: {finding['resource']}")
    print(f"Principal: {finding['principal']}")
    print(f"Action: {finding['action']}")
```

#### 4. Service Control Policies (SCP) - 조직 수준 가드레일
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyLeavingOrganization",
            "Effect": "Deny",
            "Action": "organizations:LeaveOrganization",
            "Resource": "*"
        },
        {
            "Sid": "RequireIMDSv2",
            "Effect": "Deny",
            "Action": "ec2:RunInstances",
            "Resource": "arn:aws:ec2:*:*:instance/*",
            "Condition": {
                "StringNotEquals": {
                    "ec2:MetadataHttpTokens": "required"
                }
            }
        },
        {
            "Sid": "DenyRootAccount",
            "Effect": "Deny",
            "Action": "*",
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "aws:PrincipalArn": "arn:aws:iam::*:root"
                }
            }
        }
    ]
}
```

---

## AWS 네트워킹

### VPC 구조
```
┌─────────────────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16)                                 │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Internet Gateway (IGW)                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  ┌───────────────────────────┼───────────────────────────────────┐  │
│  │     Availability Zone A   │     Availability Zone B           │  │
│  │                           │                                    │  │
│  │  ┌─────────────────┐     │     ┌─────────────────┐           │  │
│  │  │ Public Subnet   │     │     │ Public Subnet   │           │  │
│  │  │ 10.0.1.0/24     │     │     │ 10.0.2.0/24     │           │  │
│  │  │                 │     │     │                 │           │  │
│  │  │  ┌───────────┐  │     │     │  ┌───────────┐  │           │  │
│  │  │  │   ALB     │  │     │     │  │   ALB     │  │           │  │
│  │  │  └───────────┘  │     │     │  └───────────┘  │           │  │
│  │  │  ┌───────────┐  │     │     │  ┌───────────┐  │           │  │
│  │  │  │NAT Gateway│  │     │     │  │NAT Gateway│  │           │  │
│  │  │  └───────────┘  │     │     │  └───────────┘  │           │  │
│  │  └─────────────────┘     │     └─────────────────┘           │  │
│  │           │              │              │                     │  │
│  │  ┌─────────────────┐     │     ┌─────────────────┐           │  │
│  │  │ Private Subnet  │     │     │ Private Subnet  │           │  │
│  │  │ 10.0.3.0/24     │     │     │ 10.0.4.0/24     │           │  │
│  │  │  (App Tier)     │     │     │  (App Tier)     │           │  │
│  │  │                 │     │     │                 │           │  │
│  │  │  ┌───┐ ┌───┐   │     │     │  ┌───┐ ┌───┐   │           │  │
│  │  │  │EC2│ │EC2│   │     │     │  │EC2│ │EC2│   │           │  │
│  │  │  └───┘ └───┘   │     │     │  └───┘ └───┘   │           │  │
│  │  └─────────────────┘     │     └─────────────────┘           │  │
│  │           │              │              │                     │  │
│  │  ┌─────────────────┐     │     ┌─────────────────┐           │  │
│  │  │ Private Subnet  │     │     │ Private Subnet  │           │  │
│  │  │ 10.0.5.0/24     │     │     │ 10.0.6.0/24     │           │  │
│  │  │  (DB Tier)      │◄────┼─────►│  (DB Tier)      │           │  │
│  │  │                 │ Sync│     │                 │           │  │
│  │  │  ┌───────────┐  │     │     │  ┌───────────┐  │           │  │
│  │  │  │    RDS    │  │     │     │  │    RDS    │  │           │  │
│  │  │  │ (Primary) │  │     │     │  │ (Standby) │  │           │  │
│  │  │  └───────────┘  │     │     │  └───────────┘  │           │  │
│  │  └─────────────────┘     │     └─────────────────┘           │  │
│  │                          │                                    │  │
│  └──────────────────────────┴────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### VPC CIDR 설계 가이드라인 (2025)
```
권장 CIDR 크기:
- /16 (65,536 IPs): 대규모 프로덕션 VPC
- /20 (4,096 IPs): 중규모 애플리케이션
- /24 (256 IPs): 소규모 또는 개발/테스트 환경

서브넷 세분화 예시:
┌──────────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                             │
├──────────────────────────────────────────────────────────────┤
│ Public Subnets (ALB, NAT):                                   │
│   AZ-A: 10.0.0.0/24, AZ-B: 10.0.1.0/24, AZ-C: 10.0.2.0/24   │
├──────────────────────────────────────────────────────────────┤
│ Private App Subnets:                                         │
│   AZ-A: 10.0.10.0/24, AZ-B: 10.0.11.0/24, AZ-C: 10.0.12.0/24│
├──────────────────────────────────────────────────────────────┤
│ Private DB Subnets:                                          │
│   AZ-A: 10.0.20.0/24, AZ-B: 10.0.21.0/24, AZ-C: 10.0.22.0/24│
├──────────────────────────────────────────────────────────────┤
│ EKS Pod Subnets (대규모 IP 필요):                             │
│   AZ-A: 10.0.100.0/18, AZ-B: 10.0.128.0/18                   │
└──────────────────────────────────────────────────────────────┘
```

### Security Group vs NACL

#### Security Group (상태 저장 방화벽)
```python
import boto3

ec2 = boto3.resource('ec2')

# 웹 서버용 Security Group
web_sg = ec2.create_security_group(
    GroupName='web-server-sg',
    Description='Security group for web servers',
    VpcId='vpc-0123456789abcdef0'
)

# 인바운드 규칙
web_sg.authorize_ingress(
    IpPermissions=[
        {
            'IpProtocol': 'tcp',
            'FromPort': 443,
            'ToPort': 443,
            'IpRanges': [{'CidrIp': '0.0.0.0/0', 'Description': 'HTTPS from anywhere'}]
        },
        {
            'IpProtocol': 'tcp',
            'FromPort': 80,
            'ToPort': 80,
            'IpRanges': [{'CidrIp': '0.0.0.0/0', 'Description': 'HTTP from anywhere'}]
        }
    ]
)

# 앱 서버용 Security Group (웹 서버에서만 접근 허용)
app_sg = ec2.create_security_group(
    GroupName='app-server-sg',
    Description='Security group for application servers',
    VpcId='vpc-0123456789abcdef0'
)

app_sg.authorize_ingress(
    IpPermissions=[
        {
            'IpProtocol': 'tcp',
            'FromPort': 8080,
            'ToPort': 8080,
            'UserIdGroupPairs': [
                {'GroupId': web_sg.id, 'Description': 'From web servers only'}
            ]
        }
    ]
)
```

#### Network ACL (상태 비저장 방화벽)
```python
# NACL 규칙 예시 - Private Subnet용
"""
Inbound Rules:
+-------+----------+------------+-----------------+-------+
| Rule  | Type     | Port Range | Source          | Allow |
+-------+----------+------------+-----------------+-------+
| 100   | Custom   | 8080       | 10.0.1.0/24    | Allow |
| 110   | Custom   | 8080       | 10.0.2.0/24    | Allow |
| 120   | Custom   | 1024-65535 | 0.0.0.0/0      | Allow | <- 응답 트래픽용
| *     | All      | All        | 0.0.0.0/0      | Deny  |
+-------+----------+------------+-----------------+-------+

Outbound Rules:
+-------+----------+------------+-----------------+-------+
| Rule  | Type     | Port Range | Destination     | Allow |
+-------+----------+------------+-----------------+-------+
| 100   | HTTPS    | 443        | 0.0.0.0/0      | Allow |
| 110   | Custom   | 3306       | 10.0.5.0/24    | Allow | <- DB 접근
| 120   | Custom   | 1024-65535 | 0.0.0.0/0      | Allow | <- 응답 트래픽용
| *     | All      | All        | 0.0.0.0/0      | Deny  |
+-------+----------+------------+-----------------+-------+
"""
```

### VPC Endpoints
```
┌─────────────────────────────────────────────────────────────────┐
│                    VPC Endpoint Types                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Gateway Endpoint (무료)                                      │
│     └── S3, DynamoDB                                            │
│         - 라우팅 테이블에 추가                                    │
│         - 트래픽이 AWS 네트워크 내에서 유지                        │
│                                                                  │
│  2. Interface Endpoint (유료)                                    │
│     └── 대부분의 AWS 서비스                                      │
│         - ENI 생성 (Private IP 할당)                             │
│         - PrivateLink 사용                                       │
│         - Security Group 적용 가능                               │
│                                                                  │
│  3. Gateway Load Balancer Endpoint                               │
│     └── 네트워크 어플라이언스 통합                                │
│         - 방화벽, IDS/IPS 연동                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### VPC Endpoint 생성 (Terraform)
```hcl
# S3 Gateway Endpoint
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-northeast-2.s3"

  route_table_ids = [
    aws_route_table.private.id
  ]

  tags = {
    Name = "s3-gateway-endpoint"
  }
}

# ECR Interface Endpoint (컨테이너 이미지 풀링용)
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-northeast-2.ecr.api"
  vpc_endpoint_type   = "Interface"

  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true

  tags = {
    Name = "ecr-api-endpoint"
  }
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.ap-northeast-2.ecr.dkr"
  vpc_endpoint_type   = "Interface"

  subnet_ids          = aws_subnet.private[*].id
  security_group_ids  = [aws_security_group.vpc_endpoints.id]

  private_dns_enabled = true

  tags = {
    Name = "ecr-dkr-endpoint"
  }
}
```

---

## Auto Scaling

### Target Tracking Scaling Policy

Target Tracking은 자동 스케일링의 권장 방식으로, 지정한 메트릭이 목표값을 유지하도록 자동으로 조정합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Target Tracking 동작 원리                     │
│                                                                  │
│  목표: CPU 사용률 50% 유지                                       │
│                                                                  │
│  100%│                                                          │
│      │    ┌──┐                                                  │
│   80%│    │  │ ← Scale Out 트리거                               │
│      │    │  │                                                  │
│   60%│────┤  ├────┬────                                         │
│      │    │  │    │    ← 목표값 50%                              │
│   40%│    │  │    └──────────                                   │
│      │    │  │         ↓ 인스턴스 추가 후 안정화                  │
│   20%│    │  │                                                  │
│      │    │  │                                                  │
│    0%└────┴──┴────────────────────────────► Time                │
│        T1  T2  T3    T4    T5                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### Auto Scaling 구성 (Terraform)
```hcl
# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"

  min_size         = 2
  max_size         = 10
  desired_capacity = 3

  # Warm Pool 설정 (스케일링 속도 향상)
  warm_pool {
    pool_state                  = "Stopped"
    min_size                    = 1
    max_group_prepared_capacity = 5
  }

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 75
    }
  }

  tag {
    key                 = "Name"
    value               = "app-server"
    propagate_at_launch = true
  }
}

# Target Tracking Policy - CPU
resource "aws_autoscaling_policy" "cpu_target" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 50.0

    # 스케일인 시 쿨다운
    disable_scale_in = false
  }
}

# Target Tracking Policy - ALB Request Count
resource "aws_autoscaling_policy" "request_count_target" {
  name                   = "request-count-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.main.arn_suffix}/${aws_lb_target_group.app.arn_suffix}"
    }
    target_value = 1000.0  # 인스턴스당 1000 요청
  }
}

# Predictive Scaling (2025 신규 리전 확대)
resource "aws_autoscaling_policy" "predictive" {
  name                   = "predictive-scaling"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "PredictiveScaling"

  predictive_scaling_configuration {
    mode                         = "ForecastAndScale"
    scheduling_buffer_time       = 300
    max_capacity_breach_behavior = "IncreaseMaxCapacity"
    max_capacity_buffer          = 10

    metric_specification {
      target_value = 50

      predefined_load_metric_specification {
        predefined_metric_type = "ASGTotalCPUUtilization"
      }

      predefined_scaling_metric_specification {
        predefined_metric_type = "ASGAverageCPUUtilization"
      }
    }
  }
}
```

### Instance Warm-up 설정
```python
import boto3

autoscaling = boto3.client('autoscaling')

# 인스턴스 준비 시간 설정
# 신규 인스턴스가 완전히 준비될 때까지 메트릭에서 제외
autoscaling.put_scaling_policy(
    AutoScalingGroupName='app-asg',
    PolicyName='cpu-target-tracking',
    PolicyType='TargetTrackingScaling',
    TargetTrackingConfiguration={
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'ASGAverageCPUUtilization'
        },
        'TargetValue': 50.0,
        'DisableScaleIn': False
    },
    EstimatedInstanceWarmup=300  # 5분 웜업 시간
)
```

---

## 아키텍처 패턴

### 1. 3-Tier 아키텍처
```
┌─────────────────────────────────────────────────────────────────────┐
│                         3-Tier Architecture                          │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Presentation Tier                       │    │
│  │  CloudFront (CDN) → ALB → Web Servers (EC2/ECS/EKS)         │    │
│  │  - Static Content Caching                                    │    │
│  │  - SSL/TLS Termination                                       │    │
│  │  - WAF Integration                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Application Tier                        │    │
│  │  Internal ALB → App Servers (EC2/ECS/EKS)                   │    │
│  │  - Business Logic                                            │    │
│  │  - API Processing                                            │    │
│  │  - ElastiCache (Redis) for Session/Cache                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                        Data Tier                             │    │
│  │  RDS Multi-AZ (Primary + Standby)                           │    │
│  │  + Read Replicas for Read-Heavy Workloads                   │    │
│  │  S3 for Object Storage                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Serverless 아키텍처
```
┌─────────────────────────────────────────────────────────────────────┐
│                      Serverless Architecture                         │
│                                                                      │
│  ┌──────────────┐   ┌─────────────┐   ┌──────────────────────────┐ │
│  │   Client     │──►│  CloudFront │──►│      S3 (Static Web)     │ │
│  │   Browser    │   │     (CDN)   │   │       - React/Vue        │ │
│  └──────────────┘   └──────┬──────┘   └──────────────────────────┘ │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    API Gateway (REST/HTTP)                   │   │
│  │  - Request Validation   - Rate Limiting   - API Key Auth    │   │
│  │  - Request/Response Transformation                          │   │
│  └─────────────────────────┬───────────────────────────────────┘   │
│                            │                                        │
│         ┌──────────────────┼──────────────────┐                    │
│         ▼                  ▼                  ▼                    │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐               │
│  │  Lambda    │    │  Lambda    │    │  Lambda    │               │
│  │  (Users)   │    │  (Orders)  │    │ (Products) │               │
│  └─────┬──────┘    └─────┬──────┘    └─────┬──────┘               │
│        │                 │                 │                       │
│        ▼                 ▼                 ▼                       │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                      DynamoDB                                │  │
│  │   - Single-table Design   - On-Demand Capacity              │  │
│  │   - Global Secondary Indexes                                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  비동기 처리:                                                       │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐               │
│  │    SQS     │──►│   Lambda   │──►│     S3     │               │
│  │   Queue    │    │ (Processor)│    │  (Archive) │               │
│  └────────────┘    └────────────┘    └────────────┘               │
│                                                                     │
│  실시간 처리:                                                       │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐               │
│  │  Kinesis   │──►│   Lambda   │──►│ OpenSearch │               │
│  │   Stream   │    │(Transformer)│   │  (Search)  │               │
│  └────────────┘    └────────────┘    └────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. 마이크로서비스 아키텍처 (EKS 기반)
```
┌─────────────────────────────────────────────────────────────────────┐
│                  Microservices on EKS                                │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     Ingress Layer                            │    │
│  │  AWS ALB Ingress Controller + AWS WAF                       │    │
│  └────────────────────────────┬────────────────────────────────┘    │
│                               │                                      │
│  ┌────────────────────────────┼────────────────────────────────┐    │
│  │                    Service Mesh (Istio/App Mesh)             │    │
│  │                               │                              │    │
│  │    ┌──────────┐    ┌──────────┐    ┌──────────┐            │    │
│  │    │  User    │    │  Order   │    │ Product  │            │    │
│  │    │ Service  │◄──►│ Service  │◄──►│ Service  │            │    │
│  │    └────┬─────┘    └────┬─────┘    └────┬─────┘            │    │
│  │         │               │               │                   │    │
│  │    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐              │    │
│  │    │ Envoy   │    │ Envoy   │    │ Envoy   │ (Sidecar)    │    │
│  │    │ Proxy   │    │ Proxy   │    │ Proxy   │              │    │
│  │    └─────────┘    └─────────┘    └─────────┘              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Data Layer:                                                        │
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────────┐      │
│  │ Aurora   │    │ DynamoDB │    │ ElastiCache (Redis)      │      │
│  │ (Users)  │    │ (Orders) │    │ (Session/Cache)          │      │
│  └──────────┘    └──────────┘    └──────────────────────────┘      │
│                                                                      │
│  Observability:                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ CloudWatch   │  │   X-Ray      │  │  Prometheus  │              │
│  │ Container    │  │  (Tracing)   │  │  + Grafana   │              │
│  │  Insights    │  │              │  │              │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 참고 자료

- [AWS 공식 문서](https://docs.aws.amazon.com/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS re:Invent 2025 주요 발표](https://aws.amazon.com/blogs/aws/top-announcements-of-aws-reinvent-2025/)
- [AWS IAM 보안 모범 사례](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS VPC 보안 모범 사례](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
- [EC2 Auto Scaling 문서](https://docs.aws.amazon.com/autoscaling/ec2/userguide/)

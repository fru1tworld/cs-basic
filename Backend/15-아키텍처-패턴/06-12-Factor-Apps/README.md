# 12 Factor Apps

## 목차
1. [개요 및 배경](#1-개요-및-배경)
2. [12가지 원칙](#2-12가지-원칙)
   - [I. Codebase](#i-codebase)
   - [II. Dependencies](#ii-dependencies)
   - [III. Config](#iii-config)
   - [IV. Backing Services](#iv-backing-services)
   - [V. Build, Release, Run](#v-build-release-run)
   - [VI. Processes](#vi-processes)
   - [VII. Port Binding](#vii-port-binding)
   - [VIII. Concurrency](#viii-concurrency)
   - [IX. Disposability](#ix-disposability)
   - [X. Dev/Prod Parity](#x-devprod-parity)
   - [XI. Logs](#xi-logs)
   - [XII. Admin Processes](#xii-admin-processes)
3. [실제 적용 예제](#3-실제-적용-예제)
4. [Beyond the Twelve-Factor App](#4-beyond-the-twelve-factor-app)
5. [참고 자료](#5-참고-자료)

---

## 1. 개요 및 배경

### 12 Factor App이란?

12 Factor App은 **Software-as-a-Service(SaaS) 애플리케이션을 구축하기 위한 방법론**이다. 2011년 Heroku의 공동 창업자 Adam Wiggins가 처음 발표했으며, 수백 개의 앱 개발 및 배포 경험과 Heroku 플랫폼에서의 수십만 개 앱 운영 경험을 바탕으로 작성되었다.

### 탄생 배경

2011년 당시, 개발자들은 로컬 환경에서 애플리케이션을 실행할 수 있었지만, 프로덕션 환경으로 배포할 때 많은 문제에 직면했다. 12 Factor App의 기본 목표는 **개발자가 프레임워크 문서를 따라 로컬에서 개발한 앱을 특정 플랫폼에 맞추지 않고도 프로덕션에 배포할 수 있게 하는 것**이었다.

### 핵심 목표

12 Factor 방법론은 다음과 같은 앱을 구축하는 것을 목표로 한다:

- **선언적 형식**을 사용하여 설정 자동화를 통해 프로젝트 참여 시간과 비용을 최소화
- 운영체제와의 **명확한 계약**을 통해 실행 환경 간 **최대한의 이식성** 확보
- 현대적인 **클라우드 플랫폼에 배포**하기 적합하여 서버 및 시스템 관리 필요성 제거
- 개발과 프로덕션 간의 **차이를 최소화**하여 최대한의 민첩성을 위한 **지속적 배포** 가능
- 도구, 아키텍처, 개발 방식의 큰 변경 없이 **수평적 확장** 가능

### 적용 범위

12 Factor 방법론은 **모든 프로그래밍 언어**로 작성된 앱에 적용될 수 있으며, 데이터베이스, 큐, 메모리 캐시 등 **다양한 백킹 서비스 조합**과 함께 사용할 수 있다.

---

## 2. 12가지 원칙

### I. Codebase

> **"버전 관리되는 하나의 코드베이스와 다양한 배포"**

#### 핵심 개념

- 앱당 **하나의 코드베이스**만 존재하며, 여러 배포(deploy)가 있을 수 있다
- 배포는 앱의 실행 중인 인스턴스를 의미한다 (프로덕션, 스테이징, 개발 등)
- 코드베이스는 Git, Subversion 등의 버전 관리 시스템에서 관리된다

#### 규칙

```
[코드베이스] ─────┬──> [개발 배포]
                 ├──> [스테이징 배포]
                 └──> [프로덕션 배포]
```

- **하나의 앱 = 하나의 코드베이스** (위반 시 별도의 앱으로 분리)
- **여러 앱이 동일한 코드를 공유**하면 12 Factor 위반 → 라이브러리로 분리하여 의존성으로 관리
- 각 배포는 다른 버전이 활성화될 수 있지만, 코드베이스 자체는 동일

#### Docker/Kubernetes 적용 예제

```yaml
# Kubernetes Deployment - 동일 이미지, 다른 환경
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-production
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:v1.2.3  # 동일 이미지
        envFrom:
        - configMapRef:
            name: production-config  # 환경별 설정
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-staging
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        image: myregistry/myapp:v1.2.3  # 동일 이미지
        envFrom:
        - configMapRef:
            name: staging-config  # 환경별 설정
```

---

### II. Dependencies

> **"명시적으로 선언되고 격리된 의존성"**

#### 핵심 개념

12 Factor 앱은 **시스템 전역 패키지의 암묵적 존재에 의존하지 않는다**. 모든 의존성을 **의존성 선언 매니페스트**를 통해 완전하고 정확하게 선언하며, 실행 중에는 **의존성 격리 도구**를 사용하여 주변 시스템으로부터 암묵적 의존성이 "유입"되지 않도록 한다.

#### 언어별 도구

| 언어 | 의존성 선언 | 의존성 격리 |
|------|------------|------------|
| Ruby | Gemfile (Bundler) | bundle exec |
| Python | requirements.txt (Pip) | virtualenv, venv |
| Node.js | package.json (npm/yarn) | node_modules |
| Java | pom.xml (Maven), build.gradle (Gradle) | 클래스패스 격리 |
| Go | go.mod (Go Modules) | 벤더링 |

#### 장점

- **새로운 개발자의 온보딩 간소화**: 언어 런타임과 의존성 관리자만 설치하면 결정론적 빌드 명령으로 모든 것을 설정 가능
- **재현 가능한 빌드**: 의존성 버전이 고정되어 어디서든 동일한 결과 보장

#### Docker 적용 예제

```dockerfile
# Dockerfile - 의존성 명시적 선언 및 격리
FROM python:3.11-slim

WORKDIR /app

# 의존성 선언 파일 복사 및 설치
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 시스템 도구도 명시적으로 선언
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY . .

CMD ["python", "app.py"]
```

```txt
# requirements.txt - 버전 고정으로 재현 가능한 빌드
Flask==2.3.3
SQLAlchemy==2.0.21
redis==5.0.0
psycopg2-binary==2.9.7
```

---

### III. Config

> **"환경에 설정을 저장"**

#### 핵심 개념

설정(Config)은 **배포 간에 달라지는 모든 것**을 의미한다:
- 데이터베이스, Memcached 등 백킹 서비스의 리소스 핸들
- Amazon S3, Twitter 등 외부 서비스 자격 증명
- 배포별 고유 값 (호스트명, 포트 등)

#### 핵심 규칙

- **설정은 코드와 엄격히 분리**되어야 한다
- 설정은 **환경 변수(env vars)**에 저장한다
- 환경 변수는 배포마다 독립적으로 관리되며, 코드 변경 없이 쉽게 변경 가능

#### 설정이 아닌 것

모든 배포에서 동일한 값은 설정이 아니다 (예: 프레임워크 내부 설정, 라우팅 정보 등)

#### 왜 환경 변수인가?

- 실수로 코드 저장소에 체크인될 가능성이 없다
- 언어나 OS에 독립적이다
- 특정 설정 파일 형식에 의존하지 않는다

#### Kubernetes 적용 예제

```yaml
# ConfigMap - 비밀이 아닌 설정
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  REDIS_HOST: "redis-service"
  LOG_LEVEL: "INFO"
---
# Secret - 민감한 정보
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "super-secret-password"
  API_KEY: "external-service-api-key"
---
# Deployment에서 설정 사용
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
```

```python
# Python 애플리케이션에서 환경 변수 사용
import os

DATABASE_URL = os.environ.get('DATABASE_URL')
REDIS_HOST = os.environ.get('REDIS_HOST', 'localhost')
LOG_LEVEL = os.environ.get('LOG_LEVEL', 'INFO')
API_KEY = os.environ['API_KEY']  # 필수 값은 예외 발생하도록
```

---

### IV. Backing Services

> **"백킹 서비스를 연결된 리소스로 취급"**

#### 핵심 개념

**백킹 서비스**는 앱이 정상 동작하면서 네트워크를 통해 소비하는 모든 서비스를 의미한다:
- 데이터 저장소 (MySQL, PostgreSQL, MongoDB)
- 메시지 큐 (RabbitMQ, Kafka, Amazon SQS)
- 캐시 서비스 (Redis, Memcached)
- SMTP 서비스 (Postfix, SendGrid)
- 메트릭 수집 (New Relic, Datadog)
- 객체 저장소 (Amazon S3, Google Cloud Storage)

#### 핵심 규칙

12 Factor 앱은 **로컬에서 관리하는 서비스와 서드파티 서비스를 구분하지 않는다**. 둘 다 URL이나 설정에 저장된 자격 증명을 통해 접근하는 **연결된 리소스**로 취급한다.

```
[App] ──> [로컬 MySQL]        # 교체 가능
[App] ──> [Amazon RDS MySQL]  # 코드 변경 없음
```

#### 느슨한 결합의 장점

- 코드 변경 없이 로컬 데이터베이스를 관리형 서비스로 교체 가능
- 장애 시 다른 인스턴스로 빠르게 전환 가능
- 벤더 종속성 최소화

#### Kubernetes 적용 예제

```yaml
# 외부 서비스를 Kubernetes Service로 추상화
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ExternalName
  externalName: mydb.rds.amazonaws.com  # 외부 RDS
---
# 또는 로컬 데이터베이스
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
```

```python
# 백킹 서비스 연결 - 환경 변수로 추상화
import os
from sqlalchemy import create_engine

# 로컬이든 RDS든 동일한 코드
DATABASE_URL = os.environ['DATABASE_URL']
engine = create_engine(DATABASE_URL)

# Redis도 동일하게 처리
REDIS_URL = os.environ['REDIS_URL']
```

---

### V. Build, Release, Run

> **"빌드와 실행 단계를 엄격히 분리"**

#### 세 단계의 정의

```
[코드베이스]
    │
    ▼ (빌드)
[빌드 결과물] + [설정]
    │
    ▼ (릴리스)
[릴리스]
    │
    ▼ (실행)
[실행 중인 프로세스]
```

1. **빌드 단계**: 코드를 실행 가능한 번들로 변환
   - 의존성 가져오기 및 바이너리/에셋 컴파일

2. **릴리스 단계**: 빌드 결과물과 배포 설정을 결합
   - 각 릴리스는 고유 ID를 가짐 (타임스탬프나 버전 번호)

3. **실행 단계**: 선택된 릴리스에 대해 앱 프로세스를 시작

#### 핵심 규칙

- **런타임에 코드 변경 불가**: 변경사항을 빌드 단계로 다시 전파할 방법이 없기 때문
- **릴리스는 불변**: 변경이 필요하면 새로운 릴리스를 생성
- 모든 릴리스는 **롤백을 쉽게** 하기 위해 고유 ID를 가져야 함

#### Docker/Kubernetes CI/CD 예제

```yaml
# GitLab CI/CD 파이프라인
stages:
  - build
  - release
  - deploy

# 빌드 단계
build:
  stage: build
  script:
    - docker build -t myapp:${CI_COMMIT_SHA} .
    - docker push myregistry/myapp:${CI_COMMIT_SHA}

# 릴리스 단계 - 빌드 + 설정 결합
release:
  stage: release
  script:
    - export RELEASE_ID=$(date +%Y%m%d-%H%M%S)
    - docker tag myregistry/myapp:${CI_COMMIT_SHA} myregistry/myapp:${RELEASE_ID}
    - docker push myregistry/myapp:${RELEASE_ID}
    - kubectl set image deployment/myapp myapp=myregistry/myapp:${RELEASE_ID}

# 실행 단계
deploy:
  stage: deploy
  script:
    - kubectl apply -f k8s/deployment.yaml
    - kubectl rollout status deployment/myapp
```

```dockerfile
# 멀티스테이지 빌드로 빌드/실행 분리
# 빌드 단계
FROM node:18 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 실행 단계 - 최소한의 이미지
FROM node:18-slim
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

---

### VI. Processes

> **"앱을 하나 이상의 무상태 프로세스로 실행"**

#### 핵심 개념

12 Factor 앱의 프로세스는 **무상태(Stateless)**이며 **아무것도 공유하지 않는다(Share-nothing)**. 지속되어야 하는 모든 데이터는 상태를 가진 백킹 서비스(일반적으로 데이터베이스)에 저장해야 한다.

#### 규칙

- 프로세스 메모리나 파일시스템은 단일 트랜잭션 내에서만 캐시로 사용 가능
- **Sticky Session(세션 고정)은 12 Factor를 위반**한다
- 세션 데이터는 Redis, Memcached 같은 데이터스토어에 저장

#### 왜 무상태인가?

```
[요청] ──> [프로세스 A] ──> [응답]
                          (상태 없음, 다음 요청은 어느 프로세스든 처리 가능)
[요청] ──> [프로세스 B] ──> [응답]
```

- **수평 확장 용이**: 언제든 프로세스 추가/제거 가능
- **장애 복구 용이**: 프로세스가 죽어도 상태 손실 없음
- **배포 간소화**: 새 버전 배포 시 기존 프로세스 교체만 하면 됨

#### Kubernetes 적용 예제

```yaml
# 무상태 Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 5  # 수평 확장
  template:
    spec:
      containers:
      - name: api
        image: myapi:latest
        # 상태는 외부에 저장
        env:
        - name: SESSION_STORE
          value: "redis://redis-cluster:6379"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

```python
# 세션을 Redis에 저장하는 Flask 예제
from flask import Flask
from flask_session import Session
import redis

app = Flask(__name__)
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.from_url(os.environ['REDIS_URL'])
Session(app)

# 이제 세션은 Redis에 저장되어 모든 인스턴스에서 공유
```

---

### VII. Port Binding

> **"포트 바인딩을 통해 서비스 공개"**

#### 핵심 개념

12 Factor 앱은 **완전히 자체 포함적(Self-contained)**이다. 웹 서버가 웹 서비스를 만들기 위해 런타임에 주입되는 것에 의존하지 않으며, **웹앱이 HTTP를 서비스로 내보내기 위해 포트에 바인딩**한다.

#### 핵심 규칙

- 앱은 외부 웹 서버(Apache, Nginx 등)에 의존하지 않는다
- 앱 자체가 HTTP 요청을 수신할 수 있다
- 포트 번호로 서비스를 식별한다

#### 적용 대상

HTTP뿐만 아니라 거의 모든 종류의 서버 소프트웨어에 적용:
- HTTP (웹 서비스)
- AMQP (메시지 큐)
- MQTT (IoT)
- gRPC
- WebSocket

#### Docker/Kubernetes 적용 예제

```dockerfile
# 자체 포함적 앱 - 포트 노출
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt

# 앱이 직접 포트 바인딩
EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

```yaml
# Kubernetes Service로 포트 노출
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
  - port: 80          # 외부 포트
    targetPort: 8080  # 컨테이너 포트
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080  # 앱이 바인딩하는 포트
```

```python
# Flask 앱 - 직접 포트 바인딩
from flask import Flask
import os

app = Flask(__name__)

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

---

### VIII. Concurrency

> **"프로세스 모델을 통한 수평 확장"**

#### 핵심 개념

12 Factor 앱에서 프로세스는 일급 시민이다. **앱을 더 크게 만드는 대신 더 많은 프로세스 복사본을 배포**하여 확장한다. 이것이 **수직 확장(Scale Up)** 대신 **수평 확장(Scale Out)**이다.

#### 프로세스 유형별 분리

```
[웹 프로세스] ─────────────────> HTTP 요청 처리
[워커 프로세스] ────────────────> 백그라운드 작업
[스케줄러 프로세스] ────────────> 크론 작업
[큐 프로세스] ──────────────────> 메시지 처리
```

각 프로세스 유형은 **독립적으로 확장** 가능:
- 웹 트래픽 증가 → 웹 프로세스만 확장
- 백그라운드 작업 증가 → 워커 프로세스만 확장

#### 핵심 규칙

- 앱은 자체적인 프로세스 관리나 데몬화를 시도하지 않는다
- Systemd, Upstart, Kubernetes 같은 **프로세스 매니저**에 의존
- 각 프로세스 유형을 독립적으로 확장

#### Kubernetes 적용 예제

```yaml
# 웹 프로세스
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 10  # 높은 트래픽 처리
  template:
    spec:
      containers:
      - name: web
        image: myapp:latest
        command: ["gunicorn", "app:app"]
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
---
# 워커 프로세스
apiVersion: apps/v1
kind: Deployment
metadata:
  name: worker
spec:
  replicas: 5  # 백그라운드 작업용
  template:
    spec:
      containers:
      - name: worker
        image: myapp:latest
        command: ["celery", "-A", "tasks", "worker"]
---
# HPA로 자동 확장
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

### IX. Disposability

> **"빠른 시작과 우아한 종료로 견고성 극대화"**

#### 핵심 개념

12 Factor 앱의 프로세스는 **일회용(Disposable)**이다. 언제든 시작하거나 중지될 수 있어야 한다. 이를 통해 **탄력적 확장**, **빠른 배포**, **설정 변경**이 용이해진다.

#### 핵심 요구사항

1. **시작 시간 최소화**
   - 프로세스는 시작 명령 실행 후 **수 초 내에** 요청 처리 준비 완료
   - 빠른 시작 = 빠른 확장, 빠른 복구

2. **우아한 종료(Graceful Shutdown)**
   - SIGTERM 신호 수신 시 새 요청 수신 중단
   - 현재 처리 중인 요청 완료
   - 그 후 종료

3. **갑작스러운 종료에 대한 견고성**
   - 하드웨어 장애 등에 대비
   - 작업 큐 사용 시: 작업을 큐로 반환하여 다른 워커가 처리

#### Kubernetes 적용 예제

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 30  # 종료 대기 시간
      containers:
      - name: app
        image: myapp:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]  # 종료 전 대기
        # Readiness/Liveness 프로브
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
```

```python
# Python에서 우아한 종료 구현
import signal
import sys
from flask import Flask

app = Flask(__name__)
shutdown_flag = False

def handle_shutdown(signum, frame):
    global shutdown_flag
    print("Received shutdown signal, finishing current requests...")
    shutdown_flag = True
    # 현재 요청 완료 후 종료
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_shutdown)
signal.signal(signal.SIGINT, handle_shutdown)

@app.before_request
def check_shutdown():
    if shutdown_flag:
        return "Service shutting down", 503
```

```python
# Celery 워커 - 작업을 큐로 반환
from celery import Celery

app = Celery('tasks')

@app.task(acks_late=True)  # 작업 완료 후에만 ACK
def long_running_task(data):
    # 갑작스러운 종료 시 작업이 큐로 반환됨
    process(data)
```

---

### X. Dev/Prod Parity

> **"개발, 스테이징, 프로덕션을 최대한 유사하게 유지"**

#### 세 가지 격차

| 격차 | 기존 앱 | 12 Factor 앱 |
|------|--------|-------------|
| **시간 격차** | 개발 후 몇 주/몇 달 후 배포 | 몇 시간 또는 몇 분 후 배포 |
| **인력 격차** | 개발자가 코드 작성, 운영팀이 배포 | 개발자가 배포에 밀접하게 참여 |
| **도구 격차** | 개발: SQLite, 프로덕션: PostgreSQL | 모든 환경에서 동일한 도구 사용 |

#### 왜 중요한가?

**도구 격차**가 특히 위험하다:
- 개발에서 SQLite, 프로덕션에서 PostgreSQL 사용 → 예상치 못한 버그
- 개발에서 메모리 큐, 프로덕션에서 RabbitMQ 사용 → 동작 차이

**"그냥 잘 작동할 거야"라는 생각은 위험하다.**

#### Docker/Kubernetes 적용 예제

```yaml
# docker-compose.yml - 로컬 개발 환경
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis

  db:
    image: postgres:15  # 프로덕션과 동일한 버전
    environment:
      POSTGRES_PASSWORD: pass
      POSTGRES_USER: user
      POSTGRES_DB: myapp

  redis:
    image: redis:7  # 프로덕션과 동일한 버전
```

```yaml
# 프로덕션 Kubernetes - 동일한 이미지와 백킹 서비스
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest  # 동일한 이미지
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: REDIS_URL
          value: "redis://redis-cluster:6379"
```

---

### XI. Logs

> **"로그를 이벤트 스트림으로 취급"**

#### 핵심 개념

**로그**는 실행 중인 모든 프로세스와 백킹 서비스의 출력 스트림에서 수집된 **시간순 이벤트의 집계**이다.

12 Factor 앱은 출력 스트림의 **라우팅이나 저장에 관여하지 않는다**. 앱은 로그를 **stdout과 stderr에 버퍼링 없이 출력**하기만 한다.

#### 로그 파일이 아닌 스트림인 이유

```
[앱] ──> stdout ──> [로그 라우터] ──> [로그 저장소/분석 시스템]
                         │
                         ├──> Elasticsearch
                         ├──> Splunk
                         ├──> CloudWatch
                         └──> Datadog
```

- 로컬 개발: 개발자가 터미널에서 직접 확인
- 스테이징/프로덕션: 로그 라우터가 수집하여 분석 시스템으로 전달

#### 장점

- 앱은 로그 저장 위치를 알 필요 없음
- 로그 저장소 변경 시 앱 코드 변경 불필요
- 중앙집중식 로그 관리 가능

#### Kubernetes 적용 예제

```yaml
# Fluentd DaemonSet으로 로그 수집
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  template:
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

```python
# Python 앱 - stdout으로 구조화된 로그 출력
import logging
import json
import sys

# JSON 형식 로거 설정
class JsonFormatter(logging.Formatter):
    def format(self, record):
        log_obj = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "message": record.getMessage(),
            "logger": record.name,
        }
        if record.exc_info:
            log_obj["exception"] = self.formatException(record.exc_info)
        return json.dumps(log_obj)

handler = logging.StreamHandler(sys.stdout)  # stdout으로 출력
handler.setFormatter(JsonFormatter())
logger = logging.getLogger()
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# 사용
logger.info("User logged in", extra={"user_id": 123})
```

---

### XII. Admin Processes

> **"관리 작업을 일회성 프로세스로 실행"**

#### 관리 작업의 예

- **데이터베이스 마이그레이션** (예: `python manage.py migrate`)
- **일회성 스크립트** 실행
- **REPL을 통한 라이브 검사** (예: `python manage.py shell`)
- **데이터 정리 또는 수정**

#### 핵심 규칙

1. **동일한 환경에서 실행**: 관리 프로세스는 앱의 일반 프로세스와 동일한 코드베이스와 설정을 사용
2. **동일한 릴리스**: 앱과 함께 배포된 코드로 실행
3. **의존성 격리**: 앱과 동일한 의존성 격리 기법 사용

#### 드리프트 방지

관리 코드를 앱과 함께 배포함으로써 **코드 드리프트**를 방지한다. 프로덕션 환경에서 실행되는 관리 스크립트가 로컬 개발 환경의 코드와 달라지는 것을 막는다.

#### Kubernetes 적용 예제

```yaml
# Job으로 데이터베이스 마이그레이션
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myapp:latest  # 앱과 동일한 이미지
        command: ["python", "manage.py", "migrate"]
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
      restartPolicy: Never
  backoffLimit: 3
```

```yaml
# CronJob으로 정기적인 관리 작업
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-job
spec:
  schedule: "0 2 * * *"  # 매일 새벽 2시
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: myapp:latest
            command: ["python", "manage.py", "cleanup_old_sessions"]
            envFrom:
            - configMapRef:
                name: app-config
          restartPolicy: OnFailure
```

```bash
# kubectl exec으로 일회성 관리 명령 실행
kubectl exec -it deployment/myapp -- python manage.py shell

# 또는 임시 Pod 생성
kubectl run admin-task --rm -it --image=myapp:latest \
  --env="DATABASE_URL=..." \
  -- python manage.py migrate
```

---

## 3. 실제 적용 예제

### 완전한 12 Factor 앱 구조

```
myapp/
├── Dockerfile              # 빌드/실행 분리
├── docker-compose.yml      # 로컬 개발 환경
├── requirements.txt        # 의존성 선언
├── src/
│   ├── app.py             # 메인 앱 (포트 바인딩)
│   ├── config.py          # 환경 변수 로드
│   ├── worker.py          # 워커 프로세스
│   └── tasks/
│       └── admin.py       # 관리 작업
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    ├── configmap.yaml
    ├── secrets.yaml
    ├── hpa.yaml
    └── jobs/
        └── migration.yaml
```

### 종합 Kubernetes 매니페스트

```yaml
# configmap.yaml - III. Config
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  LOG_LEVEL: "INFO"
  REDIS_HOST: "redis"
  DATABASE_NAME: "myapp"
---
# secrets.yaml - III. Config (민감 정보)
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  DATABASE_PASSWORD: "secret"
  API_KEY: "external-key"
---
# deployment.yaml - VI. Processes, VII. Port Binding, IX. Disposability
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-web
  labels:
    app: myapp
    component: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
      component: web
  template:
    metadata:
      labels:
        app: myapp
        component: web
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: web
        image: myregistry/myapp:v1.0.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: myapp-config
        - secretRef:
            name: myapp-secrets
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 15
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
---
# worker-deployment.yaml - VIII. Concurrency
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-worker
spec:
  replicas: 2
  template:
    spec:
      containers:
      - name: worker
        image: myregistry/myapp:v1.0.0
        command: ["celery", "-A", "tasks", "worker", "-l", "info"]
        envFrom:
        - configMapRef:
            name: myapp-config
        - secretRef:
            name: myapp-secrets
---
# hpa.yaml - VIII. Concurrency
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-web
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
# service.yaml - VII. Port Binding
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    component: web
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
# migration-job.yaml - XII. Admin Processes
apiVersion: batch/v1
kind: Job
metadata:
  name: myapp-migration
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: myregistry/myapp:v1.0.0
        command: ["python", "manage.py", "migrate"]
        envFrom:
        - configMapRef:
            name: myapp-config
        - secretRef:
            name: myapp-secrets
      restartPolicy: Never
```

---

## 4. Beyond the Twelve-Factor App

12 Factor App은 2011년에 만들어졌다. 그 이후 클라우드 네이티브 기술이 크게 발전하면서, Kevin Hoffman이 저술한 **"Beyond the Twelve-Factor App"**에서는 기존 12가지 원칙을 확장하여 **3가지 추가 원칙**을 제안했다.

### 추가된 3가지 원칙

#### XIII. API First

> **"API를 먼저 설계하라"**

**Contract-First 개발 패턴**의 확장으로, 애플리케이션의 경계(Edge)나 이음새(Seam)를 먼저 구축하는 데 집중한다.

```yaml
# OpenAPI 스펙 먼저 정의
openapi: 3.0.3
info:
  title: User Service API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      operationId: getUser
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
```

**장점**:
- 프론트엔드와 백엔드 팀의 병렬 개발 가능
- API 문서화가 자연스럽게 이루어짐
- 계약 기반 테스트 가능

#### XIV. Telemetry

> **"원격 측정(Telemetry)을 설계에 포함하라"**

로깅은 앱의 내부 구조에 초점을 맞추지만, **텔레메트리는 앱이 실제 환경에 배포된 후의 동작**에 초점을 맞춘다.

```yaml
# OpenTelemetry Collector 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  template:
    spec:
      containers:
      - name: collector
        image: otel/opentelemetry-collector:latest
        args: ["--config=/conf/otel-config.yaml"]
```

**세 가지 관점**:
- **앱 성능 모니터링(APM)**: 응답 시간, 처리량, 오류율
- **도메인별 텔레메트리**: 비즈니스 메트릭 (주문 수, 전환율 등)
- **헬스 및 시스템 메트릭**: CPU, 메모리, 디스크 사용량

#### XV. Authentication & Authorization

> **"보안을 처음부터 설계에 포함하라"**

보안은 나중에 추가하는 것이 아니라 **설계 단계에서부터 고려**해야 한다.

```yaml
# Istio AuthorizationPolicy 예제
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
spec:
  selector:
    matchLabels:
      app: myapp
  rules:
  - from:
    - source:
        requestPrincipals: ["*"]
    when:
    - key: request.auth.claims[iss]
      values: ["https://auth.example.com"]
```

**핵심 기술**:
- OAuth 2.0 / OpenID Connect
- JWT (JSON Web Tokens)
- RBAC (Role-Based Access Control)
- mTLS (Mutual TLS)

### 2025년 이후: 16 Factor App (AI 시대)

Google Cloud는 AI 애플리케이션을 위한 4가지 추가 원칙을 제안했다:

1. **대화형 메모리 관리**: AI의 컨텍스트 관리
2. **비결정적 동작 처리**: AI 응답의 가변성 처리
3. **AI 특화 보안**: 프롬프트 인젝션 등 새로운 위협 대응
4. **AI 모델 버전 관리**: 모델 업데이트와 롤백

---

## 5. 참고 자료

### 공식 문서
- [The Twelve-Factor App (공식 사이트)](https://12factor.net/)
- [Heroku - Twelve-Factor App Open Source Announcement](https://www.heroku.com/blog/heroku-open-sources-twelve-factor-app-definition/)
- [GitHub - heroku/12factor](https://github.com/heroku/12factor)

### 확장 자료
- [Beyond the Twelve-Factor App (O'Reilly Book)](https://www.oreilly.com/library/view/beyond-the-twelve-factor/9781492042631/)
- [IBM - 15-factor cloud-native applications](https://developer.ibm.com/articles/15-factor-applications/)
- [Google Cloud - 16 Factor App for AI](https://cloud.google.com/transform/from-the-twelve-to-sixteen-factor-app)

### 실무 적용 가이드
- [Red Hat - An illustrated guide to 12 Factor Apps](https://www.redhat.com/en/blog/12-factor-app)
- [Mirantis - How to build 12-factor apps using Kubernetes](https://www.mirantis.com/blog/how-do-you-build-12-factor-apps-using-kubernetes/)
- [VMware Tanzu - Twelve-Factor Apps](https://tanzu.vmware.com/content/blog/beyond-the-twelve-factor-app)

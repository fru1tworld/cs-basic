# Docker

## 컨테이너 vs VM

### 가상화의 두 가지 접근 방식

- 구분: 가상화 수준:
  - 가상 머신 (VM): 하드웨어 레벨 (Hypervisor)
  - 컨테이너: OS 레벨 (커널 공유)
- 구분: Guest OS:
  - 가상 머신 (VM): 전체 OS 포함
  - 컨테이너: 호스트 OS 커널 공유
- 구분: 이미지 크기:
  - 가상 머신 (VM): 수 GB ~ 수십 GB
  - 컨테이너: 수 MB ~ 수백 MB
- 구분: 부팅 시간:
  - 가상 머신 (VM): 수 분
  - 컨테이너: 수 초
- 구분: 성능 오버헤드:
  - 가상 머신 (VM): 높음
  - 컨테이너: 낮음
- 구분: 격리 수준:
  - 가상 머신 (VM): 강함 (완전한 OS 격리)
  - 컨테이너: 상대적으로 약함
- 구분: 이식성:
  - 가상 머신 (VM): 낮음
  - 컨테이너: 높음

### 아키텍처 비교

VM은 인프라와 호스트 OS 위에 하이퍼바이저를 두고, 각 VM에서 게스트 OS와 애플리케이션을 실행한다. App A, B, C가 각각 별도 VM에서 실행된다면 실행 파일과 라이브러리뿐 아니라 게스트 OS도 각각 필요하다.

컨테이너는 호스트 OS 위의 컨테이너 런타임을 통해 애플리케이션을 실행한다. App A, B, C의 실행 파일과 라이브러리는 분리하되, 호스트 OS의 커널은 공유한다. 이 차이가 VM과 컨테이너의 자원 사용량과 시작 시간에 영향을 준다.

### 컨테이너 격리 기술

컨테이너는 Linux 커널의 Namespaces와 Cgroups를 활용해 실행 환경을 분리하고 자원 사용을 제어한다. Namespaces가 프로세스에 보이는 환경을 나눈다면, Cgroups는 각 컨테이너가 사용할 수 있는 자원의 범위를 제한한다.

#### Namespaces (격리)

```bash
# 컨테이너가 사용하는 Linux Namespace 종류
- PID Namespace    : 프로세스 ID 격리
- Network Namespace: 네트워크 스택 격리 (IP, 라우팅 테이블, 포트)
- Mount Namespace  : 파일시스템 마운트 포인트 격리
- UTS Namespace    : 호스트명과 도메인명 격리
- IPC Namespace    : 프로세스간 통신 격리
- User Namespace   : 사용자 및 그룹 ID 격리
```

#### Cgroups (자원 제한)

```bash
# 컨테이너 리소스 제한 예시
docker run -d \
  --memory="512m" \            # 메모리 제한
  --cpus="1.5" \               # CPU 제한 (1.5 코어)
  --memory-swap="1g" \         # 스왑 포함 메모리 제한
  --cpu-shares=512 \           # CPU 공유 가중치
  nginx
```

## Docker 아키텍처

### Docker 엔진 구성요소

- Docker는 Client-Server 아키텍처를 사용하며, 다음과 같은 계층 구조로 동작함:

- Docker CLI
- (docker build, run, ps...)
- REST API (Unix Socket)
- /var/run/docker.sock
- Docker Daemon (dockerd)
- Image Mgmt, Network Mgmt, Volume Mgmt
- gRPC
- containerd
- (High-Level Container Runtime)
- 이미지 Pull/Push
- 컨테이너 생명주기 관리
- 스토리지 및 네트워크 관리
- containerd-shim
- (부모 프로세스 역할, daemon 재시작 시 컨테이너 유지)
- runc
- (Low-Level Container Runtime)
- OCI Runtime Specification 구현체
- 실제 컨테이너 프로세스 생성
- Namespaces, Cgroups 설정
- Linux Kernel
- (Namespaces, Cgroups, Union FS)

### 각 컴포넌트의 역할

- 컴포넌트: Docker CLI:
  - 역할: 사용자 명령어 인터페이스
  - 프로토콜: REST API
- 컴포넌트: Docker Daemon (dockerd):
  - 역할: API 서버, 이미지/컨테이너 관리
  - 프로토콜: Unix Socket
- 컴포넌트: containerd:
  - 역할: 컨테이너 생명주기, 이미지 관리
  - 프로토콜: gRPC
- 컴포넌트: containerd-shim:
  - 역할: 컨테이너와 containerd 사이의 중간 프로세스
  - 프로토콜: -
- 컴포넌트: runc:
  - 역할: 실제 컨테이너 생성 (OCI 표준 구현체)
  - 프로토콜: -

### OCI (Open Container Initiative) 표준

```yaml
# OCI 표준은 두 가지 스펙을 정의
1. Runtime Specification (runtime-spec)
   - 컨테이너 실행 방법 정의
   - config.json: 컨테이너 설정 (환경변수, 마운트, 리소스 제한 등)
   - rootfs: 컨테이너 루트 파일시스템

2. Image Specification (image-spec)
   - 컨테이너 이미지 형식 정의
   - Manifest, Config, Layers 구조
   - Docker, containerd, CRI-O 등에서 호환
```

### 대안 런타임

```bash
# runc 대안 런타임들
- gVisor (runsc): Google이 개발한 샌드박스 런타임, 추가 격리 계층 제공
- Kata Containers: 경량 VM 기반 런타임, 강력한 격리
- youki: Rust로 작성된 runc 대안, 더 빠르고 메모리 효율적
```

## Dockerfile Best Practice

### 기본 원칙

#### 최소 베이스 이미지 사용

```dockerfile
# Bad - 전체 OS 이미지
FROM ubuntu:22.04

# Good - 경량 이미지
FROM alpine:3.19

# Better - Distroless 이미지 (쉘 없음, 보안 강화)
FROM gcr.io/distroless/static-debian12
```

#### 레이어 최적화

```dockerfile
# Bad - 불필요한 레이어 생성
RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y pip
RUN apt-get clean

# Good - 단일 레이어로 결합 + 캐시 정리
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3 \
        pip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

#### 캐시 효율적인 레이어 순서

```dockerfile
# Bad - 소스 코드 변경 시 의존성도 다시 설치
COPY . /app
RUN pip install -r requirements.txt

# Good - 의존성 레이어를 먼저 배치
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app
```

### 보안 Best Practice

```dockerfile
# 1. Non-root 사용자 사용
FROM node:20-alpine
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001
USER nextjs

# 2. 시크릿 관리 - BuildKit secret 사용
# syntax=docker/dockerfile:1.4
RUN --mount=type=secret,id=github_token \
    GITHUB_TOKEN=$(cat /run/secrets/github_token) \
    npm install private-package

# 빌드 시: docker build --secret id=github_token,src=./token.txt .

# 3. 불필요한 파일 제외 (.dockerignore)
# .dockerignore 파일:
# .git
# node_modules
# *.log
# .env
# Dockerfile
# docker-compose.yml
```

### 실용적인 Dockerfile 예시 (Spring Boot)

```dockerfile
# syntax=docker/dockerfile:1.4
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /app

# Gradle Wrapper 및 빌드 스크립트 복사 (캐시 레이어)
COPY gradlew .
COPY gradle gradle
COPY build.gradle.kts settings.gradle.kts ./

# 의존성 다운로드 (캐시 레이어)
RUN ./gradlew dependencies --no-daemon

# 소스 코드 복사 및 빌드
COPY src src
RUN ./gradlew bootJar --no-daemon

# 최종 이미지
FROM eclipse-temurin:21-jre-alpine

RUN addgroup -g 1001 -S spring && \
    adduser -S spring -u 1001

WORKDIR /app

# 빌드 결과물만 복사
COPY --from=builder --chown=spring:spring /app/build/libs/*.jar app.jar

USER spring

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 멀티스테이지 빌드

### 멀티스테이지 빌드란?

애플리케이션을 빌드할 때는 컴파일러와 개발 의존성이 필요하지만, 실행할 때까지 모두 필요한 것은 아니다. Docker 17.05부터 지원하는 멀티스테이지 빌드는 하나의 Dockerfile에 여러 `FROM` 문을 두어 빌드 환경과 실행 환경을 분리한다.

### 핵심 이점

예를 들어 단일 스테이지 이미지에 빌드 도구, 소스 코드, 의존성, 최종 바이너리를 모두 담으면 880 MB가 된다. 멀티스테이지 빌드에서는 첫 번째 스테이지에서 빌드한 뒤, 두 번째 스테이지에 최종 바이너리와 실행 의존성만 복사한다. 이 예시에서는 빌드용 파일을 최종 이미지에서 제외해 크기가 428 MB로, 약 51% 줄어든다.

### Go 애플리케이션 예시

```dockerfile
# ==================== Build Stage ====================
FROM golang:1.22-alpine AS builder

# 보안: 최신 CA 인증서 설치
RUN apk add --no-cache ca-certificates git

WORKDIR /app

# 의존성 캐싱을 위해 go.mod, go.sum 먼저 복사
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# 소스 코드 복사 및 빌드
COPY . .

# CGO 비활성화로 정적 바이너리 생성
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app/server ./cmd/server

# ==================== Runtime Stage ====================
FROM scratch

# 빌더에서 CA 인증서 복사 (HTTPS 통신 필요 시)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 바이너리만 복사
COPY --from=builder /app/server /server

EXPOSE 8080

ENTRYPOINT ["/server"]
```

### Node.js 애플리케이션 예시

```dockerfile
# ==================== Dependencies Stage ====================
FROM node:20-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --only=production && npm cache clean --force

# ==================== Build Stage ====================
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build

# ==================== Runtime Stage ====================
FROM node:20-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production

# Non-root user 생성
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Production 의존성만 복사
COPY --from=deps --chown=nextjs:nodejs /app/node_modules ./node_modules

# 빌드 결과물 복사
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./

USER nextjs

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

### 특정 스테이지만 빌드

```bash
# 특정 스테이지를 타겟으로 빌드
docker build --target builder -t myapp:builder .

# 개발 스테이지와 프로덕션 스테이지 분리
docker build --target development -t myapp:dev .
docker build --target production -t myapp:prod .
```

### 병렬 빌드 (BuildKit)

```dockerfile
# BuildKit은 독립적인 스테이지를 병렬로 빌드
FROM alpine AS stage1
RUN sleep 10 && echo "stage1"

FROM alpine AS stage2
RUN sleep 10 && echo "stage2"

FROM alpine AS final
COPY --from=stage1 /etc/os-release /stage1.txt
COPY --from=stage2 /etc/os-release /stage2.txt
# stage1과 stage2가 병렬로 빌드됨 (총 10초)
```

## Docker Compose

### Docker Compose란?

- Docker Compose는 멀티 컨테이너 애플리케이션을 정의하고 실행하기 위한 도구임. YAML 파일로 서비스, 네트워크, 볼륨을 정의함.

### 기본 구조

```yaml
# docker-compose.yml (Compose V2 형식)
version: '3.8'

services:
  # 서비스 정의
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://db:5432/mydb
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend
    volumes:
      - app-data:/app/data

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5

networks:
  backend:
    driver: bridge

volumes:
  app-data:
  postgres-data:
```

### 실전 예시: Spring Boot + PostgreSQL + Redis

```yaml
version: '3.8'

services:
  # Spring Boot Application
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: spring-api
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/appdb
      - SPRING_DATASOURCE_USERNAME=appuser
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - SPRING_REDIS_HOST=redis
      - SPRING_REDIS_PORT=6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: postgres-db
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: unless-stopped

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    networks:
      - app-network
    restart: unless-stopped

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
```

### 주요 명령어

```bash
# 서비스 시작 (백그라운드)
docker compose up -d

# 서비스 시작 + 빌드
docker compose up -d --build

# 특정 서비스만 시작
docker compose up -d api postgres

# 로그 확인
docker compose logs -f api

# 서비스 스케일링
docker compose up -d --scale api=3

# 서비스 중지 (컨테이너 유지)
docker compose stop

# 서비스 중지 + 컨테이너 삭제
docker compose down

# 볼륨까지 삭제
docker compose down -v

# 서비스 상태 확인
docker compose ps

# 서비스에 명령 실행
docker compose exec api sh
```

### Override 파일 활용

```yaml
# docker-compose.yml (기본)
version: '3.8'
services:
  api:
    image: myapp:latest
    ports:
      - "8080:8080"

# docker-compose.override.yml (개발 환경, 자동 적용)
version: '3.8'
services:
  api:
    build: .
    volumes:
      - ./src:/app/src
    environment:
      - DEBUG=true

# docker-compose.prod.yml (프로덕션 환경)
version: '3.8'
services:
  api:
    deploy:
      replicas: 3
    environment:
      - DEBUG=false
```

```bash
# 프로덕션 환경으로 실행
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## 이미지 레이어와 캐싱

### 이미지 레이어 구조

- Docker 이미지는 읽기 전용 레이어들의 스택으로 구성됨.

- Container Layer (R/W)
- (컨테이너 실행 시 생성)
- Layer 5: CMD ["python", "app.py"]
- Layer 4: COPY . /app
- Layer 3: RUN pip install -r requirements.txt
- Layer 2: COPY requirements.txt /app/
- Layer 1: FROM python:3.11-slim
- (Base Image Layers)
- 읽기 전용 (Read-Only)
- 각 레이어는 이전 레이어와의 차이만 저장 (Union File System)

### Union File System (OverlayFS)

```bash
# Docker가 사용하는 스토리지 드라이버 확인
docker info | grep "Storage Driver"

# OverlayFS 구조
Lower Layer (읽기 전용)
    ↓ merged
Upper Layer (쓰기 가능)
    ↓
Merged View (컨테이너가 보는 파일시스템)
```

### 레이어 캐시 동작 원리

```dockerfile
# Layer 1: 캐시됨 (베이스 이미지 변경 없음)
FROM node:20-alpine

# Layer 2: 캐시됨 (WORKDIR 변경 없음)
WORKDIR /app

# Layer 3: 캐시됨 (package.json 변경 없으면)
COPY package.json package-lock.json ./

# Layer 4: 캐시됨 (Layer 3이 캐시되면)
RUN npm ci

# Layer 5: 항상 재빌드 (소스 코드 변경 시)
COPY . .

# Layer 6: 재빌드 (Layer 5가 재빌드되면)
RUN npm run build
```

이전 레이어가 변경되면 그 뒤에 쌓이는 레이어의 캐시도 무효화된다. 따라서 자주 바뀌는 소스 코드를 복사하기 전에, 상대적으로 덜 바뀌는 의존성을 설치하는 순서로 Dockerfile을 작성한다.

캐시 사용 여부를 판단하는 기준은 명령어에 따라 다르다. `COPY`와 `ADD`는 파일 체크섬을 비교하고, `RUN`은 실행 결과물이 아니라 명령어 문자열을 비교한다.

### 캐시 최적화 전략

```dockerfile
# 전략 1: 변경 빈도에 따른 레이어 순서
# 자주 변경 X → 자주 변경 O 순서로 배치

# 전략 2: 의존성과 소스 코드 분리
COPY go.mod go.sum ./          # 의존성 정의 (자주 안 변함)
RUN go mod download            # 의존성 다운로드 (캐시됨)
COPY . .                       # 소스 코드 (자주 변함)

# 전략 3: 빌드 컨텍스트 최소화 (.dockerignore)
# .dockerignore 파일:
.git
node_modules
dist
*.log
.env*
Dockerfile*
docker-compose*
```

### 레이어 분석

```bash
# 이미지 레이어 히스토리 확인
docker history myapp:latest

# 레이어별 크기 상세 분석
docker history --no-trunc myapp:latest

# dive 도구로 레이어 분석 (추천)
dive myapp:latest

# 이미지 크기 확인
docker images myapp --format "{{.Repository}}:{{.Tag}} - {{.Size}}"
```

## Docker 네트워크와 볼륨

### Docker 네트워크 유형

- 네트워크 드라이버: bridge:
  - 설명: 기본 드라이버, 단일 호스트 내 컨테이너 통신
  - 사용 사례: 대부분의 일반적인 경우
- 네트워크 드라이버: host:
  - 설명: 호스트 네트워크 스택 직접 사용
  - 사용 사례: 고성능이 필요한 경우
- 네트워크 드라이버: overlay:
  - 설명: 여러 Docker 호스트 간 통신
  - 사용 사례: Swarm, 멀티호스트
- 네트워크 드라이버: macvlan:
  - 설명: 컨테이너에 MAC 주소 할당
  - 사용 사례: 레거시 애플리케이션
- 네트워크 드라이버: none:
  - 설명: 네트워킹 비활성화
  - 사용 사례: 완전한 격리 필요 시

### Bridge 네트워크

```bash
# 사용자 정의 브릿지 네트워크 생성
docker network create --driver bridge \
  --subnet 172.20.0.0/16 \
  --gateway 172.20.0.1 \
  my-network

# 네트워크에 컨테이너 연결
docker run -d --name web --network my-network nginx
docker run -d --name app --network my-network myapp

# 같은 네트워크 내 컨테이너는 이름으로 통신 가능 (내장 DNS)
# app 컨테이너에서: curl http://web:80
```

- Docker Host
- User-defined Bridge (172.20.0.0/16)
- web, app
- 172.20.0.2, ← →, 172.20.0.3
- DNS
- Resolution
- eth0
- NAT/Bridge

### Host 네트워크

```bash
# Host 네트워크 사용 (Linux만 지원)
docker run -d --network host nginx

# 컨테이너가 호스트의 80 포트에 직접 바인딩
# 네트워크 오버헤드 없음, 최대 성능
# 포트 충돌 주의 필요
```

### Overlay 네트워크 (Swarm/Multi-host)

```bash
# Swarm 모드에서 overlay 네트워크 생성
docker network create --driver overlay \
  --attachable \
  --subnet 10.0.0.0/24 \
  my-overlay

# 다른 호스트의 컨테이너와 통신 가능
# VXLAN 터널링 사용
```

### Docker 볼륨

#### 볼륨 유형

```bash
# 1. Named Volume (권장)
docker volume create my-data
docker run -v my-data:/app/data nginx

# 2. Bind Mount
docker run -v /host/path:/container/path nginx

# 3. tmpfs Mount (메모리)
docker run --tmpfs /app/cache nginx
```

컨테이너의 `/container/path`에 데이터를 연결하더라도, 데이터를 어디에 두고 누가 관리하는지에 따라 용도가 달라진다.

| 종류 | 저장 위치와 관리 | 주로 사용하는 상황 |
| --- | --- | --- |
| Named Volume | Docker가 관리하는 저장 공간 | 영구 저장과 이식성이 필요한 데이터 |
| Bind Mount | 호스트의 지정 경로 | 파일 변경을 실시간으로 반영해야 하는 개발 환경. 호스트 경로에 의존한다. |
| tmpfs | RAM | 빠른 I/O가 필요한 임시 데이터. 컨테이너가 종료되면 삭제된다. |

### 볼륨 관리 명령어

```bash
# 볼륨 생성
docker volume create --name postgres-data

# 볼륨 목록
docker volume ls

# 볼륨 상세 정보
docker volume inspect postgres-data

# 사용하지 않는 볼륨 정리
docker volume prune

# 볼륨 삭제
docker volume rm postgres-data

# 볼륨과 함께 컨테이너 실행
docker run -d \
  --name postgres \
  -v postgres-data:/var/lib/postgresql/data \
  -v ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro \
  postgres:16
```

### 볼륨 백업과 복원

```bash
# 볼륨 백업
docker run --rm \
  -v postgres-data:/data:ro \
  -v $(pwd):/backup \
  alpine tar cvf /backup/postgres-backup.tar /data

# 볼륨 복원
docker run --rm \
  -v postgres-data:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xvf /backup/postgres-backup.tar --strip 1"
```

## 참고 자료

- [Docker 공식 문서](https://docs.docker.com/)
- [Docker Best Practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [containerd 공식 사이트](https://containerd.io/)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [Docker Networking Deep Dive](https://docs.docker.com/engine/network/)

## 관련 학습

- [Kubernetes](02-Kubernetes.md)

# Docker

## 목차
1. [컨테이너 vs VM](#1-컨테이너-vs-vm)
2. [Docker 아키텍처](#2-docker-아키텍처)
3. [Dockerfile Best Practice](#3-dockerfile-best-practice)
4. [멀티스테이지 빌드](#4-멀티스테이지-빌드)
5. [Docker Compose](#5-docker-compose)
6. [이미지 레이어와 캐싱](#6-이미지-레이어와-캐싱)
7. [Docker 네트워크와 볼륨](#7-docker-네트워크와-볼륨)
---

## 1. 컨테이너 vs VM

### 1.1 가상화의 두 가지 접근 방식

| 구분 | 가상 머신 (VM) | 컨테이너 |
|------|---------------|---------|
| **가상화 수준** | 하드웨어 레벨 (Hypervisor) | OS 레벨 (커널 공유) |
| **Guest OS** | 전체 OS 포함 | 호스트 OS 커널 공유 |
| **이미지 크기** | 수 GB ~ 수십 GB | 수 MB ~ 수백 MB |
| **부팅 시간** | 수 분 | 수 초 |
| **성능 오버헤드** | 높음 | 낮음 |
| **격리 수준** | 강함 (완전한 OS 격리) | 상대적으로 약함 |
| **이식성** | 낮음 | 높음 |

### 1.2 아키텍처 비교

```
┌──────────────────────────────────────────────────────────────────┐
│                    Virtual Machine Architecture                   │
├──────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │   App A  │  │   App B  │  │   App C  │                       │
│  ├──────────┤  ├──────────┤  ├──────────┤                       │
│  │  Bins/   │  │  Bins/   │  │  Bins/   │                       │
│  │  Libs    │  │  Libs    │  │  Libs    │                       │
│  ├──────────┤  ├──────────┤  ├──────────┤                       │
│  │ Guest OS │  │ Guest OS │  │ Guest OS │  ← 각 VM마다 OS 필요  │
│  └──────────┘  └──────────┘  └──────────┘                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      Hypervisor                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      Host OS                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Infrastructure                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                     Container Architecture                        │
├──────────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │   App A  │  │   App B  │  │   App C  │                       │
│  ├──────────┤  ├──────────┤  ├──────────┤                       │
│  │  Bins/   │  │  Bins/   │  │  Bins/   │                       │
│  │  Libs    │  │  Libs    │  │  Libs    │                       │
│  └──────────┘  └──────────┘  └──────────┘                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Container Runtime                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      Host OS                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Infrastructure                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 컨테이너 격리 기술

컨테이너는 Linux 커널의 두 가지 핵심 기술을 활용합니다:

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

---

## 2. Docker 아키텍처

### 2.1 Docker 엔진 구성요소

Docker는 Client-Server 아키텍처를 사용하며, 다음과 같은 계층 구조로 동작합니다:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker CLI                                │
│                    (docker build, run, ps...)                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ REST API (Unix Socket)
                              │ /var/run/docker.sock
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Docker Daemon (dockerd)                      │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Image Mgmt   │  │ Network Mgmt │  │ Volume Mgmt          │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ gRPC
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        containerd                                │
│                 (High-Level Container Runtime)                   │
│                                                                  │
│  - 이미지 Pull/Push                                              │
│  - 컨테이너 생명주기 관리                                          │
│  - 스토리지 및 네트워크 관리                                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      containerd-shim                             │
│          (부모 프로세스 역할, daemon 재시작 시 컨테이너 유지)         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          runc                                    │
│                  (Low-Level Container Runtime)                   │
│                                                                  │
│  - OCI Runtime Specification 구현체                              │
│  - 실제 컨테이너 프로세스 생성                                      │
│  - Namespaces, Cgroups 설정                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Linux Kernel                                │
│              (Namespaces, Cgroups, Union FS)                    │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 각 컴포넌트의 역할

| 컴포넌트 | 역할 | 프로토콜 |
|---------|------|---------|
| **Docker CLI** | 사용자 명령어 인터페이스 | REST API |
| **Docker Daemon (dockerd)** | API 서버, 이미지/컨테이너 관리 | Unix Socket |
| **containerd** | 컨테이너 생명주기, 이미지 관리 | gRPC |
| **containerd-shim** | 컨테이너와 containerd 사이의 중간 프로세스 | - |
| **runc** | 실제 컨테이너 생성 (OCI 표준 구현체) | - |

### 2.3 OCI (Open Container Initiative) 표준

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

### 2.4 대안 런타임

```bash
# runc 대안 런타임들
- gVisor (runsc): Google이 개발한 샌드박스 런타임, 추가 격리 계층 제공
- Kata Containers: 경량 VM 기반 런타임, 강력한 격리
- youki: Rust로 작성된 runc 대안, 더 빠르고 메모리 효율적
```

---

## 3. Dockerfile Best Practice

### 3.1 기본 원칙

#### 1) 최소 베이스 이미지 사용
```dockerfile
# Bad - 전체 OS 이미지
FROM ubuntu:22.04

# Good - 경량 이미지
FROM alpine:3.19

# Better - Distroless 이미지 (쉘 없음, 보안 강화)
FROM gcr.io/distroless/static-debian12
```

#### 2) 레이어 최적화
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

#### 3) 캐시 효율적인 레이어 순서
```dockerfile
# Bad - 소스 코드 변경 시 의존성도 다시 설치
COPY . /app
RUN pip install -r requirements.txt

# Good - 의존성 레이어를 먼저 배치
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app
```

### 3.2 보안 Best Practice

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

### 3.3 실용적인 Dockerfile 예시 (Spring Boot)

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

---

## 4. 멀티스테이지 빌드

### 4.1 멀티스테이지 빌드란?

멀티스테이지 빌드는 하나의 Dockerfile에서 여러 개의 FROM 문을 사용하여 빌드 환경과 실행 환경을 분리하는 기법입니다. Docker 17.05부터 지원됩니다.

### 4.2 핵심 이점

```
┌─────────────────────────────────────────────────────────────────┐
│              Single Stage Build (Before)                         │
├─────────────────────────────────────────────────────────────────┤
│  Build Tools + Source Code + Dependencies + Final Binary         │
│                        = 880 MB                                  │
└─────────────────────────────────────────────────────────────────┘

                              ▼

┌─────────────────────────────────────────────────────────────────┐
│              Multi-Stage Build (After)                           │
├─────────────────────────────────────────────────────────────────┤
│  Stage 1: Build     │  Stage 2: Runtime                         │
│  ─────────────────  │  ─────────────────                        │
│  Build Tools        │  Final Binary Only                        │
│  Source Code        │  Runtime Dependencies                     │
│  Dependencies       │                                           │
│  (discarded)        │  = 428 MB (51% 감소)                      │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Go 애플리케이션 예시

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

### 4.4 Node.js 애플리케이션 예시

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

### 4.5 특정 스테이지만 빌드

```bash
# 특정 스테이지를 타겟으로 빌드
docker build --target builder -t myapp:builder .

# 개발 스테이지와 프로덕션 스테이지 분리
docker build --target development -t myapp:dev .
docker build --target production -t myapp:prod .
```

### 4.6 병렬 빌드 (BuildKit)

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

---

## 5. Docker Compose

### 5.1 Docker Compose란?

Docker Compose는 멀티 컨테이너 애플리케이션을 정의하고 실행하기 위한 도구입니다. YAML 파일로 서비스, 네트워크, 볼륨을 정의합니다.

### 5.2 기본 구조

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

### 5.3 실전 예시: Spring Boot + PostgreSQL + Redis

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

### 5.4 주요 명령어

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

### 5.5 Override 파일 활용

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

---

## 6. 이미지 레이어와 캐싱

### 6.1 이미지 레이어 구조

Docker 이미지는 읽기 전용 레이어들의 스택으로 구성됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Container Layer (R/W)                        │
│                    (컨테이너 실행 시 생성)                         │
├─────────────────────────────────────────────────────────────────┤
│                     Layer 5: CMD ["python", "app.py"]            │
├─────────────────────────────────────────────────────────────────┤
│                     Layer 4: COPY . /app                         │
├─────────────────────────────────────────────────────────────────┤
│                     Layer 3: RUN pip install -r requirements.txt │
├─────────────────────────────────────────────────────────────────┤
│                     Layer 2: COPY requirements.txt /app/         │
├─────────────────────────────────────────────────────────────────┤
│                     Layer 1: FROM python:3.11-slim              │
│                     (Base Image Layers)                          │
└─────────────────────────────────────────────────────────────────┘
        ▲
        │ 읽기 전용 (Read-Only)
        │ 각 레이어는 이전 레이어와의 차이만 저장 (Union File System)
```

### 6.2 Union File System (OverlayFS)

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

### 6.3 레이어 캐시 동작 원리

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

```
캐시 무효화 규칙:
1. 이전 레이어가 변경되면 이후 모든 레이어 캐시 무효화
2. COPY/ADD 명령어: 파일 체크섬 비교
3. RUN 명령어: 명령어 문자열 비교 (결과물이 아님!)
```

### 6.4 캐시 최적화 전략

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

### 6.5 레이어 분석

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

---

## 7. Docker 네트워크와 볼륨

### 7.1 Docker 네트워크 유형

| 네트워크 드라이버 | 설명 | 사용 사례 |
|-----------------|------|----------|
| **bridge** | 기본 드라이버, 단일 호스트 내 컨테이너 통신 | 대부분의 일반적인 경우 |
| **host** | 호스트 네트워크 스택 직접 사용 | 고성능이 필요한 경우 |
| **overlay** | 여러 Docker 호스트 간 통신 | Swarm, 멀티호스트 |
| **macvlan** | 컨테이너에 MAC 주소 할당 | 레거시 애플리케이션 |
| **none** | 네트워킹 비활성화 | 완전한 격리 필요 시 |

### 7.2 Bridge 네트워크

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

```
┌─────────────────────────────────────────────────────────────────┐
│                         Docker Host                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              User-defined Bridge (172.20.0.0/16)        │    │
│  │                                                          │    │
│  │   ┌──────────────┐            ┌──────────────┐          │    │
│  │   │     web      │            │     app      │          │    │
│  │   │ 172.20.0.2   │◄──────────►│ 172.20.0.3   │          │    │
│  │   └──────────────┘   DNS      └──────────────┘          │    │
│  │                    Resolution                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                          │                                       │
│                    ┌─────┴─────┐                                 │
│                    │  eth0     │                                 │
│                    │ NAT/Bridge│                                 │
│                    └───────────┘                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 Host 네트워크

```bash
# Host 네트워크 사용 (Linux만 지원)
docker run -d --network host nginx

# 컨테이너가 호스트의 80 포트에 직접 바인딩
# 네트워크 오버헤드 없음, 최대 성능
# 포트 충돌 주의 필요
```

### 7.4 Overlay 네트워크 (Swarm/Multi-host)

```bash
# Swarm 모드에서 overlay 네트워크 생성
docker network create --driver overlay \
  --attachable \
  --subnet 10.0.0.0/24 \
  my-overlay

# 다른 호스트의 컨테이너와 통신 가능
# VXLAN 터널링 사용
```

### 7.5 Docker 볼륨

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

```
┌─────────────────────────────────────────────────────────────────┐
│                       Volume Types                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Named Volume          Bind Mount           tmpfs               │
│  ┌─────────┐          ┌─────────┐          ┌─────────┐         │
│  │ Docker  │          │  Host   │          │  RAM    │         │
│  │ managed │          │  Path   │          │         │         │
│  └────┬────┘          └────┬────┘          └────┬────┘         │
│       │                    │                    │               │
│       ▼                    ▼                    ▼               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Container                            │   │
│  │                  /container/path                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  - 영구 저장         - 개발 시 유용       - 임시 데이터          │
│  - Docker 관리       - 실시간 반영        - 빠른 I/O            │
│  - 이식성 높음       - 호스트 의존적      - 컨테이너 종료 시 삭제 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.6 볼륨 관리 명령어

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

### 7.7 볼륨 백업과 복원

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

---

## 참고 자료

- [Docker 공식 문서](https://docs.docker.com/)
- [Docker Best Practices](https://docs.docker.com/build/building/best-practices/)
- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [containerd 공식 사이트](https://containerd.io/)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [Docker Networking Deep Dive](https://docs.docker.com/engine/network/)

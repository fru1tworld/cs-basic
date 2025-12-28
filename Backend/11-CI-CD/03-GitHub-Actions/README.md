# GitHub Actions

## 목차
1. [Workflow 문법](#1-workflow-문법)
2. [재사용 가능한 Workflow](#2-재사용-가능한-workflow)
3. [Self-hosted Runner](#3-self-hosted-runner)
4. [Secrets 관리](#4-secrets-관리)
5. [Matrix Build](#5-matrix-build)
6. [고급 패턴 및 최적화](#6-고급-패턴-및-최적화)
---

## 1. Workflow 문법

### 1.1 Workflow 기본 구조

GitHub Actions Workflow는 `.github/workflows/` 디렉토리에 YAML 파일로 정의된다.

```yaml
# .github/workflows/ci.yml
name: CI Pipeline                    # Workflow 이름

on:                                  # 트리거 이벤트
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:                 # 수동 실행 허용
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

env:                                 # Workflow 전역 환경 변수
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:                                # 작업 정의
  build:
    name: Build Application
    runs-on: ubuntu-latest           # 실행 환경

    permissions:                     # GITHUB_TOKEN 권한
      contents: read
      packages: write

    outputs:                         # 다른 Job으로 전달할 출력
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build
        run: |
          echo "Building..."
```

### 1.2 트리거 이벤트 상세

```yaml
on:
  # Push 이벤트
  push:
    branches:
      - main
      - 'releases/**'           # 와일드카드 패턴
    branches-ignore:
      - 'feature/**'
    paths:
      - 'src/**'                # 특정 경로 변경 시만
      - '!src/**/*.md'          # 제외 패턴
    tags:
      - 'v*'                    # 태그 패턴

  # Pull Request 이벤트
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main]

  # 스케줄 (cron)
  schedule:
    - cron: '0 2 * * *'         # 매일 UTC 02:00

  # 다른 Workflow 완료 시
  workflow_run:
    workflows: ["Build"]
    types: [completed]
    branches: [main]

  # 외부 이벤트
  repository_dispatch:
    types: [deploy-command]

  # 이슈/PR 코멘트
  issue_comment:
    types: [created]
```

### 1.3 조건부 실행 (Conditionals)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    # Job 레벨 조건
    if: github.event_name == 'push' || github.event.pull_request.draft == false

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Step 레벨 조건
      - name: Deploy to Production
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: ./deploy.sh production

      # 이전 Step 성공/실패에 따른 실행
      - name: Notify on Success
        if: success()
        run: echo "Build succeeded!"

      - name: Notify on Failure
        if: failure()
        run: echo "Build failed!"

      # 항상 실행 (cleanup 등)
      - name: Cleanup
        if: always()
        run: ./cleanup.sh

      # 취소되지 않은 경우 실행
      - name: Final Step
        if: "!cancelled()"
        run: echo "Not cancelled"

  # 이전 Job 상태에 따른 조건
  notify:
    needs: [build, test]
    if: always() && (needs.build.result == 'failure' || needs.test.result == 'failure')
    runs-on: ubuntu-latest
    steps:
      - name: Send Failure Notification
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -H 'Content-Type: application/json' \
            -d '{"text": "Build or Test Failed!"}'
```

### 1.4 컨텍스트와 표현식

```yaml
jobs:
  context-demo:
    runs-on: ubuntu-latest
    steps:
      - name: GitHub Context
        run: |
          echo "Repository: ${{ github.repository }}"
          echo "Branch: ${{ github.ref_name }}"
          echo "SHA: ${{ github.sha }}"
          echo "Actor: ${{ github.actor }}"
          echo "Event: ${{ github.event_name }}"
          echo "Workflow: ${{ github.workflow }}"
          echo "Run ID: ${{ github.run_id }}"
          echo "Run Number: ${{ github.run_number }}"

      - name: Expressions
        run: |
          # 문자열 비교
          echo "Is main: ${{ github.ref == 'refs/heads/main' }}"

          # contains 함수
          echo "Has hotfix: ${{ contains(github.ref, 'hotfix') }}"

          # startsWith / endsWith
          echo "Starts with feature: ${{ startsWith(github.ref, 'refs/heads/feature') }}"

          # 삼항 연산자 형태
          echo "Env: ${{ github.ref == 'refs/heads/main' && 'production' || 'staging' }}"

          # JSON 처리
          echo "Event JSON: ${{ toJSON(github.event) }}"

          # 배열/객체 접근
          echo "PR Title: ${{ github.event.pull_request.title }}"
```

### 1.5 환경변수와 시크릿

```yaml
jobs:
  env-demo:
    runs-on: ubuntu-latest

    # Job 레벨 환경변수
    env:
      NODE_ENV: production
      API_URL: https://api.example.com

    steps:
      - name: Set Dynamic Environment Variables
        run: |
          # Step 내에서 환경변수 설정 (이후 Step에서 사용)
          echo "BUILD_VERSION=$(date +%Y%m%d%H%M%S)" >> $GITHUB_ENV
          echo "COMMIT_SHORT=$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_ENV

      - name: Use Environment Variables
        env:
          # Step 레벨 환경변수
          SECRET_KEY: ${{ secrets.SECRET_KEY }}
          GH_TOKEN: ${{ github.token }}
        run: |
          echo "Build Version: $BUILD_VERSION"
          echo "Commit: $COMMIT_SHORT"
          echo "Node Env: $NODE_ENV"

      - name: Mask Sensitive Values
        run: |
          # 로그에서 마스킹
          echo "::add-mask::${{ secrets.API_KEY }}"

          # 동적으로 생성된 값 마스킹
          SENSITIVE_VALUE=$(./generate-token.sh)
          echo "::add-mask::$SENSITIVE_VALUE"
          echo "Token: $SENSITIVE_VALUE"  # 로그에 ***로 표시됨
```

---

## 2. 재사용 가능한 Workflow

### 2.1 개념

**재사용 가능한 Workflow (Reusable Workflows)** 는 다른 Workflow에서 호출할 수 있는 템플릿 Workflow이다. 2025년 기준으로 최대 10단계 중첩과 50개의 Workflow 호출이 가능하다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Reusable Workflow vs Composite Action                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Reusable Workflow                    Composite Action                   │
│  ┌─────────────────────┐              ┌─────────────────────┐           │
│  │  workflow_call      │              │  action.yml         │           │
│  │  ─────────────────  │              │  ─────────────────  │           │
│  │  Job 1              │              │  Step 1             │           │
│  │    Step 1           │              │  Step 2             │           │
│  │    Step 2           │              │  Step 3             │           │
│  │  Job 2              │              │                     │           │
│  │    Step 1           │              │                     │           │
│  └─────────────────────┘              └─────────────────────┘           │
│                                                                          │
│  • 전체 Job을 재사용                  • Step 집합을 재사용               │
│  • Job 내에서 직접 호출               • Step 내에서 사용                  │
│  • 독립적인 실행 환경                 • 호출자의 실행 환경 공유            │
│  • Pipeline Template 역할            • Task Template 역할                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Reusable Workflow 정의

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build Workflow

on:
  workflow_call:
    # 입력 파라미터
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        type: string
      node-version:
        description: 'Node.js version'
        required: false
        type: string
        default: '20'
      enable-cache:
        description: 'Enable npm cache'
        required: false
        type: boolean
        default: true

    # 시크릿 (명시적 전달)
    secrets:
      npm-token:
        description: 'NPM authentication token'
        required: true
      deploy-key:
        description: 'Deployment key'
        required: false

    # 출력
    outputs:
      artifact-url:
        description: 'URL of the built artifact'
        value: ${{ jobs.build.outputs.artifact-url }}
      build-version:
        description: 'Build version'
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    name: Build for ${{ inputs.environment }}
    runs-on: ubuntu-latest

    outputs:
      artifact-url: ${{ steps.upload.outputs.artifact-url }}
      version: ${{ steps.version.outputs.version }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: ${{ inputs.enable-cache && 'npm' || '' }}

      - name: Configure NPM
        run: |
          echo "//registry.npmjs.org/:_authToken=${{ secrets.npm-token }}" > ~/.npmrc

      - name: Install Dependencies
        run: npm ci

      - name: Build
        run: npm run build
        env:
          NODE_ENV: ${{ inputs.environment }}

      - name: Generate Version
        id: version
        run: |
          VERSION="${{ github.sha }}-$(date +%Y%m%d%H%M%S)"
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - name: Upload Artifact
        id: upload
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ inputs.environment }}-${{ steps.version.outputs.version }}
          path: dist/

      - name: Set Artifact URL
        run: |
          echo "artifact-url=https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}" >> $GITHUB_OUTPUT
```

### 2.3 Reusable Workflow 호출

```yaml
# .github/workflows/main.yml
name: Main CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # Reusable Workflow 호출
  build-staging:
    uses: ./.github/workflows/reusable-build.yml
    with:
      environment: staging
      node-version: '20'
      enable-cache: true
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
      deploy-key: ${{ secrets.STAGING_DEPLOY_KEY }}

  # 다른 저장소의 Reusable Workflow 호출
  build-production:
    if: github.ref == 'refs/heads/main'
    uses: my-org/shared-workflows/.github/workflows/build.yml@v1.0.0
    with:
      environment: production
    secrets: inherit  # 모든 시크릿 전달

  # 출력 사용
  deploy:
    needs: [build-staging]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Deploying version: ${{ needs.build-staging.outputs.build-version }}"
          echo "Artifact: ${{ needs.build-staging.outputs.artifact-url }}"
```

### 2.4 Composite Action 정의

```yaml
# .github/actions/setup-environment/action.yml
name: 'Setup Environment'
description: 'Sets up the development environment with Node.js and dependencies'

inputs:
  node-version:
    description: 'Node.js version to use'
    required: false
    default: '20'
  install-deps:
    description: 'Whether to install dependencies'
    required: false
    default: 'true'
  working-directory:
    description: 'Working directory'
    required: false
    default: '.'

outputs:
  node-version:
    description: 'Actual Node.js version installed'
    value: ${{ steps.node.outputs.node-version }}
  cache-hit:
    description: 'Whether cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      id: node
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Get npm cache directory
      id: npm-cache-dir
      shell: bash
      run: echo "dir=$(npm config get cache)" >> $GITHUB_OUTPUT

    - name: Cache npm dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: ${{ steps.npm-cache-dir.outputs.dir }}
        key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          npm-${{ runner.os }}-

    - name: Install Dependencies
      if: inputs.install-deps == 'true'
      shell: bash
      working-directory: ${{ inputs.working-directory }}
      run: npm ci

    - name: Verify Installation
      shell: bash
      run: |
        echo "Node.js version: $(node --version)"
        echo "npm version: $(npm --version)"
```

### 2.5 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Reusable Workflow 베스트 프랙티스                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 버전 관리                                                            │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  # 프로덕션: commit SHA 또는 태그 사용                          │ │
│     │  uses: org/workflows/.github/workflows/build.yml@v1.0.0        │ │
│     │  uses: org/workflows/.github/workflows/build.yml@abc123def     │ │
│     │                                                                 │ │
│     │  # 개발/테스트: 브랜치 사용                                     │ │
│     │  uses: org/workflows/.github/workflows/build.yml@feature-xyz   │ │
│     │                                                                 │ │
│     │  # 피하기: main 직접 참조 (breaking change 위험)                │ │
│     │  uses: org/workflows/.github/workflows/build.yml@main          │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  2. 중앙화된 관리                                                        │
│     • 공유 Workflow는 전용 저장소에서 관리                               │
│     • CODEOWNERS로 변경 승인 프로세스 적용                               │
│     • 시맨틱 버저닝 사용                                                 │
│                                                                          │
│  3. 역할 분리                                                            │
│     • Reusable Workflow: Pipeline 템플릿 (Job 오케스트레이션)           │
│     • Composite Action: Task 템플릿 (Step 집합)                         │
│     • 책임 혼합 피하기                                                   │
│                                                                          │
│  4. 테스트                                                               │
│     • 브랜치 기반 테스트 후 머지                                         │
│     • Sandbox 저장소에서 검증                                            │
│     • 단위 테스트 포함 (act 또는 Docker 활용)                            │
│                                                                          │
│  5. 시크릿 전달                                                          │
│     • 명시적 secrets 정의 권장                                           │
│     • secrets: inherit는 신중하게 사용                                   │
│     • 환경변수는 inputs로 전달                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Self-hosted Runner

### 3.1 개념

**Self-hosted Runner**는 직접 호스팅하는 GitHub Actions 실행 환경으로, GitHub-hosted Runner와 달리 하드웨어, OS, 소프트웨어를 자유롭게 구성할 수 있다.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  GitHub-hosted vs Self-hosted Runner                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  GitHub-hosted Runner               Self-hosted Runner                   │
│  ─────────────────────              ─────────────────────               │
│  • 자동 프로비저닝                   • 직접 설치 및 관리                  │
│  • 매 실행마다 깨끗한 환경           • 영구적 환경 (캐시 유지)             │
│  • 제한된 하드웨어 옵션              • 커스텀 하드웨어                     │
│  • 사용량 기반 과금                  • 고정 비용 (인프라)                  │
│  • GitHub 네트워크                  • 프라이빗 네트워크 접근              │
│  • 제한된 실행 시간                  • 무제한 실행 시간                    │
│                                                                          │
│  언제 Self-hosted를 사용하나?                                            │
│  • 프라이빗 네트워크의 리소스 접근 필요                                   │
│  • 특수 하드웨어 (GPU, ARM) 필요                                        │
│  • 대용량 캐시/아티팩트 유지                                              │
│  • 보안 정책상 외부 실행 불가                                             │
│  • 비용 최적화 (대량 빌드)                                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Self-hosted Runner 설정

```bash
# Linux에서 Runner 설치 스크립트
#!/bin/bash

# 설치 디렉토리 생성
mkdir -p /opt/actions-runner && cd /opt/actions-runner

# Runner 패키지 다운로드
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz

# 체크섬 검증
echo "29fc8cf2dab4c195f06a00dc2ca11ad8c9c68ef6e75e57dffb42bdeafd1b"  \
  "1c52  actions-runner-linux-x64-2.311.0.tar.gz" | shasum -a 256 -c

# 압축 해제
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# 설정 (토큰은 GitHub에서 생성)
./config.sh --url https://github.com/your-org/your-repo \
  --token AXXXXXXXXXXXXXXXXXXXXXXXX \
  --name "production-runner-01" \
  --labels "production,linux,x64" \
  --runnergroup "production-runners" \
  --work "_work"

# 서비스로 설치 (systemd)
sudo ./svc.sh install
sudo ./svc.sh start
```

### 3.3 Kubernetes에서 Self-hosted Runner (ARC)

**Actions Runner Controller (ARC)** 를 사용하여 Kubernetes에서 Self-hosted Runner를 동적으로 관리할 수 있다.

```yaml
# runner-deployment.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: production-runner
  namespace: actions-runner-system
spec:
  replicas: 3
  template:
    spec:
      repository: your-org/your-repo
      labels:
        - production
        - k8s
      env:
        - name: DOCKER_HOST
          value: tcp://localhost:2376
      resources:
        limits:
          cpu: "2"
          memory: "4Gi"
        requests:
          cpu: "1"
          memory: "2Gi"
      volumes:
        - name: docker-sock
          emptyDir: {}
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock

---
# 자동 스케일링 설정
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: production-runner-autoscaler
  namespace: actions-runner-system
spec:
  scaleTargetRef:
    kind: RunnerDeployment
    name: production-runner
  scaleUpTriggers:
    - githubEvent:
        workflowJob: {}
      duration: "30m"
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: PercentageRunnersBusy
      scaleUpThreshold: "0.75"
      scaleDownThreshold: "0.25"
      scaleUpFactor: "2"
      scaleDownFactor: "0.5"
```

### 3.4 Self-hosted Runner 보안

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   Self-hosted Runner 보안 고려사항                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. Private Repository 전용                                              │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  ⚠️ Public Repository에서 Self-hosted Runner 사용 금지!          │ │
│     │  • 악의적인 Fork에서 시크릿 탈취 가능                            │ │
│     │  • 내부 네트워크 접근 위험                                       │ │
│     │  • 다른 Workflow의 GITHUB_TOKEN 탈취 가능                       │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  2. Workflow 제한                                                        │
│     • 특정 Workflow만 Runner Group 접근 허용                             │
│     • Reusable Workflow와 결합하여 일관된 보안 정책 적용                  │
│     • Repository Settings → Actions → Runner groups                     │
│                                                                          │
│  3. 네트워크 격리                                                        │
│     • Runner를 별도 네트워크 세그먼트에 배치                              │
│     • 필요한 아웃바운드만 허용                                           │
│     • 프로덕션 환경과 분리                                               │
│                                                                          │
│  4. 권한 최소화                                                          │
│     • 비특권 사용자로 Runner 실행                                        │
│     • 필요한 도구만 설치                                                 │
│     • 정기적인 보안 패치                                                 │
│                                                                          │
│  5. 임시 Runner 사용                                                     │
│     • 가능하면 Ephemeral Runner 사용                                     │
│     • 실행 후 환경 초기화                                                │
│     • Kubernetes ARC로 동적 스케일링                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.5 Workflow에서 Self-hosted Runner 사용

```yaml
name: Production Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    # GitHub-hosted runner에서 빌드
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.build.outputs.tag }}
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker Image
        id: build
        run: |
          docker build -t myapp:${{ github.sha }} .
          echo "tag=${{ github.sha }}" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    # Self-hosted runner에서 배포 (프라이빗 네트워크 접근 필요)
    runs-on: [self-hosted, production, linux]
    environment: production
    steps:
      - name: Pull Image from Private Registry
        run: |
          docker pull registry.internal:5000/myapp:${{ needs.build.outputs.image-tag }}

      - name: Deploy to Internal Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=registry.internal:5000/myapp:${{ needs.build.outputs.image-tag }}

  cleanup:
    needs: deploy
    if: always()
    runs-on: [self-hosted, production]
    steps:
      - name: Cleanup workspace
        run: |
          # Self-hosted Runner 작업 디렉토리 정리
          rm -rf ${{ github.workspace }}/*
```

---

## 4. Secrets 관리

### 4.1 Secrets 종류

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          GitHub Secrets 계층                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                      Organization Secrets                           ││
│  │  • 조직 전체에서 공유                                                ││
│  │  • 특정 저장소에만 접근 제한 가능                                     ││
│  │  • 중앙 집중식 관리                                                  ││
│  └─────────────────────────────────────────────────────────────────────┘│
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                      Repository Secrets                             ││
│  │  • 해당 저장소에서만 사용                                            ││
│  │  • 저장소 관리자가 설정                                              ││
│  │  • Organization Secrets 오버라이드 가능                              ││
│  └─────────────────────────────────────────────────────────────────────┘│
│         │                                                                │
│         ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                      Environment Secrets                            ││
│  │  • 특정 환경 (production, staging)에서만 사용                        ││
│  │  • 승인 워크플로우 연동 가능                                         ││
│  │  • 가장 높은 우선순위                                                ││
│  └─────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  우선순위: Environment > Repository > Organization                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Secrets 사용 패턴

```yaml
name: Deploy with Secrets

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    # Environment 사용 (Secrets + 승인 워크플로우)
    environment:
      name: production
      url: https://myapp.example.com

    steps:
      - uses: actions/checkout@v4

      # 환경변수로 전달
      - name: Deploy Application
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          # 환경변수로 접근
          aws s3 cp ./dist s3://my-bucket/

      # 명령줄 인수로 전달 (마스킹 자동 적용)
      - name: Configure Database
        run: |
          ./configure-db.sh --password="${{ secrets.DB_PASSWORD }}"

      # 파일로 저장
      - name: Setup Credentials File
        run: |
          echo "${{ secrets.KUBECONFIG }}" > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      # JSON 시크릿 파싱
      - name: Parse JSON Secret
        run: |
          echo '${{ secrets.CONFIG_JSON }}' > config.json
          API_KEY=$(jq -r '.api_key' config.json)
          echo "::add-mask::$API_KEY"
```

### 4.3 OIDC를 활용한 시크릿 없는 인증

```yaml
name: Deploy to AWS (OIDC)

on:
  push:
    branches: [main]

permissions:
  id-token: write   # OIDC 토큰 요청에 필요
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # AWS 자격 증명 (시크릿 없이 OIDC 사용)
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          role-session-name: GitHubActionsSession
          aws-region: ap-northeast-2

      # GCP 자격 증명 (OIDC)
      - name: Configure GCP Credentials
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/123456/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'github-actions@my-project.iam.gserviceaccount.com'

      - name: Deploy
        run: |
          aws s3 sync ./dist s3://my-bucket/
          gcloud app deploy
```

### 4.4 Secrets 보안 베스트 프랙티스

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Secrets 관리 베스트 프랙티스                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. OIDC 우선 사용                                                       │
│     • 가능하면 장기 자격 증명 대신 OIDC 사용                              │
│     • AWS, GCP, Azure 모두 지원                                          │
│     • 시크릿 노출 위험 제거                                              │
│                                                                          │
│  2. 최소 권한 원칙                                                       │
│     • 필요한 권한만 부여                                                 │
│     • Environment별로 다른 자격 증명 사용                                 │
│     • 읽기 전용 자격 증명 분리                                           │
│                                                                          │
│  3. 로그 보안                                                            │
│     • 시크릿은 자동 마스킹되지만 추가 주의                                │
│     • 동적 값은 명시적 마스킹: echo "::add-mask::$VALUE"                 │
│     • 디버그 모드에서도 노출 방지                                        │
│                                                                          │
│  4. 시크릿 로테이션                                                       │
│     • 정기적인 시크릿 교체 정책                                          │
│     • Vault, AWS Secrets Manager 연동 고려                               │
│     • 만료 알림 설정                                                     │
│                                                                          │
│  5. PR에서의 시크릿                                                       │
│     ┌─────────────────────────────────────────────────────────────────┐ │
│     │  • Fork에서의 PR은 시크릿 접근 불가                              │ │
│     │  • pull_request_target 사용 시 주의                              │ │
│     │  • 외부 기여자 PR은 수동 승인 필요                               │ │
│     └─────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  6. 시크릿 스캐닝                                                        │
│     • GitHub Secret Scanning 활성화                                      │
│     • Push Protection 사용                                               │
│     • pre-commit hook으로 로컬 검사                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Matrix Build

### 5.1 개념

**Matrix Build**는 여러 환경 조합에서 동일한 작업을 병렬로 실행하는 전략이다. 다양한 OS, 언어 버전, 설정 조합을 효율적으로 테스트할 수 있다.

```yaml
name: Matrix Build Example

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}

    strategy:
      # 실패해도 다른 조합 계속 실행
      fail-fast: false

      # 동시 실행 제한 (Self-hosted runner 리소스 관리)
      max-parallel: 4

      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node-version: [18, 20, 22]

        # 특정 조합 포함
        include:
          - os: ubuntu-latest
            node-version: 20
            experimental: true
            coverage: true
          - os: ubuntu-latest
            node-version: 22
            experimental: true

        # 특정 조합 제외
        exclude:
          - os: windows-latest
            node-version: 18

    name: Test on ${{ matrix.os }} / Node ${{ matrix.node-version }}

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Install Dependencies
        run: npm ci

      - name: Run Tests
        run: npm test
        continue-on-error: ${{ matrix.experimental == true }}

      - name: Run Coverage
        if: matrix.coverage == true
        run: npm run test:coverage
```

### 5.2 동적 Matrix 생성

```yaml
name: Dynamic Matrix

on:
  push:
    branches: [main]

jobs:
  # Matrix 값을 동적으로 생성
  prepare:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4

      - name: Generate Matrix
        id: set-matrix
        run: |
          # 변경된 서비스 감지
          CHANGED_SERVICES=$(git diff --name-only HEAD~1 | \
            grep '^services/' | \
            cut -d'/' -f2 | \
            sort -u | \
            jq -R -s -c 'split("\n")[:-1]')

          echo "matrix={\"service\":$CHANGED_SERVICES}" >> $GITHUB_OUTPUT

  build:
    needs: prepare
    if: needs.prepare.outputs.matrix != '{"service":[]}'
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.prepare.outputs.matrix) }}

    steps:
      - uses: actions/checkout@v4

      - name: Build Service
        run: |
          echo "Building service: ${{ matrix.service }}"
          cd services/${{ matrix.service }}
          docker build -t ${{ matrix.service }}:latest .
```

### 5.3 복합 Matrix 전략

```yaml
name: Comprehensive Test Matrix

on: [push]

jobs:
  test:
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-22.04, ubuntu-24.04]
        java-version: [17, 21]
        database: [postgresql, mysql]
        include:
          # 특정 조합에 추가 변수
          - database: postgresql
            db-image: postgres:15
            db-port: 5432
          - database: mysql
            db-image: mysql:8
            db-port: 3306

    services:
      database:
        image: ${{ matrix.db-image }}
        env:
          POSTGRES_PASSWORD: testpass
          MYSQL_ROOT_PASSWORD: testpass
        ports:
          - ${{ matrix.db-port }}:${{ matrix.db-port }}
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup Java ${{ matrix.java-version }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'

      - name: Run Integration Tests
        env:
          DB_TYPE: ${{ matrix.database }}
          DB_PORT: ${{ matrix.db-port }}
        run: |
          ./gradlew integrationTest \
            -Ddb.type=${{ matrix.database }} \
            -Ddb.port=${{ matrix.db-port }}
```

### 5.4 Matrix 최적화 팁

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Matrix Build 최적화                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 최대 조합 수 제한                                                    │
│     • Workflow당 최대 256개 Job                                         │
│     • 필요한 조합만 선별                                                 │
│     • exclude로 불필요한 조합 제거                                       │
│                                                                          │
│  2. 병렬 실행 관리                                                       │
│     strategy:                                                            │
│       max-parallel: 4  # Self-hosted runner 리소스 고려                  │
│                                                                          │
│  3. Fail-fast 전략                                                       │
│     • fail-fast: true (기본값) - 하나 실패 시 전체 취소                  │
│     • fail-fast: false - 모든 조합 완료 (디버깅에 유용)                  │
│                                                                          │
│  4. 캐싱 활용                                                            │
│     • 의존성 캐시 키에 matrix 값 포함                                    │
│     cache-key: deps-${{ matrix.os }}-${{ matrix.node }}                 │
│                                                                          │
│  5. 선택적 실행                                                          │
│     • 변경된 파일에 따라 필요한 조합만 실행                               │
│     • 동적 Matrix 생성 활용                                              │
│                                                                          │
│  6. 시간 기반 Matrix                                                     │
│     • 야간 빌드: 전체 Matrix                                             │
│     • PR 빌드: 핵심 조합만 (최신 버전, 주요 OS)                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 고급 패턴 및 최적화

### 6.1 캐싱 전략

```yaml
name: Optimized Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # npm 의존성 캐싱
      - name: Cache npm dependencies
        uses: actions/cache@v4
        id: npm-cache
        with:
          path: ~/.npm
          key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            npm-${{ runner.os }}-

      # node_modules 캐싱 (더 빠른 복원)
      - name: Cache node_modules
        uses: actions/cache@v4
        id: node-modules-cache
        with:
          path: node_modules
          key: modules-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}

      - name: Install Dependencies
        if: steps.node-modules-cache.outputs.cache-hit != 'true'
        run: npm ci

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

### 6.2 병렬 처리 및 Job 의존성

```yaml
name: Parallel Pipeline

on: [push]

jobs:
  # Stage 1: 병렬 빌드
  build-frontend:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - name: Build Frontend
        run: npm run build:frontend
      - id: version
        run: echo "version=$(cat package.json | jq -r .version)" >> $GITHUB_OUTPUT

  build-backend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Backend
        run: ./gradlew build

  # Stage 2: 병렬 테스트
  test-unit:
    needs: [build-frontend, build-backend]
    runs-on: ubuntu-latest
    steps:
      - name: Unit Tests
        run: npm test

  test-integration:
    needs: [build-frontend, build-backend]
    runs-on: ubuntu-latest
    steps:
      - name: Integration Tests
        run: npm run test:integration

  test-e2e:
    needs: [build-frontend, build-backend]
    runs-on: ubuntu-latest
    steps:
      - name: E2E Tests
        run: npm run test:e2e

  # Stage 3: 배포 (모든 테스트 통과 후)
  deploy:
    needs: [test-unit, test-integration, test-e2e]
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: |
          echo "Deploying version: ${{ needs.build-frontend.outputs.version }}"
```

### 6.3 조건부 Job 실행

```yaml
name: Conditional Workflow

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  # 변경된 파일 감지
  changes:
    runs-on: ubuntu-latest
    outputs:
      frontend: ${{ steps.filter.outputs.frontend }}
      backend: ${{ steps.filter.outputs.backend }}
      docs: ${{ steps.filter.outputs.docs }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            frontend:
              - 'frontend/**'
              - 'package.json'
            backend:
              - 'backend/**'
              - 'build.gradle'
            docs:
              - 'docs/**'
              - '**.md'

  # Frontend 변경 시만 실행
  build-frontend:
    needs: changes
    if: needs.changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Build Frontend
        run: echo "Building frontend..."

  # Backend 변경 시만 실행
  build-backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Build Backend
        run: echo "Building backend..."

  # 문서만 변경된 경우 빌드 스킵
  deploy:
    needs: [changes, build-frontend, build-backend]
    if: |
      always() &&
      (needs.build-frontend.result == 'success' || needs.build-frontend.result == 'skipped') &&
      (needs.build-backend.result == 'success' || needs.build-backend.result == 'skipped') &&
      needs.changes.outputs.docs != 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: echo "Deploying..."
```

### 6.4 디버깅 및 트러블슈팅

```yaml
name: Debug Workflow

on: [push]

jobs:
  debug:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 디버그 정보 출력
      - name: Dump GitHub context
        env:
          GITHUB_CONTEXT: ${{ toJSON(github) }}
        run: echo "$GITHUB_CONTEXT"

      - name: Dump job context
        env:
          JOB_CONTEXT: ${{ toJSON(job) }}
        run: echo "$JOB_CONTEXT"

      # SSH 디버깅 (수동 트리거 시)
      - name: Setup tmate session
        if: github.event_name == 'workflow_dispatch' && github.event.inputs.debug == 'true'
        uses: mxschmitt/action-tmate@v3
        timeout-minutes: 15

      # 아티팩트로 로그 저장
      - name: Run with verbose logging
        run: |
          set -x  # 명령어 출력
          npm run build 2>&1 | tee build.log
        continue-on-error: true

      - name: Upload logs
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: build-logs
          path: build.log
          retention-days: 5
```

---

## 참고 자료

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow Syntax Reference](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions)
- [Reusable Workflows Guide](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows)
- [Self-hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners)
- [Best Practices for Reusable Workflows](https://earthly.dev/blog/github-actions-reusable-workflows/)
- [Matrix Strategy Guide](https://www.blacksmith.sh/blog/matrix-builds-with-github-actions)

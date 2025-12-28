# GitHub & GitLab 실무 가이드

> PR/MR 워크플로우, CI/CD, 보안 기능

## 목차
1. [PR/MR 워크플로우](#1-prmr-워크플로우)
2. [Code Review Best Practices](#2-code-review-best-practices)
3. [GitHub Actions 기초](#3-github-actions-기초)
4. [GitLab CI/CD 기초](#4-gitlab-cicd-기초)
5. [보안 기능](#5-보안-기능)
6. [GitHub vs GitLab 비교](#6-github-vs-gitlab-비교)
---

## 1. PR/MR 워크플로우

### 1.1 용어 정리

| GitHub | GitLab | 설명 |
|--------|--------|------|
| Pull Request (PR) | Merge Request (MR) | 코드 변경을 머지 요청 |
| Repository | Project | 저장소 |
| Organization | Group | 조직/그룹 |
| GitHub Actions | GitLab CI/CD | CI/CD 도구 |
| Gist | Snippet | 코드 조각 공유 |

### 1.2 PR/MR 생성 워크플로우

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         PR/MR Lifecycle                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   1. Branch      2. Commit       3. Push         4. Create PR            │
│   ─────────►     ─────────►     ─────────►      ─────────►              │
│                                                                          │
│   ┌─────┐        ┌─────┐        ┌─────┐        ┌──────────┐             │
│   │ git │        │ git │        │ git │        │ GitHub   │             │
│   │ co  │   ──►  │ add │   ──►  │push │   ──►  │ PR 생성  │             │
│   │ -b  │        │ -A  │        │ -u  │        │          │             │
│   └─────┘        └─────┘        └─────┘        └──────────┘             │
│                                                      │                   │
│                                                      ▼                   │
│   8. Merge       7. Approve      6. Review      5. CI Checks            │
│   ◄─────────     ◄─────────     ◄─────────     ◄─────────               │
│                                                                          │
│   ┌─────┐        ┌─────┐        ┌─────┐        ┌──────────┐             │
│   │Merge│   ◄──  │LGTM │   ◄──  │코드 │   ◄──  │ 테스트   │             │
│   │ to  │        │ +1  │        │리뷰 │        │ 린트     │             │
│   │main │        │     │        │     │        │ 보안     │             │
│   └─────┘        └─────┘        └─────┘        └──────────┘             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Git CLI로 PR 워크플로우

```bash
# 1. 최신 main 동기화
$ git checkout main
$ git pull origin main

# 2. Feature 브랜치 생성
$ git checkout -b feature/user-authentication

# 3. 개발 및 커밋 (의미있는 단위로)
$ git add src/auth/
$ git commit -m "feat: Add JWT token generation"

$ git add src/middleware/
$ git commit -m "feat: Add auth middleware"

$ git add tests/
$ git commit -m "test: Add authentication tests"

# 4. 원격 푸시
$ git push -u origin feature/user-authentication

# 5. GitHub CLI로 PR 생성
$ gh pr create \
    --title "feat: User authentication system" \
    --body "## Summary
- JWT 기반 인증 시스템 구현
- 미들웨어 추가
- 테스트 커버리지 90%

## Test Plan
- [ ] Unit tests 통과
- [ ] Integration tests 통과
- [ ] 수동 테스트 완료"

# 6. PR 상태 확인
$ gh pr status
$ gh pr checks

# 7. 리뷰 요청
$ gh pr edit --add-reviewer teammate1,teammate2

# 8. 리뷰 확인 및 대응
$ gh pr view --comments

# 9. 머지 (승인 후)
$ gh pr merge --squash --delete-branch
```

### 1.4 PR/MR 템플릿

#### GitHub PR Template (`.github/PULL_REQUEST_TEMPLATE.md`)

```markdown
## Summary
<!-- 변경사항 간략 설명 (1-3 문장) -->

## Type of Change
- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that causes existing functionality to not work as expected)
- [ ] Refactoring (no functional changes)
- [ ] Documentation update

## Related Issues
<!-- Closes #123 -->

## Changes Made
<!-- 상세 변경 내용 -->
-
-
-

## Screenshots (if applicable)
<!-- UI 변경 시 스크린샷 첨부 -->

## Test Plan
- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist
- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review
- [ ] I have commented my code where necessary
- [ ] I have updated documentation
- [ ] My changes generate no new warnings
- [ ] New and existing tests pass locally
```

#### GitLab MR Template (`.gitlab/merge_request_templates/default.md`)

```markdown
## What does this MR do?
<!-- 변경사항 설명 -->

## Related issues
<!-- Closes #123 -->

## Author's checklist
- [ ] Follow the [style guidelines]
- [ ] Tests added
- [ ] Documentation updated

## Review checklist
- [ ] Code quality
- [ ] Test coverage
- [ ] Security considerations

/label ~"needs review"
/assign @reviewer
```

### 1.5 Branch Protection Rules

#### GitHub 설정

```yaml
# 저장소 설정 > Branches > Branch protection rules

main 브랜치 보호:
  # 필수 리뷰
  - Require a pull request before merging: true
    - Required approvals: 2
    - Dismiss stale reviews: true
    - Require review from Code Owners: true

  # 필수 CI 체크
  - Require status checks to pass: true
    - Required checks:
      - build
      - test
      - lint
    - Require branches to be up to date: true

  # 추가 보호
  - Require conversation resolution: true
  - Require signed commits: true
  - Require linear history: false  # squash merge 사용 시

  # 관리자 포함
  - Include administrators: true

  # 강제 푸시 금지
  - Allow force pushes: false
  - Allow deletions: false
```

#### GitLab 설정

```yaml
# Settings > Repository > Protected branches

main 브랜치 보호:
  - Allowed to merge: Maintainers
  - Allowed to push: No one
  - Require approval: 2 approvals

# Settings > Merge requests
Merge request approvals:
  - Prevent approval by author: true
  - Prevent editing approval rules: true
  - Remove all approvals when commits added: true
```

---

## 2. Code Review Best Practices

### 2.1 PR 작성자 가이드

#### PR 크기 제한

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Ideal PR Size Guidelines                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Lines Changed    │ Review Time │ Quality      │ Recommendation         │
│   ─────────────────┼─────────────┼──────────────┼──────────────────      │
│   < 200 lines      │ 15-30 min   │ Excellent    │ ✅ Ideal               │
│   200-400 lines    │ 30-60 min   │ Good         │ ✅ Acceptable          │
│   400-800 lines    │ 1-2 hours   │ Moderate     │ ⚠️  Consider splitting │
│   > 800 lines      │ 2+ hours    │ Poor         │ ❌ Must split          │
│                                                                          │
│   * Google: ~200 lines 권장                                              │
│   * Microsoft Azure: ~400 lines 권장                                     │
│   * 연구 결과: 200-400 lines에서 defect detection 최적                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 좋은 커밋 메시지

```bash
# Conventional Commits 형식
<type>(<scope>): <subject>

<body>

<footer>

# 예시
feat(auth): Add JWT refresh token support

- Implement refresh token generation
- Add token rotation on refresh
- Store refresh tokens in Redis with TTL

Closes #123
```

| Type | 설명 |
|------|------|
| `feat` | 새로운 기능 |
| `fix` | 버그 수정 |
| `docs` | 문서 변경 |
| `style` | 코드 스타일 (포매팅 등) |
| `refactor` | 리팩토링 |
| `test` | 테스트 추가/수정 |
| `chore` | 빌드, 설정 변경 |
| `perf` | 성능 개선 |

#### Self-Review 체크리스트

```markdown
# PR 제출 전 자가 점검

## 코드 품질
- [ ] 불필요한 코드, 주석 제거
- [ ] console.log, print 문 제거
- [ ] 하드코딩된 값 확인
- [ ] 에러 핸들링 적절함
- [ ] 엣지 케이스 처리

## 테스트
- [ ] 새 기능에 대한 테스트 작성
- [ ] 기존 테스트 통과
- [ ] 테스트 커버리지 유지/증가

## 보안
- [ ] 민감 정보 하드코딩 없음
- [ ] SQL Injection 취약점 없음
- [ ] XSS 취약점 없음
- [ ] 적절한 인증/인가 확인

## 성능
- [ ] N+1 쿼리 문제 없음
- [ ] 불필요한 DB 호출 없음
- [ ] 대용량 데이터 처리 고려
```

### 2.2 리뷰어 가이드

#### 리뷰 포인트

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Code Review Focus Areas                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   1. Correctness (정확성)                                                │
│      - 요구사항 충족 여부                                               │
│      - 로직 오류                                                         │
│      - 엣지 케이스 처리                                                  │
│                                                                          │
│   2. Security (보안)                                                     │
│      - 인증/인가 검증                                                    │
│      - 입력 검증                                                         │
│      - 민감 데이터 처리                                                  │
│                                                                          │
│   3. Performance (성능)                                                  │
│      - 알고리즘 효율성                                                   │
│      - 데이터베이스 쿼리 최적화                                         │
│      - 메모리 사용                                                       │
│                                                                          │
│   4. Maintainability (유지보수성)                                        │
│      - 코드 가독성                                                       │
│      - 적절한 추상화                                                     │
│      - SOLID 원칙 준수                                                   │
│                                                                          │
│   5. Testing (테스트)                                                    │
│      - 테스트 커버리지                                                   │
│      - 테스트 케이스 품질                                               │
│      - 엣지 케이스 테스트                                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 건설적인 피드백 예시

```markdown
# ❌ Bad Comments (비건설적)
"This is wrong."
"Why did you do this?"
"This code is messy."

# ✅ Good Comments (건설적)

## 질문 형태로
"Have you considered using a Map here instead of an Array?
It would improve the lookup time from O(n) to O(1)."

## 제안 형태로
"**Suggestion:** We could extract this logic into a separate function
for better reusability. Something like:
```java
private boolean isValidUser(User user) {
    return user != null && user.isActive();
}
```"

## 칭찬도 함께
"Nice use of the Strategy pattern here! 👍
One small suggestion: consider adding a null check on line 45."

## 중요도 명시
"**[Blocking]** This SQL query is vulnerable to injection.
Please use parameterized queries."

"**[Nit]** Minor style: prefer `const` over `let` for this variable."

"**[Optional]** Consider adding a comment explaining this algorithm."
```

#### 리뷰 레이블 시스템

```markdown
# 코멘트 접두사 컨벤션

[Blocking] - 반드시 수정 필요, 머지 불가
[Major]    - 중요 이슈, 수정 권장
[Minor]    - 작은 개선 제안
[Nit]      - 사소한 스타일/취향 문제
[Question] - 이해를 위한 질문
[FYI]      - 참고 정보 공유
[Praise]   - 좋은 코드에 대한 칭찬
```

### 2.3 리뷰 SLA (Service Level Agreement)

```yaml
# 팀 리뷰 SLA 예시

Response Time:
  First Response: < 4 hours (업무 시간 기준)
  Complete Review: < 24 hours

Review Expectations:
  PR Size < 200 lines: Same day review
  PR Size 200-400 lines: Next business day
  PR Size > 400 lines: Request split or schedule dedicated time

Escalation:
  No response after 24h: Ping in Slack
  No response after 48h: Escalate to team lead
  Urgent/Hotfix: Direct message + Slack mention
```

---

## 3. GitHub Actions 기초

### 3.1 기본 구조

```yaml
# .github/workflows/ci.yml

name: CI Pipeline

# 트리거 조건
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'  # 매일 자정
  workflow_dispatch:  # 수동 실행

# 환경 변수
env:
  NODE_VERSION: '20'
  JAVA_VERSION: '17'

# 작업 정의
jobs:
  # 첫 번째 Job
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-output
          path: dist/

  # 두 번째 Job (build에 의존)
  test:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: build-output
          path: dist/

      - name: Run tests
        run: npm test

  # 병렬 실행 Job
  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
```

### 3.2 실용적인 워크플로우 예시

#### Node.js/TypeScript 프로젝트

```yaml
# .github/workflows/nodejs.yml

name: Node.js CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npm run typecheck

      - name: Test
        run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

  build:
    needs: test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci
      - run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
          retention-days: 7
```

#### Spring Boot 프로젝트

```yaml
# .github/workflows/spring.yml

name: Spring Boot CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Build with Gradle
        run: ./gradlew build

      - name: Run tests
        run: ./gradlew test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test
          SPRING_REDIS_HOST: localhost
          SPRING_REDIS_PORT: 6379

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: build/reports/tests/

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
```

#### Docker 빌드 및 푸시

```yaml
# .github/workflows/docker.yml

name: Docker Build and Push

on:
  push:
    tags:
      - 'v*'

jobs:
  docker:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: myorg/myapp
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### 3.3 Secrets 관리

```yaml
# Secrets 사용 예시
steps:
  - name: Deploy
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    run: |
      aws s3 sync dist/ s3://my-bucket/

# Environment secrets (환경별 분리)
jobs:
  deploy-staging:
    environment: staging
    steps:
      - run: echo "Deploying to ${{ vars.DEPLOY_URL }}"

  deploy-production:
    environment: production
    needs: deploy-staging
    steps:
      - run: echo "Deploying to ${{ vars.DEPLOY_URL }}"
```

---

## 4. GitLab CI/CD 기초

### 4.1 기본 구조

```yaml
# .gitlab-ci.yml

# 전역 설정
default:
  image: node:20-alpine
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/

# 변수 정의
variables:
  DOCKER_DRIVER: overlay2
  GIT_DEPTH: 1

# 스테이지 정의 (순차 실행)
stages:
  - build
  - test
  - security
  - deploy

# 빌드 Job
build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour

# 테스트 Job
test:
  stage: test
  script:
    - npm ci
    - npm test -- --coverage
  coverage: '/Lines\s*:\s*(\d+\.?\d*)%/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

# 린트 Job (테스트와 병렬)
lint:
  stage: test
  script:
    - npm ci
    - npm run lint

# 보안 스캔
security_scan:
  stage: security
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker run --rm -v $(pwd):/app aquasec/trivy fs /app

# 스테이징 배포
deploy_staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - echo "Deploying to staging..."
  only:
    - develop

# 프로덕션 배포
deploy_production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - echo "Deploying to production..."
  when: manual  # 수동 승인 필요
  only:
    - main
```

### 4.2 고급 기능

#### Dynamic Child Pipelines

```yaml
# 동적으로 파이프라인 생성
generate-config:
  stage: build
  script:
    - generate-gitlab-ci > generated-config.yml
  artifacts:
    paths:
      - generated-config.yml

trigger-child:
  stage: deploy
  trigger:
    include:
      - artifact: generated-config.yml
        job: generate-config
```

#### DAG (Directed Acyclic Graph)

```yaml
# needs 키워드로 의존성 정의 (스테이지 무시)
stages:
  - build
  - test
  - deploy

build-frontend:
  stage: build
  script: npm run build:frontend

build-backend:
  stage: build
  script: npm run build:backend

test-frontend:
  stage: test
  needs: [build-frontend]  # build-backend 기다리지 않음
  script: npm run test:frontend

test-backend:
  stage: test
  needs: [build-backend]
  script: npm run test:backend

deploy:
  stage: deploy
  needs: [test-frontend, test-backend]
  script: ./deploy.sh
```

#### 환경별 배포

```yaml
.deploy_template: &deploy_template
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy_staging:
  <<: *deploy_template
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  variables:
    KUBE_CONTEXT: staging
  only:
    - develop

deploy_production:
  <<: *deploy_template
  stage: deploy
  environment:
    name: production
    url: https://example.com
  variables:
    KUBE_CONTEXT: production
  when: manual
  only:
    - main
```

---

## 5. 보안 기능

### 5.1 GitHub 보안 기능

#### Dependabot

```yaml
# .github/dependabot.yml

version: 2
updates:
  # npm 의존성
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Seoul"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
    commit-message:
      prefix: "chore(deps)"

  # Docker 이미지
  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"

  # Gradle (Java)
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "daily"
    ignore:
      - dependency-name: "org.springframework.boot:*"
        update-types: ["version-update:semver-major"]
```

#### Secret Scanning

```yaml
# GitHub Advanced Security 활성화 시 자동 감지되는 시크릿:
# - AWS Access Keys
# - GitHub Tokens
# - Google API Keys
# - Slack Tokens
# - Database Connection Strings
# - Private Keys
# 등 200+ 패턴

# Push Protection: 커밋 전 시크릿 차단
# - 저장소 설정에서 활성화
# - 개발자가 실수로 시크릿 푸시 방지
```

#### Code Scanning (CodeQL)

```yaml
# .github/workflows/codeql.yml

name: "CodeQL"

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # 매주 일요일

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      actions: read
      contents: read
      security-events: write

    strategy:
      fail-fast: false
      matrix:
        language: ['javascript', 'python', 'java']

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          # 커스텀 쿼리 사용
          queries: +security-extended,security-and-quality

      - name: Autobuild
        uses: github/codeql-action/autobuild@v3

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{matrix.language}}"
```

### 5.2 GitLab 보안 기능

```yaml
# .gitlab-ci.yml 보안 스캔 통합

include:
  # SAST (Static Application Security Testing)
  - template: Security/SAST.gitlab-ci.yml

  # Dependency Scanning
  - template: Security/Dependency-Scanning.gitlab-ci.yml

  # Container Scanning
  - template: Security/Container-Scanning.gitlab-ci.yml

  # Secret Detection
  - template: Security/Secret-Detection.gitlab-ci.yml

  # DAST (Dynamic Application Security Testing)
  - template: Security/DAST.gitlab-ci.yml

# SAST 커스터마이징
sast:
  variables:
    SAST_EXCLUDED_PATHS: "test/, vendor/"

# Container Scanning 커스터마이징
container_scanning:
  variables:
    CS_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# Secret Detection 커스터마이징
secret_detection:
  variables:
    SECRET_DETECTION_EXCLUDED_PATHS: "test/"
```

### 5.3 보안 Best Practices

```yaml
# 1. Secrets 관리
# ❌ Bad
env:
  API_KEY: "sk-1234567890abcdef"

# ✅ Good
env:
  API_KEY: ${{ secrets.API_KEY }}

# 2. 최소 권한 원칙
permissions:
  contents: read
  packages: write
  # 필요한 권한만 명시

# 3. 의존성 고정
# ❌ Bad
uses: actions/checkout@main

# ✅ Good (SHA 고정)
uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1

# 4. 환경 분리
jobs:
  deploy:
    environment:
      name: production
      url: https://example.com
    # production 환경에는 추가 보호 규칙 적용
```

---

## 6. GitHub vs GitLab 비교

### 6.1 기능 비교

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     GitHub vs GitLab Comparison                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Feature              │ GitHub           │ GitLab                       │
│   ─────────────────────┼──────────────────┼─────────────────────────     │
│   호스팅               │ Cloud 중심       │ Self-hosted 강점             │
│   CI/CD                │ Actions (2019~)  │ 내장 CI/CD (성숙)            │
│   Container Registry   │ Packages         │ 내장 Registry                │
│   Issue Tracking       │ Issues + Projects│ Issues + Boards (강력)       │
│   Code Review          │ PR + Review      │ MR + Review (상세)           │
│   Security             │ Advanced Security│ Ultimate (내장)              │
│   Wiki                 │ Wiki             │ Wiki + Docs                  │
│   Pages                │ GitHub Pages     │ GitLab Pages                 │
│   API                  │ REST + GraphQL   │ REST + GraphQL               │
│   가격 (팀)            │ $4/user/month    │ Free (Self-hosted)           │
│   오픈소스             │ Closed           │ Core는 Open                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 CI/CD 비교

| 항목 | GitHub Actions | GitLab CI/CD |
|------|----------------|--------------|
| **설정 파일** | `.github/workflows/*.yml` | `.gitlab-ci.yml` |
| **Runner** | GitHub-hosted / Self-hosted | Shared / Specific / Group |
| **Marketplace** | 풍부한 생태계 | 제한적 |
| **병렬 실행** | Matrix strategy | Parallel keyword |
| **캐싱** | actions/cache | 내장 cache |
| **Artifacts** | 90일 보관 | 설정 가능 |
| **무료 사용량** | 2000분/월 (Free) | 400분/월 (Free) |

### 6.3 선택 가이드

```
GitHub 선택 시:
├── 오픈소스 프로젝트
├── GitHub 생태계 활용 (Actions, Packages, Copilot)
├── 외부 협업자 많음
├── 간단한 CI/CD 요구
└── 미국/글로벌 팀

GitLab 선택 시:
├── Self-hosted 필수 (보안 규정)
├── 올인원 DevOps 플랫폼 원함
├── 복잡한 CI/CD 파이프라인
├── 내장 Container Registry 필요
├── 유럽 기반 / GDPR 준수
└── 비용 최적화 (Self-hosted)
```

---

## 참고 자료

- [GitHub Docs](https://docs.github.com)
- [GitLab Docs](https://docs.gitlab.com)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [GitHub Advanced Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)
- [Code Review Best Practices - Axolo](https://axolo.co/blog/p/code-review-best-practices-and-tools-2025)
- [GitLab CI vs GitHub Actions - Bytebase](https://www.bytebase.com/blog/gitlab-ci-vs-github-actions/)

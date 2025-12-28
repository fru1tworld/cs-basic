# Git 브랜치 전략

> 브랜치 전략 이해 및 실무 적용

## 목차
1. [Git Flow](#1-git-flow)
2. [GitHub Flow](#2-github-flow)
3. [GitLab Flow](#3-gitlab-flow)
4. [Trunk-Based Development](#4-trunk-based-development)
5. [전략 비교 및 선택 가이드](#5-전략-비교-및-선택-가이드)
---

## 브랜치 전략이란?

브랜치 전략은 팀이 **코드 변경을 어떻게 관리하고 통합할지**를 정의하는 규칙입니다. 적절한 전략 선택은 다음에 영향을 미칩니다:

- **배포 주기**: 얼마나 자주 배포할 수 있는가
- **협업 효율**: 팀원 간 충돌을 얼마나 최소화하는가
- **코드 품질**: 리뷰와 테스트를 어떻게 보장하는가
- **롤백 용이성**: 문제 발생 시 얼마나 빠르게 대응하는가

---

## 1. Git Flow

**Vincent Driessen**이 2010년에 제안한 전통적인 브랜치 모델입니다. 명확한 릴리스 주기가 있는 프로젝트에 적합합니다.

### 1.1 브랜치 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Git Flow Diagram                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   main        ●─────────────────●───────────────────●──────────────●    │
│               │                 ↑                   ↑              ↑     │
│               │                 │ (tag v1.0)        │ (tag v1.1)   │     │
│               ▼                 │                   │              │     │
│   hotfix          ●────────●────┘                   │          ●───┘     │
│                   │        │                        │          │         │
│               ────┼────────┼────────────────────────┼──────────┼───      │
│                   │        │                        │          │         │
│   release         │        │    ●───────────●───────┘          │         │
│                   │        │    │           │                  │         │
│               ────┼────────┼────┼───────────┼──────────────────┼───      │
│                   │        │    ↑           │                  │         │
│   develop   ●─────┼────────┼────●───────────●──────────────────┼────●    │
│             │     │        │    │                              ↓    │    │
│             │     ▼        ▼    │                              │    │    │
│   feature   │  ●──────●    │    ●──────────────●               │    │    │
│             │     (feat1)  │         (feat2)                   │    │    │
│             │              ▼                                   │    │    │
│             │         ●────────●                               │    │    │
│             │              (feat3)                             │    │    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 브랜치 역할

#### Main (Master) 브랜치
- **역할**: 프로덕션에 배포된 코드만 존재
- **규칙**: 직접 커밋 금지, release/hotfix 브랜치에서만 머지
- **태깅**: 모든 머지에 버전 태그 필수

```bash
# main에는 항상 태그가 붙음
$ git checkout main
$ git merge --no-ff release/1.0.0
$ git tag -a v1.0.0 -m "Release version 1.0.0"
$ git push origin main --tags
```

#### Develop 브랜치
- **역할**: 다음 릴리스를 위한 개발 통합 브랜치
- **규칙**: feature 브랜치들이 머지되는 곳
- **상태**: 항상 최신 개발 상태 유지

#### Feature 브랜치
- **시작점**: develop
- **머지 대상**: develop
- **네이밍**: `feature/기능명` 또는 `feature/JIRA-123-기능명`

```bash
# Feature 브랜치 시작
$ git checkout develop
$ git checkout -b feature/user-authentication

# 개발 완료 후
$ git checkout develop
$ git merge --no-ff feature/user-authentication
$ git branch -d feature/user-authentication
```

#### Release 브랜치
- **시작점**: develop
- **머지 대상**: main AND develop
- **네이밍**: `release/버전번호`
- **목적**: 릴리스 준비 (버그 수정, 문서화, 버전 번호 업데이트)

```bash
# Release 브랜치 시작
$ git checkout develop
$ git checkout -b release/1.0.0

# 릴리스 준비 작업 (버그 수정, 버전 업데이트 등)
$ echo "1.0.0" > VERSION
$ git commit -am "Bump version to 1.0.0"

# 릴리스 완료
$ git checkout main
$ git merge --no-ff release/1.0.0
$ git tag -a v1.0.0 -m "Release version 1.0.0"

$ git checkout develop
$ git merge --no-ff release/1.0.0

$ git branch -d release/1.0.0
```

#### Hotfix 브랜치
- **시작점**: main
- **머지 대상**: main AND develop
- **네이밍**: `hotfix/버전번호` 또는 `hotfix/이슈명`
- **목적**: 프로덕션 긴급 버그 수정

```bash
# Hotfix 브랜치 시작
$ git checkout main
$ git checkout -b hotfix/1.0.1

# 버그 수정
$ git commit -am "Fix critical security vulnerability"

# Hotfix 완료
$ git checkout main
$ git merge --no-ff hotfix/1.0.1
$ git tag -a v1.0.1 -m "Hotfix version 1.0.1"

$ git checkout develop
$ git merge --no-ff hotfix/1.0.1

$ git branch -d hotfix/1.0.1
```

### 1.3 Git Flow 도구

```bash
# git-flow 설치 (macOS)
$ brew install git-flow-avh

# 초기화
$ git flow init

# Feature 시작/종료
$ git flow feature start user-auth
$ git flow feature finish user-auth

# Release 시작/종료
$ git flow release start 1.0.0
$ git flow release finish 1.0.0

# Hotfix 시작/종료
$ git flow hotfix start 1.0.1
$ git flow hotfix finish 1.0.1
```

### 1.4 Git Flow 장단점

**장점:**
- 명확한 릴리스 버전 관리
- 역할이 분명한 브랜치 구조
- 대규모 팀과 복잡한 프로젝트에 적합
- 여러 버전 동시 유지 가능

**단점:**
- 복잡한 브랜치 구조
- CI/CD와 잘 맞지 않음 (지속적 배포에 부적합)
- Merge 충돌 가능성 높음
- 오버헤드가 큼

---

## 2. GitHub Flow

GitHub에서 제안한 **단순하고 가벼운** 브랜치 전략입니다. 지속적 배포(Continuous Deployment)에 최적화되어 있습니다.

### 2.1 브랜치 구조

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          GitHub Flow Diagram                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   main     ●───────●───────●───────●───────●───────●───────●────────    │
│            │       ↑       ↑       ↑       ↑       ↑       ↑            │
│            │       │       │       │       │       │       │            │
│            │       │       │       │       │       │       │            │
│   feature  └──●────┘       │       │       │       │       │            │
│               (PR)         │       │       │       │       │            │
│                            │       │       │       │       │            │
│   feature       └──●───────┘       │       │       │       │            │
│                    (PR)            │       │       │       │            │
│                                    │       │       │       │            │
│   bugfix                └──●───────┘       │       │       │            │
│                            (PR)            │       │       │            │
│                                            │       │       │            │
│   feature                       └──●───────┘       │       │            │
│                                    (PR)            │       │            │
│                                                    │       │            │
│   feature                               └──●───────┘       │            │
│                                            (PR)            │            │
│                                                            │            │
│   hotfix                                        └──●───────┘            │
│                                                    (PR)                 │
│                                                                          │
│   * 모든 변경은 PR을 통해 main에 직접 머지                              │
│   * main은 항상 배포 가능한 상태 유지                                   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 핵심 규칙

1. **main 브랜치는 항상 배포 가능한 상태**
2. **새 작업은 main에서 분기한 브랜치에서 진행**
3. **브랜치명은 설명적으로 작성**
4. **정기적으로 원격에 푸시**
5. **Pull Request로 코드 리뷰**
6. **리뷰 후 main에 머지하면 즉시 배포**

### 2.3 워크플로우

```bash
# 1. main에서 브랜치 생성
$ git checkout main
$ git pull origin main
$ git checkout -b add-user-profile-api

# 2. 작업 및 커밋
$ git add .
$ git commit -m "Add user profile API endpoint"

# 3. 원격에 푸시
$ git push -u origin add-user-profile-api

# 4. GitHub에서 Pull Request 생성
# - 코드 리뷰 진행
# - CI 테스트 통과 확인
# - 필요시 수정

# 5. 리뷰 승인 후 Merge
# (GitHub UI에서 또는 CLI로)
$ git checkout main
$ git merge --no-ff add-user-profile-api
$ git push origin main

# 6. 배포 (자동화)
# CI/CD 파이프라인이 main 머지를 감지하고 자동 배포
```

### 2.4 Branch Protection Rules

```yaml
# GitHub 저장소 설정 권장사항
main 브랜치 보호 규칙:
  - Require pull request reviews: 1명 이상
  - Require status checks to pass: CI 테스트
  - Require conversation resolution
  - Require linear history (선택적)
  - Include administrators
```

### 2.5 GitHub Flow 장단점

**장점:**
- 매우 단순하고 이해하기 쉬움
- CI/CD와 완벽하게 호환
- 빠른 피드백 루프
- 작은 팀에 최적

**단점:**
- 복잡한 릴리스 관리 불가
- 여러 버전 동시 유지 어려움
- 롤백이 새 배포로 이루어짐
- 스테이징 환경 관리 부재

---

## 3. GitLab Flow

GitHub Flow의 단순함과 Git Flow의 체계성을 결합한 전략입니다. **환경 브랜치**를 도입하여 다양한 배포 시나리오를 지원합니다.

### 3.1 Environment Branches (환경 브랜치)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GitLab Flow - Environment Branches                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   feature    ●──────●                                                    │
│              │      │                                                    │
│              │      ▼                                                    │
│   main       ●──────●───────●───────●───────●──────────────────────     │
│                             │               │                            │
│                             │ (Merge)       │ (Merge)                   │
│                             ▼               ▼                            │
│   staging    ───────────────●───────────────●──────────────────────     │
│                             │               │                            │
│                             │ (Test Pass)   │ (Test Pass)               │
│                             ▼               ▼                            │
│   production ───────────────●───────────────●──────────────────────     │
│                                                                          │
│   * main → staging → production 순서로 머지                             │
│   * 각 환경별 브랜치 유지                                               │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Release Branches (릴리스 브랜치)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    GitLab Flow - Release Branches                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   feature    ●──────●       ●──────●                                     │
│              │      │       │      │                                     │
│              │      ▼       │      ▼                                     │
│   main       ●──────●───────●──────●───────●───────●────────────────    │
│                     │              │               │                     │
│                     │ (Tag v2.0)   │               │ (Tag v3.0)         │
│                     ▼              │               ▼                     │
│   2-stable   ───────●──────────────┼───────────────────────────────     │
│                     │              │                                     │
│                     │ (cherry-pick │                                     │
│                     │  hotfix)     ▼                                     │
│   3-stable   ──────────────────────●───────●───────────────────────     │
│                                    │       │                             │
│                                    │       │ (cherry-pick hotfix)       │
│                                    │       │                             │
│                                                                          │
│   * 버전별 안정 브랜치 유지                                             │
│   * 핫픽스는 main에서 해당 버전으로 cherry-pick                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 워크플로우

```bash
# 1. Feature 개발 (GitHub Flow와 동일)
$ git checkout -b feature/new-api main
# ... 개발 ...
$ git push origin feature/new-api
# MR 생성 및 리뷰

# 2. main에 머지 후 staging으로 전파
$ git checkout staging
$ git merge main
$ git push origin staging
# staging 환경에 자동 배포

# 3. staging 테스트 통과 후 production으로
$ git checkout production
$ git merge staging
$ git push origin production
# production 환경에 자동 배포

# 4. 핫픽스 (upstream first 원칙)
$ git checkout -b hotfix/critical-bug main
# ... 수정 ...
$ git push origin hotfix/critical-bug
# MR로 main에 머지
# 이후 staging, production으로 순차 머지
# 또는 긴급 시 cherry-pick으로 직접 적용
```

### 3.4 Upstream First 원칙

**핫픽스는 항상 main에 먼저 적용**한 후, 하위 브랜치로 전파합니다.

```bash
# 잘못된 방법 (Anti-pattern)
$ git checkout production
$ git commit -m "Hotfix"  # ❌ production에 직접 커밋

# 올바른 방법
$ git checkout main
$ git commit -m "Hotfix"
$ git checkout staging && git merge main
$ git checkout production && git merge staging
# 또는 긴급 시
$ git checkout production
$ git cherry-pick <hotfix-commit>
```

### 3.5 GitLab Flow 장단점

**장점:**
- GitHub Flow보다 체계적인 배포 관리
- Git Flow보다 단순
- 다양한 환경(dev/staging/prod) 지원
- 여러 릴리스 버전 관리 가능

**단점:**
- GitHub Flow보다 복잡
- 환경 브랜치 관리 오버헤드
- cherry-pick 충돌 가능성

---

## 4. Trunk-Based Development

**단일 브랜치(trunk/main)**를 중심으로 개발하는 전략입니다. CI/CD와 가장 잘 맞는 현대적 접근법입니다.

### 4.1 기본 개념

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Trunk-Based Development                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Small Teams (직접 커밋):                                               │
│                                                                          │
│   trunk    ●───●───●───●───●───●───●───●───●───●───●───●────────────    │
│            ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑                │
│            A   B   A   C   B   A   C   B   A   B   C   A                 │
│            (개발자들이 직접 trunk에 커밋)                                │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Scaled Teams (Short-Lived Feature Branches):                           │
│                                                                          │
│   trunk    ●───────●───────●───────●───────●───────●───────●────────    │
│            │       ↑       ↑       ↑       ↑       ↑       ↑            │
│            │       │       │       │       │       │       │            │
│   feature  └──●────┘       │       │       │       │       │            │
│               (1-2일)      │       │       │       │       │            │
│                            │       │       │       │       │            │
│   feature       └──●───────┘       │       │       │       │            │
│                    (1일)           │       │       │       │            │
│                                    │       │       │       │            │
│   feature              └──●────────┘       │       │       │            │
│                           (2일)            │       │       │            │
│                                                                          │
│   * Feature 브랜치 수명: 최대 2일                                       │
│   * 하루에 여러 번 trunk에 통합                                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 핵심 원칙

1. **단일 통합 지점 (Trunk)**
2. **짧은 수명의 Feature 브랜치 (최대 1-2일)**
3. **하루에 한 번 이상 통합**
4. **Feature Flag로 미완성 기능 숨김**
5. **강력한 자동화 테스트 필수**

### 4.3 Feature Flags (기능 플래그)

```java
// 미완성 기능을 숨기는 Feature Flag 예시
public class UserService {

    @Autowired
    private FeatureFlagService featureFlags;

    public UserProfile getProfile(Long userId) {
        UserProfile profile = userRepository.findById(userId);

        // 새 프로필 UI는 Feature Flag로 제어
        if (featureFlags.isEnabled("NEW_PROFILE_UI")) {
            return enhanceWithNewUI(profile);
        }

        return profile;
    }
}
```

```yaml
# Feature Flag 설정 예시 (LaunchDarkly, Unleash 등)
features:
  NEW_PROFILE_UI:
    enabled: false
    rollout_percentage: 0
    target_users: ["internal_testers"]

  NEW_PAYMENT_FLOW:
    enabled: true
    rollout_percentage: 25  # 25%의 사용자에게만 노출
```

### 4.4 워크플로우

```bash
# 1. 최신 trunk 가져오기
$ git checkout main
$ git pull origin main

# 2. 짧은 수명의 feature 브랜치 생성 (선택적)
$ git checkout -b feat/small-change

# 3. 작은 단위로 커밋
$ git commit -m "Add user validation - part 1"
$ git commit -m "Add user validation - part 2"

# 4. 같은 날 trunk에 머지
$ git checkout main
$ git pull origin main
$ git merge feat/small-change
$ git push origin main

# 5. 또는 직접 trunk에 커밋 (소규모 팀)
$ git checkout main
$ git commit -m "Add input validation"
$ git push origin main
```

### 4.5 Release 전략

```
┌─────────────────────────────────────────────────────────────────────────┐
│              Trunk-Based Development - Release Methods                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   방법 1: Release from Trunk (권장)                                      │
│                                                                          │
│   trunk    ●───●───●───●───●───●───●───●───●───●───●───●────────────    │
│                    ↑           ↑           ↑                             │
│                    │           │           │                             │
│                (v1.0.0)    (v1.1.0)    (v1.2.0)                          │
│                (태그)       (태그)       (태그)                          │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   방법 2: Release Branches (필요시)                                      │
│                                                                          │
│   trunk    ●───●───●───●───●───●───●───●───●───●───●───●────────────    │
│                    │           │                                         │
│                    ▼           ▼                                         │
│   rel/1.0  ────────●───●       │                                        │
│                    │   │       │                                         │
│                (1.0.0)(1.0.1)  ▼                                        │
│                                                                          │
│   rel/1.1  ────────────────────●───●───●                                │
│                                │   │   │                                 │
│                            (1.1.0)(1.1.1)(1.1.2)                        │
│                                                                          │
│   * Release 브랜치는 trunk에서 분기                                     │
│   * 핫픽스는 trunk에서 cherry-pick                                      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.6 Trunk-Based Development 장단점

**장점:**
- CI/CD에 최적화
- 머지 충돌 최소화
- 빠른 피드백 루프
- 팀 협업 촉진
- 코드 품질 향상 (지속적 통합으로)

**단점:**
- 강력한 테스트 자동화 필수
- Feature Flag 관리 복잡성
- 시니어 개발자 수준 요구
- 오픈소스 프로젝트에 부적합
- 초기 설정 투자 필요

---

## 5. 전략 비교 및 선택 가이드

### 5.1 종합 비교표

| 구분 | Git Flow | GitHub Flow | GitLab Flow | Trunk-Based |
|------|----------|-------------|-------------|-------------|
| **복잡도** | 높음 | 낮음 | 중간 | 낮음 |
| **브랜치 수** | 많음 (5+) | 적음 (2) | 중간 (3-4) | 매우 적음 (1-2) |
| **배포 주기** | 주기적 | 지속적 | 유연함 | 지속적 |
| **팀 규모** | 대규모 | 소규모 | 중규모 | 소-중규모 |
| **릴리스 관리** | 체계적 | 단순 | 체계적 | 유연함 |
| **CI/CD 호환** | 낮음 | 높음 | 높음 | 매우 높음 |
| **학습 곡선** | 가파름 | 완만함 | 보통 | 보통 |
| **롤백** | 쉬움 | 새 배포 | 쉬움 | 새 배포 |

### 5.2 상황별 추천

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         브랜치 전략 선택 가이드                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   Q1. 배포 주기는?                                                       │
│   ├── 정기 릴리스 (월/분기) ──────────────────────┐                     │
│   │                                               ▼                      │
│   │                                           Git Flow                   │
│   │                                                                      │
│   └── 지속적 배포 (일/주) ──┐                                           │
│                             │                                            │
│   Q2. 팀 규모는?           ▼                                            │
│   ├── 소규모 (1-5명) ──────┐                                            │
│   │                        │                                             │
│   │   Q3. 경험 수준?       │                                            │
│   │   ├── 시니어 중심 ────►│──► Trunk-Based                             │
│   │   └── 주니어 포함 ────►│──► GitHub Flow                             │
│   │                        │                                             │
│   └── 중대규모 (5명+) ─────┤                                            │
│                            │                                             │
│       Q4. 다중 환경?       │                                            │
│       ├── 있음 ───────────►│──► GitLab Flow                             │
│       └── 없음 ───────────►│──► GitHub Flow                             │
│                                                                          │
│   Q5. 다중 버전 유지?                                                    │
│   ├── 필요 ────────────────────────────────────► Git Flow               │
│   │                                              또는 GitLab Flow        │
│   └── 불필요 ──────────────────────────────────► 위 결과 유지            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.3 실제 기업 사례

| 기업 | 전략 | 이유 |
|------|------|------|
| **Google** | Trunk-Based | 모노레포, 강력한 테스트 인프라 |
| **Facebook** | Trunk-Based | 빠른 배포 주기, 대규모 테스트 |
| **Netflix** | GitHub Flow 변형 | 마이크로서비스, 지속적 배포 |
| **Microsoft (Office)** | Git Flow | 정기 릴리스, 다중 버전 |
| **GitLab** | GitLab Flow | 자체 도구 사용, 월간 릴리스 |

### 5.4 하이브리드 접근

많은 팀이 순수한 하나의 전략보다 **하이브리드 접근**을 사용합니다:

```bash
# 예: GitHub Flow + Release Branches
# 평소에는 GitHub Flow
$ git checkout -b feature/new-api main
# PR로 main에 머지

# 릴리스 시점에 Release 브랜치 생성
$ git checkout -b release/v2.0 main

# 핫픽스는 main에 먼저, 필요시 cherry-pick
$ git checkout main
$ git commit -m "Fix security issue"
$ git checkout release/v2.0
$ git cherry-pick <commit-hash>
```

---

## 참고 자료

- [A successful Git branching model (Git Flow 원본)](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Flow Guide](https://docs.github.com/en/get-started/quickstart/github-flow)
- [GitLab Flow](https://about.gitlab.com/topics/version-control/what-is-gitlab-flow/)
- [Trunk Based Development](https://trunkbaseddevelopment.com/)
- [Atlassian - Comparing Workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)
- [Choosing the Right Git Branching Strategy](https://medium.com/@sreekanth.thummala/choosing-the-right-git-branching-strategy-a-comparative-analysis-f5e635443423)

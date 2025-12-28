# Git 심화

> Git 내부 구조 및 심화 명령어

## 목차
1. [Git 내부 구조](#1-git-내부-구조)
2. [Git 명령어 심화](#2-git-명령어-심화)
3. [Rebase vs Merge](#3-rebase-vs-merge)
4. [Cherry-pick과 Reset](#4-cherry-pick과-reset)
5. [.gitignore와 .gitattributes](#5-gitignore와-gitattributes)
---

## 1. Git 내부 구조

Git은 단순한 버전 관리 시스템이 아닌 **Content-Addressable Filesystem**입니다. 모든 데이터를 **SHA-1 해시**를 키로 사용하여 저장합니다.

### 1.1 Git의 4가지 객체 타입

```
┌─────────────────────────────────────────────────────────────┐
│                      Git Object Model                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│    ┌──────────┐                                              │
│    │  Commit  │ ─────────────────┐                          │
│    └────┬─────┘                  │                          │
│         │ parent                  │ tree                     │
│         ▼                         ▼                          │
│    ┌──────────┐            ┌──────────┐                     │
│    │  Commit  │            │   Tree   │                     │
│    └──────────┘            └────┬─────┘                     │
│                                  │                           │
│                    ┌─────────────┼─────────────┐            │
│                    │             │             │             │
│                    ▼             ▼             ▼             │
│              ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│              │   Blob   │  │   Blob   │  │   Tree   │       │
│              │ (file1)  │  │ (file2)  │  │  (dir)   │       │
│              └──────────┘  └──────────┘  └────┬─────┘       │
│                                               │              │
│                                               ▼              │
│                                         ┌──────────┐        │
│                                         │   Blob   │        │
│                                         │ (file3)  │        │
│                                         └──────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### Blob (Binary Large Object)
파일의 **내용**만을 저장합니다. 파일명이나 경로 정보는 포함하지 않습니다.

```bash
# Blob 객체 직접 확인
$ echo "Hello, Git" | git hash-object -w --stdin
b7aec520dec0a7516c18eb4c68b64ae1eb9b5a5e

# 저장된 Blob 확인
$ git cat-file -p b7aec520
Hello, Git

# Blob 타입 확인
$ git cat-file -t b7aec520
blob
```

**핵심 특징:**
- 동일한 내용은 항상 동일한 해시를 가짐 (Content-Addressable)
- 파일명이 달라도 내용이 같으면 하나의 Blob만 저장
- 저장 공간 효율성 극대화

#### Tree
**디렉토리 구조**를 표현합니다. 파일명, 권한, Blob/Tree 참조를 포함합니다.

```bash
# Tree 객체 확인
$ git cat-file -p HEAD^{tree}
100644 blob a906cb2a4a904a152e80877d4088654daad0c859    README.md
100644 blob 8f94139338f9404f26296befa88755fc2598c289    index.js
040000 tree 99f1a6d12cb4b6f19c8655fca46c3ecf317074e0    src

# 권한 설명
# 100644 - 일반 파일
# 100755 - 실행 파일
# 040000 - 디렉토리 (Tree)
# 120000 - 심볼릭 링크
```

#### Commit
**스냅샷 메타데이터**를 저장합니다. Tree, 부모 커밋, 작성자, 커미터, 메시지를 포함합니다.

```bash
# Commit 객체 구조 확인
$ git cat-file -p HEAD
tree 92b8b694ffb1675e5975148e1121810081dbdffe
parent 5ba3db8f9f29a5a08a72c0a6e91c9d0b1e9c0f12
author John Doe <john@example.com> 1609459200 +0900
committer John Doe <john@example.com> 1609459200 +0900

Initial commit with project setup
```

**중요:** Git은 **델타(차이점)**가 아닌 **전체 스냅샷**을 저장합니다. SVN, CVS와 달리 각 커밋은 완전한 프로젝트 상태를 참조합니다.

#### Tag
커밋을 가리키는 **영구적인 참조**입니다. Annotated Tag는 별도의 객체로 저장됩니다.

```bash
# Lightweight Tag (단순 참조)
$ git tag v1.0.0

# Annotated Tag (객체로 저장)
$ git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag 객체 확인
$ git cat-file -p v1.0.0
object 5ba3db8f9f29a5a08a72c0a6e91c9d0b1e9c0f12
type commit
tag v1.0.0
tagger John Doe <john@example.com> 1609459200 +0900

Release version 1.0.0
```

### 1.2 .git 디렉토리 구조

```
.git/
├── HEAD                 # 현재 체크아웃된 브랜치 참조
├── config               # 로컬 저장소 설정
├── description          # GitWeb용 설명 (거의 사용 안함)
├── hooks/               # 클라이언트/서버 사이드 훅 스크립트
│   ├── pre-commit       # 커밋 전 실행
│   ├── commit-msg       # 커밋 메시지 검증
│   ├── pre-push         # 푸시 전 실행
│   └── ...
├── index                # 스테이징 영역 (바이너리)
├── info/
│   └── exclude          # 로컬 전용 ignore 패턴
├── logs/                # reflog 기록
│   ├── HEAD
│   └── refs/
├── objects/             # 모든 Git 객체 저장소
│   ├── info/
│   ├── pack/            # 압축된 객체 (Packfile)
│   └── [0-9a-f]{2}/     # Loose 객체 (해시 첫 2자리)
└── refs/                # 브랜치, 태그 참조
    ├── heads/           # 로컬 브랜치
    ├── remotes/         # 원격 추적 브랜치
    └── tags/            # 태그
```

### 1.3 객체 저장 메커니즘

```bash
# 객체 저장 과정 시뮬레이션
$ echo "Hello Git" > test.txt
$ git hash-object -w test.txt
9f4d96d5b00d98959ea9960f069585ce42b1349a

# 실제 저장 위치 확인
$ find .git/objects -type f
.git/objects/9f/4d96d5b00d98959ea9960f069585ce42b1349a

# zlib 압축되어 저장됨
$ python3 -c "import zlib; print(zlib.decompress(open('.git/objects/9f/4d96d5b00d98959ea9960f069585ce42b1349a', 'rb').read()))"
b'blob 10\x00Hello Git\n'
```

### 1.4 Packfile과 저장 최적화

시간이 지나면 Git은 Loose 객체들을 **Packfile**로 압축합니다.

```bash
# 수동으로 Pack 생성
$ git gc

# Pack 파일 확인
$ ls .git/objects/pack/
pack-xxxx.idx   # 인덱스 파일
pack-xxxx.pack  # 실제 객체 데이터

# Pack 내용 확인
$ git verify-pack -v .git/objects/pack/pack-xxxx.idx
```

**Packfile 특징:**
- 유사한 객체들을 **델타 압축**하여 저장
- 큰 파일의 작은 변경은 델타만 저장
- 네트워크 전송 및 저장 공간 최적화

---

## 2. Git 명령어 심화

### 2.1 Reset - 3가지 모드의 이해

```
┌─────────────────────────────────────────────────────────────┐
│                    Git Reset Modes                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Working Directory ◄──┐                                    │
│         │              │ --hard                              │
│         ▼              │                                     │
│   Staging Area    ◄────┼───┐                                │
│         │              │   │ --mixed (default)              │
│         ▼              │   │                                 │
│   HEAD (Repository) ◄──┴───┴───┐                            │
│                                │ --soft                      │
│                                │                             │
└─────────────────────────────────────────────────────────────┘
```

#### --soft
HEAD만 이동, Staging Area와 Working Directory는 유지

```bash
# 마지막 커밋 취소, 변경사항은 staged 상태로 유지
$ git reset --soft HEAD~1

# 여러 커밋을 하나로 합치기 (squash 대안)
$ git reset --soft HEAD~3
$ git commit -m "Combined commit message"
```

**사용 케이스:**
- 커밋 메시지만 수정하고 싶을 때
- 여러 커밋을 하나로 합치고 싶을 때
- 커밋을 취소하되 변경사항은 바로 다시 커밋할 수 있게 유지

#### --mixed (기본값)
HEAD와 Staging Area 초기화, Working Directory는 유지

```bash
# 스테이징 취소 (파일은 수정된 상태로 유지)
$ git reset HEAD~1
$ git reset --mixed HEAD~1  # 동일

# 특정 파일만 언스테이징
$ git reset HEAD -- file.txt
```

**사용 케이스:**
- 커밋은 취소하되 파일 수정은 유지하고 싶을 때
- 실수로 스테이징한 파일 취소
- 커밋 전에 변경사항 재검토

#### --hard
HEAD, Staging Area, Working Directory 모두 초기화

```bash
# 위험! 모든 변경사항 삭제
$ git reset --hard HEAD~1

# 특정 커밋으로 완전히 되돌리기
$ git reset --hard abc1234

# 원격 브랜치 상태로 강제 리셋
$ git reset --hard origin/main
```

**주의사항:**
- 커밋되지 않은 변경사항은 **영구 삭제**
- 복구하려면 `git reflog` 사용 필요
- 공유된 브랜치에서는 사용 자제

### 2.2 Revert - 안전한 되돌리기

```bash
# 특정 커밋의 변경사항만 되돌리는 새 커밋 생성
$ git revert <commit-hash>

# 여러 커밋 되돌리기
$ git revert HEAD~3..HEAD

# 커밋 없이 되돌리기 (나중에 한번에 커밋)
$ git revert --no-commit HEAD~3..HEAD
$ git commit -m "Revert multiple commits"

# Merge 커밋 되돌리기 (부모 지정 필요)
$ git revert -m 1 <merge-commit-hash>
```

**Reset vs Revert 비교:**

| 구분 | Reset | Revert |
|------|-------|--------|
| 히스토리 | 삭제/변경 | 보존 (새 커밋 추가) |
| 공유 브랜치 | 위험 | 안전 |
| 원격 푸시 | force push 필요 | 일반 push 가능 |
| 사용 케이스 | 로컬 정리 | 배포된 코드 롤백 |

### 2.3 Stash - 임시 저장

```bash
# 기본 stash
$ git stash
$ git stash push -m "작업 중인 기능 A"

# 특정 파일만 stash
$ git stash push -m "message" -- path/to/file

# Untracked 파일 포함
$ git stash -u
$ git stash --include-untracked

# 모든 파일 포함 (ignored 파일도)
$ git stash -a
$ git stash --all

# Stash 목록 확인
$ git stash list
stash@{0}: On main: 작업 중인 기능 A
stash@{1}: WIP on main: abc1234 Previous commit message

# Stash 적용
$ git stash pop           # 적용 후 삭제
$ git stash apply         # 적용만 (삭제 안함)
$ git stash apply stash@{1}  # 특정 stash 적용

# Stash 내용 확인
$ git stash show          # 요약
$ git stash show -p       # 상세 diff
$ git stash show stash@{1} -p

# Stash 삭제
$ git stash drop stash@{0}
$ git stash clear         # 모든 stash 삭제

# Stash를 브랜치로 변환
$ git stash branch new-feature stash@{0}
```

#### Interactive Stash (부분 Stash)

```bash
# 변경사항을 선택적으로 stash
$ git stash push -p

# 각 hunk에 대해:
# y - 이 hunk stash
# n - 이 hunk 건너뛰기
# s - 더 작은 hunk로 분할
# q - 종료
```

### 2.4 Reflog - 시간 여행

```bash
# HEAD 변경 이력 조회
$ git reflog
abc1234 HEAD@{0}: commit: Add new feature
def5678 HEAD@{1}: checkout: moving from feature to main
ghi9012 HEAD@{2}: reset: moving to HEAD~1

# 특정 브랜치의 reflog
$ git reflog show feature-branch

# 삭제된 커밋 복구
$ git reset --hard HEAD@{5}

# 삭제된 브랜치 복구
$ git checkout -b recovered-branch HEAD@{10}

# 특정 시간 기준 복구
$ git checkout HEAD@{2.hours.ago}
$ git checkout HEAD@{"2024-01-15 14:30:00"}
```

---

## 3. Rebase vs Merge

### 3.1 동작 원리

```
        Merge                              Rebase

    A---B---C (main)                   A---B---C (main)
         \                                      \
          D---E (feature)                        D'---E' (feature)
               \
                M (merge commit)       After rebase, feature is linear
                                       with main as its base
```

#### Git Merge

```bash
# main에 feature 브랜치 병합
$ git checkout main
$ git merge feature

# 결과: 새로운 Merge 커밋(M) 생성
# 히스토리: A---B---C---M
#                \     /
#                 D---E
```

#### Git Rebase

```bash
# feature 브랜치를 main 기준으로 재배치
$ git checkout feature
$ git rebase main

# 결과: feature의 커밋들이 main 뒤로 재배치
# 히스토리: A---B---C---D'---E'
```

### 3.2 장단점 비교

| 구분 | Merge | Rebase |
|------|-------|--------|
| **히스토리** | 분기점 보존 (비선형) | 선형 히스토리 |
| **안전성** | 기존 커밋 변경 없음 | 커밋 해시 변경됨 |
| **협업** | 안전함 | 공유 브랜치에서 위험 |
| **충돌 해결** | 한 번에 해결 | 커밋마다 해결 필요할 수 있음 |
| **bisect** | 복잡해질 수 있음 | 용이함 |
| **되돌리기** | merge 커밋 revert로 쉬움 | 복잡할 수 있음 |

### 3.3 Best Practices

```bash
# Golden Rule: 공유된 브랜치는 절대 rebase 하지 않는다

# 추천 워크플로우
# 1. 기능 개발 시 - feature 브랜치에서 작업
$ git checkout -b feature/new-feature

# 2. 주기적으로 main 변경사항 반영 (rebase)
$ git fetch origin
$ git rebase origin/main

# 3. 충돌 해결
$ git add .
$ git rebase --continue

# 4. 완료 후 main에 머지 (merge commit 생성)
$ git checkout main
$ git merge --no-ff feature/new-feature
```

#### Interactive Rebase

```bash
# 최근 3개 커밋 수정
$ git rebase -i HEAD~3

# 에디터에서:
pick abc1234 First commit
squash def5678 Second commit (이전 커밋에 합침)
reword ghi9012 Third commit (메시지 수정)

# 명령어 옵션
# pick (p)   - 커밋 사용
# reword (r) - 커밋 사용, 메시지 수정
# edit (e)   - 커밋에서 멈춤 (수정 가능)
# squash (s) - 이전 커밋에 합침
# fixup (f)  - squash와 같지만 메시지 버림
# drop (d)   - 커밋 삭제
```

#### Autostash 활용

```bash
# rebase 시 자동으로 stash/unstash
$ git rebase --autostash main

# 전역 설정
$ git config --global rebase.autoStash true
```

---

## 4. Cherry-pick과 Reset

### 4.1 Cherry-pick

특정 커밋의 변경사항만 현재 브랜치에 적용합니다.

```bash
# 단일 커밋 cherry-pick
$ git cherry-pick <commit-hash>

# 여러 커밋 cherry-pick
$ git cherry-pick <commit1> <commit2> <commit3>

# 범위 cherry-pick (commit1 제외, commit2 포함)
$ git cherry-pick <commit1>..<commit2>

# 범위 cherry-pick (commit1 포함)
$ git cherry-pick <commit1>^..<commit2>
```

#### Cherry-pick 옵션

```bash
# 커밋하지 않고 변경사항만 적용
$ git cherry-pick --no-commit <commit-hash>
$ git cherry-pick -n <commit-hash>

# 원본 커밋 정보 추가
$ git cherry-pick -x <commit-hash>
# 커밋 메시지에 "(cherry picked from commit xxx...)" 추가

# 충돌 시
$ git cherry-pick <commit-hash>
# ... 충돌 해결 ...
$ git add .
$ git cherry-pick --continue

# cherry-pick 취소
$ git cherry-pick --abort
```

#### 사용 케이스

```bash
# 1. 핫픽스를 여러 브랜치에 적용
$ git checkout release/v1.0
$ git cherry-pick <hotfix-commit>
$ git checkout release/v1.1
$ git cherry-pick <hotfix-commit>

# 2. 특정 기능만 다른 브랜치로 이동
$ git checkout main
$ git cherry-pick <feature-commit-1> <feature-commit-2>

# 3. 잘못된 브랜치에서 작업한 커밋 이동
$ git checkout correct-branch
$ git cherry-pick <wrong-branch-commit>
$ git checkout wrong-branch
$ git reset --hard HEAD~1
```

### 4.2 Reset 심화

#### 특정 파일만 Reset

```bash
# 특정 파일을 특정 커밋 상태로 복원
$ git checkout <commit-hash> -- path/to/file

# 또는 Git 2.23+
$ git restore --source=<commit-hash> path/to/file

# Staging Area에서만 제거 (파일 내용 유지)
$ git reset HEAD -- path/to/file
$ git restore --staged path/to/file  # Git 2.23+
```

#### ORIG_HEAD 활용

```bash
# reset/merge/rebase 전 HEAD를 ORIG_HEAD에 저장
$ git reset --hard HEAD~3

# 실수로 reset한 경우 복구
$ git reset --hard ORIG_HEAD
```

### 4.3 Cherry-pick과 Reset 조합

```bash
# 시나리오: 중간 커밋만 제거하고 싶을 때
# A - B - C - D (HEAD)
# B 커밋만 제거하고 싶음

# 방법 1: Interactive Rebase
$ git rebase -i A
# B 커밋을 drop

# 방법 2: Reset + Cherry-pick
$ git reset --hard A
$ git cherry-pick C D
# 결과: A - C' - D'
```

---

## 5. .gitignore와 .gitattributes

### 5.1 .gitignore

#### 패턴 문법

```gitignore
# 주석
# 특정 파일
secret.txt

# 특정 확장자 (전체 디렉토리에서)
*.log
*.tmp

# 특정 디렉토리
node_modules/
build/
dist/

# 특정 경로
/config/local.yml        # 루트의 config/local.yml만
config/local.yml         # 어디든 config/local.yml

# 와일드카드
*.py[cod]                # .pyc, .pyo, .pyd
**/logs                  # 모든 하위의 logs 디렉토리
**/logs/**               # logs 디렉토리 내 모든 것

# 부정 패턴 (예외)
*.log
!important.log           # important.log는 추적

# 디렉토리만 매칭
temp/                    # temp 디렉토리 (파일 아님)

# 깊이 제한
/*.txt                   # 루트 레벨의 .txt만
/**/*.txt                # 모든 레벨의 .txt
```

#### 프로젝트별 권장 설정

```gitignore
# === Node.js ===
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.pnpm-store/

# === Python ===
__pycache__/
*.py[cod]
*$py.class
.Python
venv/
.venv/
*.egg-info/
.pytest_cache/

# === Java/Kotlin ===
*.class
*.jar
*.war
build/
.gradle/
target/

# === IDE ===
.idea/
*.iml
.vscode/
*.swp
*.swo
*~

# === OS ===
.DS_Store
Thumbs.db
desktop.ini

# === 환경 설정 ===
.env
.env.local
*.local
config/local.yml

# === 빌드 결과물 ===
dist/
build/
out/
*.min.js
*.min.css
```

#### Global Gitignore

```bash
# 전역 gitignore 설정
$ git config --global core.excludesfile ~/.gitignore_global

# ~/.gitignore_global
.DS_Store
Thumbs.db
*.swp
.idea/
.vscode/
```

#### 이미 추적 중인 파일 제거

```bash
# 파일은 유지하고 추적만 중지
$ git rm --cached <file>
$ git rm -r --cached <directory>

# .gitignore 추가 후 전체 캐시 재설정
$ git rm -r --cached .
$ git add .
$ git commit -m "Apply .gitignore"
```

### 5.2 .gitattributes

파일별 속성을 정의합니다. Line ending, diff, merge 전략 등을 설정합니다.

#### Line Ending 설정

```gitattributes
# 기본 설정 (권장)
* text=auto

# 특정 파일 타입 강제
*.sh text eol=lf
*.bat text eol=crlf
*.ps1 text eol=crlf

# 바이너리 파일
*.png binary
*.jpg binary
*.gif binary
*.ico binary
*.pdf binary

# 특정 확장자를 텍스트로 강제
*.sql text
*.md text
```

#### Diff 및 Merge 설정

```gitattributes
# 바이너리로 diff 비활성화
*.min.js -diff
*.min.css -diff

# 커스텀 diff 드라이버
*.png diff=exif

# Lock 파일 - merge 시 ours 전략
package-lock.json merge=ours
yarn.lock merge=ours
Gemfile.lock merge=ours

# 자동 생성 파일 - merge 충돌 시 수동 재생성
*.generated.* -merge
```

#### 언어별 Linguist 설정 (GitHub용)

```gitattributes
# 언어 통계에서 제외
*.min.js linguist-vendored
docs/** linguist-documentation

# 언어 강제 지정
*.blade.php linguist-language=PHP

# 생성된 파일로 마킹
generated/** linguist-generated=true
```

#### Export 제어

```gitattributes
# git archive에서 제외
.gitignore export-ignore
.gitattributes export-ignore
.github/ export-ignore
tests/ export-ignore
docs/ export-ignore
```

---

## 참고 자료

- [Pro Git Book (공식)](https://git-scm.com/book/en/v2)
- [Git 공식 문서](https://git-scm.com/docs)
- [Git Internals - Git Objects](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [Atlassian Git Tutorials](https://www.atlassian.com/git/tutorials)
- [A Visual Guide to Git Internals - freeCodeCamp](https://www.freecodecamp.org/news/git-internals-objects-branches-create-repo/)

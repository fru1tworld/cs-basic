# 터미널과 쉘 명령어 (Terminal and Shell Commands)

> 터미널 환경과 쉘 명령어

---

## 목차

1. [쉘 종류와 특징](#1-쉘-종류와-특징)
2. [필수 명령어](#2-필수-명령어)
3. [쉘 스크립트 기초](#3-쉘-스크립트-기초)
4. [시스템 모니터링 명령어](#4-시스템-모니터링-명령어)
6. [참고 자료](#6-참고-자료)

---

## 1. 쉘 종류와 특징

### 1.1 쉘의 역할

```
사용자 ◄──► 쉘 (Shell) ◄──► 커널 (Kernel) ◄──► 하드웨어

쉘의 역할:
├── 명령어 해석 (Command Interpreter)
├── 스크립트 실행 (Script Execution)
├── 환경 변수 관리 (Environment Variables)
├── 파이프라인 및 리다이렉션 (Pipeline & Redirection)
├── 작업 제어 (Job Control)
└── 명령어 히스토리 (Command History)
```

### 1.2 주요 쉘 비교

| 쉘 | 개발 | 특징 | 기본 설정 파일 |
|-----|------|------|---------------|
| **sh (Bourne Shell)** | 1977, Bell Labs | POSIX 표준, 기본 기능 | ~/.profile |
| **bash** | 1989, GNU | Linux 기본, 풍부한 기능 | ~/.bashrc, ~/.bash_profile |
| **zsh** | 1990 | 강력한 자동완성, 테마 | ~/.zshrc |
| **fish** | 2005 | 사용자 친화적, 컬러 문법 | ~/.config/fish/config.fish |
| **dash** | 2002 | 경량, 빠른 스크립트 실행 | /etc/profile |

### 1.3 Bash (Bourne Again Shell)

가장 널리 사용되는 쉘로, Linux 배포판의 기본 쉘입니다.

```bash
# 현재 쉘 확인
echo $SHELL
echo $0

# Bash 버전 확인
bash --version

# 사용 가능한 쉘 목록
cat /etc/shells
```

#### Bash 주요 기능

```bash
# 1. 명령어 히스토리
history              # 히스토리 출력
!100                 # 100번 명령어 재실행
!!                   # 마지막 명령어 재실행
!$                   # 마지막 인자
!*                   # 모든 인자
Ctrl+R               # 역방향 검색

# 2. 브레이스 확장 (Brace Expansion)
echo {a,b,c}         # a b c
echo file{1..5}.txt  # file1.txt file2.txt file3.txt file4.txt file5.txt
mkdir -p project/{src,bin,lib}

# 3. 변수 확장
name="World"
echo "Hello, ${name}!"
echo "Length: ${#name}"
echo "Default: ${undefined:-default_value}"

# 4. 명령어 치환
date=$(date +%Y%m%d)
files=$(ls -la | wc -l)

# 5. 프로세스 치환
diff <(ls dir1) <(ls dir2)

# 6. 배열
arr=(one two three)
echo ${arr[0]}       # one
echo ${arr[@]}       # 모든 요소
echo ${#arr[@]}      # 요소 개수
```

#### Bash 설정 파일

```bash
# ~/.bashrc - 대화형 쉘 설정
export EDITOR=vim
export HISTSIZE=10000
export HISTCONTROL=ignoredups:erasedups

# 별칭 (Alias)
alias ll='ls -la'
alias gs='git status'
alias ..='cd ..'
alias ...='cd ../..'

# 함수
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# 프롬프트 설정
PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '

# PATH 설정
export PATH="$HOME/bin:$PATH"
```

### 1.4 Zsh

macOS Catalina 이후 기본 쉘. bash와 호환되며 더 강력한 기능을 제공합니다.

```zsh
# zsh 설치 (Ubuntu)
sudo apt install zsh

# 기본 쉘 변경
chsh -s $(which zsh)

# Oh My Zsh 설치 (플러그인/테마 관리)
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

#### Zsh 주요 기능

```zsh
# 1. 고급 자동완성
cd /u/l/b<Tab>  # → /usr/local/bin

# 2. 경로 확장
cd ~user        # 사용자 홈 디렉토리
cd -            # 이전 디렉토리

# 3. 글로브 패턴 확장
ls **/*.py      # 재귀적 파일 검색
ls *.txt~*.log  # 제외 패턴

# 4. 스펠링 교정
cd /usrr/local  # → Did you mean /usr/local?

# 5. 히스토리 공유
# 여러 터미널 세션 간 히스토리 공유

# 6. 확장된 프롬프트
# Git 브랜치, 실행 시간, 종료 코드 등 표시
```

#### ~/.zshrc 설정

```zsh
# Oh My Zsh 설정
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="agnoster"  # 테마 설정

# 플러그인
plugins=(
    git
    docker
    kubectl
    zsh-autosuggestions
    zsh-syntax-highlighting
)

source $ZSH/oh-my-zsh.sh

# 사용자 설정
export EDITOR='vim'
export LANG='en_US.UTF-8'

# 별칭
alias k='kubectl'
alias d='docker'
alias dc='docker-compose'

# 함수
function gcom() {
    git add . && git commit -m "$1"
}
```

### 1.5 Fish (Friendly Interactive Shell)

사용자 친화적인 설계로, 설정 없이도 강력한 기능을 제공합니다.

```fish
# fish 설치 (Ubuntu)
sudo apt install fish

# fish 시작
fish

# 설정 웹 인터페이스
fish_config
```

#### Fish 주요 특징

```fish
# 1. 자동 제안 (Autosuggestion)
# 타이핑 중 회색 글씨로 명령어 제안
# → 방향키로 수락

# 2. 문법 강조
# 존재하는 명령어: 녹색
# 존재하지 않는 명령어: 빨간색

# 3. Tab 완성
# 파일, 옵션, 변수 모두 자동완성

# 4. 변수 설정 (POSIX와 다름)
set MY_VAR "value"          # fish 방식
set -x PATH $HOME/bin $PATH # 환경 변수 내보내기

# 5. 함수 정의
function mkcd
    mkdir -p $argv[1] && cd $argv[1]
end

# 6. 별칭 (abbr 권장)
abbr -a gs 'git status'
abbr -a ll 'ls -la'
```

### 1.6 쉘 간 차이점

```bash
# 변수 할당
bash/zsh: VAR=value
fish:     set VAR value

# 환경 변수 내보내기
bash/zsh: export VAR=value
fish:     set -x VAR value

# 조건문
bash/zsh: if [ condition ]; then ... fi
fish:     if test condition; ... end

# 반복문
bash/zsh: for i in 1 2 3; do echo $i; done
fish:     for i in 1 2 3; echo $i; end

# 함수
bash/zsh: function_name() { ... }
fish:     function function_name; ... end

# 명령어 치환
bash/zsh: $(command)
fish:     (command)
```

---

## 2. 필수 명령어

### 2.1 ps (Process Status)

```bash
# 현재 터미널의 프로세스
ps

# 모든 프로세스 (BSD 스타일)
ps aux

# 모든 프로세스 (POSIX 스타일)
ps -ef

# 특정 프로세스 검색
ps aux | grep nginx
ps -ef | grep java

# 트리 형태로 표시
ps auxf
pstree -p

# 특정 열 선택
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu

# 스레드 표시
ps -eLf
ps auxH

# 주요 열 설명
# PID   - 프로세스 ID
# PPID  - 부모 프로세스 ID
# TTY   - 터미널
# STAT  - 상태 (R:실행, S:슬립, D:IO대기, Z:좀비, T:정지)
# TIME  - CPU 사용 시간
# CMD   - 명령어
```

### 2.2 top / htop

```bash
# top 기본 실행
top

# htop (더 직관적인 UI)
htop

# top 단축키
# q - 종료
# M - 메모리 사용량 정렬
# P - CPU 사용량 정렬
# k - 프로세스 kill
# 1 - 각 CPU 코어 표시
# H - 스레드 표시
# c - 전체 명령어 경로 표시

# top 배치 모드 (스크립트용)
top -b -n 1

# 특정 프로세스만 모니터링
top -p $(pgrep -d, nginx)

# 출력 형식 지정
top -b -n 1 -o %CPU | head -20
```

### 2.3 netstat / ss

```bash
# netstat (레거시, 대부분의 시스템에서 사용 가능)
netstat -tuln          # TCP/UDP 리스닝 포트
netstat -tulnp         # 프로세스 정보 포함 (root)
netstat -an            # 모든 연결
netstat -s             # 프로토콜 통계

# ss (Socket Statistics, 현대적 대체)
ss -tuln               # TCP/UDP 리스닝 포트
ss -tulnp              # 프로세스 정보 포함
ss -s                  # 요약 통계
ss -o state established # 연결된 소켓만

# 특정 포트 확인
ss -tuln | grep :8080
netstat -tuln | grep :3306

# 연결 상태별 카운트
ss -ant | awk '{print $1}' | sort | uniq -c

# 옵션 설명
# -t : TCP
# -u : UDP
# -l : 리스닝 소켓만
# -n : 숫자로 표시 (DNS 조회 안함)
# -p : 프로세스 정보
# -a : 모든 소켓
```

### 2.4 lsof (List Open Files)

```bash
# 모든 열린 파일
lsof

# 특정 프로세스의 열린 파일
lsof -p $(pgrep nginx)

# 특정 포트 사용 프로세스
lsof -i :8080
lsof -i :80,443

# TCP 연결
lsof -i TCP
lsof -i TCP:80

# 특정 사용자
lsof -u nginx

# 특정 파일 사용 프로세스
lsof /var/log/syslog

# 삭제된 파일 (아직 열려있는)
lsof | grep deleted

# 네트워크 연결 상태
lsof -i -n -P

# 디렉토리 내 열린 파일
lsof +D /var/log

# 결과 예시 해석
# COMMAND  PID   USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
# nginx    1234  root   3u   IPv4  12345    0t0    TCP *:80 (LISTEN)
# FD: 파일 디스크립터 (r:읽기, w:쓰기, u:읽기/쓰기)
```

### 2.5 grep / ripgrep

```bash
# 기본 검색
grep "error" /var/log/syslog
grep -i "error" file.txt    # 대소문자 무시
grep -r "TODO" ./           # 재귀 검색
grep -n "pattern" file.txt  # 줄 번호 표시
grep -c "error" file.txt    # 일치 횟수
grep -l "error" *.log       # 파일명만 출력

# 정규표현식
grep -E "error|warning" file.txt
grep -P '\d{3}-\d{4}' file.txt  # Perl 정규식

# 컨텍스트 출력
grep -A 3 "error" file.txt  # 일치 후 3줄
grep -B 3 "error" file.txt  # 일치 전 3줄
grep -C 3 "error" file.txt  # 전후 3줄

# 역 검색 (일치하지 않는 줄)
grep -v "debug" file.txt

# 바이너리 파일 제외
grep -I "text" *

# ripgrep (rg) - 더 빠른 대안
rg "pattern"
rg -i "error" --type py    # Python 파일만
rg -l "TODO"               # 파일명만
rg -c "error"              # 카운트
rg -g "*.log" "error"      # glob 패턴
rg --hidden "secret"       # 숨김 파일 포함
```

### 2.6 find

```bash
# 이름으로 검색
find /var -name "*.log"
find . -iname "readme*"    # 대소문자 무시

# 타입으로 검색
find . -type f             # 파일만
find . -type d             # 디렉토리만
find . -type l             # 심볼릭 링크만

# 크기로 검색
find . -size +100M         # 100MB 이상
find . -size -1k           # 1KB 미만
find . -empty              # 빈 파일/디렉토리

# 시간으로 검색
find . -mtime -1           # 1일 내 수정된 파일
find . -mtime +30          # 30일 이상 된 파일
find . -mmin -60           # 60분 내 수정된 파일
find . -newer file.txt     # file.txt보다 새로운 파일

# 권한으로 검색
find . -perm 755
find . -perm -u+x          # 실행 가능한 파일

# 조합
find . -name "*.log" -size +10M -mtime +7

# 명령어 실행
find . -name "*.tmp" -exec rm {} \;
find . -name "*.py" -exec grep -l "import" {} +
find . -type f -exec chmod 644 {} \;

# xargs와 조합
find . -name "*.txt" | xargs grep "pattern"
find . -name "*.log" -print0 | xargs -0 rm  # 공백 처리
```

### 2.7 awk / sed

```bash
# awk: 텍스트 처리
awk '{print $1, $3}' file.txt        # 1, 3번째 필드
awk -F: '{print $1}' /etc/passwd     # 구분자 지정
awk '$3 > 100 {print $0}' file.txt   # 조건 필터
awk '{sum += $1} END {print sum}'    # 합계
awk 'NR > 1' file.txt                # 헤더 제외

# 실용적인 awk 예제
ps aux | awk '{print $2, $11}'       # PID와 명령어
df -h | awk 'NR>1 {print $5, $6}'    # 디스크 사용량
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head

# sed: 스트림 에디터
sed 's/old/new/' file.txt            # 첫 번째만 치환
sed 's/old/new/g' file.txt           # 모두 치환
sed -i 's/old/new/g' file.txt        # 원본 파일 수정
sed -n '5,10p' file.txt              # 5-10줄 출력
sed '/pattern/d' file.txt            # 패턴 삭제
sed '/^#/d' file.txt                 # 주석 삭제
sed '1i\header' file.txt             # 첫 줄에 삽입

# 실용적인 sed 예제
sed -i 's/localhost/0.0.0.0/g' config.yaml
sed -n '/error/p' /var/log/syslog
```

### 2.8 curl / wget

```bash
# curl: HTTP 클라이언트
curl http://example.com              # GET 요청
curl -o file.html http://example.com # 파일로 저장
curl -O http://example.com/file.zip  # 원본 파일명으로 저장

curl -X POST http://api.example.com/data
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" http://api.example.com

curl -I http://example.com           # 헤더만
curl -v http://example.com           # 상세 출력
curl -s http://example.com           # 조용히 (스크립트용)
curl -L http://example.com           # 리다이렉트 따라가기
curl -u user:pass http://example.com # 인증
curl -k https://example.com          # SSL 검증 무시

# 다운로드 진행률
curl -# -O http://example.com/large_file.zip

# wget: 파일 다운로드
wget http://example.com/file.zip
wget -O output.zip http://example.com/file.zip
wget -c http://example.com/file.zip  # 이어받기
wget -r -l 2 http://example.com      # 재귀 다운로드
wget -q http://example.com           # 조용히
wget --mirror http://example.com     # 미러링
```

---

## 3. 쉘 스크립트 기초

### 3.1 스크립트 기본 구조

```bash
#!/bin/bash
# 스크립트 설명
# Author: Your Name
# Date: 2024-12-15

# Strict 모드 (권장)
set -euo pipefail

# 변수
NAME="World"
readonly CONST_VAR="immutable"

# 함수
greet() {
    local name="${1:-Guest}"  # 기본값
    echo "Hello, ${name}!"
}

# 메인 로직
main() {
    greet "$NAME"
}

# 스크립트 직접 실행 시에만 main 호출
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
```

### 3.2 변수와 인용

```bash
# 변수 할당 (= 주위에 공백 없음)
name="John"
age=30

# 변수 사용
echo "$name"
echo "${name}"
echo "Name: ${name}, Age: ${age}"

# 특수 변수
echo $0    # 스크립트 이름
echo $1    # 첫 번째 인자
echo $#    # 인자 개수
echo $@    # 모든 인자 (개별)
echo $*    # 모든 인자 (하나의 문자열)
echo $?    # 마지막 명령어 종료 코드
echo $$    # 현재 프로세스 ID
echo $!    # 마지막 백그라운드 프로세스 ID

# 인용 (Quoting)
echo "$name"          # 변수 확장됨
echo '$name'          # 문자 그대로 '$name'
echo "It's \"$name\"" # 이스케이프

# 명령어 치환
date=$(date +%Y-%m-%d)
files=$(ls *.txt 2>/dev/null)

# 산술 연산
result=$((1 + 2))
((count++))
```

### 3.3 조건문

```bash
# if 문
if [[ condition ]]; then
    echo "true"
elif [[ other_condition ]]; then
    echo "other"
else
    echo "false"
fi

# 문자열 비교
if [[ "$str1" == "$str2" ]]; then echo "equal"; fi
if [[ "$str1" != "$str2" ]]; then echo "not equal"; fi
if [[ -z "$str" ]]; then echo "empty"; fi
if [[ -n "$str" ]]; then echo "not empty"; fi
if [[ "$str" == *"substring"* ]]; then echo "contains"; fi
if [[ "$str" =~ ^[0-9]+$ ]]; then echo "numeric"; fi

# 숫자 비교
if [[ $num -eq 10 ]]; then echo "equal"; fi
if [[ $num -ne 10 ]]; then echo "not equal"; fi
if [[ $num -lt 10 ]]; then echo "less than"; fi
if [[ $num -gt 10 ]]; then echo "greater than"; fi
if [[ $num -le 10 ]]; then echo "less or equal"; fi
if [[ $num -ge 10 ]]; then echo "greater or equal"; fi

# (()) 산술 비교
if (( num > 10 )); then echo "greater"; fi
if (( num >= 10 && num <= 20 )); then echo "in range"; fi

# 파일 테스트
if [[ -e "$file" ]]; then echo "exists"; fi
if [[ -f "$file" ]]; then echo "is file"; fi
if [[ -d "$dir" ]]; then echo "is directory"; fi
if [[ -r "$file" ]]; then echo "readable"; fi
if [[ -w "$file" ]]; then echo "writable"; fi
if [[ -x "$file" ]]; then echo "executable"; fi
if [[ -s "$file" ]]; then echo "not empty"; fi

# case 문
case "$choice" in
    start|run)
        echo "Starting..."
        ;;
    stop)
        echo "Stopping..."
        ;;
    *)
        echo "Unknown option"
        ;;
esac
```

### 3.4 반복문

```bash
# for 문
for i in 1 2 3 4 5; do
    echo "$i"
done

for i in {1..10}; do
    echo "$i"
done

for i in {1..10..2}; do  # 1, 3, 5, 7, 9
    echo "$i"
done

for file in *.txt; do
    echo "Processing: $file"
done

for ((i=0; i<10; i++)); do
    echo "$i"
done

# while 문
count=0
while [[ $count -lt 5 ]]; do
    echo "$count"
    ((count++))
done

# 파일 읽기
while IFS= read -r line; do
    echo "$line"
done < file.txt

# until 문
until [[ $count -ge 5 ]]; do
    echo "$count"
    ((count++))
done

# break / continue
for i in {1..10}; do
    if [[ $i -eq 5 ]]; then
        continue
    fi
    if [[ $i -eq 8 ]]; then
        break
    fi
    echo "$i"
done
```

### 3.5 함수

```bash
# 함수 정의
greet() {
    echo "Hello, $1!"
}

# local 변수
calculate() {
    local num1=$1
    local num2=$2
    local result=$((num1 + num2))
    echo $result
}

# 반환값
is_even() {
    if (( $1 % 2 == 0 )); then
        return 0  # true
    else
        return 1  # false
    fi
}

# 사용
greet "World"
sum=$(calculate 5 3)
echo "Sum: $sum"

if is_even 4; then
    echo "4 is even"
fi

# 배열 인자
process_array() {
    local arr=("$@")
    for item in "${arr[@]}"; do
        echo "Item: $item"
    done
}

my_array=(a b c d e)
process_array "${my_array[@]}"
```

### 3.6 실용적인 스크립트 예제

#### 로그 분석 스크립트

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="${1:-/var/log/syslog}"
OUTPUT_FILE="${2:-log_analysis.txt}"

analyze_logs() {
    local log_file="$1"

    echo "=== Log Analysis Report ===" > "$OUTPUT_FILE"
    echo "File: $log_file" >> "$OUTPUT_FILE"
    echo "Date: $(date)" >> "$OUTPUT_FILE"
    echo "" >> "$OUTPUT_FILE"

    # 에러 카운트
    echo "=== Error Count ===" >> "$OUTPUT_FILE"
    grep -c -i "error" "$log_file" 2>/dev/null || echo "0" >> "$OUTPUT_FILE"

    # 상위 IP 주소 (access log의 경우)
    echo "" >> "$OUTPUT_FILE"
    echo "=== Top 10 IPs ===" >> "$OUTPUT_FILE"
    awk '{print $1}' "$log_file" 2>/dev/null | \
        sort | uniq -c | sort -rn | head -10 >> "$OUTPUT_FILE" || echo "N/A"

    # 시간대별 요청 수
    echo "" >> "$OUTPUT_FILE"
    echo "=== Requests by Hour ===" >> "$OUTPUT_FILE"
    awk -F'[: ]' '{print $5}' "$log_file" 2>/dev/null | \
        sort | uniq -c | sort -n >> "$OUTPUT_FILE" || echo "N/A"

    echo "Analysis complete. Output: $OUTPUT_FILE"
}

# 파일 존재 확인
if [[ ! -f "$LOG_FILE" ]]; then
    echo "Error: File not found: $LOG_FILE"
    exit 1
fi

analyze_logs "$LOG_FILE"
```

#### 백업 스크립트

```bash
#!/bin/bash
set -euo pipefail

# 설정
SOURCE_DIR="${SOURCE_DIR:-/var/www}"
BACKUP_DIR="${BACKUP_DIR:-/backup}"
RETENTION_DAYS="${RETENTION_DAYS:-7}"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${DATE}.tar.gz"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

create_backup() {
    log "Starting backup of $SOURCE_DIR"

    # 백업 디렉토리 생성
    mkdir -p "$BACKUP_DIR"

    # tar로 압축
    tar -czf "${BACKUP_DIR}/${BACKUP_NAME}" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

    log "Backup created: ${BACKUP_DIR}/${BACKUP_NAME}"

    # 크기 출력
    local size=$(du -h "${BACKUP_DIR}/${BACKUP_NAME}" | cut -f1)
    log "Backup size: $size"
}

cleanup_old_backups() {
    log "Cleaning up backups older than $RETENTION_DAYS days"

    find "$BACKUP_DIR" -name "backup_*.tar.gz" -type f -mtime +$RETENTION_DAYS -delete

    log "Cleanup complete"
}

# 메인
main() {
    create_backup
    cleanup_old_backups
    log "Backup process completed"
}

main
```

#### 서비스 헬스체크 스크립트

```bash
#!/bin/bash
set -euo pipefail

# 서비스 목록
declare -A SERVICES=(
    ["web"]="http://localhost:8080/health"
    ["api"]="http://localhost:3000/ping"
    ["db"]="localhost:5432"
)

SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

check_http() {
    local name="$1"
    local url="$2"

    if curl -sf --max-time 5 "$url" > /dev/null 2>&1; then
        log "[OK] $name is healthy"
        return 0
    else
        log "[FAIL] $name is not responding"
        return 1
    fi
}

check_port() {
    local name="$1"
    local host_port="$2"

    local host="${host_port%:*}"
    local port="${host_port#*:}"

    if nc -z -w5 "$host" "$port" 2>/dev/null; then
        log "[OK] $name is reachable on port $port"
        return 0
    else
        log "[FAIL] $name is not reachable on port $port"
        return 1
    fi
}

send_alert() {
    local message="$1"

    if [[ -n "$SLACK_WEBHOOK" ]]; then
        curl -sf -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"$message\"}" \
            "$SLACK_WEBHOOK" > /dev/null
    fi
}

main() {
    local failed_services=()

    for name in "${!SERVICES[@]}"; do
        local endpoint="${SERVICES[$name]}"

        if [[ "$endpoint" == http* ]]; then
            if ! check_http "$name" "$endpoint"; then
                failed_services+=("$name")
            fi
        else
            if ! check_port "$name" "$endpoint"; then
                failed_services+=("$name")
            fi
        fi
    done

    if [[ ${#failed_services[@]} -gt 0 ]]; then
        local message="Service Alert: ${failed_services[*]} failed health check"
        send_alert "$message"
        exit 1
    fi

    log "All services are healthy"
}

main
```

---

## 4. 시스템 모니터링 명령어

### 4.1 CPU 모니터링

```bash
# CPU 정보
lscpu
cat /proc/cpuinfo

# CPU 사용률 (실시간)
top
htop
mpstat 1           # 1초 간격
mpstat -P ALL 1    # 모든 코어

# CPU 로드 평균
uptime
cat /proc/loadavg

# 프로세스별 CPU
ps aux --sort=-%cpu | head

# vmstat
vmstat 1 5         # 1초 간격, 5회
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  r: 실행 대기 프로세스
#  b: 블록된 프로세스
#  us: 사용자 CPU
#  sy: 시스템 CPU
#  id: 유휴
#  wa: I/O 대기
```

### 4.2 메모리 모니터링

```bash
# 메모리 정보
free -h
cat /proc/meminfo

# 상세 메모리 사용
vmstat -s

# 프로세스별 메모리
ps aux --sort=-%mem | head
pmap -x <PID>

# 캐시/버퍼 정리 (root 권한)
sync; echo 3 > /proc/sys/vm/drop_caches

# 스왑 사용량
swapon --show
cat /proc/swaps

# OOM 킬러 로그
dmesg | grep -i "out of memory"
journalctl -k | grep -i "oom"
```

### 4.3 디스크 모니터링

```bash
# 디스크 사용량
df -h
df -i              # inode 사용량

# 디렉토리별 사용량
du -sh *
du -sh /var/*
du -ah --max-depth=1 | sort -rh | head

# 큰 파일 찾기
find / -type f -size +100M 2>/dev/null

# 디스크 I/O
iostat -x 1        # 1초 간격
iotop              # 프로세스별 I/O (root 권한)

# iostat 출력 해석
# %util: 장치 사용률 (100% = 병목)
# await: 평균 I/O 대기 시간 (ms)
# r/s, w/s: 초당 읽기/쓰기 횟수

# 디스크 상태
smartctl -a /dev/sda   # SMART 정보 (smartmontools)
hdparm -t /dev/sda     # 읽기 속도 테스트
```

### 4.4 네트워크 모니터링

```bash
# 네트워크 인터페이스
ip addr
ip link
ifconfig           # 레거시

# 라우팅 테이블
ip route
route -n           # 레거시

# 네트워크 통계
netstat -i
ip -s link

# 연결 상태
ss -s              # 요약
ss -ant            # TCP 연결
ss -antu           # TCP + UDP

# 연결 상태별 카운트
ss -ant | awk 'NR>1 {print $1}' | sort | uniq -c

# 실시간 트래픽
iftop              # 인터페이스별 트래픽
nethogs            # 프로세스별 트래픽
vnstat             # 트래픽 통계

# 패킷 분석
tcpdump -i eth0
tcpdump -i eth0 port 80
tcpdump -i eth0 host 192.168.1.1

# DNS 조회
nslookup example.com
dig example.com
host example.com

# 연결 테스트
ping -c 4 google.com
traceroute google.com
mtr google.com     # 실시간 traceroute
```

### 4.5 시스템 로그

```bash
# 시스템 로그 (systemd)
journalctl
journalctl -u nginx          # 특정 서비스
journalctl -f                # 실시간 (tail -f)
journalctl --since "1 hour ago"
journalctl -p err            # 에러만
journalctl -k                # 커널 메시지

# 전통적인 로그 파일
tail -f /var/log/syslog
tail -f /var/log/messages
tail -f /var/log/auth.log    # 인증 로그
tail -f /var/log/kern.log    # 커널 로그

# 로그 검색
grep -r "error" /var/log/
zgrep "error" /var/log/*.gz  # 압축 파일 검색

# 부팅 로그
dmesg
dmesg -T                     # 사람이 읽을 수 있는 시간

# 마지막 로그인
last
lastlog
```

### 4.6 종합 모니터링 스크립트

```bash
#!/bin/bash
# system_health.sh - 시스템 상태 요약

echo "=========================================="
echo "System Health Report - $(date)"
echo "=========================================="

echo ""
echo "=== System Info ==="
hostname
uname -a
uptime

echo ""
echo "=== CPU Usage ==="
mpstat 1 1 | tail -1

echo ""
echo "=== Memory Usage ==="
free -h

echo ""
echo "=== Disk Usage ==="
df -h | grep -v tmpfs

echo ""
echo "=== Top 5 CPU Consuming Processes ==="
ps aux --sort=-%cpu | head -6

echo ""
echo "=== Top 5 Memory Consuming Processes ==="
ps aux --sort=-%mem | head -6

echo ""
echo "=== Network Connections ==="
ss -s

echo ""
echo "=== Recent Errors (last 10) ==="
journalctl -p err --no-pager -n 10 2>/dev/null || tail -20 /var/log/syslog | grep -i error

echo ""
echo "=========================================="
```

---

## 6. 참고 자료

### 공식 문서

- [GNU Bash Manual](https://www.gnu.org/software/bash/manual/)
- [Zsh Documentation](https://zsh.sourceforge.io/Doc/)
- [Fish Shell Documentation](https://fishshell.com/docs/current/)
- [POSIX Shell Command Language](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/V3_chap02.html)

### 학습 자료

- [Bash Scripting Tutorial](https://linuxconfig.org/bash-scripting-tutorial)
- [ShellCheck - 스크립트 정적 분석](https://www.shellcheck.net/)
- [explainshell.com - 명령어 설명](https://explainshell.com/)

### 도구

```bash
# 스크립트 품질 검사
shellcheck script.sh

# 스크립트 포매팅
shfmt -w script.sh

# 명령어 설명
man <command>
tldr <command>     # 간단한 예제
```

---

> **문서 작성일**: 2024년 12월
>
> **참고**: 이 문서는 GNU Bash, POSIX 표준, Linux man-pages를 기반으로 작성되었습니다.

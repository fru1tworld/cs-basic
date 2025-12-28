# POSIX (Portable Operating System Interface)

> **POSIX.1-2017** 표준 기반

---

## 목차

1. [POSIX 표준 개요](#1-posix-표준-개요)
2. [파일 시스템 인터페이스](#2-파일-시스템-인터페이스)
3. [프로세스 관리 API](#3-프로세스-관리-api)
4. [신호 (Signal) 처리](#4-신호-signal-처리)
6. [참고 자료](#6-참고-자료)

---

## 1. POSIX 표준 개요

### 1.1 POSIX란?

**POSIX(Portable Operating System Interface)**는 IEEE에서 정의한 UNIX 계열 운영체제의 표준 인터페이스입니다.

```
POSIX의 목표:
├── 이식성 (Portability): 다양한 UNIX 시스템에서 동일 코드 실행
├── 호환성 (Compatibility): 표준 API를 통한 상호 운용성
└── 표준화 (Standardization): 시스템 프로그래밍 통일

POSIX 준수 시스템:
├── Linux (대부분 준수)
├── macOS (인증 받음)
├── FreeBSD, OpenBSD
├── AIX, HP-UX, Solaris
└── Windows (WSL, Cygwin을 통해 부분 지원)
```

### 1.2 POSIX 표준 구성

```
POSIX.1-2017 (IEEE Std 1003.1-2017)
├── Base Definitions (XBD)
│   └── 헤더 파일, 데이터 타입, 상수 정의
├── System Interfaces (XSH)
│   └── C 언어 API (fork, exec, open, read, write, ...)
├── Shell & Utilities (XCU)
│   └── 셸 명령어 (sh, ls, grep, awk, ...)
└── Rationale (XRAT)
    └── 표준 결정 근거 설명
```

### 1.3 POSIX 준수 확인

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    // POSIX 버전 확인
    #ifdef _POSIX_VERSION
        printf("POSIX Version: %ld\n", (long)_POSIX_VERSION);
    #endif

    // POSIX.2 (Shell & Utilities) 버전
    #ifdef _POSIX2_VERSION
        printf("POSIX.2 Version: %ld\n", (long)_POSIX2_VERSION);
    #endif

    // sysconf를 통한 런타임 확인
    printf("\nRuntime Configuration:\n");
    printf("  _SC_VERSION: %ld\n", sysconf(_SC_VERSION));
    printf("  _SC_CLK_TCK: %ld\n", sysconf(_SC_CLK_TCK));
    printf("  _SC_OPEN_MAX: %ld\n", sysconf(_SC_OPEN_MAX));
    printf("  _SC_PAGESIZE: %ld\n", sysconf(_SC_PAGESIZE));
    printf("  _SC_NPROCESSORS_ONLN: %ld\n", sysconf(_SC_NPROCESSORS_ONLN));

    // pathconf를 통한 파일 시스템 설정
    printf("\nFile System Configuration (/):\n");
    printf("  _PC_NAME_MAX: %ld\n", pathconf("/", _PC_NAME_MAX));
    printf("  _PC_PATH_MAX: %ld\n", pathconf("/", _PC_PATH_MAX));

    return 0;
}
```

### 1.4 주요 POSIX 헤더 파일

| 헤더 | 설명 |
|------|------|
| `<unistd.h>` | POSIX 표준 심볼 상수, 시스템 콜 |
| `<fcntl.h>` | 파일 제어 옵션 |
| `<sys/types.h>` | 기본 데이터 타입 |
| `<sys/stat.h>` | 파일 상태 정보 |
| `<sys/wait.h>` | 프로세스 대기 |
| `<signal.h>` | 시그널 처리 |
| `<errno.h>` | 에러 번호 |
| `<pthread.h>` | POSIX 스레드 |

---

## 2. 파일 시스템 인터페이스

### 2.1 파일 디스크립터

모든 I/O는 파일 디스크립터(정수)를 통해 수행됩니다.

```
표준 파일 디스크립터:
┌────┬────────────────┬────────────┐
│ FD │     이름       │   설명     │
├────┼────────────────┼────────────┤
│  0 │ STDIN_FILENO   │ 표준 입력  │
│  1 │ STDOUT_FILENO  │ 표준 출력  │
│  2 │ STDERR_FILENO  │ 표준 에러  │
└────┴────────────────┴────────────┘

프로세스별 파일 디스크립터 테이블:
┌─────────────────────────────────────────────┐
│ Process                                      │
│ ┌─────────────────────────────────────────┐ │
│ │ File Descriptor Table                   │ │
│ ├─────┬────────────────────────────────────┤ │
│ │  0  │ → stdin  → tty                     │ │
│ │  1  │ → stdout → tty                     │ │
│ │  2  │ → stderr → tty                     │ │
│ │  3  │ → socket → network                 │ │
│ │  4  │ → file   → /var/log/app.log        │ │
│ └─────┴────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 2.2 파일 기본 연산

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <errno.h>

void demonstrate_file_operations() {
    const char *filepath = "/tmp/posix_demo.txt";
    int fd;
    char buffer[256];
    ssize_t bytes;

    // 1. 파일 생성 및 쓰기
    fd = open(filepath, O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd == -1) {
        perror("open (write)");
        return;
    }

    const char *content = "Hello, POSIX!\nLine 2\nLine 3\n";
    bytes = write(fd, content, strlen(content));
    printf("Written %zd bytes\n", bytes);

    // fsync: 버퍼를 디스크에 동기화
    if (fsync(fd) == -1) {
        perror("fsync");
    }

    close(fd);

    // 2. 파일 읽기
    fd = open(filepath, O_RDONLY);
    if (fd == -1) {
        perror("open (read)");
        return;
    }

    while ((bytes = read(fd, buffer, sizeof(buffer) - 1)) > 0) {
        buffer[bytes] = '\0';
        printf("Read: %s", buffer);
    }

    // 3. 파일 오프셋 이동
    off_t new_pos = lseek(fd, 0, SEEK_SET);  // 시작으로 이동
    printf("\nSeeked to position: %lld\n", (long long)new_pos);

    // 부분 읽기
    bytes = read(fd, buffer, 5);
    buffer[bytes] = '\0';
    printf("First 5 bytes: '%s'\n", buffer);

    close(fd);

    // 4. 파일 정보 조회
    struct stat st;
    if (stat(filepath, &st) == 0) {
        printf("\nFile Stats:\n");
        printf("  Size: %lld bytes\n", (long long)st.st_size);
        printf("  Mode: %o\n", st.st_mode & 0777);
        printf("  inode: %llu\n", (unsigned long long)st.st_ino);
        printf("  Links: %lu\n", (unsigned long)st.st_nlink);
        printf("  UID: %d, GID: %d\n", st.st_uid, st.st_gid);
    }

    // 5. 파일 삭제
    if (unlink(filepath) == 0) {
        printf("\nFile deleted successfully\n");
    }
}

int main() {
    demonstrate_file_operations();
    return 0;
}
```

### 2.3 open() 플래그

```c
// 필수 플래그 (하나만 선택)
O_RDONLY    // 읽기 전용
O_WRONLY    // 쓰기 전용
O_RDWR      // 읽기/쓰기

// 선택적 플래그 (OR 연산으로 조합)
O_CREAT     // 파일 없으면 생성
O_EXCL      // O_CREAT와 함께, 이미 존재하면 에러
O_TRUNC     // 기존 파일 내용 삭제
O_APPEND    // 항상 파일 끝에 쓰기
O_NONBLOCK  // Non-blocking 모드
O_SYNC      // 동기 I/O (데이터+메타데이터)
O_DSYNC     // 동기 I/O (데이터만)
O_CLOEXEC   // exec 시 자동 닫힘

// 예시
int fd = open("file.txt",
              O_WRONLY | O_CREAT | O_TRUNC | O_CLOEXEC,
              0644);
```

### 2.4 디렉토리 연산

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <dirent.h>
#include <sys/stat.h>
#include <unistd.h>
#include <errno.h>

void list_directory(const char *path, int depth) {
    DIR *dir;
    struct dirent *entry;
    struct stat st;
    char fullpath[PATH_MAX];

    dir = opendir(path);
    if (dir == NULL) {
        perror("opendir");
        return;
    }

    while ((entry = readdir(dir)) != NULL) {
        // . 과 .. 건너뛰기
        if (strcmp(entry->d_name, ".") == 0 ||
            strcmp(entry->d_name, "..") == 0) {
            continue;
        }

        // 들여쓰기
        for (int i = 0; i < depth; i++) {
            printf("  ");
        }

        snprintf(fullpath, sizeof(fullpath), "%s/%s", path, entry->d_name);

        if (stat(fullpath, &st) == 0) {
            if (S_ISDIR(st.st_mode)) {
                printf("[DIR]  %s/\n", entry->d_name);
                // 재귀적으로 서브디렉토리 탐색 (depth 제한)
                if (depth < 2) {
                    list_directory(fullpath, depth + 1);
                }
            } else if (S_ISREG(st.st_mode)) {
                printf("[FILE] %s (%lld bytes)\n",
                       entry->d_name, (long long)st.st_size);
            } else if (S_ISLNK(st.st_mode)) {
                printf("[LINK] %s\n", entry->d_name);
            }
        }
    }

    closedir(dir);
}

void directory_operations_demo() {
    const char *dirname = "/tmp/posix_dir_demo";

    // 디렉토리 생성
    if (mkdir(dirname, 0755) == -1) {
        if (errno != EEXIST) {
            perror("mkdir");
            return;
        }
    }
    printf("Created directory: %s\n", dirname);

    // 현재 작업 디렉토리 확인
    char cwd[PATH_MAX];
    if (getcwd(cwd, sizeof(cwd)) != NULL) {
        printf("Current directory: %s\n", cwd);
    }

    // 작업 디렉토리 변경
    if (chdir(dirname) == 0) {
        getcwd(cwd, sizeof(cwd));
        printf("Changed to: %s\n", cwd);
    }

    // 파일 생성
    int fd = open("test.txt", O_WRONLY | O_CREAT, 0644);
    write(fd, "Hello\n", 6);
    close(fd);

    // 심볼릭 링크 생성
    symlink("test.txt", "link_to_test.txt");

    // 디렉토리 내용 출력
    printf("\nDirectory listing:\n");
    list_directory(dirname, 0);

    // 정리
    chdir("/tmp");
    unlink("/tmp/posix_dir_demo/link_to_test.txt");
    unlink("/tmp/posix_dir_demo/test.txt");
    rmdir(dirname);
    printf("\nCleaned up.\n");
}

int main() {
    directory_operations_demo();
    return 0;
}
```

### 2.5 파일 잠금 (File Locking)

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

// POSIX fcntl 기반 파일 잠금
int acquire_lock(int fd, int type, int wait) {
    struct flock lock;

    lock.l_type = type;      // F_RDLCK, F_WRLCK, F_UNLCK
    lock.l_whence = SEEK_SET;
    lock.l_start = 0;        // 잠금 시작 오프셋
    lock.l_len = 0;          // 0 = 파일 전체

    int cmd = wait ? F_SETLKW : F_SETLK;  // W = Wait (blocking)
    return fcntl(fd, cmd, &lock);
}

void lock_demo() {
    const char *filepath = "/tmp/lockfile.txt";
    int fd = open(filepath, O_RDWR | O_CREAT, 0644);

    if (fd == -1) {
        perror("open");
        return;
    }

    printf("Acquiring write lock...\n");

    // 쓰기 잠금 획득 (blocking)
    if (acquire_lock(fd, F_WRLCK, 1) == -1) {
        perror("fcntl (lock)");
        close(fd);
        return;
    }

    printf("Lock acquired. Writing...\n");
    write(fd, "Locked content\n", 15);

    // 작업 수행
    sleep(5);

    // 잠금 해제
    printf("Releasing lock...\n");
    acquire_lock(fd, F_UNLCK, 0);

    close(fd);
    unlink(filepath);
}

int main() {
    lock_demo();
    return 0;
}
```

### 2.6 Python 파일 시스템 API

```python
import os
import stat
import fcntl
from pathlib import Path

def file_operations_demo():
    """POSIX 파일 연산 Python 예제"""

    filepath = Path("/tmp/python_posix_demo.txt")

    # 1. Low-level I/O (os 모듈)
    fd = os.open(str(filepath), os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o644)
    os.write(fd, b"Hello from Python!\n")
    os.fsync(fd)  # 디스크 동기화
    os.close(fd)

    # 2. 파일 정보 조회
    st = os.stat(filepath)
    print(f"File Stats:")
    print(f"  Size: {st.st_size} bytes")
    print(f"  Mode: {oct(stat.S_IMODE(st.st_mode))}")
    print(f"  inode: {st.st_ino}")
    print(f"  UID: {st.st_uid}, GID: {st.st_gid}")

    # 파일 타입 확인
    print(f"  Is file: {stat.S_ISREG(st.st_mode)}")
    print(f"  Is directory: {stat.S_ISDIR(st.st_mode)}")

    # 3. 권한 변경
    os.chmod(filepath, 0o600)

    # 4. 심볼릭 링크
    link_path = Path("/tmp/link_demo")
    if link_path.exists():
        link_path.unlink()
    os.symlink(filepath, link_path)
    print(f"\nSymlink target: {os.readlink(link_path)}")

    # 5. 파일 잠금 (fcntl)
    with open(filepath, 'r+') as f:
        fcntl.flock(f.fileno(), fcntl.LOCK_EX)  # 배타적 잠금
        print("File locked")
        # 작업 수행...
        fcntl.flock(f.fileno(), fcntl.LOCK_UN)  # 잠금 해제
        print("File unlocked")

    # 정리
    link_path.unlink()
    filepath.unlink()
    print("\nCleaned up")

def directory_operations_demo():
    """POSIX 디렉토리 연산 Python 예제"""

    # 현재 디렉토리
    print(f"Current directory: {os.getcwd()}")

    # 디렉토리 생성
    dirpath = Path("/tmp/python_dir_demo")
    dirpath.mkdir(exist_ok=True)

    # 디렉토리 내용 열거
    for entry in os.scandir("/tmp"):
        if entry.is_file():
            print(f"[FILE] {entry.name}")
        elif entry.is_dir():
            print(f"[DIR]  {entry.name}/")
        elif entry.is_symlink():
            print(f"[LINK] {entry.name}")

    # 재귀적 탐색
    print("\nRecursive walk:")
    for root, dirs, files in os.walk("/tmp/python_dir_demo"):
        for name in files:
            print(f"  {os.path.join(root, name)}")

    # 정리
    dirpath.rmdir()

if __name__ == "__main__":
    file_operations_demo()
    print("\n" + "="*50 + "\n")
    directory_operations_demo()
```

---

## 3. 프로세스 관리 API

### 3.1 프로세스 생성

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>

void fork_demo() {
    printf("Parent PID: %d\n", getpid());

    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        exit(EXIT_FAILURE);

    } else if (pid == 0) {
        // 자식 프로세스
        printf("Child: PID=%d, Parent PID=%d\n", getpid(), getppid());

        // 환경 변수 접근
        printf("Child: HOME=%s\n", getenv("HOME"));

        // 자식은 부모의 메모리 복사본을 가짐 (Copy-on-Write)
        sleep(2);
        exit(42);  // 종료 코드

    } else {
        // 부모 프로세스
        printf("Parent: Created child with PID=%d\n", pid);

        // 자식 종료 대기
        int status;
        pid_t waited = waitpid(pid, &status, 0);

        if (WIFEXITED(status)) {
            printf("Parent: Child exited with code %d\n", WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Parent: Child killed by signal %d\n", WTERMSIG(status));
        }
    }
}

int main() {
    fork_demo();
    return 0;
}
```

### 3.2 exec 계열 함수

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

void exec_demo() {
    pid_t pid = fork();

    if (pid == 0) {
        // 자식: 다른 프로그램으로 대체

        // execl: 경로 + 인자 리스트
        // execl("/bin/ls", "ls", "-la", "/tmp", NULL);

        // execlp: PATH에서 검색 + 인자 리스트
        // execlp("ls", "ls", "-la", "/tmp", NULL);

        // execv: 경로 + 인자 배열
        char *args[] = {"ls", "-la", "/tmp", NULL};
        execv("/bin/ls", args);

        // execvp: PATH에서 검색 + 인자 배열
        // char *args[] = {"ls", "-la", "/tmp", NULL};
        // execvp("ls", args);

        // execve: 경로 + 인자 배열 + 환경 변수
        // char *envp[] = {"PATH=/bin:/usr/bin", NULL};
        // execve("/bin/ls", args, envp);

        // exec가 성공하면 여기에 도달하지 않음
        perror("exec");
        exit(EXIT_FAILURE);
    }

    waitpid(pid, NULL, 0);
}

int main() {
    exec_demo();
    return 0;
}
```

### 3.3 exec 계열 함수 비교

```
exec 함수 명명 규칙:

exec + [l|v] + [p] + [e]

l: 인자를 리스트로 전달 (가변 인자)
v: 인자를 배열(vector)로 전달
p: PATH 환경 변수에서 실행 파일 검색
e: 환경 변수 직접 지정

┌──────────┬────────────┬──────────┬────────────┐
│  함수    │   인자     │  PATH    │   환경     │
├──────────┼────────────┼──────────┼────────────┤
│ execl    │  리스트    │    X     │  상속      │
│ execv    │  배열      │    X     │  상속      │
│ execlp   │  리스트    │    O     │  상속      │
│ execvp   │  배열      │    O     │  상속      │
│ execle   │  리스트    │    X     │  지정      │
│ execve   │  배열      │    X     │  지정      │
│ execvpe  │  배열      │    O     │  지정      │
└──────────┴────────────┴──────────┴────────────┘
```

### 3.4 프로세스 대기

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

void wait_demo() {
    // 여러 자식 생성
    for (int i = 0; i < 3; i++) {
        pid_t pid = fork();

        if (pid == 0) {
            // 자식
            printf("Child %d (PID=%d) starting\n", i, getpid());
            sleep(i + 1);
            printf("Child %d exiting with code %d\n", i, i * 10);
            exit(i * 10);
        }
    }

    // 부모: 모든 자식 대기
    int status;
    pid_t pid;

    // wait(): 아무 자식이나 종료될 때까지 대기
    // waitpid(): 특정 자식 대기 가능

    while ((pid = wait(&status)) > 0) {
        if (WIFEXITED(status)) {
            printf("Child %d exited with status %d\n",
                   pid, WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Child %d killed by signal %d\n",
                   pid, WTERMSIG(status));
        } else if (WIFSTOPPED(status)) {
            printf("Child %d stopped by signal %d\n",
                   pid, WSTOPSIG(status));
        }
    }

    printf("All children terminated\n");
}

void waitpid_options_demo() {
    pid_t pid = fork();

    if (pid == 0) {
        sleep(3);
        exit(0);
    }

    // WNOHANG: Non-blocking 대기
    int status;
    pid_t result;

    while (1) {
        result = waitpid(pid, &status, WNOHANG);

        if (result == 0) {
            printf("Child still running...\n");
            sleep(1);
        } else if (result > 0) {
            printf("Child exited\n");
            break;
        } else {
            perror("waitpid");
            break;
        }
    }
}

int main() {
    printf("=== wait() demo ===\n");
    wait_demo();

    printf("\n=== waitpid() with WNOHANG ===\n");
    waitpid_options_demo();

    return 0;
}
```

### 3.5 프로세스 속성

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/resource.h>

void process_attributes_demo() {
    // 프로세스 ID
    printf("PID: %d\n", getpid());
    printf("PPID: %d\n", getppid());

    // 사용자/그룹 ID
    printf("\nUser/Group IDs:\n");
    printf("  Real UID: %d\n", getuid());
    printf("  Effective UID: %d\n", geteuid());
    printf("  Real GID: %d\n", getgid());
    printf("  Effective GID: %d\n", getegid());

    // 프로세스 그룹
    printf("\nProcess Group: %d\n", getpgrp());

    // 세션 ID
    printf("Session ID: %d\n", getsid(0));

    // nice 값 (우선순위)
    printf("\nNice value: %d\n", nice(0));

    // 자원 제한
    struct rlimit rlim;

    printf("\nResource Limits:\n");

    getrlimit(RLIMIT_NOFILE, &rlim);
    printf("  Max open files: %lld (soft), %lld (hard)\n",
           (long long)rlim.rlim_cur, (long long)rlim.rlim_max);

    getrlimit(RLIMIT_STACK, &rlim);
    printf("  Stack size: %lld KB (soft), %lld KB (hard)\n",
           (long long)rlim.rlim_cur / 1024,
           (long long)rlim.rlim_max / 1024);

    getrlimit(RLIMIT_AS, &rlim);
    printf("  Address space: %lld MB (soft), %lld MB (hard)\n",
           (long long)rlim.rlim_cur / (1024*1024),
           rlim.rlim_max == RLIM_INFINITY ? -1 :
           (long long)rlim.rlim_max / (1024*1024));
}

int main() {
    process_attributes_demo();
    return 0;
}
```

### 3.6 Python 프로세스 관리

```python
import os
import sys
import subprocess
import resource

def process_info():
    """프로세스 정보 조회"""
    print("Process Information:")
    print(f"  PID: {os.getpid()}")
    print(f"  PPID: {os.getppid()}")
    print(f"  UID: {os.getuid()}")
    print(f"  GID: {os.getgid()}")
    print(f"  CWD: {os.getcwd()}")

def fork_example():
    """fork() 예제 (Unix only)"""
    print("\n=== Fork Example ===")

    pid = os.fork()

    if pid == 0:
        # 자식 프로세스
        print(f"Child: PID={os.getpid()}, PPID={os.getppid()}")
        os._exit(0)  # 자식은 _exit 사용
    else:
        # 부모 프로세스
        print(f"Parent: Created child {pid}")
        pid, status = os.waitpid(pid, 0)
        print(f"Parent: Child {pid} exited with {os.WEXITSTATUS(status)}")

def subprocess_example():
    """subprocess 모듈 사용 예제"""
    print("\n=== Subprocess Example ===")

    # 간단한 명령어 실행
    result = subprocess.run(['ls', '-la', '/tmp'],
                           capture_output=True, text=True)
    print(f"Return code: {result.returncode}")
    print(f"Output (first 200 chars): {result.stdout[:200]}")

    # 파이프라인 구성
    p1 = subprocess.Popen(['ps', 'aux'],
                          stdout=subprocess.PIPE)
    p2 = subprocess.Popen(['grep', 'python'],
                          stdin=p1.stdout,
                          stdout=subprocess.PIPE)
    p1.stdout.close()
    output, _ = p2.communicate()
    print(f"\nPython processes:\n{output.decode()}")

def resource_limits():
    """자원 제한 조회"""
    print("\n=== Resource Limits ===")

    limits = [
        ('RLIMIT_NOFILE', resource.RLIMIT_NOFILE, 'Open files'),
        ('RLIMIT_STACK', resource.RLIMIT_STACK, 'Stack size'),
        ('RLIMIT_AS', resource.RLIMIT_AS, 'Address space'),
        ('RLIMIT_CPU', resource.RLIMIT_CPU, 'CPU time'),
    ]

    for name, const, desc in limits:
        soft, hard = resource.getrlimit(const)
        soft_str = str(soft) if soft != resource.RLIM_INFINITY else 'unlimited'
        hard_str = str(hard) if hard != resource.RLIM_INFINITY else 'unlimited'
        print(f"  {desc}: soft={soft_str}, hard={hard_str}")

if __name__ == "__main__":
    process_info()

    if sys.platform != 'win32':
        fork_example()

    subprocess_example()
    resource_limits()
```

---

## 4. 신호 (Signal) 처리

### 4.1 시그널 개요

시그널은 프로세스에 비동기적으로 전달되는 소프트웨어 인터럽트입니다.

```
주요 POSIX 시그널:
┌─────────────┬──────┬────────────────────────────────────┐
│   시그널    │ 번호 │              설명                  │
├─────────────┼──────┼────────────────────────────────────┤
│ SIGHUP      │   1  │ 터미널 연결 끊김                   │
│ SIGINT      │   2  │ 인터럽트 (Ctrl+C)                  │
│ SIGQUIT     │   3  │ 종료 + 코어 덤프 (Ctrl+\)          │
│ SIGILL      │   4  │ 잘못된 명령어                       │
│ SIGTRAP     │   5  │ 디버거 트랩                        │
│ SIGABRT     │   6  │ abort() 호출                       │
│ SIGFPE      │   8  │ 부동소수점 예외                    │
│ SIGKILL     │   9  │ 강제 종료 (캐치 불가)              │
│ SIGSEGV     │  11  │ 세그멘테이션 폴트                  │
│ SIGPIPE     │  13  │ 끊어진 파이프에 쓰기               │
│ SIGALRM     │  14  │ 알람 타이머 만료                   │
│ SIGTERM     │  15  │ 종료 요청                          │
│ SIGCHLD     │  17  │ 자식 프로세스 상태 변경            │
│ SIGCONT     │  18  │ 정지된 프로세스 계속               │
│ SIGSTOP     │  19  │ 프로세스 정지 (캐치 불가)          │
│ SIGTSTP     │  20  │ 터미널 정지 (Ctrl+Z)               │
│ SIGUSR1     │  10  │ 사용자 정의 시그널 1               │
│ SIGUSR2     │  12  │ 사용자 정의 시그널 2               │
└─────────────┴──────┴────────────────────────────────────┘
```

### 4.2 시그널 핸들러 (signal)

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

volatile sig_atomic_t got_signal = 0;

void signal_handler(int signum) {
    // 시그널 핸들러에서는 async-signal-safe 함수만 호출
    got_signal = signum;
}

void basic_signal_demo() {
    // SIGINT 핸들러 등록
    signal(SIGINT, signal_handler);

    printf("Press Ctrl+C to send SIGINT...\n");

    while (!got_signal) {
        printf(".");
        fflush(stdout);
        sleep(1);
    }

    printf("\nReceived signal: %d\n", got_signal);

    // 기본 동작 복원
    signal(SIGINT, SIG_DFL);
}

int main() {
    basic_signal_demo();
    return 0;
}
```

### 4.3 시그널 핸들러 (sigaction) - 권장

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>

volatile sig_atomic_t running = 1;

void sigterm_handler(int signum, siginfo_t *info, void *context) {
    // siginfo_t로 추가 정보 접근 가능
    // 주의: printf는 async-signal-safe가 아님 (데모 목적)
    write(STDOUT_FILENO, "Received SIGTERM\n", 17);
    running = 0;
}

void sigaction_demo() {
    struct sigaction sa;

    // 핸들러 설정
    memset(&sa, 0, sizeof(sa));
    sa.sa_sigaction = sigterm_handler;
    sa.sa_flags = SA_SIGINFO;  // siginfo_t 사용

    // 핸들러 실행 중 블록할 시그널
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask, SIGINT);

    // SIGTERM에 핸들러 등록
    if (sigaction(SIGTERM, &sa, NULL) == -1) {
        perror("sigaction");
        exit(EXIT_FAILURE);
    }

    printf("PID: %d\n", getpid());
    printf("Send SIGTERM to terminate (kill %d)\n", getpid());

    while (running) {
        printf("Running...\n");
        sleep(2);
    }

    printf("Graceful shutdown complete\n");
}

int main() {
    sigaction_demo();
    return 0;
}
```

### 4.4 시그널 블로킹

```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

void signal_blocking_demo() {
    sigset_t blocked, pending, oldmask;

    // 블록할 시그널 집합 설정
    sigemptyset(&blocked);
    sigaddset(&blocked, SIGINT);
    sigaddset(&blocked, SIGTERM);

    printf("Blocking SIGINT and SIGTERM for 5 seconds...\n");
    printf("Press Ctrl+C during this time\n");

    // 시그널 블록
    sigprocmask(SIG_BLOCK, &blocked, &oldmask);

    sleep(5);

    // 대기 중인 시그널 확인
    sigpending(&pending);
    if (sigismember(&pending, SIGINT)) {
        printf("SIGINT is pending\n");
    }
    if (sigismember(&pending, SIGTERM)) {
        printf("SIGTERM is pending\n");
    }

    printf("Unblocking signals...\n");

    // 시그널 언블록 (대기 중인 시그널이 전달됨)
    sigprocmask(SIG_SETMASK, &oldmask, NULL);

    printf("Signals unblocked\n");
}

int main() {
    signal_blocking_demo();
    return 0;
}
```

### 4.5 시그널 대기 (sigsuspend, sigwait)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include <unistd.h>
#include <pthread.h>

void sigwait_demo() {
    sigset_t sigset;
    int sig;

    // 대기할 시그널 블록
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGUSR1);
    sigaddset(&sigset, SIGUSR2);
    sigprocmask(SIG_BLOCK, &sigset, NULL);

    printf("PID: %d\n", getpid());
    printf("Waiting for SIGUSR1 or SIGUSR2...\n");
    printf("Send signal: kill -USR1 %d\n", getpid());

    // 시그널 동기적 대기
    if (sigwait(&sigset, &sig) == 0) {
        printf("Received signal: %d\n", sig);
        if (sig == SIGUSR1) {
            printf("SIGUSR1 received\n");
        } else if (sig == SIGUSR2) {
            printf("SIGUSR2 received\n");
        }
    }
}

int main() {
    sigwait_demo();
    return 0;
}
```

### 4.6 시그널 전송

```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/wait.h>

void kill_demo() {
    pid_t pid = fork();

    if (pid == 0) {
        // 자식: 시그널 대기
        printf("Child: PID=%d, waiting for signals...\n", getpid());

        while (1) {
            pause();  // 시그널 대기
        }
    } else {
        // 부모: 시그널 전송
        sleep(1);

        // SIGUSR1 전송
        printf("Parent: Sending SIGUSR1 to child\n");
        kill(pid, SIGUSR1);
        sleep(1);

        // SIGTERM 전송
        printf("Parent: Sending SIGTERM to child\n");
        kill(pid, SIGTERM);

        waitpid(pid, NULL, 0);
        printf("Parent: Child terminated\n");
    }
}

void raise_demo() {
    printf("Raising SIGINT to myself...\n");
    raise(SIGINT);  // 자신에게 시그널 전송
    printf("This won't be printed (default SIGINT handler)\n");
}

void alarm_demo() {
    signal(SIGALRM, signal_handler);

    printf("Setting alarm for 3 seconds...\n");
    alarm(3);

    printf("Waiting...\n");
    pause();  // 시그널 대기

    printf("Alarm received!\n");
}
```

### 4.7 Python 시그널 처리

```python
import signal
import os
import sys
import time

def signal_handler(signum, frame):
    """시그널 핸들러"""
    print(f"\nReceived signal: {signum} ({signal.Signals(signum).name})")

    if signum == signal.SIGINT:
        print("Graceful shutdown initiated...")
        sys.exit(0)
    elif signum == signal.SIGUSR1:
        print("Got SIGUSR1 - performing custom action")
    elif signum == signal.SIGTERM:
        print("Got SIGTERM - cleaning up...")
        sys.exit(0)

def setup_signal_handlers():
    """시그널 핸들러 등록"""
    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    # Unix only
    if hasattr(signal, 'SIGUSR1'):
        signal.signal(signal.SIGUSR1, signal_handler)
        signal.signal(signal.SIGUSR2, signal_handler)

def signal_demo():
    """시그널 데모"""
    setup_signal_handlers()

    print(f"PID: {os.getpid()}")
    print("Registered handlers for SIGINT, SIGTERM, SIGUSR1, SIGUSR2")
    print(f"Send signal: kill -USR1 {os.getpid()}")
    print("Press Ctrl+C to exit")

    # 메인 루프
    while True:
        print("Working...")
        time.sleep(2)

def alarm_demo():
    """알람 시그널 데모 (Unix only)"""
    def alarm_handler(signum, frame):
        print("Alarm triggered!")

    signal.signal(signal.SIGALRM, alarm_handler)
    signal.alarm(3)  # 3초 후 SIGALRM

    print("Waiting for alarm...")
    signal.pause()  # 시그널 대기
    print("Done")

if __name__ == "__main__":
    try:
        signal_demo()
    except KeyboardInterrupt:
        print("\nExiting...")
```

### 4.8 Async-Signal-Safe 함수

시그널 핸들러에서 안전하게 호출할 수 있는 함수 목록입니다.

```
주요 async-signal-safe 함수:
├── _Exit, _exit
├── abort
├── accept
├── access
├── alarm
├── bind
├── close
├── connect
├── dup, dup2
├── execve (exec 계열 중 유일)
├── fork
├── fsync
├── kill
├── listen
├── lseek
├── open
├── pause
├── pipe
├── poll (일부 구현)
├── read
├── recv, recvfrom
├── rename
├── send, sendto
├── shutdown
├── sigaction
├── sigprocmask
├── socket
├── stat
├── unlink
├── waitpid
└── write

주의: 다음 함수는 async-signal-safe가 아님
├── printf, fprintf (버퍼링 문제)
├── malloc, free (잠금 문제)
├── exit (atexit 핸들러 호출)
└── 대부분의 표준 라이브러리 함수
```

---

## 6. 참고 자료

### 공식 문서

- [POSIX.1-2017 (The Open Group)](https://pubs.opengroup.org/onlinepubs/9699919799/)
- [Linux man-pages Project](https://www.kernel.org/doc/man-pages/)
- [GNU C Library Manual](https://www.gnu.org/software/libc/manual/)

### 관련 서적

- "Advanced Programming in the UNIX Environment" - W. Richard Stevens
- "The Linux Programming Interface" - Michael Kerrisk
- "UNIX Network Programming" - W. Richard Stevens

### 추가 학습 자료

- [Beej's Guide to Unix IPC](https://beej.us/guide/bgipc/)
- [Signal Safety](https://man7.org/linux/man-pages/man7/signal-safety.7.html)
- [POSIX Signals Tutorial](https://www.thegeekstuff.com/2012/03/linux-signals-fundamentals/)

---

> **문서 작성일**: 2024년 12월
>
> **참고**: 이 문서는 POSIX.1-2017, Linux man-pages를 기반으로 작성되었습니다.

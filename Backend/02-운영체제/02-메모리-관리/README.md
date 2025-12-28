# 메모리 관리 (Memory Management)

> **OSTEP(Operating Systems: Three Easy Pieces)** 기반

---

## 목차

1. [가상 메모리와 물리 메모리](#1-가상-메모리와-물리-메모리)
2. [페이징과 세그멘테이션](#2-페이징과-세그멘테이션)
3. [페이지 교체 알고리즘](#3-페이지-교체-알고리즘)
4. [TLB (Translation Lookaside Buffer)](#4-tlb-translation-lookaside-buffer)
5. [메모리 단편화와 해결법](#5-메모리-단편화와-해결법)
7. [참고 자료](#7-참고-자료)

---

## 1. 가상 메모리와 물리 메모리

### 1.1 메모리 관리의 필요성

운영체제의 메모리 관리 목표:

```
1. 투명성 (Transparency)
   - 프로세스는 자신만의 연속된 메모리 공간을 가진 것처럼 동작

2. 효율성 (Efficiency)
   - 시간: 주소 변환 오버헤드 최소화
   - 공간: 메모리 낭비 최소화

3. 보호 (Protection)
   - 프로세스 간 메모리 격리
   - 커널 영역 보호
```

### 1.2 물리 메모리 (Physical Memory)

실제 RAM 하드웨어의 메모리 공간입니다.

```
Physical Memory Layout (x86_64)
+------------------------+ 0x0000_0000_0000_0000
|    Real Mode IVT       |
+------------------------+ 0x0000_0000_0000_0400
|    BIOS Data Area      |
+------------------------+ 0x0000_0000_0000_0500
|    Bootloader          |
+------------------------+ 0x0000_0000_0000_7C00
|         ...            |
+------------------------+ 0x0000_0000_0010_0000 (1MB)
|    Kernel Code/Data    |
+------------------------+
|    Kernel Heap         |
+------------------------+
|    User Space Pages    |
+------------------------+
|         ...            |
+------------------------+ RAM Top
```

### 1.3 가상 메모리 (Virtual Memory)

각 프로세스에게 독립된 연속 주소 공간을 제공하는 추상화 계층입니다.

```
Virtual Address Space (64-bit Linux, 48-bit addressing)

User Space (128TB)
+------------------------+ 0x0000_0000_0000_0000
|        NULL Guard      |
+------------------------+ 0x0000_0000_0001_0000
|         Text           | ← 실행 코드
+------------------------+
|         Data           | ← 초기화된 전역 변수
+------------------------+
|          BSS           | ← 초기화되지 않은 전역 변수
+------------------------+
|          Heap          | ← 동적 할당 (brk, mmap)
|           ↓            |
+------------------------+
|    Memory Mapped       | ← 공유 라이브러리, mmap
|         Region         |
+------------------------+
|           ↑            |
|         Stack          | ← 지역 변수, 함수 호출
+------------------------+ 0x0000_7FFF_FFFF_FFFF

Non-canonical Gap (16EB)
+------------------------+ 0x0000_8000_0000_0000
|      Not Used          |
+------------------------+ 0xFFFF_7FFF_FFFF_FFFF

Kernel Space (128TB)
+------------------------+ 0xFFFF_8000_0000_0000
|    Kernel Memory       |
+------------------------+ 0xFFFF_FFFF_FFFF_FFFF
```

### 1.4 주소 변환 (Address Translation)

```
           Virtual Address
                 │
                 ▼
         ┌──────────────┐
         │     MMU      │  Memory Management Unit
         │              │
         │  Page Table  │
         │   Lookup     │
         └──────────────┘
                 │
                 ▼
          Physical Address
```

```c
// 개념적 주소 변환 구현
#include <stdint.h>

#define PAGE_SIZE 4096
#define PAGE_SHIFT 12
#define PAGE_MASK (~(PAGE_SIZE - 1))

// 가상 주소에서 페이지 번호 추출
static inline uint64_t get_page_number(uint64_t virtual_addr) {
    return virtual_addr >> PAGE_SHIFT;
}

// 가상 주소에서 오프셋 추출
static inline uint64_t get_page_offset(uint64_t virtual_addr) {
    return virtual_addr & (PAGE_SIZE - 1);
}

// 페이지 테이블 엔트리에서 물리 프레임 번호 추출
static inline uint64_t get_frame_number(uint64_t pte) {
    return (pte & PAGE_MASK) >> PAGE_SHIFT;
}

// 주소 변환
uint64_t translate_address(uint64_t virtual_addr, uint64_t *page_table) {
    uint64_t page_num = get_page_number(virtual_addr);
    uint64_t offset = get_page_offset(virtual_addr);

    // 페이지 테이블에서 PTE 조회
    uint64_t pte = page_table[page_num];

    // Present 비트 확인
    if (!(pte & 0x1)) {
        // Page Fault!
        return -1;
    }

    uint64_t frame_num = get_frame_number(pte);
    uint64_t physical_addr = (frame_num << PAGE_SHIFT) | offset;

    return physical_addr;
}
```

### 1.5 가상 메모리 정보 조회 (Python)

```python
import os
import re
from dataclasses import dataclass
from typing import List

@dataclass
class MemoryRegion:
    start: int
    end: int
    permissions: str
    offset: int
    device: str
    inode: int
    pathname: str

    @property
    def size(self) -> int:
        return self.end - self.start

    @property
    def size_human(self) -> str:
        size = self.size
        for unit in ['B', 'KB', 'MB', 'GB']:
            if size < 1024:
                return f"{size:.1f} {unit}"
            size /= 1024
        return f"{size:.1f} TB"

def parse_maps(pid: int = None) -> List[MemoryRegion]:
    """프로세스의 메모리 맵 파싱 (/proc/[pid]/maps)"""
    if pid is None:
        pid = os.getpid()

    regions = []
    maps_path = f"/proc/{pid}/maps"

    try:
        with open(maps_path, 'r') as f:
            for line in f:
                parts = line.strip().split()
                addr_range = parts[0].split('-')

                region = MemoryRegion(
                    start=int(addr_range[0], 16),
                    end=int(addr_range[1], 16),
                    permissions=parts[1],
                    offset=int(parts[2], 16),
                    device=parts[3],
                    inode=int(parts[4]),
                    pathname=parts[5] if len(parts) > 5 else ""
                )
                regions.append(region)
    except FileNotFoundError:
        print(f"Process {pid} not found")
        return []

    return regions

def analyze_memory_layout(pid: int = None):
    """메모리 레이아웃 분석"""
    regions = parse_maps(pid)

    # 영역별 분류
    categories = {
        'heap': [],
        'stack': [],
        'code': [],
        'data': [],
        'libraries': [],
        'mmap': [],
        'other': []
    }

    for region in regions:
        path = region.pathname.lower()
        perms = region.permissions

        if '[heap]' in path:
            categories['heap'].append(region)
        elif '[stack]' in path:
            categories['stack'].append(region)
        elif '[vdso]' in path or '[vvar]' in path:
            categories['other'].append(region)
        elif '.so' in path:
            categories['libraries'].append(region)
        elif path and '/' in path:
            if 'x' in perms:
                categories['code'].append(region)
            else:
                categories['data'].append(region)
        else:
            categories['mmap'].append(region)

    print("=== Memory Layout Analysis ===\n")
    for cat, regs in categories.items():
        if regs:
            total_size = sum(r.size for r in regs)
            print(f"{cat.upper()}: {len(regs)} regions, Total: {total_size / 1024 / 1024:.2f} MB")
            for r in regs[:3]:  # 상위 3개만 출력
                print(f"  {r.start:016x}-{r.end:016x} {r.permissions} {r.size_human:>10} {r.pathname}")
            if len(regs) > 3:
                print(f"  ... and {len(regs) - 3} more regions")
        print()

if __name__ == "__main__":
    analyze_memory_layout()
```

---

## 2. 페이징과 세그멘테이션

### 2.1 페이징 (Paging)

메모리를 고정 크기의 **페이지(Page)**로 분할하는 기법입니다.

```
Virtual Memory              Physical Memory
(Pages)                     (Frames)

+----------+                +----------+
| Page 0   | ───────────────► Frame 5  |
+----------+                +----------+
| Page 1   | ───────────────► Frame 2  |
+----------+                +----------+
| Page 2   | ─────┐         | Frame 0  |
+----------+      │         +----------+
| Page 3   | ──┐  │         | Frame 1  |
+----------+   │  │         +----------+
               │  └────────►| Frame 2  |
               │            +----------+
               │            | Frame 3  |
               │            +----------+
               │            | Frame 4  |
               │            +----------+
               └───────────►| Frame 5  |
                            +----------+
```

#### 페이지 테이블 엔트리 (PTE) 구조 - x86_64

```
63    62    52 51        12 11   9 8 7 6 5 4 3 2 1 0
+--+--+------+-------------+------+-+-+-+-+-+-+-+-+-+
|XD|  | RSVD |  PFN (40b)  | AVL  |G|S|D|A|C|W|U|R|P|
+--+--+------+-------------+------+-+-+-+-+-+-+-+-+-+

P   (Present):       페이지가 물리 메모리에 존재하는지
R/W (Read/Write):    읽기 전용(0) 또는 읽기/쓰기(1)
U/S (User/Supervisor): 사용자 모드 접근 허용(1)
PWT (Write-Through):  Write-through 캐시 정책
PCD (Cache Disable):  캐시 비활성화
A   (Accessed):      최근 접근 여부
D   (Dirty):         페이지 수정 여부
PS  (Page Size):     대용량 페이지 (2MB/1GB)
G   (Global):        TLB에서 플러시하지 않음
PFN (Page Frame Number): 물리 프레임 번호
XD  (Execute Disable): 실행 금지 (NX 비트)
```

#### 다단계 페이지 테이블 (Multi-level Page Table)

```
48-bit Virtual Address (x86_64 4-level paging)

┌─────────────────────────────────────────────────────────────┐
│ PML4  │  PDPT  │   PD   │   PT   │    Page Offset          │
│ [47:39]│ [38:30]│ [29:21]│ [20:12]│      [11:0]             │
│  9bit │  9bit  │  9bit  │  9bit  │      12bit              │
└─────────────────────────────────────────────────────────────┘
    │        │        │        │            │
    │        │        │        │            │
    ▼        ▼        ▼        ▼            │
┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         │
│PML4  │→│PDPT  │→│ PD   │→│ PT   │→ Frame ─┘
│Table │ │Table │ │Table │ │Table │   + Offset
└──────┘ └──────┘ └──────┘ └──────┘     ↓
                                    Physical Address
```

```c
// 4-level 페이지 테이블 워킹 구현
#include <stdint.h>

#define PAGE_PRESENT    (1UL << 0)
#define PAGE_WRITE      (1UL << 1)
#define PAGE_USER       (1UL << 2)
#define PAGE_SIZE_FLAG  (1UL << 7)

// 각 레벨의 인덱스 추출
#define PML4_INDEX(va)  (((va) >> 39) & 0x1FF)
#define PDPT_INDEX(va)  (((va) >> 30) & 0x1FF)
#define PD_INDEX(va)    (((va) >> 21) & 0x1FF)
#define PT_INDEX(va)    (((va) >> 12) & 0x1FF)
#define PAGE_OFFSET(va) ((va) & 0xFFF)

typedef uint64_t pte_t;
typedef uint64_t phys_addr_t;
typedef uint64_t virt_addr_t;

phys_addr_t walk_page_table(pte_t *pml4, virt_addr_t va) {
    pte_t pml4e = pml4[PML4_INDEX(va)];
    if (!(pml4e & PAGE_PRESENT)) {
        return (phys_addr_t)-1;  // Page fault
    }

    pte_t *pdpt = (pte_t *)(pml4e & ~0xFFFUL);
    pte_t pdpte = pdpt[PDPT_INDEX(va)];
    if (!(pdpte & PAGE_PRESENT)) {
        return (phys_addr_t)-1;
    }

    // 1GB Huge Page 체크
    if (pdpte & PAGE_SIZE_FLAG) {
        return (pdpte & ~0x3FFFFFFFUL) | (va & 0x3FFFFFFFUL);
    }

    pte_t *pd = (pte_t *)(pdpte & ~0xFFFUL);
    pte_t pde = pd[PD_INDEX(va)];
    if (!(pde & PAGE_PRESENT)) {
        return (phys_addr_t)-1;
    }

    // 2MB Huge Page 체크
    if (pde & PAGE_SIZE_FLAG) {
        return (pde & ~0x1FFFFFUL) | (va & 0x1FFFFFUL);
    }

    pte_t *pt = (pte_t *)(pde & ~0xFFFUL);
    pte_t pte = pt[PT_INDEX(va)];
    if (!(pte & PAGE_PRESENT)) {
        return (phys_addr_t)-1;
    }

    return (pte & ~0xFFFUL) | PAGE_OFFSET(va);
}
```

### 2.2 세그멘테이션 (Segmentation)

메모리를 논리적 단위(코드, 데이터, 스택)로 분할하는 기법입니다.

```
Logical Address
┌────────────────────────────────────┐
│  Segment Number  │     Offset     │
└────────────────────────────────────┘
         │                  │
         ▼                  │
   ┌───────────┐           │
   │  Segment  │           │
   │   Table   │           │
   ├───────────┤           │
   │ Base│Limit│           │
   └──┬──┴──┬──┘           │
      │     │              │
      │     └── Bound Check ◄──┘
      │              │
      ▼              │
   Base + Offset ◄───┘
      │
      ▼
Physical Address
```

#### 세그멘트 테이블 예시

```c
// 세그멘트 디스크립터 (x86)
struct segment_descriptor {
    uint16_t limit_low;      // Limit [15:0]
    uint16_t base_low;       // Base [15:0]
    uint8_t  base_mid;       // Base [23:16]
    uint8_t  type:4;         // Segment type
    uint8_t  s:1;            // Descriptor type (0=system, 1=code/data)
    uint8_t  dpl:2;          // Privilege level
    uint8_t  p:1;            // Present
    uint8_t  limit_high:4;   // Limit [19:16]
    uint8_t  avl:1;          // Available for system use
    uint8_t  l:1;            // 64-bit code segment
    uint8_t  db:1;           // Default operation size
    uint8_t  g:1;            // Granularity
    uint8_t  base_high;      // Base [31:24]
} __attribute__((packed));

// 세그먼트 기반 주소 변환
uint32_t segment_translate(uint16_t selector, uint32_t offset,
                           struct segment_descriptor *gdt) {
    int index = selector >> 3;
    struct segment_descriptor *seg = &gdt[index];

    // Base 주소 조합
    uint32_t base = seg->base_low |
                   (seg->base_mid << 16) |
                   (seg->base_high << 24);

    // Limit 조합
    uint32_t limit = seg->limit_low | (seg->limit_high << 16);
    if (seg->g) {
        limit = (limit << 12) | 0xFFF;  // 4KB 단위
    }

    // 범위 검사
    if (offset > limit) {
        // General Protection Fault
        return (uint32_t)-1;
    }

    return base + offset;
}
```

### 2.3 페이징 vs 세그멘테이션

| 특징 | 페이징 | 세그멘테이션 |
|------|--------|--------------|
| **분할 단위** | 고정 크기 (4KB, 2MB, 1GB) | 가변 크기 |
| **외부 단편화** | 없음 | 발생 가능 |
| **내부 단편화** | 발생 가능 | 없음 |
| **보호** | 페이지 단위 | 세그먼트 단위 (논리적) |
| **공유** | 페이지 단위 | 세그먼트 단위 |
| **현대 OS** | 주로 사용 | 거의 사용 안 함 |

### 2.4 Segmentation with Paging

현대 x86에서는 세그멘테이션과 페이징을 함께 사용합니다.

```
Logical Address
      │
      ▼
┌─────────────┐
│ Segmentation│  (대부분 Flat Model: Base=0)
│    Unit     │
└─────────────┘
      │
      ▼
Linear Address (= Virtual Address)
      │
      ▼
┌─────────────┐
│   Paging    │
│    Unit     │
└─────────────┘
      │
      ▼
Physical Address
```

---

## 3. 페이지 교체 알고리즘

### 3.1 페이지 폴트 (Page Fault)

접근하려는 페이지가 물리 메모리에 없을 때 발생하는 예외입니다.

```
Page Fault 처리 과정:

1. CPU가 Page Fault 예외 발생
2. 커널 Page Fault Handler 실행
3. 폴트 원인 분석
   ├─ 유효한 페이지 접근 → 페이지 로드
   ├─ 보호 위반 → SIGSEGV
   └─ 무효 주소 → SIGSEGV
4. 빈 프레임 확보 (필요시 페이지 교체)
5. 디스크에서 페이지 로드
6. 페이지 테이블 갱신
7. 명령어 재실행
```

```c
// 페이지 폴트 핸들러 개념적 구현
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <ucontext.h>

void page_fault_handler(int sig, siginfo_t *info, void *context) {
    void *fault_addr = info->si_addr;

    printf("Page fault at address: %p\n", fault_addr);

    // 실제로는 여기서:
    // 1. VMA (Virtual Memory Area) 검색
    // 2. 접근 권한 확인
    // 3. 페이지 할당 또는 스왑 인

    // 시뮬레이션: 그냥 종료
    exit(1);
}

int main() {
    struct sigaction sa;
    sa.sa_sigaction = page_fault_handler;
    sa.sa_flags = SA_SIGINFO;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGSEGV, &sa, NULL);

    // 의도적으로 페이지 폴트 발생
    int *p = (int *)0xDEADBEEF;
    *p = 42;

    return 0;
}
```

### 3.2 페이지 교체 알고리즘 개요

```
프레임 수가 제한될 때, 어떤 페이지를 교체할 것인가?

목표: Page Fault 횟수 최소화 (Miss Rate ↓)

이상적: 가장 오랫동안 사용되지 않을 페이지 교체 (OPT)
       → 미래 예측 불가능하므로 근사 알고리즘 사용
```

### 3.3 FIFO (First-In, First-Out)

가장 먼저 들어온 페이지를 교체합니다.

```python
from collections import deque

def fifo_page_replacement(pages: list, num_frames: int) -> dict:
    """
    FIFO 페이지 교체 알고리즘
    """
    frames = deque(maxlen=num_frames)
    page_faults = 0
    history = []

    for page in pages:
        is_fault = page not in frames

        if is_fault:
            page_faults += 1
            if len(frames) == num_frames:
                evicted = frames.popleft()
            else:
                evicted = None
            frames.append(page)
        else:
            evicted = None

        history.append({
            'page': page,
            'frames': list(frames),
            'fault': is_fault,
            'evicted': evicted
        })

    return {
        'page_faults': page_faults,
        'hit_rate': (len(pages) - page_faults) / len(pages) * 100,
        'history': history
    }

# 테스트: Belady's Anomaly 시연
pages = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]

print("=== FIFO Page Replacement ===\n")
for frames in [3, 4]:
    result = fifo_page_replacement(pages, frames)
    print(f"Frames: {frames}")
    print(f"Page Faults: {result['page_faults']}")
    print(f"Hit Rate: {result['hit_rate']:.1f}%")
    print()

# Belady's Anomaly: 프레임이 늘어도 페이지 폴트가 증가할 수 있음!
```

**Belady's Anomaly**: FIFO에서 프레임 수를 늘렸는데 오히려 페이지 폴트가 증가하는 현상

### 3.4 LRU (Least Recently Used)

가장 오랫동안 사용되지 않은 페이지를 교체합니다.

```python
from collections import OrderedDict

class LRUCache:
    """
    LRU 페이지 교체 알고리즘 (OrderedDict 활용)
    """
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()
        self.page_faults = 0
        self.accesses = 0

    def access(self, page: int) -> bool:
        """
        페이지 접근
        Returns: True if page fault occurred
        """
        self.accesses += 1

        if page in self.cache:
            # Hit: 페이지를 가장 최근으로 이동
            self.cache.move_to_end(page)
            return False
        else:
            # Miss: 페이지 폴트
            self.page_faults += 1

            if len(self.cache) >= self.capacity:
                # LRU 페이지 제거 (가장 앞의 항목)
                evicted = self.cache.popitem(last=False)

            self.cache[page] = True
            return True

    def get_stats(self) -> dict:
        return {
            'page_faults': self.page_faults,
            'accesses': self.accesses,
            'hit_rate': (self.accesses - self.page_faults) / self.accesses * 100
        }

def lru_page_replacement(pages: list, num_frames: int) -> dict:
    """LRU 시뮬레이션"""
    cache = LRUCache(num_frames)
    history = []

    for page in pages:
        fault = cache.access(page)
        history.append({
            'page': page,
            'frames': list(cache.cache.keys()),
            'fault': fault
        })

    return {**cache.get_stats(), 'history': history}

# 테스트
pages = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]

print("=== LRU Page Replacement ===\n")
result = lru_page_replacement(pages, 3)
print(f"Page Faults: {result['page_faults']}")
print(f"Hit Rate: {result['hit_rate']:.1f}%")
print("\nStep-by-step:")
for i, h in enumerate(result['history']):
    status = "FAULT" if h['fault'] else "HIT"
    print(f"  Access {h['page']}: {h['frames']} [{status}]")
```

#### LRU 하드웨어 구현

```c
// 카운터 기반 LRU (단순화)
struct page_frame {
    int page_number;
    uint64_t last_access_time;
    int valid;
};

struct page_frame frames[MAX_FRAMES];
uint64_t global_time = 0;

int find_lru_frame() {
    int lru_index = 0;
    uint64_t min_time = UINT64_MAX;

    for (int i = 0; i < MAX_FRAMES; i++) {
        if (frames[i].valid && frames[i].last_access_time < min_time) {
            min_time = frames[i].last_access_time;
            lru_index = i;
        }
    }

    return lru_index;
}

void access_page(int page) {
    global_time++;

    // 페이지 검색
    for (int i = 0; i < MAX_FRAMES; i++) {
        if (frames[i].valid && frames[i].page_number == page) {
            frames[i].last_access_time = global_time;
            return;  // Hit
        }
    }

    // Page Fault
    int victim = find_lru_frame();
    frames[victim].page_number = page;
    frames[victim].last_access_time = global_time;
    frames[victim].valid = 1;
}
```

### 3.5 Clock Algorithm (Second Chance)

LRU의 근사 알고리즘으로, 하드웨어 Reference Bit을 활용합니다.

```
Clock 알고리즘 동작:

     ┌───┐  ┌───┐  ┌───┐  ┌───┐
     │ A │──│ B │──│ C │──│ D │
     │R=1│  │R=0│  │R=1│  │R=0│
     └───┘  └───┘  └───┘  └───┘
       ↑                    │
       └────────────────────┘
              Clock Hand

교체 알고리즘:
1. Clock Hand가 가리키는 페이지 확인
2. Reference Bit = 1 → 0으로 설정, 다음으로 이동 (Second Chance)
3. Reference Bit = 0 → 이 페이지 교체
4. 한 바퀴 돌아도 못 찾으면 처음 페이지 교체
```

```python
class ClockPageReplacement:
    """
    Clock (Second Chance) 알고리즘
    """
    def __init__(self, num_frames: int):
        self.num_frames = num_frames
        self.frames = [None] * num_frames  # (page, reference_bit)
        self.clock_hand = 0
        self.page_faults = 0

    def _find_frame(self, page: int) -> int:
        """페이지가 있는 프레임 인덱스 반환, 없으면 -1"""
        for i, frame in enumerate(self.frames):
            if frame and frame[0] == page:
                return i
        return -1

    def _find_empty_frame(self) -> int:
        """빈 프레임 인덱스 반환, 없으면 -1"""
        for i, frame in enumerate(self.frames):
            if frame is None:
                return i
        return -1

    def _find_victim(self) -> int:
        """교체할 프레임 찾기 (Clock 알고리즘)"""
        while True:
            page, ref_bit = self.frames[self.clock_hand]

            if ref_bit == 0:
                # 이 페이지 교체
                victim = self.clock_hand
                self.clock_hand = (self.clock_hand + 1) % self.num_frames
                return victim
            else:
                # Second Chance: ref_bit를 0으로 설정
                self.frames[self.clock_hand] = (page, 0)
                self.clock_hand = (self.clock_hand + 1) % self.num_frames

    def access(self, page: int) -> dict:
        """페이지 접근"""
        result = {'page': page, 'fault': False, 'evicted': None}

        # 이미 메모리에 있는지 확인
        frame_idx = self._find_frame(page)
        if frame_idx != -1:
            # Hit: Reference Bit 설정
            self.frames[frame_idx] = (page, 1)
            result['frames'] = [(f[0] if f else None, f[1] if f else None)
                               for f in self.frames]
            return result

        # Page Fault
        self.page_faults += 1
        result['fault'] = True

        # 빈 프레임 찾기
        empty_idx = self._find_empty_frame()
        if empty_idx != -1:
            self.frames[empty_idx] = (page, 1)
        else:
            # 교체 대상 찾기
            victim_idx = self._find_victim()
            result['evicted'] = self.frames[victim_idx][0]
            self.frames[victim_idx] = (page, 1)

        result['frames'] = [(f[0] if f else None, f[1] if f else None)
                           for f in self.frames]
        return result

def clock_simulation(pages: list, num_frames: int):
    """Clock 알고리즘 시뮬레이션"""
    clock = ClockPageReplacement(num_frames)
    history = []

    for page in pages:
        result = clock.access(page)
        history.append(result)

    return {
        'page_faults': clock.page_faults,
        'hit_rate': (len(pages) - clock.page_faults) / len(pages) * 100,
        'history': history
    }

# 테스트
pages = [1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5]

print("=== Clock (Second Chance) Algorithm ===\n")
result = clock_simulation(pages, 3)
print(f"Page Faults: {result['page_faults']}")
print(f"Hit Rate: {result['hit_rate']:.1f}%")
print("\nStep-by-step:")
for h in result['history']:
    status = "FAULT" if h['fault'] else "HIT"
    frames_str = [(f"P{p}(R={r})" if p else "Empty") for p, r in h['frames']]
    evicted = f" [Evicted: P{h['evicted']}]" if h['evicted'] else ""
    print(f"  Access P{h['page']}: {frames_str} [{status}]{evicted}")
```

### 3.6 Enhanced Second Chance (NRU)

Reference Bit과 Modified Bit을 함께 사용합니다.

```
우선순위 (낮은 것 먼저 교체):
1. (R=0, M=0): 최근 사용 안 됨, 수정 안 됨 (최우선 교체)
2. (R=0, M=1): 최근 사용 안 됨, 수정됨
3. (R=1, M=0): 최근 사용됨, 수정 안 됨
4. (R=1, M=1): 최근 사용됨, 수정됨 (교체 최소화)
```

### 3.7 알고리즘 성능 비교

```python
def compare_algorithms(pages: list, num_frames: int):
    """페이지 교체 알고리즘 성능 비교"""
    algorithms = {
        'FIFO': fifo_page_replacement,
        'LRU': lru_page_replacement,
        'Clock': clock_simulation,
    }

    print(f"=== Algorithm Comparison ===")
    print(f"Page Reference String: {pages}")
    print(f"Number of Frames: {num_frames}\n")

    results = {}
    for name, func in algorithms.items():
        result = func(pages, num_frames)
        results[name] = result
        print(f"{name:10s}: Page Faults = {result['page_faults']:2d}, "
              f"Hit Rate = {result['hit_rate']:.1f}%")

    return results

# 테스트
pages = [7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1]
compare_algorithms(pages, 3)
```

---

## 4. TLB (Translation Lookaside Buffer)

### 4.1 TLB란?

TLB는 **페이지 테이블 엔트리를 캐싱하는 하드웨어**입니다. 주소 변환 속도를 획기적으로 향상시킵니다.

```
Without TLB:
Virtual Address → Page Table Walk (4 memory accesses) → Physical Address
                  약 100-400 cycles

With TLB (Hit):
Virtual Address → TLB Lookup (1-2 cycles) → Physical Address
```

### 4.2 TLB 구조

```
TLB Entry:
┌─────────────────────────────────────────────────────────────┐
│ Valid │ ASID │ Virtual Page Number │ PFN │ Permission Bits │
└─────────────────────────────────────────────────────────────┘

TLB 타입:
├── Fully Associative: 어디든 저장 가능 (비쌈, 작은 TLB)
├── Set-Associative: 일부 위치에 저장 (일반적)
└── Direct-Mapped: 정해진 위치에만 저장 (저렴, 충돌 많음)
```

### 4.3 TLB 미스 처리

```c
// 소프트웨어 관리 TLB (MIPS 스타일) 개념적 구현
#include <stdint.h>

#define TLB_SIZE 64

struct tlb_entry {
    uint64_t vpn;       // Virtual Page Number
    uint64_t pfn;       // Physical Frame Number
    uint8_t  valid;
    uint8_t  asid;      // Address Space ID
    uint8_t  flags;     // Read, Write, Execute, Global
};

struct tlb_entry tlb[TLB_SIZE];

// TLB Lookup
int tlb_lookup(uint64_t vpn, uint8_t asid, uint64_t *pfn) {
    for (int i = 0; i < TLB_SIZE; i++) {
        if (tlb[i].valid &&
            (tlb[i].vpn == vpn) &&
            (tlb[i].flags & TLB_GLOBAL || tlb[i].asid == asid)) {
            *pfn = tlb[i].pfn;
            return 1;  // TLB Hit
        }
    }
    return 0;  // TLB Miss
}

// TLB Miss Handler (페이지 테이블 워킹)
void tlb_miss_handler(uint64_t faulting_addr, uint8_t asid) {
    uint64_t vpn = faulting_addr >> 12;
    uint64_t pfn;

    // 페이지 테이블에서 PTE 조회
    if (!page_table_walk(vpn, &pfn)) {
        // 페이지 폴트 - 페이지가 메모리에 없음
        handle_page_fault(faulting_addr);
        return;
    }

    // TLB 엔트리 추가 (랜덤 또는 LRU 교체)
    int victim = random() % TLB_SIZE;
    tlb[victim].vpn = vpn;
    tlb[victim].pfn = pfn;
    tlb[victim].valid = 1;
    tlb[victim].asid = asid;
}
```

### 4.4 ASID (Address Space Identifier)

컨텍스트 스위칭 시 TLB 플러시를 피하기 위한 기법입니다.

```
Without ASID:
┌────────────────────────────────────────────────────────────┐
│ Context Switch → TLB Flush (모든 엔트리 무효화)            │
│               → 새 프로세스에서 TLB Miss 폭주              │
└────────────────────────────────────────────────────────────┘

With ASID:
┌────────────────────────────────────────────────────────────┐
│ Context Switch → ASID 레지스터만 변경                       │
│               → 다른 ASID의 엔트리는 자동으로 Miss          │
│               → 같은 프로세스로 돌아오면 TLB 재활용         │
└────────────────────────────────────────────────────────────┘
```

### 4.5 TLB 성능 측정

```python
import time
import random
import mmap
import os

def measure_tlb_performance():
    """TLB 성능 측정 (간접적)"""
    page_size = 4096

    # 다양한 페이지 수로 테스트
    test_sizes = [16, 32, 64, 128, 256, 512, 1024, 2048]
    results = []

    for num_pages in test_sizes:
        # 메모리 할당
        size = num_pages * page_size
        buf = mmap.mmap(-1, size, mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS,
                        mmap.PROT_READ | mmap.PROT_WRITE)

        # 각 페이지 터치 (TLB 로드)
        for i in range(num_pages):
            buf[i * page_size] = 0

        # 랜덤 접근 시간 측정
        iterations = 1000000
        indices = [random.randint(0, num_pages - 1) * page_size
                   for _ in range(iterations)]

        start = time.perf_counter()
        for idx in indices:
            _ = buf[idx]
        end = time.perf_counter()

        avg_time_ns = (end - start) / iterations * 1e9
        results.append((num_pages, avg_time_ns))

        buf.close()

    print("=== TLB Performance Test ===")
    print("Pages\tAvg Access Time (ns)")
    print("-" * 30)
    for pages, time_ns in results:
        marker = " <-- Possible TLB thrashing" if pages > 64 and time_ns > results[0][1] * 1.5 else ""
        print(f"{pages}\t{time_ns:.2f}{marker}")

if __name__ == "__main__":
    # 실제 실행 시 Linux에서만 작동
    try:
        measure_tlb_performance()
    except Exception as e:
        print(f"Test failed: {e}")
        print("Note: This test works best on Linux systems")
```

### 4.6 Huge Pages

TLB 효율성을 높이기 위해 큰 페이지를 사용합니다.

```bash
# Linux에서 Huge Pages 설정

# 현재 설정 확인
cat /proc/meminfo | grep -i huge

# Huge Pages 할당 (root 권한)
echo 1024 > /proc/sys/vm/nr_hugepages

# Transparent Huge Pages (THP) 상태
cat /sys/kernel/mm/transparent_hugepage/enabled
```

```c
// Huge Page 할당 (C)
#include <sys/mman.h>
#include <stdio.h>

#define HUGE_PAGE_SIZE (2 * 1024 * 1024)  // 2MB

int main() {
    // 2MB Huge Page 할당
    void *ptr = mmap(NULL, HUGE_PAGE_SIZE,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
                     -1, 0);

    if (ptr == MAP_FAILED) {
        perror("mmap with MAP_HUGETLB failed");
        return 1;
    }

    printf("Huge page allocated at: %p\n", ptr);

    // 사용...

    munmap(ptr, HUGE_PAGE_SIZE);
    return 0;
}
```

---

## 5. 메모리 단편화와 해결법

### 5.1 외부 단편화 (External Fragmentation)

할당된 메모리 블록들 사이에 작은 빈 공간들이 생기는 현상입니다.

```
External Fragmentation:

| Used | Free | Used |  Free  | Used | Free |
|  20  |  10  |  30  |   15   |  25  |  5   |

총 Free: 30 bytes
하지만 연속 25 bytes 할당 불가!
```

#### 해결 방법

1. **압축 (Compaction)**: 사용 중인 블록을 한쪽으로 모음
2. **페이징**: 비연속 할당으로 외부 단편화 제거
3. **버디 시스템 (Buddy System)**: 2의 거듭제곱 크기로만 할당

### 5.2 내부 단편화 (Internal Fragmentation)

할당된 블록 내부에서 낭비되는 공간입니다.

```
Internal Fragmentation:

요청: 17 bytes
할당: 32 bytes (페이지 또는 청크 단위)
낭비: 15 bytes

┌────────────────────────────────────┐
│▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░│
│   Used (17B)    │  Wasted (15B)   │
└────────────────────────────────────┘
```

### 5.3 버디 시스템 (Buddy System)

```
Buddy System 예시 (초기 128 bytes):

1. 32 bytes 요청 (128 → 64 → 32)
   [████████████████|                |                |                ]
         32B              32B              64B

2. 16 bytes 요청 (64 → 32 → 16)
   [████████████████|▓▓▓▓▓▓▓▓|        |                |                ]
         32B          16B     16B           32B              ?

3. 32 bytes 해제
   [                |▓▓▓▓▓▓▓▓|        |                |                ]
         32B          16B     16B           32B

4. 16 bytes 해제 → 버디 병합
   [                |                |                ]
         32B              32B              64B
   →
   [                                  |                ]
              64B                           64B
   →
   [                                                  ]
                          128B
```

```python
class BuddyAllocator:
    """
    버디 시스템 메모리 할당자
    """
    def __init__(self, total_size: int):
        # total_size는 2의 거듭제곱이어야 함
        self.total_size = total_size
        self.min_block = 16  # 최소 블록 크기
        self.max_order = self._log2(total_size // self.min_block)

        # 각 order별 free list
        self.free_lists = [[] for _ in range(self.max_order + 1)]
        self.free_lists[self.max_order].append(0)  # 초기 전체 블록

        # 할당 상태 추적
        self.allocated = {}

    def _log2(self, n: int) -> int:
        order = 0
        while (1 << order) < n:
            order += 1
        return order

    def _get_order(self, size: int) -> int:
        """요청 크기에 맞는 order 계산"""
        size = max(size, self.min_block)
        order = 0
        while (self.min_block << order) < size:
            order += 1
        return order

    def _block_size(self, order: int) -> int:
        return self.min_block << order

    def _buddy_address(self, addr: int, order: int) -> int:
        """버디 주소 계산"""
        return addr ^ self._block_size(order)

    def allocate(self, size: int) -> int:
        """메모리 할당"""
        order = self._get_order(size)

        if order > self.max_order:
            raise MemoryError(f"Requested size {size} too large")

        # 해당 order 또는 더 큰 블록 찾기
        for o in range(order, self.max_order + 1):
            if self.free_lists[o]:
                addr = self.free_lists[o].pop()

                # 필요하면 분할
                while o > order:
                    o -= 1
                    buddy = addr + self._block_size(o)
                    self.free_lists[o].append(buddy)

                self.allocated[addr] = order
                return addr

        raise MemoryError("Out of memory")

    def free(self, addr: int):
        """메모리 해제"""
        if addr not in self.allocated:
            raise ValueError(f"Address {addr} not allocated")

        order = self.allocated.pop(addr)

        # 버디 병합 시도
        while order < self.max_order:
            buddy = self._buddy_address(addr, order)

            if buddy in self.free_lists[order]:
                self.free_lists[order].remove(buddy)
                addr = min(addr, buddy)
                order += 1
            else:
                break

        self.free_lists[order].append(addr)

    def status(self) -> str:
        """현재 상태 출력"""
        lines = ["Buddy Allocator Status:"]
        for o in range(self.max_order + 1):
            size = self._block_size(o)
            blocks = self.free_lists[o]
            if blocks:
                lines.append(f"  Order {o} ({size}B): {blocks}")
        lines.append(f"  Allocated: {self.allocated}")
        return "\n".join(lines)


# 테스트
print("=== Buddy System Demo ===\n")
allocator = BuddyAllocator(256)
print(allocator.status())
print()

# 할당
addrs = []
for size in [32, 16, 64, 16]:
    addr = allocator.allocate(size)
    addrs.append(addr)
    print(f"Allocated {size}B at address {addr}")

print()
print(allocator.status())
print()

# 해제 및 병합
for addr in addrs:
    print(f"Freeing address {addr}")
    allocator.free(addr)
    print(allocator.status())
    print()
```

### 5.4 Slab Allocator

커널 객체 할당을 위한 캐시 기반 할당자입니다.

```
Slab Allocator 구조:

Cache (예: task_struct용)
├── Slab 1 (Full)
│   ├── Object 1 [Used]
│   ├── Object 2 [Used]
│   └── Object 3 [Used]
├── Slab 2 (Partial)
│   ├── Object 1 [Used]
│   ├── Object 2 [Free]
│   └── Object 3 [Used]
└── Slab 3 (Empty)
    ├── Object 1 [Free]
    ├── Object 2 [Free]
    └── Object 3 [Free]

장점:
- 내부 단편화 최소화 (객체 크기에 맞춤)
- 빠른 할당/해제 (사전 초기화된 객체 재사용)
- 캐시 효율성 (같은 타입 객체 연속 배치)
```

```bash
# Linux에서 Slab 정보 확인
cat /proc/slabinfo

# 또는 slabtop 명령어 (실시간)
slabtop -s c  # 캐시 크기순 정렬
```

### 5.5 메모리 풀 (Memory Pool)

```c
// 간단한 메모리 풀 구현
#include <stdlib.h>
#include <stdint.h>
#include <string.h>

#define POOL_SIZE 1024
#define OBJECT_SIZE 64

typedef struct {
    uint8_t memory[POOL_SIZE];
    uint8_t bitmap[POOL_SIZE / OBJECT_SIZE / 8];
    size_t num_objects;
} memory_pool_t;

void pool_init(memory_pool_t *pool) {
    memset(pool->memory, 0, POOL_SIZE);
    memset(pool->bitmap, 0, sizeof(pool->bitmap));
    pool->num_objects = POOL_SIZE / OBJECT_SIZE;
}

void* pool_alloc(memory_pool_t *pool) {
    for (size_t i = 0; i < pool->num_objects; i++) {
        size_t byte_idx = i / 8;
        size_t bit_idx = i % 8;

        if (!(pool->bitmap[byte_idx] & (1 << bit_idx))) {
            pool->bitmap[byte_idx] |= (1 << bit_idx);
            return &pool->memory[i * OBJECT_SIZE];
        }
    }
    return NULL;  // Pool exhausted
}

void pool_free(memory_pool_t *pool, void *ptr) {
    size_t offset = (uint8_t*)ptr - pool->memory;
    size_t idx = offset / OBJECT_SIZE;
    size_t byte_idx = idx / 8;
    size_t bit_idx = idx % 8;

    pool->bitmap[byte_idx] &= ~(1 << bit_idx);
}
```

### 5.6 Python에서의 메모리 관리

```python
import sys
import gc

def memory_management_demo():
    """Python 메모리 관리 시연"""

    # 1. 객체 크기 확인
    print("=== Object Sizes ===")
    objects = [
        (42, "int"),
        (3.14, "float"),
        ("hello", "str"),
        ([1, 2, 3], "list"),
        ({'a': 1}, "dict"),
    ]

    for obj, name in objects:
        print(f"{name}: {sys.getsizeof(obj)} bytes")

    print()

    # 2. 레퍼런스 카운팅
    print("=== Reference Counting ===")
    a = [1, 2, 3]
    print(f"Initial refcount: {sys.getrefcount(a) - 1}")  # -1 for getrefcount arg

    b = a  # 참조 증가
    print(f"After b = a: {sys.getrefcount(a) - 1}")

    del b  # 참조 감소
    print(f"After del b: {sys.getrefcount(a) - 1}")

    print()

    # 3. 가비지 컬렉션
    print("=== Garbage Collection ===")
    print(f"GC enabled: {gc.isenabled()}")
    print(f"GC thresholds: {gc.get_threshold()}")
    print(f"GC counts: {gc.get_count()}")

    # 순환 참조 생성
    class Node:
        def __init__(self):
            self.ref = None

    a = Node()
    b = Node()
    a.ref = b
    b.ref = a  # 순환 참조

    del a, b  # 레퍼런스 카운팅으로 해제 불가

    # 강제 GC 실행
    collected = gc.collect()
    print(f"Objects collected: {collected}")

if __name__ == "__main__":
    memory_management_demo()
```

---

## 7. 참고 자료

### 공식 문서 및 교재

- [OSTEP - Virtualization (Memory)](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [Intel 64 and IA-32 Architectures Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Linux Kernel Memory Management](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)

### 추가 학습 자료

- [What Every Programmer Should Know About Memory - Ulrich Drepper](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [Understanding the Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/)

### 도구

- `/proc/[pid]/maps` - 프로세스 메모리 맵
- `/proc/meminfo` - 시스템 메모리 정보
- `vmstat`, `free`, `top` - 메모리 모니터링
- `perf` - TLB 미스 등 성능 분석

---

> **문서 작성일**: 2024년 12월
>
> **참고**: 이 문서는 OSTEP, Linux 커널 문서, Intel 아키텍처 매뉴얼을 기반으로 작성되었습니다.

# 페이지 교체 알고리즘 (Page Replacement Algorithms)

## 개요

물리 메모리가 가득 찬 상태에서 새 페이지를 로드해야 할 때, **어떤 페이지를 내보낼지(교체할지)** 결정하는 알고리즘입니다. 좋은 페이지 교체 알고리즘은 Page Fault를 최소화합니다.

```
메모리 가득 참 + 새 페이지 필요
              │
              ▼
      ┌───────────────────┐
      │  어떤 페이지를    │
      │  교체할 것인가?   │
      └───────────────────┘
              │
              ▼
    페이지 교체 알고리즘 적용
              │
              ▼
    희생자(Victim) 페이지 선정
              │
              ▼
    ┌─────────────────────┐
    │ Dirty?              │
    │ Yes → 스왑에 기록   │
    │ No → 그냥 버림      │
    └─────────────────────┘
              │
              ▼
      새 페이지 로드
```

## 핵심 개념

### 1. 최적 알고리즘 (Optimal, OPT, MIN)

**가장 오랫동안 사용되지 않을** 페이지를 교체합니다.

#### 동작 원리

```
미래에 가장 늦게 사용될 페이지를 교체

참조 문자열: 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5
프레임 수: 3

시간   참조   메모리 상태        Page Fault
─────────────────────────────────────────────
 1      1    [1, -, -]          ✓ (빈 프레임)
 2      2    [1, 2, -]          ✓ (빈 프레임)
 3      3    [1, 2, 3]          ✓ (빈 프레임)
 4      4    [1, 2, 4]          ✓ (3 교체: 가장 늦게 사용)
 5      1    [1, 2, 4]          (hit)
 6      2    [1, 2, 4]          (hit)
 7      5    [1, 2, 5]          ✓ (4 교체: 가장 늦게 사용)
 8      1    [1, 2, 5]          (hit)
 9      2    [1, 2, 5]          (hit)
10      3    [3, 2, 5]          ✓ (1 교체: 가장 늦게 사용)
11      4    [3, 4, 5]          ✓ (2 교체: 가장 늦게 사용)
12      5    [3, 4, 5]          (hit)

총 Page Fault: 7
```

#### 특징

| 항목 | 설명 |
|------|------|
| 장점 | Page Fault 최소 (이론적 최적) |
| 단점 | **미래를 알 수 없어 구현 불가** |
| 용도 | 다른 알고리즘 성능 비교 기준 |

### 2. FIFO (First-In First-Out)

**가장 먼저 들어온** 페이지를 교체합니다.

#### 동작 원리

```
참조 문자열: 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5
프레임 수: 3

시간   참조   메모리 상태 (FIFO 순서)   Page Fault
─────────────────────────────────────────────────
 1      1    [1, -, -]                 ✓
 2      2    [1, 2, -]                 ✓
 3      3    [1, 2, 3]                 ✓
 4      4    [4, 2, 3]                 ✓ (1 교체: 가장 오래됨)
 5      1    [4, 1, 3]                 ✓ (2 교체)
 6      2    [4, 1, 2]                 ✓ (3 교체)
 7      5    [5, 1, 2]                 ✓ (4 교체)
 8      1    [5, 1, 2]                 (hit)
 9      2    [5, 1, 2]                 (hit)
10      3    [5, 3, 2]                 ✓ (1 교체)
11      4    [5, 3, 4]                 ✓ (2 교체)
12      5    [5, 3, 4]                 (hit)

총 Page Fault: 9
```

#### 구현

```python
from collections import deque

def fifo_page_replacement(pages, frame_count):
    frames = deque(maxlen=frame_count)
    page_faults = 0

    for page in pages:
        if page not in frames:
            page_faults += 1
            if len(frames) == frame_count:
                frames.popleft()  # 가장 오래된 페이지 제거
            frames.append(page)

    return page_faults
```

#### Belady's Anomaly (벨라디의 역설)

FIFO는 **프레임 수가 늘어도 Page Fault가 증가**할 수 있습니다.

```
참조 문자열: 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5

프레임 3개: Page Fault = 9
프레임 4개: Page Fault = 10  ← 더 많음!

프레임이 늘었는데 Page Fault가 증가하는 비정상적 현상
```

#### 특징

| 항목 | 설명 |
|------|------|
| 장점 | 구현이 매우 간단 |
| 단점 | 최근 사용 여부 고려 안 함, Belady's Anomaly |
| 시간 복잡도 | O(1) |

### 3. LRU (Least Recently Used)

**가장 오랫동안 사용되지 않은** 페이지를 교체합니다.

#### 동작 원리

```
시간적 지역성(Temporal Locality) 활용:
"최근에 사용된 페이지는 곧 다시 사용될 가능성이 높다"

참조 문자열: 1, 2, 3, 4, 1, 2, 5, 1, 2, 3, 4, 5
프레임 수: 3

시간   참조   메모리 상태        최근 사용 순서    Page Fault
───────────────────────────────────────────────────────────
 1      1    [1, -, -]          1                 ✓
 2      2    [1, 2, -]          2, 1              ✓
 3      3    [1, 2, 3]          3, 2, 1           ✓
 4      4    [4, 2, 3]          4, 3, 2           ✓ (1 교체: LRU)
 5      1    [4, 1, 3]          1, 4, 3           ✓ (2 교체: LRU)
 6      2    [4, 1, 2]          2, 1, 4           ✓ (3 교체: LRU)
 7      5    [5, 1, 2]          5, 2, 1           ✓ (4 교체: LRU)
 8      1    [5, 1, 2]          1, 5, 2           (hit)
 9      2    [5, 1, 2]          2, 1, 5           (hit)
10      3    [3, 1, 2]          3, 2, 1           ✓ (5 교체: LRU)
11      4    [3, 4, 2]          4, 3, 2           ✓ (1 교체: LRU)
12      5    [3, 4, 5]          5, 4, 3           ✓ (2 교체: LRU)

총 Page Fault: 10
```

#### 구현 방법

**방법 1: 카운터 (Counter)**

```c
struct PageTableEntry {
    int frame_number;
    int counter;  // 마지막 접근 시각
};

// 페이지 접근 시
page_table[page].counter = current_time++;

// 교체 시
victim = min(all_pages, by=counter);  // 가장 작은 카운터 값
```

**방법 2: 스택**

```
접근: 1 → 2 → 3 → 4 → 1 → 2

Stack 변화 (top이 MRU):
[1]
[2, 1]
[3, 2, 1]
[4, 3, 2, 1]
[1, 4, 3, 2]   ← 1 접근 시 1을 top으로 이동
[2, 1, 4, 3]   ← 2 접근 시 2를 top으로 이동

LRU 페이지 = Stack 바닥
```

**방법 3: 이중 연결 리스트 + 해시맵 (실무)**

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cache = OrderedDict()
        self.capacity = capacity

    def access(self, page):
        if page in self.cache:
            self.cache.move_to_end(page)  # MRU로 이동
            return False  # hit
        else:
            if len(self.cache) >= self.capacity:
                self.cache.popitem(last=False)  # LRU 제거
            self.cache[page] = True
            return True  # page fault
```

#### 특징

| 항목 | 설명 |
|------|------|
| 장점 | OPT에 근접한 성능, Belady's Anomaly 없음 |
| 단점 | 하드웨어 지원 필요 (비용), 순수 소프트웨어 구현은 느림 |
| 시간 복잡도 | O(1) (해시맵 + 이중 연결 리스트) |

### 4. LRU 근사 알고리즘

LRU의 정확한 구현은 비용이 크므로, 근사 방식을 사용합니다.

#### 4.1 Reference Bit (참조 비트)

```
Page Table Entry에 Reference Bit 추가:
┌──────────────────────────────────────────┐
│ Valid │ R │ Dirty │ Frame Number        │
└──────────────────────────────────────────┘
        │
        └── R (Reference Bit)
            - 페이지 접근 시 하드웨어가 1로 설정
            - 주기적으로 OS가 0으로 초기화

교체 시:
- R = 0인 페이지 교체 (최근에 사용 안 됨)
- 모두 R = 1이면 모두 0으로 리셋 후 재시도
```

#### 4.2 Second Chance (Clock) 알고리즘

FIFO + Reference Bit 조합입니다.

```
페이지들이 원형 큐에 배치
시계 바늘(clock hand)이 순환

    ┌─── Page 1 (R=1) ◄── clock hand
    │
    │   Page 2 (R=0)
    │
    │   Page 3 (R=1)
    │
    └── Page 4 (R=0)

교체 과정:
1. clock hand가 가리키는 페이지 확인
2. R = 0이면 → 이 페이지 교체
3. R = 1이면 → R을 0으로 바꾸고, 다음 페이지로 이동
4. 반복

"두 번째 기회": R=1인 페이지는 한 바퀴 더 생존
```

```python
def clock_page_replacement(pages, frame_count):
    frames = [None] * frame_count
    reference_bits = [0] * frame_count
    clock_hand = 0
    page_faults = 0

    for page in pages:
        # 이미 메모리에 있으면 hit
        if page in frames:
            idx = frames.index(page)
            reference_bits[idx] = 1
            continue

        # Page Fault
        page_faults += 1

        # 희생자 찾기
        while reference_bits[clock_hand] == 1:
            reference_bits[clock_hand] = 0  # 두 번째 기회 부여
            clock_hand = (clock_hand + 1) % frame_count

        # 교체
        frames[clock_hand] = page
        reference_bits[clock_hand] = 1
        clock_hand = (clock_hand + 1) % frame_count

    return page_faults
```

#### 4.3 Enhanced Second Chance (개선된 클록)

Reference Bit + Dirty Bit 조합입니다.

```
(R, D) 페어로 우선순위 결정:

┌────────┬────────┬─────────────────────────────┐
│ R Bit  │ D Bit  │ 의미                        │
├────────┼────────┼─────────────────────────────┤
│   0    │   0    │ 최근 미사용 + 수정 안 됨   │ ← 최우선 교체
│   0    │   1    │ 최근 미사용 + 수정됨       │
│   1    │   0    │ 최근 사용 + 수정 안 됨     │
│   1    │   1    │ 최근 사용 + 수정됨         │ ← 최후 교체
└────────┴────────┴─────────────────────────────┘

교체 순서: (0,0) → (0,1) → (1,0) → (1,1)

이유:
- Dirty Page 교체 시 디스크 쓰기 필요 (비용 큼)
- Clean Page 교체가 더 효율적
```

### 5. LFU (Least Frequently Used)

**사용 빈도가 가장 낮은** 페이지를 교체합니다.

```
참조 문자열: 1, 1, 2, 1, 3, 3, 4
프레임 수: 3

시간   참조   메모리 상태        사용 빈도           Page Fault
─────────────────────────────────────────────────────────────
 1      1    [1, -, -]          1:1                 ✓
 2      1    [1, -, -]          1:2                 (hit)
 3      2    [1, 2, -]          1:2, 2:1            ✓
 4      1    [1, 2, -]          1:3, 2:1            (hit)
 5      3    [1, 2, 3]          1:3, 2:1, 3:1       ✓
 6      3    [1, 2, 3]          1:3, 2:1, 3:2       (hit)
 7      4    [1, 4, 3]          1:3, 4:1, 3:2       ✓ (2 교체: 빈도 최소)

동률 시: FIFO 또는 LRU로 결정
```

#### 특징

| 항목 | 설명 |
|------|------|
| 장점 | 자주 사용되는 페이지 보호 |
| 단점 | 초기에 많이 사용된 페이지가 오래 남음, 구현 복잡 |

### 6. MFU (Most Frequently Used)

**사용 빈도가 가장 높은** 페이지를 교체합니다.

```
아이디어: 많이 사용된 페이지는 이미 충분히 사용됨
         → 앞으로는 덜 사용될 것

실제로는 LFU보다 성능이 좋지 않아 잘 사용되지 않음
```

### 7. 알고리즘 비교

```
참조 문자열: 7, 0, 1, 2, 0, 3, 0, 4, 2, 3, 0, 3, 2, 1, 2, 0, 1, 7, 0, 1
프레임 수: 3

┌──────────┬──────────────┬──────────────┐
│ 알고리즘 │ Page Faults  │ 특징         │
├──────────┼──────────────┼──────────────┤
│ Optimal  │     9        │ 이론적 최적  │
│ LRU      │    12        │ 실용적 최적  │
│ FIFO     │    15        │ 단순함       │
│ Clock    │    13        │ LRU 근사     │
└──────────┴──────────────┴──────────────┘
```

### 8. 프레임 할당 (Frame Allocation)

여러 프로세스에게 프레임을 어떻게 분배할지 결정합니다.

#### 8.1 할당 방식

```
균등 할당 (Equal Allocation):
─────────────────────────────
총 100 프레임, 5개 프로세스
→ 각 프로세스에 20 프레임씩

문제: 프로세스 크기가 다른데 동일 할당?


비례 할당 (Proportional Allocation):
─────────────────────────────
프로세스 크기에 비례하여 할당

a_i = (s_i / Σs_j) × m

s_i: 프로세스 i의 가상 메모리 크기
m: 총 프레임 수

예: P1(10KB), P2(127KB), 총 62 프레임
P1: (10/137) × 62 ≈ 5 프레임
P2: (127/137) × 62 ≈ 57 프레임
```

#### 8.2 전역 vs 지역 교체

```
전역 교체 (Global Replacement):
─────────────────────────────
모든 프로세스의 프레임 중에서 희생자 선정

장점: 전체 시스템 성능 최적화
단점: 한 프로세스가 다른 프로세스 영향


지역 교체 (Local Replacement):
─────────────────────────────
자기 프레임 내에서만 희생자 선정

장점: 프로세스 간 간섭 없음
단점: 사용 안 하는 프레임 낭비 가능
```

## 실무 적용

### Linux의 페이지 교체

Linux는 **LRU 근사 알고리즘**을 사용합니다.

```
Active List: 최근 사용된 페이지
Inactive List: 오래된 페이지

동작:
1. 새 페이지 → Inactive List 끝에 추가
2. Inactive 페이지 접근 → Active List로 이동
3. 메모리 부족 → Inactive List 앞에서 교체
4. Active List 가득 참 → 오래된 것 Inactive로 이동
```

```bash
# Linux 메모리 상태 확인
cat /proc/meminfo

# 페이지 교체 통계
vmstat 1

# 특정 프로세스의 메모리 매핑
cat /proc/<pid>/smaps
```

### 스래싱 (Thrashing)

Page Fault가 너무 많이 발생하여 실제 작업보다 페이지 교체에 더 많은 시간을 쓰는 현상입니다.

```
Working Set > 할당된 프레임
        │
        ▼
   Page Fault 급증
        │
        ▼
  I/O 대기 시간 증가
        │
        ▼
   CPU 사용률 저하
        │
        ▼
OS가 더 많은 프로세스 실행 시도 (CPU 활용을 위해)
        │
        ▼
   상황 악화 (악순환)

해결:
- Working Set 모델 사용
- 프로세스 수 제한
- 메모리 추가
- 스왑 공간 확대
```

## 참고 자료

- Operating System Concepts (Silberschatz) - Chapter 10: Virtual Memory
- [OSTEP - Swapping: Policies](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys-policy.pdf)
- [Linux Page Frame Reclaiming](https://www.kernel.org/doc/html/latest/admin-guide/mm/concepts.html)
- Modern Operating Systems (Tanenbaum) - Memory Management

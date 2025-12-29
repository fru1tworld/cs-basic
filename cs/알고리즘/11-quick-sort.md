# 퀵 정렬 (Quick Sort)

## 개요

퀵 정렬은 **분할 정복(Divide and Conquer)** 기반의 정렬 알고리즘으로, **피봇(Pivot)**을 기준으로 배열을 분할하여 정렬한다. 평균적으로 가장 빠른 범용 정렬 알고리즘 중 하나이다.

**핵심 특성**:
- 시간 복잡도: **O(n log n)** 평균, O(n²) 최악
- 공간 복잡도: O(log n) ~ O(n) - 재귀 스택
- **제자리 정렬(In-place)**
- **불안정 정렬(Unstable)**
- 캐시 효율이 뛰어남

## 핵심 개념

### 1. 알고리즘 동작 원리

```
원본: [3, 7, 8, 5, 2, 1, 9, 5, 4]
피봇 = 4 (마지막 원소)

Partition 후: [3, 2, 1, 4, 7, 8, 9, 5, 5]
                        ↑
                    피봇 위치

왼쪽 [3, 2, 1] 재귀 → [1, 2, 3]
오른쪽 [7, 8, 9, 5, 5] 재귀 → [5, 5, 7, 8, 9]

결과: [1, 2, 3, 4, 5, 5, 7, 8, 9]
```

### 2. Lomuto Partition Scheme

```python
def quick_sort_lomuto(arr, low, high):
    """Lomuto 파티션을 사용한 퀵 정렬"""
    if low < high:
        pivot_idx = partition_lomuto(arr, low, high)
        quick_sort_lomuto(arr, low, pivot_idx - 1)
        quick_sort_lomuto(arr, pivot_idx + 1, high)

def partition_lomuto(arr, low, high):
    """Lomuto 파티션: 마지막 원소를 피봇으로 사용"""
    pivot = arr[high]
    i = low - 1  # 피봇보다 작은 원소들의 경계

    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
```

**Lomuto의 동작 과정**:
```
[3, 7, 8, 5, 2, 1, 9, 5, 4], pivot=4
 ↑i                      ↑pivot

j=0: 3<=4, i=0, swap(0,0) → [3, 7, 8, 5, 2, 1, 9, 5, 4]
j=1: 7>4, skip
j=2: 8>4, skip
j=3: 5>4, skip
j=4: 2<=4, i=1, swap(1,4) → [3, 2, 8, 5, 7, 1, 9, 5, 4]
j=5: 1<=4, i=2, swap(2,5) → [3, 2, 1, 5, 7, 8, 9, 5, 4]
...
최종 pivot 이동: swap(i+1, high)
```

### 3. Hoare Partition Scheme

```python
def quick_sort_hoare(arr, low, high):
    """Hoare 파티션을 사용한 퀵 정렬"""
    if low < high:
        pivot_idx = partition_hoare(arr, low, high)
        quick_sort_hoare(arr, low, pivot_idx)      # 주의: pivot_idx 포함
        quick_sort_hoare(arr, pivot_idx + 1, high)

def partition_hoare(arr, low, high):
    """Hoare 파티션: 첫 번째 원소를 피봇으로 사용"""
    pivot = arr[low]
    i = low - 1
    j = high + 1

    while True:
        i += 1
        while arr[i] < pivot:
            i += 1

        j -= 1
        while arr[j] > pivot:
            j -= 1

        if i >= j:
            return j

        arr[i], arr[j] = arr[j], arr[i]
```

**Lomuto vs Hoare 비교**:

| 특성 | Lomuto | Hoare |
|------|--------|-------|
| 피봇 위치 | 마지막 | 첫 번째 (보통) |
| 스왑 횟수 | 더 많음 | 평균 3배 적음 |
| 이해 난이도 | 쉬움 | 어려움 |
| 피봇 반환 | 정확한 위치 | 분할점 |
| 중복 처리 | 비효율적 | 효율적 |

### 4. 피봇 선택 전략

#### 4.1 고정 피봇 (문제점)

```python
# 최악의 경우: 이미 정렬된 배열
[1, 2, 3, 4, 5, 6, 7, 8, 9]
피봇 = 9 (마지막)
→ 분할: [1,2,3,4,5,6,7,8] | [9] | []
→ 불균형 분할 → O(n²)
```

#### 4.2 Random Pivot

```python
import random

def partition_random(arr, low, high):
    """랜덤 피봇 선택"""
    rand_idx = random.randint(low, high)
    arr[rand_idx], arr[high] = arr[high], arr[rand_idx]
    return partition_lomuto(arr, low, high)
```

**장점**: 최악의 경우를 확률적으로 회피
**기대 시간 복잡도**: O(n log n)

#### 4.3 Median-of-Three

```python
def median_of_three(arr, low, high):
    """세 원소의 중앙값을 피봇으로 선택"""
    mid = (low + high) // 2

    # arr[low], arr[mid], arr[high] 중 중앙값 선택
    if arr[low] > arr[mid]:
        arr[low], arr[mid] = arr[mid], arr[low]
    if arr[low] > arr[high]:
        arr[low], arr[high] = arr[high], arr[low]
    if arr[mid] > arr[high]:
        arr[mid], arr[high] = arr[high], arr[mid]

    # 중앙값을 high-1 위치로 이동
    arr[mid], arr[high - 1] = arr[high - 1], arr[mid]
    return arr[high - 1]
```

**장점**: 정렬된/역순 배열에서 좋은 피봇 선택

#### 4.4 Ninther (Median-of-Medians-of-Three)

```python
def ninther(arr, low, high):
    """9개 원소에서 피봇 선택 (대용량 배열용)"""
    n = high - low + 1
    if n < 9:
        return median_of_three(arr, low, high)

    step = n // 8
    m1 = median_of_three_idx(arr, low, low + step, low + 2*step)
    m2 = median_of_three_idx(arr, low + 3*step, low + 4*step, low + 5*step)
    m3 = median_of_three_idx(arr, low + 6*step, low + 7*step, high)

    return median_of_three_idx(arr, m1, m2, m3)
```

### 5. 3-Way Partitioning (Dutch National Flag)

중복 원소가 많을 때 효율적:

```python
def quick_sort_3way(arr, low, high):
    """3-way 파티션 퀵 정렬"""
    if low >= high:
        return

    lt, gt = partition_3way(arr, low, high)
    quick_sort_3way(arr, low, lt - 1)
    quick_sort_3way(arr, gt + 1, high)

def partition_3way(arr, low, high):
    """Dutch National Flag 파티션
    결과: [< pivot] [== pivot] [> pivot]
    반환: lt (첫 번째 == pivot), gt (마지막 == pivot)
    """
    pivot = arr[low]
    lt = low      # arr[low..lt-1] < pivot
    gt = high     # arr[gt+1..high] > pivot
    i = low + 1   # arr[lt..i-1] == pivot

    while i <= gt:
        if arr[i] < pivot:
            arr[lt], arr[i] = arr[i], arr[lt]
            lt += 1
            i += 1
        elif arr[i] > pivot:
            arr[gt], arr[i] = arr[i], arr[gt]
            gt -= 1
        else:
            i += 1

    return lt, gt
```

**3-way의 장점**:
```
입력: [4, 4, 4, 4, 4, 4, 4]

일반 퀵 정렬: O(n²) - 매번 하나만 분리
3-way 퀵 정렬: O(n) - 모든 4를 한 번에 처리
```

### 6. 시간 복잡도 분석

#### 최악의 경우: O(n²)

```
T(n) = T(n-1) + T(0) + Θ(n)
     = T(n-1) + Θ(n)
     = Θ(n²)
```

불균형 분할이 계속될 때:
- 이미 정렬된 배열 + 마지막 피봇
- 모든 원소가 동일 + 일반 파티션

#### 최선의 경우: O(n log n)

```
T(n) = 2T(n/2) + Θ(n)
     = Θ(n log n)
```

#### 평균의 경우: O(n log n)

**직관적 설명**:
- 랜덤 피봇은 평균적으로 1:9 ~ 9:1 분할을 제공
- 심지어 1:9 분할도 O(n log n)

```
T(n) = T(n/10) + T(9n/10) + Θ(n)

재귀 깊이: log_{10/9}(n) ≈ 6.5 log n
각 레벨 비용: O(n)
총합: O(n log n)
```

**엄밀한 평균 분석**:
```
E[T(n)] = (1/n) × Σ(k=0 to n-1) [E[T(k)] + E[T(n-1-k)]] + Θ(n)
        = (2/n) × Σ(k=0 to n-1) E[T(k)] + Θ(n)
        = Θ(n log n)
```

### 7. 꼬리 재귀 최적화

스택 오버플로우 방지:

```python
def quick_sort_tail_optimized(arr, low, high):
    """꼬리 재귀 최적화된 퀵 정렬"""
    while low < high:
        pivot_idx = partition_lomuto(arr, low, high)

        # 작은 쪽을 먼저 재귀, 큰 쪽은 반복
        if pivot_idx - low < high - pivot_idx:
            quick_sort_tail_optimized(arr, low, pivot_idx - 1)
            low = pivot_idx + 1
        else:
            quick_sort_tail_optimized(arr, pivot_idx + 1, high)
            high = pivot_idx - 1
```

**공간 복잡도**: O(log n) 보장 (최악의 경우에도)

## 실무 적용

### 1. Java의 Dual-Pivot Quick Sort

Java 7+에서 Arrays.sort() 구현:

```java
// 의사 코드
void dualPivotQuickSort(int[] a, int low, int high) {
    // 두 개의 피봇 사용
    int pivot1 = a[low];
    int pivot2 = a[high];

    // 3개 영역으로 분할
    // [< pivot1] [pivot1 <= x <= pivot2] [> pivot2]

    // 재귀
    dualPivotQuickSort(a, low, lt - 1);
    dualPivotQuickSort(a, lt + 1, gt - 1);
    dualPivotQuickSort(a, gt + 1, high);
}
```

### 2. Introsort (Introspective Sort)

재귀 깊이 제한으로 O(n²) 방지:

```python
import math

def introsort(arr):
    """Introsort: Quick Sort + Heap Sort + Insertion Sort"""
    max_depth = 2 * math.floor(math.log2(len(arr)))
    introsort_helper(arr, 0, len(arr) - 1, max_depth)

def introsort_helper(arr, low, high, depth_limit):
    size = high - low + 1

    if size < 16:
        # 작은 배열: 삽입 정렬
        insertion_sort(arr, low, high)
        return

    if depth_limit == 0:
        # 깊이 초과: 힙 정렬
        heap_sort(arr, low, high)
        return

    # 일반적인 경우: 퀵 정렬
    pivot = partition_median_of_three(arr, low, high)
    introsort_helper(arr, low, pivot - 1, depth_limit - 1)
    introsort_helper(arr, pivot + 1, high, depth_limit - 1)
```

**C++ std::sort**와 **Java Arrays.sort(Object[])**에서 사용.

### 3. pdqsort (Pattern-Defeating Quick Sort)

현대적인 퀵 정렬 구현:

```
특징:
1. 랜덤화 없이도 O(n log n) 보장
2. 정렬된/역순 입력에서 O(n) 달성
3. 중복 원소 효율적 처리
4. 캐시 친화적

Rust의 기본 정렬 알고리즘
```

### 4. 병렬 퀵 정렬

```python
from concurrent.futures import ThreadPoolExecutor

def parallel_quick_sort(arr, low=0, high=None, depth=0, max_depth=4):
    """병렬 퀵 정렬"""
    if high is None:
        high = len(arr) - 1

    if low >= high:
        return

    pivot_idx = partition_hoare(arr, low, high)

    if depth < max_depth:
        with ThreadPoolExecutor(max_workers=2) as executor:
            f1 = executor.submit(parallel_quick_sort, arr, low, pivot_idx, depth+1, max_depth)
            f2 = executor.submit(parallel_quick_sort, arr, pivot_idx+1, high, depth+1, max_depth)
            f1.result()
            f2.result()
    else:
        quick_sort_hoare(arr, low, pivot_idx)
        quick_sort_hoare(arr, pivot_idx + 1, high)
```

### 5. k번째 원소 찾기 (Quick Select)

```python
def quick_select(arr, k):
    """k번째로 작은 원소 찾기 - 평균 O(n)"""
    low, high = 0, len(arr) - 1

    while low <= high:
        pivot_idx = partition_random(arr, low, high)

        if pivot_idx == k:
            return arr[k]
        elif pivot_idx > k:
            high = pivot_idx - 1
        else:
            low = pivot_idx + 1

    return arr[k]
```

## 참고 자료

- Introduction to Algorithms (CLRS) - Chapter 7
- The Art of Computer Programming, Vol. 3 (Knuth)
- Engineering a Sort Function (Bentley & McIlroy)
- [Dual-Pivot Quick Sort](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/DualPivotQuicksort.java)
- [pdqsort: Pattern-defeating quicksort](https://github.com/orlp/pdqsort)

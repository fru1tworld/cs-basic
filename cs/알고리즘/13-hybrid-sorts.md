# 하이브리드 정렬 (Hybrid Sorts)

## 개요

하이브리드 정렬은 여러 정렬 알고리즘의 장점을 결합하여 실제 환경에서 최적의 성능을 달성하는 정렬 알고리즘이다. 대부분의 프로그래밍 언어 표준 라이브러리는 하이브리드 정렬을 사용한다.

**핵심 아이디어**:
- 큰 배열: O(n log n) 알고리즘 (Quick Sort, Merge Sort)
- 작은 배열: 삽입 정렬 (오버헤드 적음)
- 최악 케이스 방지: Heap Sort로 전환

## 핵심 개념

### 1. Introsort (Introspective Sort)

Quick Sort + Heap Sort + Insertion Sort

```python
import math

def introsort(arr):
    """C++ std::sort의 기반 알고리즘"""
    max_depth = 2 * math.floor(math.log2(len(arr)))
    _introsort(arr, 0, len(arr) - 1, max_depth)
    return arr

def _introsort(arr, low, high, depth_limit):
    size = high - low + 1

    # 작은 배열: 삽입 정렬
    if size < 16:
        _insertion_sort(arr, low, high)
        return

    # 재귀 깊이 초과: 힙 정렬
    if depth_limit == 0:
        _heap_sort(arr, low, high)
        return

    # 일반적인 경우: 퀵 정렬
    pivot = _median_of_three_partition(arr, low, high)
    _introsort(arr, low, pivot - 1, depth_limit - 1)
    _introsort(arr, pivot + 1, high, depth_limit - 1)

def _insertion_sort(arr, low, high):
    for i in range(low + 1, high + 1):
        key = arr[i]
        j = i - 1
        while j >= low and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key

def _heap_sort(arr, low, high):
    """부분 배열에 대한 힙 정렬"""
    n = high - low + 1

    # Build max heap
    for i in range(n // 2 - 1, -1, -1):
        _heapify(arr, n, i, low)

    # Extract elements
    for i in range(n - 1, 0, -1):
        arr[low], arr[low + i] = arr[low + i], arr[low]
        _heapify(arr, i, 0, low)

def _heapify(arr, n, i, offset):
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2

    if left < n and arr[offset + left] > arr[offset + largest]:
        largest = left
    if right < n and arr[offset + right] > arr[offset + largest]:
        largest = right

    if largest != i:
        arr[offset + i], arr[offset + largest] = arr[offset + largest], arr[offset + i]
        _heapify(arr, n, largest, offset)

def _median_of_three_partition(arr, low, high):
    """Median-of-three 피봇 선택 + Lomuto 파티션"""
    mid = (low + high) // 2

    # 세 원소를 정렬
    if arr[low] > arr[mid]:
        arr[low], arr[mid] = arr[mid], arr[low]
    if arr[low] > arr[high]:
        arr[low], arr[high] = arr[high], arr[low]
    if arr[mid] > arr[high]:
        arr[mid], arr[high] = arr[high], arr[mid]

    # 중앙값을 high-1로 이동
    arr[mid], arr[high - 1] = arr[high - 1], arr[mid]
    pivot = arr[high - 1]

    # 파티션
    i = low
    j = high - 1

    while True:
        i += 1
        while arr[i] < pivot:
            i += 1
        j -= 1
        while arr[j] > pivot:
            j -= 1
        if i >= j:
            break
        arr[i], arr[j] = arr[j], arr[i]

    arr[i], arr[high - 1] = arr[high - 1], arr[i]
    return i
```

**Introsort 장점**:
- 평균 O(n log n): Quick Sort의 빠른 성능
- 최악 O(n log n): Heap Sort로 보장
- 작은 배열 최적화: Insertion Sort

**사용처**:
- C++ `std::sort`
- .NET `Array.Sort`
- Go `sort.Sort`

### 2. Timsort

Merge Sort + Insertion Sort (+ Galloping)

```python
MIN_MERGE = 32

def timsort(arr):
    """Python, Java의 기본 정렬 알고리즘"""
    n = len(arr)

    # 최소 run 길이 계산
    min_run = _calc_min_run(n)

    # 1단계: run 생성 (자연스러운 run 또는 삽입 정렬로)
    runs = []
    i = 0
    while i < n:
        run_start = i

        # 자연스러운 run 찾기
        if i + 1 < n:
            if arr[i] <= arr[i + 1]:
                # 오름차순 run
                while i + 1 < n and arr[i] <= arr[i + 1]:
                    i += 1
            else:
                # 내림차순 run → 뒤집기
                while i + 1 < n and arr[i] > arr[i + 1]:
                    i += 1
                _reverse(arr, run_start, i)
        i += 1

        # run이 min_run보다 작으면 확장 (삽입 정렬)
        run_end = min(run_start + min_run - 1, n - 1)
        if i <= run_end:
            _binary_insertion_sort(arr, run_start, run_end, i)
            i = run_end + 1

        runs.append((run_start, i - run_start))

    # 2단계: run 병합
    _merge_runs(arr, runs)

    return arr

def _calc_min_run(n):
    """최소 run 길이 계산 (32~64)"""
    r = 0
    while n >= MIN_MERGE:
        r |= n & 1
        n >>= 1
    return n + r

def _binary_insertion_sort(arr, low, high, start):
    """이진 삽입 정렬"""
    for i in range(start, high + 1):
        key = arr[i]

        # 이진 탐색으로 삽입 위치 찾기
        left, right = low, i
        while left < right:
            mid = (left + right) // 2
            if arr[mid] > key:
                right = mid
            else:
                left = mid + 1

        # 이동
        for j in range(i, left, -1):
            arr[j] = arr[j - 1]
        arr[left] = key

def _reverse(arr, start, end):
    """부분 배열 뒤집기"""
    while start < end:
        arr[start], arr[end] = arr[end], arr[start]
        start += 1
        end -= 1

def _merge_runs(arr, runs):
    """run들을 병합 (스택 기반)"""
    stack = []

    for run_start, run_len in runs:
        stack.append((run_start, run_len))

        # 병합 규칙 확인 및 적용
        while len(stack) > 1:
            if len(stack) >= 3:
                X, Y, Z = stack[-3], stack[-2], stack[-1]
                if X[1] <= Y[1] + Z[1] or Y[1] <= Z[1]:
                    if X[1] < Z[1]:
                        _merge_at(arr, stack, -3)
                    else:
                        _merge_at(arr, stack, -2)
                else:
                    break
            elif len(stack) >= 2:
                X, Y = stack[-2], stack[-1]
                if X[1] <= Y[1]:
                    _merge_at(arr, stack, -2)
                else:
                    break

    # 남은 run 모두 병합
    while len(stack) > 1:
        _merge_at(arr, stack, -2)

def _merge_at(arr, stack, idx):
    """스택에서 idx와 idx+1 병합"""
    run1 = stack[idx]
    run2 = stack[idx + 1]

    # 병합 수행
    merged_start = run1[0]
    merged_len = run1[1] + run2[1]

    _merge(arr, run1[0], run1[1], run2[0], run2[1])

    # 스택 업데이트
    stack[idx] = (merged_start, merged_len)
    del stack[idx + 1]

def _merge(arr, start1, len1, start2, len2):
    """두 인접 run 병합"""
    temp = arr[start1:start1 + len1]
    i, j, k = 0, start2, start1

    while i < len1 and j < start2 + len2:
        if temp[i] <= arr[j]:
            arr[k] = temp[i]
            i += 1
        else:
            arr[k] = arr[j]
            j += 1
        k += 1

    while i < len1:
        arr[k] = temp[i]
        i += 1
        k += 1
```

**Timsort 특징**:

| 특성 | 설명 |
|------|------|
| 시간 복잡도 | O(n log n) 최악, O(n) 최선 |
| 공간 복잡도 | O(n) |
| 안정성 | 안정 정렬 |
| 적응성 | 정렬된 데이터에 최적화 |

**Galloping Mode**:
병합 중 한쪽 run에서 연속으로 원소가 선택되면, 이진 탐색으로 전환하여 효율 증가.

**사용처**:
- Python `sorted()`, `list.sort()`
- Java `Arrays.sort(Object[])`
- Android

### 3. pdqsort (Pattern-Defeating Quick Sort)

```
특징:
1. Block Partition: 캐시 친화적
2. Pattern Detection: 정렬된 입력 감지
3. Pivot Selection: adaptive
4. Fallback: Heap Sort (최악 방지)

시간 복잡도:
- 최선: O(n) - 정렬된/역순 입력
- 평균: O(n log n)
- 최악: O(n log n) - 보장
```

**동작 원리**:
```python
def pdqsort_concept(arr, low, high, bad_allowed):
    """pdqsort 개념 의사 코드"""
    while high - low > INSERTION_THRESHOLD:
        # 1. 정렬된 패턴 감지
        if is_sorted(arr, low, high):
            return

        # 2. bad partition이 너무 많으면 heapsort
        if bad_allowed == 0:
            heapsort(arr, low, high)
            return

        # 3. 피봇 선택 및 파티션
        pivot = choose_pivot(arr, low, high)
        pivot_pos, already_partitioned = partition(arr, low, high, pivot)

        # 4. bad partition 감지
        left_size = pivot_pos - low
        right_size = high - pivot_pos
        highly_unbalanced = (left_size < (high - low) / 8 or
                           right_size < (high - low) / 8)

        if highly_unbalanced:
            bad_allowed -= 1
            if not already_partitioned:
                shuffle(arr, low, high)  # 패턴 깨기

        # 5. 재귀
        pdqsort_concept(arr, low, pivot_pos, bad_allowed)
        low = pivot_pos + 1

    # 작은 배열은 삽입 정렬
    insertion_sort(arr, low, high)
```

**사용처**:
- Rust 표준 라이브러리
- C++ Boost

### 4. 비교: 주요 하이브리드 정렬

| 알고리즘 | 기반 | 안정성 | 최악 시간 | 공간 | 사용처 |
|----------|------|--------|-----------|------|--------|
| Introsort | Quick + Heap + Insertion | 불안정 | O(n log n) | O(log n) | C++ std::sort |
| Timsort | Merge + Insertion | 안정 | O(n log n) | O(n) | Python, Java |
| pdqsort | Quick + Heap + Insertion | 불안정 | O(n log n) | O(log n) | Rust |

### 5. 임계값 선택

왜 작은 배열에서 삽입 정렬을 사용하는가?

```python
# 비교 횟수 분석 (n원소)

# Quick Sort: ~1.39n log n 비교
# Insertion Sort: ~n²/4 비교 (평균)

# 교차점: 1.39n log n = n²/4
# n ≈ 10~20
```

**실제 임계값**:
- Java: 7 (Dual-Pivot Quick Sort), 32 (Timsort run)
- Python: 32 (Timsort)
- C++ (libstdc++): 16
- .NET: 16

```python
import time
import random

def benchmark_threshold():
    """최적 임계값 찾기"""
    for threshold in [4, 8, 16, 32, 64, 128]:
        total_time = 0
        for _ in range(100):
            arr = [random.randint(0, 10000) for _ in range(10000)]

            start = time.time()
            hybrid_sort(arr, threshold)
            total_time += time.time() - start

        print(f"Threshold {threshold}: {total_time:.4f}s")

def hybrid_sort(arr, threshold):
    _hybrid_sort(arr, 0, len(arr) - 1, threshold)

def _hybrid_sort(arr, low, high, threshold):
    if high - low < threshold:
        _insertion_sort(arr, low, high)
        return

    pivot = _partition(arr, low, high)
    _hybrid_sort(arr, low, pivot - 1, threshold)
    _hybrid_sort(arr, pivot + 1, high, threshold)
```

## 실무 적용

### 1. Java의 정렬 전략

```java
// Arrays.sort(int[]) - Dual-Pivot Quick Sort (Introsort 변형)
// Arrays.sort(Object[]) - Timsort

// 왜 다른가?
// - 기본형: 안정성 불필요 → 더 빠른 Quick Sort 변형
// - 객체: 안정성 필요 → Timsort
```

### 2. 커스텀 하이브리드 정렬

```python
def adaptive_sort(arr):
    """입력 특성에 따른 적응적 정렬"""
    n = len(arr)

    # 1. 크기에 따른 분기
    if n < 16:
        return insertion_sort(arr)

    # 2. 이미 정렬된 정도 확인
    sorted_ratio = check_sorted_ratio(arr)
    if sorted_ratio > 0.9:
        return timsort_like(arr)  # 부분 정렬된 데이터에 최적

    # 3. 중복 비율 확인
    unique_ratio = len(set(arr)) / n
    if unique_ratio < 0.1:
        return three_way_quicksort(arr)  # 중복 많으면 3-way

    # 4. 기본: Introsort
    return introsort(arr)

def check_sorted_ratio(arr):
    """연속으로 정렬된 쌍의 비율"""
    if len(arr) < 2:
        return 1.0
    sorted_pairs = sum(1 for i in range(len(arr)-1) if arr[i] <= arr[i+1])
    return sorted_pairs / (len(arr) - 1)
```

### 3. 병렬 하이브리드 정렬

```python
from concurrent.futures import ThreadPoolExecutor

def parallel_hybrid_sort(arr, threshold=10000, workers=4):
    """병렬 하이브리드 정렬"""
    if len(arr) < threshold:
        return sorted(arr)  # 작은 배열은 순차 정렬

    # 배열을 workers 개로 분할하여 병렬 정렬
    chunk_size = len(arr) // workers
    chunks = [arr[i:i+chunk_size] for i in range(0, len(arr), chunk_size)]

    with ThreadPoolExecutor(max_workers=workers) as executor:
        sorted_chunks = list(executor.map(sorted, chunks))

    # K-way 병합
    return k_way_merge(sorted_chunks)

def k_way_merge(sorted_lists):
    """k개의 정렬된 리스트 병합"""
    import heapq
    result = []
    heap = [(lst[0], i, 0) for i, lst in enumerate(sorted_lists) if lst]
    heapq.heapify(heap)

    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)

        if elem_idx + 1 < len(sorted_lists[list_idx]):
            heapq.heappush(heap, (sorted_lists[list_idx][elem_idx + 1], list_idx, elem_idx + 1))

    return result
```

## 참고 자료

- Timsort: https://bugs.python.org/file4451/timsort.txt
- Introsort: Musser, D.R. "Introspective Sorting and Selection Algorithms"
- pdqsort: https://github.com/orlp/pdqsort
- [Java DualPivotQuicksort](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/DualPivotQuicksort.java)
- [Rust sort implementation](https://doc.rust-lang.org/std/primitive.slice.html#method.sort)

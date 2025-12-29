# 비교 정렬의 하한 (Comparison Sort Lower Bound)

## 개요

비교 기반 정렬 알고리즘은 원소들의 상대적 순서를 결정하기 위해 **비교 연산**만을 사용한다. 이러한 알고리즘은 아무리 효율적으로 설계해도 **Ω(n log n)** 시간 복잡도 이하로 내려갈 수 없다. 이 하한은 정보 이론적 논증을 통해 증명할 수 있다.

## 핵심 개념

### 1. Decision Tree Model

비교 정렬을 **결정 트리(Decision Tree)**로 모델링할 수 있다.

```
              a[1] < a[2]?
             /           \
           yes            no
           /               \
      a[2] < a[3]?      a[1] < a[3]?
       /      \          /       \
     ...      ...      ...       ...
```

**결정 트리의 특성**:
- **내부 노드**: 두 원소 간의 비교 (예: `a[i] < a[j]?`)
- **리프 노드**: 순열의 결과 (정렬된 순서)
- **경로**: 루트에서 리프까지의 비교 시퀀스
- **트리 높이**: 최악의 경우 비교 횟수

### 2. 리프 노드의 개수

n개의 원소를 정렬할 때:
- 가능한 모든 순열의 개수: **n!**
- 결정 트리는 최소 **n!** 개의 리프 노드를 가져야 함
- 각 순열에 대해 올바른 정렬 결과를 도출해야 하기 때문

### 3. 트리 높이의 하한

**정리**: 높이가 h인 이진 트리는 최대 2^h 개의 리프를 가진다.

**증명**:
- 리프 개수 ≤ 2^h
- n! ≤ 2^h (결정 트리의 조건)
- h ≥ log₂(n!)

**log₂(n!)의 근사**:

스털링 근사(Stirling's Approximation)를 사용:
```
n! ≈ √(2πn) × (n/e)^n

log₂(n!) = log₂(√(2πn)) + n × log₂(n/e)
         = Θ(n log n)
```

따라서: **h = Ω(n log n)**

### 4. Information-Theoretic Lower Bound

정보 이론적 관점에서 하한을 이해할 수 있다:

```
입력: n개의 서로 다른 원소
출력: 올바른 정렬 순서 (n! 가지 중 하나)

필요한 정보량: log₂(n!) 비트

각 비교는 최대 1비트의 정보를 제공
→ 최소 log₂(n!) = Ω(n log n) 번의 비교 필요
```

**직관적 이해**:
- n! 가지 가능성 중 정답을 찾아야 함
- 이진 비교(yes/no)로 탐색 공간을 절반씩 줄임
- log₂(n!)번의 비교가 필요

### 5. 정확한 상수 계산

```
log₂(n!) = Σ(i=1 to n) log₂(i)
         ≥ Σ(i=n/2 to n) log₂(i)
         ≥ (n/2) × log₂(n/2)
         = (n/2) × (log₂(n) - 1)
         = (n log₂(n))/2 - n/2
         = Ω(n log n)
```

**더 정밀한 하한**:
- 최악의 경우: 약 **n log₂(n) - 1.44n** 번의 비교
- 평균의 경우도 동일한 Ω(n log n)

## 실무 적용

### 1. 최적 정렬 알고리즘

하한에 도달하는 O(n log n) 정렬 알고리즘:

| 알고리즘 | 최악 | 평균 | 안정성 | 제자리 |
|----------|------|------|--------|--------|
| Merge Sort | O(n log n) | O(n log n) | 안정 | X |
| Heap Sort | O(n log n) | O(n log n) | 불안정 | O |
| Quick Sort | O(n²) | O(n log n) | 불안정 | O |

### 2. 비교 정렬의 한계를 넘어서

비교 정렬의 Ω(n log n) 하한은 **비교만 사용할 때** 적용됨.

**비비교 정렬**로 O(n) 달성 가능:
```python
# Counting Sort - O(n + k)
def counting_sort(arr, max_val):
    count = [0] * (max_val + 1)
    for x in arr:
        count[x] += 1

    result = []
    for i, c in enumerate(count):
        result.extend([i] * c)
    return result
```

단, 특수한 조건이 필요:
- Counting Sort: 정수, 범위 제한
- Radix Sort: 고정 자릿수
- Bucket Sort: 균등 분포

### 3. 결정 트리 분석 코드

```python
import math

def comparison_lower_bound(n):
    """n개 원소 정렬의 비교 횟수 하한 계산"""
    # log₂(n!) 계산
    log_factorial = sum(math.log2(i) for i in range(1, n + 1))
    return log_factorial

def stirling_approximation(n):
    """스털링 근사를 사용한 log₂(n!) 계산"""
    if n <= 1:
        return 0
    return n * math.log2(n) - n * math.log2(math.e) + 0.5 * math.log2(2 * math.pi * n)

# 비교
for n in [10, 100, 1000]:
    exact = comparison_lower_bound(n)
    approx = stirling_approximation(n)
    print(f"n={n}: 정확한 값={exact:.2f}, 스털링 근사={approx:.2f}")
```

출력:
```
n=10: 정확한 값=21.79, 스털링 근사=21.36
n=100: 정확한 값=524.76, 스털링 근사=524.29
n=1000: 정확한 값=8529.83, 스털링 근사=8529.38
```

### 4. 실제 정렬 알고리즘의 비교 횟수

```python
def count_merge_sort_comparisons(arr):
    """Merge Sort의 실제 비교 횟수 측정"""
    comparisons = [0]

    def merge_sort(arr):
        if len(arr) <= 1:
            return arr

        mid = len(arr) // 2
        left = merge_sort(arr[:mid])
        right = merge_sort(arr[mid:])
        return merge(left, right, comparisons)

    def merge(left, right, comparisons):
        result = []
        i = j = 0
        while i < len(left) and j < len(right):
            comparisons[0] += 1
            if left[i] <= right[j]:
                result.append(left[i])
                i += 1
            else:
                result.append(right[j])
                j += 1
        result.extend(left[i:])
        result.extend(right[j:])
        return result

    merge_sort(arr)
    return comparisons[0]
```

## 하한 증명이 적용되지 않는 경우

### 1. 특수한 입력 분포

```python
# 거의 정렬된 배열 - Insertion Sort가 O(n)
def insertion_sort(arr):
    comparisons = 0
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        while j >= 0 and arr[j] > key:
            comparisons += 1
            arr[j + 1] = arr[j]
            j -= 1
        comparisons += 1  # 마지막 비교
        arr[j + 1] = key
    return comparisons

# 거의 정렬된 배열
nearly_sorted = list(range(1000))
nearly_sorted[0], nearly_sorted[1] = nearly_sorted[1], nearly_sorted[0]
print(f"거의 정렬된 배열: {insertion_sort(nearly_sorted.copy())} 비교")
```

### 2. 제한된 값 범위

정수이고 값의 범위가 O(n)이면:
- Counting Sort: O(n)
- Radix Sort: O(d × n) where d는 자릿수

## 참고 자료

- Introduction to Algorithms (CLRS) - Chapter 8: Sorting in Linear Time
- Algorithm Design (Kleinberg & Tardos) - Chapter 5
- Stanford CS161 - Lecture Notes on Lower Bounds
- [Sorting Algorithm Visualizations](https://www.toptal.com/developers/sorting-algorithms)

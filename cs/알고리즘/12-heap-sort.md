# 힙 정렬 (Heap Sort)

## 개요

힙 정렬은 **힙(Heap)** 자료구조를 활용한 정렬 알고리즘이다. 최대 힙을 구성한 후, 루트(최댓값)를 반복적으로 추출하여 정렬한다.

**핵심 특성**:
- 시간 복잡도: **O(n log n)** - 항상 보장
- 공간 복잡도: **O(1)** - 제자리 정렬
- **불안정 정렬(Unstable)**
- 캐시 효율이 낮음

## 핵심 개념

### 1. 힙의 개념

**힙(Heap)**: 완전 이진 트리이면서 힙 속성을 만족하는 자료구조

```
최대 힙(Max Heap): 부모 ≥ 자식
       16
      /  \
    14    10
   /  \  /  \
  8   7  9   3

배열 표현: [16, 14, 10, 8, 7, 9, 3]
인덱스:     [0]  [1] [2] [3][4][5][6]
```

**배열 인덱스 관계** (0-based):
- 부모: `(i - 1) // 2`
- 왼쪽 자식: `2 * i + 1`
- 오른쪽 자식: `2 * i + 2`

### 2. Heapify (힙 복구)

```python
def heapify(arr, n, i):
    """i번째 노드를 루트로 하는 서브트리를 힙으로 만듦"""
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2

    # 왼쪽 자식이 루트보다 크면
    if left < n and arr[left] > arr[largest]:
        largest = left

    # 오른쪽 자식이 가장 크면
    if right < n and arr[right] > arr[largest]:
        largest = right

    # 루트가 가장 크지 않으면 교환 후 재귀
    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
```

**시간 복잡도**: O(log n) - 트리의 높이만큼

### 3. Build-Heap

```python
def build_max_heap(arr):
    """배열을 최대 힙으로 변환"""
    n = len(arr)

    # 리프가 아닌 노드부터 역순으로 heapify
    # 마지막 비리프 노드: (n // 2) - 1
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)
```

**시간 복잡도 분석**:

직관적으로 O(n log n)으로 보이지만, 실제로는 **O(n)**이다.

```
높이 h인 노드 수: n / 2^(h+1)
높이 h에서 heapify 비용: O(h)

총 비용 = Σ(h=0 to log n) [n / 2^(h+1)] × O(h)
        = O(n) × Σ(h=0 to ∞) [h / 2^h]
        = O(n) × 2  (급수의 합)
        = O(n)
```

### 4. 힙 정렬 구현

```python
def heap_sort(arr):
    """힙 정렬 구현"""
    n = len(arr)

    # 1단계: 최대 힙 구성 - O(n)
    build_max_heap(arr)

    # 2단계: 하나씩 추출하여 정렬 - O(n log n)
    for i in range(n - 1, 0, -1):
        # 루트(최대값)를 맨 뒤로 이동
        arr[0], arr[i] = arr[i], arr[0]

        # 힙 크기를 줄이고 heapify
        heapify(arr, i, 0)

    return arr
```

```java
public static void heapSort(int[] arr) {
    int n = arr.length;

    // Build max heap
    for (int i = n / 2 - 1; i >= 0; i--) {
        heapify(arr, n, i);
    }

    // Extract elements from heap
    for (int i = n - 1; i > 0; i--) {
        // Move root to end
        int temp = arr[0];
        arr[0] = arr[i];
        arr[i] = temp;

        // Heapify reduced heap
        heapify(arr, i, 0);
    }
}

public static void heapify(int[] arr, int n, int i) {
    int largest = i;
    int left = 2 * i + 1;
    int right = 2 * i + 2;

    if (left < n && arr[left] > arr[largest])
        largest = left;

    if (right < n && arr[right] > arr[largest])
        largest = right;

    if (largest != i) {
        int temp = arr[i];
        arr[i] = arr[largest];
        arr[largest] = temp;

        heapify(arr, n, largest);
    }
}
```

### 5. 동작 과정 시각화

```
초기 배열: [4, 10, 3, 5, 1]

Build Max Heap:
    4           10           10
   / \    →    /  \    →    /  \
  10  3       5    3       5    3
 / \         / \          / \
5   1       4   1        4   1

힙: [10, 5, 3, 4, 1]

추출 과정:
[10, 5, 3, 4, 1] → swap(10,1) → [1, 5, 3, 4, |10] → heapify → [5, 4, 3, 1, |10]
[5, 4, 3, 1, |10] → swap(5,1) → [1, 4, 3, |5, 10] → heapify → [4, 1, 3, |5, 10]
[4, 1, 3, |5, 10] → swap(4,3) → [3, 1, |4, 5, 10] → heapify → [3, 1, |4, 5, 10]
[3, 1, |4, 5, 10] → swap(3,1) → [1, |3, 4, 5, 10]

결과: [1, 3, 4, 5, 10]
```

### 6. 힙 정렬이 불안정한 이유

```python
# 예: [(A,5), (B,5), (C,3), (D,5)]
# 키 값만 고려한 배열: [5, 5, 3, 5]

# Build Heap 후 (최대 힙):
#      5(D)
#     /    \
#   5(B)   3(C)
#   /
#  5(A)

# 첫 번째 추출:
# 5(D)와 5(A) 교환 → 5(A)를 힙에서 heapify
# → 동일 키 원소의 순서가 변경됨
```

힙 구조에서 교환이 발생할 때 동일 키의 상대적 순서가 보존되지 않음.

### 7. 반복적 Heapify (스택 절약)

```python
def heapify_iterative(arr, n, i):
    """반복문을 사용한 heapify"""
    while True:
        largest = i
        left = 2 * i + 1
        right = 2 * i + 2

        if left < n and arr[left] > arr[largest]:
            largest = left
        if right < n and arr[right] > arr[largest]:
            largest = right

        if largest == i:
            break

        arr[i], arr[largest] = arr[largest], arr[i]
        i = largest
```

### 8. Bottom-Up Heapify (Floyd's Method)

```python
def sift_down(arr, start, end):
    """Floyd's sift-down: 반 교환 사용"""
    root = start
    temp = arr[root]

    while 2 * root + 1 <= end:
        child = 2 * root + 1

        # 더 큰 자식 선택
        if child + 1 <= end and arr[child] < arr[child + 1]:
            child += 1

        if temp >= arr[child]:
            break

        # 교환 대신 이동만
        arr[root] = arr[child]
        root = child

    arr[root] = temp
```

**장점**: 교환 대신 이동으로 약 50% 효율 향상

## 실무 적용

### 1. 우선순위 큐 구현

```python
class MaxHeap:
    """최대 힙 기반 우선순위 큐"""

    def __init__(self):
        self.heap = []

    def push(self, val):
        """삽입 - O(log n)"""
        self.heap.append(val)
        self._sift_up(len(self.heap) - 1)

    def pop(self):
        """최댓값 추출 - O(log n)"""
        if not self.heap:
            raise IndexError("Heap is empty")

        max_val = self.heap[0]
        last = self.heap.pop()

        if self.heap:
            self.heap[0] = last
            self._sift_down(0)

        return max_val

    def peek(self):
        """최댓값 조회 - O(1)"""
        return self.heap[0] if self.heap else None

    def _sift_up(self, i):
        parent = (i - 1) // 2
        while i > 0 and self.heap[i] > self.heap[parent]:
            self.heap[i], self.heap[parent] = self.heap[parent], self.heap[i]
            i = parent
            parent = (i - 1) // 2

    def _sift_down(self, i):
        n = len(self.heap)
        while True:
            largest = i
            left = 2 * i + 1
            right = 2 * i + 2

            if left < n and self.heap[left] > self.heap[largest]:
                largest = left
            if right < n and self.heap[right] > self.heap[largest]:
                largest = right

            if largest == i:
                break

            self.heap[i], self.heap[largest] = self.heap[largest], self.heap[i]
            i = largest
```

### 2. K번째 큰/작은 원소

```python
import heapq

def kth_largest(arr, k):
    """k번째로 큰 원소 - O(n log k)"""
    # 크기 k의 최소 힙 유지
    min_heap = arr[:k]
    heapq.heapify(min_heap)

    for num in arr[k:]:
        if num > min_heap[0]:
            heapq.heapreplace(min_heap, num)

    return min_heap[0]

def kth_smallest(arr, k):
    """k번째로 작은 원소 - O(n log k)"""
    # 크기 k의 최대 힙 유지 (음수로 변환)
    max_heap = [-x for x in arr[:k]]
    heapq.heapify(max_heap)

    for num in arr[k:]:
        if num < -max_heap[0]:
            heapq.heapreplace(max_heap, -num)

    return -max_heap[0]
```

### 3. 중앙값 스트림

```python
class MedianFinder:
    """실시간 중앙값 찾기"""

    def __init__(self):
        self.small = []  # 최대 힙 (작은 절반)
        self.large = []  # 최소 힙 (큰 절반)

    def add_num(self, num):
        # 최대 힙에 추가
        heapq.heappush(self.small, -num)

        # 균형 맞추기
        if self.small and self.large and -self.small[0] > self.large[0]:
            heapq.heappush(self.large, -heapq.heappop(self.small))

        # 크기 조절
        if len(self.small) > len(self.large) + 1:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        elif len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def find_median(self):
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2
```

### 4. K-way Merge (외부 정렬)

```python
import heapq

def k_way_merge(sorted_lists):
    """k개의 정렬된 리스트 병합 - O(n log k)"""
    result = []
    heap = []

    # 각 리스트의 첫 원소를 힙에 추가
    for i, lst in enumerate(sorted_lists):
        if lst:
            heapq.heappush(heap, (lst[0], i, 0))

    while heap:
        val, list_idx, elem_idx = heapq.heappop(heap)
        result.append(val)

        # 다음 원소가 있으면 힙에 추가
        if elem_idx + 1 < len(sorted_lists[list_idx]):
            next_val = sorted_lists[list_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, list_idx, elem_idx + 1))

    return result
```

### 5. Smoothsort 개념

힙 정렬의 적응적(adaptive) 변형:

```
특징:
- 거의 정렬된 입력에서 O(n)에 가까움
- Leonardo 힙 사용
- 최악의 경우 O(n log n)
- 구현이 복잡함

Leonardo 수열: 1, 1, 3, 5, 9, 15, 25, ...
L(n) = L(n-1) + L(n-2) + 1
```

## 참고 자료

- Introduction to Algorithms (CLRS) - Chapter 6
- Algorithms (Sedgewick) - Chapter 2.4
- [Visualgo - Heap](https://visualgo.net/en/heap)
- [HeapSort - Wikipedia](https://en.wikipedia.org/wiki/Heapsort)

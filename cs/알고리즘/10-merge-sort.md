# 병합 정렬 (Merge Sort)

## 개요

병합 정렬은 **분할 정복(Divide and Conquer)** 패러다임의 대표적인 알고리즘이다. 배열을 절반으로 나누고, 각각을 재귀적으로 정렬한 뒤, 두 정렬된 배열을 **병합(Merge)**한다.

**핵심 특성**:
- 시간 복잡도: **O(n log n)** - 항상 보장
- 공간 복잡도: O(n) - 추가 배열 필요
- **안정 정렬(Stable Sort)**
- 외부 정렬(External Sort)에 적합

## 핵심 개념

### 1. 분할 정복 원리

```
원본: [38, 27, 43, 3, 9, 82, 10]

분할(Divide):
[38, 27, 43, 3] | [9, 82, 10]
[38, 27] | [43, 3] | [9, 82] | [10]
[38] | [27] | [43] | [3] | [9] | [82] | [10]

정복(Conquer) & 병합(Combine):
[27, 38] | [3, 43] | [9, 82] | [10]
[3, 27, 38, 43] | [9, 10, 82]
[3, 9, 10, 27, 38, 43, 82]
```

### 2. 기본 구현 (Top-Down)

```python
def merge_sort(arr):
    """Top-down 재귀적 병합 정렬"""
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    return merge(left, right)

def merge(left, right):
    """두 정렬된 배열을 병합"""
    result = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:  # <= 로 안정성 보장
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

```java
public static void mergeSort(int[] arr, int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;  // 오버플로우 방지

        mergeSort(arr, left, mid);
        mergeSort(arr, mid + 1, right);
        merge(arr, left, mid, right);
    }
}

public static void merge(int[] arr, int left, int mid, int right) {
    int[] temp = new int[right - left + 1];
    int i = left, j = mid + 1, k = 0;

    while (i <= mid && j <= right) {
        if (arr[i] <= arr[j]) {
            temp[k++] = arr[i++];
        } else {
            temp[k++] = arr[j++];
        }
    }

    while (i <= mid) temp[k++] = arr[i++];
    while (j <= right) temp[k++] = arr[j++];

    System.arraycopy(temp, 0, arr, left, temp.length);
}
```

### 3. 시간 복잡도 분석

**점화식**:
```
T(n) = 2T(n/2) + Θ(n)
       ↑         ↑
  두 부분 재귀   병합 비용
```

**Master Theorem 적용**:
- a = 2, b = 2, f(n) = n
- n^(log_b(a)) = n^(log_2(2)) = n
- f(n) = Θ(n^(log_b(a)))  → Case 2
- **T(n) = Θ(n log n)**

**재귀 트리로 이해**:
```
레벨 0:                n                    → n
레벨 1:        n/2          n/2             → n
레벨 2:    n/4    n/4   n/4    n/4          → n
...
레벨 log n: 1 1 1 ... 1 1 1                 → n

총합: n × log n = Θ(n log n)
```

### 4. 공간 복잡도 분석

```
호출 스택 깊이: O(log n)
병합에 필요한 임시 배열: O(n)

총 공간 복잡도: O(n)
```

**공간 최적화**:
```python
def merge_sort_inplace(arr, aux, left, right):
    """보조 배열을 재사용하는 버전"""
    if left >= right:
        return

    mid = (left + right) // 2
    merge_sort_inplace(arr, aux, left, mid)
    merge_sort_inplace(arr, aux, mid + 1, right)
    merge_inplace(arr, aux, left, mid, right)

def merge_inplace(arr, aux, left, mid, right):
    # 보조 배열에 복사
    for i in range(left, right + 1):
        aux[i] = arr[i]

    i, j = left, mid + 1
    for k in range(left, right + 1):
        if i > mid:
            arr[k] = aux[j]
            j += 1
        elif j > right:
            arr[k] = aux[i]
            i += 1
        elif aux[i] <= aux[j]:
            arr[k] = aux[i]
            i += 1
        else:
            arr[k] = aux[j]
            j += 1
```

### 5. Bottom-Up 병합 정렬

재귀 없이 반복문으로 구현:

```python
def merge_sort_bottom_up(arr):
    """Bottom-up 반복적 병합 정렬"""
    n = len(arr)
    aux = arr.copy()

    size = 1
    while size < n:
        for left in range(0, n, 2 * size):
            mid = min(left + size - 1, n - 1)
            right = min(left + 2 * size - 1, n - 1)

            if mid < right:
                merge_bottom_up(arr, aux, left, mid, right)

        size *= 2

    return arr

def merge_bottom_up(arr, aux, left, mid, right):
    for i in range(left, right + 1):
        aux[i] = arr[i]

    i, j = left, mid + 1
    for k in range(left, right + 1):
        if i > mid:
            arr[k] = aux[j]
            j += 1
        elif j > right:
            arr[k] = aux[i]
            i += 1
        elif aux[i] <= aux[j]:
            arr[k] = aux[i]
            i += 1
        else:
            arr[k] = aux[j]
            j += 1
```

**Bottom-Up 장점**:
- 재귀 호출 오버헤드 없음
- 연결 리스트에 자연스럽게 적용
- 외부 정렬(External Sort)에 적합

### 6. 안정성(Stability) 증명

**정의**: 동일한 키를 가진 원소들의 상대적 순서가 정렬 후에도 유지됨.

**증명**:
```python
# merge 함수에서 left[i] <= right[j]일 때 left를 먼저 선택
if left[i] <= right[j]:  # 등호 중요!
    result.append(left[i])
```

왼쪽 배열의 원소가 같은 값일 때 먼저 선택되므로:
- 원래 앞에 있던 원소가 병합 후에도 앞에 위치
- 안정성 보장

**반례 (불안정 구현)**:
```python
# < 사용 시 안정성 깨짐
if left[i] < right[j]:  # 엄격한 부등호
    result.append(left[i])
```

### 7. 자연 병합 정렬 (Natural Merge Sort)

이미 정렬된 부분(run)을 활용:

```python
def natural_merge_sort(arr):
    """자연 병합 정렬 - 기존 정렬된 run 활용"""

    def find_runs(arr):
        """오름차순 run 찾기"""
        runs = []
        i = 0
        while i < len(arr):
            start = i
            while i < len(arr) - 1 and arr[i] <= arr[i + 1]:
                i += 1
            runs.append((start, i))
            i += 1
        return runs

    while True:
        runs = find_runs(arr)
        if len(runs) == 1:
            break

        # 인접한 run들을 병합
        for i in range(0, len(runs) - 1, 2):
            left_start, left_end = runs[i]
            right_start, right_end = runs[i + 1]
            merge_inplace(arr, left_start, left_end, right_end)

    return arr
```

Timsort의 기반이 되는 아이디어.

## 실무 적용

### 1. 외부 정렬 (External Sort)

메모리에 담을 수 없는 대용량 데이터 정렬:

```python
import heapq
import tempfile
import os

def external_sort(input_file, output_file, chunk_size=1000000):
    """외부 병합 정렬"""
    temp_files = []

    # 1단계: 청크별로 정렬하여 임시 파일 생성
    with open(input_file, 'r') as f:
        while True:
            chunk = []
            for _ in range(chunk_size):
                line = f.readline()
                if not line:
                    break
                chunk.append(int(line.strip()))

            if not chunk:
                break

            chunk.sort()

            temp_file = tempfile.NamedTemporaryFile(mode='w', delete=False)
            for num in chunk:
                temp_file.write(f"{num}\n")
            temp_file.close()
            temp_files.append(temp_file.name)

    # 2단계: k-way 병합
    file_handles = [open(f, 'r') for f in temp_files]
    heap = []

    # 각 파일의 첫 원소를 힙에 추가
    for i, fh in enumerate(file_handles):
        line = fh.readline()
        if line:
            heapq.heappush(heap, (int(line.strip()), i))

    # 병합하여 출력
    with open(output_file, 'w') as out:
        while heap:
            val, file_idx = heapq.heappop(heap)
            out.write(f"{val}\n")

            line = file_handles[file_idx].readline()
            if line:
                heapq.heappush(heap, (int(line.strip()), file_idx))

    # 정리
    for fh in file_handles:
        fh.close()
    for f in temp_files:
        os.remove(f)
```

### 2. 연결 리스트 정렬

배열과 달리 O(1) 공간으로 병합 가능:

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def merge_sort_list(head):
    """연결 리스트 병합 정렬 - O(n log n) 시간, O(log n) 공간"""
    if not head or not head.next:
        return head

    # 중간 노드 찾기 (slow-fast 포인터)
    slow, fast = head, head.next
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next

    mid = slow.next
    slow.next = None  # 리스트 분할

    left = merge_sort_list(head)
    right = merge_sort_list(mid)

    return merge_lists(left, right)

def merge_lists(l1, l2):
    """두 정렬된 연결 리스트 병합"""
    dummy = ListNode()
    tail = dummy

    while l1 and l2:
        if l1.val <= l2.val:
            tail.next = l1
            l1 = l1.next
        else:
            tail.next = l2
            l2 = l2.next
        tail = tail.next

    tail.next = l1 or l2
    return dummy.next
```

### 3. 역전 쌍 세기 (Inversion Count)

병합 과정에서 역전 쌍 카운트:

```python
def count_inversions(arr):
    """병합 정렬 변형으로 역전 쌍 세기"""
    def sort_and_count(arr):
        if len(arr) <= 1:
            return arr, 0

        mid = len(arr) // 2
        left, left_inv = sort_and_count(arr[:mid])
        right, right_inv = sort_and_count(arr[mid:])
        merged, split_inv = merge_and_count(left, right)

        return merged, left_inv + right_inv + split_inv

    def merge_and_count(left, right):
        result = []
        inversions = 0
        i = j = 0

        while i < len(left) and j < len(right):
            if left[i] <= right[j]:
                result.append(left[i])
                i += 1
            else:
                result.append(right[j])
                # left[i:]의 모든 원소가 right[j]보다 큼
                inversions += len(left) - i
                j += 1

        result.extend(left[i:])
        result.extend(right[j:])
        return result, inversions

    _, total = sort_and_count(arr)
    return total

# 예: [2, 4, 1, 3, 5]
# 역전 쌍: (2,1), (4,1), (4,3) = 3개
```

### 4. 병렬 병합 정렬

```python
from concurrent.futures import ThreadPoolExecutor

def parallel_merge_sort(arr, threshold=1000):
    """병렬 병합 정렬"""
    if len(arr) <= threshold:
        return sorted(arr)  # 작은 배열은 순차 정렬

    mid = len(arr) // 2

    with ThreadPoolExecutor(max_workers=2) as executor:
        left_future = executor.submit(parallel_merge_sort, arr[:mid], threshold)
        right_future = executor.submit(parallel_merge_sort, arr[mid:], threshold)

        left = left_future.result()
        right = right_future.result()

    return merge(left, right)
```

## 참고 자료

- Introduction to Algorithms (CLRS) - Chapter 2
- Algorithms (Sedgewick) - Chapter 2.2
- [Visualgo - Sorting](https://visualgo.net/en/sorting)
- [Timsort - Python's Sorting Algorithm](https://bugs.python.org/file4451/timsort.txt)

# 삽입 정렬 (Insertion Sort)

## 개요

삽입 정렬은 카드를 손에 정리하는 방식과 유사한 직관적인 정렬 알고리즘이다. 배열을 정렬된 부분과 미정렬 부분으로 나누고, 미정렬 부분의 원소를 하나씩 정렬된 부분의 올바른 위치에 **삽입**한다.

**핵심 특성**:
- 시간 복잡도: O(n²) 최악/평균, **O(n) 최선**
- 공간 복잡도: O(1) - 제자리 정렬
- **안정 정렬(Stable Sort)**
- **적응적(Adaptive)**: 거의 정렬된 입력에서 매우 효율적

## 핵심 개념

### 1. 알고리즘 동작 원리

```
초기 배열: [5, 2, 4, 6, 1, 3]

i=1: 2를 삽입 → [2, 5, 4, 6, 1, 3]
     정렬됨: [2, 5]

i=2: 4를 삽입 → [2, 4, 5, 6, 1, 3]
     정렬됨: [2, 4, 5]

i=3: 6을 삽입 → [2, 4, 5, 6, 1, 3]
     정렬됨: [2, 4, 5, 6]

i=4: 1을 삽입 → [1, 2, 4, 5, 6, 3]
     정렬됨: [1, 2, 4, 5, 6]

i=5: 3을 삽입 → [1, 2, 3, 4, 5, 6]
     정렬됨: [1, 2, 3, 4, 5, 6]
```

### 2. 기본 구현

```python
def insertion_sort(arr):
    """기본 삽입 정렬"""
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1

        # key보다 큰 원소들을 오른쪽으로 이동
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1

        arr[j + 1] = key

    return arr
```

```java
public static void insertionSort(int[] arr) {
    for (int i = 1; i < arr.length; i++) {
        int key = arr[i];
        int j = i - 1;

        while (j >= 0 && arr[j] > key) {
            arr[j + 1] = arr[j];
            j--;
        }

        arr[j + 1] = key;
    }
}
```

### 3. Loop Invariant를 통한 정확성 증명

**불변식(Loop Invariant)**: 반복문 i 시작 시, arr[0..i-1]은 원래 arr[0..i-1]의 원소들을 정렬된 순서로 포함한다.

**증명**:

1. **초기화(Initialization)**: i=1일 때, arr[0..0]은 단일 원소이므로 자명하게 정렬됨.

2. **유지(Maintenance)**: arr[j] > key인 모든 원소를 오른쪽으로 이동하고, key를 올바른 위치에 삽입. 따라서 arr[0..i]가 정렬된 상태가 됨.

3. **종료(Termination)**: i = n일 때 종료. 불변식에 의해 arr[0..n-1] 전체가 정렬됨.

### 4. 시간 복잡도 분석

```python
def insertion_sort_with_analysis(arr):
    comparisons = 0
    swaps = 0

    for i in range(1, len(arr)):         # n-1번
        key = arr[i]
        j = i - 1

        while j >= 0 and arr[j] > key:   # 최대 i번
            comparisons += 1
            arr[j + 1] = arr[j]
            swaps += 1
            j -= 1

        if j >= 0:
            comparisons += 1  # while 탈출 시 마지막 비교

        arr[j + 1] = key

    return arr, comparisons, swaps
```

**케이스별 분석**:

| 케이스 | 비교 횟수 | 이동 횟수 | 시간 복잡도 |
|--------|-----------|-----------|-------------|
| 최선 (이미 정렬) | n-1 | 0 | O(n) |
| 최악 (역순 정렬) | n(n-1)/2 | n(n-1)/2 | O(n²) |
| 평균 | n(n-1)/4 | n(n-1)/4 | O(n²) |

**최악의 경우 유도**:
```
역순 배열: [n, n-1, n-2, ..., 2, 1]

i=1: 1번 비교, 1번 이동
i=2: 2번 비교, 2번 이동
...
i=n-1: (n-1)번 비교, (n-1)번 이동

총합: 1 + 2 + ... + (n-1) = n(n-1)/2 = O(n²)
```

### 5. Adaptive 특성

삽입 정렬은 **역전(Inversion)**의 수에 비례하는 시간이 걸린다.

**역전(Inversion)**: i < j이지만 arr[i] > arr[j]인 쌍 (i, j)

```python
def count_inversions(arr):
    """역전의 개수 세기"""
    count = 0
    for i in range(len(arr)):
        for j in range(i + 1, len(arr)):
            if arr[i] > arr[j]:
                count += 1
    return count
```

**정리**: 삽입 정렬의 실행 시간은 O(n + I)이다. 여기서 I는 역전의 수.

- 완전히 정렬된 배열: I = 0 → O(n)
- 역순 배열: I = n(n-1)/2 → O(n²)
- k개의 원소만 잘못된 위치: I ≤ k(n-1) → O(kn)

### 6. Binary Insertion Sort

비교 횟수를 줄이기 위해 이진 탐색 활용:

```python
import bisect

def binary_insertion_sort(arr):
    """이진 탐색을 활용한 삽입 정렬"""
    for i in range(1, len(arr)):
        key = arr[i]

        # 이진 탐색으로 삽입 위치 찾기: O(log i)
        pos = bisect.bisect_left(arr, key, 0, i)

        # 원소 이동은 여전히 O(i)
        arr[pos+1:i+1] = arr[pos:i]
        arr[pos] = key

    return arr
```

**분석**:
- 비교 횟수: O(n log n)
- 이동 횟수: O(n²) (여전히)
- 총 시간 복잡도: O(n²)

이진 삽입 정렬은 비교 비용이 큰 경우(예: 문자열, 복잡한 객체)에 유용.

### 7. Shell Sort - 삽입 정렬의 확장

삽입 정렬의 약점(원소가 멀리 이동해야 함)을 개선:

```python
def shell_sort(arr):
    """Shell Sort - 간격 시퀀스를 사용한 삽입 정렬 개선"""
    n = len(arr)
    gap = n // 2

    while gap > 0:
        # gap 간격으로 삽입 정렬
        for i in range(gap, n):
            temp = arr[i]
            j = i

            while j >= gap and arr[j - gap] > temp:
                arr[j] = arr[j - gap]
                j -= gap

            arr[j] = temp

        gap //= 2

    return arr
```

**간격 시퀀스에 따른 복잡도**:

| 시퀀스 | 복잡도 |
|--------|--------|
| Shell's (n/2, n/4, ..., 1) | O(n²) |
| Hibbard's (2^k - 1) | O(n^1.5) |
| Sedgewick's | O(n^4/3) |
| Ciura's empirical | 매우 좋은 실제 성능 |

```python
def shell_sort_ciura(arr):
    """Ciura의 경험적 간격 시퀀스"""
    gaps = [701, 301, 132, 57, 23, 10, 4, 1]
    n = len(arr)

    for gap in gaps:
        if gap >= n:
            continue

        for i in range(gap, n):
            temp = arr[i]
            j = i

            while j >= gap and arr[j - gap] > temp:
                arr[j] = arr[j - gap]
                j -= gap

            arr[j] = temp

    return arr
```

## 실무 적용

### 1. 언제 삽입 정렬을 사용하는가?

**적합한 경우**:
1. **작은 배열** (n < 10~50): 오버헤드가 적어 Quick/Merge Sort보다 빠름
2. **거의 정렬된 데이터**: O(n)에 근접
3. **실시간 정렬**: 데이터가 하나씩 들어올 때
4. **안정성 필요**: 동일 키의 상대적 순서 유지

### 2. 하이브리드 정렬에서의 활용

```python
def hybrid_sort(arr, threshold=16):
    """Quick Sort + Insertion Sort 하이브리드"""
    def quicksort(arr, low, high):
        if high - low < threshold:
            # 작은 배열은 삽입 정렬
            insertion_sort_range(arr, low, high)
            return

        # 일반적인 퀵소트 로직
        pivot = partition(arr, low, high)
        quicksort(arr, low, pivot - 1)
        quicksort(arr, pivot + 1, high)

    def insertion_sort_range(arr, low, high):
        for i in range(low + 1, high + 1):
            key = arr[i]
            j = i - 1
            while j >= low and arr[j] > key:
                arr[j + 1] = arr[j]
                j -= 1
            arr[j + 1] = key

    quicksort(arr, 0, len(arr) - 1)
    return arr
```

**실제 사례**:
- Java의 `Arrays.sort()`: 작은 배열에서 삽입 정렬 사용
- Python의 Timsort: 작은 run에서 삽입 정렬 사용
- C++ `std::sort`: Introsort 내에서 삽입 정렬 사용

### 3. 온라인 정렬

```python
class OnlineSorter:
    """데이터가 스트림으로 들어올 때 정렬 유지"""

    def __init__(self):
        self.data = []

    def insert(self, value):
        """새 값을 정렬된 위치에 삽입 - O(n)"""
        # 이진 탐색으로 위치 찾기
        import bisect
        bisect.insort(self.data, value)

    def get_sorted(self):
        return self.data
```

### 4. 연결 리스트에서의 삽입 정렬

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def insertion_sort_list(head):
    """연결 리스트 삽입 정렬 - O(1) 추가 공간"""
    if not head:
        return head

    dummy = ListNode(float('-inf'))
    curr = head

    while curr:
        # 삽입 위치 찾기
        prev = dummy
        while prev.next and prev.next.val < curr.val:
            prev = prev.next

        # 삽입
        next_curr = curr.next
        curr.next = prev.next
        prev.next = curr
        curr = next_curr

    return dummy.next
```

## 참고 자료

- Introduction to Algorithms (CLRS) - Chapter 2
- The Art of Computer Programming, Vol. 3 (Knuth) - Sorting and Searching
- [Visualgo - Sorting](https://visualgo.net/en/sorting)
- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/)

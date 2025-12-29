# 계수 정렬 (Counting Sort)

## 개요

계수 정렬은 **비비교 정렬** 알고리즘으로, 원소의 개수를 세어 정렬한다. 정수처럼 값의 범위가 제한된 데이터에서 **O(n + k)** 시간에 정렬할 수 있다 (k는 값의 범위).

**핵심 특성**:
- 시간 복잡도: **O(n + k)**
- 공간 복잡도: **O(k)** (counting 배열)
- **안정 정렬(Stable Sort)** 가능
- 정수/이산 데이터에만 적용

## 핵심 개념

### 1. 기본 아이디어

```
입력: [4, 2, 2, 8, 3, 3, 1]
범위: 0 ~ 8

1단계: 각 값의 개수 세기
count[1] = 1, count[2] = 2, count[3] = 2, count[4] = 1, count[8] = 1

2단계: 누적합 계산 (위치 결정)
count[1] = 1, count[2] = 3, count[3] = 5, count[4] = 6, count[8] = 7

3단계: 역순으로 배치 (안정성)
결과: [1, 2, 2, 3, 3, 4, 8]
```

### 2. 기본 구현 (안정 정렬)

```python
def counting_sort(arr, max_val=None):
    """안정적인 계수 정렬"""
    if not arr:
        return arr

    if max_val is None:
        max_val = max(arr)

    n = len(arr)
    k = max_val + 1

    # 1단계: 개수 세기
    count = [0] * k
    for num in arr:
        count[num] += 1

    # 2단계: 누적합 (각 값이 끝나는 위치)
    for i in range(1, k):
        count[i] += count[i - 1]

    # 3단계: 역순으로 결과 배열에 배치 (안정성 보장)
    output = [0] * n
    for i in range(n - 1, -1, -1):  # 역순 순회
        num = arr[i]
        count[num] -= 1
        output[count[num]] = num

    return output
```

```java
public static int[] countingSort(int[] arr, int maxVal) {
    int n = arr.length;
    int[] count = new int[maxVal + 1];
    int[] output = new int[n];

    // Count occurrences
    for (int num : arr) {
        count[num]++;
    }

    // Cumulative count
    for (int i = 1; i <= maxVal; i++) {
        count[i] += count[i - 1];
    }

    // Build output (reverse for stability)
    for (int i = n - 1; i >= 0; i--) {
        int num = arr[i];
        output[count[num] - 1] = num;
        count[num]--;
    }

    return output;
}
```

### 3. 안정성 보장 원리

```
입력: [(A,2), (B,1), (C,2), (D,1)]
키만: [2, 1, 2, 1]

count 후: count[1]=2, count[2]=2
누적합 후: count[1]=2, count[2]=4

역순 배치:
i=3: D(1) → output[count[1]-1] = output[1], count[1]=1
i=2: C(2) → output[count[2]-1] = output[3], count[2]=3
i=1: B(1) → output[count[1]-1] = output[0], count[1]=0
i=0: A(2) → output[count[2]-1] = output[2], count[2]=2

결과: [B(1), D(1), A(2), C(2)]
→ 같은 키(1,1)에서 B가 D 앞 (원래 순서 유지)
→ 같은 키(2,2)에서 A가 C 앞 (원래 순서 유지)
```

### 4. 간단한 버전 (불안정)

위치 정보가 필요 없는 경우:

```python
def counting_sort_simple(arr, max_val=None):
    """간단한 계수 정렬 (불안정)"""
    if not arr:
        return arr

    if max_val is None:
        max_val = max(arr)

    count = [0] * (max_val + 1)

    for num in arr:
        count[num] += 1

    result = []
    for i, c in enumerate(count):
        result.extend([i] * c)

    return result
```

### 5. 음수 처리

```python
def counting_sort_negative(arr):
    """음수를 포함하는 계수 정렬"""
    if not arr:
        return arr

    min_val = min(arr)
    max_val = max(arr)
    range_size = max_val - min_val + 1

    count = [0] * range_size

    # 오프셋 적용
    for num in arr:
        count[num - min_val] += 1

    # 누적합
    for i in range(1, range_size):
        count[i] += count[i - 1]

    # 배치
    output = [0] * len(arr)
    for i in range(len(arr) - 1, -1, -1):
        num = arr[i]
        count[num - min_val] -= 1
        output[count[num - min_val]] = num

    return output
```

### 6. 시간/공간 복잡도 분석

```
n = 배열 크기
k = 값의 범위 (max - min + 1)

시간 복잡도:
- 개수 세기: O(n)
- 누적합 계산: O(k)
- 결과 배치: O(n)
총합: O(n + k)

공간 복잡도:
- count 배열: O(k)
- output 배열: O(n)
총합: O(n + k)
```

**적용 조건**:
- k = O(n)이면 → O(n) 정렬
- k = O(n²)이면 → O(n²)로 비효율적

### 7. 비교 정렬 하한과의 관계

```
비교 정렬: Ω(n log n)
계수 정렬: O(n + k)

왜 하한을 피할 수 있는가?
→ 계수 정렬은 비교를 하지 않음
→ 원소 값 자체를 인덱스로 사용
→ 값의 범위라는 추가 정보 활용
```

## 실무 적용

### 1. Radix Sort의 기반

```python
def radix_sort(arr):
    """기수 정렬 - 계수 정렬을 각 자릿수에 적용"""
    if not arr:
        return arr

    max_val = max(arr)
    exp = 1

    while max_val // exp > 0:
        arr = counting_sort_by_digit(arr, exp)
        exp *= 10

    return arr

def counting_sort_by_digit(arr, exp):
    """특정 자릿수 기준 계수 정렬"""
    n = len(arr)
    output = [0] * n
    count = [0] * 10

    for num in arr:
        digit = (num // exp) % 10
        count[digit] += 1

    for i in range(1, 10):
        count[i] += count[i - 1]

    for i in range(n - 1, -1, -1):
        digit = (arr[i] // exp) % 10
        output[count[digit] - 1] = arr[i]
        count[digit] -= 1

    return output
```

### 2. 문자열 정렬

```python
def sort_strings_by_length(strings, max_len=None):
    """문자열을 길이 기준으로 정렬"""
    if not strings:
        return strings

    if max_len is None:
        max_len = max(len(s) for s in strings)

    count = [0] * (max_len + 1)

    for s in strings:
        count[len(s)] += 1

    for i in range(1, max_len + 1):
        count[i] += count[i - 1]

    output = [None] * len(strings)
    for i in range(len(strings) - 1, -1, -1):
        s = strings[i]
        count[len(s)] -= 1
        output[count[len(s)]] = s

    return output
```

### 3. 나이/점수 정렬

```python
def sort_by_age(people):
    """나이 기준 정렬 (0~150세)"""
    MAX_AGE = 150

    count = [0] * (MAX_AGE + 1)

    for person in people:
        count[person.age] += 1

    for i in range(1, MAX_AGE + 1):
        count[i] += count[i - 1]

    output = [None] * len(people)
    for i in range(len(people) - 1, -1, -1):
        person = people[i]
        count[person.age] -= 1
        output[count[person.age]] = person

    return output
```

### 4. 히스토그램/빈도 분석

```python
def frequency_analysis(arr, max_val):
    """원소 빈도 분석"""
    count = [0] * (max_val + 1)

    for num in arr:
        count[num] += 1

    # 빈도순 정렬
    freq_pairs = [(i, c) for i, c in enumerate(count) if c > 0]
    freq_pairs.sort(key=lambda x: -x[1])

    return freq_pairs
```

### 5. 중복 제거 및 카운트

```python
def count_unique(arr, max_val):
    """고유 원소 개수 및 각 빈도"""
    count = [0] * (max_val + 1)

    for num in arr:
        count[num] += 1

    unique_count = sum(1 for c in count if c > 0)
    return unique_count, count
```

### 6. 제자리(In-place) 계수 정렬

공간을 절약하는 버전 (안정성 희생):

```python
def counting_sort_inplace(arr, max_val):
    """제자리 계수 정렬 (불안정)"""
    count = [0] * (max_val + 1)

    for num in arr:
        count[num] += 1

    idx = 0
    for val in range(max_val + 1):
        for _ in range(count[val]):
            arr[idx] = val
            idx += 1

    return arr
```

## 참고 자료

- Introduction to Algorithms (CLRS) - Chapter 8
- Algorithms (Sedgewick) - Chapter 2.5
- [Visualgo - Counting Sort](https://visualgo.net/en/sorting)
- [GeeksforGeeks - Counting Sort](https://www.geeksforgeeks.org/counting-sort/)

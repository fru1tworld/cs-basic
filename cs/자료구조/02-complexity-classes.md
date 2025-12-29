# 복잡도 클래스 (Complexity Classes)

## 개요

복잡도 클래스는 알고리즘의 시간 또는 공간 요구량을 입력 크기의 함수로 분류한 것이다. 각 클래스는 특정 성장률을 가지며, 이를 이해하면 알고리즘의 확장성을 예측할 수 있다.

## 핵심 개념

### 복잡도 클래스 계층

```
성장률 순서 (느린 것부터 빠른 것):
O(1) < O(log n) < O(√n) < O(n) < O(n log n) < O(n²) < O(n³) < O(2ⁿ) < O(n!) < O(nⁿ)
```

### 1. 상수 시간 - O(1)

**특징**: 입력 크기와 무관하게 일정한 시간 소요

```java
// 배열 인덱스 접근
int getElement(int[] arr, int index) {
    return arr[index];  // O(1)
}

// 해시 테이블 조회 (평균)
boolean contains(HashSet<Integer> set, int key) {
    return set.contains(key);  // O(1) average
}

// 스택 push/pop
void pushAndPop(Stack<Integer> stack, int value) {
    stack.push(value);  // O(1)
    stack.pop();        // O(1)
}
```

**실제 예시**:
- 배열 인덱스 접근
- 해시 테이블 삽입/조회/삭제 (평균)
- 연결 리스트 헤드 삽입
- 스택/큐의 기본 연산

### 2. 로그 시간 - O(log n)

**특징**: 입력이 2배가 되면 연산 횟수가 1만 증가

```java
// 이진 탐색
int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;

    while (left <= right) {
        int mid = left + (right - left) / 2;

        if (arr[mid] == target) return mid;
        if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
// 반복마다 탐색 범위가 절반 → O(log n)
```

```java
// 균형 이진 탐색 트리 검색
TreeNode search(TreeNode root, int key) {
    if (root == null || root.val == key) return root;

    if (key < root.val) return search(root.left, key);
    return search(root.right, key);
}
// 높이가 log n인 균형 트리 → O(log n)
```

**왜 log n인가?**
```
n = 1024일 때 이진 탐색 최대 비교 횟수:
1024 → 512 → 256 → 128 → 64 → 32 → 16 → 8 → 4 → 2 → 1
= 10번 = log₂(1024)
```

**실제 예시**:
- 이진 탐색
- 균형 BST (AVL, Red-Black) 연산
- 힙 삽입/삭제
- 숫자의 자릿수 계산

### 3. 선형 시간 - O(n)

**특징**: 입력 크기에 비례하여 시간 증가

```java
// 배열 순회
int findMax(int[] arr) {
    int max = arr[0];
    for (int i = 1; i < arr.length; i++) {  // n-1번 반복
        if (arr[i] > max) max = arr[i];
    }
    return max;
}
```

```java
// 연결 리스트 탐색
ListNode findNode(ListNode head, int target) {
    ListNode curr = head;
    while (curr != null) {  // 최악 n번
        if (curr.val == target) return curr;
        curr = curr.next;
    }
    return null;
}
```

**실제 예시**:
- 배열/리스트 순회
- 선형 탐색
- 단순 문자열 비교
- 카운팅 정렬 (범위가 n에 비례할 때)

### 4. 선형 로그 시간 - O(n log n)

**특징**: 효율적인 비교 기반 정렬의 하한

```java
// 병합 정렬
void mergeSort(int[] arr, int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;

        mergeSort(arr, left, mid);      // T(n/2)
        mergeSort(arr, mid + 1, right); // T(n/2)
        merge(arr, left, mid, right);   // O(n)
    }
}
// T(n) = 2T(n/2) + O(n) = O(n log n)
```

**왜 n log n인가?**
```
분할 정복 재귀 트리:
레벨 0:       n           → n번 연산
레벨 1:    n/2  n/2       → n번 연산
레벨 2:  n/4 n/4 n/4 n/4  → n번 연산
...
레벨 log n: 1 1 1 ... 1   → n번 연산

총 log n개 레벨 × 각 레벨 n번 연산 = O(n log n)
```

**실제 예시**:
- 병합 정렬, 힙 정렬, 퀵 정렬 (평균)
- 균형 BST 구축
- FFT (Fast Fourier Transform)

### 5. 다항 시간 - O(n²), O(n³)

**O(n²) - 이차 시간**:

```java
// 버블 정렬
void bubbleSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {        // n-1번
        for (int j = 0; j < n - i - 1; j++) { // 평균 n/2번
            if (arr[j] > arr[j + 1]) {
                swap(arr, j, j + 1);
            }
        }
    }
}
// (n-1) + (n-2) + ... + 1 = n(n-1)/2 = O(n²)
```

```java
// 2차원 배열 순회
void print2DArray(int[][] matrix) {
    for (int i = 0; i < matrix.length; i++) {       // n번
        for (int j = 0; j < matrix[0].length; j++) { // n번
            System.out.print(matrix[i][j] + " ");
        }
    }
}
// n × n = O(n²)
```

**O(n³) - 삼차 시간**:

```java
// 행렬 곱셈 (기본 알고리즘)
int[][] matrixMultiply(int[][] A, int[][] B) {
    int n = A.length;
    int[][] C = new int[n][n];

    for (int i = 0; i < n; i++) {         // n번
        for (int j = 0; j < n; j++) {     // n번
            for (int k = 0; k < n; k++) { // n번
                C[i][j] += A[i][k] * B[k][j];
            }
        }
    }
    return C;
}
// n × n × n = O(n³)
```

**실제 예시**:
- O(n²): 버블/선택/삽입 정렬, 간단한 중복 검사
- O(n³): 행렬 곱셈 (나이브), 플로이드-워셜 알고리즘

### 6. 지수 시간 - O(2ⁿ)

**특징**: 입력이 1 증가하면 시간이 2배

```java
// 피보나치 (나이브 재귀)
int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
// T(n) = T(n-1) + T(n-2) + O(1) ≈ O(2ⁿ)
```

```java
// 부분집합 생성
void generateSubsets(int[] nums) {
    int n = nums.length;
    for (int mask = 0; mask < (1 << n); mask++) {  // 2ⁿ개의 부분집합
        List<Integer> subset = new ArrayList<>();
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) {
                subset.add(nums[i]);
            }
        }
        System.out.println(subset);
    }
}
// 2ⁿ개의 부분집합 × O(n) = O(n × 2ⁿ)
```

**실제 예시**:
- 모든 부분집합 생성
- 외판원 문제 (브루트포스)
- 나이브 피보나치 재귀
- 배낭 문제 (브루트포스)

### 7. 팩토리얼 시간 - O(n!)

**특징**: 극도로 빠르게 증가, 작은 입력에서만 실용적

```java
// 순열 생성
void permute(int[] nums, int start, List<List<Integer>> result) {
    if (start == nums.length) {
        // 하나의 순열 완성
        List<Integer> perm = new ArrayList<>();
        for (int num : nums) perm.add(num);
        result.add(perm);
        return;
    }

    for (int i = start; i < nums.length; i++) {  // n-start가지 선택
        swap(nums, start, i);
        permute(nums, start + 1, result);
        swap(nums, start, i);
    }
}
// n × (n-1) × (n-2) × ... × 1 = n!
```

**성장률 비교**:
```
n    | 2ⁿ      | n!
-----|---------|------------
5    | 32      | 120
10   | 1024    | 3,628,800
15   | 32,768  | 1,307,674,368,000
20   | 1M      | 2.4 × 10¹⁸
```

**실제 예시**:
- 모든 순열 생성
- 외판원 문제 (완전 탐색)
- 브루트포스 최적화 문제

### 복잡도 클래스 비교표

| 복잡도 | n=10 | n=100 | n=1000 | n=10000 | 실용성 |
|--------|------|-------|--------|---------|--------|
| O(1) | 1 | 1 | 1 | 1 | 항상 |
| O(log n) | 3 | 7 | 10 | 13 | 항상 |
| O(n) | 10 | 100 | 1000 | 10000 | 항상 |
| O(n log n) | 33 | 664 | 9966 | 132877 | 항상 |
| O(n²) | 100 | 10000 | 10⁶ | 10⁸ | n ≤ ~10⁴ |
| O(n³) | 1000 | 10⁶ | 10⁹ | 10¹² | n ≤ ~10³ |
| O(2ⁿ) | 1024 | 10³⁰ | - | - | n ≤ ~25 |
| O(n!) | 3.6×10⁶ | - | - | - | n ≤ ~12 |

## 실무 적용

### 입력 크기에 따른 알고리즘 선택

```python
# 코딩 테스트에서의 경험적 기준 (1초 시간 제한 기준)

def choose_algorithm(n):
    """
    입력 크기 n에 따라 사용 가능한 최대 복잡도
    """
    if n <= 12:
        return "O(n!) 가능 - 완전 탐색, 순열"
    elif n <= 25:
        return "O(2^n) 가능 - 부분집합, 비트마스크 DP"
    elif n <= 500:
        return "O(n³) 가능 - 플로이드-워셜"
    elif n <= 5000:
        return "O(n²) 가능 - 단순 DP, 버블 정렬"
    elif n <= 10**6:
        return "O(n log n) 필요 - 정렬 기반, 이분 탐색"
    elif n <= 10**8:
        return "O(n) 필요 - 선형 스캔"
    else:
        return "O(log n) 또는 O(1) 필요"
```

### 알고리즘 개선 예시

```java
// 두 배열에서 공통 원소 찾기

// O(n²) 방법
List<Integer> findCommonNaive(int[] a, int[] b) {
    List<Integer> result = new ArrayList<>();
    for (int x : a) {           // O(n)
        for (int y : b) {       // O(n)
            if (x == y) {
                result.add(x);
                break;
            }
        }
    }
    return result;
}

// O(n log n) 방법 - 정렬 활용
List<Integer> findCommonSorted(int[] a, int[] b) {
    Arrays.sort(a);  // O(n log n)
    Arrays.sort(b);  // O(n log n)

    List<Integer> result = new ArrayList<>();
    int i = 0, j = 0;
    while (i < a.length && j < b.length) {  // O(n)
        if (a[i] == b[j]) {
            result.add(a[i]);
            i++; j++;
        } else if (a[i] < b[j]) {
            i++;
        } else {
            j++;
        }
    }
    return result;
}

// O(n) 방법 - 해시 활용
List<Integer> findCommonHash(int[] a, int[] b) {
    Set<Integer> setA = new HashSet<>();
    for (int x : a) setA.add(x);  // O(n)

    List<Integer> result = new ArrayList<>();
    for (int y : b) {             // O(n)
        if (setA.contains(y)) {   // O(1) average
            result.add(y);
        }
    }
    return result;
}
```

## 참고 자료

- Introduction to Algorithms (CLRS), Chapter 3
- MIT 6.006 Lecture Notes on Algorithmic Complexity
- Big-O Cheat Sheet: https://www.bigocheatsheet.com/

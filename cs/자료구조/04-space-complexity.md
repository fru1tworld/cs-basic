# 공간 복잡도 (Space Complexity)

## 개요

공간 복잡도는 알고리즘이 실행되는 동안 사용하는 메모리 양을 입력 크기의 함수로 표현한 것이다. 메모리 제한이 있는 환경이나 대용량 데이터 처리 시 중요한 고려 사항이다.

## 핵심 개념

### Auxiliary Space vs Total Space

```
Total Space = Input Space + Auxiliary Space

- Input Space: 입력 데이터가 차지하는 공간
- Auxiliary Space: 알고리즘 실행에 필요한 추가 공간 (임시 변수, 자료구조 등)
```

#### 예시: 배열 정렬

```java
// Merge Sort
// Input Space: O(n) - 입력 배열
// Auxiliary Space: O(n) - 임시 배열
// Total Space: O(n) + O(n) = O(n)
void mergeSort(int[] arr, int[] temp, int left, int right) {
    if (left < right) {
        int mid = (left + right) / 2;
        mergeSort(arr, temp, left, mid);
        mergeSort(arr, temp, mid + 1, right);
        merge(arr, temp, left, mid, right);
    }
}

// Quick Sort (in-place)
// Input Space: O(n)
// Auxiliary Space: O(log n) - 재귀 스택
// Total Space: O(n) + O(log n) = O(n)
void quickSort(int[] arr, int low, int high) {
    if (low < high) {
        int pi = partition(arr, low, high);
        quickSort(arr, low, pi - 1);
        quickSort(arr, pi + 1, high);
    }
}

// Heap Sort (in-place)
// Input Space: O(n)
// Auxiliary Space: O(1) - 상수 개의 변수만 사용
// Total Space: O(n) + O(1) = O(n)
void heapSort(int[] arr) {
    int n = arr.length;
    for (int i = n / 2 - 1; i >= 0; i--)
        heapify(arr, n, i);
    for (int i = n - 1; i > 0; i--) {
        swap(arr, 0, i);
        heapify(arr, i, 0);
    }
}
```

### In-place Algorithm

**정의**: Auxiliary Space가 O(1) 또는 O(log n)인 알고리즘

```java
// In-place: 버블 정렬
// Auxiliary Space: O(1)
void bubbleSort(int[] arr) {
    int n = arr.length;
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - i - 1; j++) {
            if (arr[j] > arr[j + 1]) {
                // 임시 변수 1개만 사용
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}

// Not in-place: 카운팅 정렬
// Auxiliary Space: O(k) where k = 값의 범위
int[] countingSort(int[] arr, int k) {
    int[] count = new int[k + 1];  // O(k) 추가 공간
    int[] output = new int[arr.length];  // O(n) 추가 공간

    for (int x : arr) count[x]++;
    for (int i = 1; i <= k; i++) count[i] += count[i-1];
    for (int i = arr.length - 1; i >= 0; i--) {
        output[count[arr[i]] - 1] = arr[i];
        count[arr[i]]--;
    }
    return output;
}
```

### 재귀와 공간 복잡도

재귀 알고리즘은 호출 스택이 공간을 차지함

```java
// 피보나치 - 재귀 (비효율적)
// 시간: O(2^n)
// 공간: O(n) - 최대 재귀 깊이
int fibRecursive(int n) {
    if (n <= 1) return n;
    return fibRecursive(n - 1) + fibRecursive(n - 2);
}

// 피보나치 - 반복 (효율적)
// 시간: O(n)
// 공간: O(1)
int fibIterative(int n) {
    if (n <= 1) return n;
    int prev2 = 0, prev1 = 1;
    for (int i = 2; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    return prev1;
}

// 피보나치 - 메모이제이션
// 시간: O(n)
// 공간: O(n) - 메모 배열 + 재귀 스택
int[] memo;
int fibMemo(int n) {
    if (n <= 1) return n;
    if (memo[n] != 0) return memo[n];
    memo[n] = fibMemo(n - 1) + fibMemo(n - 2);
    return memo[n];
}
```

### 꼬리 재귀 최적화 (Tail Call Optimization)

```java
// 일반 재귀: O(n) 스택 공간
int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);  // 곱셈이 대기 중
}

// 꼬리 재귀: 최적화 시 O(1) 스택 공간
int factorialTail(int n, int acc) {
    if (n <= 1) return acc;
    return factorialTail(n - 1, n * acc);  // 대기 연산 없음
}

// 참고: Java는 TCO를 지원하지 않음
// Kotlin, Scala, Scheme 등은 지원
```

### Space-Time Tradeoff

메모리를 더 사용하여 시간을 단축하거나, 반대로 시간을 더 사용하여 메모리를 절약

#### 예시 1: 해시 테이블 vs 선형 탐색

```java
// 선형 탐색: O(1) 공간, O(n) 시간
boolean containsLinear(int[] arr, int target) {
    for (int x : arr) {
        if (x == target) return true;
    }
    return false;
}

// 해시 셋: O(n) 공간, O(1) 평균 시간
class FastLookup {
    private Set<Integer> set = new HashSet<>();

    void preprocess(int[] arr) {
        for (int x : arr) set.add(x);  // O(n) 공간 사용
    }

    boolean contains(int target) {
        return set.contains(target);  // O(1) 시간
    }
}
```

#### 예시 2: 메모이제이션 vs 재계산

```java
// 재계산: O(1) 공간, O(2^n) 시간
int fibNoMemo(int n) {
    if (n <= 1) return n;
    return fibNoMemo(n-1) + fibNoMemo(n-2);
}

// 메모이제이션: O(n) 공간, O(n) 시간
int fibWithMemo(int n, int[] cache) {
    if (n <= 1) return n;
    if (cache[n] != -1) return cache[n];
    cache[n] = fibWithMemo(n-1, cache) + fibWithMemo(n-2, cache);
    return cache[n];
}
```

#### 예시 3: 비트 벡터

```java
// 배열로 방문 체크: O(n) 공간
boolean[] visited = new boolean[n];

// 비트 벡터로 방문 체크: O(n/8) 공간 (8배 절약)
// n이 작을 때 (n <= 64) long 1개로 가능
long visited = 0L;
visited |= (1L << i);  // i번 방문
boolean isVisited = (visited & (1L << i)) != 0;
```

### 공간 복잡도 계산 예시

#### 그래프 표현

```java
// 인접 행렬: O(V²) 공간
int[][] adjMatrix = new int[V][V];

// 인접 리스트: O(V + E) 공간
List<List<Integer>> adjList = new ArrayList<>();
for (int i = 0; i < V; i++) {
    adjList.add(new ArrayList<>());
}

// 희소 그래프 (E << V²): 인접 리스트가 유리
// 밀집 그래프 (E ≈ V²): 인접 행렬도 고려
```

#### 동적 프로그래밍 최적화

```java
// LCS (Longest Common Subsequence)
// 기본: O(mn) 공간
int[][] dp = new int[m + 1][n + 1];
for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        if (s1.charAt(i-1) == s2.charAt(j-1)) {
            dp[i][j] = dp[i-1][j-1] + 1;
        } else {
            dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
        }
    }
}

// 공간 최적화: O(n) 공간
// 현재 행은 이전 행에만 의존
int[] prev = new int[n + 1];
int[] curr = new int[n + 1];
for (int i = 1; i <= m; i++) {
    for (int j = 1; j <= n; j++) {
        if (s1.charAt(i-1) == s2.charAt(j-1)) {
            curr[j] = prev[j-1] + 1;
        } else {
            curr[j] = Math.max(prev[j], curr[j-1]);
        }
    }
    int[] temp = prev;
    prev = curr;
    curr = temp;
}
```

### 자료구조별 공간 복잡도

| 자료구조 | 공간 복잡도 | 비고 |
|----------|-------------|------|
| 배열 | O(n) | 연속 메모리 |
| 연결 리스트 | O(n) | 노드당 포인터 오버헤드 |
| 해시 테이블 | O(n) | 부하율에 따라 변동 |
| 이진 트리 | O(n) | 노드당 2개 포인터 |
| 힙 (배열) | O(n) | 오버헤드 적음 |
| 그래프 (인접 리스트) | O(V + E) | 희소 그래프에 적합 |
| 그래프 (인접 행렬) | O(V²) | 밀집 그래프에 적합 |

### 알고리즘별 공간 복잡도

| 알고리즘 | Auxiliary Space | 비고 |
|----------|-----------------|------|
| 버블/선택/삽입 정렬 | O(1) | In-place |
| 힙 정렬 | O(1) | In-place |
| 퀵 정렬 | O(log n) | 재귀 스택 |
| 병합 정렬 | O(n) | 임시 배열 |
| BFS | O(V) | 큐 |
| DFS (재귀) | O(V) | 재귀 스택 |
| DFS (반복) | O(V) | 명시적 스택 |
| 다익스트라 | O(V) | 우선순위 큐 |

## 실무 적용

### 메모리 제한 환경

```java
// 대용량 파일 처리: 전체 로드 대신 스트리밍
// Bad: O(n) 공간
List<String> lines = Files.readAllLines(path);

// Good: O(1) 공간 (한 번에 한 줄만)
try (BufferedReader reader = Files.newBufferedReader(path)) {
    String line;
    while ((line = reader.readLine()) != null) {
        process(line);
    }
}
```

### 외부 정렬 (External Sort)

```java
/**
 * 메모리보다 큰 데이터 정렬
 *
 * 1단계: 메모리에 맞는 청크로 분할하여 정렬, 임시 파일로 저장
 * 2단계: K-way merge로 임시 파일들을 병합
 *
 * 공간: O(M) where M = 사용 가능한 메모리
 * I/O: O(N/B × log_{M/B}(N/B)) where B = 블록 크기
 */
void externalSort(String inputFile, String outputFile, long availableMemory) {
    // 구현 생략
}
```

### 메모리 프로파일링

```java
// Java에서 객체 크기 추정
// 기본 타입: int(4), long(8), double(8), reference(4 or 8)
// 객체 헤더: 보통 12-16 바이트
// 배열: 헤더 + length(4) + 원소들

// ArrayList<Integer> vs int[]
// ArrayList<Integer>: n × (4 + 16 + 4) = 24n 바이트 (Integer 객체 오버헤드)
// int[]: 16 + 4n 바이트 (배열 헤더 + 원소)
```

## 참고 자료

- Introduction to Algorithms (CLRS), Chapter 2: Getting Started
- The Art of Computer Programming Vol. 1, Section 2.2.6: Space Efficiency
- MIT 6.006 Lecture on Memory Hierarchy

# 배열 기초 (Array Fundamentals)

## 개요

배열(Array)은 **동일한 타입의 원소들이 연속된 메모리 공간**에 저장되는 가장 기본적인 자료구조이다. 인덱스를 통한 O(1) 랜덤 액세스가 가능하며, 캐시 효율성이 뛰어나 현대 컴퓨터에서 가장 빠른 자료구조 중 하나이다.

## 핵심 개념

### 메모리 레이아웃

```
배열의 메모리 구조:
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  10 │  20 │  30 │  40 │  50 │  60 │  70 │  80 │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
  [0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]

메모리 주소:
0x1000  0x1004  0x1008  0x100C  0x1010  0x1014  0x1018  0x101C
        (+4)    (+4)    (+4)    (+4)    (+4)    (+4)    (+4)

주소 계산 공식:
address(arr[i]) = base_address + i × element_size
```

```java
// Java에서 배열 선언과 메모리
int[] arr = new int[8];

// arr 변수: 배열 객체의 참조(주소)를 저장
// 실제 배열: 힙 메모리에 연속적으로 할당
// 각 int: 4바이트, 총 32바이트 + 객체 헤더
```

### Random Access (임의 접근)

**핵심 원리**: 주소 계산이 O(1)에 가능하므로 어떤 인덱스든 즉시 접근

```java
// O(1) 접근 - 인덱스로 바로 계산
int value = arr[5];  // base + 5 * 4 = 즉시 주소 계산

// vs 연결 리스트의 O(n) 접근
// head → node → node → node → node → node (5번 이동)
```

**왜 O(1)인가?**
1. 원소 크기가 고정되어 있음
2. 메모리가 연속적임
3. 곱셈/덧셈만으로 주소 계산 가능

### Cache Locality (캐시 지역성)

현대 CPU는 메모리 접근 시 캐시 라인(보통 64바이트)을 한 번에 로드한다. 배열은 연속된 메모리를 사용하므로 캐시 효율이 높다.

```
캐시 라인 로드 예시 (64바이트 캐시 라인, int 배열):

arr[0] 접근 시:
┌─────────────────────────────────────────────────────────────────┐
│  arr[0] ~ arr[15] 까지 16개의 int가 캐시에 로드됨               │
└─────────────────────────────────────────────────────────────────┘

이후 arr[1] ~ arr[15] 접근은 캐시 히트 (매우 빠름)
```

```java
// 캐시 친화적인 순회 (순차 접근)
// 캐시 히트율 높음
int sum = 0;
for (int i = 0; i < arr.length; i++) {
    sum += arr[i];  // 순차 접근
}

// 캐시 비친화적인 순회 (점프 접근)
// 캐시 미스율 높음
int sum = 0;
for (int i = 0; i < arr.length; i += 64) {
    sum += arr[i];  // 큰 간격으로 점프
}
```

**성능 차이 예시**:
```
순차 접근: ~1 ns/element (캐시 히트)
랜덤 접근: ~100 ns/element (메인 메모리 접근)
→ 최대 100배 차이
```

### Row-major vs Column-major Order

다차원 배열을 1차원 메모리에 저장하는 두 가지 방식:

```
2D 배열:
    col 0  col 1  col 2
row 0  [1]    [2]    [3]
row 1  [4]    [5]    [6]

Row-major (행 우선): C, C++, Java, Python
메모리: [1][2][3][4][5][6]
순서: (0,0) → (0,1) → (0,2) → (1,0) → (1,1) → (1,2)

Column-major (열 우선): Fortran, MATLAB, R
메모리: [1][4][2][5][3][6]
순서: (0,0) → (1,0) → (0,1) → (1,1) → (0,2) → (1,2)
```

**Row-major 주소 계산**:
```
address(arr[i][j]) = base + (i × cols + j) × element_size

예: arr[1][2], cols=3, element_size=4
address = base + (1 × 3 + 2) × 4 = base + 20
```

**성능 최적화**:
```java
// Row-major에서 캐시 효율적인 순회
// 행을 따라 순회 (메모리 순서와 일치)
for (int i = 0; i < rows; i++) {
    for (int j = 0; j < cols; j++) {
        process(matrix[i][j]);  // 캐시 히트
    }
}

// 비효율적인 순회 (열을 따라)
// 메모리 점프가 발생
for (int j = 0; j < cols; j++) {
    for (int i = 0; i < rows; i++) {
        process(matrix[i][j]);  // 캐시 미스 많음
    }
}
```

### 배열의 연산과 복잡도

| 연산 | 시간 복잡도 | 설명 |
|------|-------------|------|
| 접근 (Access) | O(1) | 인덱스로 직접 접근 |
| 검색 (Search) | O(n) | 선형 탐색 (정렬 시 O(log n)) |
| 삽입 (끝) | O(1) | 크기가 충분한 경우 |
| 삽입 (중간) | O(n) | 뒤의 원소들 이동 필요 |
| 삭제 (끝) | O(1) | |
| 삭제 (중간) | O(n) | 뒤의 원소들 이동 필요 |

```java
// 중간 삽입의 비용
void insertAt(int[] arr, int index, int value, int size) {
    // index부터 끝까지 모든 원소를 한 칸씩 뒤로 이동
    for (int i = size - 1; i >= index; i--) {
        arr[i + 1] = arr[i];  // O(n - index) 연산
    }
    arr[index] = value;
}

// 중간 삭제의 비용
void removeAt(int[] arr, int index, int size) {
    // index+1부터 끝까지 모든 원소를 한 칸씩 앞으로 이동
    for (int i = index; i < size - 1; i++) {
        arr[i] = arr[i + 1];  // O(n - index) 연산
    }
}
```

### 배열 vs 연결 리스트

| 특성 | 배열 | 연결 리스트 |
|------|------|-------------|
| 메모리 | 연속적 | 분산 |
| 접근 | O(1) | O(n) |
| 삽입/삭제 (중간) | O(n) | O(1)* |
| 메모리 오버헤드 | 없음 | 포인터 공간 |
| 캐시 효율 | 높음 | 낮음 |
| 크기 변경 | 비용 큼 | 쉬움 |

*위치를 알고 있는 경우

## 실무 적용

### 배열 기반 자료구조

```java
// 스택 (배열 기반)
class ArrayStack<T> {
    private Object[] array;
    private int top = -1;

    public void push(T item) {
        array[++top] = item;  // O(1)
    }

    public T pop() {
        return (T) array[top--];  // O(1)
    }
}

// 큐 (원형 배열)
class CircularQueue<T> {
    private Object[] array;
    private int front = 0, rear = 0;

    public void enqueue(T item) {
        array[rear] = item;
        rear = (rear + 1) % array.length;  // O(1)
    }

    public T dequeue() {
        T item = (T) array[front];
        front = (front + 1) % array.length;  // O(1)
        return item;
    }
}

// 힙 (배열 표현)
// parent(i) = (i-1)/2
// leftChild(i) = 2i + 1
// rightChild(i) = 2i + 2
```

### 언어별 배열 특성

```java
// Java
int[] primitiveArray = new int[10];      // 힙에 할당, 0으로 초기화
Integer[] objectArray = new Integer[10]; // null로 초기화
int[][] jagged = new int[3][];           // 가변 길이 행 가능

// Python
import array
arr = array.array('i', [1, 2, 3])  # 타입 고정 배열
list = [1, 2, 3]                    # 동적 배열 (내부적으로)

// C/C++
int arr[10];              // 스택에 할당
int* arr = new int[10];   // 힙에 할당
```

### 배열 사용 시 주의점

```java
// 1. 인덱스 범위 체크
if (index >= 0 && index < arr.length) {
    return arr[index];
}

// 2. 배열 복사 (얕은 복사 vs 깊은 복사)
int[] shallow = arr;               // 같은 배열 참조
int[] deep = arr.clone();          // 새 배열 생성 (1차원)
int[][] deep2D = Arrays.stream(matrix)
    .map(int[]::clone)
    .toArray(int[][]::new);        // 2차원 깊은 복사

// 3. 배열 크기 변경
// Java에서 배열 크기는 고정, 새 배열로 복사 필요
int[] newArr = Arrays.copyOf(arr, newSize);
```

## 참고 자료

- Introduction to Algorithms (CLRS), Chapter 10.1: Stacks and Queues
- Computer Systems: A Programmer's Perspective, Chapter 6: Memory Hierarchy
- What Every Programmer Should Know About Memory - Ulrich Drepper

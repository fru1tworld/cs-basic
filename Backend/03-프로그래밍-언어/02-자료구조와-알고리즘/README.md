# 자료구조와 알고리즘

## 목차
1. [시간/공간 복잡도 분석](#1-시간공간-복잡도-분석)
2. [핵심 자료구조](#2-핵심-자료구조)
3. [정렬 알고리즘](#3-정렬-알고리즘)
4. [탐색 알고리즘](#4-탐색-알고리즘)
5. [동적 프로그래밍](#5-동적-프로그래밍)
6. [그래프 알고리즘](#6-그래프-알고리즘)
---

## 1. 시간/공간 복잡도 분석

### 1.1 Big-O 표기법

알고리즘의 효율성을 입력 크기 n에 대한 함수로 표현한다.

```
┌────────────────────────────────────────────────────────────┐
│                     시간 복잡도 비교                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  시간 ▲                                                    │
│       │                                         O(n!)     │
│       │                                    O(2^n)         │
│       │                               O(n²)               │
│       │                          O(n log n)               │
│       │                     O(n)                          │
│       │               O(log n)                            │
│       │         O(1)                                      │
│       └─────────────────────────────────────────▶ n       │
└────────────────────────────────────────────────────────────┘
```

### 1.2 주요 시간 복잡도

| 복잡도 | 이름 | 예시 | n=1,000일 때 연산 수 |
|--------|------|------|---------------------|
| O(1) | 상수 | 배열 인덱스 접근, 해시맵 조회 | 1 |
| O(log n) | 로그 | 이진 탐색, 균형 트리 연산 | ~10 |
| O(n) | 선형 | 배열 순회, 선형 탐색 | 1,000 |
| O(n log n) | 선형 로그 | 병합 정렬, 퀵 정렬 (평균) | ~10,000 |
| O(n²) | 이차 | 버블 정렬, 이중 반복문 | 1,000,000 |
| O(n³) | 삼차 | 행렬 곱셈 (naive) | 1,000,000,000 |
| O(2^n) | 지수 | 부분집합, 피보나치 (naive) | 천문학적 |
| O(n!) | 팩토리얼 | 순열 생성, 외판원 (brute force) | 계산 불가 |

### 1.3 복잡도 분석 예제

```python
# O(1) - 상수 시간
def get_first(arr):
    return arr[0] if arr else None

# O(log n) - 로그 시간
def binary_search(arr, target):
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = (left + right) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1

# O(n) - 선형 시간
def find_max(arr):
    max_val = arr[0]
    for num in arr:  # n번 반복
        if num > max_val:
            max_val = num
    return max_val

# O(n log n) - 선형 로그 시간
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])   # T(n/2)
    right = merge_sort(arr[mid:])  # T(n/2)
    return merge(left, right)      # O(n)
    # T(n) = 2T(n/2) + O(n) = O(n log n)

# O(n²) - 이차 시간
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):           # n번
        for j in range(n-i-1):   # n-i-1번
            if arr[j] > arr[j+1]:
                arr[j], arr[j+1] = arr[j+1], arr[j]
    return arr
```

```java
// O(2^n) - 지수 시간 (피보나치 naive)
public class Fibonacci {
    // 재귀 호출이 트리 형태로 증가
    public static int fibNaive(int n) {
        if (n <= 1) return n;
        return fibNaive(n - 1) + fibNaive(n - 2);
    }

    // O(n) - 동적 프로그래밍으로 최적화
    public static int fibDP(int n) {
        if (n <= 1) return n;
        int[] dp = new int[n + 1];
        dp[0] = 0;
        dp[1] = 1;
        for (int i = 2; i <= n; i++) {
            dp[i] = dp[i-1] + dp[i-2];
        }
        return dp[n];
    }
}
```

### 1.4 공간 복잡도

```python
# O(1) 공간 - 상수 공간
def sum_array(arr):
    total = 0  # 변수 1개만 사용
    for num in arr:
        total += num
    return total

# O(n) 공간 - 선형 공간
def reverse_array(arr):
    result = []  # 입력 크기만큼 새 배열
    for i in range(len(arr) - 1, -1, -1):
        result.append(arr[i])
    return result

# O(n) 공간 - 재귀 호출 스택
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)  # 호출 스택 깊이 = n

# O(log n) 공간 - 이진 탐색 재귀
def binary_search_recursive(arr, target, left, right):
    if left > right:
        return -1
    mid = (left + right) // 2
    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search_recursive(arr, target, mid + 1, right)
    else:
        return binary_search_recursive(arr, target, left, mid - 1)
```

### 1.5 Amortized Analysis (분할 상환 분석)

```java
// ArrayList의 add 연산
// 최악: O(n) - 배열 확장 시
// 평균(분할 상환): O(1)

public class DynamicArray<T> {
    private Object[] data;
    private int size;
    private int capacity;

    public DynamicArray() {
        this.capacity = 10;
        this.data = new Object[capacity];
        this.size = 0;
    }

    // 분할 상환 O(1)
    public void add(T element) {
        if (size == capacity) {
            // O(n) - 가끔 발생
            resize();
        }
        data[size++] = element;  // O(1)
    }

    private void resize() {
        capacity *= 2;  // 2배로 확장
        Object[] newData = new Object[capacity];
        System.arraycopy(data, 0, newData, 0, size);
        data = newData;
    }

    // n개 삽입 시 총 비용:
    // n + (1 + 2 + 4 + 8 + ... + n) ≈ 3n
    // 따라서 평균: O(1)
}
```

---

## 2. 핵심 자료구조

### 2.1 Array (배열)

```
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  Index
├───┼───┼───┼───┼───┼───┼───┼───┤
│ A │ B │ C │ D │ E │ F │ G │ H │  Value
└───┴───┴───┴───┴───┴───┴───┴───┘
  ↑
Base Address (연속된 메모리)
```

| 연산 | 시간 복잡도 |
|------|------------|
| 인덱스 접근 | O(1) |
| 맨 끝 삽입/삭제 | O(1) |
| 중간 삽입/삭제 | O(n) |
| 탐색 (정렬 X) | O(n) |
| 탐색 (정렬 O) | O(log n) |

```java
// Java - 배열 vs ArrayList
public class ArrayExample {
    public static void main(String[] args) {
        // 정적 배열
        int[] staticArray = new int[5];
        staticArray[0] = 10;

        // 동적 배열 (ArrayList)
        ArrayList<Integer> dynamicArray = new ArrayList<>();
        dynamicArray.add(10);  // O(1) amortized
        dynamicArray.get(0);   // O(1)
        dynamicArray.add(0, 5); // O(n) - 맨 앞 삽입
    }
}
```

### 2.2 LinkedList (연결 리스트)

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│ Data: A    │    │ Data: B    │    │ Data: C    │    │ Data: D    │
│ Next: ─────┼───▶│ Next: ─────┼───▶│ Next: ─────┼───▶│ Next: null │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
     Head                                                   Tail

Doubly Linked List:
┌────────────┐    ┌────────────┐    ┌────────────┐
│ Prev: null │◀───┤ Prev: ─────│◀───┤ Prev: ─────│
│ Data: A    │    │ Data: B    │    │ Data: C    │
│ Next: ─────┼───▶│ Next: ─────┼───▶│ Next: null │
└────────────┘    └────────────┘    └────────────┘
```

| 연산 | 시간 복잡도 |
|------|------------|
| 맨 앞 삽입/삭제 | O(1) |
| 맨 뒤 삽입/삭제 (tail 유지 시) | O(1) |
| 중간 삽입/삭제 (위치 알 때) | O(1) |
| 인덱스 접근 | O(n) |
| 탐색 | O(n) |

```python
# Python - LinkedList 구현
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

class LinkedList:
    def __init__(self):
        self.head = None
        self.size = 0

    def add_first(self, data):  # O(1)
        new_node = Node(data)
        new_node.next = self.head
        self.head = new_node
        self.size += 1

    def add_last(self, data):  # O(n) - tail 없는 경우
        new_node = Node(data)
        if not self.head:
            self.head = new_node
            return
        current = self.head
        while current.next:
            current = current.next
        current.next = new_node
        self.size += 1

    def remove(self, data):  # O(n)
        if not self.head:
            return False
        if self.head.data == data:
            self.head = self.head.next
            self.size -= 1
            return True
        current = self.head
        while current.next:
            if current.next.data == data:
                current.next = current.next.next
                self.size -= 1
                return True
            current = current.next
        return False
```

### 2.3 Stack (스택)

```
     ┌─────┐
     │  D  │  ← Top (Last In)
     ├─────┤
     │  C  │
     ├─────┤
     │  B  │
     ├─────┤
     │  A  │  ← Bottom (First In)
     └─────┘

     LIFO: Last In First Out
```

```java
// Java - Stack 구현 및 활용
public class StackExample {
    public static void main(String[] args) {
        // java.util.Stack 사용 (Vector 기반, 권장하지 않음)
        Stack<Integer> stack = new Stack<>();

        // Deque 사용 권장
        Deque<Integer> betterStack = new ArrayDeque<>();
        betterStack.push(1);  // O(1)
        betterStack.push(2);
        int top = betterStack.pop();  // O(1), 2 반환
        int peek = betterStack.peek();  // O(1), 1 반환 (제거 안함)
    }

    // 활용 예: 괄호 유효성 검사
    public static boolean isValidParentheses(String s) {
        Deque<Character> stack = new ArrayDeque<>();
        Map<Character, Character> pairs = Map.of(
            ')', '(',
            '}', '{',
            ']', '['
        );

        for (char c : s.toCharArray()) {
            if (c == '(' || c == '{' || c == '[') {
                stack.push(c);
            } else if (pairs.containsKey(c)) {
                if (stack.isEmpty() || stack.pop() != pairs.get(c)) {
                    return false;
                }
            }
        }
        return stack.isEmpty();
    }
}
```

### 2.4 Queue (큐)

```
Front                              Rear
  ↓                                  ↓
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ A │ B │ C │ D │ E │ F │ G │ H │
└───┴───┴───┴───┴───┴───┴───┴───┘
  ↑                              ↑
Dequeue                       Enqueue

FIFO: First In First Out
```

```python
from collections import deque
import queue

# deque 사용 (일반적인 경우)
q = deque()
q.append(1)     # enqueue - O(1)
q.append(2)
front = q.popleft()  # dequeue - O(1)

# Priority Queue (우선순위 큐)
import heapq

pq = []
heapq.heappush(pq, (3, "low"))      # O(log n)
heapq.heappush(pq, (1, "high"))
heapq.heappush(pq, (2, "medium"))

# 우선순위가 높은(숫자가 낮은) 순서로 pop
while pq:
    priority, task = heapq.heappop(pq)  # O(log n)
    print(f"{task}: {priority}")
# 출력: high: 1, medium: 2, low: 3
```

### 2.5 Tree (트리)

```
               ┌───┐
               │ 8 │  Root
               └─┬─┘
           ┌────┴────┐
         ┌─┴─┐     ┌─┴─┐
         │ 3 │     │10 │
         └─┬─┘     └─┬─┘
        ┌──┴──┐      └──┐
      ┌─┴─┐ ┌─┴─┐    ┌──┴─┐
      │ 1 │ │ 6 │    │ 14 │
      └───┘ └─┬─┘    └──┬─┘
           ┌──┴──┐   ┌──┴──┐
         ┌─┴─┐ ┌─┴─┐ │ 13 │
         │ 4 │ │ 7 │ └────┘
         └───┘ └───┘

BST (Binary Search Tree):
- 왼쪽 서브트리: 부모보다 작은 값
- 오른쪽 서브트리: 부모보다 큰 값
```

```java
// Java - BST 구현
public class BinarySearchTree {
    private Node root;

    private static class Node {
        int value;
        Node left, right;

        Node(int value) {
            this.value = value;
        }
    }

    // O(log n) 평균, O(n) 최악 (편향 트리)
    public void insert(int value) {
        root = insertRecursive(root, value);
    }

    private Node insertRecursive(Node node, int value) {
        if (node == null) return new Node(value);

        if (value < node.value) {
            node.left = insertRecursive(node.left, value);
        } else if (value > node.value) {
            node.right = insertRecursive(node.right, value);
        }
        return node;
    }

    // 중위 순회 (In-order): 정렬된 순서 출력
    public void inOrder(Node node) {
        if (node == null) return;
        inOrder(node.left);     // 왼쪽
        System.out.print(node.value + " ");  // 현재
        inOrder(node.right);    // 오른쪽
    }

    // 전위 순회 (Pre-order): 복사에 유용
    public void preOrder(Node node) {
        if (node == null) return;
        System.out.print(node.value + " ");  // 현재
        preOrder(node.left);    // 왼쪽
        preOrder(node.right);   // 오른쪽
    }

    // 후위 순회 (Post-order): 삭제에 유용
    public void postOrder(Node node) {
        if (node == null) return;
        postOrder(node.left);   // 왼쪽
        postOrder(node.right);  // 오른쪽
        System.out.print(node.value + " ");  // 현재
    }
}
```

#### 균형 이진 트리

| 트리 종류 | 특징 | 삽입/삭제/탐색 |
|----------|------|---------------|
| AVL 트리 | 엄격한 균형 (높이 차 ≤ 1) | O(log n) |
| Red-Black 트리 | 느슨한 균형, Java TreeMap | O(log n) |
| B-트리 | 다진 트리, DB 인덱스 | O(log n) |
| B+ 트리 | B-트리 변형, 리프에만 데이터 | O(log n) |

### 2.6 Heap (힙)

```
Max Heap (최대 힙):
               ┌────┐
               │ 90 │
               └─┬──┘
           ┌────┴────┐
         ┌─┴─┐     ┌─┴─┐
         │79 │     │72 │
         └─┬─┘     └─┬─┘
        ┌──┴──┐   ┌──┴──┐
      ┌─┴─┐ ┌─┴─┐│ 65 ││ 55 │
      │62 │ │52 │└────┘└────┘
      └───┘ └───┘

배열 표현: [90, 79, 72, 62, 52, 65, 55]
           0   1   2   3   4   5   6

부모: (i-1) / 2
왼쪽 자식: 2*i + 1
오른쪽 자식: 2*i + 2
```

```python
import heapq

# Python heapq는 최소 힙
min_heap = []
heapq.heappush(min_heap, 5)  # O(log n)
heapq.heappush(min_heap, 3)
heapq.heappush(min_heap, 7)
heapq.heappush(min_heap, 1)

print(heapq.heappop(min_heap))  # 1 (최소값)

# 최대 힙 구현 (값에 -1 곱하기)
max_heap = []
for num in [5, 3, 7, 1]:
    heapq.heappush(max_heap, -num)

print(-heapq.heappop(max_heap))  # 7 (최대값)

# 힙 정렬 O(n log n)
def heap_sort(arr):
    heapq.heapify(arr)  # O(n)
    return [heapq.heappop(arr) for _ in range(len(arr))]

# Top K 문제
def find_top_k(arr, k):
    return heapq.nlargest(k, arr)  # O(n log k)
```

### 2.7 Hash Table (해시 테이블)

```
Key: "apple" ──▶ hash("apple") = 3 ──┐
                                      │
┌─────┬──────────────────────────┐   │
│  0  │ null                     │   │
├─────┼──────────────────────────┤   │
│  1  │ ("banana", 2) → null     │   │
├─────┼──────────────────────────┤   │
│  2  │ null                     │   │
├─────┼──────────────────────────┤   │
│  3  │ ("apple", 5) → ("grape", 8) → null │ ← Collision (Chaining)
├─────┼──────────────────────────┤
│  4  │ ("orange", 3) → null     │
└─────┴──────────────────────────┘
```

| 연산 | 평균 | 최악 |
|------|------|------|
| 삽입 | O(1) | O(n) |
| 삭제 | O(1) | O(n) |
| 탐색 | O(1) | O(n) |

```java
// Java HashMap 내부 동작
public class HashMapExample {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();

        // O(1) 평균
        map.put("apple", 5);    // hash("apple") % capacity
        map.put("banana", 3);
        map.get("apple");       // 5

        // Java 8+: 버킷 크기가 8 이상이면 Red-Black Tree로 변환
        // → 최악 O(log n)으로 개선

        // Load Factor: 0.75 (기본값)
        // 용량의 75% 찼을 때 2배로 확장 (rehashing)
    }
}

// 충돌 해결 방법
// 1. Chaining: 연결 리스트로 연결
// 2. Open Addressing: 다른 빈 슬롯 탐색
//    - Linear Probing: 순차 탐색
//    - Quadratic Probing: 제곱 간격 탐색
//    - Double Hashing: 두 번째 해시 함수 사용
```

```python
# Python - 해시 테이블 활용
from collections import Counter, defaultdict

# 빈도수 계산
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
counter = Counter(words)
print(counter.most_common(2))  # [('apple', 3), ('banana', 2)]

# 그룹화
from itertools import groupby

def group_anagrams(strs):
    anagrams = defaultdict(list)
    for s in strs:
        key = tuple(sorted(s))  # 정렬된 문자열을 키로
        anagrams[key].append(s)
    return list(anagrams.values())

# Two Sum 문제 - O(n)
def two_sum(nums, target):
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i
    return []
```

### 2.8 Graph (그래프)

```
무방향 그래프:
    A ─── B
    │ ╲   │
    │   ╲ │
    D ─── C

방향 그래프:
    A ───▶ B
    │      │
    ▼      ▼
    D ◀─── C

가중치 그래프:
    A ──5── B
    │       │
    3       2
    │       │
    D ──4── C
```

```python
# 그래프 표현 방법

# 1. 인접 행렬 (Adjacency Matrix)
# 공간: O(V²), 간선 확인: O(1)
adj_matrix = [
    [0, 1, 1, 1],  # A -> B, C, D
    [1, 0, 1, 0],  # B -> A, C
    [1, 1, 0, 1],  # C -> A, B, D
    [1, 0, 1, 0],  # D -> A, C
]

# 2. 인접 리스트 (Adjacency List)
# 공간: O(V + E), 간선 확인: O(degree)
adj_list = {
    'A': ['B', 'C', 'D'],
    'B': ['A', 'C'],
    'C': ['A', 'B', 'D'],
    'D': ['A', 'C'],
}

# 가중치 그래프 인접 리스트
weighted_graph = {
    'A': [('B', 5), ('D', 3)],
    'B': [('A', 5), ('C', 2)],
    'C': [('B', 2), ('D', 4)],
    'D': [('A', 3), ('C', 4)],
}
```

---

## 3. 정렬 알고리즘

### 3.1 정렬 알고리즘 비교

| 알고리즘 | 평균 | 최악 | 공간 | 안정성 |
|---------|------|------|------|--------|
| 버블 정렬 | O(n²) | O(n²) | O(1) | 안정 |
| 선택 정렬 | O(n²) | O(n²) | O(1) | 불안정 |
| 삽입 정렬 | O(n²) | O(n²) | O(1) | 안정 |
| 병합 정렬 | O(n log n) | O(n log n) | O(n) | 안정 |
| 퀵 정렬 | O(n log n) | O(n²) | O(log n) | 불안정 |
| 힙 정렬 | O(n log n) | O(n log n) | O(1) | 불안정 |
| 계수 정렬 | O(n + k) | O(n + k) | O(k) | 안정 |
| 기수 정렬 | O(d(n + k)) | O(d(n + k)) | O(n + k) | 안정 |
| 팀 정렬 | O(n log n) | O(n log n) | O(n) | 안정 |

> **안정 정렬(Stable Sort)**: 동일한 값의 원소들이 정렬 후에도 원래 순서를 유지

### 3.2 Quick Sort (퀵 정렬)

```python
def quick_sort(arr, low=0, high=None):
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition(arr, low, high)
        quick_sort(arr, low, pivot_idx - 1)
        quick_sort(arr, pivot_idx + 1, high)
    return arr

def partition(arr, low, high):
    # 피벗 선택 (마지막 원소)
    pivot = arr[high]
    i = low - 1

    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1

# 피벗 선택 최적화: Median of Three
def median_of_three(arr, low, high):
    mid = (low + high) // 2
    if arr[low] > arr[mid]:
        arr[low], arr[mid] = arr[mid], arr[low]
    if arr[low] > arr[high]:
        arr[low], arr[high] = arr[high], arr[low]
    if arr[mid] > arr[high]:
        arr[mid], arr[high] = arr[high], arr[mid]
    return mid
```

### 3.3 Merge Sort (병합 정렬)

```java
public class MergeSort {
    public static void mergeSort(int[] arr, int left, int right) {
        if (left < right) {
            int mid = left + (right - left) / 2;

            mergeSort(arr, left, mid);      // 왼쪽 정렬
            mergeSort(arr, mid + 1, right); // 오른쪽 정렬
            merge(arr, left, mid, right);   // 병합
        }
    }

    private static void merge(int[] arr, int left, int mid, int right) {
        // 임시 배열 생성
        int[] leftArr = Arrays.copyOfRange(arr, left, mid + 1);
        int[] rightArr = Arrays.copyOfRange(arr, mid + 1, right + 1);

        int i = 0, j = 0, k = left;

        // 두 배열 병합
        while (i < leftArr.length && j < rightArr.length) {
            if (leftArr[i] <= rightArr[j]) {
                arr[k++] = leftArr[i++];
            } else {
                arr[k++] = rightArr[j++];
            }
        }

        // 나머지 복사
        while (i < leftArr.length) arr[k++] = leftArr[i++];
        while (j < rightArr.length) arr[k++] = rightArr[j++];
    }
}
```

### 3.4 Tim Sort

Python과 Java의 기본 정렬 알고리즘. 병합 정렬과 삽입 정렬의 하이브리드.

```python
# Python의 sorted()와 list.sort()는 Tim Sort 사용

# Tim Sort 특징:
# 1. 작은 배열에는 삽입 정렬 (Run 생성)
# 2. Run들을 병합 정렬로 합침
# 3. 이미 정렬된 데이터에서 O(n) 성능
# 4. 안정 정렬

# Java의 Arrays.sort()
# - 기본형(int, double 등): Dual-Pivot Quick Sort
# - 객체(Object[]): Tim Sort
```

---

## 4. 탐색 알고리즘

### 4.1 Binary Search (이진 탐색)

```python
def binary_search(arr, target):
    """정렬된 배열에서 target 찾기 - O(log n)"""
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2  # 오버플로우 방지

        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1

# 변형: Lower Bound (target 이상인 첫 번째 인덱스)
def lower_bound(arr, target):
    left, right = 0, len(arr)
    while left < right:
        mid = (left + right) // 2
        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid
    return left

# 변형: Upper Bound (target 초과인 첫 번째 인덱스)
def upper_bound(arr, target):
    left, right = 0, len(arr)
    while left < right:
        mid = (left + right) // 2
        if arr[mid] <= target:
            left = mid + 1
        else:
            right = mid
    return left

# 활용: 정답을 이진 탐색 (Parametric Search)
def min_max_problem(arr, k):
    """최댓값의 최솟값 구하기"""
    left, right = max(arr), sum(arr)

    while left < right:
        mid = (left + right) // 2
        if can_divide(arr, mid, k):
            right = mid
        else:
            left = mid + 1

    return left
```

---

## 5. 동적 프로그래밍

### 5.1 DP 기본 개념

```
동적 프로그래밍 조건:
1. Optimal Substructure (최적 부분 구조)
   - 문제의 최적해가 부분 문제의 최적해로 구성

2. Overlapping Subproblems (중복되는 부분 문제)
   - 동일한 부분 문제가 반복적으로 계산됨

접근 방법:
1. Top-Down (Memoization): 재귀 + 캐싱
2. Bottom-Up (Tabulation): 반복문으로 테이블 채우기
```

### 5.2 대표 DP 문제

```python
# 1. 피보나치 수열
# Top-Down (Memoization)
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_memo(n):
    if n <= 1:
        return n
    return fib_memo(n-1) + fib_memo(n-2)

# Bottom-Up (Tabulation)
def fib_tabulation(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# 공간 최적화
def fib_optimized(n):
    if n <= 1:
        return n
    prev2, prev1 = 0, 1
    for _ in range(2, n + 1):
        curr = prev1 + prev2
        prev2, prev1 = prev1, curr
    return prev1
```

```java
// 2. 최장 공통 부분 수열 (LCS)
public class LCS {
    public static int lcs(String s1, String s2) {
        int m = s1.length();
        int n = s2.length();
        int[][] dp = new int[m + 1][n + 1];

        // dp[i][j] = s1[0..i-1]과 s2[0..j-1]의 LCS 길이
        for (int i = 1; i <= m; i++) {
            for (int j = 1; j <= n; j++) {
                if (s1.charAt(i-1) == s2.charAt(j-1)) {
                    dp[i][j] = dp[i-1][j-1] + 1;
                } else {
                    dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
                }
            }
        }

        return dp[m][n];
    }
}
```

```python
# 3. 0/1 배낭 문제 (Knapsack)
def knapsack(weights, values, capacity):
    n = len(weights)
    # dp[i][w] = i번째 아이템까지 고려, 용량 w일 때 최대 가치
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            # i번째 아이템을 넣지 않는 경우
            dp[i][w] = dp[i-1][w]

            # i번째 아이템을 넣는 경우
            if weights[i-1] <= w:
                dp[i][w] = max(
                    dp[i][w],
                    dp[i-1][w - weights[i-1]] + values[i-1]
                )

    return dp[n][capacity]

# 공간 최적화: O(n*W) -> O(W)
def knapsack_optimized(weights, values, capacity):
    dp = [0] * (capacity + 1)

    for i in range(len(weights)):
        # 역순으로 순회 (이전 상태 보존)
        for w in range(capacity, weights[i] - 1, -1):
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]
```

```python
# 4. 최장 증가 부분 수열 (LIS)
# O(n²) 풀이
def lis_n2(arr):
    n = len(arr)
    dp = [1] * n  # dp[i] = arr[i]로 끝나는 LIS 길이

    for i in range(1, n):
        for j in range(i):
            if arr[j] < arr[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)

# O(n log n) 풀이 - 이진 탐색 활용
import bisect

def lis_nlogn(arr):
    tails = []  # tails[i] = 길이 i+1인 증가 수열의 마지막 원소 최솟값

    for num in arr:
        idx = bisect.bisect_left(tails, num)
        if idx == len(tails):
            tails.append(num)
        else:
            tails[idx] = num

    return len(tails)
```

---

## 6. 그래프 알고리즘

### 6.1 BFS (너비 우선 탐색)

```
시작점으로부터의 거리 순서로 탐색
레벨 0: A
레벨 1: B, C, D
레벨 2: E, F
레벨 3: G

탐색 순서: A → B → C → D → E → F → G
```

```python
from collections import deque

def bfs(graph, start):
    visited = set([start])
    queue = deque([start])
    result = []

    while queue:
        node = queue.popleft()
        result.append(node)

        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

    return result

# 최단 거리 (가중치 없는 그래프)
def shortest_path_bfs(graph, start, end):
    visited = {start: 0}  # 거리 기록
    queue = deque([start])

    while queue:
        node = queue.popleft()

        if node == end:
            return visited[end]

        for neighbor in graph[node]:
            if neighbor not in visited:
                visited[neighbor] = visited[node] + 1
                queue.append(neighbor)

    return -1  # 도달 불가
```

### 6.2 DFS (깊이 우선 탐색)

```python
# 재귀 DFS
def dfs_recursive(graph, node, visited=None):
    if visited is None:
        visited = set()

    visited.add(node)
    result = [node]

    for neighbor in graph[node]:
        if neighbor not in visited:
            result.extend(dfs_recursive(graph, neighbor, visited))

    return result

# 스택 DFS
def dfs_iterative(graph, start):
    visited = set()
    stack = [start]
    result = []

    while stack:
        node = stack.pop()
        if node not in visited:
            visited.add(node)
            result.append(node)

            # 역순으로 추가 (순서 유지)
            for neighbor in reversed(graph[node]):
                if neighbor not in visited:
                    stack.append(neighbor)

    return result

# 사이클 탐지
def has_cycle(graph):
    visited = set()
    rec_stack = set()  # 현재 경로의 노드들

    def dfs(node):
        visited.add(node)
        rec_stack.add(node)

        for neighbor in graph[node]:
            if neighbor not in visited:
                if dfs(neighbor):
                    return True
            elif neighbor in rec_stack:
                return True  # 사이클 발견

        rec_stack.remove(node)
        return False

    for node in graph:
        if node not in visited:
            if dfs(node):
                return True
    return False
```

### 6.3 Dijkstra's Algorithm

```
가중치 그래프에서 최단 경로

      2
  A ────── B
  │        │
 1│        │3
  │        │
  D ────── C
      5

A에서 시작:
A: 0
B: 2 (A→B)
D: 1 (A→D)
C: 5 (A→B→C) 또는 6 (A→D→C) → 최소: 5
```

```python
import heapq

def dijkstra(graph, start):
    # graph = {node: [(neighbor, weight), ...]}
    distances = {node: float('inf') for node in graph}
    distances[start] = 0
    pq = [(0, start)]  # (distance, node)

    while pq:
        current_dist, current = heapq.heappop(pq)

        # 이미 처리된 노드 스킵
        if current_dist > distances[current]:
            continue

        for neighbor, weight in graph[current]:
            distance = current_dist + weight

            # 더 짧은 경로 발견
            if distance < distances[neighbor]:
                distances[neighbor] = distance
                heapq.heappush(pq, (distance, neighbor))

    return distances

# 경로 복원
def dijkstra_with_path(graph, start, end):
    distances = {node: float('inf') for node in graph}
    distances[start] = 0
    previous = {node: None for node in graph}
    pq = [(0, start)]

    while pq:
        current_dist, current = heapq.heappop(pq)

        if current == end:
            break

        if current_dist > distances[current]:
            continue

        for neighbor, weight in graph[current]:
            distance = current_dist + weight
            if distance < distances[neighbor]:
                distances[neighbor] = distance
                previous[neighbor] = current
                heapq.heappush(pq, (distance, neighbor))

    # 경로 복원
    path = []
    node = end
    while node is not None:
        path.append(node)
        node = previous[node]
    path.reverse()

    return distances[end], path
```

### 6.4 기타 그래프 알고리즘

```python
# 위상 정렬 (Topological Sort) - DAG용
from collections import deque

def topological_sort(graph, indegree):
    queue = deque([node for node in graph if indegree[node] == 0])
    result = []

    while queue:
        node = queue.popleft()
        result.append(node)

        for neighbor in graph[node]:
            indegree[neighbor] -= 1
            if indegree[neighbor] == 0:
                queue.append(neighbor)

    if len(result) != len(graph):
        return None  # 사이클 존재
    return result

# Union-Find (Disjoint Set)
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        # 경로 압축
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        # 랭크 기반 합치기
        px, py = self.find(x), self.find(y)
        if px == py:
            return False

        if self.rank[px] < self.rank[py]:
            px, py = py, px
        self.parent[py] = px
        if self.rank[px] == self.rank[py]:
            self.rank[px] += 1
        return True

# 크루스칼 알고리즘 (MST)
def kruskal(n, edges):
    # edges = [(weight, u, v), ...]
    edges.sort()
    uf = UnionFind(n)
    mst_weight = 0
    mst_edges = []

    for weight, u, v in edges:
        if uf.union(u, v):
            mst_weight += weight
            mst_edges.append((u, v, weight))

    return mst_weight, mst_edges
```

---

## 참고 자료

- "Introduction to Algorithms" (CLRS) - Cormen, Leiserson, Rivest, Stein
- "알고리즘 문제 해결 전략" - 구종만
- LeetCode, 프로그래머스 - 알고리즘 문제 풀이
- GeeksforGeeks - Data Structures and Algorithms
